### PRD: Order Management System, Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-08-30
Responsável: Marcos (Product Manager)

---

### Resumo

Três clientes B2B integrados à nossa plataforma pediram formalmente para ser notificados quando o status dos pedidos deles muda, em vez de ficar consultando a API de tempos em tempos (`[09:00]` Marcos). Esta feature entrega essa notificação ativa: o cliente cadastra uma URL, escolhe quais mudanças de status quer receber, e a plataforma avisa essa URL sempre que uma delas acontece, em menos de 10 segundos no caso normal (`[09:02]` Marcos).

O fluxo é apenas de saída: a plataforma envia, o cliente recebe, e não há canal de entrada do cliente para nós (`[09:02]`-`[09:03]` Marcos/Sofia). Cada envio é assinado, para o cliente conseguir provar que a mensagem veio mesmo de nós e não foi adulterada no caminho (`[09:20]` Sofia). Se o cliente estiver fora do ar, a plataforma tenta de novo ao longo de aproximadamente 15 horas antes de parar e deixar o evento em uma fila de falhas, reprocessável sob demanda por um administrador (`[09:17]`-`[09:19]` Diego/Larissa).

---

### Contexto e problema

Público-alvo
- Clientes B2B integrados por API, hoje representados por Atlas Comercial, MaxDistribuição e Nova Cargo, que consomem informação de pedidos de forma automatizada (`[09:00]` Marcos).
- Usuários operadores autenticados na plataforma, que cadastram e mantêm os webhooks em nome de um cliente (`[09:32]` Marcos/Larissa).
- Usuários com perfil de administrador, únicos autorizados a reprocessar entregas que falharam em definitivo (`[09:36]` Sofia/Larissa).

Cenários de uso chave
- O cliente cadastra uma URL e escolhe receber apenas os status que interessam a ele, por exemplo somente quando o pedido é despachado e quando é entregue (`[09:33]` Marcos).
- Um pedido do cliente muda de status na plataforma e o sistema dele é avisado em segundos, sem precisar perguntar (`[09:00]`-`[09:02]` Marcos).
- O sistema do cliente fica fora do ar durante uma manutenção e volta a receber as notificações quando retorna, sem perder o que aconteceu nesse intervalo (`[09:16]` Diego).
- O time do cliente quer conferir o que foi enviado e o que deu errado, e consulta o histórico das últimas entregas daquele webhook (`[09:34]` Marcos).
- Um administrador identifica que um evento esgotou as tentativas e o reprocessa manualmente, ficando registrado quem executou a ação (`[09:35]`-`[09:36]` Diego/Sofia).
- O cliente suspeita que sua secret vazou, pede uma nova e tem 24 horas para migrar seus sistemas com as duas válidas em paralelo (`[09:21]`-`[09:22]` Sofia/Diego).

Onde essa feature será implantada
- Em sistema que já existe: o Order Management System em produção, que hoje já controla o ciclo de vida do pedido, o histórico de mudanças de status e o estoque, e que não possui nenhum mecanismo de notificação externa (`[09:04]` Bruno; `src/modules/orders/order.service.ts`).

Problemas priorizados
- Prioridade alta. Os clientes descobrem mudanças de status consultando repetidamente a API de pedidos, o que torna a integração deles lenta e cara. Impacto: custo de integração recorrente do lado do cliente e carga desnecessária na nossa API (`[09:00]` Marcos).
- Prioridade alta. Risco de perda de conta: a Atlas sinalizou que pode migrar para um concorrente se a entrega não sair no prazo combinado. Impacto: perda de receita e de referência comercial em um segmento onde os três clientes pediram a mesma coisa (`[09:00]` Marcos).
- Prioridade alta. Sem notificação ativa, o pedido fica pendurado do lado do cliente e alguém precisa atualizar manualmente para saber o que houve. Impacto: trabalho operacional manual e atraso na reação do cliente a eventos do próprio negócio dele (`[09:02]` Marcos).

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Substituir a consulta repetida à API por notificação ativa de mudança de status | Tempo entre a mudança de status e o envio da notificação | Abaixo de 10 segundos (`[09:02]` Marcos) |
| Nunca perder um evento em silêncio | Proporção de eventos gerados que terminam entregues ou registrados na fila de falhas | 100 por cento, sem descarte sem rastro (`[09:40]`-`[09:41]` Bruno/Diego) |
| Tolerar indisponibilidade do sistema do cliente sem intervenção | Janela coberta pelas novas tentativas antes de a entrega ir para a fila de falhas | 5 tentativas ao longo de aproximadamente 15 horas (`[09:17]` Diego) |
| Permitir que o cliente valide a origem e a integridade do que recebe | Proporção de envios assinados e feitos sobre canal seguro | 100 por cento dos envios assinados, com URL obrigatoriamente https (`[09:20]`-`[09:23]` Sofia) |
| Atender o compromisso comercial assumido com a Atlas | Data de entrega da feature em produção | Fim de novembro, com estimativa de três sprints (`[09:45]`-`[09:46]` Marcos/Larissa) |

---

### Escopo

Incluso
- Cadastro de webhook por cliente, com URL e escolha dos status de interesse, e secret gerada pela plataforma e devolvida na criação (`[09:31]`-`[09:32]` Marcos/Bruno).
- Edição, remoção e listagem dos webhooks de um cliente (`[09:33]` Bruno).
- Filtro de eventos por webhook: cada endpoint recebe apenas os status que declarou querer ouvir (`[09:33]`-`[09:34]` Marcos/Bruno).
- Envio da notificação para a URL cadastrada, assinado, com o conteúdo do pedido congelado no instante da mudança de status (`[09:20]` Sofia; `[09:52]` Larissa).
- Nova tentativa automática quando o cliente não recebe, com espaçamento crescente, e encerramento em uma fila de falhas quando as tentativas se esgotam (`[09:15]`-`[09:18]` Diego).
- Consulta ao histórico das últimas 100 entregas de um webhook, com resultado, conteúdo enviado, resposta e tempo de resposta (`[09:34]` Marcos).
- Reprocessamento manual, restrito a administradores e registrado para auditoria, de entregas que caíram na fila de falhas (`[09:18]` Diego; `[09:36]` Sofia).
- Troca de secret sob demanda do cliente, com a secret anterior válida por mais 24 horas em paralelo (`[09:21]` Sofia).

Fora de escopo
- Receber webhooks enviados pelo cliente para a plataforma. O fluxo é exclusivamente de saída (`[09:02]`-`[09:03]` Marcos/Sofia).
- Painel visual para o cliente acompanhar seus webhooks. Fica como projeto separado do time de frontend (`[09:39]`-`[09:40]` Marcos/Larissa).
- Aviso por e-mail ao cliente quando o webhook dele falha repetidamente. Adiado para uma fase futura, depois de medido o impacto real (`[09:37]`-`[09:38]` Marcos/Larissa).
- Controle de vazão do envio em picos de mudança de status. Registrado na reunião como ponto a observar e decidir depois (`[09:38]`-`[09:39]` Diego/Larissa).
- Garantia de entrega única. A plataforma pode repetir um envio, e é o cliente que descarta a repetição (`[09:24]`-`[09:25]` Diego/Sofia).
- Retenção e arquivamento de eventos antigos já entregues. Declarado fora do escopo desta feature (`[09:08]` Diego).

---

### Requisitos funcionais

#### [PRD-FR-01] Notificação automática de mudança de status
Sempre que o status de um pedido muda, a plataforma avisa as URLs que o cliente dono daquele pedido cadastrou e que declararam interesse naquele status.

**Fluxo principal**
- Um pedido de um cliente muda de status na plataforma (`[09:00]` Marcos).
- A plataforma verifica se aquele cliente tem algum webhook ativo interessado no novo status (`[09:33]`-`[09:34]` Marcos/Bruno).
- O evento é registrado junto com a própria mudança de status, de modo que um não possa existir sem o outro (`[09:40]`-`[09:41]` Bruno/Diego).
- A notificação é enviada para cada URL interessada, em menos de 10 segundos no caso normal (`[09:02]` Marcos; `[09:09]`-`[09:10]` Diego/Larissa).
- O envio é registrado no histórico daquele webhook, com resultado e tempo de resposta (`[09:34]` Marcos).

**Fluxos alternativos e exceções**
- Nenhum webhook do cliente interessado naquele status: nenhuma notificação é gerada e nada é registrado para envio (`[09:34]` Bruno).
- A mudança de status não se completa: nenhuma notificação é gerada, porque evento e mudança de status vivem ou morrem juntos (`[09:40]`-`[09:41]` Bruno/Diego).
- O mesmo pedido muda de status várias vezes em sequência rápida: as notificações daquele pedido saem na ordem em que as mudanças aconteceram (`[09:12]`-`[09:13]` Diego/Larissa).
- O cliente não responde ou responde com erro: a entrega passa para a política de novas tentativas descrita em PRD-FR-06 (`[09:14]`-`[09:15]` Larissa/Diego).

**Erros previstos**
- Cliente que não responde dentro de 10 segundos é tratado como falha e a entrega é reagendada (`[09:42]` Diego).
- Evento que ultrapassa 64KB não é enviado e falha de forma explícita, sem cortar conteúdo (`[09:23]`-`[09:24]` Sofia/Diego/Larissa).

**Prioridade:** alta

---

#### [PRD-FR-02] Cadastro de webhook pelo cliente
Um usuário autenticado cadastra uma URL de webhook para um cliente, escolhendo os status de interesse, e recebe de volta a secret daquele endpoint.

**Fluxo principal**
- O usuário autenticado envia o cadastro informando o cliente, a URL e a lista de status que quer receber (`[09:31]` Marcos).
- A plataforma recusa o cadastro se a URL não usar https (`[09:23]` Sofia).
- A plataforma gera uma secret exclusiva daquele endpoint e a devolve na resposta da criação (`[09:31]` Marcos; `[09:21]` Sofia).
- O webhook passa a receber as notificações dos status escolhidos.

**Fluxos alternativos e exceções**
- O cliente ao qual o webhook pertence é informado no próprio cadastro, e não deduzido do usuário autenticado, porque o login é do operador da nossa plataforma e não do cliente final (`[09:32]` Bruno/Marcos/Larissa).
- Um mesmo cliente pode manter mais de um webhook cadastrado, com listas de status diferentes, e cada envio identifica a qual cadastro se refere (`[09:44]` Sofia).

**Erros previstos**
- Cadastro com URL http é recusado por validação, sem ser persistido (`[09:23]` Sofia).

**Prioridade:** alta

---

#### [PRD-FR-03] Edição, remoção e listagem de webhooks
O usuário autenticado consegue alterar, remover e listar os webhooks cadastrados de um cliente.

**Fluxo principal**
- O usuário lista os webhooks de um cliente e vê os cadastros existentes (`[09:33]` Bruno).
- O usuário altera a URL ou a lista de status de um webhook existente (`[09:33]` Bruno).
- O usuário remove um webhook que não deve mais receber notificações (`[09:33]` Bruno).

**Fluxos alternativos e exceções**
- Qualquer usuário autenticado pode executar essas operações, sem exigência de perfil administrador nesta fase (`[09:36]`-`[09:37]` Marcos/Sofia).
- A secret não é devolvida em listagem nem em edição, apenas na criação e na troca (`[09:31]` Marcos; `[09:21]` Sofia).

**Erros previstos**
- Operação sobre um webhook inexistente é recusada com erro de recurso não encontrado (`[09:28]` Bruno).

**Prioridade:** alta

---

#### [PRD-FR-04] Filtro de status por webhook
Cada webhook declara quais mudanças de status quer receber, e só é notificado sobre elas.

**Fluxo principal**
- No cadastro ou na edição, o usuário informa a lista de status de interesse daquele webhook, por exemplo apenas despachado e entregue (`[09:33]` Marcos).
- Quando um pedido muda de status, apenas os webhooks daquele cliente que listaram aquele status são considerados (`[09:34]` Bruno).

**Fluxos alternativos e exceções**
- Se nenhum webhook do cliente quiser aquele status, o evento não chega a ser registrado para envio (`[09:34]` Bruno/Diego).
- O filtro vale a partir do momento em que é configurado. Mudanças de status ocorridas antes do cadastro do webhook não são recuperadas depois (`[09:34]` Bruno).

**Erros previstos**
- Não há erro específico registrado na reunião para este requisito.

**Prioridade:** alta

---

#### [PRD-FR-05] Histórico de entregas do webhook
O cliente consegue consultar o que a plataforma enviou para um webhook e o que aconteceu em cada envio.

**Fluxo principal**
- O usuário consulta o histórico de um webhook específico (`[09:34]` Marcos).
- A plataforma devolve os últimos 100 envios daquele webhook, com sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta (`[09:34]` Marcos).

**Fluxos alternativos e exceções**
- Webhook sem nenhum envio ainda devolve histórico vazio (`[09:34]` Marcos).
- Envios anteriores aos 100 mais recentes não são expostos por este requisito (`[09:34]` Marcos).

**Erros previstos**
- Consulta a um webhook inexistente é recusada com erro de recurso não encontrado (`[09:28]` Bruno).

**Prioridade:** media

---

#### [PRD-FR-06] Nova tentativa automática e fila de falhas
Quando o cliente não recebe a notificação, a plataforma tenta de novo por um período determinado e, esgotadas as tentativas, guarda a entrega em uma fila de falhas.

**Fluxo principal**
- A primeira tentativa de envio falha, seja por erro do cliente, por indisponibilidade ou por não responder no tempo limite (`[09:14]` Larissa; `[09:42]` Diego).
- A plataforma repete o envio com intervalos crescentes: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando 5 tentativas em aproximadamente 15 horas (`[09:17]` Diego).
- Se alguma tentativa tiver sucesso, a entrega é considerada concluída e o ciclo termina (`[09:15]` Diego).
- Se as 5 tentativas falharem, a entrega é movida para a fila de falhas, guardando o conteúdo, o motivo da falha e o momento em que ocorreu (`[09:18]` Diego).

**Fluxos alternativos e exceções**
- A janela de aproximadamente 15 horas foi dimensionada a partir de um caso real de cliente com 2 horas de indisponibilidade em manutenção planejada (`[09:16]` Diego).
- Não existe repetição infinita: sem teto de tentativas, um evento ficaria pendurado para sempre caso o cliente desaparecesse (`[09:15]` Diego).

**Erros previstos**
- Cliente que não responde em 10 segundos conta como falha da tentativa (`[09:42]` Diego).
- Evento acima de 64KB falha de forma explícita, sem truncar o conteúdo (`[09:23]`-`[09:24]` Sofia/Diego/Larissa).

**Prioridade:** alta

---

#### [PRD-FR-07] Reprocessamento manual por administrador
Um administrador consegue reenviar uma entrega que esgotou as tentativas e ficou na fila de falhas.

**Fluxo principal**
- O administrador identifica uma entrega na fila de falhas (`[09:18]` Diego).
- O administrador aciona o reprocessamento daquela entrega (`[09:35]` Diego).
- A entrega volta a ser tratada como pendente e é enviada novamente (`[09:18]` Diego).
- A plataforma registra quem executou o reprocessamento, para auditoria (`[09:36]` Sofia).

**Fluxos alternativos e exceções**
- Usuário sem perfil de administrador não consegue executar a ação, porque mexer em fila de entrega de notificação não é atribuição de operador (`[09:36]` Sofia/Larissa).
- O reprocessamento é sempre manual e individual. Não há reprocessamento automático da fila de falhas (`[09:18]` Diego).

**Erros previstos**
- Reprocessamento solicitado por usuário sem perfil de administrador é recusado (`[09:36]` Sofia).

**Prioridade:** media

---

#### [PRD-FR-08] Troca de secret com convivência de 24 horas
O cliente consegue pedir uma nova secret para um webhook, com tempo para migrar seus sistemas.

**Fluxo principal**
- O cliente solicita uma nova secret para um webhook específico (`[09:21]` Sofia).
- A plataforma gera a nova secret e a devolve na resposta (`[09:21]` Sofia).
- Durante 24 horas, a secret anterior continua válida em paralelo com a nova (`[09:21]` Sofia).
- Passadas as 24 horas, a secret anterior deixa de valer (`[09:21]` Sofia).

**Fluxos alternativos e exceções**
- A troca existe justamente para o caso de suspeita de vazamento, situação já vivida com um cliente que expôs a secret no log da aplicação dele (`[09:22]` Diego).
- Cada webhook tem sua própria secret, então a troca afeta apenas aquele endpoint (`[09:21]` Sofia).

**Erros previstos**
- Solicitação de troca para um webhook inexistente é recusada com erro de recurso não encontrado (`[09:28]` Bruno).

**Prioridade:** media

---

#### [PRD-FR-09] Identificador único de evento para deduplicação pelo cliente
Cada notificação carrega um identificador único do evento, para o cliente reconhecer e descartar um recebimento repetido.

**Fluxo principal**
- No momento em que o evento é registrado, a plataforma gera um identificador único para ele (`[09:25]` Diego).
- Toda tentativa de envio daquele mesmo evento carrega esse mesmo identificador (`[09:25]` Diego).
- O cliente guarda os identificadores já processados e descarta o que chegar repetido (`[09:25]` Diego).

**Fluxos alternativos e exceções**
- A garantia é de entrega ao menos uma vez, então receber o mesmo evento duas vezes é comportamento esperado, não defeito (`[09:24]` Diego).
- A regra é documentada de forma destacada no portal do desenvolvedor, para o cliente não ser pego de surpresa (`[09:26]` Marcos).

**Erros previstos**
- Não há erro específico registrado na reunião para este requisito. O risco associado, de o cliente não implementar a deduplicação, está tratado na seção de riscos (`[09:25]` Sofia).

**Prioridade:** alta

---

#### [PRD-FR-10] Envio assinado para validação de origem
Cada notificação é assinada, para o cliente confirmar que veio da nossa plataforma e que não foi alterada no caminho.

**Fluxo principal**
- A plataforma assina o conteúdo de cada envio usando a secret daquele webhook (`[09:20]` Sofia).
- A assinatura acompanha o envio, junto com o momento em que ele foi feito (`[09:20]`, `[09:44]` Sofia/Diego).
- O cliente recalcula a assinatura do seu lado com a mesma secret e compara antes de processar o conteúdo (`[09:19]`-`[09:20]` Sofia).

**Fluxos alternativos e exceções**
- A assinatura usa HMAC-SHA256, escolhido por ser padrão de mercado com biblioteca disponível em qualquer stack de cliente (`[09:20]` Sofia).
- Cada endpoint tem secret própria, de forma que o vazamento de uma não comprometa os demais webhooks (`[09:21]` Sofia).
- O envio inclui a identificação do cadastro de webhook usado, para o cliente que mantém vários saber qual deles originou aquela chamada (`[09:44]` Sofia).

**Erros previstos**
- Não há erro específico registrado na reunião para este requisito.

**Prioridade:** alta

---

### Requisitos não funcionais

Performance
- Tempo entre a mudança de status e o envio da notificação abaixo de 10 segundos, definição de tempo real acordada com o cliente (`[09:02]` Marcos).
- Verificação de novos eventos a cada 2 segundos, o que sustenta com folga o limite de 10 segundos (`[09:09]`-`[09:10]` Diego/Larissa).
- Tempo máximo de espera pela resposta do cliente de 10 segundos por tentativa (`[09:42]` Diego).
- Tamanho máximo do conteúdo de um evento de 64KB (`[09:24]` Diego/Larissa).

Disponibilidade
- A reunião não definiu meta de disponibilidade para a feature, e nenhum número desse tipo foi assumido aqui.
- Evento pendente não se perde caso o envio seja interrompido: ele permanece aguardando e é retomado quando o envio volta a operar (`[09:40]`-`[09:41]` Bruno/Diego).
- Indisponibilidade do sistema do cliente é tolerada por aproximadamente 15 horas antes de a entrega ir para a fila de falhas (`[09:17]` Diego).

Segurança e autorização
- Todo envio é assinado com HMAC-SHA256 sobre o conteúdo enviado (`[09:20]` Sofia).
- Cada webhook tem secret exclusiva, nunca uma secret única compartilhada por toda a plataforma (`[09:21]` Sofia).
- Secret rotacionável sob demanda, com 24 horas de convivência entre a anterior e a nova (`[09:21]` Sofia).
- URL de webhook obrigatoriamente https, com cadastro em http recusado por validação (`[09:23]` Sofia).
- Cadastro, edição, remoção e listagem exigem usuário autenticado, sem restrição de perfil nesta fase (`[09:36]`-`[09:37]` Sofia/Marcos).
- Reprocessamento de entrega em falha exige perfil de administrador (`[09:36]` Sofia/Larissa).
- A secret não pode aparecer em log da plataforma. O filtro de dados sensíveis do log existente hoje não cobre esse campo e precisa ser ajustado antes do lançamento (`src/shared/logger/index.ts`).

Observabilidade
- Histórico das últimas 100 entregas por webhook, consultável pelo cliente, com resultado, conteúdo enviado, resposta e tempo de resposta (`[09:34]` Marcos).
- Registro de auditoria de todo reprocessamento de entrega em falha, identificando quem executou (`[09:36]` Sofia).
- Registro estruturado das tentativas de envio, reaproveitando o mecanismo de log já usado por toda a aplicação (`[09:29]` Bruno; `src/shared/logger/index.ts`).

Confiabilidade e integridade de dados
- A notificação e a mudança de status são registradas de forma atômica: se a mudança de status não se completa, o evento não existe (`[09:40]`-`[09:41]` Bruno/Diego).
- O conteúdo do evento é congelado no instante da mudança de status, de modo que alterações posteriores no pedido não mudem o que já foi registrado para envio (`[09:52]` Larissa/Diego/Bruno).
- Entrega ao menos uma vez: um evento pode chegar repetido, mas nunca é descartado sem registro (`[09:24]` Diego; `[09:18]` Diego).
- A ordem de chegada é garantida apenas entre eventos de um mesmo pedido. Não há garantia de ordem global entre pedidos diferentes, limitação assumida e conhecida (`[09:12]`-`[09:13]` Diego/Larissa).

Compatibilidade e portabilidade
- Clientes que já integram com a plataforma continuam funcionando sem qualquer alteração: nenhum contrato existente muda por causa desta feature (`[09:30]` Larissa; `docs/FDD.md`, seção 8).
- A feature reaproveita a stack e os padrões já em produção, sem introduzir infraestrutura nova para o time operar (`[09:07]` Diego/Larissa; `[09:30]` Larissa).

Compliance
- Não houve requisito de compliance discutido na reunião. Nada foi assumido aqui por conta própria.

Acessibilidade
- Não se aplica a esta entrega e não foi discutido na reunião: a feature não tem interface de usuário, e o painel visual para o cliente foi explicitamente deixado fora de escopo (`[09:39]`-`[09:40]` Marcos/Larissa).

---

### Decisões e trade-offs

#### Decisão: notificar fora da transação que muda o status do pedido
- **Justificativa:** a mudança de status já é uma operação pesada, e esperar a resposta do cliente dentro dela travaria mudanças de status de outros pedidos sempre que um cliente estivesse lento. Além disso, não haveria o que fazer se o cliente estivesse fora do ar, já que desfazer a mudança de status por causa disso não é aceitável (`[09:03]`-`[09:04]` Larissa/Bruno; `[09:06]` Diego).
- **Trade-off:** a notificação deixa de ser instantânea e passa a chegar com um pequeno atraso, aceito porque continua bem abaixo dos 10 segundos combinados com o cliente (`[09:10]` Larissa).

#### Decisão: entregar ao menos uma vez e deixar a deduplicação com o cliente
- **Justificativa:** garantir entrega única exigiria coordenação entre os dois lados e complexidade muito maior. Entregar ao menos uma vez, com identificador único por evento, resolve a quase totalidade dos casos e é o padrão adotado por players de mercado como Stripe e GitHub (`[09:25]` Diego).
- **Trade-off:** transfere trabalho de integração para o cliente, objeção levantada explicitamente na reunião. A contrapartida assumida foi documentar a regra de forma destacada no portal do desenvolvedor (`[09:25]` Sofia; `[09:26]` Marcos).

#### Decisão: cinco tentativas em aproximadamente 15 horas, depois reprocessamento manual
- **Justificativa:** três tentativas se esgotariam em cerca de 30 minutos, insuficiente diante de um caso real de cliente com 2 horas de indisponibilidade planejada. Repetição infinita, por outro lado, deixaria eventos pendurados para sempre se o cliente desaparecesse (`[09:15]`-`[09:16]` Diego).
- **Trade-off:** um evento pode levar até cerca de 15 horas para ser dado como falho, e a partir daí depende de alguém identificar e acionar o reprocessamento, que não é automático (`[09:17]`-`[09:18]` Diego).

#### Decisão: secret exclusiva por webhook, com troca e 24 horas de convivência
- **Justificativa:** uma secret única para toda a plataforma faria o vazamento de um cliente comprometer todos os demais. O risco não é teórico: já houve cliente que expôs a secret no log da própria aplicação (`[09:21]` Sofia; `[09:22]` Diego).
- **Trade-off:** o número de segredos a gerir cresce com a quantidade de webhooks cadastrados, e durante a janela de troca a plataforma precisa aceitar duas secrets válidas ao mesmo tempo (`[09:21]`-`[09:22]` Sofia).

#### Decisão: aplicar o filtro de status no momento de registrar o evento
- **Justificativa:** se nenhum webhook do cliente quer aquele status, não faz sentido registrar um evento que nunca será entregue a ninguém (`[09:34]` Bruno/Diego).
- **Trade-off:** um cliente que cadastre um webhook novo, ou amplie sua lista de status depois, não consegue receber retroativamente os eventos que já ocorreram e não foram registrados (`[09:34]` Bruno).

#### Decisão: garantir ordem apenas dentro de um mesmo pedido
- **Justificativa:** manter a ordem por pedido atende o que os clientes efetivamente pedem, que é saber o que aconteceu com cada pedido deles. Ordem global entre pedidos nunca foi solicitada (`[09:13]`-`[09:14]` Diego/Larissa/Marcos).
- **Trade-off:** a garantia depende do arranjo atual de execução; ampliar a capacidade de envio no futuro exigirá rever esse desenho antes, sob pena de perder a ordem por pedido (`[09:12]`-`[09:13]` Diego/Bruno).

---

### Dependências

#### Organizacional: revisão de segurança antes do deploy
A revisão do código de assinatura e de geração de secret precisa acontecer antes de a feature subir para produção, com no mínimo dois dias úteis reservados para isso. É condição colocada pela engenharia de segurança na própria reunião e faz parte da estimativa de três sprints (`[09:46]` Sofia/Larissa).

#### Organizacional: documentação no portal do desenvolvedor
O cliente precisa saber como integrar, como validar a assinatura e, principalmente, que a entrega é ao menos uma vez e cabe a ele descartar repetições. A publicação dessa documentação foi assumida pelo Product Manager em dois momentos da reunião (`[09:26]` e `[09:40]` Marcos).

#### Externa: preparo do lado do cliente
O cliente precisa expor uma URL sobre https, capaz de receber a notificação, validar a assinatura com a secret dele e responder em até 10 segundos. Sem isso, a feature não entrega valor, por mais correta que esteja do nosso lado (`[09:19]`-`[09:23]` Sofia; `[09:42]` Diego).

#### Técnica: fluxo de mudança de status como gatilho
Toda a feature depende do fluxo de mudança de status já existente, que é onde o evento passa a ser registrado. Qualquer caminho novo de mudança de status que venha a ser criado no futuro precisa passar por esse mesmo ponto, senão deixa de gerar notificação (`[09:40]`-`[09:41]` Bruno/Diego; `src/modules/orders/order.service.ts`).

#### Técnica: operação de um segundo processo contínuo
O envio das notificações roda separado da API, para não ser derrubado junto com ela em um reinício. A operação precisa comportar esse segundo processo em execução contínua, com acompanhamento próprio (`[09:11]` Diego/Larissa/Bruno).

#### Técnica: proteção da secret nos registros de log
O filtro de dados sensíveis usado hoje pelo log da aplicação não cobre o campo de secret. Esse ajuste é pré-requisito de lançamento, porque a secret é justamente o que garante ao cliente a autenticidade do que ele recebe (`src/shared/logger/index.ts`; risco reforçado pelo precedente citado em `[09:22]` Diego).

---

### Riscos e mitigação

#### O prazo de fim de novembro fica sob pressão e a entrega atrasa
- **Probabilidade:** media
- **Impacto:** alto. O compromisso foi assumido diretamente com a Atlas, que sinalizou possibilidade de migrar para o concorrente; a estimativa é de três sprints e inclui a revisão de segurança no fim (`[09:00]`, `[09:45]` Marcos; `[09:46]`-`[09:47]` Larissa)
- **Mitigação:**
  - Reservar os dois dias úteis de revisão de segurança desde o planejamento, em vez de espremê-los no fim (`[09:46]` Sofia)
  - Manter o escopo já enxugado na reunião, com controle de vazão e aviso por e-mail adiados (`[09:37]`-`[09:39]` Marcos/Larissa/Diego)
- **Plano de contingência:** priorizar o caminho central, que é notificar com nova tentativa e fila de falhas, adiando refinamentos de gestão de webhook e histórico de entregas.

#### Um pico de mudanças de status sobrecarrega o sistema do cliente
- **Probabilidade:** media
- **Impacto:** medio. Cinquenta pedidos mudando de status em um minuto geram dezenas de chamadas quase simultâneas para o mesmo cliente, e não há controle de vazão nesta entrega (`[09:38]`-`[09:39]` Diego/Larissa)
- **Mitigação:**
  - Tratar o controle de vazão como decisão explicitamente adiada, e não como esquecimento (`[09:39]` Larissa)
  - Acompanhar o volume de envios por webhook depois do lançamento, para decidir com dado real (`[09:39]` Diego)
- **Plano de contingência:** implementar controle de vazão por webhook caso o volume observado ou uma reclamação de cliente confirmem o problema.

#### O cliente não implementa a deduplicação e processa o mesmo evento duas vezes
- **Probabilidade:** media
- **Impacto:** alto. Um evento processado em duplicidade pode gerar efeito indesejado no sistema do cliente, e a responsabilidade de evitá-lo foi deliberadamente colocada do lado dele (`[09:25]` Sofia/Diego)
- **Mitigação:**
  - Documentar a regra de forma destacada no portal do desenvolvedor, compromisso assumido na reunião (`[09:26]` Marcos)
  - Enviar sempre o identificador único do evento, para que a deduplicação seja trivial de implementar (`[09:25]` Diego)
- **Plano de contingência:** apoiar diretamente o cliente afetado no ajuste da integração dele, usando o histórico de entregas como evidência do que foi enviado e quando.

#### A secret de assinatura vaza
- **Probabilidade:** media. Há precedente real de cliente que expôs a secret no log da aplicação dele, e o filtro de log da nossa plataforma hoje não cobre esse campo (`[09:22]` Diego; `src/shared/logger/index.ts`)
- **Impacto:** alto. Compromete a garantia de autenticidade dos envios daquele endpoint, que é a base da confiança do cliente no que recebe (`[09:19]`-`[09:21]` Sofia)
- **Mitigação:**
  - Ajustar o filtro de dados sensíveis do log antes do lançamento
  - Manter secret exclusiva por webhook, limitando o alcance de um vazamento a um único endpoint (`[09:21]` Sofia)
  - Revisão de segurança dedicada à assinatura e à geração de secret antes do deploy (`[09:46]` Sofia)
- **Plano de contingência:** troca imediata da secret comprometida, aproveitando a convivência de 24 horas para o cliente migrar sem interrupção.

#### Entregas ficam paradas na fila de falhas até alguém agir
- **Probabilidade:** alta. É o comportamento projetado, não um defeito: o reprocessamento é sempre manual (`[09:18]` Diego)
- **Impacto:** medio. Eventos deixam de chegar ao cliente por tempo indeterminado se ninguém acompanhar a fila
- **Mitigação:**
  - Definir rotina operacional de acompanhamento da fila de falhas
  - Usar o histórico de entregas para identificar clientes com falha recorrente (`[09:34]` Marcos)
- **Plano de contingência:** acionar o reprocessamento manual em lote pelo administrador, priorizando os clientes com maior volume parado.

#### Uma mudança futura no formato do evento quebra integrações já em produção
- **Probabilidade:** baixa no curto prazo, e cresce com o tempo
- **Impacto:** alto. O formato do evento foi definido sem nenhum indicador de versão, então qualquer alteração futura chega sem aviso ao cliente já integrado (`[09:43]` Diego; `docs/FDD.md`, seção 8)
- **Mitigação:**
  - Documentar o formato atual de forma clara no portal do desenvolvedor (`[09:26]` Marcos)
- **Plano de contingência:** introduzir versionamento do evento com um período de convivência entre o formato antigo e o novo, antes de aposentar o atual.

#### O envio é interrompido e as notificações atrasam
- **Probabilidade:** media
- **Impacto:** medio. Enquanto o envio estiver fora do ar, nenhum cliente é notificado, ainda que nada se perca: os eventos ficam aguardando e são retomados depois (`[09:11]` Diego; `[09:40]`-`[09:41]` Bruno/Diego)
- **Mitigação:**
  - Rodar o envio separado da API, justamente para que um reinício da API não o derrube junto (`[09:11]` Diego)
  - Acompanhar o acúmulo de eventos aguardando envio como sinal de que algo parou
- **Plano de contingência:** restabelecer o processo de envio e deixar que a fila acumulada seja drenada, sem necessidade de reprocessamento manual, já que os eventos continuam pendentes.

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Um usuário autenticado cadastra um webhook informando cliente, URL e status de interesse, e recebe a secret na resposta da criação (`[09:31]` Marcos).
- Cadastro com URL que não seja https é recusado e não é persistido (`[09:23]` Sofia).
- Uma mudança de status em pedido com webhook interessado gera notificação em menos de 10 segundos em condições normais (`[09:02]` Marcos).
- Uma mudança de status sem nenhum webhook interessado não gera notificação nem registro de envio (`[09:34]` Bruno).
- Toda notificação carrega um identificador único de evento, estável entre as tentativas do mesmo evento (`[09:25]` Diego).
- Toda notificação é assinada e a assinatura é verificável pelo cliente com a secret daquele webhook (`[09:20]` Sofia).
- Cliente indisponível recebe 5 tentativas, com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas (`[09:17]` Diego).
- Esgotadas as 5 tentativas, a entrega vai para a fila de falhas guardando conteúdo, motivo e momento da falha (`[09:18]` Diego).
- A consulta de histórico devolve os últimos 100 envios do webhook, com resultado, conteúdo, resposta e tempo de resposta (`[09:34]` Marcos).
- A troca de secret mantém a secret anterior válida por 24 horas em paralelo com a nova (`[09:21]` Sofia).
- Apenas usuário com perfil de administrador reprocessa entrega em falha, e o reprocessamento registra quem o executou (`[09:36]` Sofia).
- Uma mudança de status que não se completa não deixa nenhuma notificação registrada (`[09:40]`-`[09:41]` Bruno/Diego).
- O conteúdo da notificação reflete o estado do pedido no instante da mudança de status, mesmo que o pedido mude depois (`[09:52]` Larissa).
- Revisão de segurança concluída e documentação publicada no portal do desenvolvedor antes do lançamento (`[09:46]` Sofia; `[09:26]` e `[09:40]` Marcos).

---

### Testes e validação

Tipos de teste obrigatórios
- Testes de integração ponta a ponta, no padrão de teste já vigente no projeto, cobrindo o caminho completo da mudança de status até o envio da notificação (`[09:46]` Larissa; `tests/orders.test.ts`, `tests/helpers/factories.ts`).
- Testes automatizados das regras críticas de entrega: mudança de status não concluída não gera notificação, as 5 tentativas respeitam a janela de aproximadamente 15 horas, e o filtro de status de interesse é respeitado (`[09:17]`, `[09:34]`, `[09:40]`-`[09:41]`).
- Testes automatizados das regras de segurança: recusa de URL http, assinatura verificável e convivência de 24 horas entre secret anterior e nova (`[09:21]`, `[09:23]` Sofia).
- Revisão de segurança manual da assinatura e da geração de secret, com no mínimo dois dias úteis reservados antes do deploy (`[09:46]` Sofia).
- **Hipótese:** ensaio de rajada, medindo o efeito de dezenas de mudanças de status em sequência sobre um mesmo cliente. Não foi proposto na reunião; entra ancorado no risco de pico levantado em `[09:38]` por Diego, que ficou sem proteção nesta entrega.

Estratégia de validação
- Seguir a suíte de testes automatizados já existente no projeto, escrevendo os novos testes no mesmo padrão, e tratar a revisão de segurança como portão obrigatório antes de subir para produção (`[09:46]` Larissa/Sofia; `tests/orders.test.ts`).
- **Hipótese:** validar em piloto com um cliente antes do lançamento geral, usando o histórico de entregas como evidência do que foi enviado e do que o cliente conseguiu processar. Não foi proposto na reunião; entra ancorado no compromisso do Product Manager de alinhar a entrega diretamente com os clientes (`[09:47]` Marcos).
