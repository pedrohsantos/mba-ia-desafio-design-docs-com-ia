# ADR-007 — Payload enxuto renderizado como snapshot na inserção, com filtro de eventos na inserção

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:33–09:34, 09:43–09:44 e 09:51–09:52
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md), [FDD §5.7](../FDD.md#57-payload-do-evento-orderstatus_changed)

## Contexto

Três perguntas de modelagem do evento ficaram em aberto até o fim da reunião:

1. **Quando** filtrar os eventos que cada webhook quer receber: na inserção na outbox ou no envio ([09:34] Diego)?
2. **O que** vai no payload: o pedido completo, com itens, ou só um resumo ([09:43] Marcos)?
3. A outbox guarda o payload **já renderizado** ou só o `order_id`, renderizando na hora do envio ([09:51] Bruno)?

O model `Order` (`prisma/schema.prisma`) tem `orderNumber`, `customerId`, `status`, `subtotalCents`, `discountCents`, `totalCents` e as relações `items`, `history` e `customer`.

## Decisão

1. **Filtro na inserção:** cada webhook tem uma lista de status que quer ouvir (ex.: só `SHIPPED` e `DELIVERED`). Em `changeStatus`, só são inseridas linhas na outbox para webhooks **ativos** do customer cujo filtro contém o `to_status`. Se nenhum webhook se interessa, nada é inserido ([09:33] Marcos, [09:34] Bruno, [09:34] Diego).
2. **Payload enxuto:** JSON com `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e os campos básicos da order (`subtotal_cents`, `discount_cents`, `total_cents`). **Não inclui `items`**: quem quiser detalhes consulta `GET /api/v1/orders/:id` ([09:43] Diego, [09:44] Bruno).
3. **Snapshot na inserção:** o payload é renderizado e gravado na outbox dentro da transação de `changeStatus`. O evento reflete o estado do pedido **no momento da mudança**, mesmo que o pedido mude depois ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
4. **Limite de 64 KB:** um payload maior que 64 KB não é enviado nem truncado; é tratado como erro ([09:23] Sofia, [09:24] Diego, [09:24] Larissa). Como o payload é enxuto, o limite funciona como salvaguarda, não como restrição do dia a dia.

## Alternativas consideradas

### A1. Filtrar no momento do envio
- **Prós:** mudar o filtro afeta também eventos já enfileirados.
- **Contras:** gera linhas inúteis na outbox para eventos que nenhum webhook quer.
- **Motivo do descarte:** "Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela" ([09:34] Bruno).

### A2. Renderizar o payload na hora do envio (guardar só `order_id`)
- **Prós:** outbox menor; payload sempre "atual".
- **Contras:** um evento `PAID` enviado com atraso (retry de horas) mostraria dados de um estado posterior, criando "caso esquisito" ([09:52] Larissa); também exige leitura extra no worker.
- **Motivo do descarte:** snapshot na inserção, por consenso ([09:52] Diego, [09:52] Bruno).

### A3. Incluir `items` no payload
- **Prós:** o cliente não precisa de segunda chamada.
- **Contras:** infla o payload e aproxima do limite de 64 KB.
- **Motivo do descarte:** "Não manda items pra não inflar" ([09:43] Diego).

### A4. Truncar payloads acima do limite
- **Prós:** sempre entrega algo.
- **Contras:** entrega dado incompleto e potencialmente inválido; esconde um bug.
- **Motivo do descarte:** "Eu sou a favor de erra. Se chegou nesse tamanho, tem algo errado" ([09:23] Sofia).

## Consequências

### Positivas
- Outbox só contém trabalho útil.
- O evento é imutável, auditável e coerente com o `from_status`/`to_status` que o originou.
- Payload pequeno: baixa latência de transmissão e longe do limite.

### Negativas
- Alterar o filtro de eventos (`PATCH`) não afeta eventos já inseridos.
- Cada evento grava uma cópia do resumo do pedido (mais armazenamento por linha).
- Clientes que precisam de itens fazem uma segunda chamada `GET /orders/:id`, sujeita ao JWT da nossa API.
- A verificação dos 64 KB precisa acontecer **fora** da transação de `changeStatus`, senão um payload anômalo bloquearia a mudança de status. O FDD coloca a checagem no worker, antes do envio, com envio direto para a DLQ.

### Trade-off explícito
Trocamos **flexibilidade** (filtro e conteúdo decididos no envio) por **eficiência e fidelidade histórica** (outbox enxuta e evento imutável).
