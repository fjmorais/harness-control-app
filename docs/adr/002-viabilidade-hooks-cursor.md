# 002 — Viabilidade de hooks equivalentes no Cursor (spike da task 10)

**Status:** Aceita
**Data:** 2026-08-16
**Autores:** fjmorais + Claude (spike de pesquisa)

---

## Contexto

O ADR-001 (Opção B, Pergunta 7 do grill) deixou em aberto se o Cursor expõe um sistema de hooks
equivalente ao do Claude Code — sem essa confirmação, o adapter de telemetria da task 11 não
tinha escopo definido: implementar de verdade, ou declarar `unavailable` no manifest e adiar.

Esta é uma pesquisa (não código de produção) contra a documentação oficial do Cursor
(`https://cursor.com/docs/hooks`, acessada em 2026-08-16).

## Decisão

**Cursor expõe hooks estruturalmente equivalentes aos do Claude Code** — suficiente para
justificar um adapter real na task 11, não uma queda para `unavailable`.

### O que existe

Configuração em `.cursor/hooks.json` (nível projeto: `<raiz>/.cursor/hooks.json`; nível usuário:
`~/.cursor/hooks.json`), `version: 1`, mapeamento evento → lista de hooks. Comunicação via
**stdin/stdout JSON** — mesmo padrão do Claude Code (script recebe payload por stdin, não
argumento de CLI).

Eventos relevantes pro nosso design (mapeamento direto com o que a task 07 já implementou):

| Claude Code (task 07) | Cursor | Equivalência |
|---|---|---|
| `SessionStart` | `sessionStart` | Direta — `session_id` presente nos dois |
| `SessionEnd` | `sessionEnd` | Direta |
| `PreToolUse`/`PostToolUse` | `preToolUse`/`postToolUse` | Direta — dispara para todas as tools |
| `SubagentStop` | `subagentStop` (+ `subagentStart`, que o Claude Code não tem) | Cursor é mais rico — dá pra abrir o evento de início do subagente também, não só o de fim |

### Diferenças de payload a tratar no adapter (task 11)

- `postToolUse` do Cursor traz `tool_output` como **string JSON serializada** (não dict como o
  `tool_response` do Claude Code) — o adapter precisa fazer `json.loads()` antes de checar
  `is_error` ou payload equivalente.
- Cursor tem `tool_use_id` por chamada de tool e `conversation_id`/`generation_id` estáveis
  entre turnos — isso é **mais informação de correlação do que o Claude Code expõe hoje**.
  Vale reavaliar a limitação "correlation_id sempre None" da task 07 especificamente para o
  adapter Cursor (fora de escopo desta ADR, mas registrado para quando a task 11 for aberta).
- Resposta do hook via **stdout JSON + exit code** (0 = sucesso, 2 = bloqueia ação, outros =
  fail-open) — mais rico que o modelo do Claude Code (que não bloqueia ação via hook de
  telemetria neste design), mas nosso adapter só precisa emitir sem usar o campo `permission`
  (não queremos bloquear nada, só observar).
- `subagentStart`/`subagentStop` trazem `subagent_id`, `subagent_type`, `parent_conversation_id`
  — mapeável directly para o `writer_id` que a task 06/07 já usa (`"subagent_" + subagent_type`).

### O que não existe / risco residual

- Cloud agents do Cursor só rodam hooks `command-based` de `.cursor/hooks.json` do repositório
  (não hooks de nível usuário) — irrelevante pro nosso caso (canônico roda local).
- Não há confirmação de que Cursor tenha uma capability documentada equivalente ao mapeamento
  fino `capabilities.telemetry.cursor` do manifest v2 — isso é responsabilidade nossa (task 03),
  não do Cursor.

## Alternativas consideradas

### Opção A — Declarar `unavailable` sem investigar (não seguir a task 10) ❌ Rejeitada
Foi a hipótese de risco registrada no ADR-001 antes do spike. A pesquisa mostrou que é
desnecessariamente pessimista — o Cursor tem cobertura de hook comparável.

### Opção B — Implementar adapter Cursor reaproveitando `harness_hook.py` (task 07) com uma
camada de tradução de payload ✅ Escolhida
`harness_redact.py`, `harness_event_writer.py` e o núcleo de `harness_hook.py`
(`_build_event`, `find_run_dir`, etc.) são agnósticos de formato de payload — só a função de
dispatch/parsing do payload precisa de uma variante Cursor. Evita duplicar lógica de negócio.

## Recomendação para a task 11

**Implementar o adapter de hooks Cursor de verdade** (não cair para `unavailable`), com escopo:
1. `.cursor/hooks.json` gerado pelo `install-harness` apontando pros mesmos scripts
   (`scripts/harness_hook.py`), com uma função de parsing extra pra normalizar o payload Cursor
   pro formato interno já usado (`tool_output` JSON-string → dict, `subagent_id`/`subagent_type`
   → `writer_id`).
2. `capabilities.telemetry.cursor: true` no manifest quando o adapter estiver de fato ativo.
3. Reavaliar (task futura, fora de escopo aqui) se `correlation_id` pode deixar de ser sempre
   `None` no adapter Cursor, dado que `conversation_id`/`tool_use_id` existem lá.

## Consequências

### Positivas
- Task 11 sai do risco "talvez seja `unavailable`" para escopo concreto e testável.
- `harness_hook.py` não precisa ser reescrito — só estendido com uma camada de normalização de
  payload por executor.

### Negativas / Tradeoffs
- Mais um formato de payload pra manter em sincronia se a Cursor mudar o contrato de hooks (sem
  aviso de breaking change conhecido no momento desta pesquisa).

### Riscos
- Este spike não testou hooks Cursor rodando de verdade (só leu documentação) — a task 11 deve
  incluir teste de integração real (sessão Cursor de fato disparando o hook), não só simulação
  de payload, para confirmar que a documentação bate com o comportamento real.

## Revisão

Esta ADR deve ser revisada se a task 11 encontrar divergência entre o payload documentado aqui e
o payload real observado numa sessão Cursor de verdade.

## Fontes

- [Hooks | Cursor Docs](https://cursor.com/docs/hooks) — documentação oficial, acessada em
  2026-08-16.
