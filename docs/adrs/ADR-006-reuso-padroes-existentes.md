# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

## Status

Aceito

## Contexto

Bruno assume o bloco de estrutura de código em `[09:27]` e propõe seguir o padrão já existente na
base: cada domínio é um módulo em `src/modules` com `controller`, `service`, `repository`,
`routes` e `schemas` — o módulo de webhooks replicaria essa mesma estrutura
(`src/modules/orders/order.repository.ts` como referência de estilo). Sobre erros, Bruno propõe
reaproveitar a hierarquia existente — a classe `AppError` (`src/shared/errors/app-error.ts:3`) e
subclasses específicas como `InsufficientStockError` e `InvalidStatusTransitionError` — usando o
prefixo `WEBHOOK_` nos novos códigos de erro (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`,
`WEBHOOK_SECRET_REQUIRED`), no mesmo padrão dos códigos já existentes `[09:28]`–`[09:29]`. Confirma
também que o logger Pino (`src/shared/logger/index.ts:32`) e o middleware de erro central
(`src/middlewares/error.middleware.ts:14`, que já trata `AppError`, `ZodError` e erros do Prisma)
cobrem o módulo novo sem qualquer alteração `[09:29]`.

Diego confirma que o worker abre seu próprio `PrismaClient` (mesmo banco, mesma `DATABASE_URL`,
mas instância separada por ser outro processo Node) `[09:29]`–`[09:30]`, e Larissa fecha o bloco
reforçando reuso máximo dos padrões do projeto `[09:30]`.

Mais à frente, ao discutir a integração com `order.service.ts`, Bruno levanta a tensão de design:
"vai me obrigar a passar um repository do webhook pro `OrderService` ou uma função de 'enqueue
event'?" `[09:41]`. Propõe uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que
recebe o `tx` client da transação corrente, em vez de o `OrderService` depender do repository
inteiro do módulo webhooks. Diego endossa: "boa, função pura recebendo o `tx`. Não precisa injetar
repository inteiro" `[09:41]`–`[09:42]`.

## Decisão

O novo módulo `src/modules/webhooks`, a ser criado, segue exatamente o padrão dos módulos existentes
(`controller`/`service`/`repository`/`routes`/`schemas`). Os erros do módulo herdam de `AppError`
com prefixo `WEBHOOK_` nos códigos. O logger Pino e o middleware de erro central são reaproveitados
sem alteração. A integração com `order.service.ts` acontece por meio de uma função
`publishWebhookEvent(tx, order, fromStatus, toStatus)`, chamada de dentro da transação existente,
em vez de injetar um repository do módulo webhooks no `OrderService`. O worker usa um `PrismaClient`
próprio, apontando para o mesmo banco.

## Alternativas Consideradas

- **Injetar o repository do módulo webhooks inteiro no `OrderService`** — rejeitada em `[09:41]`
  (Bruno levantou, Diego rejeitou): acoplaria o `OrderService` a toda a superfície do repository de
  webhooks, quando a única necessidade real é registrar um evento dentro da transação corrente; uma
  função pura recebendo o `tx` mantém o acoplamento mínimo.

## Consequências

**Positivas:**
- Onboarding rápido para quem já conhece o projeto: mesma estrutura de módulo, mesmo tratamento de
  erro, mesmo logger — nada disso precisa ser reaprendido para o módulo de webhooks.
- Middlewares de erro e resposta HTTP já cobrem os erros do novo módulo sem qualquer mudança em
  `src/middlewares/error.middleware.ts`.

**Negativas:**
- Acoplamento ao estilo do projeto-base: se o padrão geral de módulos, erros ou logger mudar no
  restante do projeto, o módulo de webhooks precisa acompanhar essa mudança.
- A função `publishWebhookEvent(tx, ...)` cria uma dependência implícita: toda chamada em
  `order.service.ts` que mude o status de um pedido precisa conhecer e invocar esse hook
  corretamente — um novo caminho de mudança de status que esqueça de chamá-la não gera evento, sem
  que isso seja capturado por tipo ou compilação.
