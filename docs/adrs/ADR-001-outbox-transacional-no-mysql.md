# ADR-001 — Padrão Outbox transacional no MySQL existente

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:06–09:08
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-005](ADR-005-worker-separado-com-polling.md), [ADR-008](ADR-008-publicacao-via-funcao-transacional-no-changestatus.md), [RFC](../RFC.md), [FDD §4.1](../FDD.md#41-fluxo-1--criação-do-evento-na-outbox)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) precisam ser notificados quando o status de um pedido muda. O ponto de origem do evento é o método `OrderService.changeStatus` em `src/modules/orders/order.service.ts`, que já executa uma transação Prisma (`this.prisma.$transaction`) relativamente pesada: valida a transição (`canTransition`, em `src/modules/orders/order.status.ts`), debita ou repõe estoque (`debitStock` / `replenishStock`), atualiza `orders` e insere em `order_status_history`.

Forças em jogo:

- Fazer a chamada HTTP dentro dessa transação acopla a latência de um cliente externo à mudança de status de **todos** os pedidos ([09:04] Bruno).
- Se o cliente estiver fora do ar, não existe resposta aceitável: não dá para fazer rollback da mudança de status ([09:04] Bruno).
- O evento **não pode** ser perdido quando o status muda, nem emitido quando a transação faz rollback ([09:06] Diego, [09:40] Bruno).
- O time é pequeno e não quer operar infraestrutura nova ([09:07] Diego).
- O banco já é MySQL 8 (`docker-compose.yml`, `prisma/schema.prisma`).

## Decisão

Adotar o **padrão Outbox** sobre o MySQL existente:

1. Dentro da **mesma transação** de `changeStatus`, inserir uma linha por webhook interessado na tabela `webhook_outbox`, com o payload do evento já renderizado.
2. Se a inserção na outbox falhar, a transação inteira faz rollback (o status não muda).
3. Um worker separado ([ADR-005](ADR-005-worker-separado-com-polling.md)) lê a outbox e faz as chamadas HTTP fora da transação de negócio.
4. A outbox tem índice por `status` (pendente, processando, entregue, falhou) e por `created_at` ([09:08] Diego).
5. O arquivamento de linhas entregues (ideia de "depois de 30 dias") **fica fora do escopo** desta feature ([09:08] Diego).

## Alternativas consideradas

### A1. Disparo HTTP síncrono dentro de `changeStatus`
- **Prós:** implementação mais simples; latência mínima.
- **Contras:** cliente lento trava mudanças de status de outros pedidos; cliente fora do ar não tem tratamento correto (rollback de status é inaceitável).
- **Motivo do descarte:** "Síncrono não rola" ([09:04] Bruno); "Síncrono está fora de questão" ([09:06] Diego).

### A2. Fila externa (Redis Streams / Redis Cluster)
- **Prós:** fila reativa, sem polling; escala horizontal mais natural.
- **Contras:** mais infraestrutura para subir e operar; perde a atomicidade com a transação do MySQL (seria preciso outro mecanismo para garantir "commit ⇒ evento publicado").
- **Motivo do descarte:** "a gente acabaria precisando subir mais infra" ([09:07] Larissa); "Subir Redis Cluster pra isso é overengineering" ([09:07] Diego).

## Consequências

### Positivas
- Consistência forte entre mudança de status e registro do evento: commit ⇒ evento registrado; rollback ⇒ evento descartado.
- Nenhuma infraestrutura nova: mesma instância MySQL, mesmo Prisma, mesma `DATABASE_URL`.
- A outbox vira também a fonte do histórico de entregas (`GET /webhooks/:id/deliveries`).

### Negativas
- A transação de `changeStatus` fica um pouco maior (consulta de webhooks interessados + `INSERT` na outbox).
- A tabela cresce sem limite até existir rotina de arquivamento, que está fora do escopo.
- O banco transacional passa a ser também o "broker": carga do worker concorre com a carga da API.

### Trade-off explícito
Trocamos **latência mínima e reatividade** (fila dedicada ou chamada síncrona) por **atomicidade e zero infraestrutura nova**. A latência adicional é limitada pelo intervalo de polling (2 s), aceito como compatível com o requisito de "< 10 segundos" ([09:02] Marcos, [09:10] Larissa).
