# ADR-001: Padrão Outbox no MySQL para entrega de eventos de webhook

## Status

Aceito

## Contexto

Larissa abre a primeira pergunta de arquitetura em `[09:03]`: disparar o webhook de forma síncrona
dentro do `order.service`, ou usar algum tipo de fila/outbox? Bruno se posiciona contra o
disparo síncrono em `[09:04]`: a transação de mudança de status já é pesada — atualiza `orders`,
insere em `order_status_history` e decrementa `stock_quantity` — e um `HTTP call` lento no meio
dela travaria mudanças de status de outros pedidos. Bruno também aponta que, se o cliente estiver
fora do ar, não há como fazer rollback da mudança de status por causa disso.

Diego propõe o padrão outbox em `[09:06]`: dentro da mesma transação SQL que atualiza `orders` e
`order_status_history`, uma linha é inserida em `webhook_outbox`; um worker separado lê essa
tabela e dispara as chamadas HTTP. Se a transação principal commitou, o evento foi registrado; se
deu rollback, o evento some junto — "não tem inconsistência possível" `[09:06]`.

Em `[09:51]`–`[09:52]`, já ao final da reunião, Bruno levanta uma última dúvida: o evento guarda o
payload já renderizado, ou só o `order_id`, renderizando na hora do envio? Larissa decide pelo
snapshot na inserção — "se o pedido mudar depois, o evento ainda reflete o estado de quando o
status mudou. Senão tem caso esquisito" `[09:52]` — e Diego concorda.

## Decisão

Inserir uma linha na tabela `webhook_outbox` dentro da mesma transação SQL do `changeStatus` do
pedido — se a inserção falhar, toda a transação sofre rollback (`[09:40]`–`[09:41]` Bruno, Diego).
A tabela tem índice em `status` e `created_at` (`[09:08]` Diego); arquivamento de linhas entregues
fica fora do escopo desta feature. O payload do evento é renderizado ("snapshot") no momento da
inserção, não recalculado no momento do envio (`[09:51]`–`[09:52]` Larissa, Diego, Bruno).

## Alternativas Consideradas

- **Disparo síncrono dentro da transação de `order.service`** — rejeitado `[09:03]`–`[09:04]`
  (Bruno, Larissa): travaria mudanças de status de outros pedidos se o cliente estiver lento, e
  não há como fazer rollback se o cliente estiver fora do ar.
- **Fila dedicada (ex.: Redis Streams)** — rejeitado `[09:07]` (Diego, Larissa): exigiria subir
  infraestrutura nova para um time pequeno; overengineering quando o outbox no MySQL já existente
  resolve.

## Consequências

**Positivas:**
- Atomicidade garantida entre a mudança de status e o registro do evento — sem estado
  inconsistente possível entre `orders` e `webhook_outbox` `[09:06]`.
- Nenhuma infraestrutura nova: reaproveita o MySQL já em produção `[09:07]`.

**Negativas:**
- O snapshot do payload pode ficar "desatualizado" em relação ao estado atual do pedido: se o
  pedido mudar novamente logo depois, o evento já enviado reflete o estado de quando a mudança de
  status ocorreu, não o estado mais recente — trade-off aceito conscientemente em `[09:52]`.
- Arquivamento de linhas já entregues foi declarado fora do escopo desta feature `[09:08]`; sem
  uma rotina de limpeza definida, a tabela cresce indefinidamente até que isso seja tratado.
