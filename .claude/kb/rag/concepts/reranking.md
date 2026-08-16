# Reranking — Melhorar Precisão Pós-Retrieval

## Por que rerankar

Busca vetorial (top-k) recupera por similaridade bruta — rápida mas imprecisa em coleções grandes.

Reranking aplica um modelo mais caro e preciso sobre os candidatos já recuperados:

```
Vetor: top-100 candidatos em ~50ms   ← rápido, amplo
Reranker: top-5 dos 100 em ~200ms   ← preciso, sobre conjunto pequeno

Trade-off: +200ms de latência → muito melhor precisão
```

**Regra:** Reranking obrigatório para coleções > 10k documentos.

## Opção 1: Cohere Rerank API (recomendado para produção)

```python
import cohere
import os

co = cohere.Client(api_key=os.getenv("COHERE_API_KEY"))

def rerank_with_cohere(
    query: str,
    results: list,
    top_n: int = 3,
    model: str = "rerank-multilingual-v3.0",  # pt-BR ok
) -> list:
    docs = [r.payload["content"] for r in results]

    reranked = co.rerank(
        query=query,
        documents=docs,
        model=model,
        top_n=top_n,
    )

    # Retorna na ordem rerankeada com score
    return [
        {
            "chunk": results[hit.index],
            "relevance_score": hit.relevance_score,
        }
        for hit in reranked.results
    ]

# Uso:
candidates = qdrant.search(collection_name="docs", query_vector=query_vec, limit=20)
top_chunks = rerank_with_cohere(user_query, candidates, top_n=3)
```

**Custo:** ~$0.001 por 1k tokens de entrada — barato relativo ao LLM final.

## Opção 2: Cross-encoder local (sem custo por API)

```python
from sentence_transformers import CrossEncoder

model = CrossEncoder(
    "cross-encoder/ms-marco-MiniLM-L-6-v2",  # inglês
    # "cross-encoder/mmarco-mMiniLMv2-L12-H384-v1",  # multilíngue / pt-BR
)

def rerank_local(query: str, results: list, top_n: int = 3) -> list:
    pairs = [(query, r.payload["content"]) for r in results]
    scores = model.predict(pairs)

    ranked = sorted(
        zip(scores, results),
        key=lambda x: x[0],
        reverse=True,
    )
    return [doc for _, doc in ranked[:top_n]]
```

**Quando preferir:** self-hosted, air-gapped, budget-constrained.
**Latência:** ~300-500ms para 20 candidatos com GPU; ~2s sem GPU.

## Opção 3: RRF (já embutido em busca híbrida)

RRF não é um reranker de precisão — é um método de fusão. Mas já dá um "reranking gratuito" ao combinar múltiplos rankings.

Use Cohere/cross-encoder **depois** do RRF para máximo ganho.

## Quando NÃO rerankar

| Situação | Motivo para pular |
|---|---|
| Coleção < 1k docs | Top-k já é preciso o suficiente |
| Latência crítica < 100ms | Reranker adiciona ~200ms |
| Queries muito longas (> 512 tokens) | Cross-encoders têm limite de contexto |
| Prototipagem / desenvolvimento | Complexidade desnecessária |

## Pipeline completo com reranking

```python
def query_with_rerank(
    question: str,
    tenant_id: str,
    collection: str,
    top_k: int = 3,
) -> tuple[str, list[str]]:
    # 1. Embed query
    q_vec = embed_dense(question)

    # 2. Pre-filter + busca ampla (mais candidatos para rerankar)
    candidates = qdrant.search(
        collection_name=collection,
        query_vector=q_vec,
        query_filter=Filter(must=[
            FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))
        ]),
        limit=20,  # busca mais para rerankar
    )

    if not candidates:
        return "", []

    # 3. Rerank → top-k final
    top_chunks = rerank_with_cohere(question, candidates, top_n=top_k)

    # 4. Build context com grounding
    context_parts = []
    sources = []
    for i, hit in enumerate(top_chunks, 1):
        chunk = hit["chunk"]
        context_parts.append(f"[{i}] {chunk.payload['content']}")
        sources.append(f"[{i}] {chunk.payload['source']} (score: {hit['relevance_score']:.2f})")

    return "\n\n".join(context_parts), sources
```

## Referências
- `hybrid-search.md` — RRF como etapa anterior
- `rag-architecture.md` — posição do reranker no pipeline
- `../patterns/rag-pipeline.md` — integração no pipeline completo
