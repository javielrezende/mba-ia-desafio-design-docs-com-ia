# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs — PRD, RFC, FDD, 7 ADRs e tracker de rastreabilidade — produzido a
partir da transcrição de uma reunião técnica e do código de um Order Management System em
produção. Este README documenta **como** o pacote foi produzido: as ferramentas, o workflow,
os prompts que funcionaram e os erros que precisaram ser corrigidos no caminho.

> A entrega em si está em [`docs/`](./docs). O enunciado original do desafio está no
> repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

---

## Sobre o desafio

Este repositório é a resposta a um desafio do MBA em IA da Full Cycle: transformar a
transcrição literal de uma reunião técnica de 55 minutos em um pacote completo de design
docs. O cenário é um Order Management System em produção — Node.js, TypeScript, Express,
Prisma e MySQL — que vai ganhar uma feature nova: um sistema de webhooks de notificação de
pedidos. A decisão técnica já tinha sido tomada por cinco pessoas numa call (tech lead, PM,
dois engenheiros e uma engenheira de segurança), mas nada ficou registrado além do
[`TRANSCRICAO.md`](./TRANSCRICAO.md). O que faltava era justamente a documentação: o porquê
em linguagem de negócio, a proposta submetida à revisão do time, cada decisão isolada com
suas consequências, e o detalhamento acionável o suficiente para um desenvolvedor abrir o
editor e começar.

A restrição que define o exercício não é escrever bem, é não inventar. Toda afirmação nos
documentos precisa ter origem rastreável — um `[hh:mm]` com o nome de quem falou, ou um
caminho real de arquivo do repositório — e o código da aplicação é intocável: serve de
contexto, nunca de alvo de edição. Isso muda o papel da IA no trabalho. Não dá para pedir
"escreva um PRD" e aceitar o que vier, porque a parte difícil é identificar o que a reunião
descartou, o que ela adiou e o que ela nunca discutiu, e manter essas três categorias fora
dos requisitos. O tracker de rastreabilidade existe para isso: se não é possível preencher a
coluna `Localização` de uma linha, aquela linha foi inventada e sai do documento — foi assim
que um erro real foi pego, já na fase 6.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel nesta entrega |
| --- | --- |
| **Claude Code (Opus 5)** | Ferramenta principal de produção. Leu a transcrição e o código, escreveu todos os documentos, rodou os scripts de verificação e conduziu as entrevistas guiadas. Todo commit leva o trailer `Co-Authored-By`. |
| **Skill `generate-prd-for-feature`** (própria, adaptada) | Conduziu o PRD por entrevista guiada em 11 etapas. Foi **adaptada em 8 pontos antes de rodar** — a versão crua feria a fronteira entre documentos e preenchia lacunas com números de mercado. Ver [Iterações](#iterações-e-ajustes). |
| **Subagente `fdd-architect-agent`** (próprio, adaptado) | Conduziu o FDD por entrevista sequencial. **Adaptado em 3 pontos antes de rodar**: seção de integração com o sistema existente, coluna de código `WEBHOOK_*` na matriz de erros, e obrigação de ler as fontes já produzidas antes da primeira pergunta. |
| **Subagentes `adr-analyzer`, `adr-generator`, `adr-linker`** | **Avaliados e descartados**, com o motivo registrado. São desenhados para arqueologia de código (`git log`/`git blame`) sobre um sistema já existente e não documentado; aqui a fonte é uma reunião com decisões explícitas sobre uma feature que ainda não existe no código. Descartar uma ferramenta é parte do trabalho. |
| **GitHub MCP** | Abertura e acompanhamento dos 7 PRs, com `owner`/`repo`/`base` passados como parâmetros explícitos — num fork, a interface web vem com o repositório pai pré-selecionado no dropdown, e é assim que um PR vai para o lugar errado. |
| **Scripts de verificação** (Python + `grep`, escritos na hora) | A rede de segurança mecânica: conferem cada `[hh:mm]` contra a transcrição, cada par timestamp/falante, cada caminho de arquivo contra o disco e cada citação entre aspas contra o texto literal da reunião. Não dependem da IA reler o que ela mesma escreveu. |

---

## Workflow adotado

O trabalho foi organizado em **8 fases, uma por sessão, uma branch por fase, um PR por fase**.
O motivo é prático: cada sessão começa com contexto limpo, lendo apenas o
[`plano-execucao.md`](./plano-execucao.md) e o artefato da fase anterior, em vez de arrastar
a transcrição inteira e cinco documentos por toda a conversa.

**A ordem de produção é invertida em relação à ordem de leitura.** Os documentos foram
escritos ADRs → RFC → FDD → PRD → Tracker, e não PRD → RFC → FDD. As decisões vêm primeiro
porque são o esqueleto de todo o resto: com elas fechadas, o RFC vira consolidação, o FDD vira
detalhamento e o PRD — o documento mais alto — vira tradução para linguagem de negócio, em vez
de um chute inicial que os outros documentos teriam que contradizer depois.

| Fase | Saída | PR |
| --- | --- | --- |
| 1 — Exploração | Base factual: mapa de 12 caminhos de código, 26 decisões, 12 requisitos, 8 descartes, 3 adiamentos | [#1](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/1) |
| 2 — ADRs | [`docs/adrs/`](./docs/adrs) — 7 ADRs | [#3](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/3) |
| 3 — RFC | [`docs/RFC.md`](./docs/RFC.md) | [#4](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/4) |
| 4 — FDD | [`docs/FDD.md`](./docs/FDD.md) | [#5](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/5) |
| 5 — PRD | [`docs/PRD.md`](./docs/PRD.md) | [#6](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/6) |
| 6 — Tracker | [`docs/TRACKER.md`](./docs/TRACKER.md) | [#7](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/7) |
| 7 — README do processo | Este arquivo — alimentado a cada fase, consolidado na fase 8 | — |
| 8 — Revisão final | Checklist validada item a item + verificação mecânica + este `README.md` | [#8](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/8) |

*(a correção pontual do [PR #2](https://github.com/javielrezende/mba-ia-desafio-design-docs-com-ia/pull/2)
entrou fora da numeração das fases)*

**Cada fase abre e fecha com gates fixos.** Na abertura, duas perguntas: qual skill ou
subagente usar nesta fase (a resposta pode ser "nenhum"), e aprovação do esqueleto do
documento antes de escrever uma linha. No fechamento: rodar a checklist de aceite da fase,
acumular as linhas novas de tracker, registrar o que dessa fase entra neste README, e **duas
autorizações separadas** — uma para commitar, outra, depois, para dar push e abrir o PR.
Nenhum commit foi criado sem essa confirmação explícita.

**A anotação de origem acontece na hora de escrever, não no fim.** Cada fase acumulava suas
linhas de tracker num arquivo parcial, e a fase 6 consolidava. Essa convenção funcionou
parcialmente — ver a segunda iteração abaixo, que é justamente sobre onde ela falhou.

**O código nunca foi tocado.** `git diff` entre o commit base e o final, restrito a `src/`,
`prisma/`, `tests/` e `TRANSCRICAO.md`, é vazio. A `main` avançou por 7 merges de PR e nada
mais — zero commit direto.

---

## Prompts customizados

**1. Extração da transcrição (fase 1).** O prompt que evita o resumo genérico. A instrução
que mais importa é a última: sem ela, a IA escolhe sozinha em que lista colocar um item
ambíguo, e a escolha silenciosa é exatamente onde a rastreabilidade se perde.

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

Resultado: 26 itens em DECIDIDO, 12 em REQUISITO, 8 em DESCARTADO com motivo, 3 em ADIADO e
1 EM ABERTO. As listas DESCARTADO e ADIADO são as que mais trabalharam depois — elas são a
defesa contra um requisito fantasma aparecer no PRD três fases adiante.

**2. Início do subagente do FDD (fase 4).** Um subagente de entrevista, rodado cru, pergunta
do zero — e recoleta informação que já está decidida em ADR e RFC, com o risco de contradizê-la.
Este prompt inverte isso: a entrevista vira confirmação, não coleta.

```
Inicie a entrevista conforme suas instruções (<initial_action> e <mandatory_sources>).

Contexto do projeto: a feature é o "Sistema de Webhooks de Notificação de Pedidos", já
com ADRs fechados em docs/adrs/ (ADR-001 a ADR-007), RFC em docs/RFC.md, e a extração da
transcrição em .notas/base-factual.md. TRANSCRICAO.md está na raiz do repo. Leia essas
quatro fontes integralmente antes de formular a primeira pergunta, como manda
<mandatory_sources>.

Depois de ler as fontes, envie a mensagem inicial exata definida em <initial_action> e
faça a primeira pergunta da etapa 1 do <interview_process>, já contextualizada pelo que
você leu (não pergunte o que já está decidido em ADR/RFC/transcrição - apresente e peça
confirmação/detalhamento). Pare e aguarde minha resposta antes de prosseguir.
```

A entrevista rodou em 11 etapas, cada uma fechada com resumo e confirmação antes de avançar.
Foi durante ela que apareceu o achado de código descrito abaixo.

---

## Iterações e ajustes

Cinco correções concretas, na ordem em que aconteceram.

**1. Decisão que não era decisão (fase 1).** Dois itens — TLS obrigatório na URL do webhook e
o limite de 64 KB por payload — foram discutidos e fechados na reunião como qualquer outra
decisão, e a primeira extração os classificou como DECIDIDO. Só que os próprios participantes
os desqualificam no texto: Sofia, sobre TLS, `[09:23]`: *"isso na verdade nem é decisão
arquitetural, é só uma validação no schema Zod"*; Larissa, sobre o limite, `[09:24]`: *"não
vejo como decisão arquitetural separada, é só requisito não funcional"*. Os dois foram
reclassificados como requisito não funcional. Se tivessem ficado onde estavam, o pacote teria
dois ADRs rasos, registrando como decisão de arquitetura aquilo que quem decidiu não considera
decisão de arquitetura.

**2. O tracker parcial cobria menos da metade (fase 6).** A convenção do plano mandava anotar
a origem de cada item na hora de escrevê-lo. Ao consolidar, o arquivo parcial tinha **111
linhas** — e a varredura documento a documento fechou o tracker em **229**. O delta de 118 não
é conteúdo novo: é item que já estava escrito nos documentos e não tinha sido anotado. O FDD
foi o mais subcoberto (24 linhas anotadas contra 109 itens reais; as seções 1, 2, 3, 4, 7, 8, 9
e 10 estavam sem nenhuma linha). A lição é específica e vale para quem repetir o processo: a
anotação na hora funciona bem para o que é *nomeado* (um endpoint, um código de erro, um
requisito numerado) e mal para o que é *narrado* (um objetivo, uma invariante, um critério de
aceite). O que destravou foi transformar a varredura em contagem verificável antes de escrever
qualquer linha — contar os itens identificáveis de cada seção e comparar com o parcial, seção
por seção.

**3. A citação que apontava para uma linha vazia (fase 6).** A regra do plano é que item sem
`Localização` preenchível volta para o documento de origem e é corrigido, não maquiado no
tracker. Ao conferir as citações que trazem número de linha, apareceu que o `ADR-006` citava o
logger em `src/shared/logger/index.ts:12` — e a linha 12 do arquivo é **vazia**. O
`redactPaths` está nas linhas 4–11 e o `logger` exportado, que é o que de fato se reusa, está
na 32. Corrigido para `:32` no ADR e no tracker, em commit próprio. É o tipo de erro que passa
batido em revisão por leitura: o arquivo existe, o caminho está certo, só a âncora está
deslocada. Só um script pega.

**4. O número plausível que a ferramenta empurrou (fase 5).** A skill de PRD tinha uma seção
chamada "Defaults Inteligentes", mandando preencher lacunas com valores padrão de mercado. Na
etapa de requisitos não funcionais, ela preencheria disponibilidade com 99,9 % — um número
perfeitamente razoável, que ninguém questionaria numa leitura rápida. A reunião inteira não
tem **uma única menção a uptime**. Em vez de aceitar o default, a lacuna virou pergunta, e o
PRD hoje declara com todas as letras que a meta de disponibilidade não foi definida,
descrevendo no lugar as duas garantias que de fato foram decididas: evento pendente não se
perde se o envio parar (`[09:40]`-`[09:41]`) e indisponibilidade do cliente tolerada por cerca
de 15 horas (`[09:17]`). Esse é o padrão que se repetiu com as duas ferramentas trazidas de
fora — a skill do PRD e o subagente do FDD foram **avaliados, adaptados e commitados antes de
rodar**, nunca aplicados crus.

**5. Um achado real no código, durante a entrevista do FDD (fase 4).** Não é uma correção da
IA: é algo que a checagem no código encontrou e que virou conteúdo do documento.
`src/shared/logger/index.ts` tem `redactPaths` cobrindo `password`, `passwordHash`, `token` e
`accessToken`, mas **não cobre `secret`** — e a secret HMAC de cada webhook passa por ali. Isso virou
invariante na seção de erros, item de observabilidade e risco priorizado no FDD. Pela mesma
checagem confirmou-se que o projeto não tem infraestrutura de métricas nem de tracing hoje, o
que fez essas duas subseções serem rotuladas como hipótese de infraestrutura nova, em vez de
descritas como se já existissem.

**Fase 8, a revisão final**, rodou a checklist do enunciado item a item e as verificações
mecânicas sobre o pacote inteiro: 309 pares nome↔timestamp conferidos contra a transcrição
(zero divergência), 48 timestamps distintos, 22 citações entre aspas conferidas literalmente,
todos os caminhos de código verificados no disco e todos os links relativos resolvidos. Ela
ainda produziu quatro ajustes: `Status` virou seção própria nos 7 ADRs (era campo de
metadado); as duas menções ao algoritmo de assinatura saíram do PRD, que agora não tem nenhum
nome de tecnologia; `src/worker.ts` e `src/modules/webhooks` passaram a ser marcados
explicitamente como artefatos a criar, para não serem lidos como código existente; e o
`docs/adrs/README.md`, herdado do repositório base, virou o índice real dos 7 ADRs — ele ainda
mandava nomeá-los num padrão diferente do que o desafio exige.

---

## Como navegar a entrega

Ordem de leitura sugerida, do porquê ao como:

1. **[`docs/PRD.md`](./docs/PRD.md)** — o problema, o público, o escopo e as métricas, em
   linguagem de negócio. 10 requisitos funcionais, 5 objetivos com meta, 7 riscos com plano de
   contingência.
2. **[`docs/RFC.md`](./docs/RFC.md)** — a proposta técnica submetida à revisão, com as 4
   alternativas descartadas na reunião e as 3 questões deixadas em aberto. Conciso, ~1.000
   palavras.
3. **[`docs/adrs/`](./docs/adrs)** — as 7 decisões, uma por arquivo, cada uma com contexto,
   alternativas e consequências positivas **e** negativas. Comece pelo
   [índice](./docs/adrs/README.md).
4. **[`docs/FDD.md`](./docs/FDD.md)** — o detalhamento acionável: fluxos, 8 contratos com
   payload de exemplo, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção de
   integração com 8 caminhos reais do código existente.
5. **[`docs/TRACKER.md`](./docs/TRACKER.md)** — a referência cruzada: 229 itens ligados à sua
   origem, 86 % deles a uma fala datada da reunião e o restante a um arquivo do repositório.

Para entender o processo em vez do produto: [`plano-execucao.md`](./plano-execucao.md) traz o
plano completo das 8 fases, as convenções de git e os gates — foi escrito antes de qualquer
documento e serviu de estado compartilhado entre as sessões.

A transcrição de origem é [`TRANSCRICAO.md`](./TRANSCRICAO.md), e a aplicação existente está em
[`src/`](./src), [`prisma/`](./prisma) e [`tests/`](./tests), inalterados.
