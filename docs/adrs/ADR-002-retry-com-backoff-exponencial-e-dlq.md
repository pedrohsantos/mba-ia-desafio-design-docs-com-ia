# ADR-002 — Retry com backoff exponencial (5 retentativas) e DLQ em tabela separada

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:14–09:19
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **Relacionados:** [ADR-004](ADR-004-entrega-at-least-once-com-x-event-id.md), [ADR-005](ADR-005-worker-separado-com-polling.md), [FDD §4.3–4.4](../FDD.md#43-fluxo-3--retry-com-backoff)

## Contexto

O endpoint do cliente pode estar lento, fora do ar ou em manutenção. Já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). Precisamos de uma política que:

- não desista cedo demais de um cliente temporariamente indisponível;
- não deixe eventos "pendurados para sempre" quando o cliente sumiu ([09:15] Diego);
- preserve evidência do que falhou para debug e reprocessamento ([09:18] Diego).

## Decisão

1. **Retry com backoff exponencial** com a progressão fixa **1 min → 5 min → 30 min → 2 h → 12 h** ([09:17] Diego, confirmado em [09:17] Larissa).
2. Considera-se falha de tentativa: timeout de 10 s ([09:42] Diego), erro de rede ou resposta HTTP fora da faixa 2xx.
3. Esgotadas as retentativas, o evento é marcado como `FAILED` na outbox e uma linha é criada na tabela separada **`webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego).
4. O reprocessamento é **manual**, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige role `ADMIN` e registra quem fez o replay ([09:36] Sofia).

### Nota de interpretação: "5 tentativas"

A reunião usa "5 tentativas" ([09:15] Diego, [09:48] Larissa) e, ao mesmo tempo, lista **cinco intervalos** de espera totalizando "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). Os dois enunciados só são compatíveis se os 5 intervalos separarem **5 retentativas após o envio inicial**:

| Envio | Quando ocorre (relativo à 1ª falha) |
|---|---|
| Inicial | t0 (primeiro ciclo de polling após o commit) |
| Retentativa 1 | t0 + 1 min |
| Retentativa 2 | t0 + 6 min |
| Retentativa 3 | t0 + 36 min |
| Retentativa 4 | t0 + 2 h 36 min |
| Retentativa 5 | t0 + 14 h 36 min (≈ "quase 15 horas") |

Portanto: **1 envio inicial + 5 retentativas = até 6 requisições por evento**. Essa leitura está aberta à confirmação na sessão de revisão do design doc ([09:50] Larissa) e registrada como questão em aberto no [RFC](../RFC.md#7-questões-em-aberto).

## Alternativas consideradas

### A1. Retry indefinido com backoff
- **Prós:** nunca desiste; cliente que volta depois de dias recebe tudo.
- **Contras:** evento fica pendurado para sempre se o cliente sumiu; a outbox acumula lixo operacional.
- **Motivo do descarte:** "traz o problema de evento ficar pendurado pra sempre" ([09:15] Diego).

### A2. Apenas 3 tentativas (mais agressivo)
- **Prós:** falha rápida, DLQ mais rápida.
- **Contras:** três tentativas em ~30 min matariam o evento durante uma manutenção planejada comum.
- **Motivo do descarte:** "3 é pouco" ([09:16] Diego); 5 aceito por Larissa ([09:16]).

### A3. DLQ como status `failed` na própria outbox
- **Prós:** uma tabela a menos.
- **Contras:** mistura eventos operacionais com evidência de falha; leitura da outbox fica mais suja.
- **Motivo do descarte:** tabela separada é "mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento" ([09:18] Diego). A outbox ainda marca `FAILED` para o histórico de entregas, mas a evidência e o replay partem da DLQ.

## Consequências

### Positivas
- Cobre indisponibilidades de até ~15 h sem intervenção humana.
- Teto de tentativas evita eventos eternos na outbox.
- DLQ dá rastreabilidade completa e ponto único de reprocessamento.

### Negativas
- Um evento pode chegar com até ~15 h de atraso; para o PM isso é aceitável porque "se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).
- Replay é manual: depende de um ADMIN agir.
- Retentativas quebram a ordem de entrega entre eventos de um mesmo pedido (ver [ADR-005](ADR-005-worker-separado-com-polling.md) e questão em aberto no RFC).
- Sem notificação proativa ao cliente sobre falhas: e-mail ficou para fase futura ([09:37] Larissa).

### Trade-off explícito
Escolhemos **teto fixo de tentativas com janela longa** em vez de persistência infinita ou falha rápida: aceitamos atraso de até ~15 h e trabalho manual de replay em troca de uma outbox que não acumula eventos sem destino.
