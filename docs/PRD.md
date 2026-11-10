# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) — API REST |
| **Feature** | Webhooks de notificação de mudança de status de pedido |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Envolvidos** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Status** | Aprovado em reunião; documentação em revisão |
| **Data da reunião de origem** | Não informada na transcrição (quinta-feira, 09:00) |
| **Prazo alvo** (PRD-META-01) | Fim de novembro ([09:45] Marcos), estimativa de 3 sprints ([09:47] Larissa) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

**PRD-CTX-01.** O OMS gerencia clientes, produtos e pedidos com um ciclo de vida controlado (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com possibilidade de `CANCELLED`). Hoje, quem integra com a plataforma **só descobre que um pedido mudou de status perguntando**: o sistema não tem nenhum mecanismo de notificação externa.

Esta feature entrega **webhooks de saída**. Quando o status de um pedido muda, a plataforma envia uma requisição HTTPS assinada para o endpoint cadastrado pelo cliente, em menos de 10 segundos, só para os status que ele escolheu acompanhar.

## 2. Problema e motivação

| ID | Problema / motivação | Fonte |
|---|---|---|
| PRD-PROB-01 | Três clientes B2B (**Atlas Comercial, MaxDistribuição e Nova Cargo**) fizeram pedido formal para serem notificados em tempo real sobre mudanças de status | [09:00] Marcos |
| PRD-PROB-02 | Hoje eles fazem polling periódico em `GET /orders`, o que deixa a integração **lenta e cara** para eles | [09:00] Marcos |
| PRD-PROB-03 | **Risco comercial:** a Atlas sinalizou que pode migrar para um concorrente se a feature não sair até o fim do trimestre | [09:00] Marcos |
| PRD-PROB-04 | Para os clientes, "tempo real" é qualquer coisa **abaixo de 10 segundos**. O importante é não precisar atualizar manualmente | [09:02] Marcos |

## 3. Público-alvo e cenários de uso

### 3.1 Público-alvo

| ID | Perfil | Interesse | Fonte |
|---|---|---|---|
| PRD-PUB-01 | **Clientes B2B integradores** (times de tecnologia de Atlas, MaxDistribuição, Nova Cargo) | Receber mudanças de status sem polling, validar a autenticidade e tratar duplicatas | [09:00] Marcos |
| PRD-PUB-02 | **Usuários da plataforma que representam o cliente** | Cadastrar e manter webhooks pela API, autenticados com o JWT do OMS | [09:32] Marcos |
| PRD-PUB-03 | **Administradores (role ADMIN)** | Reprocessar entregas que falharam definitivamente, com auditoria | [09:36] Sofia |
| PRD-PUB-04 | **Product Manager / portal do desenvolvedor** | Documentar a integração e a responsabilidade de deduplicação | [09:26] Marcos |

### 3.2 Cenários de uso

| ID | Cenário | Fonte |
|---|---|---|
| PRD-CEN-01 | A Atlas cadastra um endpoint `https` e escolhe receber só `SHIPPED` e `DELIVERED`. Ela passa a ser avisada em segundos quando um pedido é despachado ou entregue | [09:33] Marcos |
| PRD-CEN-02 | Um cliente fica **2 horas fora** numa manutenção planejada. As notificações do período chegam depois que ele volta, sem ação manual | [09:16] Diego |
| PRD-CEN-03 | Um cliente **vaza a secret** em log. Ele pede uma nova secret pela API e tem **24 h** em que a antiga ainda funciona, para migrar sem perder notificações | [09:21] Sofia, [09:22] Diego |
| PRD-CEN-04 | O cliente recebe a **mesma notificação duas vezes** e descarta a repetida pelo identificador do evento | [09:24] Diego |
| PRD-CEN-05 | O cliente diz "não recebi". Ele mesmo consulta o **histórico de entregas** (sucesso ou falha, payload, resposta e tempo de resposta) | [09:34] Marcos |
| PRD-CEN-06 | O endpoint fica indisponível até atingir o limite de retry que vier a ser aprovado; o evento vai à fila de falhas e um **ADMIN o reprocessa** depois que o cliente se recupera. A janela não está fixada | [09:17] Marcos, [09:18] Diego |
| PRD-CEN-07 | Um cliente com **vários endpoints** identifica qual cadastro originou cada notificação | [09:44] Sofia |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Fonte |
|---|---|---|---|---|
| PRD-OBJ-01 | Notificação percebida como "tempo real" | Tempo entre a mudança de status e a entrega bem-sucedida para endpoint saudável | **Alvo < 10 s**; percentil e perfil de carga ainda precisam ser definidos e validados (RFC-OQ-09) | [09:02] Marcos, [09:10] Larissa |
| PRD-OBJ-02 | Atender os clientes que motivaram a feature | Clientes solicitantes com webhook em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) **até o fim de novembro** | [09:00] Marcos, [09:45] Marcos |
| PRD-OBJ-03 | Nenhuma mudança de status "esquecida" | Mudanças de status com webhook inscrito **sem** evento registrado | **0** (100% de cobertura) | [09:40] Bruno |
| PRD-OBJ-04 | Tolerar indisponibilidade do cliente sem ação humana | Janela de indisponibilidade coberta pela política de retry | Cinco intervalos foram listados; a quantidade de chamadas e a janela efetiva dependem da decisão pendente RFC-OQ-08 | [09:17] Diego |
| PRD-OBJ-05 | Nenhuma notificação perdida em silêncio | Notificações que falharam definitivamente e ficaram registradas e reprocessáveis | **100%** | [09:18] Diego |
| PRD-OBJ-06 | Reduzir o polling dos clientes integrados | Volume de `GET /orders` dos clientes integrados, antes e depois (log `http_request`) | Indicador acompanhado; sem meta numérica definida na reunião (baseline a medir) | [09:00] Marcos |

## 5. Escopo

### 5.1 Incluso

| ID | Item | Fonte |
|---|---|---|
| PRD-IN-01 | Notificação **outbound** (plataforma → cliente) de mudança de status de pedido | [09:02] Marcos |
| PRD-IN-02 | Gestão de webhooks pela API: cadastrar, listar, editar e remover | [09:31] Marcos, [09:33] Bruno |
| PRD-IN-03 | Filtro por status em cada webhook | [09:33] Marcos |
| PRD-IN-04 | Assinatura das notificações e rotação de secret | [09:22] Sofia |
| PRD-IN-05 | Retentativas automáticas e fila de falhas definitivas com reprocessamento por ADMIN | [09:17] Larissa, [09:19] Larissa |
| PRD-IN-06 | Histórico de entregas consultável pelo cliente | [09:34] Marcos |

### 5.2 Fora de escopo

| ID | Item | Situação | Fonte |
|---|---|---|---|
| PRD-OUT-01 | **Aviso por e-mail** ao cliente quando o webhook falha repetidamente | Descartado nesta fase, **adiado** para a próxima, depois de medir o impacto | [09:37] Larissa, [09:38] Marcos |
| PRD-OUT-02 | **Dashboard visual** para o cliente ver seus webhooks | **Descartado**: só endpoints de API; painel é projeto separado do time de frontend | [09:40] Larissa |
| PRD-OUT-03 | **Rate limiting de envio** por cliente | **Adiado**: "observar e decidir depois" | [09:39] Diego, [09:39] Larissa |
| PRD-OUT-04 | **Webhooks de entrada** (cliente enviando para a plataforma) | **Descartado**: clientes só querem receber | [09:02] Marcos |
| PRD-OUT-05 | **Arquivamento** de notificações entregues (~30 dias) | Fora do escopo desta feature | [09:08] Diego |
| PRD-OUT-06 | **Ordenação global** das notificações e **escala para múltiplos processadores** | **Adiado**: "problema do futuro"; clientes não pediram ordenação global | [09:13] Diego, [09:14] Marcos |
| PRD-OUT-07 | **Entrega exatamente uma vez** (exactly-once) | **Descartado**: complexidade desproporcional; a garantia é "pelo menos uma vez" | [09:25] Diego |
| PRD-OUT-08 | **Restrição de perfil** para gerenciar webhooks (além de estar autenticado) | **Adiado**: "mais pra frente a gente pode endurecer" | [09:37] Sofia |

## 6. Requisitos funcionais

> Contratos, payloads e códigos de erro de cada requisito estão no [FDD §6](FDD.md#6-contratos-públicos).

| ID | Requisito | Prioridade | Fonte |
|---|---|---|---|
| PRD-FR-01 | O usuário autenticado deve **cadastrar um webhook** informando o cliente (`customerId` no corpo da requisição, não derivado do token), a URL e a lista de status de interesse | Must | [09:31] Marcos, [09:32] Larissa |
| PRD-FR-02 | A plataforma deve **gerar a secret** do webhook e devolvê-la ao usuário na criação | Must | [09:31] Marcos |
| PRD-FR-03 | O usuário deve **listar os webhooks de um cliente** | Must | [09:33] Bruno |
| PRD-FR-04 | O usuário deve **editar** um webhook (URL, status de interesse, ativo/inativo) | Must | [09:33] Bruno, [09:21] Bruno |
| PRD-FR-05 | O usuário deve **remover** um webhook | Must | [09:33] Bruno |
| PRD-FR-06 | A cada mudança de status de pedido, a plataforma deve **notificar os webhooks ativos do cliente do pedido que assinam o novo status**, e só esses | Must | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-07 | A notificação deve conter identificador do evento, tipo `order.status_changed`, data/hora, pedido (id e número), status anterior e novo, cliente e dados básicos como valor total, **sem os itens** | Must | [09:43] Diego |
| PRD-FR-08 | A notificação deve refletir o pedido **no momento da mudança de status**, mesmo se entregue horas depois | Must | [09:52] Larissa |
| PRD-FR-09 | Cada notificação deve ir **assinada**, para que o cliente verifique origem e integridade | Must | [09:19] Sofia, [09:20] Sofia |
| PRD-FR-10 | Cada notificação deve informar o **identificador do evento** (para deduplicação), o **identificador do webhook** e a **data/hora do envio** | Must | [09:25] Diego, [09:44] Diego, [09:44] Sofia |
| PRD-FR-11 | O usuário deve **rotacionar a secret** pela API; a secret anterior continua aceita por **24 h** | Must | [09:21] Sofia |
| PRD-FR-12 | A plataforma deve **retentar automaticamente** notificações que falharam, usando os intervalos discutidos de 1 min, 5 min, 30 min, 2 h e 12 h; a quantidade total de chamadas ainda depende da resolução de RFC-OQ-08 | Must; contagem pendente | [09:17] Larissa |
| PRD-FR-13 | Esgotadas as retentativas, a notificação deve ir para uma **fila de falhas definitivas**, com motivo e data/hora | Must | [09:15] Diego, [09:18] Diego |
| PRD-FR-14 | Um **ADMIN** deve poder **reprocessar** manualmente uma notificação da fila de falhas, com **registro de quem executou** | Must | [09:18] Diego, [09:36] Sofia |
| PRD-FR-15 | O usuário deve **consultar o histórico de entregas** de um webhook: sucesso/falha, payload, resposta e tempo de resposta | Must | [09:34] Marcos |
| PRD-FR-16 | A plataforma deve **recusar URLs que não sejam `https`** com erro de validação | Must | [09:23] Sofia |

## 7. Requisitos não funcionais

| ID | Categoria | Requisito | Fonte |
|---|---|---|---|
| PRD-NFR-01 | Desempenho | Alvo de notificação em **menos de 10 s** para endpoint saudável; o processamento verifica eventos novos **a cada 2 s**. Perfil de carga e critério estatístico para validar o alvo ainda não foram definidos (RFC-OQ-09) | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | Desempenho | A notificação **não pode deixar mais lenta nem bloquear** a mudança de status de pedidos por causa de clientes lentos ou fora do ar | [09:04] Bruno |
| PRD-NFR-03 | Confiabilidade | **Status mudou ⇒ evento registrado.** Se o registro falhar, a mudança de status não acontece | [09:40] Bruno |
| PRD-NFR-04 | Confiabilidade | Garantia **pelo menos uma vez** (at-least-once); duplicatas possíveis e identificáveis | [09:24] Diego, [09:26] Larissa |
| PRD-NFR-05 | Confiabilidade | O processamento das notificações roda **separado da API**: reinícios da API não interrompem entregas | [09:11] Diego |
| PRD-NFR-06 | Confiabilidade | Ordem de entrega preservada **por pedido** enquanto houver um único processador; **sem** garantia de ordem global | [09:13] Larissa |
| PRD-NFR-07 | Resiliência | Chamada ao cliente sem resposta em **10 s** é falha e entra em retentativa | [09:42] Diego |
| PRD-NFR-08 | Resiliência | A reunião listou cinco intervalos de backoff; não fechou de forma inequívoca se “5 tentativas” conta chamadas totais ou retentativas. Quantidade máxima e janela efetiva ficam pendentes em RFC-OQ-08 | [09:17] Diego, [09:17] Larissa |
| PRD-NFR-09 | Segurança | Assinatura **HMAC-SHA256** sobre o corpo, com **secret única por webhook** (nunca global) | [09:22] Sofia |
| PRD-NFR-10 | Segurança | **TLS obrigatório** (`https`) | [09:23] Sofia |
| PRD-NFR-11 | Segurança | Reprocessamento restrito à role **ADMIN**, com trilha de auditoria; gestão de webhooks para qualquer usuário autenticado | [09:36] Sofia, [09:37] Sofia |
| PRD-NFR-12 | Limites | Payload de notificação limitado a **64 KB**; acima disso a notificação **não é enviada** (erro, sem truncamento) | [09:23] Sofia, [09:24] Larissa |
| PRD-NFR-13 | Manutenibilidade | Seguir os padrões do projeto (módulos, erros com prefixo `WEBHOOK_`, logger, validação), **sem infraestrutura nova** | [09:30] Larissa, [09:07] Diego |
| PRD-NFR-14 | Conformidade de processo | **Revisão de segurança** (≥ 2 dias úteis) antes do deploy | [09:46] Sofia |

## 8. Decisões e trade-offs principais

| ID | Decisão | Trade-off aceito do ponto de vista de produto | ADR |
|---|---|---|---|
| PRD-DEC-01 | Registrar o evento junto com a mudança de status e enviar de forma assíncrona | Latência de alguns segundos em troca de nunca perder uma notificação e nunca travar pedidos | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| PRD-DEC-02 | Processador separado verificando a cada 2 s | ~2 s de atraso mínimo aceito pelo PM ("2 segundos serve") | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| PRD-DEC-03 | Backoff com os cinco intervalos discutidos e fila de falhas com reprocessamento manual; contagem de chamadas ainda pendente | A janela efetiva e o momento de encaminhar à DLQ dependem da decisão RFC-OQ-08 | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| PRD-DEC-04 | HMAC com secret por webhook e rotação de 24 h | Mais gestão de secrets em troca de conter vazamentos a um único endpoint | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| PRD-DEC-05 | Pelo menos uma vez, com identificador de evento | Cliente precisa deduplicar; mitigado com documentação destacada no portal | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| PRD-DEC-06 | Reuso dos padrões do projeto | Menos liberdade técnica, mais velocidade e consistência para cumprir o prazo | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| PRD-DEC-07 | Filtro aplicado no momento do evento; conteúdo "congelado" e enxuto | Mudança de filtro não é retroativa; itens do pedido exigem consulta adicional | [ADR-007](adrs/ADR-007-snapshot-do-payload-e-filtro-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Responsável | Fonte |
|---|---|---|---|
| PRD-DEP-01 | Documentação de integração no **portal do desenvolvedor**, com destaque para a deduplicação | Marcos (PM) | [09:26] Marcos, [09:40] Marcos |
| PRD-DEP-02 | **Revisão de segurança** de assinatura e geração de secret, com pelo menos 2 dias úteis antes do deploy | Sofia | [09:46] Sofia, [09:49] Sofia |
| PRD-DEP-03 | Alteração no fluxo de mudança de status de pedidos (time de Pedidos) | Bruno | [09:40] Bruno |
| PRD-DEP-04 | Novo processo de execução para implantar e operar (mesmo banco e mesma stack) | Diego (Plataforma) | [09:11] Diego |
| PRD-DEP-05 | Clientes com **endpoint `https`**, capazes de validar HMAC e deduplicar eventos | Clientes B2B | [09:23] Sofia, [09:24] Diego |
| PRD-DEP-06 | Confirmação do prazo com os clientes solicitantes | Marcos (PM) | [09:47] Marcos, [09:49] Marcos |
| PRD-DEP-07 | Sessão de revisão do design antes do início da implementação | Larissa, Bruno, Diego | [09:50] Larissa |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação | Fonte |
|---|---|---|---|---|---|
| PRD-RISK-01 | **Perda da Atlas para o concorrente** por atraso na entrega | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limiting fora), plano de 3 sprints, revisão de segurança já reservada no cronograma, prazo confirmado com o cliente | [09:00] Marcos, [09:47] Larissa |
| PRD-RISK-02 | **Cliente processa a mesma notificação duas vezes** por não deduplicar | Média | Médio | Identificador de evento em toda notificação; documentação destacada no portal do desenvolvedor | [09:25] Sofia, [09:26] Marcos |
| PRD-RISK-03 | **Vazamento da secret** de um cliente | Média (já ocorreu antes) | Alto | Secret única por webhook (o vazamento não afeta outros), rotação com 24 h de convivência, revisão de segurança | [09:22] Diego, [09:21] Sofia |
| PRD-RISK-04 | **Cliente lento ou fora do ar atrasa as notificações de outros clientes**, podendo exceder o alvo de 10 s | Média | Alto | Timeout de 10 s e medição de latência/idade da fila; definir perfil de carga e capacidade do worker antes de declarar o alvo validado (RFC-OQ-03, RFC-OQ-09) | [09:42] Diego, [09:13] Diego |
| PRD-RISK-05 | **Volume alto de notificações** para um mesmo cliente (ex.: 50 mudanças/min) sobrecarrega o endpoint dele | Baixa | Médio | Observar o volume por cliente depois do go-live e decidir sobre rate limiting | [09:38] Diego, [09:39] Larissa |
| PRD-RISK-06 | **Notificações falham definitivamente sem que o cliente perceba** (não há aviso por e-mail nesta fase) | Média | Médio | Histórico de entregas consultável, fila de falhas com reprocessamento por ADMIN, aviso por e-mail avaliado na próxima fase | [09:37] Larissa |
| PRD-RISK-07 | **Usuário autenticado pode informar ou consultar `customerId` sem regra documentada de vínculo/ownership entre usuário e cliente** | Não estimada | Alto | Não presumir que autenticação/role garante isolamento por cliente; decidir e documentar o modelo de autorização antes de implementar os endpoints (RFC-OQ-10) | [09:32] Larissa, [09:32] Bruno, [09:37] Sofia |

## 11. Critérios de aceitação

| ID | Critério (Dado / Quando / Então) | Requisitos |
|---|---|---|
| PRD-AC-01 | **Dado** um usuário autenticado, **quando** cadastra um webhook com URL `https`, cliente existente e status `SHIPPED`/`DELIVERED`, **então** o webhook é criado ativo e a resposta traz a secret gerada | FR-01, FR-02 |
| PRD-AC-02 | **Dado** um cadastro com URL `http://`, **quando** enviado, **então** é recusado com erro de validação | FR-16 |
| PRD-AC-03 | **Dado** um webhook inscrito em `SHIPPED`, **quando** um pedido desse cliente muda para `SHIPPED`, **então** o endpoint saudável deve receber a requisição assinada dentro do alvo de 10 s; a aprovação quantitativa depende do perfil de carga e critério estatístico ainda pendentes (RFC-OQ-09) | FR-06, FR-09, FR-10, NFR-01 |
| PRD-AC-04 | **Dado** um webhook inscrito só em `DELIVERED`, **quando** um pedido muda para `PAID`, **então** nenhuma notificação é gerada para ele | FR-06 |
| PRD-AC-05 | **Dado** um pedido que muda de status, **quando** o registro do evento falha, **então** o status do pedido permanece o anterior | NFR-03 |
| PRD-AC-06 | **Dado** um endpoint fora do ar, **quando** a notificação falha, **então** os retries usam os intervalos acordados; o número de chamadas antes da DLQ só pode ser fixado após resolver RFC-OQ-08 | FR-12, FR-13 |
| PRD-AC-07 | **Dado** uma notificação na fila de falhas, **quando** um OPERATOR tenta reprocessar, **então** recebe acesso negado; **quando** um ADMIN reprocessa, **então** a notificação volta a ser enviada com o mesmo identificador de evento e o ADMIN fica registrado | FR-14, NFR-11 |
| PRD-AC-08 | **Dado** um webhook, **quando** o usuário rotaciona a secret, **então** recebe a nova secret e, nas 24 h seguintes, as notificações são verificáveis com a nova e com a antiga | FR-11 |
| PRD-AC-09 | **Dado** um webhook com entregas, **quando** o usuário consulta o histórico, **então** vê cada tentativa com sucesso/falha, payload, resposta e tempo de resposta | FR-15 |
| PRD-AC-10 | **Dado** um pedido que mudou de status, **quando** a notificação é entregue, **então** o corpo contém os campos de PRD-FR-07 e **não** contém itens | FR-07, FR-08 |
| PRD-AC-11 | **Dado** o fluxo de pedidos existente, **quando** a feature é implantada, **então** os endpoints e respostas atuais de pedidos não mudam | NFR-02, NFR-13 |

## 12. Estratégia de testes e validação

| ID | Camada | O que valida | Como | Fonte |
|---|---|---|---|---|
| PRD-TEST-01 | Unitário | Cálculo da assinatura, calendário de retentativas, limite de 64 KB, filtro por status | Vitest (`npm test`), já usado no projeto | [09:20] Sofia, [09:17] Larissa |
| PRD-TEST-02 | Integração da API | CRUD, validação `https`, rotação, histórico, replay restrito a ADMIN | Vitest + Supertest, no padrão de `tests/orders.test.ts` | `tests/orders.test.ts` |
| PRD-TEST-03 | Integração do fluxo de pedidos | Mudança de status gera evento só para inscritos; falha no registro desfaz a mudança; testes atuais de pedidos seguem passando | Vitest contra MySQL local (`docker-compose.yml`) | [09:40] Bruno, [09:46] Larissa |
| PRD-TEST-04 | Ponta a ponta | Mudança de status → endpoint de teste recebe dentro do alvo de 10 s sob perfil de carga a definir; endpoint fora do ar → retries → fila de falhas → replay | API + processador rodando localmente com receptor HTTP de teste; perfil e contagem após resolver RFC-OQ-08 e RFC-OQ-09 | [09:46] Larissa |
| PRD-TEST-05 | Segurança | Revisão de código de assinatura e geração de secret; ausência de secret em logs e listagens | Revisão manual da Sofia (≥ 2 dias úteis) antes do deploy | [09:46] Sofia |
| PRD-TEST-06 | Validação de negócio | Metas PRD-OBJ-01 a PRD-OBJ-05 acompanhadas depois do go-live; PRD-OBJ-06 com baseline de chamadas `GET /orders` | Logs estruturados e métricas do [FDD §10](FDD.md#10-observabilidade) | [09:37] Larissa |
