# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**.
Cada decisão arquitetural fica em um arquivo próprio, nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`. O formato é uma variante do MADR com as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências.

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-transacional-no-mysql.md) | Padrão Outbox transacional no MySQL existente | Aceito |
| [ADR-002](ADR-002-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | Aceito |
| [ADR-003](ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 h | Aceito |
| [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` | Aceito |
| [ADR-005](ADR-005-worker-separado-com-polling.md) | Worker em processo separado, polling de 2 s, instância única | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes (módulos, `AppError`, Pino, middlewares) | Aceito |
| [ADR-007](ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md) | Payload enxuto em snapshot na inserção, com filtro de eventos na inserção | Aceito |
| [ADR-008](ADR-008-publicacao-via-funcao-transacional-no-changestatus.md) | Publicação via `publishWebhookEvent(tx, …)` dentro de `changeStatus` | Aceito |
