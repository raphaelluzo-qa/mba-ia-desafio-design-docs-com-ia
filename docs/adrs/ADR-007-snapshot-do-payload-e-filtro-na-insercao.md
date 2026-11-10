# ADR-007 — Payload enxuto renderizado como snapshot e filtro de eventos na inserção da outbox

## Status

**Aceito**

- **Data da decisão:** reunião técnica de quinta-feira, 09:33–09:34, 09:43 e 09:51–09:52 (data civil não informada na transcrição)
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma)
- **Consultados:** Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

Três decisões secundárias moldam **o que** vai para a outbox e **quando**:

1. **Filtro de eventos.** Cada webhook escolhe quais status quer receber ("só quero saber quando vira SHIPPED e DELIVERED") ([09:33] Marcos). O filtro poderia ser aplicado ao inserir na outbox ou na hora de enviar ([09:34] Diego).
2. **Momento de renderização.** A linha da outbox pode guardar o payload pronto ou só o `order_id`, renderizando na hora do envio ([09:51] Bruno). Se todos os cinco delays discutidos forem usados, o intervalo entre tentativas pode somar ~14,6 h (a contagem continua pendente em [RFC-OQ-08](../RFC.md#5-questões-em-aberto)); o pedido pode mudar ou até ser removido nesse meio tempo: `OrderService.delete` permite apagar pedidos `PENDING` ou `CANCELLED` (`src/modules/orders/order.service.ts`).
3. **Conteúdo.** O modelo `Order` (`prisma/schema.prisma`) tem itens, histórico e cliente. Mandar tudo infla o payload, e o limite é 64 KB ([09:24] Diego).

## Decisão

1. **Filtrar na inserção.** Em `publishWebhookEvent`, só entram na outbox os webhooks **ativos** do customer do pedido cujo filtro contém o `to_status`. Se nenhum webhook se interessa por aquele status, **nenhuma linha é inserida** ([09:34] Bruno, [09:34] Diego).
2. **Snapshot na inserção.** O payload é renderizado e serializado **dentro da transação de `changeStatus`** e guardado pronto na outbox. Ele reflete o estado do pedido no momento da mudança de status ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
3. **Payload enxuto** ([09:43] Diego, [09:44] Bruno):
   - `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601);
   - `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`;
   - campos básicos do pedido, como `total_cents`;
   - **sem `items`**. Quem quiser detalhes consulta `GET /orders/:id`.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Filtrar na hora do envio (worker)** | Grava linhas que nunca serão enviadas e ocupa a tabela e o polling à toa ([09:34] Bruno: "Economiza linha na tabela"). |
| **Guardar só `order_id` e renderizar no envio** | O evento passaria a refletir o estado do pedido no momento do envio, não o da mudança. Isso gera "caso esquisito" em retries e em pedidos alterados ou removidos depois ([09:52] Larissa). |
| **Payload completo com `items`** | Infla o payload sem necessidade. O cliente tem `GET /orders/:id` para detalhes ([09:43] Diego). |

## Consequências

**Positivas**
- A outbox só guarda trabalho real.
- O evento é imutável e fiel ao momento da mudança, inclusive em retry e replay horas depois.
- A assinatura HMAC é calculada sobre bytes estáveis: o mesmo corpo em todas as tentativas.
- Com o payload pequeno, o limite de 64 KB fica longe de ser atingido ([09:24] Diego).

**Negativas**
- **Uma alteração no filtro não afeta eventos já enfileirados.** Se o cliente remove `SHIPPED` do filtro, eventos `SHIPPED` já na outbox ainda são enviados. E se ele adiciona `SHIPPED`, não há retroativo.
- **Um webhook criado depois da mudança de status não recebe aquele evento.**
- A transação de `changeStatus` passa a fazer uma leitura extra (webhooks ativos do customer) e a serialização do payload.
- O payload fica duplicado por webhook quando o customer tem vários endpoints inscritos, porque há uma linha da outbox por endpoint.
- Clientes que precisam dos itens fazem uma chamada extra a `GET /orders/:id`.

**Trade-off explícito:** trocamos flexibilidade (filtro e conteúdo avaliados no envio) por fidelidade ao momento do evento e por uma outbox menor. Mudanças de configuração valem só para eventos futuros.
