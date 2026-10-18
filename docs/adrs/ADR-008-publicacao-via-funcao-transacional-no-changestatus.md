# ADR-008 — Publicação do evento via função `publishWebhookEvent(tx, …)` chamada dentro de `OrderService.changeStatus`

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:40–09:41
- **Decisores:** Bruno (Pedidos), Diego (Plataforma), Larissa (Tech Lead)
- **Relacionados:** [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md), [ADR-007](ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md), [FDD §11](../FDD.md#11-integração-com-o-sistema-existente)

## Contexto

A integração crítica com o código atual é em `src/modules/orders/order.service.ts`, no método `changeStatus` ([09:40] Bruno). Hoje:

- `OrderService` recebe por construtor `OrderRepository` e `PrismaClient` (montado em `src/app.ts`, `buildControllers`).
- `changeStatus` abre `this.prisma.$transaction(async (tx) => { ... })` e, com o `tx` (`Prisma.TransactionClient`, já tipado como `TxClient` no arquivo), faz `tx.order.update` e `tx.orderStatusHistory.create`, além de `debitStock` ou `replenishStock` conforme as regras de `src/modules/orders/order.status.ts`.
- O padrão de helpers privados que recebem `tx` já existe no próprio service: `debitStock(tx, ...)`, `replenishStock(tx, ...)`, `reserveOrderNumber(tx)`.

Para cumprir o [ADR-001](ADR-001-outbox-transacional-no-mysql.md), a inserção na outbox precisa usar **o mesmo `tx`**. Se a outbox falhar, a transação inteira deve fazer rollback: "Não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).

## Decisão

1. Criar em `src/modules/webhooks/` uma função exportada **`publishWebhookEvent(tx, order, fromStatus, toStatus)`**, que recebe o client da transação corrente ([09:41] Bruno).
2. `changeStatus` chama essa função **depois** de `tx.orderStatusHistory.create` e **antes** do `findUnique` final, ainda dentro do callback de `$transaction`.
3. A função, usando apenas o `tx` recebido:
   1. busca os webhooks **ativos** do `order.customerId` cujo filtro contém `toStatus` ([ADR-007](ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md));
   2. para cada um, gera o UUID do evento, renderiza o snapshot do payload e insere em `webhook_outbox` com status `PENDING`;
   3. não faz I/O de rede e não captura exceções: qualquer erro sobe e provoca rollback.
4. **Não** injetar um `WebhookRepository` inteiro no `OrderService`. A assinatura do construtor de `OrderService` e a montagem em `src/app.ts` permanecem inalteradas ([09:41] Diego).

## Alternativas consideradas

### A1. Injetar um repository/service de webhooks no `OrderService`
- **Prós:** segue a injeção por construtor usada no resto do projeto.
- **Contras:** acopla o `OrderService` a um objeto maior do que precisa; muda a montagem em `src/app.ts` e os testes; o repository injetado teria de aceitar `tx` de qualquer forma para participar da transação.
- **Motivo do descarte:** "função pura recebendo o tx. Não precisa injetar repository inteiro" ([09:41] Diego).

### A2. Publicar o evento depois do commit (fora da transação)
- **Prós:** a transação de `changeStatus` não cresce.
- **Contras:** se o processo cair entre o commit e a publicação, o status muda e o evento se perde.
- **Motivo do descarte:** "Se ficar fora da transação, perde a garantia toda" ([09:41] Diego).

## Consequências

### Positivas
- Mudança mínima e localizada em `order.service.ts` (uma chamada).
- Atomicidade garantida pelo próprio `$transaction` do Prisma.
- Função fácil de testar isoladamente, recebendo um `tx` de teste.

### Negativas
- `order.service.ts` passa a importar de `src/modules/webhooks/`, uma dependência direta entre módulos.
- Falha de banco na outbox (ex.: tabela indisponível) passa a **bloquear** mudanças de status. É um efeito desejado pela decisão, mas aumenta a superfície de falha de `changeStatus`.
- A criação de pedido (`OrderService.create`, status inicial `PENDING` com `fromStatus: null`) **não** publica evento: a reunião delimitou a integração ao `changeStatus` ([09:40] Bruno).

### Trade-off explícito
Aceitamos um **acoplamento direto pequeno** (uma função importada) e o risco de a outbox bloquear mudanças de status, em troca de **atomicidade estrita** e de **não alterar** a composição de dependências existente.
