# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| Autor | Diego (Engenheiro Sênior, Plataforma) |
| Status | Aceito |
| Data | Não há data de calendário registrada na transcrição — apenas "quinta-feira, 09:00" (`TRANSCRICAO.md:3`) |
| Revisores | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Eng. Segurança) |

---

## TL;DR

Vamos notificar clientes B2B sobre mudanças de status de pedido via webhook, em vez de exigir
polling em `GET /orders`. O evento é gravado numa tabela outbox dentro da mesma transação que
já muda o status do pedido; um worker separado, em polling, lê essa tabela e faz o envio HTTP
assinado com HMAC-SHA256, com retry exponencial e DLQ para falhas persistentes. A garantia é
*at-least-once*: o cliente deduplica pelo `X-Event-Id`.

## Contexto e problema

Clientes B2B como Atlas Comercial, MaxDistribuição e Nova Cargo hoje descobrem mudanças de
status de pedido fazendo polling em `GET /orders` (Marcos, `[09:00]`). Isso é ineficiente para
os dois lados e não atende a expectativa de "tempo real" da Atlas, definida na reunião como
latência abaixo de 10 segundos (Marcos, `[09:02]`). O escopo desta proposta é estritamente
*outbound*: o webhook sai da nossa API para o cliente; não há canal de entrada do cliente para
nós (Marcos/Sofia, `[09:02]`-`[09:03]`).

## Proposta técnica

Visão geral da arquitetura, sem detalhe de payload, schema ou contrato — isso está no FDD.

- **Outbox transacional.** A mudança de status de um pedido já ocorre dentro de uma única
  transação Prisma em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`).
  Propomos inserir o evento de webhook numa tabela `webhook_outbox` dentro dessa mesma
  transação, para que a notificação nunca fique dessincronizada do estado real do pedido
  (Diego, `[09:06]`-`[09:08]`; commit garantido junto com o resto — Bruno/Diego, `[09:40]`-`[09:41]`).
- **Worker separado, em polling.** Um processo Node independente (`src/worker.ts`, arquivo novo, a criar), com seu
  próprio `PrismaClient`, varre a outbox a cada 2 segundos e processa os eventos pendentes
  (Diego/Larissa, `[09:09]`-`[09:11]`; Bruno, `[09:29]`-`[09:30]`).
- **Retry com backoff e DLQ.** Falhas de envio (timeout, erro HTTP) são reagendadas com backoff
  exponencial em 5 tentativas; esgotadas, o evento vai para uma tabela `webhook_dead_letter`,
  reprocessável manualmente por um endpoint administrativo (Diego/Larissa, `[09:15]`-`[09:19]`).
- **Autenticação HMAC-SHA256.** Cada request de webhook carrega um `X-Signature` calculado com
  uma secret própria por endpoint (não global), rotacionável, com grace period de 24h para a
  secret antiga (Sofia, `[09:20]`-`[09:22]`).
- **Garantia at-least-once.** Cada evento carrega um `X-Event-Id` único, gerado na inserção na
  outbox; a deduplicação do lado do cliente fica a cargo dele (Diego/Larissa, `[09:24]`-`[09:26]`).
- **Configuração via API.** Cliente cadastra webhooks (URL, secret, lista de status de
  interesse) por uma API CRUD autenticada; o filtro de eventos é aplicado já na inserção na
  outbox — se nenhum webhook do customer quer aquele status, a linha nem é criada
  (Marcos/Bruno, `[09:31]`-`[09:34]`; Bruno/Diego, `[09:33]`-`[09:34]`).

## Alternativas consideradas

**Disparo síncrono do webhook dentro da transação de mudança de status.**
Descartada porque a transação de `changeStatus` já é pesada (pedido + histórico + estoque); um
HTTP call lento travaria mudanças de status de outros pedidos, e não há como fazer rollback se o
cliente estiver fora do ar (Bruno/Larissa, `[09:03]`-`[09:04]`).

**Fila dedicada (ex.: Redis Streams) em vez de outbox no MySQL.**
Descartada por exigir infraestrutura nova (ex.: cluster Redis) que o time não opera hoje —
overengineering para o tamanho do time, quando o MySQL existente já resolve o problema
(Diego/Larissa, `[09:07]`).

**Trigger de banco de dados para acordar o worker de forma reativa.**
Descartada porque o MySQL não tem um mecanismo nativo de notificação de processo externo
(equivalente ao `LISTEN`/`NOTIFY` do Postgres); um trigger só executa SQL, e simular a
notificação exigiria um workaround considerado "esquisito" pelo time (Bruno/Diego, `[09:09]`).

**Garantia de entrega exactly-once.**
Descartada por exigir coordenação dos dois lados (nossa API e o sistema do cliente) e
complexidade desproporcional ao ganho; at-least-once com `event_id` resolve a maior parte dos
casos práticos e é o padrão adotado por players de mercado como Stripe e GitHub
(Diego/Sofia, `[09:24]`-`[09:25]`).

## Questões em aberto

- **Rate limiting de envio ao cliente.** Um pico de mudanças de status (ex.: 50 pedidos
  atualizados em 1 minuto) gera 50 chamadas simultâneas para o mesmo endpoint de cliente. O
  time levantou o risco mas decidiu "observar e decidir depois", sem responsável, critério ou
  prazo definido (Diego/Larissa, `[09:38]`-`[09:39]`).
- **Notificação ao cliente em caso de falhas repetidas.** A ideia de avisar por e-mail quando um
  webhook falha várias vezes seguidas (ex.: 3 falhas) ficou fora do escopo desta fase, para ser
  reavaliada depois que o impacto real for medido (Marcos/Larissa, `[09:37]`-`[09:38]`).
- **Escalonamento para múltiplos workers em paralelo.** A garantia de ordering por `order_id`
  depende de haver um único worker rodando; paralelizar workers (com partição ou lock
  pessimista) foi explicitamente descrito como "problema do futuro, não agora"
  (Diego/Bruno, `[09:12]`-`[09:13]`).

## Impacto e riscos

**Impacto.**
- `src/modules/orders/order.service.ts` ganha uma escrita adicional (inserção na outbox) dentro
  da transação já existente de `changeStatus` — aumenta levemente o tempo dessa transação.
- Novo processo deployável em produção: o worker (`src/worker.ts`), rodando junto da API, com
  ciclo de vida e observabilidade próprios.
- `prisma/schema.prisma` recebe novas tabelas (`webhook_outbox`, `webhook_dead_letter`, e a
  configuração de webhook por customer), com migration correspondente.

**Riscos.**
- **Ordering não é global.** A garantia de ordem de entrega vale apenas por `order_id` e apenas
  enquanto houver um único worker; é uma limitação conhecida e aceita, não um bug
  (Diego/Larissa, `[09:12]`-`[09:13]`).
- **Rajada de envios sem rate limiting.** Sem uma decisão de rate limiting (ver Questões em
  aberto), um cliente pode receber dezenas de chamadas simultâneas em picos de atualização de
  pedidos (Diego/Larissa, `[09:38]`-`[09:39]`).
- **Risco de cronograma.** O prazo comprometido com a Atlas é fim de novembro (Marcos,
  `[09:45]`); a estimativa do time é de 3 sprints, incluindo a revisão de segurança da Sofia ao
  final (Larissa, `[09:46]`-`[09:47]`). Qualquer atraso na revisão de segurança pressiona esse
  prazo.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./adrs/ADR-001-padrao-outbox-mysql.md)
- [ADR-002 — Retry com backoff e DLQ](./adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003 — Autenticação HMAC-SHA256 com secret por endpoint](./adrs/ADR-003-autenticacao-hmac-secret-por-endpoint.md)
- [ADR-004 — Garantia at-least-once com X-Event-Id](./adrs/ADR-004-garantia-at-least-once-event-id.md)
- [ADR-005 — Worker em processo separado, em polling](./adrs/ADR-005-worker-processo-separado-polling.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](./adrs/ADR-006-reuso-padroes-existentes.md)
- [ADR-007 — Filtro de eventos na inserção da outbox](./adrs/ADR-007-filtro-eventos-na-insercao-outbox.md)
