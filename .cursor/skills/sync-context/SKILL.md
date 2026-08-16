---
name: sync-context
description: >-
  Detecta drift entre os arquivos de contexto vivos (CLAUDE.md, HANDOFF.md, CONTEXT.md,
  docs/adr/) e o estado real do código — e atualiza o que ficou desatualizado. Nunca inventa:
  só sincroniza o que já está implementado. Use quando: "o CLAUDE.md está desatualizado",
  "atualiza o contexto do projeto", "sincroniza o handoff", "o que mudou desde a última sessão?",
  "o CONTEXT.md não reflete mais o código".
---

# Sync Context

Detecta e corrige drift entre documentação de contexto e código real.
**Regra central:** nunca adicionar o que não está implementado. Só descrever o que existe.

## Arquivos de contexto gerenciados

| Arquivo | Propósito | Quando atualizar |
|---|---|---|
| `CLAUDE.md` | Invariantes, stack, comandos — carrega em toda sessão | Stack mudou, novo invariante, novo comando |
| `HANDOFF.md` | Estado da sessão atual — ponto de continuidade | Fim de sessão ou mudança de escopo |
| `CONTEXT.md` | Domínio e decisões — single-context do projeto | Nova decisão, novo fluxo implementado |
| `docs/adr/` | Decisões arquiteturais registradas | Nova ADR criada ou decisão revertida |

## Processo

### Passo 1 — Inventário do que existe no código

```bash
# Entry points, routers, serviços ativos
find . -name "*.py" -o -name "*.ts" -o -name "*.go" | grep -v test | head -40

# Docker services rodando
grep -A3 "services:" docker-compose.yml 2>/dev/null || true

# Variáveis de ambiente reais
cat .env.example 2>/dev/null || true
```

### Passo 2 — Leitura dos arquivos de contexto atuais

Leia cada arquivo listado na tabela acima e anote:
- O que está declarado
- O que parece desatualizado (serviço removido, rota inexistente, variável obsoleta)
- O que está implementado mas não documentado

### Passo 3 — Gerar relatório de drift

Antes de editar qualquer coisa, exibir o relatório:

```markdown
## Relatório de Drift — {data}

### Desatualizado (presente no contexto, ausente no código)
- [ ] {descrição exata do que está errado e em qual arquivo}

### Ausente (presente no código, não documentado no contexto)
- [ ] {o que falta documentar}

### Correto (sem mudança necessária)
- {lista do que está em sync}
```

Aguardar confirmação do usuário antes de aplicar as mudanças.

### Passo 4 — Aplicar atualizações

Para cada item "desatualizado" ou "ausente" confirmado:
1. Editar apenas a seção específica do arquivo — não reescrever tudo
2. Manter o estilo e tom do documento existente
3. Citar evidência do código (`arquivo:linha`) para cada adição

### Passo 5 — Confirmar sync

Exibir diff resumido do que foi alterado:
```
CLAUDE.md: seção "Stack" — adicionado Redis, removido RabbitMQ
HANDOFF.md: estado atual atualizado para refletir PR #42 mergeado
CONTEXT.md: novo fluxo de autenticação documentado
```

## O que NÃO fazer

- Não adicionar funcionalidades "planejadas" que não estão no código
- Não reescrever o estilo ou reestruturar documentos desnecessariamente
- Não atualizar ADRs — ADRs são imutáveis (nova decisão = nova ADR)
- Não editar sem mostrar o relatório de drift primeiro

## Referências

- `CLAUDE.md` — template de invariantes de projeto
- `HANDOFF.md` — template de estado de sessão
- `.claude/agents/dev/codebase-explorer.md` — para mapear o código antes do sync
