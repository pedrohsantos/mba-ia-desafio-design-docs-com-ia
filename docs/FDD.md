# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Pronto para implementação, sujeito à revisão de segurança |
| **Base** | [RFC-001](RFC.md) · [ADRs](adrs/README.md) · [PRD](PRD.md) |
| **Rastreabilidade** | Cada item com ID (`FDD-…`) tem linha no [TRACKER](TRACKER.md) |

> O RFC explica **o que** propomos e **por quê**; este FDD explica **como construir**. Decisões arquiteturais não são rediscutidas aqui: são referenciadas pelos ADRs.

---

## 1. Contexto e motivação técnica

Hoje a mudança de status de um pedido acontece apenas em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma `prisma.$transaction` que:

1. carrega o pedido com `items`;
2. rejeita transição para o mesmo status (`ConflictError` com `INVALID_STATUS_TRANSITION`) e transições inválidas (`InvalidStatusTransitionError`, a partir de `canTransition` em `src/modules/orders/order.status.ts`);
3. debita estoque em `PENDING → PAID` (`shouldDebitStock`) ou repõe em `PAID|PROCESSING → CANCELLED` (`shouldReplenishStock`);
4. atualiza `orders.status` e insere em `order_status_history`;
5. relê o pedido com relações e devolve.

Não há nenhum mecanismo de eventos, filas ou notificação externa. Os clientes B2B fazem polling em `GET /api/v1/orders` ([09:00] Marcos). A feature adiciona um **canal de saída confiável** sem tocar na semântica atual da máquina de estados, e o ponto de extensão é a transação acima ([09:40] Bruno).

## 2. Objetivos técnicos

| ID | Objetivo | Critério verificável | Origem |
|---|---|---|---|
| FDD-OBJ-01 | Atomicidade entre mudança de status e registro do evento | Se o `INSERT` na outbox falhar, `orders.status` e `order_status_history` não mudam (teste de rollback) | [09:40] Bruno |
| FDD-OBJ-02 | Latência de entrega abaixo de 10 s em operação normal | `delivered_at − created_at` < 10 s para entregas de primeira tentativa com cliente respondendo em < 1 s | [09:02] Marcos |
| FDD-OBJ-03 | Nenhum evento commitado é perdido | Eventos sobrevivem a restart da API e do worker; `PROCESSING` órfão volta para `PENDING` | [09:06] Diego |
| FDD-OBJ-04 | Isolamento de falhas do cliente | Cliente lento ou offline não altera a latência de `PATCH /orders/:id/status` | [09:04] Bruno |
| FDD-OBJ-05 | Envio autenticável pelo cliente | Todo `POST` carrega `X-Signature` HMAC-SHA256 válido para a secret do endpoint | [09:20] Sofia |
| FDD-OBJ-06 | Zero infraestrutura e dependências novas | `package.json` sem novas dependências de runtime; mesmo MySQL | [09:07] Diego, [09:29] Bruno |

## 3. Escopo e exclusões

### 3.1 Dentro do escopo

- Módulo `src/modules/webhooks/` (CRUD de configuração, rotação de secret, histórico de entregas, replay de DLQ).
- Tabelas `webhooks`, `webhook_outbox` e `webhook_dead_letter` com migration Prisma.
- Publicação transacional em `changeStatus` via `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
- Worker `src/worker.ts` + `src/modules/webhooks/webhook.processor.ts` + script `npm run worker`.
- Assinatura HMAC-SHA256, retry com backoff, DLQ e logs estruturados.

### 3.2 Fora do escopo (com origem)

| ID | Exclusão | Origem |
|---|---|---|
| FDD-EXC-01 | Webhooks **inbound** (cliente → OMS) | [09:02] Marcos |
| FDD-EXC-02 | Notificação por e-mail de falhas repetidas | [09:37] Larissa |
| FDD-EXC-03 | Rate limiting de saída (apenas observar) | [09:39] Larissa |
| FDD-EXC-04 | Dashboard visual para o cliente (projeto do time de frontend) | [09:40] Larissa |
| FDD-EXC-05 | Arquivamento de eventos entregues (ex.: após 30 dias) | [09:08] Diego |
| FDD-EXC-06 | Múltiplos workers / particionamento por `order_id` | [09:13] Diego |
| FDD-EXC-07 | Evento na **criação** do pedido (`OrderService.create`): não é mudança de status e não passa por `changeStatus` | [09:40] Bruno |
| FDD-EXC-08 | Itens do pedido no payload | [09:43] Diego |
| FDD-EXC-09 | Restrição de role no CRUD de configuração e verificação de posse do customer | [09:37] Sofia |

---

## 4. Fluxos detalhados

### 4.1 Fluxo 1 — Criação do evento na outbox

**Gatilho:** `PATCH /api/v1/orders/:id/status` → `OrderController.changeStatus` → `OrderService.changeStatus`.

```
OrderService.changeStatus(id, input, userId)
└─ prisma.$transaction(async (tx) => {
     1. tx.order.findUnique({ id, include: items })            (existente)
     2. validações de transição                                  (existente)
     3. debitStock / replenishStock                              (existente)
     4. tx.order.update({ status: to })                          (existente)
     5. tx.orderStatusHistory.create({...})                      (existente)
     6. await publishWebhookEvent(tx, order, from, to)           ← NOVO
        6.1 webhooks = tx.webhook.findMany({ customerId: order.customerId, active: true })
        6.2 interessados = webhooks.filter(w => w.events.includes(to))
        6.3 se interessados vazio → return (nada é inserido)
        6.4 occurredAt = new Date()
        6.5 para cada w: eventId = uuidv4()
                         payload = renderPayload(eventId, occurredAt, order, from, to)
                         tx.webhookOutbox.create({ id: eventId, webhookId: w.id, orderId,
                                                   eventType, payload, status: PENDING,
                                                   attemptCount: 0, nextAttemptAt: occurredAt })
     7. tx.order.findUnique(... relações ...)                    (existente)
   })
```

Regras:

| ID | Regra | Origem |
|---|---|---|
| FDD-FLX-01 | Passo 6 roda **dentro** do callback de `$transaction`, com o mesmo `tx`. Qualquer exceção provoca rollback de 1–5. | [09:40] Bruno, [09:41] Diego |
| FDD-FLX-02 | O filtro considera só webhooks **ativos** do `customerId` do pedido cujo `events` contém `to` | [09:34] Bruno |
| FDD-FLX-03 | **Uma linha de outbox por webhook interessado**; o `id` da linha é o `event_id` (UUID v4, via pacote `uuid` já presente no `package.json`) | [09:25] Diego, [09:51] Larissa |
| FDD-FLX-04 | O payload é renderizado e gravado já na inserção (snapshot) | [09:52] Larissa |
| FDD-FLX-05 | `publishWebhookEvent` não faz I/O de rede nem captura exceções | [09:41] Diego |
| FDD-FLX-06 | O tamanho do payload **não** é validado aqui (validar aqui bloquearia a mudança de status); a checagem dos 64 KB é feita no worker (§4.2, passo 4) | [09:24] Larissa, [09:04] Bruno |

### 4.2 Fluxo 2 — Processamento pelo worker

**Processo:** `npm run worker` → `src/worker.ts` → `startWebhookWorker()` de `src/modules/webhooks/webhook.processor.ts`.

```
startup
 ├─ prisma = createPrismaClient()                                  (src/config/database.ts)
 ├─ recuperação: UPDATE webhook_outbox SET status='PENDING'
 │               WHERE status='PROCESSING'                          (órfãos de crash)
 └─ loop até receber SIGINT/SIGTERM:
     1. batch = SELECT * FROM webhook_outbox
                WHERE status='PENDING' AND next_attempt_at <= NOW()
                ORDER BY created_at ASC LIMIT WEBHOOK_BATCH_SIZE
     2. UPDATE webhook_outbox SET status='PROCESSING' WHERE id IN (batch)
     3. agrupa o batch por order_id; grupos em paralelo, itens do mesmo grupo em sequência
     4. para cada evento:
        a. carrega webhook (url, secret, previousSecret, previousSecretExpiresAt, active)
        b. se !active                         → DLQ(reason=WEBHOOK_INACTIVE), sem retry
        c. body = JSON.stringify(payload)
           se Buffer.byteLength(body) > 65536 → DLQ(reason=WEBHOOK_PAYLOAD_TOO_LARGE), sem retry
        d. se secret ausente                  → DLQ(reason=WEBHOOK_SECRET_REQUIRED), sem retry
        e. headers = buildHeaders(evento, webhook, body)            (§5.8)
        f. res = fetch(url, { method:'POST', body, headers, redirect:'manual',
                              signal: AbortSignal.timeout(10_000) })
        g. 2xx            → status=DELIVERED, delivered_at=NOW()
           não-2xx/erro   → Fluxo 3 (retry)
        h. grava attempt_count, last_attempt_at, last_response_status,
           last_response_body (até 2 048 caracteres), last_duration_ms, last_error
     5. se o batch veio cheio → próximo ciclo imediato; senão → aguarda 2 000 ms
```

| ID | Regra | Origem |
|---|---|---|
| FDD-FLX-07 | Intervalo de polling de **2 s**; busca os pendentes **mais antigos** em **batch pequeno** (`WEBHOOK_BATCH_SIZE`, valor inicial 10, ajustável) | [09:09] Diego, [09:08] Diego |
| FDD-FLX-08 | Ordem de leitura por `created_at ASC`; processamento sequencial dentro do mesmo `order_id` preserva a ordenação implícita por pedido | [09:12] Diego |
| FDD-FLX-09 | Instância única do worker; não há lock distribuído | [09:12] Diego |
| FDD-FLX-10 | Timeout HTTP de **10 s**; estourou = falha que vai para retry | [09:42] Diego |
| FDD-FLX-11 | Recuperação de `PROCESSING` órfão no startup. É seguro porque a entrega é at-least-once ([ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)) | [09:24] Diego |
| FDD-FLX-12 | Payload > 64 KB **não é enviado nem truncado**: vai direto para a DLQ com erro | [09:23] Sofia, [09:24] Larissa |
| FDD-FLX-13 | Redirects HTTP não são seguidos (`redirect: 'manual'`, 3xx = falha), para que um redirect não leve a entrega para fora do `https` cadastrado. Proposta sujeita à revisão de segurança. | [09:23] Sofia |

### 4.3 Fluxo 3 — Retry com backoff

Política do [ADR-002](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md): **1 envio inicial + 5 retentativas**.

```
on failure(evento, erro):
  evento.attemptCount += 1                     // conta envios realizados
  if evento.attemptCount <= 5:
      delay = WEBHOOK_RETRY_DELAYS_MS[evento.attemptCount - 1]
      evento.status = PENDING
      evento.nextAttemptAt = NOW() + delay
  else:
      Fluxo 4 (DLQ, reason = WEBHOOK_MAX_ATTEMPTS_EXCEEDED)
```

| `attemptCount` após a falha | Próxima tentativa em | Acumulado desde a 1ª falha |
|---|---|---|
| 1 | +1 min | 1 min |
| 2 | +5 min | 6 min |
| 3 | +30 min | 36 min |
| 4 | +2 h | 2 h 36 min |
| 5 | +12 h | 14 h 36 min |
| 6 | — (DLQ) | — |

Classificação de falha → `last_error`:

| Situação | `last_error` | Retentável? |
|---|---|---|
| Resposta 2xx | — | — (sucesso) |
| Resposta 3xx/4xx/5xx | `WEBHOOK_DELIVERY_HTTP_ERROR` | Sim |
| `AbortSignal.timeout` (10 s) | `WEBHOOK_DELIVERY_TIMEOUT` | Sim |
| DNS, conexão recusada, TLS | `WEBHOOK_DELIVERY_NETWORK_ERROR` | Sim |
| Webhook inativo / payload > 64 KB / secret ausente | ver §4.2 passo 4 | Não (DLQ imediata) |

> A reunião não distinguiu 4xx de 5xx; por isso **toda** resposta não-2xx é retentável. A diferenciação fica como evolução, a decidir.

### 4.4 Fluxo 4 — Dead Letter Queue

```
moveToDeadLetter(evento, reason) — numa única transação:
  1. webhook_outbox: status = FAILED (mantida para o histórico de entregas)
  2. INSERT webhook_dead_letter {
       id: uuid, outboxId: evento.id, webhookId, payload: evento.payload,
       reason, attemptCount, lastResponseStatus, lastError, failedAt: NOW()
     }
  3. log warn "webhook_moved_to_dlq"
```

| ID | Regra | Origem |
|---|---|---|
| FDD-FLX-14 | DLQ em tabela separada com payload, motivo e timestamp | [09:18] Diego |
| FDD-FLX-15 | Outbox marcada `FAILED` (um dos 4 status previstos: pendente, processando, falhou, entregue) | [09:08] Diego |

### 4.5 Fluxo 5 — Replay manual da DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` (ADMIN), numa única transação:

1. carrega a DLQ por `id`; inexistente → `WEBHOOK_DEAD_LETTER_NOT_FOUND`;
2. `replayedAt != null` → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`;
3. webhook inativo → `WEBHOOK_INACTIVE` (reativar via `PATCH` antes);
4. reseta a linha original da outbox: `status=PENDING`, `attemptCount=0`, `nextAttemptAt=NOW()`, `lastError=null`. **O `event_id` é mantido** ([ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md));
5. grava `replayedAt=NOW()` e `replayedById=req.user.id` na DLQ;
6. log info `webhook_dlq_replayed` com `{ deadLetterId, eventId, webhookId, userId }` (auditoria, [09:36] Sofia).

### 4.6 Fluxo 6 — Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret`:

1. `previousSecret = secret`; `previousSecretExpiresAt = NOW() + 24h`; `secret = novaSecret()` ([09:21] Sofia);
2. `novaSecret()` = `crypto.randomBytes(32).toString('hex')` (módulo nativo `node:crypto`; sujeito à revisão da Sofia, [09:46] Sofia);
3. durante o grace period o worker envia **duas** assinaturas (§5.8); depois de `previousSecretExpiresAt` envia só a nova;
4. uma nova rotação durante o grace period sobrescreve `previousSecret`: a secret mais antiga morre na hora. O comportamento é documentado na resposta e no portal.

---

## 5. Contratos públicos

Convenções herdadas do projeto: prefixo `/api/v1` (`src/app.ts`), `Authorization: Bearer <JWT>` (`src/middlewares/auth.middleware.ts`), corpo de erro `{ "error": { "code", "message", "details?" } }` (`src/middlewares/error.middleware.ts`), campos em camelCase na API. O payload **enviado ao cliente** usa snake_case, como definido na reunião ([09:43] Diego).

**Visibilidade da secret:** ela só aparece nas respostas de criação (§5.1) e rotação (§5.5). Listagens e edições nunca a devolvem, para limitar exposição ("se vaza uma, vaza tudo", [09:21] Sofia).

### 5.1 `POST /api/v1/webhooks` — cadastrar webhook (FDD-CONTRATO-01)

- **Auth:** qualquer role autenticada ([09:37] Sofia).
- **Body:**

| Campo | Tipo | Regra |
|---|---|---|
| `customerId` | uuid | obrigatório; customer precisa existir. Vem do body, **não** do JWT ([09:32] Larissa) |
| `url` | string | URL válida, **`https://` obrigatório** ([09:23] Sofia), máx. 2 048 chars |
| `events` | `OrderStatus[]` | ≥ 1 item, sem repetição, valores em `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` ([09:33] Marcos). `PENDING` é rejeitado porque nenhuma transição de `order.status.ts` leva a `PENDING` |

```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "6b1f2c1e-8a4d-4a63-9b1e-0f2d4c7a9e10",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**201 Created**
```json
{
  "id": "0c9b7d2a-3f41-4e8b-a2c6-5d1e9f8b7a64",
  "customerId": "6b1f2c1e-8a4d-4a63-9b1e-0f2d4c7a9e10",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "4f9a1c7e2b8d6a3f0e5c9b1d7a2e8f4c6b0d3a9e1f7c5b2d8a4e6f0c3b9d1a7e",
  "previousSecretExpiresAt": null,
  "createdAt": "2026-11-03T13:02:11.482Z",
  "updatedAt": "2026-11-03T13:02:11.482Z"
}
```

| Status | Quando |
|---|---|
| 201 | Criado |
| 400 | `VALIDATION_ERROR` (URL não-https ⇒ `details[].message` com `WEBHOOK_INVALID_URL`; filtro inválido ⇒ `WEBHOOK_INVALID_EVENT_FILTER`) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [{ "path": "url", "message": "WEBHOOK_INVALID_URL: webhook URL must use https" }]
  }
}
```

### 5.2 `GET /api/v1/webhooks?customerId=:customerId` — listar webhooks de um customer (FDD-CONTRATO-02)

- **Auth:** qualquer role autenticada. `customerId` obrigatório na query, no mesmo padrão de `listOrdersQuerySchema` ([09:33] Bruno).

**200 OK**
```json
{
  "data": [
    {
      "id": "0c9b7d2a-3f41-4e8b-a2c6-5d1e9f8b7a64",
      "customerId": "6b1f2c1e-8a4d-4a63-9b1e-0f2d4c7a9e10",
      "url": "https://hooks.atlascomercial.com.br/oms/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "previousSecretExpiresAt": null,
      "createdAt": "2026-11-03T13:02:11.482Z",
      "updatedAt": "2026-11-03T13:02:11.482Z"
    }
  ]
}
```

| Status | Quando |
|---|---|
| 200 | Lista (possivelmente vazia) |
| 400 | `VALIDATION_ERROR` (`customerId` ausente ou não-uuid) |
| 401 | `UNAUTHORIZED` |

### 5.3 `PATCH /api/v1/webhooks/:id` — editar webhook (FDD-CONTRATO-03)

- **Body:** qualquer subconjunto de `url`, `events`, `active` (≥ 1 campo), com as mesmas regras de §5.1. `active: false` pausa o webhook sem apagar o histórico.
- Mudança de `events` vale só para eventos **futuros** ([ADR-007](adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md)).

```http
PATCH /api/v1/webhooks/0c9b7d2a-3f41-4e8b-a2c6-5d1e9f8b7a64
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"], "active": true }
```

**200 OK**: objeto webhook atualizado (mesmo formato de §5.2, sem `secret`).

| Status | Quando |
|---|---|
| 200 | Atualizado |
| 400 | `VALIDATION_ERROR` (corpo vazio, URL não-https, filtro inválido) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 5.4 `DELETE /api/v1/webhooks/:id` — remover webhook (FDD-CONTRATO-04)

Remove a configuração. Linhas de `webhook_outbox` e `webhook_dead_letter` desse webhook são removidas em cascata: eventos pendentes não teriam para onde ir e a secret deixa de existir. Para pausar sem perder histórico, use `PATCH { "active": false }`.

```http
DELETE /api/v1/webhooks/0c9b7d2a-3f41-4e8b-a2c6-5d1e9f8b7a64
Authorization: Bearer eyJhbGciOi...
```

**204 No Content** (sem corpo).

| Status | Quando |
|---|---|
| 204 | Removido |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 5.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret (FDD-CONTRATO-05)

Corpo vazio. Gera nova secret; a anterior segue aceita por 24 h ([09:21] Sofia).

**200 OK**
```json
{
  "id": "0c9b7d2a-3f41-4e8b-a2c6-5d1e9f8b7a64",
  "secret": "b7e3d9a1f5c2e8b4d0a6f3c9e1b7d5a2f8c4e0b6d2a9f7c3e5b1d8a4f6c0e2b9",
  "previousSecretExpiresAt": "2026-11-04T15:20:00.000Z"
}
```

| Status | Quando |
|---|---|
| 200 | Rotacionada |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 5.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas (FDD-CONTRATO-06)

Retorna os **últimos 100** eventos do webhook (ordem `createdAt DESC`), com sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos). Fonte: `webhook_outbox` (+ `webhook_dead_letter.id` quando houver).

**200 OK**
```json
{
  "data": [
    {
      "eventId": "a3d5f7e9-1b2c-4d6e-8f0a-9c7b5d3e1f20",
      "eventType": "order.status_changed",
      "orderId": "d2c4e6f8-0a1b-4c3d-9e5f-7a8b6c4d2e01",
      "status": "DELIVERED",
      "attemptCount": 1,
      "lastResponseStatus": 200,
      "lastResponseBody": "{\"received\":true}",
      "lastDurationMs": 184,
      "lastError": null,
      "deadLetterId": null,
      "payload": { "event_id": "a3d5f7e9-1b2c-4d6e-8f0a-9c7b5d3e1f20", "event_type": "order.status_changed", "...": "..." },
      "createdAt": "2026-11-03T14:10:02.031Z",
      "lastAttemptAt": "2026-11-03T14:10:03.512Z",
      "deliveredAt": "2026-11-03T14:10:03.696Z"
    },
    {
      "eventId": "f1e2d3c4-b5a6-4978-8a9b-0c1d2e3f4a5b",
      "eventType": "order.status_changed",
      "orderId": "9e8d7c6b-5a4f-4e3d-8c2b-1a0f9e8d7c6b",
      "status": "FAILED",
      "attemptCount": 6,
      "lastResponseStatus": 503,
      "lastResponseBody": "Service Unavailable",
      "lastDurationMs": 92,
      "lastError": "WEBHOOK_DELIVERY_HTTP_ERROR",
      "deadLetterId": "7c5a3e1f-9d8b-4a6c-b2e0-4f6a8c0e2d41",
      "payload": { "...": "..." },
      "createdAt": "2026-11-02T08:00:00.000Z",
      "lastAttemptAt": "2026-11-02T22:36:01.110Z",
      "deliveredAt": null
    }
  ]
}
```

| Status | Quando |
|---|---|
| 200 | Histórico (possivelmente vazio) |
| 401 | `UNAUTHORIZED` |
| 404 | `WEBHOOK_NOT_FOUND` |

### 5.7 Payload do evento `order.status_changed`

Definido na reunião ([09:43] Diego): enxuto, sem `items`.

```json
{
  "event_id": "a3d5f7e9-1b2c-4d6e-8f0a-9c7b5d3e1f20",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-03T14:10:01.987Z",
  "order_id": "d2c4e6f8-0a1b-4c3d-9e5f-7a8b6c4d2e01",
  "order_number": "ORD-000123",
  "customer_id": "6b1f2c1e-8a4d-4a63-9b1e-0f2d4c7a9e10",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "subtotal_cents": 18000,
  "discount_cents": 1000,
  "total_cents": 17000
}
```

| Campo | Origem no código |
|---|---|
| `event_id` | `webhook_outbox.id` (UUID) |
| `timestamp` | momento da mudança de status (`occurredAt`, §4.1), ISO 8601 |
| `order_id`, `order_number`, `customer_id` | `Order.id`, `Order.orderNumber` (`ORD-000123`, gerado por `reserveOrderNumber`), `Order.customerId` |
| `from_status`, `to_status` | `from` / `to` em `changeStatus` (enum `OrderStatus`) |
| `*_cents` | `Order.subtotalCents`, `Order.discountCents`, `Order.totalCents` |

### 5.8 Headers e assinatura do envio ao cliente

`POST <webhook.url>` ([09:44] Diego, [09:44] Sofia):

| Header | Valor | Exemplo |
|---|---|---|
| `Content-Type` | `application/json` | `application/json` |
| `X-Event-Id` | `event_id` (UUID, estável entre retentativas e replay) | `a3d5f7e9-1b2c-4d6e-8f0a-9c7b5d3e1f20` |
| `X-Webhook-Id` | `webhooks.id` do endpoint | `0c9b7d2a-3f41-4e8b-a2c6-5d1e9f8b7a64` |
| `X-Timestamp` | instante **deste envio**, ISO 8601 | `2026-11-03T14:10:03.512Z` |
| `X-Signature` | `sha256=` + hex(HMAC-SHA256(secret, corpo bruto)) | `sha256=5d41402abc4b2a76b9719d911017c592...` |

```ts
// webhook.signature.ts (esboço)
import { createHmac } from 'node:crypto';

export function sign(secret: string, rawBody: string): string {
  return 'sha256=' + createHmac('sha256', secret).update(rawBody, 'utf8').digest('hex');
}

export function buildSignatureHeader(w: Webhook, rawBody: string, now: Date): string {
  const sigs = [sign(w.secret, rawBody)];
  if (w.previousSecret && w.previousSecretExpiresAt && w.previousSecretExpiresAt > now) {
    sigs.push(sign(w.previousSecret, rawBody)); // grace period de 24 h
  }
  return sigs.join(',');
}
```

- A assinatura cobre **exatamente** os bytes enviados no corpo: serializar uma vez (`JSON.stringify`) e usar a mesma string para assinar e enviar.
- **Verificação no cliente** (para o portal, [09:26] Marcos): calcular `sha256=<hmac>` sobre o corpo bruto e aceitar se bater com **qualquer** item do header, usando comparação em tempo constante. Para detectar replay, usar o campo `timestamp` do corpo, que é assinado. `X-Timestamp` é informativo e não é assinado (ver [RFC Q-07](RFC.md#7-questões-em-aberto)).
- **Resposta esperada do cliente:** qualquer 2xx dentro de 10 s. O corpo da resposta é guardado (até 2 048 caracteres) só para o histórico de entregas.

### 5.9 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ (FDD-CONTRATO-07)

- **Auth:** `authenticate` + `requireRole('ADMIN')` ([09:36] Sofia, [09:36] Larissa).
- **Body:** vazio. O `:id` é o `deadLetterId` exposto em `GET /webhooks/:id/deliveries`.
- **Efeito:** Fluxo 5 (§4.5). O evento volta à outbox como `PENDING`, com o **mesmo** `event_id`; a entrega é assíncrona, por isso a resposta é `202`.

```http
POST /api/v1/admin/webhooks/dead-letter/7c5a3e1f-9d8b-4a6c-b2e0-4f6a8c0e2d41/replay
Authorization: Bearer eyJhbGciOi...   (role ADMIN)
```

**202 Accepted**
```json
{
  "deadLetterId": "7c5a3e1f-9d8b-4a6c-b2e0-4f6a8c0e2d41",
  "eventId": "f1e2d3c4-b5a6-4978-8a9b-0c1d2e3f4a5b",
  "webhookId": "0c9b7d2a-3f41-4e8b-a2c6-5d1e9f8b7a64",
  "status": "PENDING",
  "replayedAt": "2026-11-03T09:15:42.000Z",
  "replayedById": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d"
}
```

**403 Forbidden** (usuário OPERATOR)
```json
{ "error": { "code": "FORBIDDEN", "message": "Insufficient permissions" } }
```

| Status | Quando |
|---|---|
| 202 | Evento recolocado na outbox |
| 401 | `UNAUTHORIZED` |
| 403 | `FORBIDDEN` (role ≠ ADMIN) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` ou `WEBHOOK_INACTIVE` |

---

## 6. Matriz de erros

Todas as classes ficam em `src/modules/webhooks/webhook.errors.ts` e estendem `AppError` (`src/shared/errors/app-error.ts`). O middleware `src/middlewares/error.middleware.ts` serializa todas sem alteração ([09:29] Bruno).

### 6.1 Erros da API (HTTP)

| Código | HTTP | Classe | Endpoint(s) | Quando |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError extends AppError` | 5.3, 5.4, 5.5, 5.6 | `:id` inexistente |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError extends AppError` | 5.1 | `customerId` não existe em `customers` |
| `WEBHOOK_INVALID_URL` | 400 | regra Zod (`refine`) | 5.1, 5.3 | URL não-`https` ou inválida. Sai como `VALIDATION_ERROR` com o código em `details[].message`, porque `validate.middleware.ts` padroniza todo `ZodError` ([09:23] Sofia: "validação no schema Zod") |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | regra Zod (`refine`) | 5.1, 5.3 | `events` vazio, com repetição ou contendo `PENDING`. Mesmo formato do item acima |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError extends AppError` | replay | DLQ `:id` inexistente |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `WebhookDeadLetterAlreadyReplayedError extends ConflictError` | replay | `replayedAt` já preenchido |
| `WEBHOOK_INACTIVE` | 409 | `WebhookInactiveError extends ConflictError` | replay | Webhook da DLQ está `active=false` |
| `UNAUTHORIZED` (existente) | 401 | `UnauthorizedError` | todos | JWT ausente ou inválido (`authenticate`) |
| `FORBIDDEN` (existente) | 403 | `ForbiddenError` | replay | Role ≠ `ADMIN` (`requireRole('ADMIN')`) |

> `WebhookNotFoundError` estende `AppError` diretamente porque `NotFoundError` (`src/shared/errors/http-errors.ts`) fixa o código `NOT_FOUND` no construtor.
> Uma falha do `INSERT` na outbox dentro de `changeStatus` é erro Prisma não mapeado: o middleware responde `500 INTERNAL_SERVER_ERROR` e a transação faz rollback, como desejado.

### 6.2 Erros do worker (gravados em `last_error` / `webhook_dead_letter.reason`)

| Código | Retentável | Destino | Quando |
|---|---|---|---|
| `WEBHOOK_DELIVERY_TIMEOUT` | Sim | retry | Sem resposta em 10 s ([09:42] Diego) |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Sim | retry | Status fora de 2xx (inclui 3xx, ver FDD-FLX-13) |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | Sim | retry | DNS, conexão, TLS |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | DLQ | 6ª falha (1 + 5 retentativas) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Não | DLQ imediata | Corpo > 65 536 bytes ([09:24] Larissa) |
| `WEBHOOK_INACTIVE` | Não | DLQ imediata | Webhook desativado depois da inserção |
| `WEBHOOK_SECRET_REQUIRED` | Não | DLQ imediata | Registro sem secret; o worker nunca envia sem assinatura |

> `WEBHOOK_SECRET_REQUIRED` foi citado como exemplo na reunião ([09:28] Bruno). Como a secret é gerada pelo servidor ([09:31] Marcos), o cliente nunca a envia e esse código não se aplica à API pública; ele fica reservado para a salvaguarda interna do worker.

---

## 7. Estratégias de resiliência

| ID | Estratégia | Valor | Origem |
|---|---|---|---|
| FDD-RES-01 | Timeout por requisição | 10 000 ms (`AbortSignal.timeout`) | [09:42] Diego |
| FDD-RES-02 | Retentativas | 5, após o envio inicial | [09:17] Larissa |
| FDD-RES-03 | Backoff | 1m, 5m, 30m, 2h, 12h (tabela fixa, sem jitter) | [09:17] Diego |
| FDD-RES-04 | Fallback final | DLQ + replay manual ADMIN; **sem** e-mail nesta fase | [09:18] Diego, [09:37] Larissa |
| FDD-RES-05 | Polling | 2 000 ms entre ciclos; ciclo imediato se o batch veio cheio | [09:09] Diego |
| FDD-RES-06 | Recuperação de crash | `PROCESSING` → `PENDING` no startup | [09:24] Diego |
| FDD-RES-07 | Shutdown gracioso | Em `SIGINT`/`SIGTERM`: para de buscar, aguarda as requisições em voo (até 10 s), `prisma.$disconnect()`, igual a `src/server.ts` | [09:11] Larissa |
| FDD-RES-08 | Isolamento API × worker | Processos separados; a falha de um não derruba o outro | [09:11] Diego |
| FDD-RES-09 | Limite de payload | 65 536 bytes, erro sem truncar | [09:24] Diego |

### 7.4 Paralelismo controlado dentro do batch

Com processamento estritamente sequencial, um batch de 10 eventos para clientes que estouram o timeout levaria até 100 s, o que fere a meta de < 10 s para **todos** os clientes. Por isso:

- o batch é agrupado por `order_id`; grupos diferentes rodam em paralelo (`Promise.allSettled`), limitados ao tamanho do batch;
- eventos do **mesmo** `order_id` rodam em sequência, preservando a ordenação implícita por pedido decidida em [09:12] Diego.

Limitação conhecida: um evento em backoff não bloqueia eventos posteriores do mesmo pedido, então a ordem pode inverter sob retry ([RFC Q-05](RFC.md#7-questões-em-aberto)).

---

## 8. Observabilidade

O projeto já tem logger Pino estruturado (`src/shared/logger/index.ts`) e um log por request com `requestId` (`src/middlewares/request-logger.middleware.ts`). Não existe stack de métricas nem de tracing, e a decisão é não adicionar ferramentas novas ([09:29] Bruno). A observabilidade se apoia em **logs estruturados + consultas SQL sobre a outbox**.

### 8.1 Logs

O worker usa `logger.child({ component: 'webhook-worker' })`. Nomes de evento em snake_case, como os existentes (`server_started`, `http_request`):

| Evento (msg) | Nível | Campos | Emitido por |
|---|---|---|---|
| `webhook_event_enqueued` | debug | `eventId`, `webhookId`, `orderId`, `toStatus` | API, após o commit de `changeStatus` |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `batchSize`, `signal` | worker |
| `webhook_worker_recovered_processing` | warn | `count` | worker (startup) |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs`, `lagMs` | worker |
| `webhook_delivery_failed` | warn | `eventId`, `webhookId`, `attempt`, `errorCode`, `statusCode?`, `durationMs`, `nextAttemptAt` | worker |
| `webhook_moved_to_dlq` | warn | `eventId`, `webhookId`, `deadLetterId`, `reason`, `attempts` | worker |
| `webhook_dlq_replayed` | info | `deadLetterId`, `eventId`, `webhookId`, `userId` (**auditoria**) | API ([09:36] Sofia) |
| `webhook_secret_rotated` | info | `webhookId`, `userId`, `previousSecretExpiresAt` | API |

**Redaction:** incluir `'*.secret'` e `'*.previousSecret'` em `redactPaths` de `src/shared/logger/index.ts`. Já houve vazamento de secret em log de cliente ([09:22] Diego). Nunca logar headers `X-Signature` nem corpo de resposta completo.

### 8.2 Métricas

São derivadas dos logs acima (agregação na plataforma de logs) e de consultas SQL periódicas:

| Métrica | Como obter | Uso / alerta sugerido |
|---|---|---|
| `webhook_delivery_lag_ms` (p50/p95) | campo `lagMs` = `deliveredAt − createdAt` em `webhook_delivery_succeeded` | Meta de < 10 s ([09:02] Marcos) |
| `webhook_outbox_oldest_pending_age_s` | `SELECT TIMESTAMPDIFF(SECOND, MIN(created_at), NOW()) FROM webhook_outbox WHERE status='PENDING' AND attempt_count=0` | Worker parado ou travado |
| `webhook_outbox_pending_total` | `SELECT COUNT(*) … WHERE status='PENDING'` | Acúmulo |
| `webhook_delivery_success_rate` | `succeeded / (succeeded + failed)` por `webhookId` | Cliente com problema |
| `webhook_dlq_total` | `SELECT COUNT(*) FROM webhook_dead_letter WHERE replayed_at IS NULL` | Fila de replay manual |
| `webhook_delivery_duration_ms` (p95) | `durationMs` | Clientes lentos |
| `webhook_sends_per_minute` por webhook | contagem de `webhook_delivery_*` | Insumo para a questão de rate limiting ([09:39] Larissa) |

### 8.3 Tracing (correlação)

Sem infraestrutura de tracing distribuído no projeto, o rastreamento ponta a ponta usa **IDs de correlação** presentes em todos os logs e persistidos:

```
PATCH /orders/:id/status  ─ requestId (X-Request-Id) + orderId
        └─ webhook_event_enqueued ─ eventId, webhookId, orderId
              └─ webhook_delivery_* ─ eventId, attempt
                    └─ cliente recebe X-Event-Id = eventId, X-Webhook-Id = webhookId
                          └─ webhook_moved_to_dlq / webhook_dlq_replayed ─ eventId, deadLetterId
```

O `eventId` é a chave de trace: o suporte começa pelo `X-Event-Id` informado pelo cliente e encontra todas as tentativas, a DLQ e o replay. A instrumentação com OpenTelemetry não foi discutida e ficaria contra o [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) nesta fase.

---

## 9. Modelo de dados e migração

Novos models em `prisma/schema.prisma` (PKs UUID `Char(36)`, como no restante do schema):

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model Webhook {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // OrderStatus[]; MySQL não tem array (mesmo recurso de Customer.address)
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id], onDelete: Cascade)
  outbox      WebhookOutbox[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {
  id                 String              @id @db.Char(36) // = event_id
  webhookId          String              @db.Char(36)
  orderId            String              @db.Char(36)     // sem FK para orders (ver nota)
  eventType          String              @db.VarChar(64)
  payload            Json
  status             WebhookOutboxStatus @default(PENDING)
  attemptCount       Int                 @default(0)
  nextAttemptAt      DateTime            @default(now())
  lastAttemptAt      DateTime?
  lastResponseStatus Int?
  lastResponseBody   String?             @db.Text         // até 2 048 caracteres
  lastDurationMs     Int?
  lastError          String?             @db.VarChar(64)
  deliveredAt        DateTime?
  createdAt          DateTime            @default(now())

  webhook    Webhook            @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deadLetter WebhookDeadLetter?

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([webhookId, createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id                 String    @id @default(uuid()) @db.Char(36)
  outboxId           String    @unique @db.Char(36)
  webhookId          String    @db.Char(36)
  payload            Json
  reason             String    @db.VarChar(64)
  attemptCount       Int
  lastResponseStatus Int?
  lastError          String?   @db.Text
  failedAt           DateTime  @default(now())
  replayedAt         DateTime?
  replayedById       String?   @db.Char(36)

  outbox  WebhookOutbox @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook Webhook       @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, failedAt])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:

- Os índices `status` e `created_at` foram pedidos na reunião ([09:08] Diego). `next_attempt_at` entra no índice composto com `status` porque o worker filtra por ambos depois da introdução do backoff.
- **`orderId` sem FK:** `OrderService.delete` apaga pedidos em `PENDING`/`CANCELLED`. Uma FK `Restrict` faria esse delete falhar com erro Prisma `P2003`, que o error middleware não mapeia (500), para pedidos cancelados que geraram eventos. Uma FK `Cascade` apagaria evidência de entrega. A coluna fica indexada e sem FK.
- **`Webhook → Customer` com `onDelete: Cascade`:** a configuração de webhook não deve virar um novo bloqueio para a exclusão de customer.
- `Customer` ganha a relação inversa `webhooks Webhook[]`.
- Migration: `npm run db:migrate -- --name add_webhooks` (script `db:migrate` já existe em `package.json`).

## 10. Dependências e compatibilidade

| Item | Situação |
|---|---|
| Node.js ≥ 20 (`engines` no `package.json`) | `fetch` e `AbortSignal.timeout` nativos; `node:crypto` para HMAC e geração de secret. **Nenhuma dependência nova.** |
| `@prisma/client` / `prisma` 5.22.0, MySQL 8.0 (`docker-compose.yml`) | Novos models + migration; sem mudança de versão |
| `uuid` 11.0.3 (já dependência) | Geração do `event_id` (o `id` da outbox não usa `@default(uuid())` porque o valor é necessário antes, dentro do payload) |
| `zod` 3.23.8, `pino` 9.5.0, `express` 4.21.1 | Reuso direto |
| Contratos existentes | **Sem breaking change**: `PATCH /orders/:id/status` mantém request e response; `OrderService` mantém o construtor |
| Deploy | Novo processo `worker`, com as mesmas variáveis de ambiente da API (`DATABASE_URL`, `LOG_LEVEL`, `NODE_ENV`, `JWT_SECRET`, este último exigido por `src/config/env.ts`). A ordem de deploy é migration → API → worker |
| Revisão de segurança | 2 dias úteis da Sofia antes do deploy ([09:46] Sofia) |
| Portal de desenvolvedor | Marcos documenta HMAC, deduplicação e headers ([09:26] Marcos, [09:40] Marcos) |

## 11. Integração com o sistema existente

| # | Arquivo | Tipo de mudança | Como o módulo se integra |
|---|---|---|---|
| INT-01 | `src/modules/orders/order.service.ts` | **Alteração (1 chamada)** | Em `changeStatus`, depois de `tx.orderStatusHistory.create(...)` e antes do `tx.order.findUnique` final, adicionar `await publishWebhookEvent(tx, order, from, to)`. O `tx` já é `Prisma.TransactionClient` (alias `TxClient` no arquivo), no mesmo padrão de `debitStock(tx, …)`. O construtor e `create()` não mudam ([ADR-008](adrs/ADR-008-publicacao-via-funcao-transacional-no-changestatus.md)). |
| INT-02 | `src/modules/orders/order.status.ts` | Somente leitura | O enum e a tabela `transitions` definem os `to_status` possíveis. O schema de `events` rejeita `PENDING`, porque nenhuma transição chega nele. Não há alteração. |
| INT-03 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Reuso por herança | `webhook.errors.ts` estende `AppError` (404 com códigos próprios) e `ConflictError` (409). Não altera os arquivos existentes. |
| INT-04 | `src/middlewares/error.middleware.ts` | Sem alteração | Serializa os novos `AppError` como `{ error: { code, message, details } }` automaticamente ([09:29] Bruno). |
| INT-05 | `src/middlewares/auth.middleware.ts` | Reuso | `router.use(authenticate)` no router de webhooks; `requireRole('ADMIN')` no replay, como em `src/modules/users/user.routes.ts`; `req.user.id` alimenta `replayedById` e o log de auditoria. |
| INT-06 | `src/middlewares/validate.middleware.ts` | Reuso | `validate({ body, params, query })` com os schemas de `webhook.schemas.ts`; erros Zod saem como `VALIDATION_ERROR`. |
| INT-07 | `src/routes/index.ts` | Alteração | Adicionar `webhooks` e `adminWebhooks` ao tipo `Controllers` e `router.use('/webhooks', buildWebhookRouter(...))`, `router.use('/admin/webhooks', buildAdminWebhookRouter(...))`. |
| INT-08 | `src/app.ts` | Alteração | Em `buildControllers`: `new WebhookRepository(prisma)` → `new WebhookService(repo, prisma)` → `new WebhookController(service)`. `OrderService` continua com `(orderRepository, prisma)`. |
| INT-09 | `src/server.ts` → **novo** `src/worker.ts` | Novo arquivo (modelo) | `src/worker.ts` replica o bootstrap de `server.ts`: logger, `SIGINT`/`SIGTERM` com `prisma.$disconnect()`, `bootstrap().catch(... 'bootstrap_failed')`. Chama `startWebhookWorker({ prisma, logger })`. |
| INT-10 | `src/config/database.ts` e `src/config/env.ts` | Reuso | O worker importa `prisma` (instância própria, porque é outro processo, [09:30] Bruno) e herda a validação de ambiente de `env.ts`. |
| INT-11 | `src/shared/logger/index.ts` | Alteração mínima | Adicionar `'*.secret'` e `'*.previousSecret'` a `redactPaths`; o worker usa `logger.child({ component: 'webhook-worker' })`. |
| INT-12 | `prisma/schema.prisma` | Alteração | Models da §9 + relação `webhooks Webhook[]` em `Customer`. |
| INT-13 | `package.json` | Alteração | Scripts `"worker": "tsx watch --env-file=.env src/worker.ts"` e `"worker:start": "node --env-file=.env dist/worker.js"`, no padrão de `dev`/`start`. |
| INT-14 | `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/orders.test.ts` | Alteração | `beforeEach` limpa `webhookDeadLetter` → `webhookOutbox` → `webhook` antes de `customer` (FKs); nova factory `createTestWebhook`; casos novos em `orders.test.ts` para a outbox. |

### 11.1 Esboço de `publishWebhookEvent`

```ts
// src/modules/webhooks/webhook.publisher.ts
import type { Order, OrderStatus, Prisma } from '@prisma/client';
import { v4 as uuidv4 } from 'uuid';
import { WEBHOOK_EVENT_ORDER_STATUS_CHANGED } from './webhook.constants.js';

export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  const webhooks = await tx.webhook.findMany({
    where: { customerId: order.customerId, active: true },
    select: { id: true, events: true },
  });
  const targets = webhooks.filter((w) => (w.events as OrderStatus[]).includes(toStatus));
  if (targets.length === 0) return;

  const occurredAt = new Date();
  for (const w of targets) {
    const eventId = uuidv4();
    await tx.webhookOutbox.create({
      data: {
        id: eventId,
        webhookId: w.id,
        orderId: order.id,
        eventType: WEBHOOK_EVENT_ORDER_STATUS_CHANGED,
        nextAttemptAt: occurredAt,
        payload: {
          event_id: eventId,
          event_type: WEBHOOK_EVENT_ORDER_STATUS_CHANGED,
          timestamp: occurredAt.toISOString(),
          order_id: order.id,
          order_number: order.orderNumber,
          customer_id: order.customerId,
          from_status: fromStatus,
          to_status: toStatus,
          subtotal_cents: order.subtotalCents,
          discount_cents: order.discountCents,
          total_cents: order.totalCents,
        },
      },
    });
  }
}
```

> O `order` recebido é o objeto lido no início de `changeStatus`. Os campos usados (`id`, `orderNumber`, `customerId`, `*Cents`) não mudam na transição, então o snapshot é fiel ao estado no momento da mudança.

### 11.2 Estrutura do módulo

> **Arquivos novos (ainda não existem no repositório):** `src/worker.ts` e tudo em `src/modules/webhooks/`. Os nomes `src/worker.ts`, `src/modules/webhooks/` e `webhook.processor.ts` vêm da reunião ([09:11] Larissa, [09:28] Bruno). Os demais seguem a convenção `<domínio>.<papel>.ts` dos módulos existentes. Todos os outros caminhos citados neste pacote existem hoje no repositório.

```
src/
├── worker.ts                         (novo entry-point)
└── modules/webhooks/
    ├── webhook.constants.ts          intervalos, timeout, backoff, limites
    ├── webhook.controller.ts         CRUD, rotate-secret, deliveries, replay
    ├── webhook.errors.ts             classes WEBHOOK_*
    ├── webhook.processor.ts          loop do worker, retry, DLQ
    ├── webhook.publisher.ts          publishWebhookEvent(tx, …)
    ├── webhook.repository.ts         acesso a webhooks / outbox / DLQ
    ├── webhook.routes.ts             buildWebhookRouter, buildAdminWebhookRouter
    ├── webhook.schemas.ts            Zod
    ├── webhook.service.ts            regras de CRUD, rotação, replay
    └── webhook.signature.ts          HMAC-SHA256
```

```ts
// webhook.constants.ts
export const WEBHOOK_EVENT_ORDER_STATUS_CHANGED = 'order.status_changed';
export const WEBHOOK_POLL_INTERVAL_MS = 2_000;
export const WEBHOOK_BATCH_SIZE = 10;
export const WEBHOOK_HTTP_TIMEOUT_MS = 10_000;
export const WEBHOOK_RETRY_DELAYS_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000] as const;
export const WEBHOOK_MAX_PAYLOAD_BYTES = 65_536;
export const WEBHOOK_SECRET_GRACE_PERIOD_MS = 86_400_000;
export const WEBHOOK_DELIVERIES_LIMIT = 100;
export const WEBHOOK_RESPONSE_BODY_MAX_CHARS = 2_048;
```

```ts
// webhook.schemas.ts (trecho)
const subscribableStatus = z
  .nativeEnum(OrderStatus)
  .refine((s) => s !== OrderStatus.PENDING, 'WEBHOOK_INVALID_EVENT_FILTER: PENDING is never a target status');

const httpsUrl = z
  .string()
  .url('WEBHOOK_INVALID_URL: invalid URL')
  .max(2048)
  .refine((u) => u.startsWith('https://'), 'WEBHOOK_INVALID_URL: webhook URL must use https');

const events = z
  .array(subscribableStatus)
  .min(1, 'WEBHOOK_INVALID_EVENT_FILTER: at least one status is required')
  .refine((a) => new Set(a).size === a.length, 'WEBHOOK_INVALID_EVENT_FILTER: duplicated status');

export const createWebhookSchema = z.object({ customerId: z.string().uuid(), url: httpsUrl, events });
export const updateWebhookSchema = z
  .object({ url: httpsUrl.optional(), events: events.optional(), active: z.boolean().optional() })
  .refine((v) => Object.keys(v).length > 0, 'At least one field must be provided');
export const listWebhooksQuerySchema = z.object({ customerId: z.string().uuid() });
export const webhookIdParamSchema = z.object({ id: z.string().uuid() });
```

## 12. Estratégia de testes

Mesmo stack atual: Vitest + Supertest contra MySQL real (`vitest.config.ts`, `tests/setup.ts`). Para o worker, um receptor HTTP local (`node:http`) faz o papel do cliente. Sem dependências novas.

| ID | Teste | Tipo |
|---|---|---|
| FDD-TST-01 | `changeStatus` com webhook interessado insere 1 linha `PENDING` com payload correto | Integração |
| FDD-TST-02 | Status fora do filtro / webhook inativo / customer sem webhook ⇒ nenhuma linha | Integração |
| FDD-TST-03 | Falha forçada no `INSERT` da outbox ⇒ status e histórico inalterados (rollback) | Integração |
| FDD-TST-04 | Worker entrega, `X-Signature` confere com a secret, headers presentes, `DELIVERED` | Integração (receptor local) |
| FDD-TST-05 | Receptor retorna 500 ⇒ `attemptCount`/`nextAttemptAt` seguem a tabela de §4.3 (relógio controlado) | Unitário |
| FDD-TST-06 | 6ª falha ⇒ `FAILED` + linha na DLQ com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | Integração |
| FDD-TST-07 | Receptor demora > 10 s ⇒ `WEBHOOK_DELIVERY_TIMEOUT` | Integração |
| FDD-TST-08 | Payload > 64 KB ⇒ DLQ imediata, nenhuma requisição enviada | Unitário |
| FDD-TST-09 | Replay: 403 para OPERATOR; 202 para ADMIN; segundo replay ⇒ 409; mesmo `event_id` | Integração |
| FDD-TST-10 | `POST /webhooks` com `http://` ⇒ 400 com `WEBHOOK_INVALID_URL` nos details | Integração |
| FDD-TST-11 | Rotação: durante o grace period o header tem 2 assinaturas; após a expiração, 1 | Unitário |
| FDD-TST-12 | `PROCESSING` órfão volta a `PENDING` no startup | Unitário |
| FDD-TST-13 | `GET /deliveries` limita a 100 e ordena por `createdAt DESC` | Integração |

## 13. Critérios de aceite técnicos

- [ ] CA-01: Os testes FDD-TST-01 a 13 passam em `npm test`.
- [ ] CA-02: `PATCH /orders/:id/status` mantém contrato e semântica; os testes existentes de `tests/orders.test.ts` continuam verdes.
- [ ] CA-03: Com o worker rodando e o receptor respondendo em < 1 s, p95 de `lagMs` < 10 s em teste de carga local com 100 mudanças de status.
- [ ] CA-04: `kill -TERM` no worker durante um envio não perde o evento (ele é reenviado no próximo start).
- [ ] CA-05: Nenhuma secret aparece em logs (verificado com `LOG_LEVEL=debug`).
- [ ] CA-06: `package.json` sem novas dependências de runtime.
- [ ] CA-07: Todos os erros do módulo usam códigos com prefixo `WEBHOOK_` (exceto os genéricos herdados `UNAUTHORIZED`, `FORBIDDEN` e `VALIDATION_ERROR`).
- [ ] CA-08: Revisão de segurança da Sofia concluída e aprovada antes do deploy.

## 14. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| FDD-RSK-01 | Clientes lentos saturam o worker único | Média | Alto | Paralelismo por `order_id` (§7.4); timeout de 10 s; alerta em `webhook_outbox_oldest_pending_age_s` |
| FDD-RSK-02 | Transação de `changeStatus` mais lenta (consulta de webhooks + inserts) | Baixa | Médio | Índice `(customer_id, active)`; um `findMany` por transação; payload enxuto |
| FDD-RSK-03 | Inversão de ordem sob retry | Média | Baixo | `from_status`/`to_status`/`timestamp` no payload; documentação; [RFC Q-05](RFC.md#7-questões-em-aberto) |
| FDD-RSK-04 | Secret exposta em log ou resposta | Baixa | Alto | Redaction no Pino; secret só nas respostas de criação e rotação; revisão de segurança |
| FDD-RSK-05 | Crescimento da `webhook_outbox` | Alta | Baixo | Índices; arquivamento planejado fora do escopo ([09:08] Diego) |
| FDD-RSK-06 | `DELETE` de webhook apaga o histórico e a DLQ | Média | Médio | Documentar `PATCH active=false` como forma de pausar |
| FDD-RSK-07 | Worker parado sem alguém perceber | Baixa | Alto | Alerta na idade do pendente mais antigo; eventos preservados na outbox |
