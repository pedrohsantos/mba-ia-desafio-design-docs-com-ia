# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) |
| **Feature** | Notificação de mudança de status de pedido via webhooks |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião; documentação em revisão |
| **Data** | 2026-10-18 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto

Os clientes B2B do OMS precisam saber quando o status dos seus pedidos muda (por exemplo, quando um pedido é enviado ou entregue). Hoje a única forma de descobrir isso é consultar `GET /orders` repetidamente. Esta feature permite que cada cliente **cadastre uma URL** e passe a **receber automaticamente uma notificação assinada** sempre que um pedido dele mudar para um status de interesse, em menos de 10 segundos.

A feature atende a um pedido formal de três clientes, **Atlas Comercial, MaxDistribuição e Nova Cargo** ([09:00] Marcos), e é a primeira capacidade de notificação externa do OMS.

## 2. Problema e motivação

| # | Problema | Evidência |
|---|---|---|
| P1 | Os clientes descobrem mudanças de status por **polling** em `GET /orders`, o que torna a integração "lenta e cara pra eles". | [09:00] Marcos |
| P2 | **Risco comercial:** a Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre. | [09:00] Marcos |
| P3 | A plataforma não tem nenhum mecanismo de notificação externa: módulos de auth, users, customers, products e orders, sem eventos nem filas. | `src/app.ts` |

**Por que agora:** prazo pedido pela Atlas (fim de novembro, [09:45] Marcos) e risco explícito de churn.

## 3. Público-alvo e cenários de uso

### 3.1 Personas

| Persona | Descrição | Necessidade |
|---|---|---|
| **Integrador do cliente B2B** (Atlas, MaxDistribuição, Nova Cargo) | Time técnico do cliente que integra seus sistemas ao OMS via API | Receber mudanças de status sem polling, com garantia de autenticidade |
| **Usuário que representa o cliente na API** | Usuário do OMS (JWT nosso) que opera em nome do cliente | Cadastrar e manter webhooks e consultar o histórico de entregas ([09:32] Marcos) |
| **Administrador do OMS (role ADMIN)** | Operação interna | Reprocessar notificações que falharam definitivamente ([09:36] Sofia) |

### 3.2 Cenários

| ID | Cenário |
|---|---|
| PRD-CEN-01 | A Atlas cadastra `https://…/oms/orders` querendo só `SHIPPED` e `DELIVERED`. Quando um pedido dela é despachado, o sistema dela recebe a notificação em segundos e atualiza o rastreio sem consultar `GET /orders` ([09:33] Marcos). |
| PRD-CEN-02 | O endpoint da Nova Cargo fica 2 h fora do ar por manutenção planejada. As notificações do período chegam automaticamente quando o endpoint volta ([09:16] Diego). |
| PRD-CEN-03 | A MaxDistribuição suspeita que vazou a secret num log. Ela pede uma nova secret pela API e tem 24 h para atualizar os sistemas sem perder notificações ([09:21] Sofia, [09:22] Diego). |
| PRD-CEN-04 | Um cliente reclama que "não recebeu" um evento. O suporte consulta o histórico de entregas (status, resposta, tempo) e, se o evento estiver na fila de falhas, um ADMIN reprocessa ([09:34] Marcos, [09:18] Diego). |
| PRD-CEN-05 | O cliente recebe o mesmo evento duas vezes (por exemplo, depois de um timeout) e descarta a duplicata pelo `X-Event-Id` ([09:25] Diego). |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| PRD-OBJ-01 | Notificação em "tempo real" | Tempo entre a mudança de status e a entrega bem-sucedida na **primeira tentativa** (p95) | **< 10 segundos** | [09:02] Marcos |
| PRD-OBJ-02 | Atender os clientes que pediram a feature | Clientes solicitantes com webhook ativo em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) | [09:00] Marcos |
| PRD-OBJ-03 | Entregar no prazo comercial | Feature em produção, com revisão de segurança concluída | **Até o fim de novembro**, em **3 sprints** | [09:45] Marcos, [09:47] Larissa |
| PRD-OBJ-04 | Não perder notificações | Eventos de mudança de status commitados sem registro de entrega, retry ou DLQ | **0** | [09:06] Diego |
| PRD-OBJ-05 | Resiliência a indisponibilidade do cliente | Janela coberta por retentativas automáticas antes de falha definitiva | **≈ 15 h** (1m/5m/30m/2h/12h) | [09:17] Diego |

**Sinal de sucesso qualitativo:** os três clientes deixam de depender de polling em `GET /orders` para acompanhar status (P1).

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de webhooks por customer, com filtro de status.
- Notificação automática a cada mudança de status de pedido, para os webhooks interessados.
- Assinatura de cada notificação e rotação de secret.
- Retentativas automáticas, fila de falhas definitivas (DLQ) e reprocessamento manual por ADMIN.
- Histórico das últimas entregas por webhook.

### 5.2 Fora de escopo

| ID | Item | Situação | Origem |
|---|---|---|---|
| PRD-OUT-01 | **E-mail ao cliente** quando o webhook falha repetidamente | Adiado para a próxima fase, depois de medir o impacto | [09:37] Larissa |
| PRD-OUT-02 | **Dashboard visual** para o cliente ver seus webhooks | Descartado nesta fase; projeto separado do time de frontend | [09:40] Larissa |
| PRD-OUT-03 | **Rate limiting** de envio por cliente | Não entra; observar e decidir depois | [09:39] Larissa |
| PRD-OUT-04 | **Webhooks de entrada** (cliente → OMS) | Descartado; o fluxo é só de saída | [09:02] Marcos |
| PRD-OUT-05 | **Garantia de ordem global** de eventos | Descartado; só ordem por pedido, com limitações | [09:13] Larissa, [09:14] Marcos |
| PRD-OUT-06 | **Arquivamento** de notificações antigas (ex.: 30 dias) | Fora desta feature | [09:08] Diego |
| PRD-OUT-07 | **Restrição por role / posse** no cadastro de webhooks | Adiado ("mais pra frente a gente pode endurecer") | [09:37] Sofia |
| PRD-OUT-08 | **Itens do pedido** dentro da notificação | Descartado; o cliente consulta `GET /orders/:id` | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | O usuário autenticado pode **cadastrar um webhook** para um customer informando URL e lista de status de interesse; o sistema **gera e devolve a secret** na criação. | [09:31] Marcos |
| PRD-FR-02 | O customer do webhook é **informado na requisição**, não derivado do usuário logado. | [09:32] Larissa |
| PRD-FR-03 | O usuário pode **listar** os webhooks de um customer. | [09:33] Bruno |
| PRD-FR-04 | O usuário pode **editar** um webhook (URL, status de interesse, ativo/inativo). | [09:33] Bruno |
| PRD-FR-05 | O usuário pode **remover** um webhook. | [09:33] Bruno |
| PRD-FR-06 | Cada webhook recebe **apenas os status que escolheu** (ex.: só `SHIPPED` e `DELIVERED`). | [09:33] Marcos |
| PRD-FR-07 | A cada **mudança de status** de um pedido, cada webhook interessado do customer recebe uma notificação com identificação do evento, pedido, status anterior e novo status, momento da mudança e valores básicos do pedido. | [09:43] Diego |
| PRD-FR-08 | Cada notificação traz um **identificador único do evento** (`X-Event-Id`) para o cliente descartar duplicatas, e o **identificador do webhook** (`X-Webhook-Id`) para quem tem vários cadastros. | [09:25] Diego, [09:44] Sofia |
| PRD-FR-09 | Cada notificação é **assinada** de forma que o cliente consiga verificar origem e integridade. | [09:20] Sofia |
| PRD-FR-10 | O cliente pode **pedir uma nova secret** pela API; a anterior continua aceita por **24 horas**. | [09:21] Sofia |
| PRD-FR-11 | Se o endpoint do cliente falhar, o sistema **tenta novamente automaticamente** (5 retentativas: 1 min, 5 min, 30 min, 2 h, 12 h). | [09:17] Larissa |
| PRD-FR-12 | Esgotadas as tentativas, a notificação vai para uma **fila de falhas definitivas (DLQ)** com o motivo da falha. | [09:18] Diego |
| PRD-FR-13 | Um **ADMIN** pode **reprocessar** uma notificação da DLQ; a ação fica **registrada com o autor** para auditoria. | [09:18] Diego, [09:36] Sofia |
| PRD-FR-14 | O usuário pode ver o **histórico das últimas 100 notificações** de um webhook, com sucesso/falha, conteúdo enviado, resposta recebida e tempo de resposta. | [09:34] Marcos |

## 7. Requisitos não funcionais

| ID | Categoria | Requisito | Origem |
|---|---|---|---|
| PRD-NFR-01 | Latência | Notificação entregue em **< 10 s** após a mudança de status, em operação normal (o polling acrescenta no máximo ~2 s) | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | Confiabilidade | Mudança de status e registro da notificação são **atômicos**: não existe status alterado sem notificação registrada, nem o contrário | [09:40] Bruno |
| PRD-NFR-03 | Isolamento | Cliente lento ou fora do ar **não afeta** a mudança de status de nenhum pedido | [09:04] Bruno |
| PRD-NFR-04 | Semântica de entrega | **At-least-once**: o cliente pode receber duplicatas e deve deduplicar pelo `X-Event-Id` | [09:24] Diego |
| PRD-NFR-05 | Segurança | Assinatura **HMAC-SHA256**, com **secret única por endpoint** | [09:22] Sofia |
| PRD-NFR-06 | Segurança | URL do webhook **obrigatoriamente `https`**; `http` é recusado | [09:23] Sofia |
| PRD-NFR-07 | Limite | Notificações acima de **64 KB não são enviadas** (erro, sem truncar) | [09:24] Larissa |
| PRD-NFR-08 | Timeout | O cliente tem **10 s** para responder; senão a tentativa conta como falha | [09:42] Diego |
| PRD-NFR-09 | Ordem | Ordem preservada **por pedido** enquanto houver um único processador; **sem** garantia de ordem global | [09:13] Larissa |
| PRD-NFR-10 | Autorização | Cadastro de webhooks: qualquer usuário autenticado. Reprocessamento da DLQ: somente **ADMIN** | [09:36] Larissa, [09:37] Sofia |
| PRD-NFR-11 | Operação | O processamento de notificações roda **separado da API** e não para quando a API reinicia | [09:11] Diego |
| PRD-NFR-12 | Custo/stack | **Sem infraestrutura nova**: usa o MySQL e a stack existentes | [09:07] Diego |

## 8. Decisões e trade-offs principais

| ID | Decisão | Trade-off aceito | ADR |
|---|---|---|---|
| PRD-DEC-01 | Registrar notificações numa outbox no MySQL, na mesma transação da mudança de status | Até ~2 s de latência extra, em troca de não perder eventos e não subir infraestrutura nova | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| PRD-DEC-02 | 5 retentativas em ~15 h, depois DLQ com reprocessamento manual | Notificação pode chegar até ~15 h atrasada; replay depende de um ADMIN | [ADR-002](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md) |
| PRD-DEC-03 | Assinatura HMAC com secret por endpoint e rotação com 24 h de convivência | Mais complexidade, em troca de isolar vazamentos | [ADR-003](adrs/ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| PRD-DEC-04 | At-least-once com `X-Event-Id` | O cliente precisa deduplicar, em troca de simplicidade e de não perder eventos | [ADR-004](adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) |
| PRD-DEC-05 | Processador separado, consultando a cada 2 s | Ponto único de processamento e ordem só por pedido | [ADR-005](adrs/ADR-005-worker-separado-com-polling.md) |
| PRD-DEC-06 | Notificação enxuta, sem itens, congelada no momento da mudança | Cliente que quer detalhes faz uma segunda chamada | [ADR-007](adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Responsável | Origem |
|---|---|---|---|
| PRD-DEP-01 | **Revisão de segurança** (HMAC e geração de secret): pelo menos 2 dias úteis antes do deploy | Sofia | [09:46] Sofia |
| PRD-DEP-02 | **Documentação no portal de desenvolvedor** (integração via API, deduplicação por `event_id`) | Marcos | [09:26] Marcos, [09:40] Marcos |
| PRD-DEP-03 | **Comunicação de prazo** aos clientes | Marcos | [09:47] Marcos |
| PRD-DEP-04 | **Sessão de revisão do design doc** com Bruno e Diego antes de começar a codar | Larissa | [09:50] Larissa |
| PRD-DEP-05 | **Clientes** expõem endpoint `https`, validam a assinatura e deduplicam eventos | Integradores dos clientes | [09:23] Sofia, [09:25] Diego |
| PRD-DEP-06 | **Fluxo de mudança de status existente** (máquina de estados e auditoria de status) | Time de Pedidos | `src/modules/orders/order.service.ts` |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RSK-01 | **Atraso além do fim de novembro** e perda da Atlas | Média | Alto | Estimativa de 3 sprints com a revisão de segurança já incluída; escopo enxuto (e-mail, dashboard e rate limiting fora) ([09:46] Larissa, [09:47] Larissa) |
| PRD-RSK-02 | **Cliente não trata duplicatas** e processa o mesmo evento duas vezes | Média | Médio | Documentação em destaque no portal ([09:26] Marcos); `X-Event-Id` em todas as entregas |
| PRD-RSK-03 | **Vazamento de secret** por parte do cliente | Média | Alto | Secret por endpoint, rotação com grace period de 24 h ([09:21] Sofia); histórico de vazamento já conhecido ([09:22] Diego) |
| PRD-RSK-04 | **Cliente lento ou fora do ar** atrasa as notificações dos demais | Média | Alto | Timeout de 10 s, processamento paralelo entre pedidos, alerta de fila parada (FDD §7–8) |
| PRD-RSK-05 | **Cliente fora do ar por mais de ~15 h** perde notificações para a DLQ | Baixa | Médio | Histórico de entregas + replay manual ADMIN ([09:18] Diego); notificação por e-mail planejada para fase futura |
| PRD-RSK-06 | **Rajadas de mudanças** (ex.: 50 pedidos/minuto) sobrecarregam o cliente | Baixa | Médio | Monitorar o volume por webhook; rate limiting fica como questão em aberto ([09:38] Diego) |

## 11. Critérios de aceitação

| ID | Critério (Dado / Quando / Então) | Requisitos |
|---|---|---|
| PRD-CA-01 | **Dado** um webhook ativo que escuta `SHIPPED`, **quando** um pedido do customer muda para `SHIPPED`, **então** o endpoint recebe uma notificação assinada em menos de 10 s. | FR-06, FR-07, FR-09, NFR-01 |
| PRD-CA-02 | **Dado** um webhook que escuta só `DELIVERED`, **quando** o pedido muda para `PAID`, **então** nenhuma notificação é gerada para ele. | FR-06 |
| PRD-CA-03 | **Dado** um cadastro com URL `http://…`, **quando** enviado, **então** é recusado com erro de validação. | NFR-06 |
| PRD-CA-04 | **Dado** um endpoint que responde 500, **quando** a notificação falha, **então** o sistema tenta de novo em 1 min, 5 min, 30 min, 2 h e 12 h e, depois disso, coloca a notificação na DLQ. | FR-11, FR-12 |
| PRD-CA-05 | **Dado** um evento na DLQ, **quando** um OPERATOR tenta reprocessar, **então** recebe acesso negado; **quando** um ADMIN reprocessa, **então** o evento volta a ser entregue com o mesmo `X-Event-Id`, e o autor fica registrado. | FR-13, NFR-10 |
| PRD-CA-06 | **Dado** um pedido de rotação de secret, **quando** concluído, **então** a nova secret é devolvida e notificações assinadas com a anterior seguem verificáveis por 24 h. | FR-10 |
| PRD-CA-07 | **Dado** um webhook com entregas, **quando** o histórico é consultado, **então** aparecem até 100 itens com resultado, resposta e tempo de resposta. | FR-14 |
| PRD-CA-08 | **Dado** que a gravação da notificação falha, **quando** a mudança de status é solicitada, **então** o status do pedido **não** muda. | NFR-02 |
| PRD-CA-09 | **Dado** que a API reinicia, **quando** há notificações pendentes, **então** elas continuam sendo entregues. | NFR-11 |

## 12. Estratégia de testes e validação

| Camada | O que valida | Como |
|---|---|---|
| **Testes automatizados** | Requisitos funcionais, atomicidade, retry, DLQ, assinatura e autorização | Suíte Vitest + Supertest do projeto (`tests/`), com casos listados no [FDD §12](FDD.md#12-estratégia-de-testes) |
| **Revisão de segurança** | HMAC, geração e armazenamento de secret, logs | Sofia, 2 dias úteis antes do deploy ([09:46] Sofia) |
| **Revisão de design** | Coerência do desenho antes de codar | Sessão Larissa + Bruno + Diego ([09:50] Larissa) |
| **Validação com clientes** | Integração real ponta a ponta | Ativar os webhooks dos 3 clientes solicitantes e acompanhar PRD-OBJ-01 e PRD-OBJ-02 |
| **Monitoramento pós-lançamento** | Latência, taxa de falha, volume da DLQ, volume por cliente (insumo para decidir sobre rate limiting e e-mail) | Logs estruturados e consultas descritos no [FDD §8](FDD.md#8-observabilidade) |
