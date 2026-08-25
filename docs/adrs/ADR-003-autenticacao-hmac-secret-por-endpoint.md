# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint

**Status:** Aceito

## Contexto

Sofia entra no bloco de segurança em `[09:19]` apontando o problema central: o sistema está
expondo eventos com dados de pedidos para um endpoint fora da infraestrutura da empresa, e o
cliente precisa conseguir validar que a requisição realmente veio da empresa e que ninguém
adulterou o payload no caminho. Propõe HMAC-SHA256 sobre o corpo do request, com a assinatura
enviada em um header `X-Signature` `[09:20]` — "padrão de mercado, todo cliente sério tem
biblioteca pra isso".

Em seguida, Sofia especifica que a secret precisa ser única por endpoint de webhook, não uma
secret global da plataforma: "Senão se vaza uma, vaza tudo" `[09:21]`. Diego reforça com um
precedente real: "a gente já teve cliente que vazou secret em log de aplicação dele uma vez"
`[09:22]`. A secret também precisa ser rotacionável sob demanda do cliente, com a secret antiga
permanecendo válida por 24 horas em paralelo com a nova, dando tempo de o cliente migrar seus
sistemas `[09:21]`–`[09:22]`.

## Decisão

Assinar o corpo de cada request de webhook com HMAC-SHA256, enviando a assinatura no header
`X-Signature`. Cada endpoint de webhook cadastrado recebe uma secret própria, gerada pelo sistema
e devolvida na criação — não uma secret global da plataforma. A secret é rotacionável via endpoint
próprio; ao rotacionar, a secret antiga permanece válida por 24 horas em paralelo com a nova.

## Alternativas Consideradas

- **Secret única, compartilhada globalmente pela plataforma** — rejeitada implicitamente em
  `[09:21]` (Sofia): mais simples de gerenciar (uma única secret para todos os endpoints), mas o
  vazamento de uma única secret comprometeria a autenticidade de todos os webhooks de todos os
  clientes — trade-off descartado em favor do isolamento por endpoint.

## Consequências

**Positivas:**
- Cliente consegue verificar autenticidade e integridade do payload antes de processá-lo.
- Isolamento de risco por endpoint: o vazamento de uma secret compromete apenas aquele endpoint
  específico, não os demais webhooks cadastrados (reforçado pelo precedente real citado por Diego
  em `[09:22]`).

**Negativas:**
- A rotação com grace period de 24h exige que tanto o servidor quanto a lógica de verificação
  aceitem duas secrets válidas simultaneamente durante a janela de transição — mais complexidade
  de implementação e de teste do que uma rotação "hard-cut".
- Secret por endpoint (em vez de secret global) significa que o número de segredos armazenados e
  geridos cresce proporcionalmente ao número de webhooks cadastrados por todos os clientes.
