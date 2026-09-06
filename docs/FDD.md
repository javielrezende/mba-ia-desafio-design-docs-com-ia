# FDD: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** 2026-08-25
**Responsável:** Larissa (Tech Lead)

---

## 1. Contexto e motivação técnica

O problema técnico real que esta feature resolve é o acoplamento ineficiente por polling: clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) hoje descobrem mudanças de status de pedido consultando repetidamente `GET /orders`, o que é ineficiente para os dois lados e não atende a expectativa de "tempo real" definida como latência abaixo de 10 segundos (R1, R2; `[09:00]`-`[09:02]` Marcos). Junto a isso, há a necessidade de garantia de entrega/idempotência (o cliente precisa conseguir confiar que recebeu o evento, mesmo diante de falhas transitórias) e de rastreabilidade das tentativas de notificação (histórico de entregas, motivo de falhas).

No encaixe com o HLD existente, a mudança de status de um pedido já ocorre dentro de uma transação Prisma única em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`). A proposta técnica (RFC) insere o evento de webhook numa tabela `webhook_outbox` dentro dessa mesma transação, garantindo que a notificação nunca fique dessincronizada do estado real do pedido (ADR-001). Um processo Node separado (`src/worker.ts`, arquivo novo, a criar), com `PrismaClient` próprio, faz polling a cada 2 segundos e processa os eventos pendentes, assinando cada request com HMAC-SHA256 (ADR-003, ADR-005).

O escopo é estritamente *outbound*: a API só envia notificações para o cliente, não há canal de entrada do cliente para a plataforma (D1; `[09:02]`-`[09:03]` Marcos/Sofia).

**Atores:**
- Clientes B2B - destinatários dos webhooks, donos das URLs cadastradas.
- Usuários operadores autenticados via JWT do sistema - cadastram e gerenciam webhooks de um customer (`customer_id` vem do body/path, não do JWT) (D14).
- Usuários com role `ADMIN` - únicos autorizados a executar replay de eventos em DLQ (D17, D18).

**Suposições:**
- O time é pequeno e não opera infraestrutura de fila dedicada (Redis), por isso o MySQL existente é reaproveitado via padrão outbox (DC2; `[09:07]` Diego/Larissa).

**Restrições:**
- Sem garantia de ordering global de eventos - só por `order_id`, e apenas enquanto houver um único worker rodando (D6; ADR-005).
- Garantia de entrega é *at-least-once*, não *exactly-once*; deduplicação é responsabilidade do cliente via `X-Event-Id` (D11; ADR-004).
- Prazo comprometido com a Atlas para fim de novembro, estimativa de 3 sprints incluindo revisão de segurança (R12; `[09:45]`-`[09:47]`).

---

## 2. Objetivos técnicos

* Latência de entrega abaixo de 10s no caso normal, com pior caso dominado pelo intervalo de polling de 2s do worker (medida: R2; D3; `[09:02]`, `[09:09]`-`[09:10]`).
* Atomicidade garantida: se a transação de `changeStatus` commitar, o evento existe na outbox (quando há webhook interessado); se ela sofrer rollback, o evento não existe (invariante: D19; ADR-001).
* Garantia at-least-once com deduplicação delegada ao cliente via `X-Event-Id`, UUID único gerado na inserção na outbox (invariante: D11; ADR-004).
* Ordering garantido apenas por `order_id` e apenas em regime single-worker - limitação conhecida, documentada, não uma garantia global (invariante/limitação: D6; ADR-005).
* Resiliência a indisponibilidade do cliente por até ~15h (5 tentativas, backoff 1min/5min/30min/2h/12h) antes de mover o evento para DLQ (medida: D7; ADR-002).
* Payload limitado a 64KB, com falha explícita (sem truncamento) quando ultrapassado (medida: R9; DC7).
* Timeout de 10s por chamada HTTP do worker, tratado como falha e reagendado para retry (medida: D21).

---

## 3. Escopo e exclusões

**Incluído**
* CRUD de configuração de webhook: `POST`, `PATCH`, `DELETE`, `GET` por customer (D15; R3, R4).
* Filtro de eventos por webhook aplicado no momento da inserção na outbox, não no envio (D16; ADR-007).
* Histórico de entregas `GET /webhooks/:id/deliveries` (últimos 100 registros) (R6).
* Endpoint admin de replay de DLQ `POST /admin/webhooks/dead-letter/:id/replay`, exigindo role `ADMIN` e registrando log de auditoria de quem executou (R7; D17).
* Rotação de secret via endpoint próprio, com grace period de 24h para a secret antiga (R10; D10).
* Validações: URL obrigatoriamente `https` (R8) e limite de payload de 64KB com falha explícita (R9).
* Pipeline completo outbox → worker (polling 2s) → HTTP assinado com HMAC-SHA256 → retry/backoff → DLQ (ADR-001, ADR-002, ADR-003, ADR-005).

**Excluído**
* Disparo síncrono do webhook dentro da transação de `order.service` (DC1).
* Fila dedicada (Redis Streams ou similar) (DC2).
* Trigger de banco de dados para acordar o worker (DC3).
* Retry com apenas 3 tentativas ou retry indefinido sem teto (DC4, DC5).
* Garantia exactly-once (DC6).
* Truncamento de payload acima do limite (DC7).
* Dashboard visual para o cliente acompanhar webhooks (DC8).
* Notificação por e-mail em falhas repetidas - adiado para fase futura, após medição de impacto (AD1).
* Múltiplos workers em paralelo / particionamento por `order_id` - adiado, "problema do futuro" (AD2).
* Arquivamento de linhas entregues na outbox (~30 dias) - adiado (AD3).
* Rate limiting de envio ao cliente - explicitamente fora de escopo desta entrega; risco documentado na seção 10 (EA1).

---

## 4. Fluxos detalhados e diagramas

**Fluxo principal**
1. Uma operação de negócio dispara `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), abrindo uma transação Prisma única (`this.prisma.$transaction`).
2. Dentro da transação: valida a transição via `canTransition`, debita/repõe estoque (`shouldDebitStock`/`shouldReplenishStock`), atualiza `order.status`, insere em `orderStatusHistory`.
3. Ainda dentro da mesma transação, `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` (D20; ADR-006).
4. `publishWebhookEvent` busca os webhooks ativos do `customer_id` do pedido cujo `statusFilter` inclui o `to_status` (D16; ADR-007). Se nenhum webhook estiver interessado, nenhuma linha é inserida e o fluxo termina aqui, sem erro.
5. Para cada webhook interessado, o payload é renderizado como snapshot no momento da inserção (D25; ADR-001): `event_id` (UUID), `event_type: "order.status_changed"`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` (D22).
6. Insere uma linha por webhook interessado em `webhook_outbox` (id UUID, status `pending`) (D24).
7. A transação commita. Se a inserção na outbox falhar, toda a transação (incluindo a mudança de status) sofre rollback (D19).
8. O processo separado `src/worker.ts` faz polling a cada 2s, buscando eventos `pending` mais antigos, ordenados por `created_at` (D3; ADR-005).
9. Para cada evento, o worker calcula HMAC-SHA256 do payload com a secret ativa do webhook, monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`) e faz `POST` para a URL cadastrada, com timeout de 10s (D21, D23; ADR-003).
10. Sucesso (resposta `2xx`): o evento é marcado como entregue; o registro fica disponível em `GET /webhooks/:id/deliveries`.
11. Falha (timeout, erro HTTP não-2xx, erro de rede): o evento é reagendado conforme o backoff (1m/5m/30m/2h/12h) (D7; ADR-002).
12. Após 5 tentativas falhas, o evento é movido para `webhook_dead_letter` (payload, motivo, timestamp) (D8; ADR-002).

**Fluxos alternativos e exceções**
* CRUD de configuração: a criação gera a `secret` no servidor e a devolve apenas na resposta de criação (R3); edição, remoção e listagem são feitas por `customer_id` (D15).
* Rotação de secret: nova secret é gerada, a antiga permanece válida por 24h em paralelo com a nova (D10; ADR-003).
* Replay de DLQ: `ADMIN` chama `POST /admin/webhooks/dead-letter/:id/replay`, o evento volta para `webhook_outbox` como `pending`, e a ação é logada para auditoria (D8, D17).
* Payload acima de 64KB no momento do envio pelo worker: falha permanente, sem retry, movido direto para DLQ (**Hipótese** - não há fonte explícita na transcrição/ADRs sobre em qual momento essa checagem ocorre; a decisão adotada foi validar no envio, não na inserção).

**Diagramas (opcional)**
* Não produzidos nesta versão do FDD; recomenda-se um diagrama de sequência cobrindo os passos 1-12 do fluxo principal antes da implementação.

---

## 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

**1. Cadastrar webhook**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `POST /webhooks` (nomenclatura de rota - **Hipótese**, sem fonte explícita na transcrição/ADRs sobre o nome exato da rota; segue padrão flat de `order.routes.ts`)
* **Método:** POST
* **Semântica de status/headers:**
  * `Authorization: Bearer <JWT>` obrigatório, qualquer role autenticada (D18)
  * `201` - criado, `secret` retornada apenas nesta resposta (R3)
  * `400 WEBHOOK_INVALID_URL` - URL não é `https` (R8)
  * `400 WEBHOOK_INVALID_STATUS_FILTER` - `statusFilter` vazio ou inválido (**Hipótese**, nome de código seguindo convenção `WEBHOOK_`)
  * `401 UNAUTHORIZED` - sem JWT válido

**Exemplo de requisição**
```json
{
  "customerId": "b3f1e2a0-0000-4000-8000-000000000001",
  "url": "https://cliente.example.com/webhooks/pedidos",
  "statusFilter": ["SHIPPED", "DELIVERED"]
}
```

**Exemplo de resposta**
```json
{
  "id": "9c0a1234-0000-4000-8000-000000000002",
  "customerId": "b3f1e2a0-0000-4000-8000-000000000001",
  "url": "https://cliente.example.com/webhooks/pedidos",
  "secret": "whsec_9f8a7b6c5d4e",
  "statusFilter": ["SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-08-24T13:00:00.000Z"
}
```

---

**2. Listar webhooks de um customer**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `GET /webhooks?customerId=...` (Hipótese de nomenclatura, confirmada na entrevista)
* **Método:** GET
* **Semântica de status/headers:**
  * `Authorization: Bearer <JWT>` obrigatório, qualquer role (D18)
  * `200` - lista paginada, reaproveitando `PaginatedResponse<T>` (`src/shared/http/response.ts`); `secret` nunca retornada nesta rota (Hipótese confirmada)
  * `400 VALIDATION_ERROR` - query inválida
  * `401 UNAUTHORIZED`

**Exemplo de requisição**
```json
{
  "query": { "customerId": "b3f1e2a0-0000-4000-8000-000000000001", "page": 1, "pageSize": 20 }
}
```

**Exemplo de resposta**
```json
{
  "data": [
    {
      "id": "9c0a1234-0000-4000-8000-000000000002",
      "customerId": "b3f1e2a0-0000-4000-8000-000000000001",
      "url": "https://cliente.example.com/webhooks/pedidos",
      "statusFilter": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-08-24T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

---

**3. Editar webhook**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `PATCH /webhooks/:id`
* **Método:** PATCH
* **Semântica de status/headers:**
  * `Authorization: Bearer <JWT>` obrigatório, qualquer role (D18)
  * `200` - atualizado, sem `secret` no corpo
  * `400 WEBHOOK_INVALID_URL` / `400 WEBHOOK_INVALID_STATUS_FILTER`
  * `401 UNAUTHORIZED`
  * `404 WEBHOOK_NOT_FOUND`

**Exemplo de requisição**
```json
{
  "url": "https://cliente.example.com/webhooks/v2",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Exemplo de resposta**
```json
{
  "id": "9c0a1234-0000-4000-8000-000000000002",
  "customerId": "b3f1e2a0-0000-4000-8000-000000000001",
  "url": "https://cliente.example.com/webhooks/v2",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-08-24T13:00:00.000Z"
}
```

---

**4. Remover webhook**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `DELETE /webhooks/:id`
* **Método:** DELETE
* **Semântica de status/headers:**
  * `Authorization: Bearer <JWT>` obrigatório, qualquer role (D18)
  * `204` - removido, sem corpo
  * `401 UNAUTHORIZED`
  * `404 WEBHOOK_NOT_FOUND`

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{}
```

---

**5. Rotacionar secret**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `POST /webhooks/:id/rotate-secret` (Hipótese de nomenclatura, confirmada na entrevista)
* **Método:** POST
* **Semântica de status/headers:**
  * `Authorization: Bearer <JWT>` obrigatório, qualquer role (D18)
  * `200` - nova secret gerada; secret antiga permanece válida até `previousSecretValidUntil` (D10)
  * `401 UNAUTHORIZED`
  * `404 WEBHOOK_NOT_FOUND`

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{
  "id": "9c0a1234-0000-4000-8000-000000000002",
  "secret": "whsec_novo_valor_gerado",
  "previousSecretValidUntil": "2026-08-25T13:00:00.000Z"
}
```

---

**6. Histórico de entregas**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `GET /webhooks/:id/deliveries`
* **Método:** GET
* **Semântica de status/headers:**
  * `Authorization: Bearer <JWT>` obrigatório, qualquer role (D18)
  * `200` - até 100 registros mais recentes (R6)
  * `400 VALIDATION_ERROR` - query inválida
  * `401 UNAUTHORIZED`
  * `404 WEBHOOK_NOT_FOUND`

**Exemplo de requisição**
```json
{
  "query": { "page": 1, "pageSize": 20 }
}
```

**Exemplo de resposta**
```json
{
  "data": [
    {
      "id": "d41d0001-0000-4000-8000-000000000003",
      "eventId": "0a120002-0000-4000-8000-000000000004",
      "status": "success",
      "httpStatusCode": 200,
      "attemptNumber": 1,
      "requestedAt": "2026-08-24T13:00:02.000Z",
      "respondedAt": "2026-08-24T13:00:02.340Z",
      "responseTimeMs": 340
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 47, "totalPages": 3 }
}
```

---

**7. Replay de evento em DLQ (admin)**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `POST /admin/webhooks/dead-letter/:id/replay`
* **Método:** POST
* **Semântica de status/headers:**
  * `Authorization: Bearer <JWT>` obrigatório, exige role `ADMIN` via `requireRole('ADMIN')` (D17)
  * `200` - evento recolocado na outbox como `pending`, ação logada com `replayedBy` (D17)
  * `401 UNAUTHORIZED` - sem JWT
  * `403 FORBIDDEN` - role diferente de `ADMIN`
  * `404 WEBHOOK_DEAD_LETTER_NOT_FOUND` - id de dead-letter inexistente (Hipótese de nome de código)

**Exemplo de requisição**
```json
{}
```

**Exemplo de resposta**
```json
{
  "deadLetterId": "f2a00005-0000-4000-8000-000000000005",
  "outboxEventId": "3b910006-0000-4000-8000-000000000006",
  "status": "requeued",
  "replayedBy": "op-99110007-0000-4000-8000-000000000007",
  "replayedAt": "2026-08-24T15:00:00.000Z"
}
```

---

**8. Entrega do webhook ao cliente (contrato outbound)**
* **Tipo:** http_endpoint
* **Assinatura/Rota:** `POST` para a `url` cadastrada pelo cliente (D22, D23; ADR-003)
* **Método:** POST
* **Semântica de status/headers:**
  * `X-Event-Id` - UUID único do evento, gerado na inserção na outbox, usado para deduplicação pelo cliente (D11, D22)
  * `X-Signature` - HMAC-SHA256 do corpo, calculado com a secret do endpoint (ADR-003)
  * `X-Timestamp` - ISO 8601, momento do envio, permite ao cliente detectar replay attack (D23)
  * `X-Webhook-Id` - id do cadastro de webhook, para clientes com múltiplos endpoints (D23, Sofia `[09:44]`)
  * `Content-Type: application/json`
  * Qualquer resposta `2xx` do cliente é sucesso; qualquer outro código, timeout ou erro de rede é falha e entra em retry (D7)
* **Limites:** payload máximo 64KB (R9); timeout de 10s (D21)

**Exemplo de requisição**
```json
{
  "event_id": "0a120002-0000-4000-8000-000000000004",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-24T13:00:00.000Z",
  "order_id": "77ab0008-0000-4000-8000-000000000008",
  "order_number": "PED-00123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "b3f1e2a0-0000-4000-8000-000000000001",
  "total_cents": 458000
}
```

**Exemplo de resposta (esperada do cliente)**
```json
{
  "received": true
}
```

---

## 6. Erros, exceções e fallback

**Matriz de erros previstos e tratamentos**
| Código | Condição | Tratamento | Notas |
| :--- | :--- | :--- | :--- |
| WEBHOOK_NOT_FOUND | `id` de webhook inexistente em GET/PATCH/DELETE/rotate-secret/deliveries | HTTP 404 | Nome citado literalmente por Bruno (D12, `[09:28]`) |
| WEBHOOK_INVALID_URL | `url` cadastrada não é `https` | HTTP 400, rejeitado antes de persistir | D12 (nome), R8 (regra) |
| WEBHOOK_SECRET_REQUIRED | Worker tenta assinar/enviar evento cujo webhook está sem secret válida | Falha permanente, direto para DLQ, sem retry | Nome citado por Bruno (D12, `[09:28]`); condição de disparo é Hipótese |
| WEBHOOK_INVALID_STATUS_FILTER | `statusFilter` vazio ou com valor fora do enum `OrderStatus` | HTTP 400 na criação/edição | Hipótese (nome inventado seguindo convenção `WEBHOOK_`) |
| WEBHOOK_DEAD_LETTER_NOT_FOUND | Replay aponta para `id` inexistente em `webhook_dead_letter` | HTTP 404 | Hipótese (distingue do 404 de config de webhook) |
| WEBHOOK_PAYLOAD_TOO_LARGE | Payload renderizado ultrapassa 64KB no momento do envio pelo worker | Falha permanente, sem retry, direto para DLQ | R9 (limite); Hipótese sobre o momento da checagem |
| WEBHOOK_DELIVERY_TIMEOUT | Chamada HTTP do worker não responde em 10s | Falha transitória, entra em retry/backoff | D21 |
| WEBHOOK_DELIVERY_FAILED | Resposta HTTP do cliente não é `2xx`, ou erro de rede/DNS/TLS | Falha transitória, entra em retry/backoff | D7 |

* **Estratégias de resiliência:** timeout de 10s por chamada HTTP do worker (D21); retry com backoff exponencial, 5 tentativas, 1min/5min/30min/2h/12h (D7; ADR-002); circuit breaker **não decidido, fora desta entrega** - verificado que não há implementação no código nem menção nos ADRs/transcrição.
* **Política de fallback:** esgotadas as 5 tentativas (ou falha permanente como `WEBHOOK_PAYLOAD_TOO_LARGE`/`WEBHOOK_SECRET_REQUIRED`), o evento é movido para `webhook_dead_letter` com payload, motivo da falha e timestamp (D8). Reprocessamento é sempre manual, via `POST /admin/webhooks/dead-letter/:id/replay`, restrito a `ADMIN`, com log de auditoria (D17). Erros de validação/recursos não encontrados no CRUD são tratados pelo middleware central existente (`src/middlewares/error.middleware.ts`), sem necessidade de alteração (ADR-006).
* **Invariantes:**
  * Se a transação de `changeStatus` commitar, todo evento elegível (com pelo menos um webhook interessado) existe na outbox; se ela sofrer rollback, nenhum evento existe (D19).
  * Payload da outbox nunca é truncado; ultrapassar 64KB é sempre erro explícito (DC7, R9).
  * Payload é snapshot imutável desde a inserção - nunca recalculado no envio (D25).
  * Secret de webhook nunca aparece em log em texto plano - achado real do código: `redactPaths` (`src/shared/logger/index.ts`) precisa ser estendido para cobrir o campo `secret`.
  * Garantia at-least-once nunca degrada para "at-most-once" silenciosamente: todo evento é entregue ou vai para DLQ, nunca é descartado sem rastro (D11, D8).

---

## 7. Observabilidade

**Métricas** (candidatas, infraestrutura nova - Hipótese, não há Prometheus/OpenTelemetry no projeto hoje)
* `webhook_outbox_pending_count` (gauge) - eventos pendentes na outbox, monitora acúmulo/backlog do worker.
* `webhook_outbox_events_published_total` (counter, labels `event_type`, `to_status`) - eventos inseridos após filtro (ADR-007).
* `webhook_delivery_attempts_total` (counter, labels `outcome`, `attempt_number`) - liga-se a D7/D21.
* `webhook_delivery_duration_seconds` (histogram) - tempo de resposta da chamada HTTP, base para `GET /webhooks/:id/deliveries` (R6).
* `webhook_outbox_pickup_latency_seconds` (histogram) - tempo entre `created_at` do evento e o processamento pelo worker; valida o objetivo de latência < 10s (R2).
* `webhook_dead_letter_moved_total` (counter) - eventos movidos para DLQ (D8).
* `webhook_dead_letter_replay_total` (counter, label `outcome`, sem label de usuário por cardinalidade).

**Logs**
* **Formato e campos essenciais:** JSON estruturado via `logger` Pino central (`src/shared/logger/index.ts`), mesmo padrão de `src/middlewares/request-logger.middleware.ts`. Campos do worker: `eventId`, `webhookId`, `orderId`, `customerId`, `attemptNumber`, `outcome`, `httpStatusCode`, `durationMs`, `errorCode`. Campos do replay de DLQ (auditoria exigida por D17): `deadLetterId`, `outboxEventId`, `replayedBy` (= `req.user.id`), `replayedAt`. `redactPaths` precisa ser estendido para cobrir `secret`, e `X-Signature`/corpo assinado não devem ser logados por completo.

**Tracing**
* **Spans principais e amostragem:** candidatos (Hipótese, infraestrutura nova): `webhook.outbox.insert` (dentro da transação de `changeStatus`), `webhook.worker.poll_cycle`, `webhook.delivery.send` (um span por tentativa HTTP), `webhook.deadletter.replay`. Amostragem proposta (Hipótese): 100% para spans com erro, amostragem reduzida (ex.: 10%) para spans de sucesso.

**Dashboards e alertas**
* Painel: taxa de sucesso/falha de entrega por `webhook_id`/customer, latência de entrega, tamanho do backlog da outbox.
* Alerta: `webhook_outbox_pending_count` crescendo continuamente (worker parado ou travado).
* Alerta: taxa de `webhook_dead_letter_moved_total` acima de limiar por `webhook_id` (cliente cronicamente indisponível).
* Alerta: `webhook_outbox_pickup_latency_seconds` p95 acima de 10s (violação do objetivo R2).

---

## 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| :--- | :--- | :--- |
| Node.js | >=20 | Já é o engine do projeto (`package.json`); `fetch` nativo com `AbortController` cobre o `POST` do worker com timeout de 10s (D21), e `crypto.createHmac('sha256', ...)` nativo cobre a assinatura HMAC (ADR-003) - nenhuma dependência HTTP/cripto nova é necessária |
| @prisma/client / prisma | 5.22.0 | Já em uso; novas tabelas via migration seguindo convenções existentes do schema |
| uuid | 11.0.3 | Já em uso; reaproveitado para `event_id`, ids de webhook/outbox/dead-letter (D24) |
| zod | 3.23.8 | Já em uso; reaproveitado para schemas do módulo `webhooks` (validação `https`, `statusFilter`, etc.) |
| pino | 9.5.0 | Já em uso; logger central importado diretamente no worker (processo separado, sem `pino-http`, que é específico do Express) |

**Garantias de compatibilidade**
* Nenhuma mudança de contrato nos endpoints existentes (`orders`, `customers`, `products`, `auth`) (ADR-006).
* `OrderService.changeStatus` mantém a assinatura pública atual; a integração é interna, via chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação já existente (D20).
* Novos routers `/webhooks` e `/admin/webhooks` registrados em `src/routes/index.ts`, seguindo o padrão `buildApiRouter`, sem impacto nos módulos existentes.
* Schema Prisma é estritamente aditivo: novas tabelas (`webhook_outbox`, `webhook_dead_letter`, configuração de webhook) via migration, sem alterar tabelas/colunas existentes (D24).
* **Gap identificado (não é garantia, é lacuna real):** o payload do webhook (`event_type: "order.status_changed"`) não tem campo de versionamento (D22 não menciona isso). Qualquer mudança futura no formato do payload exigirá estratégia de versionamento própria, não coberta por esta entrega - registrado como risco na seção 10.

---

## 9. Critérios de aceite técnicos

* CRUD completo (`POST/PATCH/DELETE/GET`) funcional; `secret` retornada apenas na criação e na rotação, nunca em listagem/edição (R3, R4, R10).
* Filtro de eventos aplicado na inserção: mudança de status sem nenhum webhook interessado não gera linha na outbox (D16; ADR-007).
* Atomicidade comprovada por teste de integração: falha simulada na inserção da outbox provoca rollback também da mudança de status (D19).
* Payload é snapshot imutável desde a inserção (alteração posterior do pedido não altera evento já enfileirado) (D25).
* Cada request de webhook carrega `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` (D23), com assinatura HMAC-SHA256 validável pelo cliente (ADR-003).
* Rotação de secret: teste comprovando que a secret antiga permanece válida por até 24h em paralelo com a nova (D10).
* `GET /webhooks/:id/deliveries` retorna no máximo os 100 registros mais recentes (R6).
* `POST /admin/webhooks/dead-letter/:id/replay` retorna `403` para usuário não-`ADMIN` e loga `replayedBy` para usuário `ADMIN` (D17).
* Endpoints de CRUD de configuração aceitam qualquer role autenticada, incluindo `OPERATOR` (D18).
* Latência de entrega ponta a ponta (do `created_at` do evento até a chamada HTTP ser disparada) abaixo de 10s no p95 em condições normais de carga (R2).
* Timeout de chamada HTTP do worker fixado em 10s (D21).
* Retry ocorre em exatamente 5 tentativas, com backoff 1min/5min/30min/2h/12h, validado por teste com controle de tempo (D7).
* Após a 5ª falha, evento é movido para `webhook_dead_letter` com payload, motivo e timestamp (D8).
* Payload acima de 64KB no envio gera falha permanente direto para DLQ, sem consumir tentativas de retry (Hipótese confirmada).
* Cadastro de webhook com URL `http` (não `https`) é rejeitado com `WEBHOOK_INVALID_URL` (R8).
* Logs estruturados do worker contêm `eventId`, `webhookId`, `orderId`, `attemptNumber`, `outcome`, sem a `secret` em texto plano (validado após extensão do `redactPaths`).
* Toda execução de replay de DLQ gera log de auditoria com `replayedBy` (D17).
* Revisão de segurança da Sofia concluída (mínimo 2 dias úteis reservados) antes do deploy em produção (`[09:46]`).

---

## 10. Riscos e mitigação

**Rajada de envios sem rate limiting**
* **Probabilidade:** média
* **Impacto:** um pico de mudanças de status (ex.: 50 pedidos em 1 minuto) gera dezenas de chamadas HTTP simultâneas para o mesmo cliente, podendo sobrecarregá-lo (EA1; `[09:38]`-`[09:39]` Diego/Larissa)
* **Mitigação:**
  * Tratado como explicitamente fora de escopo desta entrega
  * Observar volume de envios por `webhook_id` via métricas propostas na seção 7
* **Plano de contingência:** revisitar a decisão e implementar throttling/rate limiting por `webhook_id` caso o volume real ou reclamações de cliente confirmem o problema

**Ordering não garantido globalmente**
* **Probabilidade:** baixa (só se materializa se o sistema escalar para múltiplos workers sem redesenho)
* **Impacto:** eventos do mesmo pedido podem chegar fora de ordem ao cliente se um dia houver mais de um worker rodando em paralelo (D6; ADR-005)
* **Mitigação:**
  * Manter regime single-worker enquanto essa garantia for necessária
  * Documentar a limitação de forma explícita no portal do desenvolvedor (compromisso de Marcos, `[09:26]`)
* **Plano de contingência:** implementar particionamento por `order_id` ou lock pessimista antes de escalar para múltiplos workers (AD2)

**Worker como ponto único de falha**
* **Probabilidade:** média
* **Impacto:** se o processo do worker cair, a outbox para de ser processada até o restart; atraso indefinido na notificação (D3, D4)
* **Mitigação:**
  * Alerta de `webhook_outbox_pending_count` crescente (seção 7) para detectar o problema rapidamente
  * Eventos não se perdem enquanto o worker está fora - continuam `pending` na outbox e são processados ao reiniciar
* **Plano de contingência:** supervisão de processo com restart automático (ex.: systemd/pm2) - Hipótese, não há decisão explícita na transcrição sobre estratégia de deploy/supervisão do worker

**Vazamento de secret HMAC em log**
* **Probabilidade:** média (precedente real: "a gente já teve cliente que vazou secret em log de aplicação dele uma vez", Diego, `[09:22]`; achado técnico real: `redactPaths` não cobre `secret` hoje)
* **Impacto:** alto - compromete a autenticidade de todos os webhooks daquele endpoint específico (ADR-003)
* **Mitigação:**
  * Estender `redactPaths` em `src/shared/logger/index.ts` antes do lançamento
  * Revisão de segurança dedicada da Sofia, focada em HMAC e geração de secret (`[09:46]`)
* **Plano de contingência:** rotação imediata da secret comprometida via `POST /webhooks/:id/rotate-secret` e notificação ao cliente afetado

**Ausência de versionamento de payload**
* **Probabilidade:** baixa no curto prazo, cresce com o tempo
* **Impacto:** médio/alto no longo prazo - qualquer mudança futura no formato do payload quebra integrações de clientes já em produção (gap identificado na seção 8, D22)
* **Mitigação:**
  * Documentar claramente o contrato atual (`event_type: "order.status_changed"`) no portal do desenvolvedor (Marcos, `[09:26]`)
* **Plano de contingência:** introduzir versionamento (novo `event_type` ou header de versão) com período de coexistência, quando necessário

**Risco de cronograma**
* **Probabilidade:** média
* **Impacto:** alto - prazo comprometido com a Atlas para fim de novembro, estimativa de 3 sprints incluindo pelo menos 2 dias úteis de revisão de segurança da Sofia; atraso na revisão pressiona a data (R12; `[09:45]`-`[09:47]`)
* **Mitigação:**
  * Reservar os 2 dias de revisão de segurança desde o início do planejamento, não espremidos no fim
  * Escopo já reduzido (rate limiting e notificação por e-mail adiados - AD1, EA1)
* **Plano de contingência:** priorizar a entrega do fluxo core (outbox + worker + HMAC + retry + DLQ) sobre refinamentos de CRUD/deliveries, se necessário

**DLQ sem reprocessamento automático**
* **Probabilidade:** alta ao longo do tempo (é o comportamento esperado, não uma falha)
* **Impacto:** médio - eventos ficam presos até intervenção manual via replay individual (ADR-002)
* **Mitigação:**
  * Alerta de `webhook_dead_letter_moved_total` acima de limiar (seção 7)
  * Processo operacional definido para revisão periódica da DLQ
* **Plano de contingência:** replay em lote via script administrativo - Hipótese de melhoria futura, não coberta pelo endpoint unitário atual (D8)

---

## 11. Integração com o sistema existente

| Caminho do arquivo | Como a feature se integra |
| :--- | :--- |
| `src/modules/orders/order.service.ts` (método `changeStatus`, linhas 126-179) | Ponto de enxerto: dentro da transação `this.prisma.$transaction(async (tx) => {...})` já existente, após `tx.orderStatusHistory.create(...)`, é inserida a chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` (D20). Nenhuma mudança na assinatura pública do método. |
| `src/shared/errors/http-errors.ts` + `src/shared/errors/app-error.ts` | Reuso 1:1 da hierarquia de erro: novas classes de domínio (ex.: `WebhookNotFoundError extends NotFoundError`, `WebhookInvalidUrlError extends ValidationError`) seguindo o mesmo padrão de `InvalidStatusTransitionError`/`InsufficientStockError`, com `errorCode` prefixado `WEBHOOK_` (D12; ADR-006). |
| `src/shared/logger/index.ts` | Reuso direto do `logger` Pino central pelo worker (processo separado) e pelos serviços do módulo; `redactPaths` precisa ganhar uma entrada para `secret` antes do lançamento. |
| `src/routes/index.ts` (`buildApiRouter`) | Novo registro `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` (e um router equivalente para `/admin/webhooks`), seguindo exatamente o padrão já usado para `orders`, `customers`, `products` (linhas 24-28). |
| `src/modules/orders/order.routes.ts` + `src/modules/orders/order.schemas.ts` | Modelo de referência direto para `webhook.routes.ts`/`webhook.schemas.ts`: `authenticate` aplicado no router inteiro, `validate({...})` por rota com schemas Zod, `requireRole('ADMIN')` adicional só na rota de replay (D14, D17, D18; ADR-006). |
| `src/middlewares/auth.middleware.ts` | `authenticate` protege todo o CRUD de configuração; `requireRole('ADMIN')` (já existente, aceita `'ADMIN' \| 'OPERATOR'`) protege exclusivamente o replay de DLQ, sem qualquer alteração no middleware (D17). |
| `prisma/schema.prisma` | Recebe três novas tabelas (`webhook_outbox`, `webhook_dead_letter`, configuração de webhook por customer) via migration, seguindo as convenções já em uso: `id String @id @default(uuid()) @db.Char(36)`, `@@map(snake_case)`, índices em `status`/`created_at` (D5, D24). |
| `src/modules/orders/order.repository.ts` | Modelo de estilo de referência para o `WebhookRepository`/`WebhookOutboxRepository`: classe fina recebendo `PrismaClient` no construtor, métodos simples equivalentes a `list`/`findById`. |

---
