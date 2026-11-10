# ADR-004 — Autenticação das entregas com HMAC-SHA256, secret por endpoint e rotação com grace period de 24h

## Status

**Aceito (sujeito à revisão de segurança antes do deploy)**

- **Data da decisão:** reunião técnica de quinta-feira, 09:19–09:23 (data civil não informada na transcrição)
- **Decisores:** Sofia (Engenharia de Segurança), Larissa (Tech Lead)
- **Consultados:** Bruno (Eng. Pleno), Diego (Eng. Sênior)
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Os webhooks são **outbound**: saem da nossa plataforma para endpoints dos clientes, fora da nossa infraestrutura. Não recebemos webhooks deles ([09:02] Marcos, [09:03] Sofia).

As requisições levam dados de pedidos. O cliente precisa conseguir verificar duas coisas: que a requisição **veio de nós** e que o **payload não foi adulterado** no caminho ([09:19] Sofia).

Existe histórico real de vazamento: um cliente já expôs uma secret em log da própria aplicação ([09:22] Diego).

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature` ([09:20] Sofia, [09:22] Sofia). É o padrão de mercado e tem biblioteca em qualquer stack de cliente ([09:20] Sofia).
2. **Uma secret única por endpoint de webhook**, nunca uma secret global da plataforma ([09:21] Sofia).
3. **Secret gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos).
4. **A configuração do webhook armazena** URL, secret, `customer_id` e estado ativo ([09:21] Bruno, confirmado por [09:21] Sofia).
5. **Rotação via API.** O cliente pede uma nova secret. A antiga continua válida **em paralelo por 24 horas** e depois é descartada ([09:21] Sofia). **Detalhamento proposto (não discutido na reunião):** para que "a antiga fica válida" tenha efeito do lado do cliente, que é quem verifica, a entrega leva durante o grace period a assinatura com as duas secrets. O formato fica no FDD e é ponto da revisão de segurança (RFC-OQ-07).
6. **TLS obrigatório:** a URL precisa ser `https`. Uma URL `http` é recusada com erro de validação no schema Zod. Por ser só validação, não é uma decisão arquitetural à parte ([09:23] Sofia).
7. **Revisão de segurança obrigatória:** pelo menos 2 dias úteis de revisão da Sofia sobre HMAC e geração de secret antes do deploy ([09:46] Sofia, [09:47] Larissa).

**Limite da decisão:** exigir `https` define o esquema, mas não define se destinos privados/internos ou resoluções DNS para esses destinos são permitidos. A política de destino não foi discutida e permanece pendente para revisão de segurança (RFC-OQ-11); não considerar a validação de esquema como mitigação completa de SSRF.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Se vaza uma, vaza tudo ([09:21] Sofia). |
| **Secret sem rotação (trocar exige recriar o webhook)** | Com vazamento já ocorrido, o cliente precisa trocar a secret sem perder entregas. O grace period de 24h dá tempo de migrar ([09:21] Sofia, [09:22] Diego). |
| **Aceitar URLs `http`** | Os dados do pedido e a assinatura trafegariam sem confidencialidade ([09:23] Sofia). |
| **Outros algoritmos de HMAC** | SHA-256 é o padrão de mercado com suporte universal nos clientes ([09:20] Sofia). Não houve discussão de alternativa concreta. |

## Consequências

**Positivas**
- O cliente verifica origem e integridade com uma primitiva padrão.
- O vazamento de uma secret fica contido em um único endpoint.
- A rotação acontece sem janela de indisponibilidade para o cliente.
- A geração da secret usa `crypto` nativo do Node, sem nova dependência.

**Negativas**
- **A secret precisa ser armazenada de forma recuperável**, porque o HMAC exige o valor original para assinar (não dá para guardar só um hash como no `passwordHash` de `users`). A proteção em repouso não foi discutida e fica para a revisão de segurança.
- A secret não pode aparecer em logs. É preciso incluí-la no `redact` do Pino (`src/shared/logger/index.ts`).
- Durante o grace period há duas assinaturas por entrega, o que é um pouco mais de CPU e um header maior.
- O HMAC cobre o corpo, não os headers. Um cliente que queira detectar replay attack usa `X-Timestamp` ([09:44] Diego), mas essa verificação é opcional e fica do lado dele.

**Trade-off explícito:** trocamos a simplicidade de uma secret única pelo custo de gerenciar uma secret por endpoint, com rotação, armazenamento recuperável e revisão de segurança dedicada. Em troca, o raio de impacto de um vazamento fica limitado a um endpoint.
