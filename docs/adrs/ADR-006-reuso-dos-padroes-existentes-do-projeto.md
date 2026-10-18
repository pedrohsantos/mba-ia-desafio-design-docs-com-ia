# ADR-006 — Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:27–09:31 e 09:36
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança)
- **Relacionados:** [ADR-005](ADR-005-worker-separado-com-polling.md), [ADR-008](ADR-008-publicacao-via-funcao-transacional-no-changestatus.md), [FDD §11](../FDD.md#11-integração-com-o-sistema-existente)

## Contexto

O OMS já tem convenções claras e consistentes entre os módulos. A pergunta era se o módulo de webhooks seguiria essas convenções ou se teria estrutura, bibliotecas e tratamento de erro próprios. As convenções verificadas no código:

| Padrão | Onde está no código |
|---|---|
| Módulo por domínio com `controller`, `service`, `repository`, `routes`, `schemas` | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/` |
| Roteadores montados sob `/api/v1` | `src/routes/index.ts` (`buildApiRouter`), `src/app.ts` (`buildControllers`, `buildApp`) |
| Erro base com `statusCode` + `errorCode` + `details` | `src/shared/errors/app-error.ts` (`AppError`) |
| Erros específicos com código em `UPPER_SNAKE_CASE` | `src/shared/errors/http-errors.ts` (`InsufficientStockError` → `INSUFFICIENT_STOCK`, `InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`) |
| Tratamento centralizado de `AppError`, `ZodError` e erros Prisma | `src/middlewares/error.middleware.ts` |
| Validação com Zod por `body`/`query`/`params` | `src/middlewares/validate.middleware.ts`, `src/modules/*/*.schemas.ts` |
| Autenticação JWT e autorização por role | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| Logger estruturado Pino com redaction | `src/shared/logger/index.ts` |
| PKs UUID `Char(36)` | `prisma/schema.prisma` |

## Decisão

O módulo de webhooks é **um módulo como os outros**, com reuso máximo do que já existe ([09:30] Larissa):

1. **Estrutura:** `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, além de `webhook.processor.ts`, executado pelo worker ([09:27] Bruno, [09:28] Bruno).
2. **Erros:** novas classes estendendo `AppError`, com códigos prefixados por **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`) ([09:28] Bruno, [09:29] Larissa). Detalhe verificado no código: `NotFoundError` fixa o código `NOT_FOUND` no construtor, por isso `WebhookNotFoundError` precisa estender `AppError` diretamente para emitir `WEBHOOK_NOT_FOUND`.
3. **Logger:** Pino existente (`logger` de `src/shared/logger/index.ts`). Nenhuma biblioteca nova ([09:29] Bruno).
4. **Error middleware:** sem alteração. Ele já trata `AppError`, Zod e Prisma e vai capturar os novos erros automaticamente ([09:29] Bruno).
5. **Validação:** schemas Zod no padrão de `order.schemas.ts`, aplicados via `validate()`.
6. **Autorização:** CRUD de configuração com `authenticate`, aceitando qualquer role autenticada. O replay de DLQ usa `requireRole('ADMIN')`, que já existe e é usado em `src/modules/users/user.routes.ts` ([09:36] Larissa, [09:37] Sofia).
7. **IDs:** UUID, como no resto do projeto ([09:51] Larissa).
8. **Banco:** novos models no mesmo `prisma/schema.prisma`, com migration Prisma padrão.

## Alternativas consideradas

### A1. Estrutura e stack próprias para o subsistema de webhooks
Por exemplo, um serviço à parte com seu próprio formato de erro ou a adoção de uma biblioteca de jobs/filas.
- **Prós:** liberdade para otimizar o subsistema isoladamente.
- **Contras:** dois padrões para o time manter; o middleware de erro não reconheceria um formato diferente; contradiz "Não vamos botar nada novo" ([09:29] Bruno).
- **Motivo do descarte:** decisão explícita por reuso máximo ([09:30] Larissa).

### A2. Reutilizar os erros genéricos existentes sem prefixo (`NOT_FOUND`, `CONFLICT`)
- **Prós:** zero classes novas.
- **Contras:** o cliente da API não distingue "webhook não encontrado" de "pedido não encontrado"; foge do padrão de códigos de domínio já usado em `INSUFFICIENT_STOCK`.
- **Motivo do descarte:** a reunião decidiu prefixo `WEBHOOK_` para tudo do módulo ([09:29] Larissa).

## Consequências

### Positivas
- Curva de aprendizado mínima; revisão de código mais simples.
- Contrato de erro (`{ error: { code, message, details } }`) idêntico ao do resto da API.
- Nenhuma dependência nova para auditar na revisão de segurança.

### Negativas
- Herda as limitações atuais: o JWT representa o **usuário operador**, não o cliente, então não há verificação de posse do `customerId` ([09:32] Bruno). Qualquer usuário autenticado pode gerenciar webhooks de qualquer customer até o endurecimento previsto para o futuro ([09:37] Sofia).
- O `validate.middleware.ts` converte qualquer `ZodError` para `VALIDATION_ERROR`. Regras validadas no schema (como `https`) saem com código de topo `VALIDATION_ERROR`, e o código `WEBHOOK_*` vai em `details[].message` (ver matriz de erros no FDD). Não mexemos no middleware para preservar o reuso sem alteração.
- Sem stack de métricas/tracing dedicada: a observabilidade fica limitada a logs estruturados e consultas SQL (ver FDD §8).

### Trade-off explícito
Priorizamos **consistência e velocidade de entrega** (3 sprints, [09:46] Larissa) sobre **otimizações específicas do subsistema**. Herdamos as limitações atuais de autorização e observabilidade em vez de resolvê-las nesta feature.
