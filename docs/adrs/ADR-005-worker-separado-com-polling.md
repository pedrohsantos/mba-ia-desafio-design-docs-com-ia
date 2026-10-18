# ADR-005 — Worker em processo separado, com polling a cada 2 segundos e instância única

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:08–09:14
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md), [FDD §4.2](../FDD.md#42-fluxo-2--processamento-pelo-worker)

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), falta decidir **como** e **onde** os eventos são consumidos. O requisito de produto é entregar em menos de 10 segundos, que para os clientes já conta como "tempo real" ([09:02] Marcos). Hoje a aplicação tem um único entry-point, `src/server.ts`, que sobe a API Express com o `prisma` exportado por `src/config/database.ts` e faz shutdown gracioso em `SIGINT` e `SIGTERM`.

## Decisão

1. **Polling em loop a cada 2 segundos:** o worker busca os eventos pendentes mais antigos em batch pequeno, processa e marca o resultado ([09:09] Diego, [09:08] Diego). Com isso, o polling acrescenta no máximo ~2 s de latência a cada evento, o que foi aceito ([09:10] Larissa, [09:10] Marcos).
2. **Processo separado da API**, com novo entry-point `src/worker.ts` e script `npm run worker` ([09:11] Diego, [09:11] Larissa). Se a API reinicia, o worker não cai junto.
3. **Mesmo banco e mesma stack**, mas com **PrismaClient próprio**, porque o PrismaClient é por processo ([09:11] Bruno, [09:30] Bruno). O worker usa a mesma `DATABASE_URL` validada por `src/config/env.ts` e a fábrica `createPrismaClient()` de `src/config/database.ts`.
4. A lógica de processamento fica dentro do módulo: `src/modules/webhooks/webhook.processor.ts`. O `src/worker.ts` só faz bootstrap e shutdown ([09:28] Bruno).
5. **Instância única (single-worker):** os eventos são processados em ordem de `created_at`, o que dá ordenação **implícita por `order_id`**. Não há garantia de ordenação global. Isso fica documentado como **limitação conhecida** ([09:12] Diego, [09:13] Larissa).

## Alternativas consideradas

### A1. Trigger no banco / mecanismo reativo
- **Prós:** reatividade, sem latência de polling.
- **Contras:** o MySQL não tem `LISTEN/NOTIFY` como o Postgres; trigger só executa SQL e não notifica processo externo. Avisar o worker exigiria gambiarras, como escrever em arquivo ou chamar um endpoint.
- **Motivo do descarte:** "fica esquisito"; polling de 2 s já atende o requisito ([09:09] Diego).

### A2. Worker dentro do processo da API
- **Prós:** um único deploy e um processo a menos.
- **Contras:** restart ou crash da API derruba o consumo de eventos; carga do worker compete com requests HTTP no mesmo event loop.
- **Motivo do descarte:** "Senão se a API reinicia, perde o worker" ([09:11] Diego).

### A3. Múltiplos workers em paralelo
- **Prós:** mais vazão.
- **Contras:** perde a ordenação implícita por pedido; exigiria particionar por `order_id` ou usar lock pessimista.
- **Motivo do descarte:** adiado como "problema do futuro" ([09:13] Diego).

## Consequências

### Positivas
- Latência de ponta a ponta dentro da meta de < 10 s em operação normal.
- Isolamento de falhas entre API e worker.
- Nenhuma tecnologia nova; reaproveita `env`, Prisma e Pino ([ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).

### Negativas
- Queries periódicas no MySQL mesmo sem eventos (custo baixo, mas constante).
- **Ponto único de falha:** com o worker parado, nenhum evento é entregue. Eles não se perdem (ficam na outbox), mas se acumulam.
- **Ordenação limitada:** a garantia por `order_id` vale para o primeiro envio. Quando um evento entra em backoff ([ADR-002](ADR-002-retry-com-backoff-exponencial-e-dlq.md)), um evento posterior do mesmo pedido pode ser entregue antes. Registrado como questão em aberto no [RFC](../RFC.md#7-questões-em-aberto).
- Um cliente lento (até 10 s de timeout por requisição) pode atrasar outros clientes se o processamento for estritamente sequencial; o FDD mitiga isso com paralelismo controlado dentro do batch, mantendo a sequência por pedido.

### Trade-off explícito
Escolhemos **simplicidade operacional e ordenação implícita** (um processo, polling) em vez de **reatividade e escala horizontal**. Aceitamos até 2 s de latência de polling e um ponto único de falha, que pode ser reavaliado quando a escala exigir.
