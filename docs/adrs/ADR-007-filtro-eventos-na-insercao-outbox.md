# ADR-007: Filtro de eventos aplicado na inserção da outbox

## Status

Aceito

## Contexto

Marcos descreve o requisito de filtro de eventos por webhook em `[09:33]`–`[09:34]`: cada webhook
cadastrado tem uma lista de status que quer ouvir — por exemplo, "só quero saber quando vira
SHIPPED e DELIVERED". Diego pergunta diretamente onde esse filtro deve ser aplicado: "Filtra na
inserção do outbox ou na hora de mandar?" `[09:34]`. Bruno responde que deve ser na inserção — "se
nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela" `[09:34]` —
e Diego concorda.

## Decisão

O filtro de status desejados por cada webhook é avaliado no momento da inserção do evento na
`webhook_outbox`, dentro da mesma transação da mudança de status. Se nenhum webhook do customer
estiver interessado naquele status específico, nenhuma linha é inserida na outbox para esse
evento.

## Alternativas Consideradas

- **Filtrar apenas no momento do envio** (inserir sempre na outbox e decidir depois quais
  webhooks recebem cada evento) — levantada explicitamente por Diego em `[09:34]` e descartada em
  favor de filtrar na inserção: manteria a outbox como um registro completo de toda mudança de
  status, mas geraria e armazenaria permanentemente linhas que nunca seriam entregues a ninguém.

## Consequências

**Positivas:**
- A outbox permanece enxuta: só acumula eventos que efetivamente serão entregues a pelo menos um
  webhook cadastrado `[09:34]`.

**Negativas:**
- Se um customer cadastrar um novo webhook ou ajustar o filtro de um existente depois, eventos de
  mudanças de status que já ocorreram e não foram inseridos (porque nenhum webhook os queria na
  época) não podem ser recuperados retroativamente a partir da outbox — o único histórico
  remanescente dessas mudanças fica em `order_status_history`, sem o formato de evento de webhook.
