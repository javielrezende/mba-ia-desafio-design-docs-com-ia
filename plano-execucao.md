# Plano de Execução — Pacote de Design Docs (Webhooks de Notificação de Pedidos)

> Documento de trabalho. Não é entregável do desafio.
> Enunciado original e regras: `.notas/planejamento-desafio.md` (fora do git, não versionado).
> **Abra este arquivo no início de cada sessão** (`@plano-execucao.md`) e leia a seção
> [Estado atual](#estado-atual) antes de qualquer coisa.

---

## Estado atual

| Fase | Nome | Status | Branch | Saída | PR |
|---|---|---|---|---|---|
| 0 | Planejamento | ✅ concluída | `main` (working tree) | `plano-execucao.md` | — |
| 1 | Exploração (código + transcrição) | ✅ concluída (PR aberto) | `fase-1/exploracao` | `.notas/base-factual.md` | [#1](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/1) |
| 2 | ADRs | ✅ concluída (PR aberto) | `fase-2/adrs` | `docs/adrs/ADR-*.md` | [#3](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/3) |
| 3 | RFC | ✅ concluída (PR aberto) | `fase-3/rfc` | `docs/RFC.md` | [#4](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/4) |
| 4 | FDD | ✅ concluída (PR aberto) | `fase-4/fdd` | `docs/FDD.md` | [#5](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/5) |
| 5 | PRD | ✅ concluída | `fase-5/prd` | `docs/PRD.md` | — |
| 6 | Tracker | ⬜ pendente | `fase-6/tracker` | `docs/TRACKER.md` | — |
| 7 | README do processo | 🔄 contínua | — (entra na branch da fase corrente) | `README.md` | — |
| 8 | Revisão final | ⬜ pendente | `fase-8/revisao-final` | checklist validada | — |

> A Fase 1 não altera arquivos versionados (a saída vai para `.notas/`, que é auto-ignorado).
> Ela ainda assim ganha branch e PR, para carregar o commit do `plano-execucao.md` e manter
> o histórico do processo — é justamente esse rastro que alimenta o README da Fase 7.

**Próximo passo:** Fase 5 fechada e commitada em `fase-5/prd`, aguardando autorização
para push e abertura do PR. Depois do merge: `git checkout main && git pull --ff-only origin main`,
e iniciar a Fase 6 (Tracker) em uma sessão nova.

**Pré-requisito da Fase 1 — resolvido:** servidor `github` do MCP promovido a escopo
`user` nesta sessão (reaproveitando o PAT já existente em `mba-ia-desafio-refactor-projects-skill`),
`claude mcp add` executado e `claude mcp list` confirma `github` conectado. Os nomes exatos
das ferramentas MCP ainda serão confirmados na hora de abrir o PR.

**Decisões pendentes (não resolver sozinho):** nenhuma no momento.

**Decisões já resolvidas nesta sessão:**
- [x] `plano-execucao.md` — versionado, commitado no PR da Fase 1.
- [x] `planejamento-desafio.md` — **não** versionado; movido para `.notas/planejamento-desafio.md` (fora do git).

---

## Fatos já levantados (Fase 0)

**Repositório.** Node.js + TypeScript, Express, Prisma + MySQL, Vitest.

```
src/
├── app.ts, server.ts
├── config/            database.ts, env.ts
├── middlewares/       auth, error, request-logger, validate
├── modules/
│   ├── auth/          controller, routes, schemas, service
│   ├── customers/     controller, repository, routes, schemas, service
│   ├── orders/        controller, repository, routes, schemas, service, order.status.ts
│   ├── products/      controller, repository, routes, schemas, service
│   └── users/         controller, repository, routes, schemas, service
├── routes/index.ts
└── shared/
    ├── errors/        app-error.ts, http-errors.ts, index.ts
    ├── http/          response.ts
    └── logger/        index.ts
prisma/  schema.prisma, seed.ts, migrations/20260519182739_init/
tests/   auth.test.ts, orders.test.ts, helpers/factories.ts, setup.ts
```

Padrão arquitetural evidente: **controller → service → repository**, schemas Zod por módulo,
erros centralizados em `src/shared/errors/`, resposta HTTP padronizada em `src/shared/http/response.ts`,
logger próprio em `src/shared/logger/`. Não há nenhum arquivo de fila, evento, worker ou webhook.

**Transcrição.** `TRANSCRICAO.md`, 323 linhas, formato `[hh:mm] Nome: fala`.
Participantes: **Larissa** (Tech Lead), **Marcos** (PM), **Bruno** (Eng. Pleno, Pedidos),
**Diego** (Eng. Sênior, Plataforma), **Sofia** (Eng. Segurança).
O formato dos timestamps é diretamente compatível com a coluna `Localização` do Tracker.

**Estado inicial de `docs/`.** `PRD.md`, `RFC.md`, `FDD.md` e `TRACKER.md` existem como
stubs de 3 linhas; `docs/adrs/` contém apenas um `README.md` explicativo. Nada foi produzido ainda.

---

## Convenções que valem para todas as fases

1. **Idioma:** todos os documentos em português (PT-BR), acompanhando a transcrição e o enunciado.
2. **Código é intocável.** Nenhuma edição em `src/`, `prisma/`, `tests/`, `package.json`,
   `tsconfig*`, `.eslintrc`, `docker-compose.yml`, `TRANSCRICAO.md`. Leitura apenas.
3. **Zero invenção.** Toda afirmação em documento entregável precisa ter origem em
   `TRANSCRICAO.md` (timestamp + falante) ou em um caminho real de `src/`/`prisma/`.
   Se não dá para preencher a coluna `Localização` do Tracker, a frase sai do documento.
4. **Anotar a origem na hora de escrever**, não no fim. Cada fase produz linhas de tracker
   junto com o documento, acumuladas em `.notas/tracker-parcial.md`. A Fase 6 só consolida.
5. **Fronteira entre documentos** (conteúdo duplicado = algo no lugar errado):
   - PRD → *por que e o quê* (negócio)
   - RFC → *o que propomos e por quê*, alternativas, questões abertas (arquitetura, 2–4 páginas)
   - ADR → *por que decidimos exatamente assim* (uma decisão por arquivo)
   - FDD → *como construir em detalhe* (implementação)
6. **Notas de trabalho fora do git:** diretório `.notas/` na raiz, contendo um `.gitignore`
   com `*` (auto-ignorado). O `.gitignore` do projeto não é tocado e `git status` fica limpo.
7. **Duas perguntas em toda fase** (ver blocos "Gate" abaixo): a skill a usar, e o incremento do README.
8. **Uma fase por sessão, uma branch por fase, um PR por fase.** Ver
   [Convenções de Git](#convenções-de-git). Nenhum `git commit` acontece sem sua
   confirmação explícita das alterações feitas (ver [Gate de fechamento](#gate-padrão-de-fechamento-de-fase),
   passo 5), e nada é enviado ao GitHub sem sua autorização explícita separada para
   push/PR — são duas confirmações distintas. Todo PR aponta para `main` **deste**
   repositório — nunca para o fork de origem.
9. **Ao fim da fase:** atualizar a tabela [Estado atual](#estado-atual) e o bloco de notas
   da fase neste arquivo.

---

## Convenções de Git

### Fatos verificados do repositório

- Remote único: `origin` → `git@github.com:javielrezende/mba-ia-desafio-design-docs-com-ia.git`
- **Branch padrão: `main`** (não `master`). Todo PR aponta para `main` **deste** repositório.
- Não existe remote `upstream` configurado — não há como um `git push` acidental chegar
  no repositório de origem do fork.
- `gh` (GitHub CLI) **não está instalado** nesta máquina.
- **GitHub MCP disponível** (`api.githubcopilot.com/mcp/`, transporte HTTP, PAT).
  ⚠️ Hoje está registrado com escopo **local de outro projeto**
  (`mba-ia-desafio-refactor-projects-skill`), então **não carrega nesta sessão**.
  Precisa ser promovido a escopo `user` (ou adicionado a este projeto) e o Claude Code
  reiniciado. Ver [Habilitar o GitHub MCP](#habilitar-o-github-mcp-neste-projeto).

### Regra de ouro do fork

> **O PR nunca aponta para o repositório do qual este projeto foi forkado.**
> Base sempre `javielrezende/mba-ia-desafio-design-docs-com-ia` → `main`.

Isso não é paranoia: numa página de *compare* do GitHub aberta a partir de um fork, o
**repositório base vem pré-selecionado como o repositório pai**. Se o PR for aberto pela
interface web sem trocar esse dropdown, ele vai para o lugar errado por padrão.

Daí a ordem de preferência dos métodos abaixo: os dois primeiros passam `owner`/`repo`/`base`
como **parâmetros explícitos**, onde não existe default para errar. Só o método web tem
estado pré-selecionado — e por isso carrega um passo de conferência obrigatório.

### Uma branch por fase

Criada **no início da fase**, sempre a partir de `main` atualizada:

```bash
git checkout main
git pull --ff-only origin main
git checkout -b fase-N/<slug>
```

Padrão do nome: `fase-<N>/<slug>` — `fase-1/exploracao`, `fase-2/adrs`, `fase-3/rfc`,
`fase-4/fdd`, `fase-5/prd`, `fase-6/tracker`, `fase-8/revisao-final`.

Correções pontuais depois de uma fase fechada usam `fix/<slug>` (ex.: `fix/tracker-timestamps`).

**Nunca commitar direto em `main`.** `main` só avança por merge de PR.

### Padrão de commit

[Conventional Commits](https://www.conventionalcommits.org), em português, imperativo,
assunto ≤ 72 caracteres, sem ponto final.

```
<tipo>(<escopo>): <assunto>

<corpo opcional: o porquê, não o quê>
```

**Tipos usados neste projeto** (entrega puramente documental):

| Tipo | Quando |
|---|---|
| `docs` | qualquer conteúdo de `docs/` ou do `README.md` — o caso dominante aqui |
| `chore` | arquivos de processo (`plano-execucao.md`, `.notas/`) |
| `fix` | correção de erro factual, link quebrado, rastreabilidade inválida |
| `refactor` | reorganização de documento sem mudar o conteúdo |

**Escopos:** `prd`, `rfc`, `fdd`, `adr`, `tracker`, `readme`, `plano`.

Exemplos reais do que vamos escrever:

```
docs(adr): adicionar ADR-001 sobre padrão outbox no MySQL
docs(adr): registrar decisões 002 a 006 discutidas na reunião
docs(rfc): consolidar proposta técnica com alternativas descartadas
docs(fdd): detalhar contratos públicos e matriz de erros WEBHOOK_*
docs(tracker): mapear rastreabilidade de PRD, RFC, FDD e ADRs
fix(fdd): corrigir caminho de arquivo inexistente na seção de integração
chore(plano): registrar fechamento da fase 2
```

**Granularidade:** um commit por unidade coerente de conteúdo, não um commit gigante
por fase. Na Fase 2, por exemplo, faz sentido um commit por ADR ou por grupo temático —
o histórico é parte do que o README da Fase 7 vai narrar.

Todo commit criado por IA leva o trailer:

```
Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
```

### Abertura do PR

**Acontece no fim da fase, e só depois da sua autorização explícita.** Sem "ok" seu,
nada é enviado — nem `push`, nem PR.

Sequência:

```bash
git status                 # conferir que só entrou o que devia
git diff --stat main       # revisar o escopo antes de enviar
git push -u origin fase-N/<slug>
```

**Método 1 (preferido) — GitHub MCP.** A abertura do PR vira uma chamada de ferramenta com
os campos explícitos, sem nenhum default herdado do fork:

| Parâmetro | Valor — sempre este |
|---|---|
| `owner` | `javielrezende` |
| `repo` | `mba-ia-desafio-design-docs-com-ia` |
| `base` | `main` |
| `head` | `fase-N/<slug>` |
| `title` | `docs(<escopo>): <resumo da fase>` |
| `body` | conteúdo de `.notas/pr-fase-N.md` |

O `git push` da branch continua por linha de comando (`git push -u origin fase-N/<slug>`);
o MCP entra na abertura do PR e no acompanhamento depois — revisões, comentários, status.

> Os **nomes exatos** das ferramentas do MCP serão confirmados na abertura da Fase 1, quando
> o servidor estiver de fato carregado na sessão. Não assumir nomes antes disso.

**Método 2 — `gh`**, caso você prefira instalar o CLI (`sudo apt install gh && gh auth login`):

```bash
gh pr create \
  --repo javielrezende/mba-ia-desafio-design-docs-com-ia \
  --base main \
  --head fase-N/<slug> \
  --title "docs(<escopo>): <resumo da fase>" \
  --body-file .notas/pr-fase-N.md
```

**Método 3 (fallback) — web**, se nenhum dos dois estiver disponível:

```
https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/compare/main...fase-N/<slug>?expand=1
```

⚠️ **Conferência obrigatória antes de clicar em "Create pull request":** o campo
*base repository* precisa estar em `javielrezende/mba-ia-desafio-design-docs-com-ia`
e o *base* em `main`. Se aparecer o nome do repositório original, troque o dropdown.

### Template do corpo do PR

Escrito em `.notas/pr-fase-N.md` antes de abrir:

```markdown
## Fase N — <nome da fase>

### O que entra
- <arquivos produzidos, com caminho>

### Origem do conteúdo
- Transcrição: <faixas de timestamp relevantes>
- Código: <caminhos consultados>

### Decisões tomadas nesta fase
- <decisão + quem decidiu: usuário ou seguindo o enunciado>

### Checklist da fase
- [x] <itens da sub-checklist da fase, marcados com evidência>

### Fora de escopo deste PR
- <o que ficou para fases seguintes>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### Habilitar o GitHub MCP neste projeto

Passo a passo, **antes da Fase 1** — o servidor precisa estar carregado na sessão:

```bash
# escopo user = disponível em todos os projetos, inclusive este
claude mcp add --scope user --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer <SEU_PAT>"
```

O PAT já existe na configuração do projeto `mba-ia-desafio-refactor-projects-skill`,
dentro de `~/.claude.json`, e pode ser reaproveitado.

**Alterações de MCP só valem após reiniciar o Claude Code.** Confirmar com `/mcp` que o
servidor `github` aparece como conectado antes de abrir a Fase 1.

### Merge

Depois do PR aprovado por você, `main` é atualizada e a próxima fase parte dela:

```bash
git checkout main && git pull --ff-only origin main
```

A branch da fase pode ser apagada após o merge — o PR preserva o histórico.

---

## Gate padrão de abertura de fase

**Passo 0 — branch.** Antes de qualquer edição, sair de `main` e criar a branch da fase
(ver [Uma branch por fase](#uma-branch-por-fase)). Se a sessão começar com alterações
pendentes na working tree, resolver isso antes de seguir.

Depois, perguntar ao usuário:

> **Skill desta fase:** você quer aplicar alguma skill pessoal ou do curso para
> este documento (`/nome-da-skill`, um arquivo, um template)? Também é válido não passar nada —
> nesse caso sigo o formato padrão de mercado + o que o enunciado exige.

E antes de produzir o conteúdo final, apresentar o **plano do documento** (esqueleto de seções +
o que vai em cada uma, com as origens) para aprovação. Só depois escrever.

## Gate padrão de fechamento de fase

1. Rodar a sub-checklist de critérios de aceite da fase (abaixo, item por item).
2. Acrescentar as linhas novas em `.notas/tracker-parcial.md`.
3. **Perguntar sobre o README** (Fase 7, contínua):
   > O que dessa fase entra no README do processo? Sugiro registrar: *(prompts usados,
   > iterações/correções, decisões tomadas)*. Confirma, ajusta ou deixa para depois?
4. Atualizar [Estado atual](#estado-atual) e o bloco da fase neste arquivo.
5. **Pedir confirmação antes de commitar:** com as alterações da fase já escritas (mas
   ainda não commitadas), apresentar o que foi feito — arquivos alterados/criados,
   `git status`/`git diff --stat` — e perguntar:
   > As alterações da Fase N estão prontas (lista acima). Posso commitar?
   Sem o seu "pode", nenhum `git commit` acontece. Isso vale a cada commit da fase,
   não só no fechamento — se a fase gerar mais de um commit, cada um passa por essa
   confirmação antes de ser criado.
6. **Commitar** na branch da fase, seguindo o [padrão de commit](#padrão-de-commit).
7. **Pedir autorização para enviar:**
   > A Fase N está fechada e commitada em `fase-N/<slug>`. Posso fazer o push e abrir
   > o PR contra `javielrezende/mba-ia-desafio-design-docs-com-ia` → `main`?
   Sem o seu "pode", nada sai da máquina. Nem `push`, nem PR.
8. Após autorização: `git push`, abrir o PR via **GitHub MCP** com `owner`/`repo`/`base`
   explícitos (ver [Abertura do PR](#abertura-do-pr)) usando o
   [template](#template-do-corpo-do-pr), e registrar o número do PR na coluna `PR`
   da tabela de [Estado atual](#estado-atual).
9. Após o seu merge: voltar para `main`, `pull --ff-only`, e a próxima fase parte daí.

---

# Fase 1 — Exploração (código + transcrição)

**Objetivo.** Construir a base factual única que todas as fases seguintes consomem, para não
reler transcrição e código do zero em cada sessão.

**Entradas.** `TRANSCRICAO.md`, `src/`, `prisma/schema.prisma`, `tests/`.

**Saída.** `.notas/base-factual.md`, com quatro blocos:

### 1.1 Mapa do código (o que a feature vai tocar)
- Ciclo de vida do pedido: `src/modules/orders/order.status.ts` — estados e transições válidas.
- `src/modules/orders/order.service.ts` — o método de mudança de status e a transação
  (orders + order_status_history + stock_quantity). Ponto de enxerto do outbox.
- `src/modules/orders/order.repository.ts` — padrão de acesso a dados.
- `src/shared/errors/app-error.ts` e `http-errors.ts` — hierarquia de erro a reutilizar.
- `src/shared/http/response.ts` — envelope de resposta HTTP.
- `src/shared/logger/index.ts` — logger a reutilizar na observabilidade.
- `src/middlewares/` — auth, validate, error handler: como uma rota nova se pluga.
- `prisma/schema.prisma` + migration inicial — convenções de tabela, tipos, enums, índices.
- `src/routes/index.ts` — registro de rotas.
- `tests/orders.test.ts`, `tests/helpers/factories.ts` — padrão de teste vigente.
> Para cada item: caminho exato + 1 linha do que existe + 1 linha de como a feature encosta nele.
> Caminhos precisam ser verificados no disco — nada de arquivo inventado.

### 1.2 Extração estruturada da transcrição — quatro listas separadas
| Lista | O que é |
|---|---|
| **DECIDIDO** | decisões fechadas na reunião, com quem decidiu e timestamp |
| **REQUISITO** | requisitos funcionais e não funcionais explícitos (meta: ≥ 8 FR) |
| **DESCARTADO** | ideias colocadas na mesa e rejeitadas, **com o motivo do descarte** |
| **ADIADO** | itens empurrados para fase futura, com a justificativa |
Toda linha carrega `[hh:mm] Nome`. Um item **não pode** aparecer em duas listas.

### 1.3 Restrições e números citados
Prazos, SLAs, limites, timeouts, quantidades — só os ditos na reunião, com timestamp.

### 1.4 Mapa decisão → ADR candidato
Rascunho da numeração dos ADRs, cobrindo as 6 decisões principais do enunciado.

**Prompt dirigido sugerido para a extração** (evita o "resumo genérico" que a IA tende a produzir):

```
Leia TRANSCRICAO.md inteira. Não resuma.
Produza QUATRO listas mutuamente exclusivas — DECIDIDO, REQUISITO, DESCARTADO, ADIADO.
Regras:
- toda linha termina com [hh:mm] Nome de quem disse;
- DESCARTADO só entra se houver no texto o motivo do descarte — cite-o;
- ADIADO só entra se alguém disser explicitamente que fica para depois/fase 2;
- se algo foi levantado e nunca concluído, vai para uma quinta lista: EM ABERTO;
- não infira, não complete lacunas, não use conhecimento geral sobre webhooks.
Se um item couber em duas listas, pare e me pergunte em vez de escolher.
```

**Checklist de fechamento da Fase 1**
- [ ] ≥ 8 requisitos funcionais na lista REQUISITO
- [ ] ≥ 2 itens em DESCARTADO, cada um com motivo
- [ ] ≥ 2 itens em ADIADO/EM ABERTO
- [ ] Todo caminho de arquivo citado existe (verificado no disco)
- [ ] As 6 decisões principais do enunciado aparecem em DECIDIDO — ou está anotado qual não aparece e por quê

---

# Fase 2 — ADRs (`docs/adrs/`)

**Objetivo.** Fechar as decisões primeiro; elas são o esqueleto dos demais documentos.

**Entrada.** `.notas/base-factual.md` (blocos 1.2 e 1.4) + trechos de código citados.

**Saída.** 5 a 8 arquivos `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`.

**Cobertura obrigatória** (≥ 5 das 6):
1. Padrão Outbox no MySQL
2. Política de retry com backoff e DLQ
3. Autenticação HMAC-SHA256 com secret por endpoint
4. Garantia at-least-once com `X-Event-Id`
5. Worker em processo separado em polling
6. Reuso dos padrões existentes do projeto ← **este é o ADR que cita o código**

**Formato (MADR)** — por arquivo: `Status`, `Contexto`, `Decisão`,
`Alternativas Consideradas` (≥ 1 real, com trade-off), `Consequências` (positivas **e** negativas).

**⚠️ Gate de decisão desta fase — trazer ao usuário, não resolver sozinho:**
> As decisões secundárias (formato do payload, timeouts, headers, versionamento do payload…)
> viram ADRs adicionais (indo para 7–8 ADRs) ou ficam registradas só no FDD (ficando em 6)?
> Vou apresentar as candidatas concretas que saíram da transcrição e discutimos caso a caso.

**Como evitar ADR raso.** Cada `Contexto` precisa conter a tensão real da reunião (quem
defendeu o quê e por quê), não uma descrição neutra do problema. Cada `Consequência negativa`
precisa ser um custo que o time vai pagar de verdade — se não dá para nomear, a decisão não
foi entendida.

**Checklist de fechamento**
- [ ] 5–8 arquivos, nomeados `ADR-NNN-titulo-em-kebab-case.md`
- [ ] Todo ADR tem as 5 seções
- [ ] ≥ 5 das 6 decisões principais cobertas
- [ ] ≥ 1 ADR cita arquivo/módulo/classe real do código
- [ ] Nenhum ADR registra algo que a transcrição descartou ou adiou

---

# Fase 3 — RFC (`docs/RFC.md`)

**Objetivo.** Consolidar a proposta técnica **em cima dos ADRs já fechados**, no tom de
documento submetido à revisão. Conciso: 2 a 4 páginas.

**Entrada.** ADRs da Fase 2 + listas DESCARTADO / ADIADO / EM ABERTO da Fase 1.

**Seções obrigatórias.** Metadados (autor, status, data, revisores = os 5 participantes) ·
TL;DR · Contexto e problema · Proposta técnica (visão geral, sem detalhe de implementação) ·
Alternativas consideradas (≥ 2 reais e descartadas na reunião, cada uma com o trade-off) ·
Questões em aberto (≥ 2 pontos não decididos/adiados) · Impacto e riscos ·
Decisões relacionadas (links para os ADRs).

**Regra de fronteira.** Se uma frase do RFC descreve payload, campo de tabela, header
ou código de erro → ela pertence ao FDD. O RFC para no "o que propomos e por quê".

**Checklist de fechamento**
- [ ] Todas as seções presentes
- [ ] ≥ 2 alternativas descartadas, cada uma com o trade-off que motivou o descarte
- [ ] ≥ 2 questões em aberto vindas da reunião
- [ ] ≥ 2 links para ADRs do pacote (caminhos relativos válidos)
- [ ] Extensão entre 2 e 4 páginas
- [ ] Nenhum trecho duplicando o nível de detalhe do FDD

---

# Fase 4 — FDD (`docs/FDD.md`)

**Objetivo.** O documento acionável: um dev pega e começa a codar.

**Entrada.** ADRs + RFC + mapa do código (bloco 1.1).

**Seções obrigatórias.** Contexto e motivação técnica · Objetivos técnicos · Escopo e exclusões ·
Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ) ·
Contratos públicos · Matriz de erros `WEBHOOK_*` · Estratégias de resiliência (timeouts,
retries, backoff, fallback) · Observabilidade (métricas, logs, tracing) · Dependências e
compatibilidade · Critérios de aceite técnicos · Riscos e mitigação ·
**Integração com o sistema existente** (seção específica do desafio).

**Integração com o sistema existente — ≥ 4 caminhos reais**, cada um com *como* o módulo
de webhooks encosta nele. Candidatos já mapeados:
`src/modules/orders/order.service.ts` (enfileirar na outbox dentro da transação existente) ·
`src/modules/orders/order.status.ts` (quais transições geram evento) ·
`src/shared/errors/app-error.ts` + `http-errors.ts` (reuso da hierarquia de erro) ·
`prisma/schema.prisma` (novas tabelas seguindo as convenções vigentes) ·
`src/shared/logger/index.ts` · `src/routes/index.ts` · `src/middlewares/`.
Todo caminho é verificado no disco antes de entrar no documento.

**Checklist de fechamento**
- [ ] Todas as seções presentes
- [ ] ≥ 4 endpoints HTTP com payload de request **e** response e status codes
- [ ] Matriz de erros com prefixo `WEBHOOK_`
- [ ] ≥ 4 caminhos reais de arquivo na seção de integração — todos existem
- [ ] Observabilidade cobre métricas **e** logs **e** tracing
- [ ] Nada aqui contradiz um ADR

---

# Fase 5 — PRD (`docs/PRD.md`)

**Objetivo.** Camada de produto/negócio. Vem por último entre os grandes: com RFC, FDD e
ADRs prontos, é consolidação — mas escrita na altura de negócio, não repetindo o técnico.

**Seções obrigatórias.** Resumo e contexto · Problema e motivação · Público-alvo e cenários de uso ·
Objetivos e métricas de sucesso · Escopo (incluso e fora) · Requisitos funcionais ·
Requisitos não funcionais · Decisões e trade-offs principais · Dependências ·
Riscos e mitigação · Critérios de aceitação · Estratégia de testes e validação.

**Checklist de fechamento**
- [ ] ≥ 8 requisitos funcionais rastreados à reunião
- [ ] ≥ 1 objetivo com métrica **e meta quantitativa**
- [ ] "Fora de escopo" com ≥ 2 itens explicitamente descartados/adiados na reunião
- [ ] "Riscos" com ≥ 2 riscos contendo probabilidade, impacto e mitigação
- [ ] Linguagem de negócio — sem payload, sem nome de tabela, sem header

---

# Fase 6 — Tracker (`docs/TRACKER.md`)

**Objetivo.** Varrer os documentos prontos e montar a referência cruzada. É a rede de
segurança contra alucinação: item sem `Localização` preenchível é item inventado — **volta
para o documento de origem e é corrigido ou removido**, não é maquiado no tracker.

**Formato obrigatório:**
`| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`

- **ID**: `PRD-FR-01`, `PRD-NFR-02`, `RFC-ALT-01`, `RFC-OPEN-02`, `FDD-CONTRATO-03`,
  `FDD-ERRO-01`, `FDD-INT-02`, `ADR-002`…
- **Fonte**: `TRANSCRICAO` ou `CODIGO`
- **Localização**: `[hh:mm] Nome` ou caminho real de arquivo

**Método.** Base = `.notas/tracker-parcial.md`, acumulado nas fases 2–5. Depois, varredura
documento a documento para pegar o que escapou. Validação final por script simples:
conferir que todo timestamp citado existe em `TRANSCRICAO.md` e que todo caminho existe no disco.

**Checklist de fechamento**
- [ ] Formato de tabela exato
- [ ] ≥ 80% dos itens identificáveis dos documentos têm linha
- [ ] ≥ 70% das linhas com `Fonte = TRANSCRICAO` e timestamp válido `[hh:mm] Nome`
- [ ] ≥ 5 linhas com `Fonte = CODIGO` e caminho real
- [ ] Todo timestamp existe na transcrição; todo caminho existe no repo

---

# Fase 7 — README do processo (`README.md`) — **contínua**

**Regra.** Não é uma fase no fim: é alimentada **a cada fase**, no gate de fechamento. O rascunho
vive em `.notas/readme-processo.md` e só é consolidado no `README.md` da raiz na Fase 8 —
assim o README da raiz não fica meio-pronto no meio do caminho. O conteúdo do enunciado atual
do README pode virar um link/seção de referência.

**Estrutura obrigatória.** Sobre o desafio (1–2 parágrafos — *proponho os textos para aprovação*) ·
Ferramentas de IA utilizadas · Workflow adotado · Prompts customizados (≥ 2 em bloco de código) ·
Iterações e ajustes (≥ 2 correções concretas, com o que a IA errou) · Como navegar a entrega.

**O que anotar a cada fase, na hora** (senão vira ficção no fim):
prompt que funcionou · o que a IA produziu raso ou errado e como foi corrigido ·
decisões que você tomou nos gates · quantas iterações até o documento fechar.

**Checklist de fechamento**
- [ ] Todas as seções presentes
- [ ] ≥ 1 ferramenta de IA listada com o papel dela
- [ ] ≥ 2 prompts customizados em bloco de código
- [ ] ≥ 2 iterações/ajustes concretos e verdadeiros
- [ ] Ordem de leitura sugerida com caminhos que existem

---

# Fase 8 — Revisão final

**Objetivo.** Passar a checklist de critérios de aceite item por item e fechar as
verificações de consistência global.

1. **Checklist do enunciado**, um a um, marcando com evidência (arquivo + seção).
2. **Verificação mecânica** (comandos, sem depender de leitura da IA):
   - todo caminho `src/...` / `prisma/...` citado nos docs existe no disco;
   - todo timestamp `[hh:mm]` citado existe em `TRANSCRICAO.md`;
   - `docs/adrs/` tem entre 5 e 8 arquivos no padrão de nome;
   - links relativos entre documentos resolvem.
3. **Leitura cruzada anti-contradição:** nada nos docs contradiz transcrição ou código;
   nada descartado/adiado na reunião aparece como requisito.
4. **Leitura de fronteira:** RFC não repete o FDD; PRD não repete o RFC; ADR não duplica ADR.
5. Consolidar `README.md` a partir de `.notas/readme-processo.md`.
6. `git status`: só os arquivos entregáveis modificados; `src/`, `prisma/`, `tests/`,
   `TRANSCRICAO.md` e configurações intactos.
7. **Auditoria do histórico Git:**
   - `git log --oneline main..fase-8/revisao-final` — mensagens no padrão acordado;
   - `git log --stat main` sobre as fases já mergeadas — nenhum commit tocou `src/`,
     `prisma/`, `tests/` ou `TRANSCRICAO.md`:
     ```bash
     git log --format='%h %s' --name-only main | grep -E '^(src|prisma|tests)/|^TRANSCRICAO.md' || echo "OK: código intocado"
     ```
   - todos os PRs das fases foram para `javielrezende/...` → `main`, nenhum para o repo pai;
   - nenhum commit direto em `main` fora dos merges de PR.

---

## Notas por fase

*(preencher ao fechar cada fase: o que foi produzido, o que precisou de retrabalho, decisões tomadas)*

### Fase 0 — Planejamento
- Exploração leve do repo para ancorar o plano em fatos (estrutura de `src/`, stubs em `docs/`, formato e participantes da transcrição).
- Decisões do usuário: estado compartilhado em `plano-execucao.md` na raiz; notas de trabalho fora do git em `.notas/`.
- Ordem de produção definida: exploração → ADRs → RFC → FDD → PRD → Tracker, com README alimentado continuamente.

### Fase 1 — Exploração (código + transcrição)
- Pré-requisito resolvido nesta sessão: GitHub MCP registrado em escopo `user`
  (`claude mcp add`), reaproveitando o PAT do outro projeto; `claude mcp list` confirma
  conectado. As ferramentas MCP em si só carregam após reiniciar o Claude Code — a
  abertura do PR desta fase fica para a próxima sessão.
- Decisões de versionamento: `plano-execucao.md` será commitado neste PR;
  `planejamento-desafio.md` fica fora do git, movido para `.notas/planejamento-desafio.md`
  (criando `.notas/` com `.gitignore` = `*` pela primeira vez no projeto).
- Sem skill externa — seguido o formato padrão já definido no plano para o documento de exploração.
- Produzido `.notas/base-factual.md`: mapa de 12 caminhos de código, 26 itens em
  DECIDIDO, 12 em REQUISITO, 8 em DESCARTADO, 3 em ADIADO, 1 em EM ABERTO, tabela de
  restrições/números, e mapa decisão→ADR cobrindo as 6 decisões obrigatórias do
  enunciado + candidatas secundárias para o gate da Fase 2.
- Retrabalho: dois itens (TLS obrigatório, limite de payload de 64KB) foram
  reclassificados de DECIDIDO para REQUISITO (NFR) após notar que os próprios
  participantes da reunião os desqualificam como decisão arquitetural (Sofia `[09:23]`,
  Larissa `[09:24]`). Detalhe registrado em `.notas/readme-processo.md`.
- Extração feita diretamente (sem subagente), para manter controle fino sobre a regra
  de parar e perguntar em casos ambíguos.
- **Correção pontual pós-fase** (`fix/gate-confirmar-commit`, PR #2): o usuário pediu
  que, de agora em diante, todo `git commit` passe por confirmação explícita antes de
  acontecer (não só o push/PR). O commit que registrava essa regra (`66c25d9`) foi
  pushado depois que o usuário já tinha feito o merge do PR #1, então ficou de fora de
  `main` — corrigido via cherry-pick numa branch `fix/`.

### Fase 2 — ADRs
- O usuário trouxe três subagentes prontos (`.claude/agents/adr-analyzer.md`,
  `adr-generator.md`, `adr-linker.md`), de outro projeto, pedindo avaliação de encaixe
  antes de rodar. Após leitura completa dos três, avaliação foi de não usá-los: são
  desenhados para arqueologia de código sobre um codebase já existente e não documentado
  (git log/git blame, scoring por Step-0 categories), enquanto nossa fonte é uma
  transcrição de reunião com decisões já explícitas para uma feature que ainda não existe
  no código; além disso o `adr-generator` grava em MADR de 7 seções sem atribuição a
  falante/timestamp, incompatível com o formato de 5 seções e a rastreabilidade
  `[hh:mm] Nome` já fixados neste plano. Decisão do usuário, com essa avaliação como
  base: escrever os ADRs diretamente a partir de `.notas/base-factual.md`, no mesmo
  método sem-subagente validado na Fase 1. Detalhe em `.notas/readme-processo.md`.
- Gate de decisões secundárias (§1.4 do base-factual.md): das candidatas D6+D25, D16,
  D14/D18+D17, D21+D24, o usuário promoveu apenas **D16** a ADR próprio (`ADR-007` —
  filtro de eventos aplicado na inserção da outbox, não no envio). As demais ficam só no
  FDD (Fase 4).
- Produzidos 7 ADRs em `docs/adrs/`: ADR-001 (outbox MySQL), ADR-002 (retry+DLQ),
  ADR-003 (HMAC-SHA256 por endpoint), ADR-004 (at-least-once + X-Event-Id), ADR-005
  (worker separado em polling), ADR-006 (reuso de padrões existentes — cita
  `src/shared/errors/app-error.ts:3`, `src/shared/logger/index.ts:12`,
  `src/middlewares/error.middleware.ts:14`, `src/modules/orders/order.repository.ts`),
  ADR-007 (filtro na inserção da outbox). Todas as 6 decisões obrigatórias do enunciado
  cobertas, mais essa uma secundária.
- Verificação mecânica: todos os `[hh:mm]` citados nos 7 ADRs existem em
  `TRANSCRICAO.md`; todos os caminhos de código citados existem no disco.
- Linhas de tracker desta fase acumuladas em `.notas/tracker-parcial.md`.

### Fase 3 — RFC
- Sem skill externa — mesmo formato-padrão fixado no plano (§Fase 3), plano do documento
  apresentado e aprovado antes da escrita.
- Autor do RFC definido como Diego (Eng. Sênior, Plataforma — proponente da maior parte
  do desenho técnico na transcrição); revisores = os 5 participantes. Campo `Data` sem
  data de calendário na transcrição — registrado explicitamente em vez de inventado
  (`TRANSCRICAO.md:3` só traz "quinta-feira, 09:00"). Detalhe em `.notas/readme-processo.md`.
- Produzido `docs/RFC.md`: TL;DR, contexto (R1/R2), proposta técnica (D2-D4, D7-D9, D11,
  D15, D16), 4 alternativas descartadas com trade-off (DC1, DC2, DC3, DC6), 3 questões em
  aberto (EA1, AD1, AD2), impacto e riscos (incluindo risco de cronograma, R12/D26), e
  links para os 7 ADRs.
- Verificação mecânica: os ~30 timestamps citados conferem contra `TRANSCRICAO.md`
  (`grep`), `src/modules/orders/order.service.ts:126` confere no disco, os 7 links
  relativos para `docs/adrs/*.md` resolvem. Sem retrabalho.
- Linhas de tracker desta fase (`RFC-CTX`, `RFC-PROP`, `RFC-ALT`, `RFC-OPEN`, `RFC-RISK`)
  acumuladas em `.notas/tracker-parcial.md`.

### Fase 4 — FDD
- Skill desta fase: o usuário trouxe um subagente pronto
  (`.claude/agents/fdd-architect-agent.md`) que conduz o FDD por entrevista sequencial.
  Diferente da Fase 2 (onde os 3 subagentes de ADR foram descartados), este tinha bom
  encaixe, mas a avaliação achou 3 gaps contra os critérios de fechamento desta fase:
  faltava seção dedicada de integração com o sistema existente, a matriz de erros não
  tinha coluna de código `WEBHOOK_*`, e o processo de entrevista não instruía ler
  base-factual/ADRs/RFC/transcrição antes de perguntar. Adaptado antes de rodar (seção
  11 nova, coluna de erro, bloco `<mandatory_sources>`) e commitado em `fase-4/fdd`
  antes da entrevista (`def5619`). Detalhe completo em `.notas/readme-processo.md`.
- Entrevista rodada em 11 etapas (via `Agent`/`SendMessage`, o orquestrador principal
  repassando pergunta-resposta entre o subagente e o usuário), cada uma fechada com
  resumo e confirmação explícita antes de avançar.
- Achado real durante a entrevista (verificado no código, não hipótese): `redactPaths`
  em `src/shared/logger/index.ts` não cobre o campo `secret` — vira invariante,
  observabilidade e risco priorizado no FDD. Também confirmado que o projeto não tem
  hoje infraestrutura de métricas nem tracing — essas duas subseções de Observabilidade
  ficaram rotuladas como Hipótese (infra nova), Logs como reuso 1:1 do Pino existente.
- Decisão de gate: EA1 (rate limiting, ficou em aberto na reunião) tratado como
  explicitamente fora de escopo desta entrega, com risco documentado na seção 10 — não
  virou seção própria de "questões em aberto" no FDD.
- Critério adotado para os ~10 pontos "Hipótese" do documento (nomenclatura de rota,
  condição de erro específica, etc.): toda Hipótese se ancora numa decisão/requisito
  real já confirmado na transcrição, só o detalhe fino foi extrapolado e sempre
  confirmado pelo usuário durante a entrevista — tratado como decisão de gate desta
  sessão (mesmo padrão da promoção de D16 a ADR-007 na Fase 2), não como invenção.
- Verificação mecânica ao fechar: os 8 caminhos de código da seção de Integração
  existem no disco (`test -f`); todos os `[hh:mm]` citados no FDD existem em
  `TRANSCRICAO.md` (`grep`); nada contradiz os 7 ADRs.
- Linhas de tracker desta fase (`FDD-CONTRATO`, `FDD-ERRO`, `FDD-INT`) acumuladas em
  `.notas/tracker-parcial.md`.

### Fase 5 — PRD
- Skill desta fase: o usuário trouxe `.claude/skills/generate-prd-for-feature/SKILL.md`,
  que conduz o PRD por entrevista guiada em 11 etapas com esqueleto de saída fixo. Segunda
  aplicação do padrão "avaliar antes de usar" (a primeira foi a Fase 4). O esqueleto cobria
  as 12 seções do enunciado, mas a avaliação achou 3 gaps contra os critérios desta fase:
  seção de arquitetura no esqueleto (fere a fronteira PRD/RFC e o critério "linguagem de
  negócio"), "Defaults Inteligentes" mandando preencher números com padrão de mercado
  (fere zero invenção e a Fase 6), e ausência de instrução para ler as fontes já
  produzidas antes de entrevistar. Adaptada em 8 pontos e commitada antes da entrevista
  (`10b0f56`). Detalhe completo em `.notas/readme-processo.md`.
- Decisão do usuário no gate de adaptação: remover a seção "Arquitetura e abordagem" do
  esqueleto (em vez de mantê-la enxuta), deixando o PRD exatamente com as 12 seções do
  enunciado.
- Entrevista rodada nas 11 etapas, conduzida diretamente (a skill carrega no contexto do
  orquestrador, diferente do subagente da Fase 4, que exigia `Agent`/`SendMessage`).
- Caso concreto de zero invenção nesta fase: a reunião não tem nenhuma menção a uptime. Em
  vez de aceitar o default de 99,9% da skill, a lacuna foi levada ao usuário e o documento
  declara explicitamente que a meta não foi definida, descrevendo no lugar as duas
  garantias realmente decididas (`[09:40]`-`[09:41]` e `[09:17]`).
- Produzido `docs/PRD.md` (463 linhas): 5 objetivos com meta, 6 itens fora de escopo,
  10 requisitos funcionais (`PRD-FR-01` a `PRD-FR-10`), 17 pontos de requisito não
  funcional em 8 categorias, 6 decisões com trade-off, 6 dependências, 7 riscos com
  contingência, 14 critérios de aceitação e 5 tipos de teste.
- Apenas 2 Hipóteses no documento inteiro, ambas em testes (ensaio de rajada e piloto com
  cliente), ambas ancoradas em risco ou compromisso real e escolhidas pelo usuário.
  Compliance e acessibilidade ficaram registradas como não discutidas na reunião, em vez
  de omitidas ou preenchidas com texto genérico.
- Export do PRD em JSON, oferecido pela skill: recusado pelo usuário, para não manter dois
  artefatos com o mesmo conteúdo em sincronia.
- Verificação mecânica ao fechar: 46 timestamps distintos e 254 pares de timestamp/falante
  conferidos contra `TRANSCRICAO.md` sem divergência; os 5 caminhos de arquivo citados
  existem no disco; varredura por termos de fronteira (`WEBHOOK_`, tabelas, headers,
  verbos HTTP, `outbox`, `polling`, nomes de tecnologia) sem nenhuma ocorrência no PRD.
- 61 linhas de tracker desta fase acumuladas em `.notas/tracker-parcial.md`
  (`PRD-OBJ`, `PRD-ESC`, `PRD-FR`, `PRD-NFR`, `PRD-DEC`, `PRD-DEP`, `PRD-RISCO`,
  `PRD-AC`, `PRD-TESTE`), todas com localização validada.
