# ADR-004 — Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:24–09:26
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-002](ADR-002-retry-com-backoff-exponencial-e-dlq.md), [ADR-007](ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md)

## Contexto

Com outbox + worker + retry ([ADR-001](ADR-001-outbox-transacional-no-mysql.md), [ADR-002](ADR-002-retry-com-backoff-exponencial-e-dlq.md)), há cenários em que o cliente recebe o mesmo evento mais de uma vez. Por exemplo: o cliente processa o request mas a resposta não chega antes do timeout de 10 s; ou o worker cai depois do envio e antes de marcar o evento como entregue. É preciso escolher uma semântica de entrega e dar ao cliente uma forma de lidar com duplicatas.

## Decisão

1. A plataforma garante **at-least-once**: todo evento registrado é entregue pelo menos uma vez (dentro da política de retry) e **pode** ser entregue mais de uma vez ([09:24] Diego).
2. Cada evento recebe um **UUID gerado quando entra na outbox**, único por evento, enviado no header **`X-Event-Id`** e também no campo `event_id` do payload ([09:25] Diego, [09:43] Diego).
3. O cliente é responsável por **deduplicar pelo `event_id`** do lado dele ([09:25] Diego).
4. O UUID segue o padrão de IDs do projeto: todas as PKs em `prisma/schema.prisma` são `String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa). Na prática, o `id` da linha da outbox **é** o `event_id`.
5. O replay manual de DLQ reaproveita a mesma linha da outbox e, portanto, **o mesmo `event_id`**: para o cliente é o mesmo evento, e a deduplicação continua funcionando.
6. O Marcos documenta essa responsabilidade em destaque no portal de desenvolvedor ([09:26] Marcos).

## Alternativas consideradas

### A1. Exactly-once
- **Prós:** o cliente não precisa deduplicar.
- **Contras:** exige coordenação entre os dois lados (confirmação transacional distribuída), o que é muito mais complexo.
- **Motivo do descarte:** "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo" ([09:25] Diego). Stripe e GitHub adotam o mesmo modelo de at-least-once.

### A2. At-most-once (enviar uma vez, sem retry)
- **Prós:** nunca duplica.
- **Contras:** perde eventos sempre que o cliente estiver indisponível, o que contradiz a decisão de retry ([ADR-002](ADR-002-retry-com-backoff-exponencial-e-dlq.md)) e o objetivo de eliminar o polling em `GET /orders`.
- **Motivo do descarte:** alternativa plausível, não defendida na reunião; é incompatível com as decisões já tomadas.

## Consequências

### Positivas
- Modelo simples, robusto e alinhado ao padrão de mercado.
- O worker pode adotar estratégias de recuperação conservadoras (por exemplo, devolver para `PENDING` eventos que ficaram em `PROCESSING` após um crash) sem risco de perda.

### Negativas
- Transfere responsabilidade ao cliente: "Isso joga responsabilidade pro cliente" ([09:25] Sofia).
- Clientes que não implementarem deduplicação podem processar o mesmo evento em duplicidade.

### Trade-off explícito
Aceitamos **duplicatas ocasionais e responsabilidade no cliente** em troca de **simplicidade e nenhuma perda de evento**. Segundo o Diego, at-least-once com `event_id` "resolve 99% dos casos" ([09:25] Diego).
