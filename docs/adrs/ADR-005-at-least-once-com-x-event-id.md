# ADR-005 — Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

## Status

**Aceito**

- **Data da decisão:** reunião técnica de quinta-feira, 09:24–09:26 (data civil não informada na transcrição)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma)
- **Consultados:** Bruno (Eng. Pleno), Sofia (Segurança), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

A combinação de outbox, worker e retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) cria cenários de envio duplicado:

- o cliente processou a requisição, mas a resposta não chegou antes do timeout de 10 s;
- o worker caiu depois do POST e antes de marcar o evento como entregue;
- um ADMIN fez replay de um evento da DLQ que o cliente, na prática, já tinha recebido.

Evitar todos esses casos exigiria coordenação transacional entre nós e o cliente.

## Decisão

1. **A garantia é at-least-once.** O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso ([09:24] Diego).
2. **Cada evento tem um `event_id` (UUID) gerado no momento em que entra na outbox** e enviado no header **`X-Event-Id`**. O cliente deduplica por esse valor ([09:25] Diego).
3. **O `event_id` é estável:** o mesmo valor vai em todas as retentativas e em replays da DLQ, senão a deduplicação não funciona.
4. **O mesmo `event_id` também vai no corpo** (`event_id`), junto com o restante do payload ([09:43] Diego).
5. **Quando um mesmo customer tem vários webhooks inscritos**, cada endpoint recebe o evento com o mesmo `event_id`. O header **`X-Webhook-Id`** identifica qual cadastro gerou aquele envio ([09:44] Sofia, [09:45] Diego).
6. **O PM documenta a responsabilidade de deduplicação com destaque** no portal do desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exigiria coordenação dos dois lados e é muito mais complexo ([09:25] Diego). |
| **At-most-once (sem retry)** | Incompatível com a política de retry e DLQ ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)). O cliente perderia eventos em qualquer instabilidade. Plausível, mas não defendida por ninguém na reunião. |
| **Não enviar identificador de evento** | Deixaria o cliente sem como deduplicar. A pergunta "como ele diferencia?" levou direto à decisão do `X-Event-Id` ([09:25] Bruno). |

## Consequências

**Positivas**
- O padrão é conhecido do mercado (Stripe e GitHub fazem assim) e resolve a grande maioria dos casos ([09:25] Diego).
- A implementação do nosso lado fica simples: nenhuma confirmação em duas fases.
- Retry e replay podem ser agressivos sem medo de "entregar demais".

**Negativas**
- **Transfere responsabilidade para o cliente** ([09:25] Sofia). Um cliente que não deduplica pode processar a mesma mudança de status duas vezes.
- Exige comunicação clara no portal do desenvolvedor, uma dependência do PM ([09:26] Marcos).
- Um cliente com vários webhooks inscritos recebe o mesmo `event_id` em cada endpoint e deve deduplicar considerando também o `X-Webhook-Id`.

**Trade-off explícito:** trocamos a garantia forte (exactly-once), cara e complexa, por uma garantia mais fraca e padrão de mercado. O custo da deduplicação passa para o cliente.
