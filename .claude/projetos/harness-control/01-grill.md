# Harness Control (app web) — Grill & Requisitos

## Contexto (da sessão /grill-me)

### Problema

Projetos preparados pelo Agent Harness Canônico (SDD + Dev Loop) hoje só são observáveis via
terminal/arquivos — não há um lugar único pra ver fase atual, tasks, runs, gates, custos e
saúde de instalação de um projeto sem abrir o repositório e ler `.harness/`/`.claude/projetos/`
manualmente. O Harness Control existe pra ser essa camada de leitura (e, depois, execução
assistida) sobre os contratos já publicados pelo canônico — sem duplicar ou reimplementar o que
o canônico já produz.

### Usuários

MVP: uso pessoal (o próprio autor), modo individual (`127.0.0.1`, um usuário, uma raiz
autorizada). Multiusuário real (`workspace/{user}/` separado por pessoa, RBAC completo) é
extensão natural pós-MVP, já prevista no schema do banco desde a primeira migration, mas não é
o que o H0 precisa provar primeiro.

### Objetivos (MoSCoW)

**MUST:**
- PostgreSQL nativo no Linux (sem Docker), reaproveitando a instância já instalada e rodando na
  máquina — nunca uma segunda instalação duplicada.
- Auth local (email/senha) com sessão server-side (tabela `sessions`, cookie `HttpOnly`),
  bootstrap do primeiro admin via CLI.
- Modo individual funcionando de ponta a ponta primeiro: login → 1 raiz autorizada → Explorer
  read-only de 1 projeto.
- Harness Doctor integrado: `.harness/`/`schema_compatibility` obrigatórios pra abrir um
  projeto; telemetria ausente/`unavailable` vira `partial`, nunca bloqueia.
- "Preparar projeto" chama `/install-harness --json`, mostra Install Plan, exige confirmação —
  nunca sobrescreve customização automaticamente (`keep` fixo em qualquer conflito no MVP).
- Admin nunca tem escrita implícita em `workspace/prd/` — leitura por padrão, escrita exige
  fluxo de confirmação explícito adicional (herda o invariante já declarado em
  `rules/seguranca.md`).

**SHOULD:**
- Multiusuário real (múltiplas raízes, múltiplos usuários, RBAC completo) como extensão
  imediatamente pós-MVP, sem exigir migration destrutiva do schema já desenhado multiusuário.
- Dashboard mostrando a matriz de cobertura de observabilidade (o que está instrumentado vs.
  não) em vez de fingir 100% de cobertura.

**COULD:**
- OIDC/SSO (adiado pra H5 — administração avançada).
- UI completa de resolução de conflito arquivo-por-arquivo no "Preparar projeto" (adiado —
  hoje só `keep` automático).
- Exclusão de conta/dados de usuário (LGPD "direito ao esquecimento") — pendência conhecida,
  não bloqueia o MVP.

### Out of Scope

- Docker para o PostgreSQL — instalação nativa via `apt`/gerenciador de pacote do Linux, já
  presente na máquina de desenvolvimento (Postgres 16, cluster `16-main` ativo via systemd).
- OIDC/SSO/LDAP no MVP — só auth local.
- Multiusuário real no primeiro corte — schema já preparado, mas UI/fluxo de múltiplas raízes
  fica pra depois do núcleo (auth + sessão + Explorer) provado.
- Execução assistida de agente (HITL/AFK, adapters Claude Code/Codex rodando de verdade),
  telemetria/custos via OpenTelemetry, grafo configurado/executado — tudo isso é H3/H4 do
  roadmap original, não H0.
- UI de resolução de conflito de instalação arquivo-por-arquivo.
- Exclusão de conta/retenção LGPD formal (pendência documentada, não implementada no MVP).

### Restrições

- **Banco:** PostgreSQL 16, nativo, já instalado e rodando (systemd, `postgresql@16-main`),
  reaproveitado — nunca segunda instalação. Conexão via TCP `localhost:5432` (não socket Unix),
  credencial em `.env`/variável de ambiente, nunca hardcoded.
- **Setup de infra é manual e documentado**, não automatizado: instalação do Postgres (já
  feita, mas documentada no README "pra constar"), criação de `role`/`database` via bloco
  `psql` no README, bootstrap do primeiro admin via CLI (`create-admin`) — sem script de setup
  automatizado no MVP.
- **ORM/migrations:** SQLAlchemy + Alembic — combinação madura pra FastAPI, schema relacional
  com FKs (`project_memberships`, `project_leases`, etc.).
- **Backend:** FastAPI (já definido no plano original). Frontend: Next.js/React (já definido,
  não revisitado nesta sessão de grill — H1 em diante).
- **Sessão:** server-side, tabela `sessions`, cookie `HttpOnly`/`Secure` (modo compartilhado)/
  `SameSite=Lax` — revogação imediata via `DELETE`/`UPDATE`, não JWT stateless.
- **Sem SLA formal** no MVP — modo individual local, sem carga de múltiplos usuários ainda.

## Especificações implícitas detectadas

**1 — Falha de dependência externa:** já respondida pelo plano original (seção 11): "read-only
quando o banco estiver indisponível" — o Control degrada pra leitura de arquivos quando o
Postgres cai, não quebra totalmente.

**2 — Concorrência:** já respondida pelo plano original (seção 6): lease exclusivo por
`project_id` com TTL/heartbeat, `.harness/locks/project.lock` na escrita, dois Controls nunca
executam o mesmo projeto simultaneamente.

**3 — Idempotência:** N/A no MVP — o CLI de bootstrap do admin roda uma vez manualmente (não é
um fluxo repetido/retry automático); leases já cobrem idempotência de execução (lente 2). Se
surgir necessidade de reprocessamento automático (ex.: reindexação), reavaliar então.

**4 — Autenticação/Autorização:** respondida extensivamente nesta sessão — auth local, sessão
server-side, RBAC (`user`/`operator`/`admin`), admin sem escrita implícita em produção.

**5 — Dados sensíveis não declarados:** coberto pelo SI Assessment nível c + `rules/pii.md` já
criados (email, `sessions`, `audit_events`, `provider_accounts`).

**6 — Abuso e limites:** N/A no MVP — modo individual, um usuário, sem exposição pública ainda;
rate limiting/quota por usuário só faz sentido a partir do modo compartilhado (H0 estendido ou
H1+), registrar como candidato quando o multiusuário real entrar em escopo.

**7 — Auditoria:** já respondida pelo plano original — `audit_events` rastreia login, seleção,
lease, escrita, execução, migration.

**8 — Estados de erro visíveis:** parcialmente respondida (Postgres indisponível → read-only,
seção 11) — comportamento de erro do Harness Doctor (`blocked`/`partial`) e do Install Plan
("Preparar projeto") já tem vocabulário definido no plano original (seção 5.3), suficiente pro
MVP.

**9 — Ciclo de vida dos dados:** exclusão de conta/"direito ao esquecimento" fica **fora do
MVP**, documentado como pendência conhecida (decisão desta sessão) — não é omissão silenciosa,
é adiamento explícito pra H5.

**10 — Dependência de terceiros mudando:** o contrato `ExecutorAdapter` (detect/start/
stream_events/approve/pause/resume/cancel/collect_usage/finalize) já isola o Control de mudança
de contrato do Claude Code/Codex CLI — decisão já tomada no plano original, não revisitada aqui
(H3/H4, fora do MVP).

## Critérios de sucesso mensuráveis

- Login local funciona de ponta a ponta (criar admin via CLI → logar → sessão persistida na
  tabela `sessions` → logout revoga imediatamente).
- Um projeto real (`agent-harness-canonico` ou `harness-control-app`, ambos já instalados via
  `/install-harness`) abre no Explorer read-only e mostra fase SDD atual, tasks, runs recentes.
- Harness Doctor reporta corretamente `ok`/`partial`/`missing`/`blocked` para pelo menos um
  projeto com telemetria `unavailable` (ex.: `agent-harness-canonico`, `telemetry.cursor:
  "unavailable"` hoje) sem bloquear a abertura.
- `/install-harness --json` disparado via "Preparar projeto" mostra Install Plan e só aplica
  após confirmação — nenhuma escrita sem clique explícito.
- README documenta o setup completo (instalação do Postgres nativo + criação de role/database +
  bootstrap do admin) de forma que outra pessoa consiga reproduzir do zero numa máquina Linux
  limpa.

## Próximo passo
Rode `harness-design` para gerar o PRD e montar o `.claude/` do projeto — aqui,
`/harness-architect` **se aplica de verdade** (é um projeto-alvo real recebendo customização de
harness, diferente da evolução do canônico em si).
