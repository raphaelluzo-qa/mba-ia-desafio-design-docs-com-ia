# Da Reunião ao Documento — Design Docs do Sistema de Webhooks

> Enunciado original do desafio: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia#readme)

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript, Express, Prisma e MySQL) decidiu numa reunião de ~55 minutos construir **webhooks de notificação de mudança de status de pedidos** para três clientes B2B. Tudo o que ficou registrado dessa reunião é a transcrição literal ([`TRANSCRICAO.md`](TRANSCRICAO.md)). A tarefa foi transformar essa conversa, junto com o código existente, num pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

O desafio real não era "escrever documentos", e sim **filtrar e posicionar informação**. Na transcrição aparecem decisões fechadas, requisitos, alternativas descartadas, pontos adiados, detalhes secundários e ambiguidades. Cada item precisava ir para o documento certo (PRD = porquê/o quê; RFC = proposta e o que está em aberto; ADR = decisão isolada; FDD = como construir) e ter origem comprovável: um timestamp da transcrição ou um arquivo real do repositório. O código da aplicação não foi alterado.

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
|---|---|
| **Claude Code (app desktop), modelo Claude Opus 5** | Ferramenta principal. Leu o repositório inteiro (`src/`, `prisma/`, `tests/`, `package.json`) e a transcrição; fez a triagem do que entra e do que não entra; redigiu ADRs, RFC, FDD, PRD e Tracker; fez o fork e o versionamento com `gh`/`git` |
| **Script de validação em Python (gerado pela IA, executado localmente)** | "Auditor" anti-alucinação. Confere se todo `[hh:mm] Nome` citado existe na transcrição com aquele falante, se todo caminho de código citado existe, se todo ID dos documentos tem linha no tracker (e vice-versa) e calcula as proporções `TRANSCRICAO`/`CODIGO` exigidas |

## Workflow adotado

A ordem seguiu a sugestão do enunciado: decisões primeiro, consolidação de produto no fim.

```
1. Contextualização ─► 2. Triagem ─► 3. ADRs ─► 4. RFC ─► 5. FDD ─► 6. PRD ─► 7. Tracker ─► 8. Validação ─► 9. Revisão crítica ─► 10. README
      (código +          (entra/não      (7)       (proposta,   (contratos,  (consolidação)  (404 linhas)  (script)       (contra transcrição
       transcrição)       entra/ambíguo)            alternativas, integração)                                          e código)
                                                    em aberto)
```

1. **Contextualização.** Leitura integral da transcrição e dos arquivos que a feature toca: `order.service.ts` (transação de `changeStatus`), `order.status.ts` (máquina de estados), `shared/errors/*`, middlewares (`auth`, `validate`, `error`, `request-logger`), `logger`, `config/*`, `app.ts`, `routes/index.ts`, `schema.prisma`, `package.json`, `tests/setup.ts` e as factories.
2. **Triagem dirigida da transcrição** em cinco baldes: decisão fechada, requisito, descartado, adiado/aberto e ambíguo. Cada item levou o timestamp e o falante. Esse inventário virou a espinha de todos os documentos.
3. **ADRs primeiro.** As 6 decisões principais viraram 6 ADRs. Uma sétima (snapshot do payload + filtro na inserção) foi promovida a ADR porque teve alternativas reais discutidas ([09:34] e [09:51]).
4. **RFC** conciso, referenciando os ADRs, com as alternativas e questões em aberto que a reunião deixou.
5. **FDD** profundo: modelo Prisma, 8 contratos, fluxos, matriz `WEBHOOK_*`, observabilidade e a seção de integração com 20 arquivos reais.
6. **PRD** por último, como consolidação de produto, sem detalhes de implementação.
7. **Tracker** montado varrendo os IDs de cada documento.
8. **Validação automatizada** com o script.
9. **Revisão crítica** manual contra a transcrição e o código, procurando contradições que o script não pega.
10. **README** do processo.

**Organização da interação com a IA:** um único agente com acesso ao repositório, trabalhando por etapas com um critério de saída em cada uma (ex.: "todo item da triagem tem timestamp"). Todo documento traz IDs estáveis (`PRD-FR-01`, `RFC-OQ-03`, `FDD-INT-07`…) para o tracker ser verificável por script, e não por inspeção visual.

## Prompts customizados

O prompt de abertura é o que dei ao agente. Os prompts 1 a 3 são as instruções dirigidas de cada etapa, formalizadas aqui em versão reutilizável.

**Prompt 0: abertura da sessão (Claude Code).**

```text
O texto abaixo descreve o desafio que precisamos resolver. Vamos implementar os design docs
conforme a orientação do desafio; o objetivo é cumprir todos os critérios de aceite:
<enunciado completo do desafio colado aqui, incluindo requisitos, critérios de aceite,
 estrutura obrigatória e ordem de execução sugerida>
```

**Prompt 1: triagem dirigida da transcrição.** Evita o "resuma a reunião" genérico e força a separação entre o que entra, o que não entra e o que é ambíguo.

```text
Leia TRANSCRICAO.md inteira. NÃO resuma. Produza um inventário em tabela com as colunas:
[hh:mm] | Falante | Item | Balde | Evidência literal (≤ 15 palavras)

Baldes permitidos (exatamente um por item):
- DECISAO_FECHADA: alguém com autoridade encerrou ("Decidido", "Anotado", "Tá decidido", resumo final da Larissa)
- REQUISITO: necessidade funcional/não funcional pedida por PM, Segurança ou Tech Lead
- DESCARTADO: alternativa colocada na mesa e rejeitada — registre o trade-off que motivou o descarte
- ADIADO_OU_ABERTO: "fica pra próxima fase", "observar", "problema do futuro", "mais pra frente"
- AMBIGUO: fala que admite duas leituras ou números que não fecham — explique as leituras possíveis

Regras:
1. Se não houver timestamp e falante, o item não existe.
2. Uma pergunta sem resposta não é requisito.
3. Marque separadamente decisões secundárias (payload, headers, timeout, UUID) — elas
   podem ir para o FDD em vez de virar ADR.
4. Liste no final tudo que o time explicitamente NÃO vai fazer nesta fase.
5. Cruze com o código: para cada item que toca a aplicação, cite o arquivo real
   (ex.: src/modules/orders/order.service.ts) ou escreva "arquivo novo proposto".
```

**Prompt 2: geração de ADR com restrição de origem.** Aplicado a cada uma das 7 decisões.

```text
Escreva docs/adrs/ADR-{NNN}-{kebab}.md em formato MADR, em português, para a decisão
"{decisão}" usando SOMENTE os itens do inventário com os timestamps {lista}.

Seções obrigatórias, nesta ordem: Status, Contexto, Decisão, Alternativas Consideradas,
Consequências (Positivas, Negativas e uma linha "Trade-off explícito").

Restrições:
- Toda afirmação de decisão termina com a origem entre parênteses: ([hh:mm] Nome).
- Alternativas: só as discutidas na reunião; se incluir uma "plausível" não discutida,
  diga isso explicitamente na célula.
- No Contexto, referencie arquivos/funções reais que a decisão afeta (leia-os antes;
  não cite arquivo que você não abriu).
- Se precisar completar uma lacuna técnica (algo que a reunião não disse), escreva
  "Detalhamento proposto (não discutido na reunião)" e não apresente como decisão.
- Não repita detalhes de implementação (schema, payload JSON, pseudocódigo): isso é do FDD.
```

**Prompt 3: revisão adversarial contra a transcrição.** Aplicado depois da primeira versão completa; gerou os ajustes A1–A3.

```text
Aja como revisor cético do pacote docs/. Para cada requisito, decisão, restrição e
contrato de API:
1. Localize a fala citada na TRANSCRICAO.md e confirme que ela diz ISSO (não algo parecido).
2. Procure contradições entre documentos (PRD × RFC × FDD × ADR) e com o código real
   (ex.: o middleware validate() sempre responde VALIDATION_ERROR? NotFoundError aceita código?).
3. Liste itens que apareceram como decisão mas são inferência sua.
4. Liste itens que a reunião descartou/adiou e que vazaram como requisito.
5. Verifique se a quebra de sprints e a ordem das entregas batem com [09:46].
Saída: tabela Problema | Documento/ID | Evidência | Correção proposta. Não corrija ainda.
```

## Iterações e ajustes

O processo de produção teve **4 iterações principais** (triagem → primeira versão completa → validação por script → revisão adversarial) e um checklist final contra os critérios de aceite. Uma revisão documental posterior acrescentou os esclarecimentos descritos em C.

### A. Erros que chegaram a ser escritos nos rascunhos e foram corrigidos

| # | O que estava no rascunho | Problema | Correção |
|---|---|---|---|
| A1 | Listagem de webhooks como `GET /webhooks?customerId=…` (FDD) | [09:32] Larissa diz que o `customer_id` vem "no body ou no path". Query string não é nenhum dos dois | Listagem movida para `GET /customers/:customerId/webhooks`, com a montagem de router ajustada em `FDD-INT-15` e a linha do tracker atualizada |
| A2 | Plano de implementação com "integração no `order.service`" na sprint 1 (FDD §15) | [09:46] Larissa coloca integração + testes ponta a ponta na **meia sprint** depois de CRUD/deliveries | Tabela `FDD-PLAN-*` reordenada conforme a fala |
| A3 | "Resposta não-2xx = falha" (ADR-003) e "assinatura dupla no grace period" (ADR-004) escritos como decisões | Nenhum dos dois foi dito na reunião: são inferências necessárias para implementar | Marcados como **detalhamento proposto**; o formato do `X-Signature` virou item da revisão de segurança (`RFC-OQ-07`) |
| A4 | Tracker com 404 linhas dado como "completo" | O script encontrou **37 IDs no tracker sem âncora nos documentos** (ex.: `RFC-PROP-*`, `FDD-PLAN-*`) e uma fonte fraca (`FDD-RES-10` apontava para [09:07] Diego, fala sobre tamanho do time, não sobre resiliência) | IDs adicionados nos documentos e a fonte corrigida para [09:11] Diego. O script terminou sem erros nos checks que implementa; ele não calcula a cobertura dos itens identificáveis exigida pelo desafio |
| A5 | ADRs com `Status` como bullet de metadado | O critério pede a **seção** Status | `## Status` promovido a seção em todos os 7 ADRs |

### B. Leituras iniciais descartadas na análise, antes de redigir

| # | Primeira leitura | Por que estava errada | Como ficou |
|---|---|---|---|
| B1 | A primeira hipótese tratou "5 tentativas" como chamada inicial + 4 retentativas | A reunião também lista cinco intervalos e fala em "quase 15 horas", mas não define explicitamente se a chamada inicial entra na contagem | A documentação mantém a interpretação em aberto: registra os cinco delays, mas deixa total de chamadas e limite da DLQ pendentes em [RFC-OQ-08](docs/RFC.md#5-questões-em-aberto) e [ADR-003](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| B2 | Erro `WebhookInvalidUrlError` lançado pelo service | Contradiz Sofia ([09:23]: "é só uma validação no schema Zod"). E, lendo `validate.middleware.ts`, todo erro Zod sai como `VALIDATION_ERROR` | Validação no Zod, sem alterar o middleware, com o código `WEBHOOK_INVALID_URL` em `details[].message`. A limitação ficou registrada no [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| B3 | Limite de 64 KB checado na inserção da outbox | Um erro ali faria **rollback da mudança de status**, contrariando o motivo de não fazer nada síncrono ([09:04] Bruno) | Checagem no worker: payload grande vai direto à DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry |
| B4 | Filtro de eventos aceitando qualquer `OrderStatus` | Pelo código, nenhuma transição em `order.status.ts` leva a `PENDING`, e `OrderService.create` grava o `PENDING` sem passar por `changeStatus`. Assinar `PENDING` nunca dispararia | `SUBSCRIBABLE_STATUSES` derivado de `canTransition`; criação do pedido listada como exclusão (`FDD-EXC-07`) |
| B5 | Manter `docs/adrs/README.md` do boilerplate | Ele pede nomes `0001-titulo.md`, conflitando com o formato exigido `ADR-NNN-titulo.md`, e entraria na contagem de 5 a 8 arquivos | Removido; o índice de ADRs fica na seção 7 do RFC |

### C. Revisão documental posterior

Uma revisão adicional identificou pressupostos não explicitados sobre latência, autorização por customer, destinos URL e exclusão/retenção. Polling e autenticação por role não definem por si só capacidade sob carga, ownership entre usuário e customer, política contra destinos internos ou retenção dos registros. Os pontos ficaram registrados como riscos e questões em aberto (`RFC-OQ-09` a `RFC-OQ-12`); nenhum comportamento pendente é apresentado como decisão aprovada.

**Resultado registrado na validação automatizada anterior à revisão documental posterior**

```text
ADR files: 7 True
rows=404 transcricao=350 (87%) codigo=54
OK: nenhum erro nos checks implementados; este resultado não demonstra por si só cobertura de 80% dos itens identificáveis
```

<details>
<summary>Script de validação usado (clique para expandir)</summary>

```python
import re, os, glob, sys
root = sys.argv[1]; os.chdir(root)
tr = open('TRANSCRICAO.md', encoding='utf-8').read()
spoken = set(re.findall(r'^\[(\d\d:\d\d)\] (\w+):', tr, flags=re.M))
docs = ['docs/PRD.md','docs/RFC.md','docs/FDD.md','docs/TRACKER.md','README.md'] + sorted(glob.glob('docs/adrs/*.md'))
errors = []
# 1. todo [hh:mm] Nome citado existe na transcrição com aquele falante
for d in docs:
    for t, n in re.findall(r'\[(\d\d:\d\d)\] ([A-Z][a-z]+)', open(d, encoding='utf-8').read()):
        if (t, n) not in spoken: errors.append(f'{d}: timestamp inexistente [{t}] {n}')
# 2. linhas do tracker: 6 colunas, fonte válida, arquivo de código existe
rows = [[c.strip() for c in l.strip().strip('|').split(' | ')]
        for l in open('docs/TRACKER.md', encoding='utf-8')
        if l.startswith('| ') and not l.startswith('| ID ')]
tx = sum(r[4] == 'TRANSCRICAO' for r in rows); cod = sum(r[4] == 'CODIGO' for r in rows)
for i, doc, _, _, fonte, loc in rows:
    if fonte == 'CODIGO' and not os.path.isfile(loc): errors.append(f'{i}: arquivo inexistente {loc}')
    if fonte == 'TRANSCRICAO' and not re.match(r'^\[\d\d:\d\d\] [A-Z][a-z]+', loc): errors.append(f'{i}: localização inválida')
    if re.match(r'(PRD|RFC|FDD)-', i) and i not in open(doc, encoding='utf-8').read(): errors.append(f'{i}: ID não aparece em {doc}')
# 3. todo ID usado nos documentos tem linha no tracker
ids = {r[0] for r in rows}
for d in docs[:3] + docs[5:]:
    for ref in set(re.findall(r'\b((?:PRD|RFC|FDD)-[A-Z]+-\d{2})\b', open(d, encoding='utf-8').read())):
        if ref not in ids: errors.append(f'{d}: {ref} sem linha no tracker')
# 4. caminhos de código citados existem (exceto arquivos novos propostos pela reunião)
novo = re.compile(r'^(src/worker\.ts|src/modules/webhooks(/.*)?)$')
for d in docs:
    for p in set(re.findall(r'`((?:src|prisma|tests)/[\w./\-]+)`', open(d, encoding='utf-8').read())):
        if not novo.match(p.rstrip('/')) and not os.path.exists(p.rstrip('/')): errors.append(f'{d}: caminho inexistente {p}')
print(f'rows={len(rows)} transcricao={tx} ({tx/len(rows):.0%}) codigo={cod}')
print('\n'.join(errors) or 'OK: nenhum erro')
```

</details>

## Como navegar a entrega

```
.
├── README.md                 ← você está aqui (processo)
├── TRANSCRICAO.md            ← fonte primária (não alterada)
└── docs/
    ├── PRD.md                ← 1. porquê e o quê: problema, métricas, escopo, 16 RFs, 14 RNFs, riscos
    ├── RFC.md                ← 2. proposta técnica, 7 alternativas descartadas, 8 questões em aberto
    ├── adrs/
    │   ├── ADR-001-outbox-no-mysql.md                          ← 3. decisões, uma por arquivo
    │   ├── ADR-002-worker-separado-em-polling.md
    │   ├── ADR-003-retry-com-backoff-e-dlq.md
    │   ├── ADR-004-hmac-sha256-com-secret-por-endpoint.md
    │   ├── ADR-005-at-least-once-com-x-event-id.md
    │   ├── ADR-006-reuso-dos-padroes-existentes.md
    │   └── ADR-007-snapshot-do-payload-e-filtro-na-insercao.md
    ├── FDD.md                ← 4. como construir: modelo, 8 contratos, fluxos, erros WEBHOOK_*, integração
    └── TRACKER.md            ← 5. de onde veio cada item (404 linhas)
```

**Ordem sugerida de leitura**

1. [PRD](docs/PRD.md), para entender o problema de negócio e o que está fora de escopo.
2. [RFC](docs/RFC.md), para a visão de arquitetura em 3 a 4 páginas, alternativas e pendências.
3. [ADRs](docs/adrs/) de 001 a 007, para o porquê de cada decisão.
4. [FDD](docs/FDD.md), para implementar. Comece pela §11 (Integração com o sistema existente) e pela §6 (Contratos).
5. [TRACKER](docs/TRACKER.md), consultado sob demanda: busque o ID (ex.: `FDD-INT-01`) para ver a origem.

**Convenções**
- **Origem:** toda afirmação relevante traz `([hh:mm] Nome)` ou um caminho de arquivo.
- **Inferências:** itens que são inferência, e não fala da reunião, aparecem como **"(derivado)"**, **"detalhamento proposto"** ou como questão em aberto no RFC.
- **Arquivos novos:** `src/worker.ts` e `src/modules/webhooks/*` são **arquivos novos propostos na reunião** ([09:11] Larissa, [09:27] Bruno) e ainda não existem no repositório. Todos os demais caminhos citados existem.
