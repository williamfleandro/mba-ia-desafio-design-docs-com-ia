# PRD: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | William Ferreira Leandro |
| Status | Proposto |
| Data | 03 de agosto de 2026 |
| Feature | Sistema de Webhooks de Notificação de Pedidos |

Documentos técnicos relacionados: [`./RFC.md`](./RFC.md) (proposta arquitetural) e [`./FDD.md`](./FDD.md) (detalhamento de implementação). Este documento trata exclusivamente do problema de produto/negócio e dos requisitos que a solução deve atender — não repete decisões técnicas já registradas nos seis ADRs em `docs/adrs/`.

## Resumo e contexto

O OMS (Order Management System) já gerencia autenticação, usuários, clientes, produtos e pedidos como funcionalidades existentes da plataforma. Hoje, clientes B2B integrados à API acompanham mudanças de status de seus pedidos fazendo consultas repetidas ao endpoint de listagem de pedidos — não existe, atualmente, nenhum mecanismo de notificação externa (evento, fila ou webhook) na aplicação.

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — enviaram um pedido formal para serem notificados em tempo real quando o status de seus pedidos muda (`[09:00] Marcos`). A Atlas sinalizou que pode migrar para um concorrente caso a feature não seja entregue dentro do prazo (inicialmente descrito como "fim do trimestre", depois refinado para **fim de novembro**) (`[09:00]`, `[09:45] Marcos`).

## Problema e motivação

- A integração atual, baseada em polling manual do endpoint de pedidos, é descrita pelo próprio Product Manager como **"lenta e cara"** para os clientes (`[09:00] Marcos`).
- Os clientes dependem de consultas repetidas e periódicas para perceber uma mudança de status, o que gera atraso na percepção do evento e desperdício de chamadas quando nada mudou.
- Esse modelo representa custo operacional direto para os clientes integrados, que precisam manter processos de polling ativos continuamente.
- Existe um **risco comercial concreto de perda de cliente**: a Atlas declarou que pode migrar para um concorrente se a notificação em tempo real não for entregue no prazo combinado (`[09:00]`, `[09:45] Marcos`).
- A necessidade de negócio é clara: notificar os clientes de forma **outbound** (o sistema envia, o cliente recebe) assim que o status do pedido muda, eliminando a dependência de polling (`[09:02]-[09:03] Marcos, Sofia`).

## Público-alvo

- **Clientes B2B integrados ao OMS** — Atlas Comercial, MaxDistribuição e Nova Cargo hoje, e potencialmente outros clientes com o mesmo perfil de integração no futuro (`[09:00] Marcos`).
- **Operadores autenticados** que administram o cadastro de webhooks (criação, edição, exclusão, consulta) em nome do cliente (`[09:31]-[09:33] Marcos, Bruno`).
- **Administradores** (role `ADMIN`) responsáveis por reprocessar entregas que falharam permanentemente (`[09:35]-[09:36] Sofia, Larissa`).
- **Times internos de operação/suporte**, na medida em que utilizam o histórico de entregas e o replay administrativo para diagnosticar e resolver problemas de integração de clientes — uso implícito nas próprias funcionalidades de histórico e replay (`[09:34]-[09:36]`), não uma persona descrita explicitamente à parte na reunião.

## Cenários de uso

1. Um cliente (via usuário autenticado) cadastra um endpoint de webhook informando a URL de destino (`[09:31]-[09:32] Marcos, Bruno`).
2. O cliente escolhe quais status de pedido deseja receber, filtrando o volume de notificações (`[09:33]-[09:34] Marcos, Bruno, Diego`).
3. O OMS envia automaticamente uma notificação assim que o status de um pedido do cliente muda (`[09:40]-[09:44]`).
4. O cliente valida a assinatura da notificação recebida para garantir que ela realmente veio do OMS (`[09:19]-[09:20] Sofia`).
5. O cliente identifica e descarta notificações duplicadas usando o identificador único do evento (`[09:24]-[09:26] Diego`).
6. Um operador consulta o histórico de entregas de um webhook para investigar problemas de integração (`[09:34]-[09:35] Marcos`).
7. Um administrador reprocessa manualmente uma notificação que falhou permanentemente (`[09:18]-[09:19]`, `[09:35]-[09:36]`).
8. O cliente solicita a rotação de sua credencial de assinatura sem interromper a integração em produção (`[09:21]-[09:22] Sofia`).

## Objetivos

- Substituir o polling como mecanismo primário de acompanhamento de status de pedidos pelos clientes B2B (`[09:00]-[09:02] Marcos`).
- Entregar notificações automáticas de mudança de status assim que ela ocorre (`[09:03]-[09:08]`).
- Manter a latência de notificação abaixo de 10 segundos em condições normais de operação, atendendo à expectativa de "tempo real" definida pelo cliente (`[09:02] Marcos`, `[09:09]-[09:10]`).
- Evitar que a disponibilidade ou a latência do endpoint externo do cliente bloqueie a mudança de status do pedido, preservando ao mesmo tempo a consistência atômica entre a alteração do pedido e o registro da notificação: a chamada HTTP ao cliente nunca ocorre dentro da transação de status nem pode atrasá-la, enquanto o registro do evento de notificação faz parte dessa mesma transação e, se falhar, deve provocar rollback da mudança de status para preservar essa atomicidade (`[09:04]-[09:08] Bruno, Diego`).
- Proteger a autenticidade e a integridade das notificações enviadas a terceiros (`[09:19]-[09:20] Sofia`).
- Tolerar indisponibilidade temporária do endpoint do cliente sem descartar a notificação prematuramente (`[09:14]-[09:17]`).
- Manter aderência à arquitetura e aos padrões já existentes no OMS, sem introduzir infraestrutura nova (`[09:27]-[09:30]`).

## Métricas de sucesso

- **Latência de entrega abaixo de 10 segundos** em condições normais — meta quantitativa definida diretamente pela expectativa do cliente de "tempo real" (`[09:02] Marcos`), e endereçada pelo desenho técnico (intervalo de polling de 2 segundos, `[09:09]-[09:10]`).

As métricas abaixo são propostas de **medição operacional** para acompanhar a saúde da feature após o lançamento; nenhuma meta numérica para elas foi definida na reunião — apenas o compromisso de latência acima:

- Número de eventos entregues com sucesso.
- Número de falhas de entrega.
- Número de retentativas realizadas.
- Número de eventos que chegam à DLQ.
- Idade do evento pendente mais antigo na fila (indicador de acúmulo/atraso).
- Redução do volume de chamadas de polling em `GET /orders` pelos clientes migrados — não há medição ou meta definida na reunião para esse indicador; fica como possível acompanhamento futuro.

Não há, na reunião, nenhum percentual de sucesso, taxa de adoção ou SLA de disponibilidade formalmente definido além do limiar de latência acima.

## Escopo

### Incluído

- Cadastro, listagem, atualização e exclusão de webhooks (`[09:31]-[09:33] Marcos, Bruno`).
- Ativação/desativação de um webhook cadastrado (`[09:21] Bruno, Sofia`).
- Filtro de eventos por status de pedido desejado (`[09:33]-[09:34]`).
- Geração automática de secret na criação do webhook (`[09:31] Marcos`).
- Rotação de secret com grace period (`[09:21]-[09:22] Sofia`).
- Entrega de eventos de mudança de status via HTTP outbound (`[09:19]-[09:26]`, `[09:42]-[09:44]`).
- Assinatura HMAC-SHA256 das notificações (`[09:19]-[09:20] Sofia`).
- Retentativas automáticas em caso de falha de entrega (`[09:14]-[09:17]`).
- Dead Letter Queue (DLQ) para falhas permanentes (`[09:17]-[09:18] Diego`).
- Replay manual de eventos em DLQ, restrito a administradores (`[09:18]-[09:19]`, `[09:35]-[09:36]`).
- Histórico de entregas por webhook (`[09:34]-[09:35] Marcos`).
- Autenticação (JWT) em todos os endpoints do módulo, e autorização administrativa (`ADMIN`) especificamente para o replay de DLQ (`[09:31]-[09:37]`).

### Fora de escopo

| Item | Classificação | Fonte |
| --- | --- | --- |
| Notificação por e-mail em caso de falhas repetidas | Adiado para fase futura | `[09:37]-[09:38] Marcos, Larissa` |
| Dashboard visual dedicado para o cliente | Descartado nesta fase (projeto separado do time de frontend) | `[09:39]-[09:40] Marcos, Larissa` |
| Webhooks inbound (recepção de eventos de clientes) | Descartado — modelo é exclusivamente outbound | `[09:02]-[09:03] Marcos, Sofia` |
| Garantia de entrega exactly-once | Descartado em favor de at-least-once | `[09:24]-[09:26] Diego` |
| Escalonamento para múltiplos workers em paralelo | Adiado, sem solução avaliada | `[09:13] Diego` |
| Arquivamento automático de eventos entregues após ~30 dias | Fora de escopo explícito desta feature | `[09:08] Diego` |
| Redis Streams ou mensageria externa dedicada | Descartado — considerado overengineering para o tamanho do time | `[09:07] Larissa, Diego` |

## Requisitos funcionais

| ID | Requisito | Ator | Critério resumido | Fonte |
| --- | --- | --- | --- | --- |
| PRD-FR-01 | Cadastro de webhook | Usuário autenticado (em nome do cliente) | Endpoint é criado com URL e status desejados; secret é gerada pelo sistema e devolvida na resposta de criação | `[09:31]-[09:32] Marcos, Bruno` |
| PRD-FR-02 | Listagem de webhooks de um cliente | Usuário autenticado | Retorna os webhooks cadastrados para um `customerId` | `[09:33] Bruno` |
| PRD-FR-03 | Atualização de webhook | Usuário autenticado | Permite alterar URL, status filtrados e outros dados do cadastro | `[09:33] Bruno` |
| PRD-FR-04 | Exclusão de webhook | Usuário autenticado | Remove um webhook cadastrado | `[09:33] Bruno` |
| PRD-FR-05 | Ativação/desativação de webhook | Usuário autenticado | Webhook inativo deixa de receber notificações | `[09:21] Bruno, Sofia` |
| PRD-FR-06 | Filtro de eventos por status | Sistema | Notificação só é gerada se algum webhook do cliente estiver inscrito naquele status | `[09:33]-[09:34] Marcos, Bruno, Diego` |
| PRD-FR-07 | Rotação de secret | Cliente (via API) | Nova secret é gerada; secret anterior permanece válida por 24h em paralelo | `[09:21]-[09:22] Sofia` |
| PRD-FR-08 | Entrega de evento de mudança de status | Sistema | Notificação HTTP assinada é enviada à URL cadastrada quando o pedido muda para um status monitorado | `[09:19]-[09:26]`, `[09:42]-[09:44]` |
| PRD-FR-09 | Retentativas automáticas em falha de entrega | Sistema | Nova tentativa é feita após intervalo de espera crescente, até um limite de tentativas | `[09:14]-[09:17]` |
| PRD-FR-10 | Envio para Dead Letter Queue (DLQ) | Sistema | Evento que esgota as tentativas é registrado separadamente para investigação | `[09:17]-[09:18] Diego` |
| PRD-FR-11 | Replay manual de evento em DLQ | Administrador | Evento em DLQ pode ser reenviado manualmente; ação é restrita a `ADMIN` e registrada para auditoria | `[09:18]-[09:19]`, `[09:35]-[09:36]` |
| PRD-FR-12 | Histórico de entregas | Usuário autenticado | Consulta dos últimos 100 envios de um webhook, com sucesso/falha, payload, resposta e tempo | `[09:34]-[09:35] Marcos` |
| PRD-FR-13 | Deduplicação por identificador de evento | Cliente | Cada notificação carrega um identificador único que permite ao cliente detectar reenvios duplicados | `[09:24]-[09:26] Diego` |

## Requisitos não funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| PRD-NFR-01 | Latência de entrega abaixo de 10 segundos em condições normais | `[09:02] Marcos`, `[09:09]-[09:10]` |
| PRD-NFR-02 | URL do webhook deve ser HTTPS; HTTP é recusado no cadastro | `[09:23] Sofia, Larissa` |
| PRD-NFR-03 | Notificações assinadas com HMAC-SHA256 | `[09:19]-[09:20] Sofia` |
| PRD-NFR-04 | Secret exclusiva por endpoint de webhook, não global | `[09:21] Sofia` |
| PRD-NFR-05 | Rotação de secret com grace period de 24 horas | `[09:21]-[09:22] Sofia` |
| PRD-NFR-06 | Payload de notificação limitado a 64 KB; acima disso, o envio falha | `[09:23]-[09:24] Sofia, Diego, Larissa` |
| PRD-NFR-07 | Timeout de 10 segundos por chamada HTTP de entrega | `[09:42] Diego, Sofia` |
| PRD-NFR-08 | Garantia de entrega at-least-once (não exactly-once) | `[09:24]-[09:26] Diego` |
| PRD-NFR-09 | Verificação de eventos pendentes a cada 2 segundos | `[09:09]-[09:10] Diego` |
| PRD-NFR-10 | Até 5 tentativas de entrega por evento antes de considerar falha permanente | `[09:15]-[09:17] Diego, Larissa` |
| PRD-NFR-11 | Ordem de entrega garantida apenas por pedido individual, e apenas enquanto o sistema operar com um único processo de envio | `[09:12]-[09:13] Diego, Larissa` |
| PRD-NFR-12 | Solução deve usar a stack e os padrões já existentes no OMS, sem infraestrutura nova | `[09:27]-[09:30]` |
| PRD-NFR-13 | Credenciais de assinatura (secret) não devem ser expostas em logs da aplicação | Requisito de segurança consolidado a partir do incidente relatado de vazamento de secret em log de cliente (`[09:22] Diego`); ver ADR-004 |

## Decisões e trade-offs principais

- **Outbox transacional no MySQL vs. envio HTTP síncrono**: o sistema registra o evento de forma atômica com a mudança de status, em vez de chamar o cliente dentro da própria transação de pedido — [`ADR-001`](./adrs/ADR-001-outbox-transacional-no-mysql.md).
- **MySQL já existente vs. Redis Streams/mensageria externa**: optou-se por reaproveitar a infraestrutura já operada pelo time em vez de introduzir um componente novo — [`ADR-001`](./adrs/ADR-001-outbox-transacional-no-mysql.md).
- **Polling vs. mecanismos reativos de banco**: um worker consulta os eventos pendentes periodicamente, já que o banco atual não oferece um mecanismo reativo equivalente ao usado em outros bancos — [`ADR-002`](./adrs/ADR-002-worker-separado-com-polling.md).
- **At-least-once vs. exactly-once**: o cliente pode eventualmente receber uma notificação duplicada e deve tratar isso, em troca de uma solução muito mais simples de operar — [`ADR-005`](./adrs/ADR-005-entrega-at-least-once-com-event-id.md).
- **Processo único (single-worker) vs. escalabilidade horizontal**: a simplicidade operacional atual foi priorizada, já que nenhum cliente pediu garantia de ordenação entre pedidos diferentes — [`ADR-002`](./adrs/ADR-002-worker-separado-com-polling.md).
- **Secret por endpoint vs. secret global da plataforma**: cada cadastro de webhook tem sua própria credencial, limitando o impacto de um eventual vazamento a um único cliente — [`ADR-004`](./adrs/ADR-004-autenticacao-hmac-sha256.md).

## Dependências

- OMS existente (módulos de pedidos, clientes, produtos, usuários) já em produção.
- Banco de dados MySQL já utilizado pela aplicação.
- Prisma como camada de acesso a dados, já adotado no projeto.
- Autenticação JWT e os papéis (`roles`) já existentes no sistema (`ADMIN`/`OPERATOR`).
- Logger estruturado (Pino) e validação de entrada (Zod), já padronizados no OMS.
- Disponibilidade dos endpoints HTTP dos clientes — fora do controle do OMS; falhas do lado do cliente impactam diretamente a taxa de entregas bem-sucedidas.
- Participação da Engenharia de Segurança: pelo menos 2 dias úteis de revisão dedicada ao fluxo de HMAC e geração de secret antes do deploy (`[09:46] Sofia`).
- Prazo de entrega estimado em **três sprints**, incluindo a revisão de segurança (`[09:45]-[09:47] Larissa`).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Crescimento contínuo da fila de eventos pendentes ao longo do tempo | Baixa a média (avaliação deste PRD; não quantificada na reunião) | Médio — pode degradar a velocidade de entrega | Leitura em lote pequeno e indexada; arquivamento fica registrado como questão em aberto (`[09:07]-[09:08]`) |
| Indisponibilidade prolongada do endpoint de um cliente | Média (evidenciada por um caso real de 2h de indisponibilidade planejada citado na reunião) | Alto — pode atrasar a percepção da mudança de status pelo cliente | Retentativas com espera crescente cobrindo cerca de 15h antes de mover para DLQ (`[09:15]-[09:17]`) |
| Cliente recebe a mesma notificação mais de uma vez | Alta — inerente ao modelo at-least-once (avaliação deste PRD) | Baixo a médio, se o cliente não deduplicar | Identificador único por evento e orientação clara ao cliente sobre deduplicação (`[09:24]-[09:26]`) |
| Vazamento da credencial de assinatura de um cliente | Média (já houve um incidente similar relatado por um cliente) | Alto — compromete a confiança na autenticidade das notificações daquele cliente | Credencial exclusiva por endpoint e rotação com grace period de 24h (`[09:21]-[09:22]`) |
| Capacidade de envio limitada pelo processo único de entrega | Baixa nesta fase (avaliação deste PRD) | Médio no longo prazo, se o volume de pedidos crescer bastante | Aceito conscientemente nesta fase; nenhum cliente solicitou ordenação global (`[09:12]-[09:14]`) |
| Erro de configuração do endpoint pelo cliente (ex.: URL inválida) | Média (avaliação deste PRD) | Baixo — falha isolada de cadastro, não afeta outros clientes | Validação obrigatória de HTTPS no momento do cadastro (`[09:23]`) |
| Perda do cliente Atlas Comercial por atraso na entrega da feature | Alta, sinalizada explicitamente pelo próprio cliente | Alto — impacto direto de receita/relacionamento comercial | Priorização da feature com estimativa de três sprints e prazo de negócio de fim de novembro (`[09:00]`, `[09:45]-[09:47]`) |

## Critérios de aceitação

- Um webhook pode ser cadastrado por um usuário autenticado, informando URL e status desejados.
- Uma URL não-HTTPS é rejeitada no cadastro/atualização.
- A secret é gerada automaticamente pelo sistema e devolvida ao cliente apenas no momento da criação (ou rotação).
- Apenas os status efetivamente escolhidos pelo cliente geram notificação para aquele webhook.
- Uma notificação é gerada sempre que o status de um pedido muda para um status monitorado.
- Em condições normais de operação, a notificação chega ao cliente em menos de 10 segundos após a mudança de status.
- O cliente consegue validar a assinatura da notificação recebida.
- Notificações duplicadas são identificáveis pelo cliente a partir de um identificador único.
- Retentativas seguem a política de tentativas e intervalos definida para a feature.
- Falhas permanentes de entrega são registradas em uma fila de eventos não entregues (DLQ), disponível para investigação.
- O reprocessamento manual de eventos na DLQ é restrito a administradores.
- O payload de uma notificação nunca excede 64 KB.
- Os itens do pedido não fazem parte do payload da notificação.
- Nenhuma notificação é enviada de forma síncrona durante a transação de mudança de status do pedido.

## Estratégia de testes e validação

- **Validação funcional**: cobertura dos fluxos de cadastro, atualização, exclusão, filtro por status e rotação de secret.
- **Testes de aceitação**: verificação de cada critério de aceitação listado acima antes do lançamento.
- **Testes de integração**: confirmação de que a mudança de status do pedido e o registro da notificação permanecem consistentes entre si.
- **Testes de segurança**: verificação da assinatura HMAC, do isolamento entre secrets de diferentes clientes e da ausência de exposição de credenciais em logs.
- **Testes de indisponibilidade**: simulação de cliente lento/indisponível para validar o comportamento de retentativas e DLQ.
- **Testes de rotação**: validação de que ambas as secrets (atual e anterior) são aceitas durante o período de transição.
- **Testes de replay**: validação de que apenas administradores conseguem reprocessar eventos da DLQ, com registro de auditoria.
- **Validação com cliente(s) piloto**: recomenda-se validar a integração com a Atlas Comercial antes do rollout completo aos demais clientes, dado ser o cliente com o prazo mais crítico — esta é uma proposta de validação deste PRD, não uma decisão explícita da reunião.
- **Revisão de segurança**: revisão dedicada da Engenharia de Segurança antes do deploy, com pelo menos 2 dias úteis reservados (`[09:46] Sofia`).

## Questões em aberto

- Necessidade e desenho de rate limiting no envio de notificações para evitar rajadas de chamadas a um mesmo cliente — sem mitigação decidida, tratado como ponto de observação (`[09:38]-[09:39]`).
- Estratégia de arquivamento/limpeza de eventos já entregues após ~30 dias — mencionada, mas fora do escopo desta feature (`[09:08] Diego`).
- Estratégia de escalonamento para múltiplos processos de envio em paralelo, caso o volume cresça — não avaliada nesta fase (`[09:13] Diego`).
- Forma de armazenamento seguro da credencial de assinatura (ex.: criptografia adicional em repouso) — não discutida na reunião.
- Definição formal de metas operacionais (SLO) de disponibilidade e latência além do limiar de 10 segundos — não discutida.
- Contrato final (forma exata de solicitação) da rotação de secret — apenas o comportamento foi definido (`[09:21]-[09:22] Sofia`).
- Possível restrição adicional de acesso (papel/role) ao CRUD de configuração de webhook no futuro — hoje aberto a qualquer usuário autenticado, com possível revisão futura sinalizada, sem compromisso de prazo (`[09:36]-[09:37] Sofia, Marcos`).

## Rastreabilidade resumida

| Item | Fonte | Localização |
| --- | --- | --- |
| Problema de negócio (polling lento e caro) | TRANSCRICAO | `[09:00] Marcos` |
| Clientes solicitantes (Atlas, MaxDistribuição, Nova Cargo) | TRANSCRICAO | `[09:00] Marcos` |
| Latência abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| Risco de churn da Atlas | TRANSCRICAO | `[09:00]`, `[09:45] Marcos` |
| Requisitos funcionais principais (cadastro, entrega, retry, DLQ, replay, histórico) | TRANSCRICAO | `[09:14]-[09:19]`, `[09:31]-[09:37]` |
| Itens fora de escopo (e-mail, dashboard, inbound, exactly-once, múltiplos workers, arquivamento, mensageria externa) | TRANSCRICAO | `[09:02]-[09:03]`, `[09:07]`, `[09:08]`, `[09:13]`, `[09:24]-[09:26]`, `[09:37]-[09:40]` |
| Métricas de sucesso (latência) | TRANSCRICAO | `[09:02]`, `[09:09]-[09:10]` |
| Decisões técnicas principais | ADR-001 a ADR-006 | `docs/adrs/` |
| Dependências (stack atual, segurança, prazo) | TRANSCRICAO | `[09:27]-[09:30]`, `[09:45]-[09:47]` |

A cobertura completa de rastreabilidade (decisão a decisão, requisito a requisito) é mantida em `docs/TRACKER.md`.
