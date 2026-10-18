# Tracker de Rastreabilidade

Este documento liga cada item identificável do pacote (PRD, RFC, FDD e ADRs) à sua origem na reunião (`TRANSCRICAO.md`) ou no código-fonte do repositório.

**Convenções**

- **Fonte = TRANSCRICAO:** a Localização é `[hh:mm] Nome`, o início de uma fala em `TRANSCRICAO.md`. Quando várias falas sustentam o item, aparece a mais representativa e as demais estão citadas no próprio documento.
- **Fonte = CODIGO:** a Localização é o caminho de um arquivo existente no repositório.
- **Tipo "Derivação técnica":** detalhe de implementação que não foi dito literalmente na reunião, mas decorre de uma decisão dela ou de uma restrição do código. A linha aponta a fala ou o arquivo que o justifica. Esses itens são os primeiros candidatos a validação na sessão de revisão do design ([09:50] Larissa).
- Todas as localizações foram verificadas por script: os timestamps e falantes existem na transcrição e os caminhos existem no repositório (ver README, seção "Iterações e ajustes").

## PRD — `docs/PRD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo por notificação de status | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-01 | docs/PRD.md | Problema | Polling em `GET /orders` torna a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-02 | docs/PRD.md | Problema | Risco de a Atlas migrar para concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-PRB-03 | docs/PRD.md | Restrição | OMS não tem mecanismo de notificação externa (só auth, users, customers, products, orders) | CODIGO | src/app.ts |
| PRD-PER-01 | docs/PRD.md | Público-alvo | Usuários do OMS representam o cliente e usam a API com JWT nosso | TRANSCRICAO | [09:32] Marcos |
| PRD-PER-02 | docs/PRD.md | Público-alvo | ADMIN opera o replay da DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-01 | docs/PRD.md | Cenário | Cliente escuta só `SHIPPED` e `DELIVERED` | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Endpoint indisponível 2 h por manutenção planejada recebe depois | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Cliente rotaciona secret após suspeita de vazamento em log | TRANSCRICAO | [09:22] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário | Suporte consulta histórico de entregas e ADMIN reprocessa DLQ | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-05 | docs/PRD.md | Cenário | Cliente descarta duplicata pelo `X-Event-Id` | TRANSCRICAO | [09:25] Diego |
| PRD-OBJ-01 | docs/PRD.md | Métrica | p95 de entrega na primeira tentativa abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 3 de 3 clientes solicitantes com webhook ativo | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Produção até o fim de novembro em 3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Zero eventos commitados perdidos | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Janela de retry de aproximadamente 15 h | TRANSCRICAO | [09:17] Diego |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo (adiado) | E-mail ao cliente em falhas repetidas fica para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo (descartado) | Dashboard visual é projeto do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo (adiado) | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo (descartado) | Webhooks inbound; fluxo só de saída | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo (descartado) | Ordem global de eventos não é garantida | TRANSCRICAO | [09:13] Larissa |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo (adiado) | Arquivamento de entregues após cerca de 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo (adiado) | Endurecimento de roles no CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| PRD-OUT-08 | docs/PRD.md | Fora de escopo (descartado) | Itens do pedido no payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL e status; secret gerada pelo servidor e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | customerId informado no body ou path, não derivado do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Notificação a cada mudança de status com payload definido | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | `X-Event-Id` para dedup e `X-Webhook-Id` para múltiplos cadastros | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Notificação assinada para verificação de origem e integridade | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com a anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retry automático 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | DLQ com motivo da falha após esgotar as tentativas | TRANSCRICAO | [09:18] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Replay de DLQ por ADMIN com registro do autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência abaixo de 10 s; polling soma no máximo 2 s | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre status e registro do evento | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não afeta mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB, com erro e sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordem apenas por pedido e com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | CRUD para qualquer role autenticada; replay só ADMIN | TRANSCRICAO | [09:36] Larissa |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Processamento separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-12 | docs/PRD.md | Restrição | Sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox no MySQL na mesma transação | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão | 5 retentativas e DLQ com replay manual | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão | HMAC com secret por endpoint e rotação de 24 h | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-04 | docs/PRD.md | Decisão | At-least-once com `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-05 | docs/PRD.md | Decisão | Worker separado com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão | Payload enxuto em snapshot | TRANSCRICAO | [09:52] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Comunicação de prazo aos clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Sessão de revisão do design com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência | Cliente valida assinatura e deduplica | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Fluxo de mudança de status existente | CODIGO | src/modules/orders/order.service.ts |
| PRD-RSK-01 | docs/PRD.md | Risco | Atraso além do fim de novembro | TRANSCRICAO | [09:46] Larissa |
| PRD-RSK-02 | docs/PRD.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| PRD-RSK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-04 | docs/PRD.md | Risco | Cliente lento atrasa os demais | TRANSCRICAO | [09:04] Bruno |
| PRD-RSK-05 | docs/PRD.md | Risco | Indisponibilidade acima de 15 h leva o evento à DLQ | TRANSCRICAO | [09:17] Marcos |
| PRD-RSK-06 | docs/PRD.md | Risco | Rajadas de eventos sobrecarregam o cliente | TRANSCRICAO | [09:38] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de aceitação | Entrega assinada em menos de 10 s para status filtrado | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de aceitação | Status fora do filtro não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-03 | docs/PRD.md | Critério de aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de aceitação | Retry na sequência e depois DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-CA-05 | docs/PRD.md | Critério de aceitação | Replay negado a OPERATOR e permitido a ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-06 | docs/PRD.md | Critério de aceitação | Rotação mantém a anterior por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de aceitação | Histórico com até 100 itens | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-08 | docs/PRD.md | Critério de aceitação | Falha na outbox impede a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-CA-09 | docs/PRD.md | Critério de aceitação | Restart da API não interrompe entregas | TRANSCRICAO | [09:11] Diego |
| PRD-TST-01 | docs/PRD.md | Estratégia de teste | Suíte Vitest + Supertest existente | CODIGO | vitest.config.ts |

## RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Metadado | Revisores são os 5 participantes; revisão com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Transação de `changeStatus` já é pesada (orders, history, estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-02 | docs/RFC.md | Contexto | `changeStatus` em `$transaction` com validação, estoque e histórico | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-03 | docs/RFC.md | Contexto | Máquina de estados de pedido | CODIGO | src/modules/orders/order.status.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Visão geral: outbox, worker, HMAC, retry, DLQ, at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-02 | docs/RFC.md | Contrato | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo HTTP síncrono em `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / Redis Cluster | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido ou 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Prazo de 3 sprints com revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída por cliente | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers (particionar ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | E-mail em falhas repetidas | TRANSCRICAO | [09:37] Marcos |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | Endurecer autorização do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-05 | docs/RFC.md | Questão em aberto (derivação técnica) | Ordem sob retry quebra a ordenação implícita por pedido | TRANSCRICAO | [09:12] Diego |
| RFC-Q-06 | docs/RFC.md | Questão em aberto (ambiguidade) | "5 tentativas" contra 5 intervalos somando cerca de 15 h | TRANSCRICAO | [09:17] Diego |
| RFC-Q-07 | docs/RFC.md | Questão em aberto | Detalhes de HMAC e secret para a revisão de segurança | TRANSCRICAO | [09:46] Sofia |

## FDD — `docs/FDD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CTX-01 | docs/FDD.md | Contexto | Passos atuais de `changeStatus` (validação, estoque, update, history) | CODIGO | src/modules/orders/order.service.ts |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Atomicidade status e outbox | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Latência abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Nenhum evento commitado perdido | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Isolamento de falhas do cliente | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Envio assinado HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| FDD-OBJ-06 | docs/FDD.md | Restrição | Sem dependências novas | TRANSCRICAO | [09:29] Bruno |
| FDD-EXC-01 | docs/FDD.md | Exclusão | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-02 | docs/FDD.md | Exclusão | E-mail de falha | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | docs/FDD.md | Exclusão | Rate limiting | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | docs/FDD.md | Exclusão | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | docs/FDD.md | Exclusão | Arquivamento de entregues | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-06 | docs/FDD.md | Exclusão | Múltiplos workers | TRANSCRICAO | [09:13] Diego |
| FDD-EXC-07 | docs/FDD.md | Exclusão | Evento na criação do pedido; integração apenas em `changeStatus` | TRANSCRICAO | [09:40] Bruno |
| FDD-EXC-08 | docs/FDD.md | Exclusão | Itens no payload | TRANSCRICAO | [09:43] Diego |
| FDD-EXC-09 | docs/FDD.md | Exclusão | Restrição de role e posse no CRUD | TRANSCRICAO | [09:37] Sofia |
| FDD-FLX-01 | docs/FDD.md | Fluxo | `publishWebhookEvent` dentro do callback de `$transaction` com o mesmo tx | TRANSCRICAO | [09:41] Bruno |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Filtro por webhooks ativos do customer que escutam o `to_status` | TRANSCRICAO | [09:34] Bruno |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Uma linha por webhook; `id` da linha = `event_id` UUID | TRANSCRICAO | [09:25] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:52] Larissa |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Publicação sem I/O de rede e sem capturar exceção | TRANSCRICAO | [09:41] Diego |
| FDD-FLX-06 | docs/FDD.md | Derivação técnica | Checagem de 64 KB no worker para não bloquear a mudança de status | TRANSCRICAO | [09:24] Larissa |
| FDD-FLX-07 | docs/FDD.md | Fluxo | Polling de 2 s com batch pequeno dos mais antigos | TRANSCRICAO | [09:09] Diego |
| FDD-FLX-08 | docs/FDD.md | Fluxo | Leitura por `created_at` e sequência por `order_id` | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-09 | docs/FDD.md | Restrição | Instância única do worker | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-10 | docs/FDD.md | Fluxo | Timeout de 10 s conta como falha com retry | TRANSCRICAO | [09:42] Diego |
| FDD-FLX-11 | docs/FDD.md | Derivação técnica | Recuperação de PROCESSING órfão, segura por ser at-least-once | TRANSCRICAO | [09:24] Diego |
| FDD-FLX-12 | docs/FDD.md | Fluxo | Payload acima de 64 KB não é enviado nem truncado | TRANSCRICAO | [09:23] Sofia |
| FDD-FLX-13 | docs/FDD.md | Derivação técnica | Não seguir redirects para preservar https | TRANSCRICAO | [09:23] Sofia |
| FDD-FLX-14 | docs/FDD.md | Fluxo | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-15 | docs/FDD.md | Fluxo | Status da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-16 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-17 | docs/FDD.md | Fluxo | Replay registra o autor para auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-FLX-18 | docs/FDD.md | Fluxo | Rotação: previousSecret válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLX-19 | docs/FDD.md | Derivação técnica | Geração de secret com `node:crypto`, sujeita à revisão | TRANSCRICAO | [09:46] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks` com url, events e customerId; devolve a secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks?customerId=` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /webhooks/:id/rotate-secret` | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` com os últimos 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Payload `order.status_changed` enxuto com campos definidos | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Header `X-Webhook-Id` | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Prefixo `/api/v1` e formato de erro padrão | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-12 | docs/FDD.md | Restrição | Filtro rejeita PENDING porque nenhuma transição leva a PENDING | CODIGO | src/modules/orders/order.status.ts |
| FDD-CONTRATO-13 | docs/FDD.md | Derivação técnica | Campos `*_cents` e `order_number` vêm do model `Order` | CODIGO | prisma/schema.prisma |
| FDD-CONTRATO-14 | docs/FDD.md | Derivação técnica | Secret só nas respostas de criação e rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | Prefixo `WEBHOOK_` em todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-02 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` via schema Zod | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-04 | docs/FDD.md | Restrição | Zod é convertido para `VALIDATION_ERROR` pelo middleware de validação | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERR-05 | docs/FDD.md | Restrição | `NotFoundError` fixa `NOT_FOUND`; erros do módulo estendem `AppError` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-06 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` reservado ao worker, pois a secret é gerada pelo servidor | TRANSCRICAO | [09:31] Marcos |
| FDD-ERR-07 | docs/FDD.md | Erro | `FORBIDDEN` do `requireRole` para não-ADMIN no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-ERR-08 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-09 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-10 | docs/FDD.md | Erro | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | TRANSCRICAO | [09:15] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | 5 retentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Fallback DLQ com replay; sem e-mail | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | Polling de 2 s | TRANSCRICAO | [09:09] Diego |
| FDD-RES-06 | docs/FDD.md | Resiliência | Recuperação de crash no startup | TRANSCRICAO | [09:24] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Shutdown gracioso no padrão do server | CODIGO | src/server.ts |
| FDD-RES-08 | docs/FDD.md | Resiliência | Processos API e worker isolados | TRANSCRICAO | [09:11] Diego |
| FDD-RES-09 | docs/FDD.md | Resiliência | Limite de 64 KB | TRANSCRICAO | [09:24] Diego |
| FDD-RES-10 | docs/FDD.md | Derivação técnica | Paralelismo por `order_id` dentro do batch para o cliente lento não travar os demais | TRANSCRICAO | [09:12] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs via Pino existente, sem ferramenta nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logger com `redactPaths` a estender com secret | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log de auditoria do replay com userId | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Métrica de lag contra a meta de 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Envios por minuto como insumo para rate limiting | TRANSCRICAO | [09:39] Larissa |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Correlação com requestId do request logger | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DAT-01 | docs/FDD.md | Modelo de dados | Tabela de configuração com url, secret, customer_id e ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DAT-02 | docs/FDD.md | Modelo de dados | Índices por status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DAT-03 | docs/FDD.md | Modelo de dados | IDs UUID | TRANSCRICAO | [09:51] Larissa |
| FDD-DAT-04 | docs/FDD.md | Derivação técnica | `orderId` sem FK porque `OrderService.delete` apaga pedidos PENDING e CANCELLED | CODIGO | src/modules/orders/order.service.ts |
| FDD-DAT-05 | docs/FDD.md | Derivação técnica | `events` como Json, no mesmo padrão de `Customer.address` | CODIGO | prisma/schema.prisma |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node 20 com fetch e AbortSignal nativos; uuid já presente | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8.0 | CODIGO | docker-compose.yml |
| FDD-DEP-03 | docs/FDD.md | Dependência | Variáveis de ambiente validadas e reaproveitadas pelo worker | CODIGO | src/config/env.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Chamada a `publishWebhookEvent(tx, order, from, to)` em `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-01b | docs/FDD.md | Decisão | Função recebendo tx em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| FDD-INT-03 | docs/FDD.md | Integração | Erros do módulo estendem `AppError` | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Error middleware sem alteração | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-05 | docs/FDD.md | Integração | `requireRole('ADMIN')` reaproveitado como em users | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-05b | docs/FDD.md | Decisão | Reaproveitar o requireRole existente | TRANSCRICAO | [09:36] Larissa |
| FDD-INT-07 | docs/FDD.md | Integração | Registro de rotas no `buildApiRouter` | CODIGO | src/routes/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Wiring em `buildControllers` | CODIGO | src/app.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Novo `src/worker.ts` no modelo de `src/server.ts` | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-10 | docs/FDD.md | Integração | PrismaClient próprio no worker | CODIGO | src/config/database.ts |
| FDD-INT-10b | docs/FDD.md | Decisão | PrismaClient separado por processo, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| FDD-INT-12 | docs/FDD.md | Integração | Novos models e relação em Customer | CODIGO | prisma/schema.prisma |
| FDD-INT-13 | docs/FDD.md | Integração | Script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-14 | docs/FDD.md | Integração | Limpeza das novas tabelas no setup de testes | CODIGO | tests/setup.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Estrutura `src/modules/webhooks` e `webhook.processor.ts` | TRANSCRICAO | [09:28] Bruno |
| FDD-TST-01 | docs/FDD.md | Teste | Inserção na outbox com payload correto | TRANSCRICAO | [09:40] Bruno |
| FDD-TST-03 | docs/FDD.md | Teste | Rollback quando a outbox falha | TRANSCRICAO | [09:40] Bruno |
| FDD-TST-04 | docs/FDD.md | Teste | Assinatura confere com a secret | TRANSCRICAO | [09:20] Sofia |
| FDD-TST-06 | docs/FDD.md | Teste | DLQ após a 6ª falha | TRANSCRICAO | [09:17] Larissa |
| FDD-TST-09 | docs/FDD.md | Teste | Replay restrito a ADMIN | TRANSCRICAO | [09:36] Sofia |
| FDD-TST-11 | docs/FDD.md | Teste | Grace period de rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-08 | docs/FDD.md | Critério de aceite | Revisão de segurança antes do deploy | TRANSCRICAO | [09:49] Sofia |
| FDD-RSK-01 | docs/FDD.md | Risco | Clientes lentos saturam o worker único | TRANSCRICAO | [09:04] Bruno |
| FDD-RSK-03 | docs/FDD.md | Risco | Inversão de ordem sob retry | TRANSCRICAO | [09:12] Diego |
| FDD-RSK-04 | docs/FDD.md | Risco | Secret exposta em log | TRANSCRICAO | [09:22] Diego |
| FDD-RSK-05 | docs/FDD.md | Risco | Crescimento da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-RSK-06 | docs/FDD.md | Derivação técnica | DELETE em cascata apaga histórico; pausar via PATCH active | TRANSCRICAO | [09:21] Bruno |

## ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Outbox transacional no MySQL existente | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Alternativa descartada | HTTP síncrono | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Alternativa descartada | Redis Streams | TRANSCRICAO | [09:07] Larissa |
| ADR-001-TO | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Trade-off | Latência de polling em troca de atomicidade e zero infraestrutura nova | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 retentativas 1m/5m/30m/2h/12h e DLQ separada | TRANSCRICAO | [09:17] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-002-ALT-03 | docs/adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | DLQ como status na outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-002-TO | docs/adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Até 15 h de atraso aceitável | TRANSCRICAO | [09:17] Marcos |
| ADR-003 | docs/adrs/ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint e rotação de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-003-ALT-01 | docs/adrs/ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-003-TLS | docs/adrs/ADR-003-assinatura-hmac-sha256-com-secret-por-endpoint.md | Restrição | https obrigatório via Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004 | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-004-ALT-01 | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-004-TO | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar no cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005 | docs/adrs/ADR-005-worker-separado-com-polling.md | Decisão | Worker separado com polling de 2 s e instância única | TRANSCRICAO | [09:10] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-worker-separado-com-polling.md | Alternativa descartada | Trigger ou LISTEN/NOTIFY | TRANSCRICAO | [09:09] Bruno |
| ADR-005-ALT-02 | docs/adrs/ADR-005-worker-separado-com-polling.md | Alternativa descartada | Worker no processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT-03 | docs/adrs/ADR-005-worker-separado-com-polling.md | Alternativa adiada | Múltiplos workers | TRANSCRICAO | [09:13] Bruno |
| ADR-005-COD | docs/adrs/ADR-005-worker-separado-com-polling.md | Referência de código | Entry-point existente usado como modelo | CODIGO | src/server.ts |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo de AppError, Pino, middleware, módulos e Zod | TRANSCRICAO | [09:30] Larissa |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Estrutura de módulo controller/service/repository/routes/schemas | CODIGO | src/modules/orders/order.routes.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | Erros de domínio `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa descartada | Stack ou biblioteca nova para o subsistema | TRANSCRICAO | [09:29] Bruno |
| ADR-006-TO | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Trade-off | JWT é do operador, sem posse de customer | TRANSCRICAO | [09:32] Bruno |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md | Decisão | Snapshot na inserção, payload enxuto, filtro na inserção | TRANSCRICAO | [09:52] Bruno |
| ADR-007-ALT-01 | docs/adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md | Alternativa descartada | Filtrar no envio | TRANSCRICAO | [09:34] Diego |
| ADR-007-ALT-02 | docs/adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md | Alternativa descartada | Renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT-04 | docs/adrs/ADR-007-payload-snapshot-enxuto-e-filtro-na-insercao.md | Alternativa descartada | Truncar payload grande | TRANSCRICAO | [09:23] Sofia |
| ADR-008 | docs/adrs/ADR-008-publicacao-via-funcao-transacional-no-changestatus.md | Decisão | `publishWebhookEvent(tx, …)` dentro de `changeStatus` | TRANSCRICAO | [09:41] Bruno |
| ADR-008-ALT-01 | docs/adrs/ADR-008-publicacao-via-funcao-transacional-no-changestatus.md | Alternativa descartada | Injetar repository no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-008-ALT-02 | docs/adrs/ADR-008-publicacao-via-funcao-transacional-no-changestatus.md | Alternativa descartada | Publicar após o commit | TRANSCRICAO | [09:41] Diego |
| ADR-008-COD | docs/adrs/ADR-008-publicacao-via-funcao-transacional-no-changestatus.md | Referência de código | Helpers com tx já existentes (debitStock, reserveOrderNumber) | CODIGO | src/modules/orders/order.service.ts |
