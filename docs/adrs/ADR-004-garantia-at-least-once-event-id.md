# ADR-004: Garantia at-least-once com X-Event-Id para deduplicação

## Status

Aceito

## Contexto

Diego afirma em `[09:24]` que a entrega será *at-least-once*: o cliente pode eventualmente
receber o mesmo evento mais de uma vez, e precisa estar preparado para isso. Bruno pergunta como o
cliente diferencia um evento repetido de um novo `[09:25]`; Diego propõe um header `X-Event-Id`
com um UUID gerado no momento em que o evento entra na outbox, único por evento — o cliente
dedupica pelo `event_id` do seu lado.

Sofia levanta a objeção direta: "isso joga responsabilidade pro cliente" `[09:25]`. Diego não nega
— "joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim" — e argumenta que garantir
*exactly-once* exigiria coordenação dos dois lados e complexidade muito maior, enquanto
*at-least-once* com `event_id` "resolve 99% dos casos" `[09:25]`. Marcos aceita a decisão e se
compromete a documentar isso de forma destacada no portal do desenvolvedor `[09:26]`.

## Decisão

Garantir entrega *at-least-once*. Cada evento carrega um `X-Event-Id` (UUID gerado na inserção na
outbox); a deduplicação do lado do cliente, por esse identificador, fica a cargo do cliente. A
regra é documentada de forma destacada no portal de desenvolvedor.

## Alternativas Consideradas

- **Garantia *exactly-once*** — rejeitada em `[09:24]`–`[09:25]` (Diego, Sofia): exigiria
  coordenação entre os dois lados (empresa e cliente) e uma complexidade de implementação muito
  maior, para resolver um problema que *at-least-once* com `event_id` já resolve na prática.

## Consequências

**Positivas:**
- Implementação simples do lado da empresa: não exige two-phase commit nem acknowledgments
  distribuídos entre outbox, worker e cliente.
- Alinhado a um padrão de mercado já adotado por players como Stripe e GitHub, o que facilita a
  integração de clientes que já lidam com webhooks de outros fornecedores.

**Negativas:**
- A responsabilidade de deduplicação é transferida ao cliente — preocupação levantada
  explicitamente por Sofia em `[09:25]` — e depende de toda integração implementar essa lógica
  corretamente; um cliente menos maduro tecnicamente pode processar o mesmo evento duas vezes.
