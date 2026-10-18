# Da Reunião ao Documento — Design Docs do Sistema de Webhooks

> Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia#readme).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript, Express, Prisma/MySQL) decidiu numa reunião de ~55 minutos construir um **Sistema de Webhooks de Notificação de Pedidos**, para avisar clientes B2B quando o status de um pedido muda. O único registro da reunião é a transcrição literal (`TRANSCRICAO.md`). A tarefa foi transformar essa conversa, junto com o código existente, num pacote de design docs acionável: PRD, RFC, FDD, 8 ADRs e um tracker de rastreabilidade.

A dificuldade não estava em escrever, e sim em **filtrar e manter a rastreabilidade**. A reunião mistura decisões fechadas, ideias descartadas (Redis, retry indefinido, exactly-once), itens adiados (e-mail, rate limiting, múltiplos workers), correções no meio da fala (o `customer_id` "implícito do JWT" que a Tech Lead corrige um minuto depois) e até ambiguidades numéricas ("5 tentativas" contra 5 intervalos de backoff). Cada afirmação dos documentos precisava apontar para um timestamp da transcrição ou para um arquivo real do repositório.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (app desktop, modelo Claude Opus 5.5) | Ferramenta principal. Leu o repositório inteiro (schema Prisma, `order.service.ts`, middlewares, erros, logger, testes) e a transcrição, extraiu decisões, produziu todos os documentos e executou as verificações automáticas. |
| **Script Python de verificação** (escrito pela IA na sessão, rodado localmente) | Antialucinação: confere se cada `[hh:mm] Nome` citado existe na transcrição, se cada caminho de arquivo existe, se os links relativos estão íntegros e calcula as proporções TRANSCRICAO/CODIGO do tracker. |

## Workflow adotado

1. **Setup:** fork do repositório base e clone local.
2. **Contextualização:** leitura integral de `TRANSCRICAO.md` e do código relevante: `prisma/schema.prisma`, `src/modules/orders/*`, `src/shared/errors/*`, `src/middlewares/*`, `src/config/*`, `src/server.ts`, `src/app.ts`, `src/routes/index.ts`, `src/shared/logger/index.ts`, `package.json` e `tests/*`.
3. **Inventário da reunião:** classificação de cada fala em *decisão fechada*, *requisito*, *restrição*, *descartado*, *adiado* ou *ponto ambíguo*, sempre com timestamp e falante (prompt 1).
4. **ADRs primeiro** (8 arquivos): as 6 decisões principais mais 2 secundárias que tinham alternativas reais (payload snapshot/filtro na inserção e publicação via `publishWebhookEvent(tx, …)`).
5. **RFC:** visão de arquitetura, alternativas descartadas e questões em aberto, com links para os ADRs e sem descer ao nível do FDD.
6. **FDD:** fluxos, contratos (7 endpoints + payload/headers de saída), matriz `WEBHOOK_*`, resiliência, observabilidade, modelo de dados e a seção "Integração com o sistema existente" com 14 arquivos reais.
7. **PRD:** consolidação em linguagem de produto, por último entre os grandes documentos.
8. **Tracker:** varredura dos documentos prontos, com uma linha por ID.
9. **Verificação automática + revisão crítica** contra o checklist de aceite (prompt 3 e script). Os achados estão em "Iterações e ajustes".

Cada documento recebeu IDs estáveis (`PRD-FR-07`, `RFC-ALT-02`, `FDD-CONTRATO-05`, `FDD-INT-01`…). O tracker referencia esses IDs, o que torna a cobertura verificável.

## Prompts customizados

**Prompt 1: inventário dirigido da transcrição** (em vez de "gere um PRD a partir da reunião")

```text
Leia TRANSCRICAO.md inteira. NÃO escreva nenhum documento ainda.
Produza uma tabela com uma linha por item, com as colunas:
  timestamp | falante | item | categoria | observação
Categorias permitidas: DECISAO_FECHADA, REQUISITO_FUNCIONAL, REQUISITO_NAO_FUNCIONAL,
RESTRICAO, DESCARTADO, ADIADO, AMBIGUO, DETALHE_SECUNDARIO.
Regras:
- Se uma fala é corrigida depois (ex.: alguém propõe X e outro participante corrige),
  registre AS DUAS e marque a primeira como DESCARTADO, citando quem corrigiu.
- Se números da reunião não fecham entre si, marque AMBIGUO e mostre a conta.
- Para cada DESCARTADO/ADIADO, cite o trade-off que motivou, com as palavras do falante.
- Proibido inferir requisitos que ninguém disse. Na dúvida, categoria AMBIGUO.
```

**Prompt 2: ADR ancorado no código**

```text
Escreva o ADR-008 (publicação do evento dentro de OrderService.changeStatus) em
formato MADR: Status, Contexto, Decisão, Alternativas Consideradas, Consequências
(positivas, negativas e trade-off explícito).
Antes de escrever, abra src/modules/orders/order.service.ts e src/app.ts e cite no
Contexto: a assinatura atual do construtor, o tipo do tx usado em $transaction e os
helpers privados que já recebem tx. A decisão deve dizer EXATAMENTE em que ponto
da transação a chamada entra.
Inclua como alternativas as opções efetivamente rejeitadas na reunião
([09:41] Diego) e cite timestamps.
Liste nas consequências negativas pelo menos um efeito colateral que a reunião
NÃO discutiu, mas que o código revela.
```

**Prompt 3: revisão adversarial antes da entrega**

```text
Atue como revisor cético do pacote em docs/. Para cada documento:
1. Liste toda afirmação que não tem timestamp da transcrição nem arquivo do repo.
   Para cada uma: remover, ancorar numa fonte ou rotular como "derivação técnica".
2. Procure itens DESCARTADOS/ADIADOS na reunião que tenham voltado como requisito.
3. Procure contradições entre a regra documentada e o comportamento real do código
   (middlewares, classes de erro, FKs do schema).
4. Verifique se o RFC está repetindo detalhe do FDD (rotas completas, schemas,
   pseudocódigo). Se estiver, mova para o FDD.
Depois rode o script de verificação e corrija tudo o que ele apontar.
```

## Iterações e ajustes

Foram **4 ciclos principais**: inventário → ADRs → RFC/FDD/PRD → revisão e verificação. Os ajustes mais relevantes:

1. **`customer_id` vindo do JWT.** Na primeira extração, o requisito de cadastro saiu como "customer_id implícito do JWT", seguindo a fala de [09:31] Marcos. A revisão mostrou que Bruno ([09:32]) aponta que o JWT é do usuário operador e Larissa corrige: "o customer_id é passado no body ou no path. Não vem do JWT". O requisito foi reescrito (PRD-FR-02) e a limitação de autorização entrou como questão em aberto (RFC Q-04).
2. **"5 tentativas" não fecha com a conta.** A política inicial dizia "5 tentativas no total". Somando os intervalos (1m+5m+30m+2h+12h ≈ 14 h 36 min), que Diego descreve como "quase 15 horas entre primeira falha e última tentativa", fica claro que são **1 envio inicial + 5 retentativas**. Isso foi documentado no ADR-002, com tabela, e levado como Q-06 no RFC para confirmação.
3. **`WEBHOOK_INVALID_URL` como código de topo era impossível.** A primeira matriz de erros devolvia `400 WEBHOOK_INVALID_URL`. Lendo `src/middlewares/validate.middleware.ts`, todo `ZodError` vira `VALIDATION_ERROR`, e a Sofia pediu explicitamente que a regra `https` ficasse no schema Zod. A matriz passou a documentar o código dentro de `details[].message`, sem mexer no middleware, o que respeita o reuso sem alteração decidido em [09:29] Bruno.
4. **`NotFoundError` não aceita código customizado.** O rascunho reaproveitava `NotFoundError` para `WEBHOOK_NOT_FOUND`, mas o construtor em `src/shared/errors/http-errors.ts` fixa `NOT_FOUND`. As classes 404 do módulo passaram a estender `AppError` diretamente.
5. **Limite de 64 KB no lugar errado.** A primeira versão validava o tamanho dentro de `publishWebhookEvent`, ou seja, dentro da transação: um payload anômalo bloquearia a mudança de status do pedido, contradizendo o motivo de não fazer o envio síncrono ([09:04] Bruno). A checagem foi movida para o worker, com DLQ imediata.
6. **FK de `webhook_outbox.order_id` quebraria um fluxo existente.** O modelo inicial tinha FK para `orders`. Como `OrderService.delete` apaga pedidos `CANCELLED`, e um cancelamento gera evento, a FK causaria erro Prisma `P2003` (500). A coluna ficou indexada e sem FK (FDD §9).
7. **Ordenação sob retry.** A reunião registrou ordenação por `order_id` com single-worker, mas um evento em backoff deixa o evento seguinte do mesmo pedido passar à frente. Isso não foi promovido a decisão inventada: virou limitação documentada e questão em aberto (RFC Q-05).
8. **Citação errada no tracker.** O script de verificação apontou uma linha de teste atribuída ao Bruno às 09:38, minuto em que ele não fala. O correto era `[09:40] Bruno`. Depois da correção, o script passou sem nenhuma citação inválida.

Resultado da verificação final: todas as linhas do tracker com fonte TRANSCRICAO usam `[hh:mm] Nome` existente (cerca de 87% das linhas); todas as linhas CODIGO apontam para arquivos reais; 8 ADRs no padrão de nome; nenhum link relativo quebrado. Os únicos caminhos citados que não existem são os **arquivos novos propostos** (`src/worker.ts` e `src/modules/webhooks/*`), explicitamente marcados como tal no FDD §11.2 e nomeados na própria reunião.

## Como navegar a entrega

```
docs/
├── PRD.md          por quê e o quê (produto)
├── RFC.md          proposta de arquitetura, alternativas, questões em aberto
├── adrs/
│   ├── README.md   índice dos ADRs
│   ├── ADR-001-outbox-transacional-no-mysql.md
│   ├── ADR-002-retry-com-backoff-exponencial-e-dlq.md
│   ├── ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md
│   ├── ADR-004-entrega-at-least-once-com-x-event-id.md
│   ├── ADR-005-worker-separado-com-polling.md
│   ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
│   ├── ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md
│   └── ADR-008-publicacao-via-funcao-transacional-no-changestatus.md
├── FDD.md          como construir (fluxos, contratos, erros, integração)
└── TRACKER.md      origem de cada item
```

**Ordem de leitura sugerida:**

1. [PRD](docs/PRD.md): problema, escopo e o que fica de fora.
2. [RFC](docs/RFC.md): a proposta em 2–4 páginas e o que ainda está aberto.
3. [ADRs](docs/adrs/README.md): o porquê de cada decisão.
4. [FDD](docs/FDD.md): para quem vai implementar; comece pela §11 (Integração com o sistema existente).
5. [TRACKER](docs/TRACKER.md): para auditar qualquer afirmação contra a transcrição ou o código.
