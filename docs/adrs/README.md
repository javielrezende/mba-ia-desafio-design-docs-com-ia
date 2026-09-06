# Architecture Decision Records

Este diretório guarda os ADRs da feature **Sistema de Webhooks de Notificação de Pedidos**,
um arquivo por decisão, no formato `ADR-NNN-titulo-em-kebab-case.md` e na estrutura MADR:
`Status`, `Contexto`, `Decisão`, `Alternativas Consideradas` e `Consequências`.

Todas as decisões abaixo saíram da reunião técnica registrada em
[`TRANSCRICAO.md`](../../TRANSCRICAO.md); cada afirmação carrega o timestamp e o falante de
origem, e a rastreabilidade consolidada está em [`docs/TRACKER.md`](../TRACKER.md).

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](./ADR-001-padrao-outbox-mysql.md) | Padrão Outbox no MySQL para entrega de eventos de webhook | Aceito |
| [ADR-002](./ADR-002-retry-backoff-dlq.md) | Política de retry com backoff exponencial e DLQ | Aceito |
| [ADR-003](./ADR-003-autenticacao-hmac-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint | Aceito |
| [ADR-004](./ADR-004-garantia-at-least-once-event-id.md) | Garantia at-least-once com `X-Event-Id` para deduplicação | Aceito |
| [ADR-005](./ADR-005-worker-processo-separado-polling.md) | Worker em processo separado, com polling | Aceito |
| [ADR-006](./ADR-006-reuso-padroes-existentes.md) | Reuso dos padrões existentes do projeto no módulo de webhooks | Aceito |
| [ADR-007](./ADR-007-filtro-eventos-na-insercao-outbox.md) | Filtro de eventos aplicado na inserção da outbox | Aceito |

A proposta técnica que amarra essas decisões está em [`docs/RFC.md`](../RFC.md) e o
detalhamento de implementação em [`docs/FDD.md`](../FDD.md).
