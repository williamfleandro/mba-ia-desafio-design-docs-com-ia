# Análise Técnica Estruturada — Sistema de Webhooks de Notificação de Pedidos

> Documento de análise preparatória. Não é PRD, RFC, FDD, ADR, Tracker ou README.
> Base rastreável para a produção posterior desses documentos.
> Fontes: `TRANSCRICAO.md` (reunião de ~55 min, quinta-feira 09:00) e código-fonte do repositório na branch `feature/design-docs-webhooks`.

---

## A. Contexto de negócio

- **Problema atual**: três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) enviaram pedido formal para serem notificados em tempo real quando o status de seus pedidos muda. Hoje eles fazem *polling* repetido em `GET /orders`, o que Marcos descreve como uma integração "lenta e cara" para eles. `[09:00] Marcos`
- **Clientes/públicos afetados**: Atlas Comercial, MaxDistribuição, Nova Cargo — todos clientes B2B que consomem a API do OMS. `[09:00] Marcos`
- **Motivação**: eliminar a necessidade de polling manual/periódico por parte desses clientes. `[09:00] Marcos`
- **Urgência**: a Atlas sinalizou que pode migrar para um concorrente se a feature não for entregue até o fim do trimestre. `[09:00] Marcos`. Mais adiante na reunião, o prazo é refinado para fim de novembro. `[09:45] Marcos`
- **Definição de "tempo real" pelo cliente**: qualquer latência abaixo de 10 segundos é aceitável; o requisito central do cliente é não precisar atualizar manualmente. `[09:02] Marcos`
- **Impacto esperado**: retenção do cliente Atlas (e por extensão dos demais clientes B2B que fizeram o mesmo pedido) e redução da carga de polling sobre `GET /orders`. Não há métrica de redução de tráfego quantificada na reunião — apenas a motivação qualitativa.
- **Definição de sucesso**: não foi formalizada de modo quantitativo além do limiar de latência de 10s. Ver `OPEN-003` na seção G.
- **Métricas/metas quantitativas mencionadas nesta seção**: limiar de "tempo real" < 10s `[09:02] Marcos`; prazo de negócio fim de novembro `[09:45] Marcos`; estimativa de 3 sprints `[09:46] Larissa`. Detalhamento completo na seção J.

---

## B. Participantes e responsabilidades

| Participante | Função | Temas em que contribuiu |
| --- | --- | --- |
| Larissa | Tech Lead (conduz a reunião) | Direciona a pauta, fecha decisões arquiteturais, decide UUID e snapshot do payload, resume a reunião no final `[09:48]` |
| Marcos | Product Manager | Traz o contexto do cliente, requisitos funcionais de CRUD e histórico de entregas, prazos de negócio, decide o que fica fora de escopo (e-mail, dashboard) do ponto de vista de produto |
| Bruno | Engenheiro Pleno, time de Pedidos | Questiona processamento síncrono, detalha a transação atual de `changeStatus`, propõe estrutura de pastas/módulo, propõe `publishWebhookEvent`, propõe códigos de erro `WEBHOOK_*` |
| Diego | Engenheiro Sênior, time de Plataforma (entra às `[09:05]`) | Propõe e explica o padrão Outbox, polling do worker, retry/backoff, DLQ, at-least-once + `X-Event-Id`, headers, timeout, payload, confirma UUID e snapshot |
| Sofia | Engenheira de Segurança | HMAC-SHA256, secret por endpoint, rotação com grace period, TLS obrigatório, limite de payload, exige role `ADMIN` para replay e auditoria |

---

## C. Decisões arquiteturais fechadas

| ID | Descrição | Justificativa | Responsável | Timestamp | Trade-offs | Consequências | ADR relacionado |
| --- | --- | --- | --- | --- | --- | --- | --- |
| DEC-001 | Webhooks são exclusivamente *outbound* (sistema → cliente); não há recepção de webhooks de clientes | Cliente só quer receber notificações, não enviar | Marcos, Sofia | `[09:02]-[09:03]` | Simplifica o escopo de segurança (não precisa validar payloads de entrada de terceiros) | Reduz superfície de ataque; escopo mais simples | ADR-004 (indiretamente) |
| DEC-002 | Padrão Outbox em tabela MySQL (não síncrono, não fila externa tipo Redis) | Síncrono trava outras mudanças de status se cliente estiver lento/fora do ar; fila externa é overengineering para time pequeno | Diego, Larissa | `[09:03]-[09:08]` | Vs. síncrono: ganha resiliência, perde simplicidade imediata. Vs. Redis: evita nova infra, mas fica preso ao MySQL | Garante atomicidade entre mudança de status e evento; exige worker de leitura | ADR-001 |
| DEC-003 | Worker lê a outbox via polling em loop, a cada 2 segundos | MySQL não tem `NOTIFY/LISTEN` nativo; trigger de banco não notifica processo externo de forma limpa | Diego | `[09:09]-[09:10]` | Polling é mais simples, mas introduz latência mínima de 2s | Atende ao requisito de <10s do cliente com folga | ADR-002 |
| DEC-004 | Worker roda como processo Node separado da API (`src/worker.ts`, script `npm run worker`) | Se rodasse na mesma instância da API, reinício da API derrubaria o worker | Diego, Larissa | `[09:11]` | Mais um processo para operar/monitorar | Maior resiliência operacional | ADR-002 |
| DEC-005 | Worker conecta no mesmo banco, mas com PrismaClient próprio (instância separada por processo) | `PrismaClient` é por processo; mesmo processo Node não pode ser compartilhado entre API e worker | Diego, Bruno | `[09:11]`, `[09:29]-[09:30]` | Nenhum discutido explicitamente além da necessidade de nova instância | Reuso de configuração (`DATABASE_URL`), sem reuso de instância em memória | ADR-002, ADR-007 |
| DEC-006 | Ordenação garantida apenas por `order_id`, e apenas enquanto o sistema operar em regime single-worker | Consequência direta do polling sequencial por `created_at`; escalar para múltiplos workers quebra a garantia | Diego, Larissa | `[09:12]-[09:13]` | Simplicidade agora vs. limitação de escalabilidade futura | Documentado como limitação conhecida (não é bug) | ADR-008 |
| DEC-007 | Retry com backoff exponencial, 5 tentativas | 3 tentativas seria insuficiente para cobrir indisponibilidades reais já observadas (cliente com 2h de manutenção); retry indefinido deixaria eventos pendurados para sempre | Diego, Larissa | `[09:14]-[09:17]` | Mais tentativas = maior janela de recuperação, mas maior tempo até declarar falha permanente | Cobre até ~15h de indisponibilidade do cliente antes de ir para DLQ | ADR-003 |
| DEC-008 | Progressão de backoff: 1min / 5min / 30min / 2h / 12h | Definida por Diego para cobrir janela de ~15h | Diego | `[09:17]` | — | Total de ~15h entre primeira falha e última tentativa | ADR-003 |
| DEC-009 | DLQ em tabela separada (`webhook_dead_letter`), não como status "failed" na própria outbox | Mantém a outbox principal limpa para leitura do worker; serve como evidência para debug/reprocessamento | Diego | `[09:17]-[09:18]` | Mais uma tabela para manter | Facilita auditoria e replay | ADR-003 |
| DEC-010 | Replay manual de eventos em DLQ via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay` | Reprocessamento precisa ser uma ação explícita, não automática | Diego, Larissa | `[09:18]-[09:19]` | Exige intervenção manual/operacional | Recoloca evento como pendente na outbox | ADR-003 |
| DEC-011 | Assinatura de payload com HMAC-SHA256, enviada no header `X-Signature` | Cliente precisa validar autenticidade e integridade do payload; SHA-256 é padrão de mercado com suporte amplo | Sofia | `[09:19]-[09:20]` | Exige que o cliente implemente verificação HMAC do lado dele | Garante autenticidade/integridade do payload | ADR-004 |
| DEC-012 | Secret única por endpoint de webhook (não secret global da plataforma) | Reduz o "blast radius" caso uma secret vaze | Sofia | `[09:21]` | Mais complexidade de armazenamento/gestão de múltiplas secrets | Vazamento de uma secret não compromete outros clientes | ADR-004 |
| DEC-013 | Rotação de secret suportada via API, com secret antiga válida em paralelo por 24h (grace period) | Já houve incidente real de vazamento de secret em log de aplicação de cliente; grace period permite migração sem downtime | Sofia, Diego | `[09:21]-[09:22]` | Janela de 24h com duas secrets válidas simultaneamente | Permite rotação sem quebrar integração do cliente | ADR-005 |
| DEC-014 | TLS obrigatório (URL do webhook deve ser `https`); tratado como validação de schema, não como decisão arquitetural separada | Requisito básico de segurança de transporte | Sofia, Larissa | `[09:23]` | — | Cadastro com `http` é recusado com erro de validação | Registrado como NFR, não ADR próprio |
| DEC-015 | Limite de payload de 64KB; se excedido, o envio falha com erro (não trunca) | Truncar poderia corromper semântica do evento; 64KB é teto generoso para os eventos previstos | Sofia, Diego, Larissa | `[09:23]-[09:24]` | — | Explicitamente classificado como requisito não funcional, não decisão arquitetural | Registrado como NFR, não ADR próprio |
| DEC-016 | Garantia de entrega *at-least-once* (não *exactly-once*), com dedup do lado do cliente via `X-Event-Id` | Exactly-once exigiria coordenação complexa entre as duas partes; at-least-once + event_id é padrão de mercado (citados Stripe e GitHub) | Diego | `[09:24]-[09:26]` | Transfere a responsabilidade de deduplicação para o cliente | Resolve "99% dos casos" segundo Diego; requer documentação clara no portal do desenvolvedor | ADR-006 |
| DEC-017 | Módulo `src/modules/webhooks` segue o padrão existente (controller, service, repository, routes, schemas) | Consistência com os demais módulos do projeto (orders, auth, users, customers, products) | Bruno, Diego | `[09:27]-[09:28]` | — | Curva de aprendizado baixa para o time; sem discussão de alternativa | ADR-007 |
| DEC-018 | Worker como entry-point separado (`src/worker.ts`); lógica de processamento em arquivo dentro do módulo webhooks | Separa bootstrap de processo da lógica de negócio de processamento | Bruno, Diego | `[09:28]` | Nome do arquivo de processamento não foi fechado (`webhook.worker.ts` ou `webhook.processor.ts`) — ver ambiguidade `U.6` | Estrutura alinhada ao padrão `server.ts` existente | ADR-002, ADR-007 |
| DEC-019 | Erros do módulo seguem o padrão `AppError`, com códigos prefixados `WEBHOOK_` | Já existe padrão consolidado (`InsufficientStockError`, `InvalidStatusTransitionError`, etc.) | Bruno, Larissa | `[09:28]-[09:29]` | — | Nenhuma mudança necessária no middleware de erro central | ADR-007 |
| DEC-020 | Reuso do logger Pino existente e do middleware de erro central, sem alterações | O middleware já trata `AppError`, `ZodError` e erros do Prisma | Bruno | `[09:29]` | — | Nenhuma nova infraestrutura de logging/erro | ADR-007 |
| DEC-021 | Inserção do evento na `webhook_outbox` ocorre dentro da mesma transação SQL de `changeStatus`; falha na inserção causa rollback de toda a transação | Garantir que não exista caso de status mudar sem o evento ser registrado (ou vice-versa) | Bruno, Diego | `[09:40]-[09:41]` | Acopla a lógica de webhook à transação crítica de pedidos | Consistência forte entre mudança de status e evento | ADR-001, ADR-007 |
| DEC-022 | Função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o `tx` da transação atual, em vez de injetar um repository inteiro no `OrderService` | Minimiza acoplamento — função pura em vez de dependência de objeto completo | Bruno, Diego | `[09:41]` | — | `OrderService` chama essa função explicitamente dentro da transação | ADR-001, ADR-007 |
| DEC-023 | Timeout do HTTP call do worker: 10 segundos; timeout é tratado como falha e entra em retry | Evita que um cliente lento prenda o worker indefinidamente | Diego, Sofia | `[09:42]` | — | Cliente que não responde em 10s sempre entra no fluxo de retry | ADR-003 |
| DEC-024 | Formato do payload: JSON com `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`; **não** inclui `items` | Manter payload enxuto; cliente busca detalhes via `GET /orders/:id` se precisar | Diego, Bruno | `[09:43]` | Payload menor vs. cliente precisa de chamada adicional para detalhes completos | Reduz custo de banda e de renderização | Pode compor ADR-006 ou ficar apenas no FDD |
| DEC-025 | Headers do envio: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type: application/json`, `X-Webhook-Id` | `X-Timestamp` permite ao cliente detectar replay attack; `X-Webhook-Id` identifica qual cadastro de endpoint disparou o envio (útil para clientes com múltiplos endpoints) | Diego, Sofia | `[09:44]` | — | Cliente consegue correlacionar evento a endpoint específico e validar frescor da requisição | ADR-004, ADR-006 |
| DEC-026 | Filtro de eventos por status é aplicado na **inserção** do evento na outbox, não no momento do envio | Evita inserir linhas na outbox que nenhum webhook do customer quer receber, economizando espaço | Bruno, Diego | `[09:33]-[09:34]` | — | Se nenhum webhook do customer quiser aquele status, o evento nem é criado | Pode compor ADR-001 |
| DEC-027 | `customer_id` é passado no body/path do endpoint de cadastro, não derivado do JWT | O JWT atual é de usuário operador do sistema, não do cliente final; não existe "JWT de cliente" | Bruno, Larissa, Marcos | `[09:31]-[09:32]` | — | Endpoint autenticado normal, mas identifica o cliente explicitamente no payload | — |
| DEC-028 | Endpoint de replay de DLQ exige role `ADMIN` (via `requireRole` existente) e deve logar quem executou o replay, para auditoria | Reprocessar fila de entrega é ação sensível, não deve ser acessível a qualquer operador | Sofia, Larissa | `[09:35]-[09:36]` | — | Reuso direto do middleware `requireRole` já existente | ADR-007 |
| DEC-029 | Demais endpoints de CRUD de configuração de webhook aceitam qualquer role autenticada (não exigem `ADMIN`), nesta fase | Não há necessidade de restrição adicional identificada agora; Sofia sinaliza que pode "endurecer" depois | Sofia, Marcos | `[09:36]-[09:37]` | Menor fricção operacional agora vs. possível necessidade de reforço futuro | Registrado como limitação aceita temporariamente (ver `LIMIT-004`) | — |
| DEC-030 | Identificadores da outbox usam UUID (não auto-incremento), seguindo o padrão do restante do projeto | Consistência com o padrão já usado em todas as tabelas do schema Prisma atual (`@default(uuid())`) | Larissa | `[09:51]` | — | — | ADR-007 |
| DEC-031 | O evento da outbox guarda o payload já **renderizado (snapshot)** no momento da inserção, não apenas `order_id` para renderizar depois | Se o pedido mudar depois da inserção do evento, o evento ainda deve refletir o estado do momento da mudança de status | Larissa, Diego, Bruno | `[09:51]-[09:52]` | — | Evita inconsistência entre o estado atual do pedido e o payload historicamente entregue | Pode compor ADR-001 |
| DEC-032 | Estimativa de 3 sprints para a feature, incluindo a revisão de segurança da Sofia (reservar ao menos 2 dias úteis) ao final | Modelagem outbox/DLQ (1 sprint) + worker/retry (1 sprint) + CRUD/deliveries (0,5 sprint) + integração no order.service e testes (0,5 sprint) + HMAC/schemas/validações (restante) | Larissa, Sofia | `[09:45]-[09:47]` | — | Prazo de negócio (fim de novembro) depende dessa estimativa | — |

---

## D. Requisitos funcionais

| ID | Descrição | Ator | Entrada/Gatilho | Resultado esperado | Fonte | Localização | Estado |
| --- | --- | --- | --- | --- | --- | --- | --- |
| FR-001 | Cadastro de webhook | Usuário autenticado (representando o cliente) | Requisição `POST` com `url`, lista de status desejados, `customer_id` | Webhook criado; `secret` gerada pelo sistema e devolvida na resposta de criação | TRANSCRICAO | `[09:31]-[09:32] Marcos, Bruno` | DECIDIDO (comportamento); path/contrato exato não especificado — ver `U.2` |
| FR-002 | Consulta/listagem de webhooks de um customer | Usuário autenticado | Requisição `GET` | Lista de webhooks cadastrados para o customer | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO (verbo); path exato não especificado |
| FR-003 | Atualização de webhook | Usuário autenticado | Requisição `PATCH` | Webhook atualizado (ex.: URL, lista de status) | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO (verbo); path exato não especificado |
| FR-004 | Ativação/desativação de webhook | Usuário autenticado | Campo "estado ativo" na configuração do webhook | Webhook fica ativo ou inativo para recebimento de eventos | TRANSCRICAO | `[09:21] Bruno, Sofia` | DECIDIDO (existência do campo); mecanismo de toggle não confirmado explicitamente — presume-se via `PATCH` genérico (inferência, não fato citado) |
| FR-005 | Exclusão de webhook | Usuário autenticado | Requisição `DELETE` | Webhook removido | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO (verbo); path exato não especificado |
| FR-006 | Rotação de secret | Usuário autenticado (cliente) | Solicitação via API de nova secret | Nova secret gerada; secret antiga permanece válida por 24h em paralelo | TRANSCRICAO | `[09:21]-[09:22] Sofia` | DECIDIDO (comportamento); endpoint/contrato exato é questão em aberto (`OPEN-002`) |
| FR-007 | Entrega de eventos via webhook HTTP | Sistema (worker) | Evento pendente na outbox | Chamada HTTP `POST` assinada enviada à URL cadastrada do cliente | TRANSCRICAO | `[09:19]-[09:26], [09:42]-[09:44] Diego, Sofia` | DECIDIDO |
| FR-008 | Filtragem de eventos por status desejado | Sistema | Mudança de status do pedido | Evento só é inserido na outbox se algum webhook do customer quiser aquele status | TRANSCRICAO | `[09:33]-[09:34] Marcos, Bruno, Diego` | DECIDIDO |
| FR-009 | Retentativas automáticas em caso de falha de entrega | Sistema (worker) | Falha/timeout na chamada HTTP | Nova tentativa após intervalo de backoff, até 5 tentativas | TRANSCRICAO | `[09:14]-[09:17] Diego, Bruno, Larissa` | DECIDIDO |
| FR-010 | Envio para Dead Letter Queue (DLQ) após esgotar tentativas | Sistema (worker) | 5ª tentativa falha | Evento movido para tabela `webhook_dead_letter` com payload, motivo da falha e timestamp | TRANSCRICAO | `[09:17]-[09:18] Diego` | DECIDIDO |
| FR-011 | Replay manual de eventos em DLQ | Usuário `ADMIN` | `POST /admin/webhooks/dead-letter/:id/replay` | Evento recolocado como pendente na outbox; ação logada para auditoria | TRANSCRICAO | `[09:18]-[09:19], [09:35]-[09:36] Diego, Sofia, Larissa` | DECIDIDO |
| FR-012 | Autenticação e autorização diferenciada por endpoint | Sistema | Requisição a qualquer endpoint do módulo | CRUD de configuração exige apenas JWT válido (qualquer role); replay de DLQ exige role `ADMIN` | TRANSCRICAO | `[09:31]-[09:32], [09:35]-[09:37] Larissa, Sofia, Marcos` | DECIDIDO |
| FR-013 | Identificação única do evento para deduplicação | Sistema / Cliente | Evento inserido na outbox | UUID gerado como `event_id`, enviado no header `X-Event-Id`; cliente deduplica localmente | TRANSCRICAO | `[09:24]-[09:26] Diego` | DECIDIDO |
| FR-014 | Consulta de histórico de entregas de um webhook | Usuário autenticado | `GET /webhooks/:id/deliveries` | Lista dos últimos 100 envios, com sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | `[09:34]-[09:35] Marcos` | DECIDIDO |

---

## E. Requisitos não funcionais

| ID | Descrição | Fonte | Localização | Estado |
| --- | --- | --- | --- | --- |
| NFR-001 | Latência: cliente considera "tempo real" qualquer entrega abaixo de 10s; latência mínima do sistema no pior caso é de 2s (intervalo de polling) | TRANSCRICAO | `[09:02] Marcos`, `[09:09]-[09:10] Diego, Larissa` | DECIDIDO |
| NFR-002 | Disponibilidade: worker deve rodar em processo separado da API para não cair junto em reinícios | TRANSCRICAO | `[09:11] Diego` | DECIDIDO |
| NFR-003 | Segurança/autenticidade de payload: HMAC-SHA256 sobre o corpo da requisição | TRANSCRICAO | `[09:19]-[09:20] Sofia` | DECIDIDO |
| NFR-004 | Integridade de credenciais: secret única por endpoint (não global) | TRANSCRICAO | `[09:21] Sofia` | DECIDIDO |
| NFR-005 | Transporte seguro: URL do webhook deve ser `https`; `http` é recusado na validação | TRANSCRICAO | `[09:23] Sofia` | DECIDIDO (classificado como NFR, não ADR, por Larissa) |
| NFR-006 | Tamanho máximo de payload: 64KB; acima disso, envio falha com erro (não trunca) | TRANSCRICAO | `[09:23]-[09:24] Sofia, Diego, Larissa` | DECIDIDO (classificado como NFR, não ADR, por Larissa) |
| NFR-007 | Timeout de chamada HTTP do worker: 10 segundos | TRANSCRICAO | `[09:42] Diego, Sofia` | DECIDIDO |
| NFR-008 | Retries: backoff exponencial, 5 tentativas, progressão 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:14]-[09:17] Diego, Larissa` | DECIDIDO |
| NFR-009 | Garantia de entrega: at-least-once (não exactly-once) | TRANSCRICAO | `[09:24]-[09:26] Diego` | DECIDIDO |
| NFR-010 | Ordenação: garantida apenas por `order_id`, apenas em regime single-worker; sem garantia de ordenação global | TRANSCRICAO | `[09:12]-[09:13] Diego, Larissa, Marcos` | DECIDIDO / LIMITAÇÃO (ver `LIMIT-001`) |
| NFR-011 | Observabilidade: log de auditoria para ações de replay administrativo; histórico de entregas visível ao cliente. Não há discussão explícita de métricas/tracing além disso | TRANSCRICAO | `[09:36] Sofia`, `[09:34]-[09:35] Marcos` | PARCIALMENTE DECIDIDO — cobertura de observabilidade além de logging/auditoria não foi discutida (lacuna, não decisão; ver seção U) |
| NFR-012 | Escalabilidade: sistema opera em regime single-worker nesta fase; escalar para múltiplos workers é "problema do futuro" | TRANSCRICAO | `[09:13] Diego` | ADIADO / LIMITAÇÃO (ver `LIMIT-005`) |
| NFR-013 | Resiliência/consistência: inserção do evento na outbox é atômica com a transação de mudança de status (rollback conjunto) | TRANSCRICAO | `[09:06]-[09:08], [09:40]-[09:41] Diego, Bruno` | DECIDIDO |
| NFR-014 | Compatibilidade de infraestrutura: worker reutiliza mesmo banco/`DATABASE_URL` e mesma stack (Prisma), com instância de `PrismaClient` própria por processo | TRANSCRICAO | `[09:29]-[09:30] Diego, Bruno` | DECIDIDO |
| NFR-015 | Retenção: linhas entregues na outbox devem ser arquivadas após ~30 dias — explicitamente fora do escopo desta feature | TRANSCRICAO | `[09:08] Diego` | FORA DE ESCOPO (documentado como decisão futura, não implementação atual) |

---

## F. Alternativas consideradas e descartadas

| ID | Alternativa | Contexto | Participante | Timestamp | Motivo do descarte | Trade-off | Solução escolhida |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ALT-001 | Disparo síncrono de webhook dentro da transação de `changeStatus` | Discussão inicial de arquitetura de disparo | Bruno, Larissa | `[09:03]-[09:05]` | Um HTTP call lento/travado no meio da transação bloquearia mudanças de status de outros pedidos; não há como fazer rollback de forma limpa se o cliente estiver fora do ar | Simplicidade imediata vs. risco de travamento de todo o fluxo transacional | Outbox assíncrono + worker separado (DEC-002) |
| ALT-002 | Fila dedicada (ex.: Redis Streams) | Discussão de como desacoplar o disparo | Larissa, Diego | `[09:07]` | Overengineering para um time pequeno; exigiria subir infraestrutura nova (ex.: Redis Cluster) | Fila dedicada oferece recursos prontos vs. custo operacional de nova infra | Outbox na tabela MySQL já existente (DEC-002) |
| ALT-003 | Trigger de banco de dados para notificar o worker de forma reativa | Discussão de como o worker deveria ler os eventos | Bruno, Diego | `[09:09]` | MySQL não possui listener nativo (tipo `NOTIFY/LISTEN` do Postgres); trigger só executa SQL, não notifica processo externo; alternativas de contorno (escrever em arquivo, bater endpoint) foram consideradas "esquisitas" | Reatividade imediata vs. complexidade de implementação e manutenção | Polling a cada 2 segundos (DEC-003) |
| ALT-004 | Retry indefinido com backoff (sem teto de tentativas) | Discussão da política de retry | Diego | `[09:15]` | Deixaria eventos "pendurados para sempre" se o cliente tivesse sumido definitivamente | Cobertura total de recuperação vs. acúmulo indefinido de eventos pendentes | Teto de 5 tentativas seguido de DLQ (DEC-007) |
| ALT-005 | 3 tentativas de retry (mais agressivo) | Discussão do número de tentativas | Bruno, Diego | `[09:16]` | Indisponibilidade real já observada em cliente (2h de manutenção planejada) mataria o evento cedo demais com apenas 3 tentativas em ~30min | Resposta mais rápida à falha permanente vs. risco de descartar eventos recuperáveis | 5 tentativas (DEC-007) |
| ALT-006 | Marcar falha permanente como status "failed" na própria tabela de outbox | Discussão de onde registrar falhas permanentes | Diego | `[09:17]-[09:18]` | Tabela separada mantém a leitura da outbox principal mais limpa e serve como evidência para debug/reprocessamento | Menos uma tabela para manter vs. outbox principal "suja" com registros de falha | Tabela `webhook_dead_letter` separada (DEC-009) |
| ALT-007 | Truncar payload que excede o limite de tamanho | Discussão do limite de payload | Sofia | `[09:23]-[09:24]` | Truncar poderia corromper a semântica do evento; se chegou nesse tamanho, "tem algo errado" | Entrega parcial vs. falha explícita e diagnosticável | Erro explícito acima de 64KB (DEC-015) |
| ALT-008 | Garantia de entrega exactly-once | Discussão de garantias de entrega | Diego | `[09:24]-[09:26]` | Exigiria coordenação complexa entre as duas partes (sistema e cliente); complexidade não justificada frente ao padrão de mercado | Garantia mais forte vs. complexidade de implementação/operação | At-least-once com `X-Event-Id` para dedup do cliente (DEC-016) |

---

## G. Questões em aberto

| ID | Questão | Contexto | Participantes | Timestamp | Impacto potencial | Documento futuro |
| --- | --- | --- | --- | --- | --- | --- |
| OPEN-001 | Rate limiting de envio ao cliente (evitar disparar muitas chamadas em rajada, ex.: 50 pedidos mudando de status em 1 minuto) | Diego levanta o ponto ao final do bloco de regras de negócio; Larissa pergunta se faz parte do escopo | Diego, Larissa | `[09:38]-[09:39]` | Cliente pode ser sobrecarregado com um volume alto de chamadas simultâneas se tiver muitos pedidos mudando de status | RFC (questões em aberto) / FDD (nota de risco futuro) |
| OPEN-002 | Contrato exato (método HTTP e path) do endpoint de rotação de secret | Sofia descreve o comportamento (secret nova, grace period de 24h) mas nenhum path/verbo foi citado literalmente | Sofia | `[09:21]-[09:22]` | Bloqueia especificação completa do contrato de API no FDD sem definição adicional | FDD (contratos públicos) |
| OPEN-003 | Se e quando o CRUD de configuração de webhook terá restrição de role adicional (hoje aceita qualquer role autenticada) | Sofia comenta que "mais pra frente a gente pode endurecer", sem compromisso de quando | Sofia, Marcos | `[09:36]-[09:37]` | Superfície de acesso mais ampla do que idealmente desejável a longo prazo | RFC (questões em aberto) |

---

## H. Itens adiados ou fora de escopo

| ID | Item | Classificação | Contexto | Timestamp |
| --- | --- | --- | --- | --- |
| OUT-001 | Notificação por e-mail ao cliente quando o webhook falha repetidamente (ex.: 3 falhas seguidas) | ADIADO PARA FASE FUTURA | Marcos pergunta se é possível avisar por e-mail; Larissa responde que está fora de escopo desta fase, possivelmente na próxima, após medir impacto | `[09:37]-[09:38]` |
| OUT-002 | Dashboard visual para o cliente ver seus webhooks | NÃO NECESSÁRIO NESTA VERSÃO (explicitamente descartado desta fase) | Marcos pergunta sobre painel visual; Larissa responde que não, que é projeto separado do time de frontend | `[09:39]-[09:40]` |
| OUT-003 | Arquivamento de linhas entregues da outbox após ~30 dias | EXPLICITAMENTE FORA DE ESCOPO desta feature | Diego menciona a necessidade futura de arquivamento, mas classifica como fora do escopo atual | `[09:08]` |
| OUT-004 | Rate limiting de envio outbound | ADIADO CONDICIONALMENTE (observar e decidir depois) — ver também `OPEN-001` | Diego propõe não implementar agora e observar se vira problema real | `[09:39]` |
| OUT-005 | Endurecimento de RBAC no CRUD de configuração de webhook (hoje qualquer role autenticada) | LIMITAÇÃO ACEITA TEMPORARIAMENTE / possível item futuro | Sofia aceita o estado atual "por enquanto", sinalizando possível revisão futura | `[09:36]-[09:37]` |

---

## I. Limitações conhecidas

| ID | Limitação | Fonte | Localização |
| --- | --- | --- | --- |
| LIMIT-001 | Ordenação de eventos garantida apenas por `order_id`, e apenas enquanto o sistema operar com um único worker; não há garantia de ordenação global | TRANSCRICAO | `[09:12]-[09:13] Diego, Larissa` |
| LIMIT-002 | Latência mínima de entrega de 2 segundos no pior caso, decorrente do intervalo de polling do worker | TRANSCRICAO | `[09:09]-[09:10] Diego, Larissa` |
| LIMIT-003 | Garantia de entrega é at-least-once, não exactly-once; cliente pode receber o mesmo evento mais de uma vez e deve implementar deduplicação por `X-Event-Id` | TRANSCRICAO | `[09:24]-[09:26] Diego, Sofia` |
| LIMIT-004 | CRUD de configuração de webhook aceita qualquer role autenticada, sem restrição adicional nesta fase | TRANSCRICAO | `[09:36]-[09:37] Sofia, Marcos` |
| LIMIT-005 | Escalar para múltiplos workers em paralelo perde a garantia de ordenação atual; exigiria particionamento por `order_id` ou lock pessimista, não implementado nesta fase | TRANSCRICAO | `[09:13] Diego` |

---

## J. Métricas e metas quantitativas

| ID | Métrica | Valor | Fonte | Localização |
| --- | --- | --- | --- | --- |
| METRIC-001 | Limiar de "tempo real" definido pelo cliente | < 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| METRIC-002 | Intervalo de polling do worker | 2 segundos | TRANSCRICAO | `[09:09]-[09:10] Diego, Larissa, Marcos` |
| METRIC-003 | Número de tentativas de retry | 5 | TRANSCRICAO | `[09:15]-[09:17] Diego, Larissa` |
| METRIC-004 | Progressão de backoff exponencial | 1min, 5min, 30min, 2h, 12h (total ~15h) | TRANSCRICAO | `[09:17] Diego` |
| METRIC-005 | Grace period de rotação de secret | 24 horas | TRANSCRICAO | `[09:21]-[09:22] Sofia` |
| METRIC-006 | Limite máximo de tamanho de payload | 64 KB | TRANSCRICAO | `[09:23]-[09:24] Diego, Larissa` |
| METRIC-007 | Timeout do HTTP call do worker | 10 segundos | TRANSCRICAO | `[09:42] Diego` |
| METRIC-008 | Retenção/arquivamento de eventos entregues (fora do escopo desta feature) | ~30 dias | TRANSCRICAO | `[09:08] Diego` |
| METRIC-009 | Histórico de entregas exibido ao cliente | últimos 100 webhooks | TRANSCRICAO | `[09:34] Marcos` |
| METRIC-010 | Prazo de negócio (pedido do cliente Atlas) | fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| METRIC-011 | Estimativa de esforço de entrega | 3 sprints (incluindo revisão de segurança) | TRANSCRICAO | `[09:45]-[09:47] Larissa` |
| METRIC-012 | Tempo reservado para revisão de segurança antes do deploy | pelo menos 2 dias úteis | TRANSCRICAO | `[09:46] Sofia` |

---

## K. Riscos e mitigações

| ID | Risco | Probabilidade/Impacto | Mitigação discutida | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RISK-001 | Cliente de webhook lento ou fora do ar trava mudanças de status de outros pedidos, caso o disparo fosse síncrono | Impacto: alto (afeta fluxo core de pedidos). Probabilidade não quantificada na reunião | Processamento assíncrono via outbox + worker separado | TRANSCRICAO | `[09:04] Bruno` |
| RISK-002 | Inconsistência entre mudança de status e evento de webhook (evento perdido ou registrado indevidamente) | Impacto: alto (quebra garantia de entrega). Probabilidade não quantificada | Inserção do evento na outbox dentro da mesma transação SQL do `changeStatus`; rollback conjunto | TRANSCRICAO | `[09:06] Diego`, `[09:40]-[09:41] Bruno, Diego` |
| RISK-003 | Vazamento de secret compartilhada expondo múltiplos clientes | Já ocorreu antes (incidente real citado: vazamento em log de aplicação de cliente). Impacto: alto | Secret única por endpoint (não global) com suporte a rotação | TRANSCRICAO | `[09:21]-[09:22] Sofia, Diego` |
| RISK-004 | Indisponibilidade prolongada do endpoint do cliente causando perda de eventos se número de retries for insuficiente | Evidência de caso real (2h de indisponibilidade planejada de um cliente). Impacto: médio-alto | 5 tentativas com backoff estendido até ~15h | TRANSCRICAO | `[09:16]-[09:17] Diego` |
| RISK-005 | Acúmulo de eventos na tabela de outbox degradando a performance de leitura do worker | Impacto: médio (degradação de latência). Probabilidade não quantificada | Índices em `status` e `created_at`, leitura em lote pequeno dos pendentes, arquivamento futuro de entregues (fora do escopo atual) | TRANSCRICAO | `[09:07]-[09:08] Bruno, Diego` |
| RISK-006 | Perda do worker (e consequente atraso de todas as entregas) se ele rodar no mesmo processo da API e a API reiniciar | Impacto: alto (paralisação total de entregas) | Worker como processo separado (`src/worker.ts`) | TRANSCRICAO | `[09:11] Diego` |
| RISK-007 | Cliente sendo "bombardeado" com muitas chamadas em rajada (rate limiting não implementado) | Impacto potencial não quantificado; sem mitigação decidida — risco aceito para observação | Nenhuma mitigação decidida nesta fase; registrado como ponto em aberto (`OPEN-001`) | TRANSCRICAO | `[09:38]-[09:39] Diego, Larissa` |
| RISK-008 | Replay de eventos DLQ sem controle, permitindo ações administrativas não rastreáveis | Impacto: médio (auditoria/compliance) | Exigir role `ADMIN` e logging de quem executou o replay | TRANSCRICAO | `[09:35]-[09:36] Sofia` |
| RISK-009 | Payload de webhook excessivamente grande causando problemas de entrega/processamento | Impacto: baixo-médio | Limite de 64KB com erro explícito acima disso | TRANSCRICAO | `[09:23]-[09:24] Sofia, Diego` |

> Nota: nenhuma classificação de probabilidade numérica foi discutida na reunião para nenhum risco; onde indicado "probabilidade não quantificada", trata-se de ausência de dado na fonte, não de avaliação técnica desta análise.

---

## L. Endpoints e contratos mencionados

| Método | Path | Ator | Autenticação | Finalidade | Campos de entrada mencionados | Resposta/Status mencionados | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| POST | *(path não especificado literalmente)* | Usuário autenticado | JWT (autenticado, qualquer role) | Cadastro de webhook | `url`, lista de status desejados, `customer_id` (no body/path); `secret` gerada pelo sistema, não enviada pelo cliente | `secret` devolvida na criação | TRANSCRICAO | `[09:31]-[09:32] Marcos, Bruno` |
| GET | *(path não especificado literalmente)* | Usuário autenticado | JWT (autenticado, qualquer role) | Listagem de webhooks de um customer | — | Lista de webhooks | TRANSCRICAO | `[09:33] Bruno` |
| PATCH | *(path não especificado literalmente)* | Usuário autenticado | JWT (autenticado, qualquer role) | Edição de webhook | — | — | TRANSCRICAO | `[09:33] Bruno` |
| DELETE | *(path não especificado literalmente)* | Usuário autenticado | JWT (autenticado, qualquer role) | Remoção de webhook | — | — | TRANSCRICAO | `[09:33] Bruno` |
| GET | `/webhooks/:id/deliveries` | Usuário autenticado (cliente) | JWT (autenticado) | Histórico de entregas (últimos 100 webhooks enviados) | — | sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | `[09:34]-[09:35] Marcos` |
| POST | `/admin/webhooks/dead-letter/:id/replay` | Usuário `ADMIN` | JWT + role `ADMIN` (via `requireRole`) | Replay manual de evento em DLQ | — | Evento recolocado como pendente na outbox | TRANSCRICAO | `[09:18]-[09:19], [09:35]-[09:36] Diego, Sofia, Larissa` |
| *(não especificado)* | *(não especificado — rotação de secret)* | Usuário autenticado (cliente) | JWT (autenticado) | Rotação de secret | — | Nova secret; antiga válida por 24h | TRANSCRICAO | `[09:21]-[09:22] Sofia` — ver `OPEN-002` |

**Endpoints existentes referenciados (já implementados hoje, contexto de uso):**

| Método | Path | Uso mencionado | Fonte | Localização |
| --- | --- | --- | --- | --- |
| GET | `/orders` | Polling atual dos clientes B2B (o problema que a feature resolve) | TRANSCRICAO | `[09:00] Marcos` |
| GET | `/orders/:id` | Cliente busca detalhes completos do pedido após receber evento enxuto | TRANSCRICAO | `[09:43] Diego` |

Não foram mencionados na reunião: status codes específicos, formatos de erro, ou payloads completos de request/response para nenhum dos endpoints do módulo de webhooks — não devem ser inventados nesta etapa.

---

## M. Eventos e payloads

| Atributo | Valor mencionado | Fonte | Localização |
| --- | --- | --- | --- |
| Nome/tipo do evento | `order.status_changed` | TRANSCRICAO | `[09:43] Diego` |
| Gatilho | Mudança de status do pedido em `OrderService.changeStatus` | TRANSCRICAO | `[09:40]-[09:41] Bruno` |
| Campos do payload | `event_id`, `event_type`, `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | TRANSCRICAO | `[09:43] Diego` |
| Campo explicitamente excluído | `items` (para não inflar o payload) | TRANSCRICAO | `[09:43] Diego` |
| Headers | `X-Event-Id` (UUID), `X-Signature` (HMAC), `X-Timestamp`, `Content-Type: application/json`, `X-Webhook-Id` | TRANSCRICAO | `[09:44] Diego, Sofia` |
| Assinatura | HMAC-SHA256 sobre o corpo da requisição | TRANSCRICAO | `[09:19]-[09:20] Sofia` |
| Event ID | UUID gerado no momento em que o evento entra na outbox; único por evento | TRANSCRICAO | `[09:25] Diego` |
| Semântica de duplicidade | At-least-once; cliente deve deduplicar pelo `event_id` | TRANSCRICAO | `[09:24]-[09:26] Diego, Sofia` |
| Tamanho máximo | 64 KB; erro se ultrapassar | TRANSCRICAO | `[09:23]-[09:24]` |
| Ordering | Implícito por `order_id`, apenas em regime single-worker; sem garantia global | TRANSCRICAO | `[09:12]-[09:13]` |
| Formato/versionamento | JSON; nenhuma estratégia de versionamento de schema de evento foi discutida | TRANSCRICAO | *(não discutido — ausência de fonte)* |

---

## N. Fluxos principais (com base exclusiva na reunião)

1. **Mudança de status do pedido**: ocorre dentro de `OrderService.changeStatus`, dentro da transação SQL já existente (atualiza `orders`, insere em `order_status_history`, ajusta `stock_quantity`). `[09:04], [09:40] Bruno`
2. **Criação do evento na outbox**: na mesma transação, insere-se uma linha na `webhook_outbox` com o payload já renderizado (snapshot no momento da inserção), somente se algum webhook do customer estiver inscrito naquele status. `[09:06] Diego`, `[09:33]-[09:34] Bruno, Diego`, `[09:51]-[09:52] Larissa, Diego, Bruno`
3. **Leitura pelo worker**: processo separado, loop de polling a cada 2 segundos, busca os eventos pendentes mais antigos em lote pequeno (batch). `[09:09] Diego`
4. **Envio HTTP**: worker faz a chamada HTTP `POST` para a URL cadastrada, com timeout de 10s e os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type`. `[09:42]-[09:44]`
5. **Sucesso**: evento é marcado como entregue. `[09:08] Diego`
6. **Falha**: timeout (10s) ou erro de resposta do cliente é tratado como falha, entra no fluxo de retry. `[09:42] Sofia, Diego`
7. **Retry**: backoff exponencial 1m/5m/30m/2h/12h, até 5 tentativas. `[09:15]-[09:17]`
8. **Envio para DLQ**: após a 5ª tentativa falhar, o evento é movido para a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp. `[09:17]-[09:18] Diego`
9. **Replay manual**: via `POST /admin/webhooks/dead-letter/:id/replay`, recoloca o evento como pendente na outbox; exige role `ADMIN`; ação deve ser logada para auditoria. `[09:18]-[09:19], [09:35]-[09:36]`
10. **Rotação de secret**: cliente solicita nova secret pela API (endpoint não especificado — `OPEN-002`); a secret antiga permanece válida em paralelo por 24h, depois é invalidada. `[09:21]-[09:22] Sofia`

Nenhum detalhe adicional (ex.: formato exato de log, nome exato de tabelas, schema de banco completo) foi decidido na reunião além do que está listado acima.

---

## Análise do código

## O. Arquitetura atual da aplicação

- **Stack**: Node.js (>=20) + TypeScript, ESM (`"type": "module"`), Express 4.21, Prisma 5.22 sobre MySQL, autenticação via `jsonwebtoken`, validação via `zod`, logging via `pino`/`pino-pretty`, hashing de senha via `bcrypt`, testes via `vitest` + `supertest`. Evidência: `package.json`.
- **Padrão modular**: cada domínio vive em `src/modules/<dominio>/` com arquivos `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts`. Confirmado nos módulos `orders`, `auth`, `users`, `customers`, `products` (via listagem de `src/modules/**`).
- **Composição de dependências**: manual, sem framework de DI. `src/app.ts` (`buildControllers(prisma)`) instancia `Repository → Service → Controller` para cada módulo e monta o `Express` app (`buildApp`).
- **Padrão de controllers**: classes finas com métodos `RequestHandler` que chamam o service e delegam erros para `next(err)` — ver `src/modules/orders/order.controller.ts`.
- **Services**: concentram regra de negócio; usam `prisma.$transaction` quando precisam de atomicidade (ex.: `OrderService.create`, `OrderService.changeStatus`).
- **Repositories**: encapsulam consultas Prisma cruas (`OrderRepository`), incluindo construção de filtros (`buildWhere`) e includes de relações.
- **Schemas**: Zod, com tipos TS derivados via `z.infer` (`order.schemas.ts`).
- **Middlewares**: `authenticate` (JWT), `requireRole(...roles)` (RBAC), `validate({body,query,params})` (Zod), `requestLogger` (correlação de `X-Request-Id` + log de request via Pino), `errorMiddleware` (tratamento centralizado, último middleware da cadeia).
- **Erros**: `AppError` (base) → subclasses em `http-errors.ts` (`BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`) e subclasses de domínio (`InvalidStatusTransitionError`, `InsufficientStockError`), todas com `statusCode` + `errorCode` + `details` opcional.
- **Logger**: singleton Pino (`src/shared/logger/index.ts`), com redact de campos sensíveis (`authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken`), campos base fixos (`service`, `env`), timestamp ISO, `pino-pretty` apenas em dev.
- **Banco**: Prisma Client singleton (`src/config/database.ts`), MySQL, log level dependente de `NODE_ENV`.
- **Testes**: Vitest + Supertest, batendo em uma instância real da aplicação (`getTestApp()`), com asserções diretas no Prisma real (sem mocks) — `tests/orders.test.ts`, `tests/auth.test.ts`.
- **Entry point da API**: `src/server.ts` — monta o app, escuta `env.PORT`, trata `SIGINT`/`SIGTERM` com shutdown gracioso (fecha servidor HTTP, desconecta Prisma).

## P. Fluxo atual de mudança de status (`OrderService.changeStatus`)

Referência: `src/modules/orders/order.service.ts:126-179`.

1. **Início da transação**: `this.prisma.$transaction(async (tx) => { ... })` — tudo executa dentro de uma transação interativa Prisma.
2. **Consulta do pedido**: `tx.order.findUnique({ where: { id }, include: { items: true } })`; se não encontrado, `throw new NotFoundError('Order')`.
3. **Validação da transição**: compara `from = order.status` com `to = input.toStatus`. Se iguais, lança `ConflictError` (`INVALID_STATUS_TRANSITION`). Em seguida, `canTransition(from, to)` (de `order.status.ts`) — se `false`, lança `InvalidStatusTransitionError`.
4. **Manipulação de estoque**: `shouldDebitStock(from, to)` é verdadeiro apenas para `PENDING → PAID`, disparando `debitStock(tx, order.items)` (valida estoque suficiente, senão lança `InsufficientStockError`, depois decrementa `stockQuantity`). `shouldReplenishStock(from, to)` é verdadeiro para transições de `PAID`/`PROCESSING` → `CANCELLED`, disparando `replenishStock` (incrementa `stockQuantity` de volta).
5. **Atualização do pedido**: `tx.order.update({ where: { id }, data: { status: to } })`.
6. **Criação do histórico**: `tx.orderStatusHistory.create({ data: { orderId: id, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null } })`.
7. **Retorno**: novo `tx.order.findUnique` com `items`, `history` (ordenado por `changedAt asc`) e `customer`, retornado ao chamador.
8. **Rollback em caso de erro**: qualquer erro lançado dentro do callback (passo 2 a 7) rejeita a promise do callback; o Prisma reverte automaticamente todas as escritas da transação interativa — não há `try/catch` explícito nem rollback manual no código.

Este é o ponto exato onde a inserção do evento na `webhook_outbox` deveria ocorrer (após o passo 6, dentro da mesma transação), conforme `DEC-021`/`DEC-022`.

## Q. Pontos de integração com a feature

| ID | Caminho real | Símbolo relevante | Comportamento atual | Integração futura proposta | Risco de alteração | Evidência |
| --- | --- | --- | --- | --- | --- | --- |
| CODE-001 | `src/modules/orders/order.service.ts` | `OrderService.changeStatus` | Transação Prisma atômica que atualiza status, histórico e estoque | Chamar `publishWebhookEvent(tx, order, from, to)` dentro da mesma transação, após criação do histórico | Médio — é o método de negócio mais crítico (mexe em estoque); falha na inserção do evento deve reverter toda a transação | linhas 126-179, uso de `this.prisma.$transaction(async (tx) => {...})` |
| CODE-002 | `prisma/schema.prisma` | Modelos `Order`, `OrderStatusHistory`, `OrderStatus` (enum) | Fornecem os campos usados no payload do evento (`order_id`, `order_number`, `from_status`/`to_status`, `customer_id`, `totalCents`) | Adicionar novos modelos para configuração de webhook, outbox e DLQ (nomes ainda não confirmados formalmente — ver seção R) | Baixo para leitura; requer nova migration Prisma | linhas 74-97, 116-131 |
| CODE-003 | `src/app.ts` | `buildControllers(prisma)`, `buildApp(deps)` | Composição manual de dependências (repository → service → controller) sem framework de DI | Registrar `WebhookRepository`/`WebhookService`/`WebhookController` (e repositórios de deliveries/DLQ, se necessários) seguindo o mesmo padrão | Baixo — é aditivo | linhas 26-53 |
| CODE-004 | `src/routes/index.ts` | `buildApiRouter(controllers)` | Monta cada módulo sob `/api/v1/<módulo>` | Adicionar `router.use('/webhooks', buildWebhookRouter(...))`; endpoint admin de replay pode exigir sub-rota própria (`/admin/...`, conforme mencionado na reunião) | Baixo | linhas 21-31 |
| CODE-005 | `src/middlewares/auth.middleware.ts` | `authenticate`, `requireRole(...roles)` | JWT via `jsonwebtoken`; popula `req.user` com `{id, email, role}`; `requireRole` valida RBAC | Reuso direto: CRUD usa apenas `authenticate`; replay de DLQ usa `authenticate` + `requireRole('ADMIN')` (`DEC-028`) | Baixo — reuso puro, sem alteração ao middleware | linhas 27-61 |
| CODE-006 | `src/middlewares/validate.middleware.ts` | `validate({ body, query, params })` | Parseia/valida `req.body`/`query`/`params` com Zod, lançando `ValidationError` em caso de `ZodError` | Novos schemas Zod (ex.: `webhook.schemas.ts`, proposto) reaproveitando este middleware, incluindo validação de URL `https`-only (`DEC-014`) | Baixo | linhas 11-37 |
| CODE-007 | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts` | `AppError`, `ConflictError`, `NotFoundError`, `UnprocessableEntityError`, etc. | Padrão consolidado de erro com `statusCode` + `errorCode` + `details` opcional | Novas subclasses específicas do módulo webhooks com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, mencionados por Bruno) | Baixo — extensão aditiva do padrão existente | app-error.ts linhas 1-16; http-errors.ts linhas 1-63 |
| CODE-008 | `src/middlewares/error.middleware.ts` | `errorMiddleware` | Trata `AppError`, `ZodError`, `Prisma.PrismaClientKnownRequestError` (`P2002`, `P2025`) e fallback 500 | Nenhuma alteração necessária — decisão explícita de reuso sem modificação (`DEC-020`) | Nulo/baixo — apenas validar que as novas subclasses de `AppError` se encaixam no contrato | linhas 14-65 |
| CODE-009 | `src/shared/logger/index.ts` | `logger` (Pino singleton), `redactPaths` | Logger estruturado com redact de campos sensíveis (`password`, `token`, etc.) | Reuso direto pelo worker (`src/worker.ts`, proposto) e pelo módulo webhooks (ex.: log de auditoria de replay, `DEC-028`) | Baixo — reuso; atenção: lista de redact atual não inclui `secret` explicitamente (observação de código, não decisão da reunião — ver `U.8`) | linhas 4-32 |
| CODE-010 | `src/config/database.ts` | `createPrismaClient()`, `prisma` (singleton) | Cria/expõe uma única instância de `PrismaClient` configurada por `env.NODE_ENV` | Worker precisa de instância própria de `PrismaClient` (processo separado), reaproveitando o mesmo padrão de criação e a mesma `DATABASE_URL` (`DEC-005`) | Baixo — reuso do padrão de criação, não do singleton em si | linhas 1-11 |
| CODE-011 | `src/server.ts` | Bootstrap da API (`buildApp`, `app.listen`, shutdown gracioso) | Entry point atual da API, com shutdown em `SIGINT`/`SIGTERM` e `prisma.$disconnect()` | Referência de padrão para o futuro `src/worker.ts` (ainda não existe): próprio bootstrap, próprio loop de polling, próprio shutdown gracioso, próprio `PrismaClient`, processo distinto (`npm run worker`, `DEC-004`) | Nulo para `server.ts` em si — não precisa ser modificado; o novo arquivo é aditivo | linhas 1-27 |
| CODE-012 | `tests/orders.test.ts`, `tests/auth.test.ts` | Padrão de teste com Vitest + Supertest, `getTestApp()`, `bootstrapAuthenticatedUser`, asserções diretas no Prisma real | Testes de integração ponta a ponta, sem mocks de banco | Novos testes do módulo webhooks (envio, retry, DLQ, HMAC) devem seguir o mesmo padrão de helpers/factories; testes de `changeStatus` estendido devem validar inserção do evento na outbox dentro da mesma transação (incluindo cenário de rollback) | Baixo — aditivo | orders.test.ts linhas 1-9, 59-87 |

## R. Elementos futuros propostos (ainda não existem no código)

Confirmado por busca no repositório (`Glob` por `webhook*` e `worker*` não retornou nenhum arquivo): **nenhum artefato de webhook existe hoje**. Todos os itens abaixo são propostas discutidas na reunião, não implementações atuais:

- Módulo `src/modules/webhooks/` (controller, service, repository, routes, schemas). `[09:27]-[09:28] Bruno`
- Entry point `src/worker.ts` e script `npm run worker`. `[09:11] Larissa`, `[09:28] Bruno`
- Arquivo de processamento do worker dentro do módulo — nome não fechado entre `webhook.worker.ts` ou `webhook.processor.ts`. `[09:28] Bruno` (ver ambiguidade `U.6`)
- Modelo(s) Prisma para configuração de webhook (campos mencionados: `url`, `secret`, `customer_id`, "estado ativo"; nome formal da tabela não confirmado). `[09:21] Bruno, Sofia`
- Tabela de outbox, referida coloquialmente como "tipo `webhook_outbox`" (nome ilustrativo, não confirmado formalmente). `[09:06] Diego`
- Tabela de dead letter, referida como `webhook_dead_letter` (nome citado, mas em tom de proposta: "eu fazia uma tabela..."). `[09:18] Diego`
- Novos endpoints: cadastro (`POST`), edição (`PATCH`), remoção (`DELETE`), listagem (`GET`), histórico de entregas (`GET /webhooks/:id/deliveries`), replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), rotação de secret (endpoint não especificado).
- Novos códigos de erro `WEBHOOK_*` (exemplos citados: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`). `[09:28] Bruno`
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)`. `[09:41] Bruno, Diego`
- Novos testes automatizados para o módulo webhooks (não existem hoje).

Nenhum desses elementos deve ser descrito como "existente" em qualquer documento futuro.

## S. Reuso de padrões existentes

| Padrão existente | Como deve ser reutilizado |
| --- | --- |
| Estrutura de módulo (controller/service/repository/routes/schemas) | Módulo `webhooks` segue exatamente a mesma decomposição dos módulos `orders`/`auth`/`users`/`customers`/`products` |
| `AppError` e subclasses | Novas classes de erro do domínio webhook herdam de `AppError` (ou de subclasses como `ConflictError`/`NotFoundError`), com códigos prefixados `WEBHOOK_` |
| Middleware de erro central (`errorMiddleware`) | Nenhuma alteração necessária; já trata qualquer `AppError` |
| `authenticate` / `requireRole` | Reuso direto para autenticação JWT e RBAC (CRUD vs. replay `ADMIN`) |
| Logger Pino | Reuso do singleton existente, tanto pela API quanto pelo worker |
| Prisma / MySQL | Mesmo banco e `DATABASE_URL`; worker usa instância própria de `PrismaClient` |
| Zod + `validate` middleware | Novos schemas seguem o mesmo padrão de `order.schemas.ts` |
| Vitest + Supertest | Novos testes seguem o padrão de `tests/orders.test.ts`/`tests/auth.test.ts`, com helpers/factories e asserções diretas no Prisma |

## T. Candidatos a ADR (5 a 8)

| Nº sugerido | Título (kebab-case) | Decisão | Fontes | Arquivos existentes relacionados | Alternativas reais | Consequências positivas | Consequências negativas |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ADR-001 | `outbox-pattern-mysql` | Usar padrão Outbox em tabela MySQL, inserida na mesma transação de `changeStatus`, em vez de processamento síncrono ou fila externa | `[09:03]-[09:08]`, `[09:40]-[09:41]`, `[09:51]-[09:52]` | `src/modules/orders/order.service.ts` (`changeStatus`), `prisma/schema.prisma` | Síncrono (`ALT-001`), Redis Streams (`ALT-002`) | Atomicidade garantida com a transação de negócio; nenhuma infraestrutura nova | Acoplamento a polling; requer arquivamento futuro das linhas entregues (fora do escopo atual) |
| ADR-002 | `worker-processo-separado-com-polling` | Worker roda como processo Node separado (`src/worker.ts`), fazendo polling da outbox a cada 2 segundos | `[09:09]-[09:11]` | `src/server.ts` (padrão de entry point), `package.json` (scripts) | Trigger de banco reativo (`ALT-003`); worker embutido na API (implicitamente descartado, `[09:11]`) | Resiliência a reinício da API; simplicidade de implementação | Latência mínima de 2s; overhead de polling constante no banco |
| ADR-003 | `retry-com-backoff-exponencial-e-dlq` | 5 tentativas com backoff 1m/5m/30m/2h/12h; após esgotar, evento vai para tabela DLQ separada, com replay manual via endpoint admin | `[09:14]-[09:19]`, `[09:42]` | — (funcionalidade nova) | Retry indefinido (`ALT-004`), 3 tentativas (`ALT-005`), status "failed" na própria outbox (`ALT-006`) | Cobre indisponibilidades reais de até ~15h; DLQ auditável e reprocessável | Evento pode levar até ~15h para ser considerado falha permanente; complexidade operacional adicional |
| ADR-004 | `autenticacao-hmac-sha256-secret-por-endpoint` | Assinar payload com HMAC-SHA256 (header `X-Signature`), com secret exclusiva por endpoint de webhook (não global) | `[09:19]-[09:22]` | — (funcionalidade nova) | Secret global (implicitamente descartada, `[09:21]`) | Autenticidade/integridade verificável; blast radius reduzido em caso de vazamento | Exige gestão de rotação e armazenamento seguro de múltiplas secrets |
| ADR-005 | `rotacao-de-secret-com-grace-period-24h` | Suportar rotação de secret via API, com a secret antiga válida em paralelo por 24h | `[09:21]-[09:22]` | — (funcionalidade nova) | Invalidação imediata da secret antiga (implicitamente descartada, pois motivada pelo incidente de vazamento citado) | Permite migração do cliente sem downtime | Janela de 24h com duas secrets válidas simultaneamente; endpoint exato ainda em aberto (`OPEN-002`) |
| ADR-006 | `entrega-at-least-once-com-x-event-id` | Garantir entrega at-least-once (não exactly-once), com deduplicação do lado do cliente via `X-Event-Id` (UUID) | `[09:24]-[09:26]` | — (funcionalidade nova) | Exactly-once (`ALT-008`) | Simplicidade operacional; alinhado a padrão de mercado (citados Stripe, GitHub) | Transfere responsabilidade de deduplicação para o cliente; depende de documentação clara no portal do desenvolvedor |
| ADR-007 | `reuso-de-padroes-existentes-no-modulo-webhooks` | Módulo `webhooks` segue a estrutura padrão de módulos do projeto e reaproveita `AppError` (prefixo `WEBHOOK_`), error middleware, Pino e Prisma, sem criar infraestrutura nova | `[09:27]-[09:30]` | `src/modules/orders/*` (referência de padrão), `src/shared/errors/*`, `src/shared/logger/index.ts`, `src/middlewares/error.middleware.ts` | Nenhuma alternativa explícita foi discutida — decisão de consenso direto ("Faz sentido?" / "Faz") | Consistência arquitetural; curva de aprendizado baixa para o time | Herda eventuais limitações dos padrões atuais (ex.: ausência de framework de DI) |
| ADR-008 | `ordenacao-condicionada-a-single-worker` | Ordenação de eventos garantida apenas por `order_id`, e apenas em regime single-worker; escalar para múltiplos workers exige trabalho futuro de particionamento | `[09:12]-[09:13]` | — (comportamento do worker proposto) | Múltiplos workers com particionamento por `order_id` ou lock pessimista (mencionada como solução futura, não implementada agora) | Simplicidade imediata; atende ao requisito real dos clientes (não pediram ordenação global, `[09:14]`) | Limita escalabilidade horizontal do worker sem trabalho adicional futuro |

## U. Possíveis inconsistências e ambiguidades

1. **Nomes de tabelas não são decisões finais**: `webhook_outbox` e `webhook_dead_letter` foram citados de forma coloquial/ilustrativa ("tabela **tipo** webhook_outbox" `[09:06]` Diego; "eu **fazia** uma tabela webhook_dead_letter" `[09:18]` Diego) — devem ser tratados como propostas de nomenclatura, não como nomes definitivos confirmados por todos os participantes.
2. **Paths de endpoints incompletos**: cadastro (`POST`), edição (`PATCH`), remoção (`DELETE`) e listagem (`GET`) de webhooks tiveram apenas o verbo HTTP mencionado, sem path literal — diferente de `GET /webhooks/:id/deliveries` e `POST /admin/webhooks/dead-letter/:id/replay`, que têm paths explícitos na transcrição.
3. **TLS e limite de payload foram explicitamente classificados como NFR, não ADR**: Larissa afirma textualmente que o limite de 64KB "não vejo como decisão arquitetural separada, é só requisito não funcional" `[09:24]`; o mesmo vale para HTTPS obrigatório `[09:23]`. Documentos futuros não devem promover esses dois pontos a ADRs próprios.
4. **Rate limiting de saída não é decisão nem item definitivamente fora de escopo**: é tratado estritamente como ponto em aberto ("fica como observar e decidir depois" `[09:39]` Larissa) — não deve ser apresentado como requisito implementado nem como exclusão permanente.
5. **"Estado ativo" do webhook é um campo confirmado, mas sem endpoint de toggle explícito**: Bruno e Sofia confirmam a existência do campo `[09:21]`, mas nenhum endpoint dedicado de ativação/desativação foi discutido — presume-se cobertura via `PATCH` genérico, o que é uma inferência de baixo risco desta análise, não um fato citado literalmente.
6. **Nome do arquivo de processamento do worker não foi fechado**: Bruno propôs `webhook.worker.ts` **ou** `webhook.processor.ts` `[09:28]`; Diego apenas respondeu "Beleza", sem escolher entre as duas opções.
7. **Código atual não possui nenhum artefato de webhook**: confirmado via busca no repositório — nenhum arquivo `webhook*` ou `worker*` existe hoje. Todos os elementos da seção R são 100% propostos.
8. **Lista de redact do logger não inclui `secret` explicitamente** (`src/shared/logger/index.ts:4-11`, campos redigidos: `password`, `passwordHash`, `token`, `accessToken`, headers de auth/cookie) — não é uma decisão da reunião, é uma observação desta análise de código que merece atenção ao integrar segredos de webhook em logs futuros, dado o incidente de vazamento mencionado por Diego `[09:22]`.
9. **"Outbox" e "fila" são tratados como conceitos distintos na reunião**: Diego afirma "eu nem chamaria de 'fila' — o que a gente quer aqui é um padrão outbox" `[09:06]` — documentos futuros não devem usar os termos como sinônimos.
10. **Não existe "JWT de cliente" no sistema atual**: `src/middlewares/auth.middleware.ts` só modela roles `ADMIN`/`OPERATOR`, sempre de um usuário operador do sistema. A decisão da reunião (`DEC-027`) de que `customer_id` vem do body/path, não do JWT, é coerente com essa realidade de código — mas deve ficar explícito que não há (e não está sendo criado) um JWT que represente diretamente o cliente final.

---

## Matriz final de evidências

| ID | Categoria | Conteúdo | Fonte | Localização | Estado | Documento futuro |
| --- | --- | --- | --- | --- | --- | --- |
| CTX-001 | Contexto | 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pedem notificação em tempo real | TRANSCRICAO | `[09:00] Marcos` | DECIDIDO | PRD |
| CTX-002 | Contexto | Risco de churn da Atlas se não entregar até fim do trimestre / fim de novembro | TRANSCRICAO | `[09:00], [09:45] Marcos` | DECIDIDO | PRD |
| CTX-003 | Contexto | "Tempo real" definido pelo cliente como <10s | TRANSCRICAO | `[09:02] Marcos` | DECIDIDO | PRD |
| CTX-004 | Contexto | Modelo é outbound-only (sem recepção de webhooks) | TRANSCRICAO | `[09:02]-[09:03] Marcos, Sofia` | DECIDIDO | PRD, RFC |
| CTX-005 | Contexto | Estimativa de 3 sprints incluindo revisão de segurança | TRANSCRICAO | `[09:45]-[09:47] Larissa, Sofia` | DECIDIDO | PRD, RFC |
| DEC-001 | Decisão | Webhooks exclusivamente outbound | TRANSCRICAO | `[09:02]-[09:03] Marcos, Sofia` | DECIDIDO | RFC, FDD |
| DEC-002 | Decisão | Padrão Outbox em MySQL, não síncrono nem fila externa | TRANSCRICAO | `[09:03]-[09:08] Bruno, Diego, Larissa` | DECIDIDO | ADR-001, RFC |
| DEC-003 | Decisão | Worker faz polling a cada 2s | TRANSCRICAO | `[09:09]-[09:10] Diego, Larissa` | DECIDIDO | ADR-002, FDD |
| DEC-004 | Decisão | Worker roda como processo separado (`src/worker.ts`) | TRANSCRICAO | `[09:11] Diego, Larissa` | DECIDIDO | ADR-002, FDD |
| DEC-005 | Decisão | Worker usa `PrismaClient` próprio, mesmo banco | TRANSCRICAO | `[09:11], [09:29]-[09:30] Diego, Bruno` | DECIDIDO | ADR-002, FDD |
| DEC-006 | Decisão | Ordenação só por `order_id`, só single-worker | TRANSCRICAO | `[09:12]-[09:13] Diego, Larissa` | DECIDIDO | ADR-008, FDD |
| DEC-007 | Decisão | Retry com 5 tentativas, backoff exponencial | TRANSCRICAO | `[09:14]-[09:17] Diego, Larissa` | DECIDIDO | ADR-003, FDD |
| DEC-008 | Decisão | Progressão de backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` | DECIDIDO | ADR-003, FDD |
| DEC-009 | Decisão | DLQ em tabela separada `webhook_dead_letter` | TRANSCRICAO | `[09:17]-[09:18] Diego` | DECIDIDO | ADR-003, FDD |
| DEC-010 | Decisão | Replay manual via `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | `[09:18]-[09:19] Diego, Larissa` | DECIDIDO | ADR-003, FDD |
| DEC-011 | Decisão | Assinatura HMAC-SHA256, header `X-Signature` | TRANSCRICAO | `[09:19]-[09:20] Sofia` | DECIDIDO | ADR-004, FDD |
| DEC-012 | Decisão | Secret única por endpoint (não global) | TRANSCRICAO | `[09:21] Sofia` | DECIDIDO | ADR-004 |
| DEC-013 | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21]-[09:22] Sofia, Diego` | DECIDIDO | ADR-005 |
| DEC-014 | Decisão | TLS obrigatório (https-only), classificado como NFR | TRANSCRICAO | `[09:23] Sofia, Larissa` | DECIDIDO | FDD (NFR) |
| DEC-015 | Decisão | Limite de payload 64KB, erro se exceder, classificado como NFR | TRANSCRICAO | `[09:23]-[09:24] Sofia, Diego, Larissa` | DECIDIDO | FDD (NFR) |
| DEC-016 | Decisão | At-least-once com dedup via `X-Event-Id` | TRANSCRICAO | `[09:24]-[09:26] Diego` | DECIDIDO | ADR-006, FDD |
| DEC-017 | Decisão | Módulo `src/modules/webhooks` segue padrão existente | TRANSCRICAO | `[09:27]-[09:28] Bruno, Diego` | DECIDIDO | ADR-007, FDD |
| DEC-018 | Decisão | Worker entry separado + lógica de processamento no módulo | TRANSCRICAO | `[09:28] Bruno, Diego` | DECIDIDO | ADR-002, ADR-007 |
| DEC-019 | Decisão | Erros `AppError` com prefixo `WEBHOOK_` | TRANSCRICAO | `[09:28]-[09:29] Bruno, Larissa` | DECIDIDO | ADR-007, FDD |
| DEC-020 | Decisão | Reuso de Pino e error middleware sem alteração | TRANSCRICAO | `[09:29] Bruno` | DECIDIDO | ADR-007, FDD |
| DEC-021 | Decisão | Inserção do evento na outbox dentro da transação de `changeStatus` | TRANSCRICAO | `[09:40]-[09:41] Bruno, Diego` | DECIDIDO | ADR-001, FDD |
| DEC-022 | Decisão | Função pura `publishWebhookEvent(tx, order, from, to)` | TRANSCRICAO | `[09:41] Bruno, Diego` | DECIDIDO | ADR-001, FDD |
| DEC-023 | Decisão | Timeout HTTP do worker: 10s | TRANSCRICAO | `[09:42] Diego, Sofia` | DECIDIDO | ADR-003, FDD |
| DEC-024 | Decisão | Payload sem `items`, campos definidos | TRANSCRICAO | `[09:43] Diego, Bruno` | DECIDIDO | FDD |
| DEC-025 | Decisão | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | TRANSCRICAO | `[09:44] Diego, Sofia` | DECIDIDO | ADR-004, ADR-006, FDD |
| DEC-026 | Decisão | Filtro de status aplicado na inserção da outbox | TRANSCRICAO | `[09:33]-[09:34] Bruno, Diego` | DECIDIDO | ADR-001, FDD |
| DEC-027 | Decisão | `customer_id` vem do body/path, não do JWT | TRANSCRICAO | `[09:31]-[09:32] Bruno, Larissa, Marcos` | DECIDIDO | FDD |
| DEC-028 | Decisão | Replay exige role `ADMIN` + log de auditoria | TRANSCRICAO | `[09:35]-[09:36] Sofia, Larissa` | DECIDIDO | ADR-007, FDD |
| DEC-029 | Decisão | CRUD aceita qualquer role autenticada, nesta fase | TRANSCRICAO | `[09:36]-[09:37] Sofia, Marcos` | DECIDIDO | FDD |
| DEC-030 | Decisão | IDs da outbox em UUID | TRANSCRICAO | `[09:51] Larissa, Diego` | DECIDIDO | ADR-007, FDD |
| DEC-031 | Decisão | Payload snapshot renderizado na inserção | TRANSCRICAO | `[09:51]-[09:52] Larissa, Diego, Bruno` | DECIDIDO | ADR-001, FDD |
| DEC-032 | Decisão | Estimativa de 3 sprints + revisão de segurança | TRANSCRICAO | `[09:45]-[09:47] Larissa, Sofia` | DECIDIDO | PRD |
| FR-001 | Requisito Funcional | Cadastro de webhook (`POST`) | TRANSCRICAO | `[09:31]-[09:32] Marcos, Bruno` | DECIDIDO | PRD, FDD |
| FR-002 | Requisito Funcional | Listagem de webhooks de um customer (`GET`) | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO | PRD, FDD |
| FR-003 | Requisito Funcional | Atualização de webhook (`PATCH`) | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO | PRD, FDD |
| FR-004 | Requisito Funcional | Estado ativo/inativo do webhook | TRANSCRICAO | `[09:21] Bruno, Sofia` | DECIDIDO | PRD, FDD |
| FR-005 | Requisito Funcional | Exclusão de webhook (`DELETE`) | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO | PRD, FDD |
| FR-006 | Requisito Funcional | Rotação de secret | TRANSCRICAO | `[09:21]-[09:22] Sofia` | DECIDIDO (endpoint aberto) | PRD, FDD |
| FR-007 | Requisito Funcional | Entrega de eventos via webhook HTTP | TRANSCRICAO | `[09:19]-[09:26], [09:42]-[09:44]` | DECIDIDO | PRD, FDD |
| FR-008 | Requisito Funcional | Filtragem de eventos por status | TRANSCRICAO | `[09:33]-[09:34] Marcos, Bruno, Diego` | DECIDIDO | PRD, FDD |
| FR-009 | Requisito Funcional | Retentativas automáticas | TRANSCRICAO | `[09:14]-[09:17]` | DECIDIDO | PRD, FDD |
| FR-010 | Requisito Funcional | Envio para DLQ | TRANSCRICAO | `[09:17]-[09:18] Diego` | DECIDIDO | PRD, FDD |
| FR-011 | Requisito Funcional | Replay manual de DLQ | TRANSCRICAO | `[09:18]-[09:19], [09:35]-[09:36]` | DECIDIDO | PRD, FDD |
| FR-012 | Requisito Funcional | Autenticação/autorização diferenciada por endpoint | TRANSCRICAO | `[09:31]-[09:37]` | DECIDIDO | PRD, FDD |
| FR-013 | Requisito Funcional | Identificação única do evento (`X-Event-Id`) | TRANSCRICAO | `[09:24]-[09:26] Diego` | DECIDIDO | PRD, FDD |
| FR-014 | Requisito Funcional | Histórico de entregas (`GET /webhooks/:id/deliveries`) | TRANSCRICAO | `[09:34]-[09:35] Marcos` | DECIDIDO | PRD, FDD |
| NFR-001 | Requisito Não Funcional | Latência <10s / mínimo 2s | TRANSCRICAO | `[09:02], [09:09]-[09:10]` | DECIDIDO | PRD, FDD |
| NFR-002 | Requisito Não Funcional | Worker em processo separado (disponibilidade) | TRANSCRICAO | `[09:11] Diego` | DECIDIDO | FDD |
| NFR-003 | Requisito Não Funcional | HMAC-SHA256 | TRANSCRICAO | `[09:19]-[09:20] Sofia` | DECIDIDO | FDD |
| NFR-004 | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | `[09:21] Sofia` | DECIDIDO | FDD |
| NFR-005 | Requisito Não Funcional | TLS obrigatório | TRANSCRICAO | `[09:23] Sofia` | DECIDIDO | FDD |
| NFR-006 | Requisito Não Funcional | Payload máx. 64KB | TRANSCRICAO | `[09:23]-[09:24]` | DECIDIDO | FDD |
| NFR-007 | Requisito Não Funcional | Timeout HTTP 10s | TRANSCRICAO | `[09:42] Diego` | DECIDIDO | FDD |
| NFR-008 | Requisito Não Funcional | Retry com backoff, 5 tentativas | TRANSCRICAO | `[09:14]-[09:17]` | DECIDIDO | FDD |
| NFR-009 | Requisito Não Funcional | At-least-once | TRANSCRICAO | `[09:24]-[09:26] Diego` | DECIDIDO | FDD |
| NFR-010 | Requisito Não Funcional | Ordenação condicionada a single-worker | TRANSCRICAO | `[09:12]-[09:13]` | LIMITACAO_ACEITA | FDD |
| NFR-011 | Requisito Não Funcional | Observabilidade parcial (log de auditoria + histórico) | TRANSCRICAO | `[09:34]-[09:36]` | ABERTO (cobertura incompleta) | FDD |
| NFR-012 | Requisito Não Funcional | Escalabilidade adiada (single-worker) | TRANSCRICAO | `[09:13] Diego` | ADIADO | FDD |
| NFR-013 | Requisito Não Funcional | Atomicidade outbox + transação | TRANSCRICAO | `[09:06]-[09:08], [09:40]-[09:41]` | DECIDIDO | FDD |
| NFR-014 | Requisito Não Funcional | Compatibilidade Prisma/DB compartilhado | TRANSCRICAO | `[09:29]-[09:30]` | DECIDIDO | FDD |
| NFR-015 | Requisito Não Funcional | Retenção/arquivamento (fora de escopo) | TRANSCRICAO | `[09:08] Diego` | ABERTO/FORA DE ESCOPO | PRD (fora de escopo), FDD |
| ALT-001 | Alternativa Descartada | Disparo síncrono | TRANSCRICAO | `[09:03]-[09:05] Bruno, Larissa` | DESCARTADO | RFC |
| ALT-002 | Alternativa Descartada | Fila externa (Redis Streams) | TRANSCRICAO | `[09:07] Larissa, Diego` | DESCARTADO | RFC |
| ALT-003 | Alternativa Descartada | Trigger de banco reativo | TRANSCRICAO | `[09:09] Bruno, Diego` | DESCARTADO | RFC |
| ALT-004 | Alternativa Descartada | Retry indefinido | TRANSCRICAO | `[09:15] Diego` | DESCARTADO | RFC, ADR-003 |
| ALT-005 | Alternativa Descartada | 3 tentativas de retry | TRANSCRICAO | `[09:16] Bruno, Diego` | DESCARTADO | RFC, ADR-003 |
| ALT-006 | Alternativa Descartada | Status "failed" na própria outbox | TRANSCRICAO | `[09:17]-[09:18] Diego` | DESCARTADO | RFC, ADR-003 |
| ALT-007 | Alternativa Descartada | Truncar payload grande | TRANSCRICAO | `[09:23]-[09:24] Sofia` | DESCARTADO | RFC |
| ALT-008 | Alternativa Descartada | Exactly-once | TRANSCRICAO | `[09:24]-[09:26] Diego` | DESCARTADO | RFC, ADR-006 |
| OPEN-001 | Questão em Aberto | Rate limiting de envio ao cliente | TRANSCRICAO | `[09:38]-[09:39] Diego, Larissa` | ABERTO | RFC |
| OPEN-002 | Questão em Aberto | Contrato do endpoint de rotação de secret | TRANSCRICAO | `[09:21]-[09:22] Sofia` | ABERTO | RFC, FDD |
| OPEN-003 | Questão em Aberto | Possível endurecimento futuro de RBAC no CRUD | TRANSCRICAO | `[09:36]-[09:37] Sofia, Marcos` | ABERTO | RFC |
| OUT-001 | Fora de Escopo/Adiado | Notificação por e-mail em falhas repetidas | TRANSCRICAO | `[09:37]-[09:38] Marcos, Larissa` | ADIADO | PRD (fora de escopo) |
| OUT-002 | Fora de Escopo/Adiado | Dashboard visual para o cliente | TRANSCRICAO | `[09:39]-[09:40] Marcos, Larissa` | DESCARTADO (desta fase) | PRD (fora de escopo) |
| OUT-003 | Fora de Escopo/Adiado | Arquivamento de eventos entregues após 30 dias | TRANSCRICAO | `[09:08] Diego` | ADIADO | PRD (fora de escopo) |
| OUT-004 | Fora de Escopo/Adiado | Rate limiting de saída (condicional) | TRANSCRICAO | `[09:39] Diego` | ADIADO | PRD (fora de escopo), RFC |
| OUT-005 | Fora de Escopo/Adiado | Endurecimento futuro de RBAC no CRUD | TRANSCRICAO | `[09:36]-[09:37] Sofia` | LIMITACAO_ACEITA | PRD (fora de escopo) |
| LIMIT-001 | Limitação | Ordenação só por `order_id`, só single-worker | TRANSCRICAO | `[09:12]-[09:13] Diego, Larissa` | LIMITACAO_ACEITA | FDD, ADR-008 |
| LIMIT-002 | Limitação | Latência mínima de 2s (polling) | TRANSCRICAO | `[09:09]-[09:10]` | LIMITACAO_ACEITA | FDD |
| LIMIT-003 | Limitação | At-least-once, não exactly-once | TRANSCRICAO | `[09:24]-[09:26]` | LIMITACAO_ACEITA | FDD, ADR-006 |
| LIMIT-004 | Limitação | CRUD sem restrição de role adicional | TRANSCRICAO | `[09:36]-[09:37]` | LIMITACAO_ACEITA | FDD |
| LIMIT-005 | Limitação | Escalar workers quebra ordenação atual | TRANSCRICAO | `[09:13] Diego` | LIMITACAO_ACEITA | FDD, ADR-008 |
| METRIC-001 | Métrica | "Tempo real" < 10s | TRANSCRICAO | `[09:02] Marcos` | DECIDIDO | PRD |
| METRIC-002 | Métrica | Polling a cada 2s | TRANSCRICAO | `[09:09]-[09:10]` | DECIDIDO | FDD |
| METRIC-003 | Métrica | 5 tentativas de retry | TRANSCRICAO | `[09:15]-[09:17]` | DECIDIDO | FDD |
| METRIC-004 | Métrica | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` | DECIDIDO | FDD |
| METRIC-005 | Métrica | Grace period de secret 24h | TRANSCRICAO | `[09:21]-[09:22]` | DECIDIDO | FDD |
| METRIC-006 | Métrica | Payload máx. 64KB | TRANSCRICAO | `[09:23]-[09:24]` | DECIDIDO | FDD |
| METRIC-007 | Métrica | Timeout HTTP 10s | TRANSCRICAO | `[09:42] Diego` | DECIDIDO | FDD |
| METRIC-008 | Métrica | Retenção ~30 dias (fora de escopo) | TRANSCRICAO | `[09:08] Diego` | ABERTO/FORA DE ESCOPO | PRD (fora de escopo) |
| METRIC-009 | Métrica | Histórico de últimos 100 webhooks | TRANSCRICAO | `[09:34] Marcos` | DECIDIDO | FDD |
| METRIC-010 | Métrica | Prazo fim de novembro | TRANSCRICAO | `[09:45] Marcos` | DECIDIDO | PRD |
| METRIC-011 | Métrica | Estimativa de 3 sprints | TRANSCRICAO | `[09:45]-[09:47] Larissa` | DECIDIDO | PRD |
| METRIC-012 | Métrica | 2 dias úteis de revisão de segurança | TRANSCRICAO | `[09:46] Sofia` | DECIDIDO | PRD |
| RISK-001 | Risco | Cliente lento trava outros pedidos (síncrono) | TRANSCRICAO | `[09:04] Bruno` | DECIDIDO (mitigado) | PRD, RFC |
| RISK-002 | Risco | Inconsistência status/evento | TRANSCRICAO | `[09:06], [09:40]-[09:41]` | DECIDIDO (mitigado) | PRD, RFC |
| RISK-003 | Risco | Vazamento de secret compartilhada | TRANSCRICAO | `[09:21]-[09:22] Sofia, Diego` | DECIDIDO (mitigado) | PRD, ADR-004 |
| RISK-004 | Risco | Indisponibilidade prolongada do cliente | TRANSCRICAO | `[09:16]-[09:17] Diego` | DECIDIDO (mitigado) | PRD, ADR-003 |
| RISK-005 | Risco | Acúmulo de eventos na outbox | TRANSCRICAO | `[09:07]-[09:08] Bruno, Diego` | DECIDIDO (mitigado) | PRD, FDD |
| RISK-006 | Risco | Perda do worker se acoplado à API | TRANSCRICAO | `[09:11] Diego` | DECIDIDO (mitigado) | PRD, ADR-002 |
| RISK-007 | Risco | Bombardeio de chamadas ao cliente | TRANSCRICAO | `[09:38]-[09:39] Diego, Larissa` | ABERTO (não mitigado) | PRD, RFC |
| RISK-008 | Risco | Replay sem auditoria | TRANSCRICAO | `[09:35]-[09:36] Sofia` | DECIDIDO (mitigado) | PRD, FDD |
| RISK-009 | Risco | Payload excessivamente grande | TRANSCRICAO | `[09:23]-[09:24]` | DECIDIDO (mitigado) | PRD, FDD |
| API-001 | Endpoint | `POST` cadastro de webhook (path não especificado) | TRANSCRICAO | `[09:31]-[09:32] Marcos, Bruno` | DECIDIDO (parcial) | FDD |
| API-002 | Endpoint | `GET` listagem de webhooks (path não especificado) | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO (parcial) | FDD |
| API-003 | Endpoint | `PATCH` edição de webhook (path não especificado) | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO (parcial) | FDD |
| API-004 | Endpoint | `DELETE` remoção de webhook (path não especificado) | TRANSCRICAO | `[09:33] Bruno` | DECIDIDO (parcial) | FDD |
| API-005 | Endpoint | `GET /webhooks/:id/deliveries` | TRANSCRICAO | `[09:34]-[09:35] Marcos` | DECIDIDO | FDD |
| API-006 | Endpoint | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | `[09:18]-[09:19], [09:35]-[09:36]` | DECIDIDO | FDD |
| API-007 | Endpoint | Rotação de secret (método/path não especificados) | TRANSCRICAO | `[09:21]-[09:22] Sofia` | ABERTO (`OPEN-002`) | FDD |
| EVENT-001 | Evento | Tipo `order.status_changed` | TRANSCRICAO | `[09:43] Diego` | DECIDIDO | FDD |
| EVENT-002 | Evento | Campos do payload (sem `items`) | TRANSCRICAO | `[09:43] Diego, Bruno` | DECIDIDO | FDD |
| EVENT-003 | Evento | Headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type`) | TRANSCRICAO | `[09:44] Diego, Sofia` | DECIDIDO | FDD |
| EVENT-004 | Evento | `event_id` UUID para dedup | TRANSCRICAO | `[09:25] Diego` | DECIDIDO | FDD |
| EVENT-005 | Evento | Tamanho máx. 64KB | TRANSCRICAO | `[09:23]-[09:24]` | DECIDIDO | FDD |
| EVENT-006 | Evento | Ordering condicionada a single-worker | TRANSCRICAO | `[09:12]-[09:13]` | LIMITACAO_ACEITA | FDD, ADR-008 |
| CODE-001 | Integração de Código | `OrderService.changeStatus` — transação atômica | CODIGO | `src/modules/orders/order.service.ts:126-179` | EXISTENTE | FDD |
| CODE-002 | Integração de Código | Modelos Prisma (`Order`, `OrderStatusHistory`, `OrderStatus`) | CODIGO | `prisma/schema.prisma:74-131` | EXISTENTE | FDD |
| CODE-003 | Integração de Código | Composição de dependências em `src/app.ts` | CODIGO | `src/app.ts:26-53` | EXISTENTE | FDD |
| CODE-004 | Integração de Código | Registro de rotas em `src/routes/index.ts` | CODIGO | `src/routes/index.ts:21-31` | EXISTENTE | FDD |
| CODE-005 | Integração de Código | Autenticação JWT (`authenticate`, `requireRole`) | CODIGO | `src/middlewares/auth.middleware.ts:27-61` | EXISTENTE | FDD |
| CODE-006 | Integração de Código | Validação Zod (`validate`) | CODIGO | `src/middlewares/validate.middleware.ts:11-37` | EXISTENTE | FDD |
| CODE-007 | Integração de Código | `AppError` e subclasses | CODIGO | `src/shared/errors/app-error.ts`, `http-errors.ts` | EXISTENTE | FDD |
| CODE-008 | Integração de Código | Middleware de erro central | CODIGO | `src/middlewares/error.middleware.ts:14-65` | EXISTENTE | FDD |
| CODE-009 | Integração de Código | Logger Pino | CODIGO | `src/shared/logger/index.ts:4-32` | EXISTENTE | FDD |
| CODE-010 | Integração de Código | Configuração do Prisma | CODIGO | `src/config/database.ts:1-11` | EXISTENTE | FDD |
| CODE-011 | Integração de Código | `server.ts` como referência para `worker.ts` | CODIGO | `src/server.ts:1-27` | EXISTENTE (referência) | FDD |
| CODE-012 | Integração de Código | Testes existentes de pedidos | CODIGO | `tests/orders.test.ts`, `tests/auth.test.ts` | EXISTENTE | FDD |
| ADR-001 | Candidato a ADR | Padrão Outbox no MySQL | TRANSCRICAO | `[09:03]-[09:08], [09:40]-[09:41], [09:51]-[09:52]` | PROPOSTO | ADRs |
| ADR-002 | Candidato a ADR | Worker separado com polling | TRANSCRICAO | `[09:09]-[09:11]` | PROPOSTO | ADRs |
| ADR-003 | Candidato a ADR | Retry com backoff e DLQ | TRANSCRICAO | `[09:14]-[09:19], [09:42]` | PROPOSTO | ADRs |
| ADR-004 | Candidato a ADR | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | `[09:19]-[09:22]` | PROPOSTO | ADRs |
| ADR-005 | Candidato a ADR | Rotação de secret com grace period 24h | TRANSCRICAO | `[09:21]-[09:22]` | PROPOSTO | ADRs |
| ADR-006 | Candidato a ADR | At-least-once com `X-Event-Id` | TRANSCRICAO | `[09:24]-[09:26]` | PROPOSTO | ADRs |
| ADR-007 | Candidato a ADR | Reuso de padrões existentes | TRANSCRICAO | `[09:27]-[09:30]` | PROPOSTO | ADRs |
| ADR-008 | Candidato a ADR | Ordering condicionada a single-worker | TRANSCRICAO | `[09:12]-[09:13]` | PROPOSTO | ADRs |

---

## Validações finais

1. **Cada decisão possui evidência?** Sim — todas as 32 decisões da seção C têm timestamp `[hh:mm]` e participante(s) responsável(is) citados literalmente da transcrição.
2. **Cada requisito funcional possui evidência?** Sim — todos os 14 FRs (seção D) têm fonte/timestamp/participante rastreáveis.
3. **Todas as alternativas descartadas foram realmente discutidas?** Sim — as 8 alternativas (seção F) correspondem a trechos explícitos de discussão e descarte na reunião; nenhuma é hipotética.
4. **Itens fora de escopo corretamente classificados?** Sim — seção H diferencia "adiado" (`OUT-001`, `OUT-003`, `OUT-004`), "descartado desta fase" (`OUT-002`) e "limitação aceita temporariamente" (`OUT-005`).
5. **Todos os caminhos de código existem?** Sim — todos os caminhos citados nas seções O a S foram lidos diretamente do repositório nesta análise (`src/app.ts`, `src/server.ts`, `src/routes/index.ts`, `src/config/database.ts`, `src/config/env.ts`, `src/middlewares/*`, `src/shared/errors/*`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/modules/orders/*`, `prisma/schema.prisma`, `tests/orders.test.ts`, `tests/auth.test.ts`, `package.json`).
6. **Nenhum elemento futuro foi descrito como existente?** Confirmado — busca por `webhook*` e `worker*` no repositório não retornou nenhum arquivo; seção R lista explicitamente todos os elementos como propostos, não implementados.
7. **Nenhuma pergunta foi convertida indevidamente em decisão?** Sim — perguntas sem resposta fechada (rate limiting, contrato de rotação de secret, endurecimento futuro de RBAC) permanecem na seção G (`OPEN-NNN`), não na seção C (`DEC-NNN`).
8. **Timestamps e nomes no formato correto?** Sim — todos no formato `[hh:mm] Nome`, extraídos literalmente de `TRANSCRICAO.md`.
9. **A análise permite produzir pelo menos 8 requisitos funcionais?** Sim — 14 FRs identificados (seção D), cobrindo todas as categorias solicitadas (cadastro, listagem, atualização, ativação/desativação, exclusão, rotação de secret, entrega, filtragem, retries, DLQ, replay, autenticação, identificação única de evento) mais histórico de entregas.
10. **Existem evidências suficientes para 5 a 8 ADRs?** Sim — 8 candidatos a ADR (seção T), todos com fonte na transcrição e ao menos um (`ADR-001`, `ADR-002`, `ADR-007`) referenciando arquivos/padrões reais do código.
11. **Há pelo menos 2 alternativas descartadas?** Sim — 8 alternativas na seção F.
12. **Há pelo menos 2 questões em aberto ou adiadas?** Sim — 3 questões em aberto (seção G) e 5 itens adiados/fora de escopo (seção H).
13. **Há pelo menos 5 pontos reais de integração com o código?** Sim — 12 pontos de integração (`CODE-001` a `CODE-012`, seção Q), cobrindo todos os itens mínimos exigidos (transação `changeStatus`, modelos Prisma, `app.ts`, `routes/index.ts`, autenticação JWT, validação Zod, `AppError`, middleware de erro, logger Pino, configuração do Prisma, referência `server.ts`→`worker.ts`, testes existentes).

**Nenhum arquivo em `src/`, `prisma/`, `tests/`, configurações ou `TRANSCRICAO.md` foi alterado durante esta análise.**
