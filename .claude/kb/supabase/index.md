---
domain: supabase
description: Supabase — Auth, RLS, Edge Functions, pgvector, storage e realtime para projetos full-stack
mcp_validated: "2026-06-27"
confidence: 0.91
---

# KB: Supabase

Base de conhecimento de padrões Supabase para autenticação, segurança por linha, funções serverless e busca vetorial.
Princípio central: **RLS é a última linha de defesa, não a única** — nunca expor dados sem política RLS ativa.

## Conceitos

| Arquivo | Tópico |
|---|---|
| [auth.md](concepts/auth.md) | Autenticação — magic link, OAuth, JWT, sessão no cliente |
| [rls.md](concepts/rls.md) | Row Level Security — políticas, `auth.uid()`, multi-tenant |
| [edge-functions.md](concepts/edge-functions.md) | Deno serverless — quando usar, padrões, segredos, CORS |
| [pgvector.md](concepts/pgvector.md) | RAG sobre Postgres — `vector`, HNSW index, `match_documents()` |

## Padrões

| Arquivo | Tópico |
|---|---|
| [rls-patterns.md](patterns/rls-patterns.md) | Políticas prontas para uso — owner, org, admin, public |
| [realtime-patterns.md](patterns/realtime-patterns.md) | Subscriptions Realtime — channel, filter, presença |

## Quick Reference

Ver [quick-reference.md](quick-reference.md) — invariantes (SB-01…SB-05), `anon` vs `service_role`,
setup mínimo do cliente. Ler só se a tarefa exigir esse nível de detalhe.
