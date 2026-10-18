# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Pedro (@pedrohsantos), redação a partir da reunião técnica conduzida por Larissa |
| **Status** | Em revisão |
| **Data** | 2026-10-18 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

> Esta RFC consolida a proposta discutida na reunião técnica e a submete à sessão de revisão combinada com Bruno e Diego antes do início da implementação ([09:50] Larissa). O detalhamento de contratos, schemas e fluxos está no [FDD](FDD.md).

---

## 1. TL;DR

Propomos notificar clientes B2B via **webhooks outbound** sempre que o status de um pedido muda. O evento é gravado numa **tabela outbox no MySQL, na mesma transação** que muda o status. Um **worker em processo separado** lê a outbox **a cada 2 s** e faz um `POST` assinado com **HMAC-SHA256** (uma secret por endpoint) para a URL `https` do cliente. Falhas têm **5 retentativas com backoff** (1m/5m/30m/2h/12h) e, depois disso, o evento vai para uma **DLQ** com replay manual por ADMIN. A entrega é **at-least-once**, com `X-Event-Id` para deduplicação no cliente. Tudo reaproveita a stack atual: Prisma/MySQL, Pino, `AppError` e o padrão de módulos. Nenhuma infraestrutura nova.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para ser notificados quando o status dos seus pedidos muda. Hoje elas fazem polling periódico em `GET /orders`, o que torna a integração "lenta e cara pra eles". A Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos). Para os clientes, qualquer coisa abaixo de 10 s é "tempo real" ([09:02] Marcos). O fluxo é só de saída, da plataforma para o cliente ([09:02] Marcos).

O OMS não tem nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), numa transação Prisma que valida a transição (`src/modules/orders/order.status.ts`), ajusta estoque, atualiza `orders` e grava `order_status_history`. Essa transação já é pesada ([09:04] Bruno). Qualquer solução precisa:

- não acoplar a mudança de status à disponibilidade ou latência de terceiros;
- não perder eventos de status que foram commitados, nem emitir eventos de transações que fizeram rollback;
- caber num time pequeno, sem infraestrutura nova ([09:07] Diego);
- ficar pronta em ~3 sprints, incluindo a revisão de segurança ([09:46] Larissa).

## 3. Proposta técnica

### 3.1 Visão geral

```
 PATCH /orders/:id/status
          │
          ▼
 ┌───────────────── transação MySQL (changeStatus) ─────────────────┐
 │ update orders → insert order_status_history → ajusta estoque      │
 │ publishWebhookEvent(tx, order, from, to)                          │
 │   └─ para cada webhook ativo do customer que ouve `to`:           │
 │        insert webhook_outbox (payload snapshot, PENDING)          │
 └───────────────────────────────────────────────────────────────────┘
          │ commit
          ▼
 ┌──────── worker (processo separado: npm run worker) ────────┐
 │ a cada 2 s: lê PENDING vencidos (ordem de created_at)       │
 │ assina HMAC-SHA256 → POST https://cliente (timeout 10 s)    │
 │   2xx → DELIVERED                                          │
 │   falha → reagenda (1m/5m/30m/2h/12h)                      │
 │   esgotou → FAILED + webhook_dead_letter                   │
 └────────────────────────────────────────────────────────────┘
          │
          ▼
 POST /admin/webhooks/dead-letter/:id/replay  (ADMIN) → volta a PENDING
```

### 3.2 Componentes

| Componente | Responsabilidade | Decisão |
|---|---|---|
| **Outbox (`webhook_outbox`)** | Registro atômico do evento junto com a mudança de status | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| **Publicação em `changeStatus`** | Função `publishWebhookEvent(tx, …)` chamada dentro da transação existente | [ADR-008](adrs/ADR-008-publicacao-via-funcao-transacional-no-changestatus.md) |
| **Filtro + payload** | Só insere para webhooks interessados no `to_status`; payload enxuto e congelado na inserção | [ADR-007](adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md) |
| **Worker** | Processo `src/worker.ts`, polling de 2 s, instância única | [ADR-005](adrs/ADR-005-worker-separado-com-polling.md) |
| **Retry + DLQ** | 5 retentativas com backoff; DLQ em tabela própria; replay manual ADMIN | [ADR-002](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md) |
| **Segurança do envio** | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24 h de grace, `https` obrigatório | [ADR-003](adrs/ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| **Semântica de entrega** | At-least-once, `X-Event-Id` para dedup | [ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) |
| **API de configuração** | CRUD de webhooks, rotação de secret, histórico de entregas, replay de DLQ | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |

### 3.3 Superfície pública (resumo)

- **API (JWT do OMS):** cadastrar, listar, editar e remover webhooks; rotacionar secret; consultar os últimos 100 envios; replay de DLQ (somente ADMIN). O `customerId` vai na requisição e **não** é derivado do JWT, porque o JWT representa o usuário operador, não o cliente ([09:32] Bruno, [09:32] Larissa).
- **Envio ao cliente:** `POST` JSON com o evento `order.status_changed` e os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` ([09:44] Diego, [09:44] Sofia).

Os contratos completos (rotas, payloads, códigos de erro `WEBHOOK_*`) estão em [FDD §5–6](FDD.md#5-contratos-públicos).

## 4. Alternativas consideradas

| # | Alternativa | Por que foi descartada (trade-off) | Fonte |
|---|---|---|---|
| ALT-01 | **Disparo HTTP síncrono dentro de `changeStatus`** | Mais simples e com latência mínima, mas um cliente lento trava a mudança de status de outros pedidos e um cliente fora do ar não tem resposta correta (não dá para fazer rollback do status). | [09:04] Bruno, [09:06] Diego |
| ALT-02 | **Fila em Redis Streams / Redis Cluster** | Seria reativa e escalaria melhor, mas exige subir e operar infraestrutura nova e perde a atomicidade com a transação MySQL. "Overengineering" para um time pequeno. | [09:07] Larissa, [09:07] Diego |
| ALT-03 | **Trigger de banco para acordar o worker** | Seria mais reativo, mas o MySQL não tem `LISTEN/NOTIFY` e trigger não notifica processo externo; exigiria gambiarras (arquivo ou endpoint). Polling de 2 s já atende a meta. | [09:09] Bruno, [09:09] Diego |
| ALT-04 | **Worker dentro do processo da API** | Um deploy a menos, mas restart da API derrubaria o consumo de eventos. | [09:11] Diego |
| ALT-05 | **Retry indefinido** / **apenas 3 tentativas** | Indefinido deixa eventos pendurados para sempre; 3 tentativas matam o evento numa manutenção planejada de 2 h. | [09:15] Diego, [09:16] Diego |
| ALT-06 | **Exactly-once** | Tiraria a deduplicação do cliente, mas exige coordenação dos dois lados e muito mais complexidade. | [09:25] Diego |
| ALT-07 | **Secret global da plataforma** | Mais simples de operar, mas um vazamento compromete todos os clientes. | [09:21] Sofia |

## 5. Impacto

| Área | Impacto |
|---|---|
| **Código existente** | Uma chamada nova em `OrderService.changeStatus`; novo módulo `src/modules/webhooks/`; novos models no `prisma/schema.prisma`; registro de rotas em `src/routes/index.ts`; novo entry-point `src/worker.ts` e script `npm run worker`. Nenhuma mudança nos middlewares de erro, autenticação ou validação. |
| **Banco** | 3 tabelas novas (`webhooks`, `webhook_outbox`, `webhook_dead_letter`); a transação de mudança de status fica um pouco maior. |
| **Operação** | Um processo novo para deployar e monitorar (o worker). |
| **Clientes** | Precisam expor endpoint `https`, validar `X-Signature` e deduplicar por `X-Event-Id`, conforme documentado no portal de desenvolvedor ([09:26] Marcos). |
| **Prazo** | ~3 sprints, com 2 dias úteis de revisão de segurança antes do deploy ([09:46] Larissa, [09:46] Sofia). A Atlas espera a entrega até o fim de novembro ([09:45] Marcos). |

## 6. Riscos

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Cliente lento (até 10 s de timeout) atrasa a fila do worker único | Média | Alto (fere a meta de < 10 s para todos) | Paralelismo controlado dentro do batch, mantendo sequência por pedido (FDD §7.4); métrica de idade do evento pendente mais antigo. |
| Worker parado por ponto único de falha | Baixa | Alto | Eventos não se perdem (ficam na outbox); shutdown gracioso; alerta pela idade do pendente mais antigo. |
| Outbox indisponível bloqueia mudanças de status | Baixa | Alto | Comportamento desejado pela garantia de atomicidade ([09:40] Bruno); mesma disponibilidade do MySQL que já sustenta `orders`. |
| Vazamento de secret pelo cliente | Média | Médio | Secret por endpoint e rotação com grace de 24 h ([09:21] Sofia); redaction da secret nos logs. |
| Cliente não deduplica e processa o evento duas vezes | Média | Médio | Documentação destacada no portal ([09:26] Marcos). |
| Crescimento contínuo da outbox | Alta | Baixo, no curto prazo | Índices por `status` e `created_at`; arquivamento planejado fora desta feature ([09:08] Diego). |

## 7. Questões em aberto

| # | Questão | Situação na reunião | Proposta para a revisão |
|---|---|---|---|
| Q-01 | **Rate limiting de saída por cliente** (ex.: 50 mudanças de status em 1 minuto geram 50 chamadas). | Fora do escopo; "observar e decidir depois". | Monitorar envios por webhook/minuto nos logs do worker e reabrir se virar problema. ([09:38] Diego, [09:39] Larissa) |
| Q-02 | **Escalar para múltiplos workers** sem perder a ordenação por pedido. | Adiado ("problema do futuro"); opções citadas: particionar por `order_id` ou lock pessimista. | Manter instância única nesta fase. ([09:13] Diego) |
| Q-03 | **Notificar o cliente por e-mail** quando o webhook falha repetidamente. | Fora desta fase; avaliar depois de medir o impacto. | Fase futura. ([09:37] Larissa) |
| Q-04 | **Endurecer a autorização do CRUD de webhooks** (hoje qualquer role autenticada, sem verificação de posse do customer). | "Por enquanto sim. Mais pra frente a gente pode endurecer." | Fase futura. ([09:37] Sofia) |
| Q-05 | **Ordenação sob retry.** A ordenação implícita por pedido ([09:12] Diego) vale para o primeiro envio; se um evento entra em backoff, um evento posterior do mesmo pedido pode chegar antes. | Não discutido explicitamente; a reunião registrou só a limitação de single-worker ([09:13] Larissa). | Documentar no portal que o cliente deve reconciliar por `from_status`/`to_status`/`timestamp`, ou bloquear eventos posteriores do mesmo pedido enquanto houver retry pendente. Decidir na revisão. |
| Q-06 | **"5 tentativas" = 5 retentativas após o envio inicial?** Os 5 intervalos (1m…12h, "quase 15 horas") só fecham com 1 envio inicial + 5 retentativas. | Ambíguo ([09:15] Diego, [09:17] Diego, [09:48] Larissa). | Adotar 1 + 5 (ver [ADR-002](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md)); confirmar. |
| Q-07 | **Detalhes de segurança para a revisão da Sofia:** armazenamento da secret (precisa ser recuperável para o HMAC), formato do `X-Signature` com duas assinaturas durante o grace period e o fato de `X-Timestamp` não ser assinado. | A Sofia reservou 2 dias úteis para revisar HMAC e geração de secret. | Levar a proposta do FDD §5.8 para a revisão de segurança. ([09:46] Sofia) |

**Explicitamente fora de escopo** (não são questões em aberto): dashboard visual para o cliente, que é projeto do time de frontend ([09:40] Larissa); webhooks inbound ([09:02] Marcos); arquivamento de eventos entregues ([09:08] Diego).

## 8. Decisões relacionadas

- [ADR-001 — Padrão Outbox transacional no MySQL](adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-003 — HMAC-SHA256 com secret por endpoint](adrs/ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-004 — At-least-once com `X-Event-Id`](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker separado com polling](adrs/ADR-005-worker-separado-com-polling.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Payload snapshot enxuto e filtro na inserção](adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md)
- [ADR-008 — Publicação via função transacional em `changeStatus`](adrs/ADR-008-publicacao-via-funcao-transacional-no-changestatus.md)
