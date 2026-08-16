---
domain: rag
topic: quick-reference
---

# RAG — Quick Reference

### Decision Matrix rápida

| Tipo de dado | Tipo de query | Canal recomendado |
|---|---|---|
| Documento/narrativa privada | "Me explique a política de..." | **RAG** (vetor) |
| ID, preço, CPF, saldo | "Qual o saldo da conta 12345?" | **LEDGER** (SQL/KV) |
| Web pública, docs de libs | "Como funciona a API do X?" | **MCP** |
| Docs privados + dados exatos | Pergunta mista | **Híbrido** (RAG + LEDGER) |
| Termos técnicos, siglas, códigos | Busca por produto/código | **Híbrido** (semântico + BM25) |

### Invariantes (RAG-01…RAG-10 — ver `rules/rag.md` para o detalhe completo)

| # | Invariante |
|---|---|
| RAG-01 | Banco vetorial = só texto corrido. Nunca ID, preço, CPF, data, saldo, enum |
| RAG-02 | Dado exato → SQL/KV (padrão LEDGER), nunca busca semântica |
| RAG-03 | Pre-filter (`tenant_id`, `type`, `status`) SEMPRE antes da busca semântica |
| RAG-04 | Grounding obrigatório — toda resposta cita o(s) chunk(s) fonte |
| RAG-05 | Collection escolhida pelo nó do grafo (determinístico), nunca pelo LLM em loop livre |
| RAG-06 | Chunk máximo 512 tokens, overlap 10-15% |
| RAG-07 | Metadata schema versionado — `source`, `section`, `date`, `type`, `tenant_id`, `content`, `chunk_index` |
| RAG-08 | Tenant isolation via pre-filter (payload index em `tenant_id`), nunca só semântica |
| RAG-09 | Reranking obrigatório em coleções > 10k documentos |
| RAG-10 | HyDE ou multi-query para queries ambíguas quando recall < 70% |

### Setup mínimo — payload index obrigatório

```python
for field in ["tenant_id", "type", "status", "date"]:
    client.create_payload_index(collection_name=..., field_name=field, ...)
```
