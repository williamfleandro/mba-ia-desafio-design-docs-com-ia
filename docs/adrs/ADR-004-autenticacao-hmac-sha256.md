# ADR-004: Autenticação de payload via HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceita — decidida em reunião técnica; implementação ainda não iniciada (ver "Relação com o sistema existente"). Sofia (Engenharia de Segurança) reservou pelo menos 2 dias úteis para revisar especificamente o código de HMAC e geração de secret antes do deploy (`[09:46] Sofia`).

## Data

Reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`; a transcrição não informa a data absoluta, apenas o dia da semana e o horário).

## Decisores

- Sofia (Engenheira de Segurança) — propõe HMAC-SHA256, secret por endpoint e rotação com grace period
- Diego (Engenheiro Sênior, time de Plataforma) — confirma com base em incidente real de vazamento

## Fontes

- `TRANSCRICAO.md`, `[09:19]-[09:22]`
- `reports/context-analysis.md`, decisões DEC-011, DEC-012, DEC-013; risco RISK-003; NFR-003, NFR-004; questão em aberto OPEN-002; nota de código U.8 (lista de redact do logger)
- Código: `src/shared/logger/index.ts:4-32` (lista de campos redigidos), `src/middlewares/validate.middleware.ts` (validação Zod)

## Contexto

Como os webhooks são exclusivamente outbound (sistema → cliente), Sofia levantou a necessidade de o cliente conseguir validar que a requisição realmente veio do sistema e que o payload não foi adulterado em trânsito (`[09:19] Sofia`). Diego reforçou a urgência do tema citando um incidente real: um cliente já vazou uma secret em log de aplicação (`[09:22] Diego`).

## Decisão

- Assinar o corpo (payload) de cada requisição de webhook com **HMAC-SHA256**, enviando a assinatura no header `X-Signature`; o cliente verifica a assinatura do lado dele (`[09:19]-[09:20] Sofia`). SHA-256 foi escolhido por ser padrão de mercado, com suporte amplo em bibliotecas cliente.
- Cada **endpoint de webhook cadastrado tem uma secret única** — não existe secret global da plataforma (`[09:21] Sofia`). A tabela de configuração de webhook armazena `url`, `secret`, `customer_id` e estado ativo.
- A secret é **rotacionável via API**. Ao rotacionar, a secret antiga permanece válida em paralelo por **24 horas (grace period)**, dando tempo ao cliente de migrar seus sistemas; depois desse período, a secret antiga é invalidada (`[09:21]-[09:22] Sofia`).

Nota explícita: a exigência de TLS obrigatório (URL do webhook deve ser `https`) foi discutida na mesma janela da reunião, mas foi classificada por Larissa como validação de schema (Zod), não como decisão arquitetural própria (`[09:23] Sofia, Larissa`) — portanto **não faz parte da decisão registrada por este ADR** e deve ser tratada como requisito não funcional na validação de entrada do módulo webhooks.

## Alternativas consideradas

- **Secret global compartilhada por toda a plataforma** (rejeitada implicitamente, `[09:21] Sofia`): descartada porque, se essa secret única vazasse, todos os clientes ficariam comprometidos simultaneamente ("Senão se vaza uma, vaza tudo").
- **Invalidação imediata da secret antiga na rotação** (rejeitada implicitamente, motivada pelo incidente citado por Diego): a equipe optou por um grace period de 24h em vez de invalidação imediata, para permitir migração do cliente sem quebrar a integração em produção.

## Consequências positivas

- Cliente consegue verificar autenticidade e integridade do payload recebido.
- Blast radius de uma secret vazada é limitado a um único endpoint/cliente, não à plataforma inteira.
- Rotação com grace period permite recuperação de um incidente de vazamento sem downtime da integração do cliente.

## Consequências negativas

- Exige gestão de armazenamento seguro de múltiplas secrets (potencialmente duas por endpoint, durante o grace period de rotação).
- Adiciona complexidade operacional: endpoint de rotação, controle de expiração da secret antiga, e necessidade de o cliente implementar verificação HMAC do lado dele.
- Observação de código (não decisão de reunião): a lista de campos redigidos do logger Pino atual (`src/shared/logger/index.ts:4-11`) inclui `password`, `passwordHash`, `token`, `accessToken` e headers de autenticação/cookie, mas **não inclui `secret`** explicitamente — ponto de atenção ao integrar segredos de webhook em logs, dado o próprio incidente de vazamento citado por Diego.

## Trade-offs

- **Isolamento por endpoint vs. mais credenciais para gerenciar**: uma secret por endpoint reduz o impacto de um vazamento individual, mas multiplica o número de segredos que o sistema precisa armazenar e rotacionar com segurança.
- **Continuidade da integração vs. janela de exposição**: o grace period de 24h evita quebrar a integração do cliente durante a rotação, mas mantém duas secrets simultaneamente válidas por esse período.

## Relação com o sistema existente

- Não há hoje nenhum modelo Prisma, endpoint ou lógica de geração/verificação de secret no repositório — todos os elementos desta decisão (tabela de configuração de webhook, geração de secret, endpoint de rotação) são propostos.
- A validação de URL `https`-only deve reaproveitar o middleware `validate` (Zod) já existente no projeto, e não uma nova camada de validação.
- O contrato exato (método HTTP e path) do endpoint de rotação de secret não foi especificado na reunião — permanece como questão em aberto (`OPEN-002` em `reports/context-analysis.md`).
- Ao implementar logging relacionado a secrets/rotação, revisar a lista de redact em `src/shared/logger/index.ts` para evitar reexposição do vazamento já relatado pelo cliente.

## Rastreabilidade

| Origem | IDs |
| --- | --- |
| Decisões (transcrição) | DEC-011, DEC-012, DEC-013 |
| Decisão excluída deste ADR (classificada como NFR) | DEC-014 (TLS obrigatório) |
| Risco mitigado | RISK-003 |
| Requisitos relacionados | NFR-003, NFR-004 |
| Questão em aberto | OPEN-002 (contrato do endpoint de rotação) |
| Observação de código | Lista de redact do logger (`src/shared/logger/index.ts:4-11`) não inclui `secret` |
| Timestamps-chave | `[09:19]-[09:22]` |
