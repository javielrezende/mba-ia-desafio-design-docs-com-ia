# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este documento é a referência cruzada do pacote de design docs: cada item relevante do PRD,
do RFC, do FDD e dos ADRs aparece aqui ligado à sua origem — uma fala da reunião ou um
arquivo real do código.

A regra que ele serve para verificar é simples: **item sem origem preenchível é item
inventado**. Toda linha abaixo teve sua `Localização` conferida contra `TRANSCRICAO.md` ou
contra o disco; nada foi acrescentado ao tracker apenas para "fechar" um documento.

## Como ler

| Coluna | O que traz |
|---|---|
| `ID` | Identificador estável do item, no formato `<DOC>-<FAMÍLIA>-<NN>`. ADRs usam o próprio número. |
| `Documento` | Caminho do documento onde o item está escrito. |
| `Tipo` | Natureza do item (Requisito Funcional, Decisão, Risco, Contrato, Erro, Integração…). |
| `Conteúdo (resumo)` | O item em uma linha. O texto completo está no documento de origem. |
| `Fonte` | `TRANSCRICAO` (a reunião) ou `CODIGO` (o repositório). |
| `Localização` | `[hh:mm] Nome` de quem disse, ou o caminho real do arquivo. |

**Critério de "item identificável"** adotado na varredura: entra no tracker todo item que o
documento apresenta com identidade própria — um requisito numerado, um objetivo, um item de
escopo, uma decisão, uma dependência, um risco, um critério de aceitação, um endpoint, um
código de erro, uma invariante, um ponto de integração. Parágrafos de contexto corrido não
geram linha própria; a afirmação que eles sustentam já aparece no item correspondente.

Itens marcados como **Hipótese** no documento de origem seguem marcados aqui: a `Localização`
aponta a decisão ou o risco real em que a hipótese foi ancorada, não uma fonte para o detalhe
extrapolado.

Duas exceções conscientes, para não inflar a tabela com linhas redundantes ou vazias:

- **Escopo incluso do PRD** não gera linhas próprias. Seus 8 itens correspondem um a um aos
  requisitos funcionais `PRD-FR-01` a `PRD-FR-08` e `PRD-FR-10`, com a mesma origem.
- **Duas linhas de requisito não funcional do PRD registram ausência de discussão** (compliance
  e acessibilidade). Elas não têm — e não devem ter — `Localização`: são o resultado de a
  reunião não ter tratado do assunto. Pela regra deste tracker, item sem fonte não vira linha;
  declarar isso aqui é mais honesto do que preencher a coluna.

## Resumo de cobertura

| Documento | Linhas | `TRANSCRICAO` | `CODIGO` |
|---|---|---|---|
| `docs/PRD.md` | 81 | 75 | 6 |
| `docs/RFC.md` | 23 | 20 | 3 |
| `docs/FDD.md` | 109 | 90 | 19 |
| `docs/adrs/` | 16 | 12 | 4 |
| **Total** | **229** | **197** (86,0%) | **32** (14,0%) |

---

## PRD — `docs/PRD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Substituir consulta repetida por notificação ativa, meta abaixo de 10s | TRANSCRICAO | `[09:02]` Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Nenhum evento perdido em silêncio, 100% entregue ou em fila de falhas | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Tolerar indisponibilidade do cliente, 5 tentativas em ~15h | TRANSCRICAO | `[09:17]` Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Cliente valida origem e integridade, 100% assinado e sobre https | TRANSCRICAO | `[09:20]`–`[09:23]` Sofia |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Entregar no compromisso com a Atlas, fim de novembro em 3 sprints | TRANSCRICAO | `[09:45]`–`[09:46]` Marcos, Larissa |
| PRD-ESC-01 | docs/PRD.md | Escopo (fora) | Não recebe webhook do cliente, fluxo apenas outbound | TRANSCRICAO | `[09:02]`–`[09:03]` Marcos, Sofia |
| PRD-ESC-02 | docs/PRD.md | Escopo (fora) | Painel visual para o cliente, projeto separado de frontend | TRANSCRICAO | `[09:39]`–`[09:40]` Marcos, Larissa |
| PRD-ESC-03 | docs/PRD.md | Escopo (fora) | Aviso por e-mail em falhas repetidas, adiado para fase futura | TRANSCRICAO | `[09:37]`–`[09:38]` Marcos, Larissa |
| PRD-ESC-04 | docs/PRD.md | Escopo (fora) | Controle de vazão em pico, registrado como ponto a observar | TRANSCRICAO | `[09:38]`–`[09:39]` Diego, Larissa |
| PRD-ESC-05 | docs/PRD.md | Escopo (fora) | Garantia de entrega única, deduplicação fica com o cliente | TRANSCRICAO | `[09:24]`–`[09:25]` Diego, Sofia |
| PRD-ESC-06 | docs/PRD.md | Escopo (fora) | Retenção e arquivamento de eventos antigos | TRANSCRICAO | `[09:08]` Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificação automática de mudança de status para webhooks interessados | TRANSCRICAO | `[09:00]` Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com url e status de interesse, secret devolvida na criação | TRANSCRICAO | `[09:31]` Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição, remoção e listagem de webhooks de um customer | TRANSCRICAO | `[09:33]` Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook, aplicado ao registrar o evento | TRANSCRICAO | `[09:33]`–`[09:34]` Marcos, Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com resultado, resposta e tempo | TRANSCRICAO | `[09:34]` Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Nova tentativa automática (5x, ~15h) e fila de falhas | TRANSCRICAO | `[09:15]`–`[09:18]` Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de entrega em falha, restrito a administrador | TRANSCRICAO | `[09:35]`–`[09:36]` Diego, Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Troca de secret com convivência de 24h | TRANSCRICAO | `[09:21]` Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Identificador único de evento para deduplicação pelo cliente | TRANSCRICAO | `[09:25]` Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Envio assinado para o cliente validar origem e integridade | TRANSCRICAO | `[09:20]` Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Notificação em menos de 10 segundos | TRANSCRICAO | `[09:02]` Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Verificação de novos eventos a cada 2 segundos | TRANSCRICAO | `[09:09]`–`[09:10]` Diego, Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Espera máxima de 10 segundos pela resposta do cliente | TRANSCRICAO | `[09:42]` Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Tamanho máximo de 64KB por evento, sem truncar | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia, Diego, Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Sem meta de uptime definida; evento pendente não se perde | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Indisponibilidade do cliente tolerada por ~15h | TRANSCRICAO | `[09:17]` Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 sobre o conteúdo enviado | TRANSCRICAO | `[09:20]` Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Secret exclusiva por webhook, rotacionável com 24h de convivência | TRANSCRICAO | `[09:21]` Sofia |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente https, http recusado por validação | TRANSCRICAO | `[09:23]` Sofia |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | CRUD para qualquer role autenticada; reprocessamento só para ADMIN | TRANSCRICAO | `[09:36]`–`[09:37]` Sofia, Marcos |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Secret não pode aparecer em log; filtro atual não cobre o campo | CODIGO | `src/shared/logger/index.ts` |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Histórico de entregas consultável pelo cliente (observabilidade) | TRANSCRICAO | `[09:34]` Marcos |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Auditoria de quem executou reprocessamento | TRANSCRICAO | `[09:36]` Sofia |
| PRD-NFR-14 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre mudança de status e registro do evento | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| PRD-NFR-15 | docs/PRD.md | Requisito Não Funcional | Conteúdo do evento congelado no instante da mudança de status | TRANSCRICAO | `[09:52]` Larissa |
| PRD-NFR-16 | docs/PRD.md | Requisito Não Funcional | Ordem garantida apenas entre eventos do mesmo pedido | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Larissa |
| PRD-NFR-17 | docs/PRD.md | Requisito Não Funcional | Nenhuma quebra de contrato para integrações existentes | TRANSCRICAO | `[09:30]` Larissa |
| PRD-NFR-18 | docs/PRD.md | Requisito Não Funcional | Registro estruturado das tentativas de envio, reaproveitando o log já usado na aplicação | CODIGO | `src/shared/logger/index.ts` |
| PRD-NFR-19 | docs/PRD.md | Requisito Não Funcional | Entrega ao menos uma vez: evento pode chegar repetido, nunca é descartado sem registro | TRANSCRICAO | `[09:18]`, `[09:24]` Diego |
| PRD-NFR-20 | docs/PRD.md | Requisito Não Funcional | Reuso da stack e dos padrões já em produção, sem infraestrutura nova para operar | TRANSCRICAO | `[09:07]` Diego, Larissa |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Notificar fora da transação do pedido; custo é não ser instantâneo | TRANSCRICAO | `[09:03]`–`[09:04]` Larissa, Bruno |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Entrega ao menos uma vez; custo é transferir dedup ao cliente | TRANSCRICAO | `[09:25]` Diego, Sofia |
| PRD-DEC-03 | docs/PRD.md | Trade-off | 5 tentativas em ~15h; custo é atraso e reprocessamento manual | TRANSCRICAO | `[09:15]`–`[09:18]` Diego |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Secret por endpoint com troca; custo é gerir mais segredos | TRANSCRICAO | `[09:21]`–`[09:22]` Sofia, Diego |
| PRD-DEC-05 | docs/PRD.md | Trade-off | Filtro no registro do evento; custo é não recuperar histórico | TRANSCRICAO | `[09:34]` Bruno, Diego |
| PRD-DEC-06 | docs/PRD.md | Trade-off | Ordem só por pedido; custo é limitar evolução da capacidade de envio | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança, mínimo 2 dias úteis antes do deploy | TRANSCRICAO | `[09:46]` Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | `[09:26]` Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Cliente expõe endpoint https e valida a assinatura | TRANSCRICAO | `[09:19]`–`[09:23]` Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Fluxo de mudança de status como gatilho da notificação | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-05 | docs/PRD.md | Dependência | Operação comporta um segundo processo contínuo | TRANSCRICAO | `[09:11]` Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Filtro de log precisa cobrir o campo de secret | CODIGO | `src/shared/logger/index.ts` |
| PRD-RISCO-01 | docs/PRD.md | Risco | Prazo de fim de novembro sob pressão | TRANSCRICAO | `[09:45]`–`[09:46]` Marcos, Larissa |
| PRD-RISCO-02 | docs/PRD.md | Risco | Pico de mudanças sobrecarrega o cliente, sem controle de vazão | TRANSCRICAO | `[09:38]`–`[09:39]` Diego, Larissa |
| PRD-RISCO-03 | docs/PRD.md | Risco | Cliente não dedupica e processa o mesmo evento duas vezes | TRANSCRICAO | `[09:25]` Sofia |
| PRD-RISCO-04 | docs/PRD.md | Risco | Vazamento da secret de assinatura, com precedente real | TRANSCRICAO | `[09:22]` Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Entregas paradas na fila de falhas até intervenção manual | TRANSCRICAO | `[09:18]` Diego |
| PRD-RISCO-06 | docs/PRD.md | Risco | Formato do evento sem versionamento quebra integração futura | TRANSCRICAO | `[09:43]` Diego |
| PRD-RISCO-07 | docs/PRD.md | Risco | Envio interrompido atrasa todas as notificações | TRANSCRICAO | `[09:11]` Diego |
| PRD-AC-01 | docs/PRD.md | Critério de aceitação | Cadastro de webhook por usuário autenticado devolve a secret na criação | TRANSCRICAO | `[09:31]` Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de aceitação | Cadastro com URL não-https é recusado e não é persistido | TRANSCRICAO | `[09:23]` Sofia |
| PRD-AC-03 | docs/PRD.md | Critério de aceitação | Mudança de status com webhook interessado notifica em menos de 10s | TRANSCRICAO | `[09:02]` Marcos |
| PRD-AC-04 | docs/PRD.md | Critério de aceitação | Mudança sem webhook interessado não gera notificação nem registro | TRANSCRICAO | `[09:34]` Bruno |
| PRD-AC-05 | docs/PRD.md | Critério de aceitação | Identificador de evento único e estável entre as tentativas do mesmo evento | TRANSCRICAO | `[09:25]` Diego |
| PRD-AC-06 | docs/PRD.md | Critério de aceitação | Toda notificação é assinada e verificável com a secret daquele webhook | TRANSCRICAO | `[09:20]` Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de aceitação | Cliente indisponível recebe 5 tentativas em 1min, 5min, 30min, 2h e 12h | TRANSCRICAO | `[09:17]` Diego |
| PRD-AC-08 | docs/PRD.md | Critério de aceitação | Esgotadas as tentativas, a entrega vai para a fila de falhas com motivo e momento | TRANSCRICAO | `[09:18]` Diego |
| PRD-AC-09 | docs/PRD.md | Critério de aceitação | Histórico devolve os últimos 100 envios com resultado, resposta e tempo | TRANSCRICAO | `[09:34]` Marcos |
| PRD-AC-10 | docs/PRD.md | Critério de aceitação | Troca de secret mantém a anterior válida por 24h em paralelo | TRANSCRICAO | `[09:21]` Sofia |
| PRD-AC-11 | docs/PRD.md | Critério de aceitação | Só administrador reprocessa, e o reprocessamento registra quem executou | TRANSCRICAO | `[09:36]` Sofia |
| PRD-AC-12 | docs/PRD.md | Critério de aceitação | Mudança de status não concluída não deixa notificação registrada | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| PRD-AC-13 | docs/PRD.md | Critério de aceitação | Conteúdo reflete o pedido no instante da mudança, mesmo que ele mude depois | TRANSCRICAO | `[09:52]` Larissa |
| PRD-AC-14 | docs/PRD.md | Critério de aceitação | Revisão de segurança concluída e documentação publicada antes do lançamento | TRANSCRICAO | `[09:46]` Sofia, `[09:26]` Marcos |
| PRD-TESTE-01 | docs/PRD.md | Teste e validação | Testes de integração ponta a ponta no padrão vigente do projeto | CODIGO | `tests/orders.test.ts` |
| PRD-TESTE-02 | docs/PRD.md | Teste e validação | Testes das regras críticas de entrega: atomicidade, janela de retry e filtro | TRANSCRICAO | `[09:17]`, `[09:34]`, `[09:40]`–`[09:41]` Diego, Bruno |
| PRD-TESTE-03 | docs/PRD.md | Teste e validação | Testes das regras de segurança: recusa de http, assinatura e convivência de 24h | TRANSCRICAO | `[09:21]`, `[09:23]` Sofia |
| PRD-TESTE-04 | docs/PRD.md | Teste e validação | Revisão de segurança manual, com no mínimo dois dias úteis antes do deploy | TRANSCRICAO | `[09:46]` Sofia |
| PRD-TESTE-05 | docs/PRD.md | Teste e validação (Hipótese) | Ensaio de rajada, ancorado no risco de pico sem controle de vazão | TRANSCRICAO | `[09:38]` Diego |
| PRD-TESTE-06 | docs/PRD.md | Teste e validação | Seguir a suíte automatizada existente, com revisão de segurança como portão | CODIGO | `tests/helpers/factories.ts` |
| PRD-TESTE-07 | docs/PRD.md | Teste e validação (Hipótese) | Piloto com um cliente, ancorado no alinhamento direto do PM com os clientes | TRANSCRICAO | `[09:47]` Marcos |

---

## RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Metadado | Revisores são os 5 participantes; sem data de calendário registrada, apenas "quinta-feira, 09:00" | TRANSCRICAO | `TRANSCRICAO.md:3` |
| RFC-CTX-01 | docs/RFC.md | Contexto | Clientes fazem polling em `GET /orders`; necessidade de tempo real abaixo de 10s | TRANSCRICAO | `[09:00]`–`[09:02]` Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | Escopo estritamente outbound, sem canal de entrada do cliente | TRANSCRICAO | `[09:02]`–`[09:03]` Marcos, Sofia |
| RFC-PROP-01 | docs/RFC.md | Proposta | Ponto de enxerto do outbox na transação de `changeStatus` | CODIGO | `src/modules/orders/order.service.ts:126` |
| RFC-PROP-02 | docs/RFC.md | Proposta | Outbox transacional: evento inserido na mesma transação da mudança de status | TRANSCRICAO | `[09:06]`–`[09:08]` Diego |
| RFC-PROP-03 | docs/RFC.md | Proposta | Worker Node separado, com Prisma próprio, varrendo a outbox a cada 2s | TRANSCRICAO | `[09:09]`–`[09:11]` Diego, Larissa |
| RFC-PROP-04 | docs/RFC.md | Proposta | Retry com backoff em 5 tentativas e DLQ reprocessável por endpoint admin | TRANSCRICAO | `[09:15]`–`[09:19]` Diego, Larissa |
| RFC-PROP-05 | docs/RFC.md | Proposta | Assinatura HMAC-SHA256 com secret por endpoint, rotacionável com grace de 24h | TRANSCRICAO | `[09:20]`–`[09:22]` Sofia |
| RFC-PROP-06 | docs/RFC.md | Proposta | Garantia at-least-once, dedup pelo cliente via identificador de evento | TRANSCRICAO | `[09:24]`–`[09:26]` Diego, Larissa |
| RFC-PROP-07 | docs/RFC.md | Proposta | Configuração por API CRUD autenticada, com filtro aplicado já na inserção | TRANSCRICAO | `[09:31]`–`[09:34]` Marcos, Bruno |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono do webhook dentro da transação de status | TRANSCRICAO | `[09:03]`–`[09:04]` Bruno, Larissa |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Fila dedicada (ex.: Redis Streams) em vez de outbox no MySQL | TRANSCRICAO | `[09:07]` Diego, Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco de dados para acordar o worker | TRANSCRICAO | `[09:09]` Bruno, Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Garantia de entrega exactly-once | TRANSCRICAO | `[09:24]`–`[09:25]` Diego, Sofia |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de envio ao cliente, sem responsável/critério definido | TRANSCRICAO | `[09:38]`–`[09:39]` Diego, Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Notificação por e-mail em falhas repetidas — adiado | TRANSCRICAO | `[09:37]`–`[09:38]` Marcos, Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Escalonamento para múltiplos workers em paralelo — adiado | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Bruno |
| RFC-IMP-01 | docs/RFC.md | Impacto | Escrita adicional dentro da transação já existente de mudança de status | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-IMP-02 | docs/RFC.md | Impacto | Novo processo deployável em produção, com ciclo de vida próprio | TRANSCRICAO | `[09:11]` Diego |
| RFC-IMP-03 | docs/RFC.md | Impacto | Schema recebe novas tabelas e migration correspondente | CODIGO | `prisma/schema.prisma` |
| RFC-RISK-01 | docs/RFC.md | Risco | Ordering não é global: vale por `order_id` e só com um worker | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Larissa |
| RFC-RISK-02 | docs/RFC.md | Risco | Rajada de envios sem rate limiting sobrecarrega o cliente | TRANSCRICAO | `[09:38]`–`[09:39]` Diego, Larissa |
| RFC-RISK-03 | docs/RFC.md | Risco | Prazo fim de novembro vs. estimativa de 3 sprints com revisão de segurança | TRANSCRICAO | `[09:45]`–`[09:47]` Marcos, Larissa |

---

## FDD — `docs/FDD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CTX-01 | docs/FDD.md | Contexto | Acoplamento ineficiente por polling; tempo real definido como latência abaixo de 10s | TRANSCRICAO | `[09:00]`–`[09:02]` Marcos |
| FDD-CTX-02 | docs/FDD.md | Contexto | Encaixe no sistema atual: transação Prisma única já existente em `changeStatus` | CODIGO | `src/modules/orders/order.service.ts:126-179` |
| FDD-CTX-03 | docs/FDD.md | Contexto | Escopo estritamente outbound, sem canal de entrada do cliente | TRANSCRICAO | `[09:02]`–`[09:03]` Marcos, Sofia |
| FDD-CTX-04 | docs/FDD.md | Ator | Operador autenticado gerencia webhooks (`customer_id` fora do JWT); replay só para ADMIN | TRANSCRICAO | `[09:31]`–`[09:32]` Bruno, `[09:36]`–`[09:37]` Sofia |
| FDD-CTX-05 | docs/FDD.md | Suposição | Time pequeno não opera fila dedicada; MySQL existente é reaproveitado | TRANSCRICAO | `[09:07]` Diego, Larissa |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Latência abaixo de 10s, pior caso dominado pelo polling de 2s do worker | TRANSCRICAO | `[09:02]` Marcos, `[09:09]`–`[09:10]` Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Atomicidade: commit garante o evento na outbox, rollback garante que ele não exista | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | At-least-once com identificador único gerado na inserção na outbox | TRANSCRICAO | `[09:24]`–`[09:26]` Diego, Larissa |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Ordering só por `order_id` e só em regime single-worker, como limitação conhecida | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Larissa |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Resiliência a indisponibilidade do cliente por até ~15h antes da DLQ | TRANSCRICAO | `[09:15]`–`[09:17]` Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Payload limitado a 64KB, com falha explícita e sem truncamento | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia, Diego, Larissa |
| FDD-OBJ-07 | docs/FDD.md | Objetivo técnico | Timeout de 10s por chamada HTTP do worker, tratado como falha e reagendado | TRANSCRICAO | `[09:42]` Sofia, Diego |
| FDD-ESC-01 | docs/FDD.md | Escopo (incluído) | CRUD de configuração de webhook por customer | TRANSCRICAO | `[09:31]`–`[09:33]` Marcos, Bruno |
| FDD-ESC-02 | docs/FDD.md | Escopo (incluído) | Filtro de eventos aplicado na inserção na outbox, não no envio | TRANSCRICAO | `[09:33]`–`[09:34]` Bruno, Diego |
| FDD-ESC-03 | docs/FDD.md | Escopo (incluído) | Histórico de entregas com os últimos 100 registros | TRANSCRICAO | `[09:34]` Marcos |
| FDD-ESC-04 | docs/FDD.md | Escopo (incluído) | Endpoint admin de replay de DLQ, com role ADMIN e log de auditoria | TRANSCRICAO | `[09:18]` Diego, `[09:35]`–`[09:36]` Sofia |
| FDD-ESC-05 | docs/FDD.md | Escopo (incluído) | Rotação de secret por endpoint próprio, com grace period de 24h | TRANSCRICAO | `[09:21]` Sofia |
| FDD-ESC-06 | docs/FDD.md | Escopo (incluído) | Validações de URL https obrigatória e limite de payload de 64KB | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia |
| FDD-ESC-07 | docs/FDD.md | Escopo (incluído) | Pipeline completo outbox → worker → HTTP assinado → retry/backoff → DLQ | TRANSCRICAO | `[09:06]`–`[09:11]` Diego, Larissa |
| FDD-EXC-01 | docs/FDD.md | Escopo (excluído) | Disparo síncrono do webhook dentro da transação de `order.service` | TRANSCRICAO | `[09:03]`–`[09:04]` Bruno, Larissa |
| FDD-EXC-02 | docs/FDD.md | Escopo (excluído) | Fila dedicada (Redis Streams ou similar) | TRANSCRICAO | `[09:07]` Diego, Larissa |
| FDD-EXC-03 | docs/FDD.md | Escopo (excluído) | Trigger de banco de dados para acordar o worker | TRANSCRICAO | `[09:09]` Bruno, Diego |
| FDD-EXC-04 | docs/FDD.md | Escopo (excluído) | Retry com apenas 3 tentativas ou retry indefinido sem teto | TRANSCRICAO | `[09:15]`–`[09:16]` Diego, Bruno |
| FDD-EXC-05 | docs/FDD.md | Escopo (excluído) | Garantia exactly-once | TRANSCRICAO | `[09:24]`–`[09:25]` Diego, Sofia |
| FDD-EXC-06 | docs/FDD.md | Escopo (excluído) | Truncamento de payload acima do limite | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia |
| FDD-EXC-07 | docs/FDD.md | Escopo (excluído) | Dashboard visual para o cliente acompanhar webhooks | TRANSCRICAO | `[09:39]`–`[09:40]` Marcos, Larissa |
| FDD-EXC-08 | docs/FDD.md | Escopo (excluído) | Notificação por e-mail em falhas repetidas — adiado para fase futura | TRANSCRICAO | `[09:37]`–`[09:38]` Marcos, Larissa |
| FDD-EXC-09 | docs/FDD.md | Escopo (excluído) | Múltiplos workers em paralelo / particionamento por `order_id` — adiado | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Bruno |
| FDD-EXC-10 | docs/FDD.md | Escopo (excluído) | Arquivamento de linhas entregues na outbox (~30 dias) — adiado | TRANSCRICAO | `[09:08]` Diego |
| FDD-EXC-11 | docs/FDD.md | Escopo (excluído) | Rate limiting de envio ao cliente — fora desta entrega, risco documentado | TRANSCRICAO | `[09:38]`–`[09:39]` Diego, Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo principal | Passos 1–2: transação única valida a transição, ajusta estoque e grava histórico | CODIGO | `src/modules/orders/order.service.ts:126-179` |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo principal | Passo 3: `publishWebhookEvent(tx, ...)` chamado de dentro da transação | TRANSCRICAO | `[09:41]`–`[09:42]` Bruno, Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo principal | Passo 4: busca dos webhooks interessados; sem interessado, nenhuma linha é criada | TRANSCRICAO | `[09:33]`–`[09:34]` Bruno, Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo principal | Passos 5–6: payload renderizado como snapshot e uma linha por webhook interessado | TRANSCRICAO | `[09:43]` Marcos, Diego, `[09:51]`–`[09:52]` Larissa |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo principal | Passo 7: falha na inserção na outbox provoca rollback de toda a transação | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo principal | Passos 8–9: polling a cada 2s, assinatura, headers e POST com timeout de 10s | TRANSCRICAO | `[09:09]`–`[09:10]` Diego, `[09:42]`–`[09:44]` Sofia, Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo principal | Passos 10–12: sucesso em 2xx, reagendamento por backoff, DLQ após 5 tentativas | TRANSCRICAO | `[09:15]`–`[09:18]` Diego |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo alternativo | CRUD de configuração, rotação de secret e replay de DLQ | TRANSCRICAO | `[09:21]` Sofia, `[09:33]` Bruno, `[09:35]`–`[09:36]` Diego |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo alternativo (Hipótese) | Payload acima de 64KB no envio: falha permanente direto para DLQ, sem retry | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia, Diego, Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks` — cadastro de webhook (nome da rota é Hipótese) | TRANSCRICAO | `[09:31]`–`[09:32]` Marcos, Bruno |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks?customerId=...` — listagem | TRANSCRICAO | `[09:33]` Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` — edição | TRANSCRICAO | `[09:33]` Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id` — remoção | TRANSCRICAO | `[09:33]` Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /webhooks/:id/rotate-secret` — rotação (nome da rota é Hipótese) | TRANSCRICAO | `[09:21]` Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` — histórico de entregas | TRANSCRICAO | `[09:34]` Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` — replay de DLQ (admin) | TRANSCRICAO | `[09:18]`, `[09:35]` Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Contrato outbound: request assinado enviado ao cliente (payload e headers) | TRANSCRICAO | `[09:43]`–`[09:44]` Marcos, Diego, Sofia |
| FDD-ERRO-01 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` — id de webhook inexistente | TRANSCRICAO | `[09:27]`–`[09:30]` Bruno, Larissa |
| FDD-ERRO-02 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` — URL cadastrada não é https | TRANSCRICAO | `[09:23]` Sofia |
| FDD-ERRO-03 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` — webhook sem secret válida (condição é Hipótese) | TRANSCRICAO | `[09:27]`–`[09:30]` Bruno, Larissa |
| FDD-ERRO-04 | docs/FDD.md | Erro | `WEBHOOK_INVALID_STATUS_FILTER` — filtro vazio ou fora do enum (nome é Hipótese) | TRANSCRICAO | `[09:33]`–`[09:34]` Bruno, Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` — replay de id inexistente (nome é Hipótese) | TRANSCRICAO | `[09:18]`–`[09:19]` Diego, Larissa |
| FDD-ERRO-06 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` — payload acima de 64KB | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia, Diego, Larissa |
| FDD-ERRO-07 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` — sem resposta em 10s, entra em retry | TRANSCRICAO | `[09:42]` Sofia, Diego |
| FDD-ERRO-08 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_FAILED` — resposta não-2xx ou erro de rede, entra em retry | TRANSCRICAO | `[09:15]`–`[09:17]` Diego |
| FDD-RESIL-01 | docs/FDD.md | Resiliência | Timeout de 10s por chamada e retry com backoff em 5 tentativas | TRANSCRICAO | `[09:15]`–`[09:17]` Diego, `[09:42]` Sofia |
| FDD-RESIL-02 | docs/FDD.md | Fallback | Esgotadas as tentativas, evento vai para a DLQ; replay manual restrito a ADMIN | TRANSCRICAO | `[09:18]` Diego, `[09:35]`–`[09:36]` Sofia |
| FDD-INV-01 | docs/FDD.md | Invariante | Commit da transação garante o evento na outbox; rollback garante que não exista | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| FDD-INV-02 | docs/FDD.md | Invariante | Payload nunca é truncado; ultrapassar 64KB é sempre erro explícito | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia |
| FDD-INV-03 | docs/FDD.md | Invariante | Payload é snapshot imutável desde a inserção, nunca recalculado no envio | TRANSCRICAO | `[09:51]`–`[09:52]` Larissa, Diego |
| FDD-INV-04 | docs/FDD.md | Invariante | Secret nunca aparece em log; `redactPaths` hoje não cobre o campo `secret` | CODIGO | `src/shared/logger/index.ts` |
| FDD-INV-05 | docs/FDD.md | Invariante | At-least-once não degrada: todo evento é entregue ou vai para a DLQ | TRANSCRICAO | `[09:18]` Diego, `[09:24]`–`[09:26]` Diego, Larissa |
| FDD-OBS-01 | docs/FDD.md | Observabilidade (Hipótese) | Métricas de backlog da outbox, tentativas, latência de pickup e movimentos para DLQ | TRANSCRICAO | `[09:02]` Marcos, `[09:09]`–`[09:10]`, `[09:18]` Diego |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Logs estruturados no logger Pino central, com campos do worker e auditoria do replay | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-03 | docs/FDD.md | Observabilidade (Hipótese) | Spans de inserção, ciclo de polling, envio e replay, com amostragem proposta | TRANSCRICAO | `[09:09]`–`[09:11]` Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade (Hipótese) | Painéis e alertas de backlog crescente, taxa de DLQ e latência acima do objetivo | TRANSCRICAO | `[09:02]` Marcos, `[09:11]`, `[09:18]` Diego |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node >=20: `fetch` com `AbortController` e `crypto.createHmac` cobrem worker e assinatura | CODIGO | `package.json` |
| FDD-DEP-02 | docs/FDD.md | Dependência | Prisma 5.22.0 já em uso; novas tabelas via migration nas convenções existentes | CODIGO | `package.json` |
| FDD-DEP-03 | docs/FDD.md | Dependência | uuid 11.0.3 já em uso, reaproveitado para identificador de evento e ids | CODIGO | `package.json` |
| FDD-DEP-04 | docs/FDD.md | Dependência | zod 3.23.8 já em uso, reaproveitado nos schemas do módulo de webhooks | CODIGO | `package.json` |
| FDD-DEP-05 | docs/FDD.md | Dependência | pino 9.5.0 já em uso; logger importado diretamente no worker | CODIGO | `package.json` |
| FDD-COMPAT-01 | docs/FDD.md | Compatibilidade | Nenhuma mudança de contrato nos endpoints existentes | TRANSCRICAO | `[09:30]` Larissa |
| FDD-COMPAT-02 | docs/FDD.md | Compatibilidade | `changeStatus` mantém a assinatura pública; integração é interna à transação | TRANSCRICAO | `[09:41]`–`[09:42]` Bruno, Diego |
| FDD-COMPAT-03 | docs/FDD.md | Compatibilidade | Novos routers registrados no padrão existente, sem impacto nos módulos atuais | CODIGO | `src/routes/index.ts` |
| FDD-COMPAT-04 | docs/FDD.md | Compatibilidade | Schema Prisma estritamente aditivo, sem alterar tabelas ou colunas existentes | TRANSCRICAO | `[09:51]` Larissa |
| FDD-COMPAT-05 | docs/FDD.md | Compatibilidade (gap) | Payload sem campo de versionamento; mudança futura de formato exigirá estratégia própria | TRANSCRICAO | `[09:43]` Marcos, Diego |
| FDD-AC-01 | docs/FDD.md | Critério de aceite técnico | CRUD completo; secret devolvida só na criação e na rotação | TRANSCRICAO | `[09:31]`–`[09:33]` Marcos, Bruno |
| FDD-AC-02 | docs/FDD.md | Critério de aceite técnico | Filtro na inserção: sem webhook interessado, nenhuma linha na outbox | TRANSCRICAO | `[09:33]`–`[09:34]` Bruno, Diego |
| FDD-AC-03 | docs/FDD.md | Critério de aceite técnico | Atomicidade comprovada por teste: falha na outbox provoca rollback do status | TRANSCRICAO | `[09:40]`–`[09:41]` Bruno, Diego |
| FDD-AC-04 | docs/FDD.md | Critério de aceite técnico | Payload é snapshot imutável desde a inserção | TRANSCRICAO | `[09:51]`–`[09:52]` Larissa |
| FDD-AC-05 | docs/FDD.md | Critério de aceite técnico | Cada request carrega os headers definidos, com assinatura validável pelo cliente | TRANSCRICAO | `[09:44]` Diego, Sofia |
| FDD-AC-06 | docs/FDD.md | Critério de aceite técnico | Rotação testada: secret antiga válida por até 24h em paralelo com a nova | TRANSCRICAO | `[09:21]` Sofia |
| FDD-AC-07 | docs/FDD.md | Critério de aceite técnico | Histórico de entregas retorna no máximo os 100 registros mais recentes | TRANSCRICAO | `[09:34]` Marcos |
| FDD-AC-08 | docs/FDD.md | Critério de aceite técnico | Replay devolve 403 para não-ADMIN e registra quem executou | TRANSCRICAO | `[09:35]`–`[09:36]` Diego, Sofia |
| FDD-AC-09 | docs/FDD.md | Critério de aceite técnico | CRUD de configuração aceita qualquer role autenticada | TRANSCRICAO | `[09:36]`–`[09:37]` Sofia, Marcos |
| FDD-AC-10 | docs/FDD.md | Critério de aceite técnico | Latência ponta a ponta abaixo de 10s no p95 em condições normais | TRANSCRICAO | `[09:02]` Marcos |
| FDD-AC-11 | docs/FDD.md | Critério de aceite técnico | Timeout da chamada HTTP do worker fixado em 10s | TRANSCRICAO | `[09:42]` Diego |
| FDD-AC-12 | docs/FDD.md | Critério de aceite técnico | Retry em exatamente 5 tentativas com o backoff definido, validado por teste | TRANSCRICAO | `[09:15]`–`[09:17]` Diego |
| FDD-AC-13 | docs/FDD.md | Critério de aceite técnico | Após a 5ª falha, evento é movido para a DLQ com payload, motivo e timestamp | TRANSCRICAO | `[09:18]` Diego |
| FDD-AC-14 | docs/FDD.md | Critério de aceite técnico | Payload acima de 64KB falha permanentemente, sem consumir tentativas de retry | TRANSCRICAO | `[09:23]`–`[09:24]` Sofia, Diego |
| FDD-AC-15 | docs/FDD.md | Critério de aceite técnico | Cadastro com URL http é rejeitado com `WEBHOOK_INVALID_URL` | TRANSCRICAO | `[09:23]` Sofia |
| FDD-AC-16 | docs/FDD.md | Critério de aceite técnico | Logs do worker trazem os campos definidos e nenhuma secret em texto plano | CODIGO | `src/shared/logger/index.ts` |
| FDD-AC-17 | docs/FDD.md | Critério de aceite técnico | Toda execução de replay gera log de auditoria com quem executou | TRANSCRICAO | `[09:36]` Sofia |
| FDD-AC-18 | docs/FDD.md | Critério de aceite técnico | Revisão de segurança concluída, com no mínimo 2 dias úteis, antes do deploy | TRANSCRICAO | `[09:46]` Sofia |
| FDD-RISCO-01 | docs/FDD.md | Risco | Rajada de envios sem rate limiting sobrecarrega o cliente | TRANSCRICAO | `[09:38]`–`[09:39]` Diego, Larissa |
| FDD-RISCO-02 | docs/FDD.md | Risco | Ordering não garantido globalmente se um dia houver mais de um worker | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Larissa |
| FDD-RISCO-03 | docs/FDD.md | Risco | Worker como ponto único de falha: outbox para de ser processada até o restart | TRANSCRICAO | `[09:11]` Diego |
| FDD-RISCO-04 | docs/FDD.md | Risco | Vazamento de secret em log, com precedente real citado na reunião | TRANSCRICAO | `[09:22]` Diego |
| FDD-RISCO-05 | docs/FDD.md | Risco | Ausência de versionamento de payload quebra integrações no longo prazo | TRANSCRICAO | `[09:43]` Diego |
| FDD-RISCO-06 | docs/FDD.md | Risco | Cronograma: prazo de fim de novembro contra estimativa de 3 sprints | TRANSCRICAO | `[09:45]`–`[09:47]` Marcos, Larissa |
| FDD-RISCO-07 | docs/FDD.md | Risco | DLQ sem reprocessamento automático: eventos presos até intervenção manual | TRANSCRICAO | `[09:18]` Diego |
| FDD-INT-01 | docs/FDD.md | Integração | Enxerto de `publishWebhookEvent` na transação existente de `changeStatus` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | docs/FDD.md | Integração | Reuso 1:1 da hierarquia de erro, com códigos prefixados `WEBHOOK_` | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-03 | docs/FDD.md | Integração | Reuso do logger Pino central; `redactPaths` precisa cobrir `secret` | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-04 | docs/FDD.md | Integração | Registro dos novos routers seguindo o padrão de `buildApiRouter` | CODIGO | `src/routes/index.ts` |
| FDD-INT-05 | docs/FDD.md | Integração | Modelo de referência de rotas e schemas vindo do módulo de pedidos | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-INT-06 | docs/FDD.md | Integração | Reuso de `authenticate` e `requireRole('ADMIN')` sem alterar o middleware | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-07 | docs/FDD.md | Integração | Novas tabelas seguindo as convenções vigentes do schema | CODIGO | `prisma/schema.prisma` |
| FDD-INT-08 | docs/FDD.md | Integração | Modelo de estilo do repository: classe fina sobre o Prisma | CODIGO | `src/modules/orders/order.repository.ts` |

---

## ADRs — `docs/adrs/`

Cada linha corresponde a uma decisão registrada no ADR. As alternativas descartadas em cada
ADR não ganham linha própria aqui: elas já estão rastreadas nas linhas `RFC-ALT-01` a
`RFC-ALT-04` e `FDD-EXC-01` a `FDD-EXC-11`, com a mesma origem.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-mysql.md | Decisão | Padrão outbox no MySQL, inserção na mesma transação do `changeStatus` | TRANSCRICAO | `[09:06]`–`[09:08]` Diego |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-mysql.md | Decisão | Snapshot do payload no momento da inserção | TRANSCRICAO | `[09:51]`–`[09:52]` Larissa |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | Retry com 5 tentativas, backoff 1min/5min/30min/2h/12h | TRANSCRICAO | `[09:15]`–`[09:17]` Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | DLQ em tabela separada, com replay manual via endpoint admin | TRANSCRICAO | `[09:18]`–`[09:19]` Diego |
| ADR-003 | docs/adrs/ADR-003-autenticacao-hmac-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint, não global | TRANSCRICAO | `[09:20]`–`[09:22]` Sofia |
| ADR-003 | docs/adrs/ADR-003-autenticacao-hmac-secret-por-endpoint.md | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21]`–`[09:22]` Sofia |
| ADR-004 | docs/adrs/ADR-004-garantia-at-least-once-event-id.md | Decisão | Garantia at-least-once, dedup via `X-Event-Id` a cargo do cliente | TRANSCRICAO | `[09:24]`–`[09:26]` Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Worker em processo Node separado, com polling a cada 2s | TRANSCRICAO | `[09:09]`–`[09:11]` Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-polling.md | Decisão | Ordering só por `order_id` e só com single-worker (limitação conhecida) | TRANSCRICAO | `[09:12]`–`[09:13]` Diego, Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso de padrão de módulo, `AppError`, Pino e error middleware; prefixo `WEBHOOK_` | TRANSCRICAO | `[09:27]`–`[09:30]` Bruno, Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Função `publishWebhookEvent(tx, ...)` em vez de injetar o repository inteiro | TRANSCRICAO | `[09:41]`–`[09:42]` Bruno, Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Código citado | Hierarquia de erro `AppError` reaproveitada | CODIGO | `src/shared/errors/app-error.ts:3` |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Código citado | Logger Pino reaproveitado sem alteração | CODIGO | `src/shared/logger/index.ts:32` |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Código citado | Middleware de erro central já cobre `AppError`, `ZodError` e Prisma | CODIGO | `src/middlewares/error.middleware.ts:14` |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Código citado | Padrão de módulo usado como referência para o repository de webhooks | CODIGO | `src/modules/orders/order.repository.ts` |
| ADR-007 | docs/adrs/ADR-007-filtro-eventos-na-insercao-outbox.md | Decisão | Filtro de eventos aplicado na inserção da outbox, não no envio | TRANSCRICAO | `[09:33]`–`[09:34]` Bruno, Diego |

---

## Verificação

A tabela acima foi validada mecanicamente ao fechar a Fase 6, não por leitura:

- **Timestamps.** Todos os `[hh:mm]` distintos citados na coluna `Localização` foram
  conferidos contra `TRANSCRICAO.md` — nenhum aponta para um horário que não existe na
  reunião.
- **Caminhos.** Todos os caminhos citados na coluna `Localização` foram conferidos no disco
  com `test -f` — nenhum arquivo inventado.
- **Documentos.** Todos os caminhos da coluna `Documento` existem no repositório.
- **Exclusividade das fontes.** Toda linha tem exatamente uma `Fonte`, e nenhuma linha ficou
  com `Localização` vazia.
