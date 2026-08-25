# ADR-002: Política de retry com backoff exponencial e DLQ

**Status:** Aceito

## Contexto

Larissa pergunta em `[09:14]` o que fazer quando o cliente está offline no momento do envio.
Diego propõe backoff exponencial com um teto de tentativas, após o qual o evento é considerado
falha permanente e vai para uma DLQ `[09:15]`. Bruno questiona se três tentativas não seriam
melhor, "mais agressivo" `[09:16]`; Diego rejeita, citando um caso real: "já tinha cliente nosso
com indisponibilidade de duas horas em manutenção planejada" `[09:16]` — três tentativas
esgotariam em 30 minutos, insuficiente para essa janela. Diego também descarta retry indefinido,
porque isso "traz o problema de evento ficar pendurado pra sempre se o cliente sumiu" `[09:15]`.
Cinco tentativas é o meio-termo fechado, cobrindo uma janela de 12 a 24 horas `[09:15]`, com
progressão 1min/5min/30min/2h/12h `[09:17]`, validada por Marcos como aceitável mesmo no pior caso
("se um cliente meu cair por 15 horas, ele já tá com problema sério dele" `[09:17]`).

Sobre a DLQ, Diego propõe uma tabela separada `webhook_dead_letter` (payload, motivo da falha,
timestamp) em vez de marcar como "failed" na própria outbox, para manter a outbox principal limpa
e servir de evidência para debug e reprocessamento `[09:18]`. O reprocessamento é manual, via
`POST /admin/webhooks/dead-letter/:id/replay` `[09:18]`–`[09:19]`.

## Decisão

Retry com backoff exponencial: 5 tentativas, com intervalos de 1 minuto, 5 minutos, 30 minutos,
2 horas e 12 horas entre a primeira falha e a última tentativa (~15h no total). Esgotadas as
tentativas, o evento é movido para a tabela `webhook_dead_letter`, com payload, motivo da falha e
timestamp. O reprocessamento é manual, via endpoint administrativo dedicado.

## Alternativas Consideradas

- **3 tentativas** — rejeitado `[09:16]` (Diego): agressivo demais frente a janelas reais de
  indisponibilidade de cliente já observadas (2h+); esgotaria em ~30 minutos.
- **Retry indefinido com backoff, sem teto** — rejeitado `[09:15]` (Diego): deixa o evento
  pendurado para sempre se o cliente sumir definitivamente.
- **Marcar falha diretamente na tabela `webhook_outbox` (sem tabela de DLQ separada)** — rejeitado
  implicitamente em `[09:18]` (Diego): manter a outbox principal "mais limpa" e ter uma tabela
  dedicada como evidência para debug e reprocessamento pesou mais.

## Consequências

**Positivas:**
- A janela de retry (~15h) cobre indisponibilidades reais já observadas em clientes, sem manter
  eventos pendurados indefinidamente.
- DLQ em tabela separada facilita auditoria e reprocessamento sem poluir a leitura da outbox
  principal `[09:18]`.

**Negativas:**
- Um cliente pode ficar até ~15 horas sem receber a notificação de um evento específico antes de
  esse evento cair definitivamente na DLQ e exigir intervenção manual via replay.
- O reprocessamento de DLQ não é automático — depende de alguém identificar a falha e acionar o
  endpoint de replay.
