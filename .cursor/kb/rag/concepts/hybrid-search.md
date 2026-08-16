# Busca Híbrida — Semântica + BM25

## O problema do pure semantic

Busca vetorial é ótima para significado, mas falha em casos específicos:

```
Query: "modelo XR-7000"
Banco tem: "impressora XR-7000 com resolução 4800 dpi..."

→ Busca semântica pode retornar "impressora de alta resolução" (semanticamente próximo)
  mas perder exatamente "XR-7000" se a coleção tem muitos modelos

→ BM25 acerta "XR-7000" na primeira posição (match exato de token)
```

**Pure semantic falha quando:** siglas, códigos, nomes de produto/pessoa/lugar, termos técnicos raros, números de série.

## O que é BM25

BM25 (Best Match 25) é o algoritmo de busca por palavras-chave que o Elasticsearch usa por padrão. Ele calcula relevância baseado em:
- **TF (term frequency):** quantas vezes o termo aparece no documento
- **IDF (inverse document frequency):** quão raro o termo é na coleção
- **Normalização de tamanho:** penaliza docs muito longos

Resultado: match exato por tokens, sem entender significado.

## Reciprocal Rank Fusion (RRF)

Combina dois rankings (semântico + BM25) de forma simples e eficaz:

```
RRF score = Σ  1 / (k + rank_i)
```

onde `k=60` (constante de suavização, valor padrão) e `rank_i` é a posição no ranking `i`.

```python
def reciprocal_rank_fusion(
    results_list: list[list],
    k: int = 60,
) -> list:
    scores = {}
    for results in results_list:
        for rank, doc_id in enumerate(results, start=1):
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

## Implementação no Qdrant (sparse + dense)

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, SparseVectorParams, Distance,
    NamedVector, NamedSparseVector, SparseVector,
)
from fastembed import SparseTextEmbedding

# Collection com vetores denso + esparso
client.create_collection(
    collection_name="docs_hibridos",
    vectors_config={
        "dense": VectorParams(size=3072, distance=Distance.COSINE),
    },
    sparse_vectors_config={
        "sparse": SparseVectorParams(),
    },
)

# Modelo BM25 esparso (fastembed — roda local)
sparse_model = SparseTextEmbedding(model_name="Qdrant/bm25")

def index_document(doc_id: str, text: str, payload: dict):
    dense_vec = embed_dense(text)      # OpenAI text-embedding-3-large
    sparse_vec = next(sparse_model.embed([text]))  # BM25 esparso

    client.upsert(
        collection_name="docs_hibridos",
        points=[PointStruct(
            id=doc_id,
            vector={
                "dense": dense_vec,
                "sparse": SparseVector(
                    indices=sparse_vec.indices.tolist(),
                    values=sparse_vec.values.tolist(),
                ),
            },
            payload=payload,
        )],
    )

def hybrid_search(query: str, query_filter=None, limit: int = 5) -> list:
    dense_vec = embed_dense(query)
    sparse_vec = next(sparse_model.embed([query]))

    results = client.query_points(
        collection_name="docs_hibridos",
        prefetch=[
            Prefetch(
                query=NamedVector(name="dense", vector=dense_vec),
                using="dense",
                limit=20,
            ),
            Prefetch(
                query=NamedSparseVector(
                    name="sparse",
                    vector=SparseVector(
                        indices=sparse_vec.indices.tolist(),
                        values=sparse_vec.values.tolist(),
                    ),
                ),
                using="sparse",
                limit=20,
            ),
        ],
        query=FusionQuery(fusion=Fusion.RRF),  # RRF built-in no Qdrant
        query_filter=query_filter,
        limit=limit,
        with_payload=True,
    )
    return results.points
```

## Quando usar busca híbrida

| Caso | Recomendação |
|---|---|
| Base de conhecimento com narrativas puras | Pure semantic (simples, suficiente) |
| Catálogo de produtos com códigos e nomes | **Híbrida** (códigos precisam de BM25) |
| Base de FAQs com termos técnicos | **Híbrida** |
| Queries com nomes próprios / siglas | **Híbrida** |
| E-mails de suporte (linguagem natural) | Pure semantic |
| Documentação técnica com identificadores | **Híbrida** |
| Base pequena (< 1k docs) | Pure semantic (complexidade não compensa) |

## Pesos (optional fine-tuning)

```python
# RRF padrão (k=60) já equilibra bem.
# Para dar mais peso ao semântico:
from qdrant_client.models import FusionQuery, Fusion

# Qdrant usa RRF nativo — sem necessidade de implementar manualmente
query=FusionQuery(fusion=Fusion.RRF)
```

## Referências
- `rag-architecture.md` — onde busca híbrida se encaixa
- `../patterns/rag-with-keywords.md` — implementação detalhada
- `reranking.md` — aplicar reranking após fusão para ganho adicional
