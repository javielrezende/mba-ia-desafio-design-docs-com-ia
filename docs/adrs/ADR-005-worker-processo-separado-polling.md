# ADR-005: Worker em processo separado, com polling

## Status

Aceito

## Contexto

Ainda discutindo como o worker lê a outbox, Diego propõe polling em loop, a cada 2 segundos
`[09:09]`. Bruno pergunta se não seria melhor usar um trigger de banco para ser mais reativo
`[09:09]`; Diego rejeita, porque o MySQL não tem um listener nativo equivalente ao
`NOTIFY`/`LISTEN` do Postgres — um trigger só executa SQL, não notifica um processo externo, e
fazer isso funcionar exigiria "improvisar algo tipo escrever em arquivo ou bater num endpoint",
o que ele descreve como "esquisito" `[09:09]`. O polling de 2 segundos atende folgadamente o
requisito de latência abaixo de 10 segundos definido por Marcos em `[09:02]`.

Diego reforça, em `[09:11]`, que o worker precisa rodar como processo separado da API — "Senão se
a API reinicia, perde o worker". Larissa propõe um entry-point dedicado (`src/worker.ts`, script
`npm run worker`), no mesmo padrão de `src/server.ts` `[09:11]`. Bruno e Diego confirmam que o
worker conecta ao mesmo banco, mas precisa de seu próprio `PrismaClient` — "só não pode ser o
mesmo processo" `[09:11]`.

Sobre ordering, Larissa pergunta se o cliente recebe eventos na ordem correta quando um mesmo
pedido muda de status várias vezes em sequência rápida `[09:12]`. Diego explica que isso só é
garantido com um único worker processando em ordem de `created_at` — se o sistema escalar para
múltiplos workers em paralelo no futuro, essa garantia se perde `[09:12]`. Bruno pergunta o que
acontece se um dia precisarem escalar; Diego responde que seria necessário particionar por
`order_id` ou usar lock pessimista, mas isso é "problema do futuro, não agora" `[09:13]`. Larissa
fecha registrando isso como limitação conhecida, e Marcos confirma que os clientes nunca pediram
garantia de ordering global `[09:13]`–`[09:14]`.

## Decisão

O worker roda como um processo Node separado da API (`src/worker.ts`, script `npm run worker`),
conectado ao mesmo banco de dados por meio de um `PrismaClient` próprio, distinto do usado pela
API. A leitura da outbox é feita por polling, a cada 2 segundos.

## Alternativas Consideradas

- **Trigger de banco de dados para notificar o worker reativamente** — rejeitado em `[09:09]`
  (Bruno propôs, Diego rejeitou): MySQL não tem um mecanismo nativo de notificação de processo
  externo; a alternativa exigiria um workaround "esquisito" (escrever em arquivo ou chamar um
  endpoint a partir do trigger).

## Consequências

**Positivas:**
- O polling de 2 segundos atende com folga o requisito de latência abaixo de 10 segundos.
- Rodar como processo separado garante que o worker sobrevive a reinícios da API.

**Negativas:**
- O ordering de eventos entregues ao cliente só é garantido por `order_id` e apenas enquanto
  houver um único worker rodando `[09:12]`–`[09:13]`; escalar para múltiplos workers no futuro
  exigiria redesenho (particionamento por `order_id` ou lock pessimista), ainda não especificado.
- Manter um `PrismaClient` separado do da API significa uma segunda pool de conexões ativa contra
  o mesmo banco.
