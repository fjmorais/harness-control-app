# harness-control-app

> Gerado pelo install-harness em 2026-08-16. Stack detectada: (genérica).
> Refine com `/grill-me` para registrar invariantes e decisões reais do projeto.

## [AVISO_LGPD]

Este projeto lida com PII (nível c do SI Assessment — ver
`.claude/projetos/harness-control/00-ideia.md`): email de login, dados de sessão e trilha de
auditoria (`audit_events`) identificam pessoa natural. Regras de mascaramento específicas em
`.claude/rules/pii.md`. Nenhum campo de PII aparece em log, URL query string, ou é exibido na UI
sem função de mascaramento.

## Stack

Next.js/React (frontend) + FastAPI (backend) + PostgreSQL (control plane) + OpenTelemetry
(telemetria) + adapters Claude Code/Codex (execução de agente). Multiusuário, multi-tenant,
autenticação obrigatória no modo compartilhado. Ver `sketch/harness-control/plan.md` para o
plano completo.

## Invariantes

(preencher com /grill-me — ver os 13 pontos em aberto do plano original, seção 14, como ponto
de partida da entrevista)
