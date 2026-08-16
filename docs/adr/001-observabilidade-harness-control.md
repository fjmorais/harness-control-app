# 001 — Arquitetura de observabilidade do Harness Control (`.harness/`)

**Status:** Aceita
**Data:** 2026-08-15
**Autores:** fjmorais + Claude (sessão de grill estruturado)

---

## Contexto

O Agent Harness Canônico gera agents, skills, rules e KBs e instala tudo via
`/install-harness`, mas nenhum projeto instalado é observável: não há estado estruturado,
evento de execução, custo por sessão/task, ou trilha de auditoria em formato consumível por
uma ferramenta externa. A única fonte de dado sobre entregas hoje é `metrics/entregas.jsonl`,
que cobre só "task concluída" — não execução em si (sessões, tool calls, tokens, subagentes
em paralelo).

Um plano inicial (`sketch/agent-harness-canonico/plan.md`) propôs uma estrutura `.harness/`
com runs, eventos, custos e auditoria, além de schemas versionados publicados no canônico.
Esse plano foi levado a uma sessão de grill estruturado (14 decisões, ver
`.claude/projetos/harness-control/01-grill.md`) porque várias partes do plano original eram
suposições não testadas contra o comportamento real do Claude Code (hooks disponíveis,
concorrência de subagentes, granularidade de sessão) ou criavam risco de dado desatualizado
silencioso (preço por token hardcoded).

O canônico continua sendo framework — nenhum código do futuro produto de observabilidade
("Harness Control") entra neste repositório. Esta ADR registra as decisões que **mudam** ou
**refinam** o plano original a partir do grill, não repete o plano inteiro.

## Decisão

Vamos instrumentar o canônico com uma camada `.harness/` observável por projeto instalado,
usando emissão híbrida de eventos (hooks do executor para telemetria objetiva + agentes para
eventos de workflow), run = sessão inteira do executor, escrita concorrente resolvida por
arquivo-por-escritor (nunca lock em `events.jsonl`), redaction só no caminho de hook, sem
cálculo de preço local (só tokens brutos por modelo), `project_id` como hash imutável, e
`harness-doctor` detectando customização por checksum de todo arquivo gerado — porque essas
escolhas mantêm o dado honesto (nunca zero silencioso), evitam reabrir fluxos já validados
(`harness-build` → `metrics/entregas.jsonl` → `/scorecard`), e não comprometem o canônico a
suposições não verificadas sobre executors além do Claude Code.

## Alternativas consideradas

### Opção A — Run = turno individual (hook `Stop`) ❌ Rejeitada
Mais fiel à granularidade nativa exposta pelo Claude Code, mas gera runs fragmentados demais
(um por turno) e exige agregação pesada só pra reconstruir "quanto custou uma task". `Stop`
dispara a cada resposta do agente, não a cada sessão — usá-lo pra abrir/fechar run quebraria
tasks de múltiplos turnos no meio da execução.

### Opção B — Run = sessão inteira (`SessionStart`/`SessionEnd`), task via `correlation_id` ✅ Escolhida
Um hook nativo por sessão é suficiente pra abrir/fechar o run; task/skill/comando de alto
nível vira `correlation_id` dentro do run — reaproveita um campo que o envelope de evento já
reservava, sem precisar de instrumentação manual por skill.

### Opção C — Lock único em `.harness/locks/project.lock` para toda escrita ❌ Rejeitada
Simples de raciocinar, mas serializa sessões/subagentes concorrentes sem necessidade — dado
que cada run já escreve numa pasta exclusiva por natureza (`run_<id>`), um lock de projeto
inteiro criaria contenção artificial, inclusive entre subagentes paralelos dentro da mesma
sessão (cenário real, comprovado nesta própria sessão de grill).

### Opção D — Arquivo de eventos único por run, protegido por `flock` por append ❌ Rejeitada
Mantém um arquivo só (mais simples de consumir direto), mas introduz lock real em cada
escrita — com múltiplos subagentes disparando tool calls em paralelo, vira gargalo. Rejeitada
em favor de eliminar a classe de problema, não mitigá-la.

### Opção E — Arquivo de eventos por escritor dentro do run, mesclado por reconstrução ✅ Escolhida
Cada escritor (agente principal, cada subagente) grava só no seu próprio arquivo dentro do
run — nunca há dois processos tocando o mesmo arquivo, eliminando a condição de corrida sem
lock algum. Reaproveita o conceito de "índice reconstruível" que o plano original já previa
(`indexes/`) — a mesclagem por `sequence`/`occurred_at`/`parent_event_id` acontece só na
leitura.

### Opção F — `pricing.json` local por projeto, com custo estimado em `$` ❌ Rejeitada
O plano original previa isso, mas exige manter tabela de preço atualizada em cada projeto
instalado — exatamente o tipo de "estimativa desatualizada silenciosa" que a própria regra de
honestidade do plano queria evitar. Preço por modelo muda com o tempo; replicar isso em N
projetos é um problema de manutenção sem necessidade.

### Opção G — Só tokens brutos por modelo em `usage.json`, sem custo em `$` ✅ Escolhida
O executor sempre sabe quantos tokens consumiu por modelo (mesmo com roteamento "auto"
variando por turno/subagente); preço é política de negócio que muda e deveria viver num lugar
central (o futuro Harness Control), não replicada em cada `.harness/costs/`.

### Opção H — Redaction centralizada num script único, chamada por hook e por agente ❌ Rejeitada
Mais defesa em profundidade, mas exige que todo caminho de escrita invoque o mesmo script via
`Bash` em vez de `Write` direto — atrito desnecessário dado que o caminho de agente já é
estruturalmente limitado a campos sem texto livre (não carrega PII por design).

### Opção I — Redaction só no caminho de hook; eventos de agente sem campo de texto livre ✅ Escolhida
Simplifica a garantia: só o caminho que pode tocar dado sensível (hook, telemetria objetiva
com paths/prompts) precisa de redaction; o caminho de agente é seguro por construção de
schema, não por disciplina de implementação.

### Opção J — `project_id` = UUID v4 aleatório na primeira instalação ❌ Rejeitada
Simples e estável, mas não é verificável/reproduzível sem depender do arquivo gravado —
menos valor para quem quer confirmar a identidade do projeto de forma determinística.

### Opção K — `project_id` = hash(path absoluto + timestamp de criação), calculado uma vez e depois imutável ✅ Escolhida
Combina determinismo (origem verificável) com estabilidade (nunca recalculado depois da
primeira instalação, então mover o projeto de pasta não muda o ID).

## Consequências

### Positivas
- Dado de telemetria nunca é zero silencioso — ausência é sempre reportada com motivo
  (`unavailable`, `blocked`), consistente com `rules/seguranca.md` e a regra de honestidade
  do plano original.
- Nenhum fluxo existente quebra: `harness-build` → `metrics/entregas.jsonl` → `/scorecard`
  continua igual; `.harness/deliveries/` é só uma view derivada.
- Concorrência de subagentes (cenário real, não hipotético) é resolvida por design (arquivo
  por escritor), não por lock — menos superfície de bug de deadlock/contenção.
- Escopo do MVP fica honesto sobre o que não foi validado (hooks do Cursor) em vez de
  assumir paridade com o Claude Code sem checar.

### Negativas / Tradeoffs
- Ler "o que aconteceu numa sessão" exige mesclar N arquivos de evento em vez de ler um só —
  mais complexidade na leitura em troca de menos complexidade (zero lock) na escrita.
- Sem custo em `$` local, qualquer visão de custo por projeto depende do consumidor externo
  (Harness Control) ter acesso à tabela de preço — o canônico sozinho não responde "quanto
  custou" em dinheiro, só em tokens.
- `harness-doctor` com checksum de tudo (`.claude/` + `.cursor/`, não só `.harness/`) é mais
  trabalho de instrumentação inicial do que um escopo reduzido só a `.harness/` teria sido.

### Riscos
- Viabilidade do adapter de Cursor não está confirmada — se o Cursor não expuser hooks de
  sessão/tool-call equivalentes, o MVP de telemetria cobre efetivamente só Claude Code até um
  spike resolver isso.
- `harness-prune` manual entra já no MVP (não adiado) — mais uma peça a construir e testar
  antes de C1/C2 estarem prontos, mas necessária porque `runs/**` não tem backup no Git.

## Revisão

Esta ADR deve ser revisada se:
- O spike de viabilidade do Cursor confirmar que não há hooks equivalentes — nesse caso, a
  Opção B pode precisar de uma variante específica para Cursor (run delimitado de outra
  forma, ou caminho de agente como fallback permanente, não só temporário).
- O Harness Control (produto externo) definir uma necessidade real de custo em `$` que não
  possa ser resolvida só com tokens brutos — reabriria a discussão da Opção F/G.
- O volume de subagentes paralelos por sessão crescer a ponto de o custo de mesclar N
  arquivos de evento na leitura se tornar um gargalo perceptível.
