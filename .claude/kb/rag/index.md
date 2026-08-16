---
domain: rag
description: "RAG e busca semântica/exata — chunking, embedding, hybrid search, reranking, padrão LEDGER"
---

# RAG + Busca Semântica/Exata — Índice

## Princípio central

> **Banco vetorial = apenas texto corrido (narrativas, descrições, documentos privados).**
> **Dados exatos = SQL/KV (padrão LEDGER). Misturar os dois degrada a qualidade de ambos.**

## Decision Matrix rápida

| Tipo de dado | Tipo de query | Canal recomendado |
|---|---|---|
| Documento/narrativa privada | "Me explique a política de..." | **RAG** (vetor) |
| ID, preço, CPF, saldo | "Qual o saldo da conta 12345?" | **LEDGER** (SQL/KV) |
| Web pública, docs de libs | "Como funciona a API do X?" | **MCP** |
| Docs privados + dados exatos | Pergunta mista | **Híbrido** (RAG + LEDGER) |
| Termos técnicos, siglas, códigos | Busca por produto/código | **Híbrido** (semântico + BM25) |

## Quando usar cada canal

```
Pergunta do usuário
       │
       ├─► Dados públicos / docs de bibliotecas?  ──► MCP (Context-7, Exa)
       │
       ├─► Resposta é única e determinística?     ──► LEDGER (SQL/KV)
       │     (preço, ID, CPF, data, saldo, enum)
       │
       ├─► Resposta requer contexto/significado?  ──► RAG (vetor)
       │     (política, explicação, narrativa)
       │
       └─► Mistura dos dois?                      ──► Híbrido (RAG + LEDGER)
```

## Conceitos

- [semantic-vs-exact.md](concepts/semantic-vs-exact.md) — Quando semântica, quando LEDGER
- [rag-architecture.md](concepts/rag-architecture.md) — Pipeline completo RAG
- [vector-db-what-not-to-store.md](concepts/vector-db-what-not-to-store.md) — O que NUNCA vai no vetor
- [chunking-strategies.md](concepts/chunking-strategies.md) — Como dividir documentos
- [embedding-selection.md](concepts/embedding-selection.md) — Escolha de modelo de embedding
- [hybrid-search.md](concepts/hybrid-search.md) — Semântico + BM25
- [reranking.md](concepts/reranking.md) — Melhorar precisão pós-retrieval
- [indexing-techniques.md](concepts/indexing-techniques.md) — HNSW, IVF, quantização

## Padrões

- [ledger-lookup.md](patterns/ledger-lookup.md) — Busca exata + two-query pattern
- [rag-pipeline.md](patterns/rag-pipeline.md) — Pipeline completo com código Python
- [query-expansion.md](patterns/query-expansion.md) — HyDE, multi-query, rewriting
- [metadata-filtering.md](patterns/metadata-filtering.md) — Pre-filter antes do semântico
- [rag-with-keywords.md](patterns/rag-with-keywords.md) — Implementação BM25 + sparse vectors

## Agentes disponíveis

- `rag-architect` — Projeta sistemas de retrieval do zero (entrevista → design → skeleton)
- `search-strategy-advisor` — Responde: "esse dado vai no vetor ou no SQL?"

## Skills disponíveis

- `/create-rag-pipeline` — Gera pipeline RAG completo (ingestão + query)
- `/search-strategy-check` — Checklist rápido: dado uma info, qual estratégia usar

## Quick Reference

Ver [quick-reference.md](quick-reference.md) — Decision Matrix, invariantes RAG-01…RAG-10,
setup mínimo de payload index. Ler só se a tarefa exigir esse nível de detalhe.

## Rule ativa

`rules/rag.md` — carrega ao tocar `**/rag/**`, `**/retrieval/**`, `**/search/**`, `**/vector*/**`, `**/embedding*/**`
