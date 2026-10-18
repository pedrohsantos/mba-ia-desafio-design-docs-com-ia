# ADR-003 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 h

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:19–09:23
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Bruno, Diego
- **Relacionados:** [FDD §5.7 e §5.8](../FDD.md#58-headers-e-assinatura-do-envio-ao-cliente), [RFC](../RFC.md)

## Contexto

Os webhooks enviam dados de pedidos para endpoints **fora da nossa infraestrutura**. O cliente precisa conseguir verificar que a requisição veio realmente de nós e que o payload não foi adulterado no caminho ([09:19] Sofia). Já houve caso de cliente que vazou secret em log da própria aplicação ([09:22] Diego), então a solução precisa limitar o raio de impacto de um vazamento e permitir troca da secret sem downtime.

## Decisão

1. Cada envio é assinado com **HMAC-SHA256 sobre o corpo bruto do request** ([09:20] Sofia, [09:22] Sofia), e a assinatura vai no header **`X-Signature`**.
2. **Uma secret única por endpoint de webhook**, nunca uma secret global da plataforma ([09:21] Sofia).
3. A secret é **gerada pelo servidor** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
4. A secret é **rotacionável via API** pelo próprio cliente. Na rotação, a secret anterior continua válida **por 24 horas em paralelo**; depois disso deixa de ser usada ([09:21] Sofia).
5. A tabela de configuração guarda `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno).
6. **TLS obrigatório:** a URL precisa ser `https`; `http` é recusado com erro de validação no schema Zod ([09:23] Sofia). Por ser uma regra de validação, não é tratada como decisão arquitetural separada.

### Como a "validade em paralelo" se traduz no envio

Como nós somos o **emissor**, "a antiga continua válida" significa que, durante o grace period, o cliente ainda precisa conseguir validar com a secret antiga. A proposta do FDD é enviar as duas assinaturas no mesmo header (`X-Signature: sha256=<nova>,sha256=<antiga>`) enquanto `previous_secret_expires_at` não tiver passado. O formato exato do header fica sujeito à revisão de segurança da Sofia, que pediu pelo menos dois dias úteis para revisar HMAC e geração de secret antes do deploy ([09:46] Sofia).

## Alternativas consideradas

### A1. Secret global da plataforma
- **Prós:** operação trivial; uma única chave para gerenciar.
- **Contras:** um vazamento compromete todos os clientes.
- **Motivo do descarte:** "Senão se vaza uma, vaza tudo" ([09:21] Sofia).

### A2. Rotação com corte imediato (sem grace period)
- **Prós:** a secret vazada deixa de valer na hora.
- **Contras:** o cliente teria downtime de validação entre pedir a nova secret e implantá-la em todos os sistemas dele.
- **Motivo do descarte:** o grace period de 24 h existe "pra ele ter tempo de migrar os sistemas dele" ([09:21] Sofia).

## Consequências

### Positivas
- Autenticidade e integridade verificáveis pelo cliente com biblioteca padrão; "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
- Raio de impacto de vazamento limitado a um endpoint.
- Rotação sem downtime para o cliente.

### Negativas
- A secret precisa ser armazenada de forma recuperável (HMAC exige a secret original, não um hash dela), o que aumenta a sensibilidade da tabela `webhooks`.
- A assinatura cobre apenas o corpo: o header `X-Timestamp` **não** é assinado. A proteção contra replay depende do campo `timestamp` dentro do payload, que é assinado. Isso fica registrado para a revisão de segurança.
- Durante as 24 h de grace period, uma secret vazada continua válida.
- Mais complexidade no worker (duas assinaturas durante a rotação) e no cliente (aceitar qualquer uma das assinaturas).

### Trade-off explícito
Aceitamos **mais complexidade operacional** (secret por endpoint, rotação e dupla assinatura temporária) em troca de **isolamento de vazamento e rotação sem downtime**. Durante as 24 h de grace period, priorizamos a continuidade do cliente em vez da revogação imediata.
