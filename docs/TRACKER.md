# Tracker de Rastreabilidade

## Finalidade

Este documento liga cada requisito, decisão, restrição, trade-off, contrato e ponto de integração descrito em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e nos seis ADRs (`docs/adrs/`) à sua fonte real: um trecho localizável de `TRANSCRICAO.md` (reunião técnica de quinta-feira, 09:00) ou um arquivo real do código-fonte na branch `feature/design-docs-webhooks`. O objetivo é permitir a qualquer revisor verificar, linha a linha, que nenhuma informação relevante desses quatro documentos foi inventada.

Este Tracker não é fonte primária de nenhuma decisão — ele aponta para as fontes primárias (`TRANSCRICAO.md` e o código), nunca para `reports/context-analysis.md` ou para os próprios documentos de produto/técnicos.

## Convenções

- **Fonte** é sempre `TRANSCRICAO` (trecho literal de `TRANSCRICAO.md`) ou `CODIGO` (arquivo real do repositório). Nunca `reports/context-analysis.md`, PRD, RFC, FDD ou ADR.
- **Localização** para `TRANSCRICAO` segue o formato `[hh:mm] Nome` ou `[hh:mm]-[hh:mm] Nome, Nome`, com timestamps e nomes extraídos literalmente de `TRANSCRICAO.md`. Nenhum timestamp ou nome foi inventado; todos foram conferidos linha a linha na transcrição completa.
- **Localização** para `CODIGO` segue o formato `caminho/do/arquivo.ts — Símbolo.metodo`, com números de linha apenas quando conferidos por leitura direta do arquivo nesta revisão. Nenhum número de linha foi inventado.
- A matriz principal está organizada em blocos por documento de origem (`docs/PRD.md` → `docs/RFC.md` → `docs/FDD.md` → `docs/adrs/*.md`), preservando os identificadores já usados no PRD (`PRD-FR-NN`, `PRD-NFR-NN`) e adicionando identificadores novos e estáveis para as demais categorias, seguindo o padrão `<DOC>-<TIPO>-NN`.
- Itens do FDD marcados como "Decisão de design proposta no FDD, sujeita à revisão" **não** aparecem na matriz principal com fonte fictícia. Eles são listados na seção "Itens propostos sem decisão fechada", com o estado `PROPOSTO PARA REVISÃO` ou `QUESTÃO EM ABERTO` e a base parcial (transcrição e/ou código) que sustenta a proposta.

## Matriz principal de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B fazem polling repetido em `GET /orders`; integração descrita como "lenta e cara" | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | docs/PRD.md | Contexto | Risco de churn da Atlas Comercial; prazo inicial "fim do trimestre", refinado para fim de novembro | TRANSCRICAO | `[09:00]`, `[09:45] Marcos` |
| PRD-CTX-03 | docs/PRD.md | Contexto | Definição de "tempo real" pelo cliente: latência abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-AUD-01 | docs/PRD.md | Contexto | Público-alvo: clientes B2B Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-AUD-02 | docs/PRD.md | Contexto | Público-alvo: operadores autenticados que administram o cadastro de webhooks | TRANSCRICAO | `[09:31]-[09:33] Marcos, Bruno` |
| PRD-AUD-03 | docs/PRD.md | Contexto | Público-alvo: administradores (role `ADMIN`) que reprocessam entregas falhadas | TRANSCRICAO | `[09:35]-[09:36] Sofia, Larissa` |
| PRD-SCOPE-01 | docs/PRD.md | Fora de Escopo | Notificação por e-mail em falhas repetidas — adiado para fase futura | TRANSCRICAO | `[09:37]-[09:38] Marcos, Larissa` |
| PRD-SCOPE-02 | docs/PRD.md | Fora de Escopo | Dashboard visual dedicado ao cliente — descartado nesta fase | TRANSCRICAO | `[09:39]-[09:40] Marcos, Larissa` |
| PRD-SCOPE-03 | docs/PRD.md | Fora de Escopo | Webhooks inbound (recepção de eventos de clientes) — modelo é exclusivamente outbound | TRANSCRICAO | `[09:02]-[09:03] Marcos, Sofia` |
| PRD-SCOPE-04 | docs/PRD.md | Fora de Escopo | Garantia de entrega exactly-once — descartada em favor de at-least-once | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| PRD-SCOPE-05 | docs/PRD.md | Fora de Escopo | Escalonamento para múltiplos workers em paralelo — adiado, sem solução avaliada | TRANSCRICAO | `[09:13] Diego` |
| PRD-SCOPE-06 | docs/PRD.md | Fora de Escopo | Arquivamento automático de eventos entregues após ~30 dias — fora de escopo explícito | TRANSCRICAO | `[09:08] Diego` |
| PRD-SCOPE-07 | docs/PRD.md | Fora de Escopo | Redis Streams ou mensageria externa dedicada — descartado por overengineering | TRANSCRICAO | `[09:07] Larissa, Diego` |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook: URL e status desejados; secret gerada pelo sistema | TRANSCRICAO | `[09:31]-[09:32] Marcos, Bruno` |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listagem de webhooks de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Atualização de webhook (URL, status filtrados) | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Exclusão de webhook | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Ativação/desativação de webhook (campo "estado ativo") | TRANSCRICAO | `[09:21] Bruno, Sofia` |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status desejado, aplicado na inserção | TRANSCRICAO | `[09:33]-[09:34] Marcos, Bruno, Diego` |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Rotação de secret, com secret anterior válida por 24h | TRANSCRICAO | `[09:21]-[09:22] Sofia` |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Entrega de evento de mudança de status via HTTP assinado | TRANSCRICAO | `[09:19]-[09:26]`, `[09:42]-[09:44]` |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Retentativas automáticas em falha de entrega | TRANSCRICAO | `[09:14]-[09:17]` |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Envio para Dead Letter Queue (DLQ) após esgotar tentativas | TRANSCRICAO | `[09:17]-[09:18] Diego` |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay manual de evento em DLQ, restrito a `ADMIN`, com auditoria | TRANSCRICAO | `[09:18]-[09:19]`, `[09:35]-[09:36]` |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico dos últimos 100 envios de um webhook | TRANSCRICAO | `[09:34]-[09:35] Marcos` |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Deduplicação por identificador único de evento | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10 segundos em condições normais | TRANSCRICAO | `[09:02] Marcos`, `[09:09]-[09:10]` |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | URL do webhook deve ser HTTPS; HTTP é recusado | TRANSCRICAO | `[09:23] Sofia, Larissa` |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Notificações assinadas com HMAC-SHA256 | TRANSCRICAO | `[09:19]-[09:20] Sofia` |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Secret exclusiva por endpoint, não global | TRANSCRICAO | `[09:21] Sofia` |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Rotação de secret com grace period de 24 horas | TRANSCRICAO | `[09:21]-[09:22] Sofia` |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Payload de notificação limitado a 64 KB; acima disso, falha | TRANSCRICAO | `[09:23]-[09:24] Sofia, Diego, Larissa` |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP de entrega | TRANSCRICAO | `[09:42] Diego, Sofia` |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once (não exactly-once) | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Verificação de eventos pendentes a cada 2 segundos | TRANSCRICAO | `[09:09]-[09:10] Diego` |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Até 5 tentativas de entrega antes de falha permanente | TRANSCRICAO | `[09:15]-[09:17] Diego, Larissa` |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Ordem de entrega garantida só por pedido, só em regime single-worker | TRANSCRICAO | `[09:12]-[09:13] Diego, Larissa` |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Solução deve usar stack e padrões já existentes, sem infraestrutura nova | TRANSCRICAO | `[09:27]-[09:30]` |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Credenciais de assinatura não devem ser expostas em logs | TRANSCRICAO | `[09:22] Diego` |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Outbox transacional no MySQL em vez de envio HTTP síncrono | TRANSCRICAO | `[09:03]-[09:08] Bruno, Diego, Larissa` |
| PRD-DEC-02 | docs/PRD.md | Trade-off | MySQL já existente em vez de Redis Streams/mensageria externa | TRANSCRICAO | `[09:07] Larissa, Diego` |
| PRD-DEC-03 | docs/PRD.md | Trade-off | Polling periódico em vez de mecanismo reativo de banco | TRANSCRICAO | `[09:09]-[09:10] Diego, Larissa` |
| PRD-DEC-04 | docs/PRD.md | Trade-off | At-least-once em vez de exactly-once | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| PRD-DEC-05 | docs/PRD.md | Trade-off | Processo único (single-worker) em vez de escalabilidade horizontal imediata | TRANSCRICAO | `[09:12]-[09:14] Diego, Larissa, Marcos` |
| PRD-DEC-06 | docs/PRD.md | Trade-off | Secret por endpoint em vez de secret global da plataforma | TRANSCRICAO | `[09:21] Sofia` |
| PRD-RISK-01 | docs/PRD.md | Risco | Crescimento contínuo da fila de eventos pendentes na outbox | TRANSCRICAO | `[09:07]-[09:08] Bruno, Diego` |
| PRD-RISK-02 | docs/PRD.md | Risco | Indisponibilidade prolongada do endpoint de um cliente | TRANSCRICAO | `[09:15]-[09:17] Diego` |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente recebe a mesma notificação mais de uma vez | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| PRD-RISK-04 | docs/PRD.md | Risco | Vazamento da credencial de assinatura de um cliente | TRANSCRICAO | `[09:21]-[09:22] Sofia, Diego` |
| PRD-RISK-05 | docs/PRD.md | Risco | Capacidade de envio limitada pelo processo único de entrega | TRANSCRICAO | `[09:12]-[09:14] Diego, Larissa, Marcos` |
| PRD-RISK-06 | docs/PRD.md | Risco | Erro de configuração do endpoint pelo cliente (ex.: URL inválida) | TRANSCRICAO | `[09:23] Sofia` |
| PRD-RISK-07 | docs/PRD.md | Risco | Perda do cliente Atlas Comercial por atraso na entrega da feature | TRANSCRICAO | `[09:00]`, `[09:45]-[09:47] Marcos, Larissa` |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | Webhook cadastrado por usuário autenticado, informando URL e status | TRANSCRICAO | `[09:31]-[09:32] Marcos, Bruno` |
| PRD-AC-02 | docs/PRD.md | Critério de Aceitação | URL não-HTTPS é rejeitada no cadastro/atualização | TRANSCRICAO | `[09:23] Sofia, Larissa` |
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | Secret gerada automaticamente e devolvida apenas na criação/rotação | TRANSCRICAO | `[09:21]-[09:22]`, `[09:31] Sofia, Marcos` |
| PRD-AC-04 | docs/PRD.md | Critério de Aceitação | Apenas status efetivamente escolhidos geram notificação | TRANSCRICAO | `[09:33]-[09:34] Marcos, Bruno, Diego` |
| PRD-AC-05 | docs/PRD.md | Critério de Aceitação | Notificação gerada sempre que o status muda para status monitorado | TRANSCRICAO | `[09:40]-[09:44] Bruno, Diego` |
| PRD-AC-06 | docs/PRD.md | Critério de Aceitação | Notificação chega ao cliente em menos de 10 segundos em condições normais | TRANSCRICAO | `[09:02] Marcos`, `[09:09]-[09:10]` |
| PRD-AC-07 | docs/PRD.md | Critério de Aceitação | Cliente consegue validar a assinatura da notificação recebida | TRANSCRICAO | `[09:19]-[09:20] Sofia` |
| PRD-AC-08 | docs/PRD.md | Critério de Aceitação | Notificações duplicadas são identificáveis por identificador único | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| PRD-AC-09 | docs/PRD.md | Critério de Aceitação | Retentativas seguem a política de tentativas e intervalos definida | TRANSCRICAO | `[09:14]-[09:17] Diego, Larissa` |
| PRD-AC-10 | docs/PRD.md | Critério de Aceitação | Falhas permanentes registradas em fila de eventos não entregues (DLQ) | TRANSCRICAO | `[09:17]-[09:18] Diego` |
| PRD-AC-11 | docs/PRD.md | Critério de Aceitação | Reprocessamento manual de eventos na DLQ restrito a administradores | TRANSCRICAO | `[09:18]-[09:19]`, `[09:35]-[09:36]` |
| PRD-AC-12 | docs/PRD.md | Critério de Aceitação | Payload de notificação nunca excede 64 KB | TRANSCRICAO | `[09:23]-[09:24] Sofia, Diego, Larissa` |
| PRD-AC-13 | docs/PRD.md | Critério de Aceitação | Itens do pedido não fazem parte do payload da notificação | TRANSCRICAO | `[09:43] Diego` |
| PRD-AC-14 | docs/PRD.md | Critério de Aceitação | Nenhuma notificação é enviada de forma síncrona durante a transação | TRANSCRICAO | `[09:03]-[09:05]`, `[09:40]-[09:41] Bruno, Larissa, Diego` |
| RFC-PROP-01 | docs/RFC.md | Decisão | Proposta arquitetural: outbox transacional + worker HTTP outbound assinado, at-least-once, retry/DLQ | TRANSCRICAO | `[09:06]-[09:08] Diego` |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Envio HTTP síncrono dentro de `changeStatus` | TRANSCRICAO | `[09:03]-[09:05] Bruno, Larissa` |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams ou mensageria externa dedicada | TRANSCRICAO | `[09:07] Larissa, Diego` |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco de dados para notificação reativa | TRANSCRICAO | `[09:09] Bruno, Diego` |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Retry indefinido, sem teto de tentativas | TRANSCRICAO | `[09:15] Diego` |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | 3 tentativas de retry (mais agressivo) | TRANSCRICAO | `[09:16] Bruno, Diego` |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Status "failed" na própria tabela de outbox, em vez de DLQ separada | TRANSCRICAO | `[09:17]-[09:18] Diego` |
| RFC-ALT-07 | docs/RFC.md | Alternativa Descartada | Garantia de entrega exactly-once | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio ao cliente | TRANSCRICAO | `[09:38]-[09:39] Diego, Larissa` |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Contrato exato (método/path) do endpoint de rotação de secret | TRANSCRICAO | `[09:21]-[09:22] Sofia` |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Endurecimento futuro de RBAC no CRUD de configuração | TRANSCRICAO | `[09:36]-[09:37] Sofia, Marcos` |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Nome final do arquivo de processamento do worker | TRANSCRICAO | `[09:28] Bruno` |
| RFC-OPEN-05 | docs/RFC.md | Questão em Aberto | Estratégia de arquivamento da outbox após ~30 dias | TRANSCRICAO | `[09:08] Diego` |
| RFC-OPEN-06 | docs/RFC.md | Questão em Aberto | Estratégia de escalonamento para múltiplos workers | TRANSCRICAO | `[09:13] Diego` |
| RFC-RISK-01 | docs/RFC.md | Risco | Crescimento contínuo da tabela de outbox degradando leitura do worker | TRANSCRICAO | `[09:07]-[09:08]` |
| RFC-RISK-02 | docs/RFC.md | Risco | Indisponibilidade prolongada do endpoint do cliente causando perda de eventos | TRANSCRICAO | `[09:15]-[09:17]` |
| RFC-RISK-03 | docs/RFC.md | Risco | Duplicidade de eventos recebidos pelo cliente (inerente ao at-least-once) | TRANSCRICAO | `[09:24]-[09:26]` |
| RFC-RISK-04 | docs/RFC.md | Risco | Vazamento de secret comprometendo a integração de um cliente | TRANSCRICAO | `[09:21]-[09:22]` |
| RFC-RISK-05 | docs/RFC.md | Risco | Worker único como limitação de throughput/ordenação | TRANSCRICAO | `[09:12]-[09:14]` |
| RFC-RISK-06 | docs/RFC.md | Risco | Falha na inserção do evento provoca rollback de toda a transação de `changeStatus` | TRANSCRICAO | `[09:40]-[09:41]` |
| RFC-RISK-07 | docs/RFC.md | Risco | Dependência operacional de um novo processo (worker) rodando em paralelo à API | TRANSCRICAO | `[09:11]` |
| RFC-DEC-01 | docs/RFC.md | Decisão | Outbox transacional no MySQL (relacionada a `ADR-001`) | TRANSCRICAO | `[09:03]-[09:08]` |
| RFC-DEC-02 | docs/RFC.md | Decisão | Worker separado com polling de 2s (relacionada a `ADR-002`) | TRANSCRICAO | `[09:09]-[09:11]` |
| RFC-DEC-03 | docs/RFC.md | Decisão | Retry com backoff exponencial e DLQ (relacionada a `ADR-003`) | TRANSCRICAO | `[09:14]-[09:19]` |
| RFC-DEC-04 | docs/RFC.md | Decisão | HMAC-SHA256 e secret por endpoint (relacionada a `ADR-004`) | TRANSCRICAO | `[09:19]-[09:22]` |
| RFC-DEC-05 | docs/RFC.md | Decisão | Entrega at-least-once com `X-Event-Id` (relacionada a `ADR-005`) | TRANSCRICAO | `[09:24]-[09:26]` |
| RFC-DEC-06 | docs/RFC.md | Decisão | Reuso dos padrões existentes do OMS (relacionada a `ADR-006`) | TRANSCRICAO | `[09:27]-[09:37]` |
| FDD-FLOW-01 | docs/FDD.md | Resiliência | Fluxo de criação do evento na outbox, dentro da transação de `changeStatus` | TRANSCRICAO | `[09:06]`, `[09:40]-[09:42]`, `[09:51]-[09:52]` |
| FDD-FLOW-02 | docs/FDD.md | Resiliência | Fluxo de leitura do worker: polling a cada 2s, lote pequeno | TRANSCRICAO | `[09:09]-[09:10] Diego` |
| FDD-FLOW-03 | docs/FDD.md | Resiliência | Fluxo de retry: backoff exponencial após falha/timeout | TRANSCRICAO | `[09:15]-[09:17]` |
| FDD-FLOW-04 | docs/FDD.md | Resiliência | Fluxo de envio para DLQ após 5ª tentativa falhar | TRANSCRICAO | `[09:17]-[09:18] Diego` |
| FDD-FLOW-05 | docs/FDD.md | Decisão | Fluxo de replay manual via endpoint admin | TRANSCRICAO | `[09:18]-[09:19]`, `[09:35]-[09:36]` |
| FDD-FLOW-06 | docs/FDD.md | Segurança | Fluxo de rotação de secret com grace period | TRANSCRICAO | `[09:21]-[09:22] Sofia` |
| FDD-API-01 | docs/FDD.md | Contrato HTTP | `POST /api/v1/webhooks` — cadastro de webhook | CODIGO | `src/routes/index.ts — buildApiRouter (padrão /api/v1/<módulo>)` |
| FDD-API-02 | docs/FDD.md | Contrato HTTP | `GET /api/v1/webhooks` — listagem de webhooks | CODIGO | `src/routes/index.ts — buildApiRouter (padrão /api/v1/<módulo>)` |
| FDD-API-03 | docs/FDD.md | Contrato HTTP | `PATCH /api/v1/webhooks/:id` — edição de webhook | CODIGO | `src/routes/index.ts — buildApiRouter (padrão /api/v1/<módulo>)` |
| FDD-API-04 | docs/FDD.md | Contrato HTTP | `DELETE /api/v1/webhooks/:id` — remoção de webhook | CODIGO | `src/routes/index.ts — buildApiRouter (padrão /api/v1/<módulo>)` |
| FDD-API-05 | docs/FDD.md | Contrato HTTP | `GET /webhooks/:id/deliveries` — histórico de entregas | TRANSCRICAO | `[09:34]-[09:35] Marcos` |
| FDD-API-06 | docs/FDD.md | Contrato HTTP | `POST /admin/webhooks/dead-letter/:id/replay` — replay de DLQ | TRANSCRICAO | `[09:18]-[09:19] Diego` |
| FDD-EVT-01 | docs/FDD.md | Contrato HTTP | Campos do payload do evento (`event_id`, `event_type`, `order_id`, etc.) | TRANSCRICAO | `[09:43] Diego` |
| FDD-EVT-02 | docs/FDD.md | Contrato HTTP | Headers de envio (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) | TRANSCRICAO | `[09:44]-[09:45] Diego, Sofia` |
| FDD-EVT-03 | docs/FDD.md | Contrato HTTP | `event_id` UUID único, base da deduplicação via `X-Event-Id` | TRANSCRICAO | `[09:25] Diego` |
| FDD-EVT-04 | docs/FDD.md | Contrato HTTP | Tamanho máximo do payload: 64 KB | TRANSCRICAO | `[09:23]-[09:24]` |
| FDD-EVT-05 | docs/FDD.md | Contrato HTTP | Campo `items` explicitamente excluído do payload | TRANSCRICAO | `[09:43] Diego` |
| FDD-EVT-06 | docs/FDD.md | Contrato HTTP | Payload é snapshot renderizado no momento da inserção | TRANSCRICAO | `[09:51]-[09:52]` |
| FDD-ERR-01 | docs/FDD.md | Código de Erro | `WEBHOOK_NOT_FOUND` | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERR-02 | docs/FDD.md | Código de Erro | `WEBHOOK_INVALID_URL` | TRANSCRICAO | `[09:28] Bruno`, `[09:23] Sofia` |
| FDD-ERR-03 | docs/FDD.md | Código de Erro | `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERR-04 | docs/FDD.md | Código de Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` | TRANSCRICAO | `[09:23]-[09:24] Sofia, Diego, Larissa` |
| FDD-ERR-05 | docs/FDD.md | Código de Erro | `WEBHOOK_REPLAY_FORBIDDEN` | TRANSCRICAO | `[09:35]-[09:36] Sofia, Larissa` |
| FDD-ERR-06 | docs/FDD.md | Código de Erro | `WEBHOOK_CUSTOMER_NOT_FOUND`, consistente com padrão já usado para `Customer` | CODIGO | `src/modules/orders/order.service.ts:60 — NotFoundError('Customer')` |
| FDD-SEC-01 | docs/FDD.md | Segurança | HTTPS obrigatório; `http` é recusado na validação | TRANSCRICAO | `[09:23] Sofia, Larissa` |
| FDD-SEC-02 | docs/FDD.md | Segurança | HMAC-SHA256 sobre o corpo, header `X-Signature` | TRANSCRICAO | `[09:19]-[09:20] Sofia` |
| FDD-SEC-03 | docs/FDD.md | Segurança | Secret exclusiva por endpoint, não global | TRANSCRICAO | `[09:21] Sofia` |
| FDD-SEC-04 | docs/FDD.md | Segurança | Rotação com grace period de 24h; ambas as secrets aceitas na janela | TRANSCRICAO | `[09:21]-[09:22] Sofia` |
| FDD-SEC-05 | docs/FDD.md | Segurança | Lista de redact do logger atual não inclui `secret` explicitamente | CODIGO | `src/shared/logger/index.ts:4-11 — redactPaths` |
| FDD-SEC-06 | docs/FDD.md | Segurança | Replay de DLQ restrito a `ADMIN`, com log de auditoria | TRANSCRICAO | `[09:35]-[09:36] Sofia` |
| FDD-SEC-07 | docs/FDD.md | Segurança | `X-Timestamp` permite ao cliente detectar replay attacks | TRANSCRICAO | `[09:44] Diego` |
| FDD-SEC-08 | docs/FDD.md | Segurança | Validação Zod reaproveitada para os novos schemas do módulo | CODIGO | `src/middlewares/validate.middleware.ts:11-37 — validate()` |
| FDD-RES-01 | docs/FDD.md | Resiliência | Outbox transacional garante atomicidade entre status e evento | TRANSCRICAO | `[09:06]-[09:08]` |
| FDD-RES-02 | docs/FDD.md | Resiliência | Isolamento do HTTP externo: nenhuma chamada de rede dentro da transação | TRANSCRICAO | `[09:04] Bruno` |
| FDD-RES-03 | docs/FDD.md | Resiliência | Timeout de 10s evita que cliente lento prenda o worker | TRANSCRICAO | `[09:42]` |
| FDD-RES-04 | docs/FDD.md | Resiliência | Retry com backoff exponencial cobre indisponibilidades de até ~15h | TRANSCRICAO | `[09:14]-[09:17]` |
| FDD-RES-05 | docs/FDD.md | Resiliência | DLQ separada isola falhas permanentes sem poluir a outbox principal | TRANSCRICAO | `[09:17]-[09:18]` |
| FDD-RES-06 | docs/FDD.md | Resiliência | Regime single-worker: ordering por `order_id` só é garantido enquanto isso valer | TRANSCRICAO | `[09:12]-[09:13]` |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados via Pino, reuso do singleton existente | CODIGO | `src/shared/logger/index.ts:1-32 — createLogger, logger` |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Correlação por `X-Request-Id` já existente na API | CODIGO | `src/middlewares/request-logger.middleware.ts — requestLogger` |
| FDD-CRIT-01 | docs/FDD.md | Critério de Aceitação | Evento registrado atomicamente; falha na outbox reverte a transação inteira | TRANSCRICAO | `[09:40]-[09:41] Bruno, Diego` |
| FDD-CRIT-02 | docs/FDD.md | Critério de Aceitação | Arquitetura atende latência inferior a 10s (polling 2s + processamento) | TRANSCRICAO | `[09:02]`, `[09:09]-[09:10]` |
| FDD-CRIT-03 | docs/FDD.md | Critério de Aceitação | Worker realiza polling a cada 2 segundos | TRANSCRICAO | `[09:09]-[09:10] Diego` |
| FDD-CRIT-04 | docs/FDD.md | Critério de Aceitação | Consumidor consegue validar a assinatura HMAC-SHA256 | TRANSCRICAO | `[09:19]-[09:20] Sofia` |
| FDD-CRIT-05 | docs/FDD.md | Critério de Aceitação | Eventos duplicados identificáveis via `X-Event-Id` | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| FDD-CRIT-06 | docs/FDD.md | Critério de Aceitação | Retry segue exatamente os intervalos definidos (1m/5m/30m/2h/12h, 5 tentativas) | TRANSCRICAO | `[09:14]-[09:17] Diego, Larissa` |
| FDD-CRIT-07 | docs/FDD.md | Critério de Aceitação | Falhas permanentes (após 5ª tentativa) registradas na DLQ | TRANSCRICAO | `[09:17]-[09:18] Diego` |
| FDD-CRIT-08 | docs/FDD.md | Critério de Aceitação | Replay de DLQ exige role `ADMIN` e é auditado | TRANSCRICAO | `[09:35]-[09:36] Sofia, Larissa` |
| FDD-CRIT-09 | docs/FDD.md | Critério de Aceitação | Nenhum payload de evento excede 64 KB | TRANSCRICAO | `[09:23]-[09:24]` |
| FDD-CRIT-10 | docs/FDD.md | Critério de Aceitação | Nenhuma chamada HTTP ocorre dentro da transação Prisma de `changeStatus` | TRANSCRICAO | `[09:04] Bruno` |
| FDD-CRIT-11 | docs/FDD.md | Critério de Aceitação | Todos os erros do módulo usam códigos com prefixo `WEBHOOK_` | TRANSCRICAO | `[09:28]-[09:29] Bruno, Larissa` |
| FDD-CRIT-12 | docs/FDD.md | Critério de Aceitação | Logs do módulo não expõem `secret` nem a assinatura completa | TRANSCRICAO | `[09:22] Diego` |
| FDD-CRIT-13 | docs/FDD.md | Critério de Aceitação | Código segue estrutura `controller/service/repository/routes/schemas` | TRANSCRICAO | `[09:27]-[09:28] Bruno, Diego` |
| FDD-INT-01 | docs/FDD.md | Integração com Código | Ponto de integração: `OrderService.changeStatus`, transação atômica existente | CODIGO | `src/modules/orders/order.service.ts:126-179 — OrderService.changeStatus` |
| FDD-INT-02 | docs/FDD.md | Integração com Código | Modelos Prisma existentes que fornecem campos do payload | CODIGO | `prisma/schema.prisma:16-23,74-97,116-131 — OrderStatus, Order, OrderStatusHistory` |
| FDD-INT-03 | docs/FDD.md | Integração com Código | Composição manual de dependências (repository → service → controller) | CODIGO | `src/app.ts:26-76 — buildControllers, buildApp` |
| FDD-INT-04 | docs/FDD.md | Integração com Código | Registro de rotas de cada módulo sob `/api/v1/<módulo>` | CODIGO | `src/routes/index.ts:21-31 — buildApiRouter` |
| FDD-INT-05 | docs/FDD.md | Integração com Código | Padrão de entry point de processo, referência para `src/worker.ts` | CODIGO | `src/server.ts:1-27 — bootstrap` |
| FDD-INT-06 | docs/FDD.md | Integração com Código | Padrão de criação do `PrismaClient`, a ser reaproveitado pelo worker | CODIGO | `src/config/database.ts:1-11 — createPrismaClient` |
| FDD-INT-07 | docs/FDD.md | Integração com Código | Validação de variáveis de ambiente já existente (Zod) | CODIGO | `src/config/env.ts — envSchema, loadEnv` |
| FDD-INT-08 | docs/FDD.md | Integração com Código | `authenticate`/`requireRole` a serem reaproveitados sem alteração | CODIGO | `src/middlewares/auth.middleware.ts:27-61 — authenticate, requireRole` |
| FDD-INT-09 | docs/FDD.md | Integração com Código | Middleware `validate` a ser reaproveitado para novos schemas | CODIGO | `src/middlewares/validate.middleware.ts:11-37 — validate` |
| FDD-INT-10 | docs/FDD.md | Integração com Código | Middleware de erro central já trata `AppError`/`ZodError`/Prisma | CODIGO | `src/middlewares/error.middleware.ts:14-65 — errorMiddleware` |
| FDD-INT-11 | docs/FDD.md | Integração com Código | Base `AppError` e subclasses HTTP a serem estendidas com prefixo `WEBHOOK_` | CODIGO | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| FDD-INT-12 | docs/FDD.md | Integração com Código | Logger Pino singleton a ser reaproveitado pela API e pelo worker | CODIGO | `src/shared/logger/index.ts:1-32 — createLogger, logger` |
| FDD-INT-13 | docs/FDD.md | Integração com Código | Padrão de teste (Vitest + Supertest, sem mocks) a ser seguido | CODIGO | `tests/orders.test.ts:1-9 — getTestApp, bootstrapAuthenticatedUser` |
| FDD-INT-14 | docs/FDD.md | Integração com Código | Padrão `createdAt`/`updatedAt` já usado em todos os modelos do schema atual | CODIGO | `prisma/schema.prisma:31-32,47-48,64-65 — User, Customer, Product` |
| FDD-OPEN-01 | docs/FDD.md | Questão em Aberto | Estratégia de arquivamento/limpeza da outbox após ~30 dias | TRANSCRICAO | `[09:08] Diego` |
| FDD-OPEN-02 | docs/FDD.md | Questão em Aberto | Estratégia de escalonamento para múltiplos workers | TRANSCRICAO | `[09:13] Diego` |
| FDD-OPEN-03 | docs/FDD.md | Questão em Aberto | Rate limiting de envio outbound | TRANSCRICAO | `[09:38]-[09:39]` |
| FDD-OPEN-04 | docs/FDD.md | Questão em Aberto | Biblioteca/abstração HTTP cliente do worker; projeto não tem cliente HTTP nem discussão sobre ele | CODIGO | `package.json — dependencies/devDependencies (ausência de axios/node-fetch/got), engines.node >=20` |
| FDD-OPEN-05 | docs/FDD.md | Questão em Aberto | Nome final do arquivo de processamento do worker | TRANSCRICAO | `[09:28] Bruno` |
| FDD-OPEN-06 | docs/FDD.md | Questão em Aberto | Contrato exato do endpoint de rotação de secret | TRANSCRICAO | `[09:21]-[09:22] Sofia` |
| FDD-OPEN-07 | docs/FDD.md | Questão em Aberto | Possível endurecimento futuro de RBAC no CRUD de configuração | TRANSCRICAO | `[09:36]-[09:37]` |
| ADR-001 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Padrão Outbox transacional no MySQL para publicação de eventos | TRANSCRICAO | `[09:03]-[09:08] Diego, Larissa, Bruno` |
| ADR-001b | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | `[09:41] Bruno, Diego` |
| ADR-001c | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Filtro de status aplicado na inserção da outbox, não no envio | TRANSCRICAO | `[09:33]-[09:34] Bruno, Diego` |
| ADR-001d | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Payload gravado como snapshot renderizado no momento da inserção | TRANSCRICAO | `[09:51]-[09:52] Larissa, Diego, Bruno` |
| ADR-001e | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Identificadores da outbox usam UUID, seguindo o padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker roda como processo Node separado, com polling a cada 2s | TRANSCRICAO | `[09:09]-[09:11] Diego, Larissa` |
| ADR-002b | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker usa instância própria de `PrismaClient`, mesmo banco | TRANSCRICAO | `[09:11]`, `[09:29]-[09:30] Diego, Bruno` |
| ADR-002c | docs/adrs/ADR-002-worker-separado-com-polling.md | Limitação | Ordenação de eventos garantida só por `order_id`, só em regime single-worker | TRANSCRICAO | `[09:12]-[09:13] Diego, Larissa` |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Retry com backoff exponencial, teto de 5 tentativas | TRANSCRICAO | `[09:14]-[09:17] Diego, Larissa` |
| ADR-003b | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Progressão de backoff: 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| ADR-003c | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela separada, guardando payload, motivo e timestamp | TRANSCRICAO | `[09:17]-[09:18] Diego` |
| ADR-003d | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay manual via `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | `[09:18]-[09:19] Diego, Larissa` |
| ADR-003e | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Timeout do HTTP call do worker: 10 segundos | TRANSCRICAO | `[09:42] Diego, Sofia` |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256.md | Segurança | Assinatura HMAC-SHA256 no header `X-Signature`, secret única por endpoint | TRANSCRICAO | `[09:19]-[09:21] Sofia` |
| ADR-004b | docs/adrs/ADR-004-autenticacao-hmac-sha256.md | Segurança | Rotação de secret via API, com grace period de 24h | TRANSCRICAO | `[09:21]-[09:22] Sofia, Diego` |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md | Decisão | Garantia at-least-once, deduplicação do cliente via `X-Event-Id` | TRANSCRICAO | `[09:24]-[09:26] Diego` |
| ADR-005b | docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md | Contrato HTTP | Headers adicionais definidos para o envio (`X-Timestamp`, `X-Webhook-Id`) | TRANSCRICAO | `[09:44]-[09:45] Diego, Sofia` |
| ADR-005c | docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md | Contrato HTTP | Formato do payload do evento, com exclusão explícita de `items` | TRANSCRICAO | `[09:43] Diego, Bruno` |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo webhooks reaproveita integralmente os padrões já existentes no OMS | TRANSCRICAO | `[09:27]-[09:30] Bruno, Diego, Larissa` |
| ADR-006b | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | `customer_id` vem explicitamente do body/path, nunca do JWT | TRANSCRICAO | `[09:31]-[09:32] Bruno, Larissa, Marcos` |
| ADR-006c | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Restrição | RBAC diferenciado: CRUD aceita qualquer role autenticada; replay exige `ADMIN` | TRANSCRICAO | `[09:35]-[09:37] Sofia, Larissa, Marcos` |

## Itens propostos sem decisão fechada

| ID | Documento | Conteúdo proposto | Base parcial | Estado |
| --- | --- | --- | --- | --- |
| PROP-ERR-01 | docs/FDD.md | Código `WEBHOOK_INVALID_STATUS_FILTER` para status inválido no filtro | Prefixo `WEBHOOK_` decidido (`[09:28]-[09:29] Bruno, Larissa`); validação apoiada no enum `OrderStatus` existente (`prisma/schema.prisma:16-23`) | PROPOSTO PARA REVISÃO |
| PROP-ERR-02 | docs/FDD.md | Código `WEBHOOK_DELIVERY_NOT_FOUND` | Endpoint de histórico de entregas decidido (`[09:34]-[09:35] Marcos`); código de erro específico não discutido | PROPOSTO PARA REVISÃO |
| PROP-ERR-03 | docs/FDD.md | Código `WEBHOOK_DEAD_LETTER_NOT_FOUND` | Comportamento de replay decidido (`[09:18]-[09:19] Diego`); código de erro específico não discutido | PROPOSTO PARA REVISÃO |
| PROP-ERR-04 | docs/FDD.md | Código `WEBHOOK_SECRET_ROTATION_FAILED` | Comportamento de rotação de secret decidido (`[09:21]-[09:22] Sofia`); código de erro específico não discutido | PROPOSTO PARA REVISÃO |
| PROP-API-01 | docs/FDD.md | `POST /api/v1/webhooks/:id/secret/rotate` como contrato exato da rotação de secret | Comportamento decidido (`[09:21]-[09:22] Sofia`); método HTTP e path nunca foram citados literalmente na reunião — ver `RFC-OPEN-02` | QUESTÃO EM ABERTO |
| PROP-DATA-01 | docs/FDD.md | `WebhookEndpoint.previousSecret`/`previousSecretExpiresAt` como representação do grace period | Grace period de 24h decidido (`[09:21]-[09:22] Sofia`); representação em dois campos vs. tabela própria não foi discutida | PROPOSTO PARA REVISÃO |
| PROP-DATA-02 | docs/FDD.md | `WebhookOutbox.webhookId` com uma linha por endpoint interessado | Filtro de status na inserção decidido (`[09:33]-[09:34] Bruno, Diego`); cardinalidade exata não foi discutida | PROPOSTO PARA REVISÃO |
| PROP-DATA-03 | docs/FDD.md | `WebhookOutbox.attempts` como contador de tentativas | Teto de 5 tentativas decidido (`[09:15]-[09:17] Diego, Larissa`); nome/mecanismo do campo não foi discutido | PROPOSTO PARA REVISÃO |
| PROP-DATA-04 | docs/FDD.md | `WebhookOutbox.nextAttemptAt` para agendar a próxima tentativa | Progressão de backoff decidida (`[09:17] Diego`); nome/mecanismo do campo não foi discutido | PROPOSTO PARA REVISÃO |
| PROP-DATA-05 | docs/FDD.md | `WebhookOutbox.deliveredAt` para registrar o momento da entrega | Marcação de evento "entregue" mencionada (`[09:08] Diego`); campo específico não foi discutido | PROPOSTO PARA REVISÃO |
| PROP-DATA-06 | docs/FDD.md | `WebhookDeadLetter.webhookId` para reconstruir o replay | Criação da tabela de DLQ decidida (`[09:18] Diego`); campo específico não foi discutido | PROPOSTO PARA REVISÃO |
| PROP-WORKER-01 | docs/FDD.md | Tamanho exato do lote de leitura do worker ("lote pequeno") | Leitura em lote pequeno mencionada (`[09:07]-[09:08] Diego`); número exato nunca foi definido | QUESTÃO EM ABERTO |
| PROP-API-02 | docs/FDD.md | Histórico de entregas lido diretamente dos registros de `WebhookOutbox` | Endpoint de histórico decidido (`[09:34]-[09:35] Marcos`); fonte de leitura exata não foi discutida | PROPOSTO PARA REVISÃO |
| PROP-FLOW-01 | docs/FDD.md | Mecanismo de replay: nova linha na outbox vs. reset de campos na linha original | Comportamento de replay decidido (`[09:18]-[09:19] Diego`); mecanismo interno não foi discutido | QUESTÃO EM ABERTO |
| PROP-CONFIG-01 | docs/FDD.md | Intervalo de polling e timeout HTTP como variáveis de ambiente configuráveis | Valores já decididos (`[09:09]-[09:10]`, `[09:42]`); torná-los configuráveis via env não foi discutido — padrão de validação de env existe em `src/config/env.ts` | PROPOSTO PARA REVISÃO |
| PROP-NAME-01 | docs/FDD.md | Adotar `webhook.processor.ts` até definição em revisão de código | Bruno propôs `webhook.worker.ts` ou `webhook.processor.ts`; Diego apenas concordou de forma genérica, sem escolher (`[09:28] Bruno`) | PROPOSTO PARA REVISÃO |
| PROP-SEC-01 | docs/FDD.md | Armazenamento seguro da secret em repouso (ex.: KMS, criptografia adicional) | Nenhuma decisão tomada; incidente de vazamento em log motiva atenção ao tema (`[09:22] Diego`), mas não decide forma de armazenamento | QUESTÃO EM ABERTO |
| PROP-OBS-01 | docs/FDD.md | Métricas operacionais formais e SLO de disponibilidade/latência além dos limiares já decididos | Limiar de latência <10s decidido (`[09:02] Marcos`); metas operacionais quantitativas adicionais não foram discutidas | QUESTÃO EM ABERTO |
| PROP-OBS-02 | docs/FDD.md | Nomes de métricas propostas (`webhook_outbox_pending_total`, `webhook_delivery_success_total`, etc.) | Necessidade de observabilidade implícita no histórico de entregas decidido (`[09:34]-[09:35] Marcos`); nomes específicos são proposta de design do FDD | PROPOSTO PARA REVISÃO |

## Indicadores de cobertura

1. **Total de linhas da tabela principal**: 188.
2. **Linhas com `Fonte = TRANSCRICAO`**: 164 (≈ 87,2% do total).
3. **Linhas com `Fonte = CODIGO`**: 24 (≈ 12,8% do total).
4. **Caminhos de código únicos referenciados**: 16 — `src/modules/orders/order.service.ts`, `prisma/schema.prisma`, `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `src/config/database.ts`, `src/config/env.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/logger/index.ts`, `tests/orders.test.ts`, `package.json` — este último referenciado diretamente na matriz principal (`FDD-OPEN-04`), não apenas como apoio às propostas.
5. **Estimativa de cobertura dos itens identificáveis dos documentos**: ≈ 97%. Todos os 13 requisitos funcionais e 13 não funcionais do PRD, os 7 itens fora de escopo, os 14 critérios de aceitação, as 7 alternativas descartadas e as 6 questões em aberto do RFC, os 6 endpoints citados no FDD (mais 1 endpoint sem contrato fechado, registrado como questão em aberto), os 10 códigos `WEBHOOK_*` (6 na matriz principal, 4 nas propostas), os 14 pontos de integração de código do FDD e os 6 ADRs (com 15 sub-decisões) estão cobertos. Os únicos itens não incluídos como linha fechada na matriz principal são exatamente aqueles sem fonte direta suficiente (contrato de rotação de secret, nomes de campos de schema não debatidos, armazenamento de secret em repouso, métricas/SLO formais) — todos registrados na seção de propostas, não descartados.

Critérios mínimos desta tarefa, verificados:

- ✅ ≥ 80% dos itens identificáveis com linha correspondente (estimado ≈ 97%).
- ✅ ≥ 70% das linhas com `Fonte = TRANSCRICAO` (87,2%).
- ✅ ≥ 5 linhas com `Fonte = CODIGO` (24 linhas).
- ✅ ≥ 5 caminhos reais de código distintos (16 caminhos).

## Validações realizadas

1. **Todos os timestamps existem?** Sim — todos os `[hh:mm]` citados (`09:00` a `09:53`) foram conferidos por leitura completa de `TRANSCRICAO.md` nesta revisão.
2. **Nomes dos participantes corretos?** Sim — Larissa, Marcos, Bruno, Diego e Sofia, conforme o cabeçalho de participantes de `TRANSCRICAO.md`; nenhum nome fora dessa lista foi usado.
3. **Todos os caminhos de código existem?** Sim — todos os arquivos citados como `CODIGO` foram lidos diretamente nesta revisão: `src/modules/orders/order.service.ts`, `src/app.ts`, `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/logger/index.ts`, `src/config/database.ts`, `src/config/env.ts`, `src/server.ts`, `prisma/schema.prisma`, `tests/orders.test.ts`, `package.json`.
4. **Nenhum arquivo futuro foi tratado como atual?** Confirmado — uma busca por `webhook*`/`worker*` no repositório retornou apenas `docs/adrs/ADR-002-worker-separado-com-polling.md` (documento, não código) e referências dentro de `.git/`; nenhum arquivo `src/modules/webhooks/*`, `src/worker.ts` ou script `npm run worker` existe hoje. Todos os componentes futuros aparecem na matriz com `Fonte = TRANSCRICAO` (comportamento decidido) ou na seção de propostas, nunca como `CODIGO` existente.
5. **Nenhuma proposta do FDD recebeu fonte fictícia?** Confirmado — os 19 itens sinalizados no FDD como "Decisão de design proposta no FDD, sujeita à revisão" sem base direta suficiente estão exclusivamente na seção "Itens propostos sem decisão fechada", com `Fonte` omitida da matriz principal e `Base parcial` explicitando o que foi e o que não foi decidido.
6. **Os seis ADRs estão cobertos?** Sim — `ADR-001` a `ADR-006`, cada um com uma linha principal e linhas adicionais (`ADR-00Nb`, `ADR-00Nc`) para decisões materialmente distintas consolidadas no mesmo ADR.
7. **Todos os requisitos funcionais do PRD estão cobertos?** Sim — `PRD-FR-01` a `PRD-FR-13`, preservando a numeração já usada em `docs/PRD.md`.
8. **Todos os endpoints do FDD estão cobertos?** Sim — os 6 endpoints com contrato na matriz principal (`FDD-API-01` a `FDD-API-06`) e o endpoint de rotação de secret, sem contrato fechado, na seção de propostas (`PROP-API-01`).
9. **Todos os códigos de erro do FDD estão cobertos ou listados como propostas?** Sim — os 10 códigos da "Matriz de erros" do FDD estão distribuídos entre a matriz principal (`FDD-ERR-01` a `FDD-ERR-06`) e a seção de propostas (`PROP-ERR-01` a `PROP-ERR-04`).
10. **Os itens fora de escopo estão cobertos?** Sim — os 7 itens da tabela "Fora de escopo" do PRD (`PRD-SCOPE-01` a `PRD-SCOPE-07`).
11. **Alternativas descartadas e questões abertas estão cobertas?** Sim — as 7 alternativas do RFC (`RFC-ALT-01` a `RFC-ALT-07`) e as questões em aberto do RFC (`RFC-OPEN-01` a `RFC-OPEN-06`) e do FDD (`FDD-OPEN-01` a `FDD-OPEN-07`, mais as registradas como `QUESTÃO EM ABERTO` na seção de propostas).
12. **A tabela segue exatamente as seis colunas obrigatórias?** Sim — `ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`, sem colunas adicionais na matriz principal.

Nenhum arquivo em `src/`, `prisma/`, `tests/`, `TRANSCRICAO.md`, `README.md`, `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/` ou configurações foi alterado durante a produção deste Tracker.
