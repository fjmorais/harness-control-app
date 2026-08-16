# Plano — Harness Control

> Plano do futuro repositório `/home/fabiano/harness-control`. Define somente o aplicativo web que
> lê e opera projetos preparados pelo Agent Harness Canônico.

## 1. Objetivo

Criar um app web local/compartilhado para usuários autenticados selecionarem seus projetos Harness,
visualizarem SDD e Dev Loop, acompanharem agentes, consultarem logs, tokens, custos, gates e
indicadores e, depois, executarem tarefas com aprovação e isolamento.

O Control não é dono dos artefatos do projeto. Lê os contratos instalados pelo canônico e usa o
`/install-harness` para qualquer instalação ou atualização.

## 1.1 Direção visual

O produto terá tema claro e tema escuro, escolhidos por um controle persistente no topo da
interface. A preferência será salva por usuário, com opção de seguir a preferência do sistema.

### Referências visuais oficiais

As imagens abaixo são os templates visuais oficiais para orientar o design e a implementação do
Harness Control:

```text
sketch/harness-control/references/harness-control-board-v2-mixed-themes.png
sketch/harness-control/references/harness-control-observability.png
sketch/harness-control/references/harness-control-workflow-graph.png
```

Uso de cada referência:

| Referência | Papel no produto |
|---|---|
| `harness-control-board-v2-mixed-themes.png` | Template principal: login, projetos autorizados, dashboard, SDD, Dev Loop, grafo e administração |
| `harness-control-observability.png` | Referência para execução ao vivo, tokens, custos, drift e auditoria |
| `harness-control-workflow-graph.png` | Referência para Workflow SDD, terminal integrado, agentes, subagentes, gates e artefatos |

Novas telas devem manter, salvo decisão registrada em ADR:

- a navegação lateral e a hierarquia de informação;
- o controle de tema claro/escuro;
- a linguagem de cards, tabelas, timelines, terminais e grafos;
- a distinção visual entre configurado, executado, pendente, aprovado, bloqueado e concluído;
- o padrão de cores, espaçamento, densidade e contraste;
- a relação entre projeto, tenant, ambiente, run, agente, artefato e custo.

Essas imagens são referências de layout e linguagem visual, não fontes de dados nem contratos de
texto. O comportamento real deve obedecer aos schemas, policies e requisitos deste plano.

Direção aprovada para as primeiras referências:

- login: preferencialmente light, com indicação de deployment seguro;
- projetos autorizados: navegação escura com conteúdo híbrido e badges de permissão;
- dashboard: light, orientado a leitura executiva;
- Workflow SDD: dark, com terminal e visualizações em tempo real;
- Dev Loop: dark, com terminal, eventos, agentes e aprovações;
- grafo configurado/executado: dark, com arestas e proveniência distintas;
- administração: light, com tabelas, usuários, roots, PostgreSQL, leases e auditoria.

O tema não muda autorização, dados ou comportamento. Todos os componentes devem ter equivalência
funcional nos dois temas e contraste acessível.

## 2. Deployment e filesystem

O app pode rodar em uma máquina local ou servidor da organização. No modo multiusuário, o filesystem
de projetos fica separado do filesystem da aplicação:

```text
/srv/harness-workspace/
└── workspace/
    ├── fabiano/
    │   ├── projeto-a/
    │   └── projeto-b/
    ├── maria/
    │   └── projeto-c/
    └── prd/
        ├── produto-x/
        └── produto-y/
```

Regras:

| Usuário | Raiz autorizada | Acesso |
|---|---|---|
| Fabiano | `workspace/fabiano/` | Projetos de Fabiano |
| Maria | `workspace/maria/` | Projetos de Maria |
| Admin | `workspace/prd/` e raízes atribuídas | Projetos de produção autorizados |

O usuário não escolhe uma raiz arbitrária. O backend resolve o `user_id` autenticado para raízes
registradas no PostgreSQL e valida cada path após resolver caminho absoluto e symlinks.

## 3. PostgreSQL do Control

Como o app será multiusuário, PostgreSQL é recomendado mesmo que logs detalhados permaneçam em
arquivos. Uma instância simples via Docker Compose ou serviço do sistema é suficiente.

### 3.1 Dados no banco

```text
users
roles
tenants
workspace_roots
projects
project_memberships
project_leases
sessions
approval_requests
audit_events
provider_accounts
pricing_versions
index_state
```

O PostgreSQL é a fonte de verdade de identidade, autorização, ownership, leases, catálogo e
sessões. Não armazena, por padrão, código, PRDs inteiros, eventos completos, prompts ou logs
grandes.

### 3.2 Dados nos projetos

O Control consome `.harness/`, `.claude/`, `.cursor/`, `metrics/`, `tasks/`, `docs/adr/` e Git. Os
arquivos são a fonte de verdade de documentos e eventos detalhados. Índices no PostgreSQL são
derivados e podem ser reconstruídos.

## 4. Login, tenants e autorização

Papéis mínimos:

```text
user      lê e opera seus projetos autorizados
operator  executa ações adicionais nas raízes atribuídas
admin     administra usuários, raízes, policies e produção autorizada
```

O admin não tem acesso implícito a todos os homes; `workspace/prd/` é uma raiz administrativa
explícita. Leituras cross-tenant e ações administrativas são auditadas.

Cada projeto terá:

```text
project_id
tenant_id
owner_user_id
canonical_path
path_fingerprint
access_policy
status
```

O frontend nunca é autoridade de autorização. Cada endpoint verifica sessão, papel, tenant, raiz,
ownership e policy no backend.

## 5. Seleção e validação de pastas

### 5.1 Seleção operacional

O usuário vê uma árvore construída pelo backend contendo somente diretórios dentro das raízes
permitidas. O sistema não varre nem expõe o filesystem inteiro.

```text
login
  -> consultar raízes permitidas
  -> listar candidatos diretos
  -> validar estrutura
  -> mostrar válidos e diagnósticos
  -> cadastrar projeto selecionado
```

Em deployment compartilhado, um path absoluto enviado pela UI nunca é autorização. O backend o
valida novamente. Um seletor nativo pode ser usado em instalação desktop, mas não substitui tenant.

### 5.2 Perfil mínimo de projeto válido

Uma pasta só pode ser cadastrada se:

- estiver dentro da raiz autorizada;
- não escapar por symlink, path traversal ou mount não permitido;
- tiver Git ou marcador de projeto suportado;
- tiver `.claude/harness-manifest.json` ou estrutura legada reconhecida;
- tiver `.harness/` ou documentos como `.claude/projetos/`, `.claude/dev/` ou `metrics/`;
- estiver legível pelo serviço;
- não estiver reservada, bloqueada ou atribuída a outro tenant.

Pastas inválidas podem ser exibidas para diagnóstico, mas não terão ação de cadastro/abertura
operacional:

```text
Esta pasta não é um projeto Harness válido.
Faltando: .claude/harness-manifest.json ou estrutura legada reconhecida.
Sugestão: selecione a subpasta do projeto, não a pasta workspace.
```

### 5.3 Onboarding e Harness Doctor

Depois que o usuário aponta para uma pasta autorizada, o Control executa uma verificação somente
leitura antes de abrir o projeto:

```text
1. validar path, tenant e permissão
2. detectar Git e stack
3. localizar manifest e versão do harness
4. validar .claude/, .cursor/, .harness/, SDD e Dev Loop
5. validar agents, subagents, skills, commands, gates e hooks
6. validar emissão de eventos, usage e custos
7. apresentar diagnóstico e nível de cobertura
```

O resultado deve separar:

| Estado | Significado |
|---|---|
| `ok` | capability instalada e operacional |
| `missing` | capability esperada não encontrada |
| `outdated` | existe, mas está em versão incompatível |
| `customized` | existe e foi protegida pelo projeto |
| `blocked` | não pode ser instalada por policy/permissão |
| `partial` | funciona, mas não produz todos os indicadores |

Se houver itens ausentes, o Control oferece **Preparar projeto**. Essa ação deve:

1. chamar o `/install-harness` do canônico em modo `--json`;
2. mostrar Install Plan, conflitos, migrations e capabilities que serão configuradas;
3. informar que a operação pode alterar arquivos do projeto;
4. exigir confirmação do usuário com permissão;
5. aplicar somente após confirmação;
6. executar o Harness Doctor novamente;
7. abrir o projeto apenas quando os requisitos mínimos estiverem `ok` ou explicitamente `partial`.

O Control nunca deve copiar agents, skills, workflows ou hooks por conta própria. A aplicação deve
orquestrar o instalador oficial e verificar o resultado.

### 5.4 Cobertura de observabilidade

O diagnóstico deve mostrar uma matriz semelhante a:

| Capability | Instalada | Produz evento | Produz usage | Fonte |
|---|---:|---:|---:|---|
| SDD workflow | sim | sim | n/a | `.claude/projetos/` |
| Dev Loop | sim | sim | sim | `.harness/runs/` |
| Agents/subagents | sim | parcial | sim | adapter |
| Skills/commands | sim | sim | parcial | hooks |
| Gates/reviews | sim | sim | n/a | `result.json` |
| Tokens/custos | parcial | sim | não | executor sem usage |

O dashboard deve exibir a cobertura e não fingir que um projeto está completamente instrumentado.

## 6. Ownership, leases e escrita segura

Mesmo com separação por usuário, a aplicação deve impedir concorrência acidental:

1. PostgreSQL mantém lease exclusivo por `project_id` para escrita/execução;
2. lease tem TTL e heartbeat;
3. leitura pode ser concorrente quando a policy permitir;
4. outra sessão só assume após expiração e confirmação;
5. backend usa `.harness/locks/project.lock` em escrita;
6. arquivos são validados e escritos atomicamente;
7. dois Controls não executam o mesmo projeto simultaneamente;
8. mudanças externas são detectadas por watcher e fingerprint.

Ordem obrigatória: autenticar, autorizar, reservar lease e só então escrever/executar.

## 7. Experiência web

### 7.1 Execução local e compartilhada

Modo individual:

```text
harness-control start -> http://127.0.0.1:4173
```

Modo compartilhado:

```text
usuário -> HTTPS/reverse proxy -> Harness Control -> PostgreSQL + workspace
```

Requisitos:

- porta configurável;
- bind `127.0.0.1` no modo individual;
- reverse proxy/TLS e autenticação no modo compartilhado;
- health/readiness endpoints;
- logs do próprio Control;
- shutdown limpo;
- não abrir navegador automaticamente no servidor compartilhado.

### 7.2 Portfólio e projeto

Mostrar somente projetos autorizados:

- fase atual;
- tasks completas, abertas e bloqueadas;
- runs recentes e ativas;
- gates, reviews e intervenções;
- tokens, custos e budget;
- versão e compatibilidade;
- drift e customizações;
- atividade Git;
- status do lease.

### 7.3 SDD e Dev Loop

```text
Ideia -> Grill -> PRD -> Harness -> Tasks -> Build -> Ship
```

```text
LOAD -> VALIDATE -> PICK -> EXECUTE -> VERIFY -> UPDATE -> CHECK -> LOOP -> LOG
```

Abrir documentos, tarefas, ADRs, critérios, evidências, decisões e logs. Editar somente quando a
policy permitir e com controle de conflito.

O Workflow SDD poderá exibir, na mesma tela, o terminal da execução atual, o grafo de agentes e
os indicadores de progresso. O terminal é uma visualização controlada do processo; ações continuam
sujeitas a policy, approval, lease e auditoria.

### 7.4 Diagramas

O Control terá duas fontes distintas:

1. **grafo configurado:** relações em commands, agents, skills, KBs e rules;
2. **grafo executado:** relações comprovadas pelos eventos de uma run.

```text
Usuário
  └── Dev Loop
       ├── prompt-crafter
       ├── dev-loop-executor
       │    ├── tool: edit
       │    ├── tool: pytest
       │    └── revisor-codigo
       └── gate/result
```

Nós exibem status, duração, tokens, custo, tentativas, arquivos e evidências. Relações de parsing
são `configured`; telemetria é `executed`. O grafo suporta subagentes, parent/child runs, filtros,
timeline, zoom/pan e atualização por SSE.

### 7.5 Dashboards

Operação: runs, filas, duração, bloqueios, approvals, gates, erros e circuit breakers.

Engenharia: critérios atendidos, tentativas até verde, review, lead time, throughput, autonomia,
intervenções, change-failure e reaberturas.

LLM/custos: tokens por provider/modelo/workflow/task/agent/skill, input/output/cache/reasoning,
custo por run/task/entrega, budget, previsão e anomalias.

Admin/segurança: logins, ações permitidas/negadas, leases, ferramentas, MCP, rede, migrations,
acessos cross-tenant e saúde da telemetria.

## 8. Execução de agentes

Contrato de adapters:

```text
ExecutorAdapter
├── detect()
├── start(run_config)
├── stream_events()
├── approve(action)
├── pause()
├── resume()
├── cancel()
├── collect_usage()
└── finalize()
```

Adapters iniciais: Claude Code e Codex. Controlar PTY/stdout/stderr, exit code, timeout, filhos,
cancelamento, orphan process, environment, worktree, approval e recovery.

Modos: dry-run, HITL, AFK e resume. Nenhuma execução ultrapassa root, policy, budget, tokens,
iterações ou timeout.

## 9. Tokens e custos

Separar:

1. uso técnico;
2. custo estimado por tabela versionada;
3. custo faturado pelo provider.

Guardar provider, modelo, tokens, preço, fonte, moeda e confiança. Suportar USD, BRL e conversão
opcional. Assinaturas Claude/ChatGPT não são custo marginal por token.

## 10. Arquitetura

```text
Browser
  ▼
Web UI (Next.js/React)
  ▼
API/Backend (FastAPI)
  ├── Auth/RBAC/Tenant Resolver
  ├── Workspace Scanner/Validator
  ├── Project Catalog
  ├── Lease/Lock Manager
  ├── Canonical Contract Reader
  ├── Index Builder
  ├── Telemetry Ingest/Cost Engine
  ├── Workflow/Approval Manager
  └── Claude/Codex Adapters
       ├── PostgreSQL: control plane
       ├── workspace: arquivos do projeto
       ├── OpenTelemetry Collector
       └── worktrees/sandbox
```

PostgreSQL é obrigatório no deployment multiusuário e opcional apenas no modo individual de
diagnóstico. SQLite pode ser índice derivado, nunca fonte de verdade.

## 11. Segurança

- autenticação obrigatória no compartilhado;
- RBAC e tenant resolvidos no backend;
- CORS fechado;
- validação de `Origin`/`Host` e proteção contra DNS rebinding;
- nenhum mutating endpoint via GET;
- canonicalização, allowlist, path traversal e symlink escape bloqueados;
- processo com menor privilégio possível;
- prompts, tool details e PII opt-in/redigidos;
- secrets fora dos logs;
- execução em worktree/sandbox;
- auditoria de login, seleção, lease, escrita, execução e migration;
- backup/restore do PostgreSQL testados;
- read-only quando o banco estiver indisponível.

## 12. Roadmap

### H0 — fundação e onboarding

PostgreSQL, login, tenants, roots, RBAC, scanner, validação, seleção autorizada, ownership, leases
e Harness Doctor.

### H1 — Explorer read-only

Um projeto selecionado, leitura dos contratos do canônico, dashboard, Markdown, tasks, Dev Loop,
Git, métricas, catálogo e diagnóstico de incompatibilidade. O usuário pode gerar um Install Plan,
mas o Explorer continua sem escrita ou execução automática.

### H2 — atualização e diagramas

File watcher, índices PostgreSQL, SSE, grafo configurado, timeline e eventos em tempo real.

### H3 — telemetria e custos

OpenTelemetry, adapters Claude/Codex, usage, pricing, ledger, custos e grafo executado.

### H4 — execução assistida

HITL, AFK, resume, dry-run, worktrees, approval, timeout, cancelamento, gates, reviews e
install/update via `/install-harness --json`, sempre precedido pelo Harness Doctor.

### H5 — administração avançada

Budgets, alertas, export/import, auditoria avançada, relatórios, OIDC/SSO e operação multi-instância.

## 13. Critérios de aceitação do MVP

- usuário autenticado só lista projetos de suas raízes;
- Fabiano não descobre nem abre `workspace/maria/`;
- admin acessa somente raízes administrativas configuradas;
- pasta sem estrutura mínima é recusada com diagnóstico;
- projeto com estrutura incompleta recebe diagnóstico de capabilities faltantes;
- Control consegue gerar Install Plan do canônico para preparar o projeto;
- instalação só ocorre após confirmação e através do `/install-harness`;
- após instalação, o Doctor confirma SDD, Dev Loop, agents, subagents, skills, hooks, eventos e usage;
- projeto válido abre em read-only;
- dois usuários não reservam o mesmo projeto para escrita;
- lease expirado é recuperável com auditoria;
- reinício do PostgreSQL não corrompe arquivos;
- Control lê projeto instalado pelo canônico sem parsing proprietário;
- dashboard mostra fase, tasks, entregas, gates e logs;
- nenhuma ação mutável ocorre sem policy/confirmação;
- individual roda em `127.0.0.1`, compartilhado exige autenticação/TLS;
- detalhes permanecem em arquivos e não são duplicados integralmente no banco.

## 14. Decisões em aberto

1. O deployment compartilhado usará OIDC/SSO, LDAP ou usuários locais?
2. Admin terá escrita em `workspace/prd/` ou somente leitura?
3. PostgreSQL será Docker Compose, pacote do sistema ou serviço gerenciado?
4. Qual política de backup e retenção do banco?
5. Qual perfil exato define projeto válido?
6. Qual biblioteca de grafo será usada?
7. O modo individual permitirá diagnóstico sem PostgreSQL?
8. Como serão solicitados acessos excepcionais?
9. Como tratar múltiplas worktrees?
10. Quais providers e versões mínimas serão suportados?
11. O Control chamará o canônico por path fixo, configuração ou versão instalada?
12. Quais capabilities são obrigatórias para abrir um projeto e quais permitem modo `partial`?
13. Quais hooks/adapters podem ser instalados automaticamente sem proteger arquivos customizados?
