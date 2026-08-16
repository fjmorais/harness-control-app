# Harness Control (app web)

## Ideia

> Fonte: `sketch/harness-control/plan.md` (copiado do canônico `agent-harness-canonico`, que
> originou o plano). Este repositório (`harness-control-app`) é o "futuro repositório
> `/home/fabiano/harness-control`" citado no plano — o app web em si, separado do canônico por
> decisão explícita (invariante "nenhum código de produto no canônico").

App web local/compartilhado para usuários autenticados selecionarem seus projetos Harness
(instalados via `/install-harness` do canônico), visualizarem SDD e Dev Loop, acompanharem
agentes, consultarem logs/tokens/custos/gates/indicadores e, depois, executarem tarefas com
aprovação e isolamento.

**O Control não é dono dos artefatos do projeto.** Lê os contratos publicados pelo canônico
(`.harness/`, `.claude/`, `.cursor/`, `metrics/`, `tasks/`, `docs/adr/`, Git) e usa o
`/install-harness` (modo `--json`) para qualquer instalação/atualização — nunca copia
agents/skills/hooks por conta própria.

### Áreas cobertas pelo plano

- **Multiusuário com filesystem separado por raiz autorizada** (`workspace/{user}/`,
  `workspace/prd/` para produção) — backend resolve `user_id` → raízes registradas no
  PostgreSQL, valida cada path (canonicalização, symlink, path traversal).
- **PostgreSQL como fonte de verdade de identidade/autorização/ownership/leases/catálogo** —
  não duplica código, PRDs, eventos completos, prompts ou logs grandes (isso fica nos arquivos
  do projeto, fonte de verdade de documentos).
- **RBAC** (`user`/`operator`/`admin`), tenants, `project_memberships`, auditoria de leitura
  cross-tenant e ações administrativas.
- **Seleção e validação de pasta** — árvore construída pelo backend só com diretórios
  autorizados; perfil mínimo de projeto válido (Git, manifest, `.harness/` ou estrutura
  legada); path enviado pela UI nunca é autorização, sempre revalidado no backend.
- **Harness Doctor via Control** — verificação read-only (`ok`/`missing`/`outdated`/
  `customized`/`blocked`/`partial`) antes de abrir um projeto; ação "Preparar projeto" chama
  `/install-harness --json`, mostra Install Plan, exige confirmação, nunca instala sozinho.
- **Ownership, leases e escrita segura** — lease exclusivo por `project_id` com TTL/heartbeat,
  `.harness/locks/project.lock` na escrita, dois Controls nunca executam o mesmo projeto
  simultaneamente, watcher + fingerprint para mudança externa.
- **Experiência web** — modo individual (`127.0.0.1`) e compartilhado (HTTPS/reverse proxy +
  auth); portfólio de projetos; SDD (Ideia→Grill→PRD→Harness→Tasks→Build→Ship) e Dev Loop
  (LOAD→VALIDATE→PICK→EXECUTE→VERIFY→UPDATE→CHECK→LOOP→LOG) navegáveis; grafo **configurado**
  (parsing estático) vs. **executado** (telemetria real) diferenciados; dashboards de operação,
  engenharia, LLM/custos e admin/segurança.
- **Execução de agentes** — contrato `ExecutorAdapter` (detect/start/stream_events/approve/
  pause/resume/cancel/collect_usage/finalize), adapters iniciais Claude Code e Codex, modos
  dry-run/HITL/AFK/resume, nunca ultrapassa root/policy/budget/tokens/iterações/timeout.
- **Tokens e custos** — separa uso técnico, custo estimado (tabela versionada) e custo faturado
  pelo provider; USD/BRL; assinaturas não são custo marginal por token.
- **Arquitetura**: Next.js/React (Web UI) → FastAPI (API/Backend: Auth/RBAC/Tenant Resolver,
  Workspace Scanner/Validator, Project Catalog, Lease/Lock Manager, Canonical Contract Reader,
  Index Builder, Telemetry Ingest/Cost Engine, Workflow/Approval Manager, Claude/Codex
  Adapters) → PostgreSQL (control plane) + workspace (arquivos) + OpenTelemetry Collector +
  worktrees/sandbox.
- **Segurança**: auth obrigatória no compartilhado, RBAC/tenant no backend, CORS fechado,
  proteção DNS rebinding, nenhum mutating endpoint via GET, path traversal/symlink escape
  bloqueados, PII opt-in/redigida, secrets fora dos logs, execução em worktree/sandbox,
  auditoria completa, backup/restore testado, read-only quando o banco estiver indisponível.
- **Roadmap em 6 fases**: H0 (fundação/onboarding) → H1 (Explorer read-only) → H2
  (atualização/diagramas) → H3 (telemetria/custos) → H4 (execução assistida) → H5
  (administração avançada).
- **13 critérios de aceitação do MVP** e **13 decisões em aberto** já registrados no plano
  original (ver `sketch/harness-control/plan.md` seções 13-14) — cobrem desde isolamento
  multi-tenant até qual biblioteca de grafo usar.

### Direção visual

Tema claro/escuro persistente por usuário. Referências oficiais de layout em
`sketch/harness-control/references/` (board principal, observabilidade, workflow graph) — usadas
como guia de linguagem visual, não como fonte de dados ou contrato de texto.

## SI Assessment

Nível: **c) Sim — PII** (email de login, `users`/`audit_events` no PostgreSQL, RBAC
multi-tenant com trilha de ação por pessoa).

Confirmado pelo usuário. Racional: diferente da evolução do canônico (nível b, sem contas de
usuário reais), este é um app com **autenticação de usuários de verdade** — login, sessões,
auditoria por `user_id`. Email e identificação de pessoa natural são PII segundo
`rules/seguranca.md`. O próprio plano (seção 11) já antecipa isso: "prompts, tool details e PII
opt-in/redigidos", auditoria de login/seleção/lease/escrita/execução/migration.

Implicações para o harness deste projeto:
- Declarar `[AVISO_LGPD]` no `CLAUDE.md` do projeto.
- Criar `rules/pii.md` com regras específicas de mascaramento (email nunca em log, PII exibida
  só via função de mascaramento).
- Todo ADR que envolva acesso a dados de usuário (tabela `users`, `sessions`, `audit_events`)
  precisa de seção "Impacto em SI/LGPD".
- Produção (`workspace/prd/`) já é tratada no plano como raiz administrativa explícita, não
  implícita — mas qualquer escrita real em produção (seção 6, leases/locks) deve respeitar o
  invariante "somente leitura por padrão" até haver ADR explícito autorizando escrita.

## Tipo de Projeto

**a) Aplicação / API / Chatbot / Agente de IA.**

Stack prevista: Next.js/React (frontend), FastAPI (backend), PostgreSQL (control plane),
OpenTelemetry (telemetria), adapters Claude Code/Codex (execução de agente). Multiusuário,
multi-tenant, com autenticação obrigatória no modo compartilhado.

## Relação com o canônico

Este repositório (`harness-control-app`) foi criado e instalado via `/install-harness`
apontando para `/home/fabiano/agent-harness-canonico` (manifest já presente em
`.claude/harness-manifest.json`, `mode: "NOVO"`, `installed_at: "2026-08-16"`). O Control
consome os contratos publicados pelo canônico (schemas, `.harness/`, ADRs) — não duplica nem
reimplementa esses conceitos aqui.

## Próximos passos

1. Rode `/grill-me` para aprofundar a ideia — atenção especial aos 13 pontos em aberto do plano
   original (seção 14: OIDC/SSO vs. usuários locais, escrita de admin em `workspace/prd/`,
   deployment do PostgreSQL, política de backup, perfil exato de "projeto válido", biblioteca
   de grafo, diagnóstico sem PostgreSQL, múltiplas worktrees, providers suportados, etc.).
2. Ao terminar, use `harness-define` para estruturar os requisitos.
3. Depois: `harness-design` → PRD + `harness-architect` (aqui, ao contrário do projeto do
   canônico, `/harness-architect` **se aplica de verdade** — este é um projeto-alvo real
   recebendo harness, não o próprio framework evoluindo).
