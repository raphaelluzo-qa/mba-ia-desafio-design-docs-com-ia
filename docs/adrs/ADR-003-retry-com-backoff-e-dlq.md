# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e Dead Letter Queue em tabela separada

## Status

**Parcialmente aceito — calendário de backoff e DLQ decididos; contagem total de chamadas pendente (RFC-OQ-08)**

- **Data da decisão:** reunião técnica de quinta-feira, 09:15–09:19 e 09:36 (data civil não informada na transcrição)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Sofia (Segurança, quanto à autorização do replay)
- **Consultados:** Bruno (Eng. Pleno), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar lento, fora do ar ou em manutenção. Um cliente já ficou duas horas indisponível numa manutenção planejada ([09:16] Diego). Precisávamos definir:

- quantas vezes tentar de novo e com que intervalo;
- o que fazer quando as tentativas se esgotam;
- como recuperar os eventos que falharam de vez.

## Decisão

### Retry

- **Backoff com progressão fixa: 1 min, 5 min, 30 min, 2 h, 12 h** ([09:17] Diego, [09:17] Larissa).
- A reunião decidiu “5 tentativas” e listou **5 intervalos** (1m/5m/30m/2h/12h), mas não esclareceu se a contagem inclui o envio inicial. A fala sobre “três vezes em 30 minutos” sugere uma leitura, mas não fecha a regra de contagem. **Quantidade total de chamadas, semântica de `attempts` e momento de DLQ permanecem em aberto** ([09:16] Diego, [09:17] Diego, [09:17] Larissa; RFC-OQ-08).
- **Não adotar ainda** um limite de 5 retries nem de 5 chamadas totais. A conta de ~14,6 h é a soma dos cinco delays, não uma decisão sobre quantas chamadas são executadas.
- **Condições de falha que disparam retry:** timeout de 10 s, decidido na reunião ([09:42] Diego). Cliente offline ([09:14] Larissa), ou seja, erro de rede, também dispara retry. Tratar resposta HTTP fora da faixa 2xx como falha é **detalhamento proposto no FDD**, não discutido explicitamente.

### Dead Letter Queue (DLQ)

- Esgotado o teto, o evento é considerado **falha permanente** e vai para a DLQ ([09:15] Diego).
- A DLQ é uma **tabela separada, `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego). A linha correspondente na outbox fica com status `FAILED` e sai do caminho de leitura do worker.
- O **reprocessamento é manual**, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
- O replay exige **role `ADMIN`**, reaproveitando o `requireRole` de `src/middlewares/auth.middleware.ts` ([09:36] Sofia, [09:36] Larissa).
- **Todo replay é logado com o usuário que o executou**, para auditoria ([09:36] Sofia).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento pode ficar pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Cobre só ~30 min. Uma indisponibilidade de uma manhã ou uma manutenção de 2 h mataria o evento ([09:16] Bruno propôs, [09:16] Diego descartou). |
| **Marcar como `failed` na própria outbox, sem tabela de DLQ** | Mistura evidência de falha com a fila ativa. A tabela separada deixa a leitura da outbox mais limpa e guarda a evidência para debug e reprocessamento ([09:17] Larissa levantou, [09:18] Diego decidiu). |
| **Replay por qualquer usuário autenticado** | Mexer na fila de entrega de notificações não é atribuição de operador ([09:36] Sofia). |

## Consequências

**Positivas**
- Os cinco delays somam ~14,6 h se todos forem usados; isso descreve a soma dos intervalos, não uma janela aprovada. A janela efetiva de retry depende da contagem a decidir (RFC-OQ-08).
- Nenhum evento é descartado em silêncio: ou foi entregue, ou está na DLQ com motivo.
- O reprocessamento é controlado e auditável.

**Negativas**
- O ponto em que o evento vai à DLQ e passa a depender de ação de ADMIN só pode ser definido após resolver RFC-OQ-08. Não há aviso automático ao cliente: notificação por e-mail ficou para fase futura ([09:37] Larissa).
- Se todos os cinco delays forem usados, o atraso entre tentativas pode somar ~14,6 h; o atraso máximo efetivo depende da contagem e política a decidir (RFC-OQ-08).
- Um replay reenvia o mesmo `event_id`, então o cliente pode receber duplicata. Isso é consistente com [ADR-005](ADR-005-at-least-once-com-x-event-id.md).
- A progressão fixa não se adapta a padrões de falha diferentes (por exemplo, respeitar `Retry-After`).

**Trade-off explícito:** foi decidido usar uma política finita de retries seguida por recuperação manual via ADMIN, mas o limite numérico exato de chamadas ainda não está fechado. Não fixar essa consequência até resolver RFC-OQ-08.
