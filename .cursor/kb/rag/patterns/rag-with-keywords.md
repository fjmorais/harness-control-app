# RAG com Keywords — Busca Híbrida (Sparse + Dense)

## Quando o pure semantic não basta

```
Cenário: catálogo de peças técnicas
Query: "parafuso M8 aço inox"

Pure semantic: retorna parafusos em geral (semântica OK), mas pode perder exatamente "M8"
Híbrido: BM25 garante match exato de "M8", semântico dá contexto de material e uso

Resultado: híbrido supera pure semantic em ~15-30% de recall nesses casos
```

**Use híbrida quando:** termos técnicos, códigos de produto, siglas, nomes próprios, versões.

## Setup do Qdrant para busca híbrida

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, SparseVectorParams, Distance,
    PointStruct, SparseVector,
)

client = QdrantClient(url="http://localhost:6333")

# Collection com vetores denso E esparso
client.create_collection(
    collection_name="catalogo_hibrido",
    vectors_config={
        "dense": VectorParams(size=3072, distance=Distance.COSINE),
    },
    sparse_vectors_config={
        "sparse": SparseVectorParams(),  # BM25 sparse vectors
    },
)

# Indexes obrigatórios
for field in ["tenant_id", "type", "categoria"]:
    client.create_payload_index(
        collection_name="catalogo_hibrido",
        field_name=field,
        field_schema="keyword",
    )
```

## Modelo BM25 esparso (fastembed)

```python
from fastembed import SparseTextEmbedding, TextEmbedding

# Modelos locais — sem custo de API
dense_model = TextEmbedding(model_name="BAAI/bge-large-en-v1.5")      # 1024d (alternativa local)
sparse_model = SparseTextEmbedding(model_name="Qdrant/bm25")           # BM25 esparso

# OU: usar OpenAI para dense + fastembed para sparse
from openai import OpenAI
oai = OpenAI()

def embed_dense_oai(text: str) -> list[float]:
    return oai.embeddings.create(
        model="text-embedding-3-large",
        input=text,
    ).data[0].embedding

def embed_sparse(text: str) -> SparseVector:
    result = next(sparse_model.embed([text]))
    return SparseVector(
        indices=result.indices.tolist(),
        values=result.values.tolist(),
    )
```

## Ingestão híbrida

```python
def ingest_hybrid(
    doc_id: str,
    text: str,
    payload: dict,
) -> None:
    dense_vec = embed_dense_oai(text)
    sparse_vec = embed_sparse(text)

    client.upsert(
        collection_name="catalogo_hibrido",
        points=[PointStruct(
            id=doc_id,
            vector={
                "dense": dense_vec,
                "sparse": sparse_vec,
            },
            payload={
                **payload,
                "content": text,
            },
        )],
    )

# Ingestão de catálogo:
for produto in produtos:
    # Indexa a descrição narrativa — NÃO o preço (que vai no SQL)
    ingest_hybrid(
        doc_id=f"prod-{produto['id']}",
        text=f"{produto['nome']} {produto['descricao']} {produto['categoria']}",
        payload={
            "produto_id": produto["id"],   # referência para LEDGER lookup posterior
            "tenant_id": "empresa-a",
            "type": "produto",
            "categoria": produto["categoria"],
            "source": f"catalogo/produto-{produto['id']}",
            "date": produto["data_atualizacao"],
        },
    )
    # produto["preco"] → NÃO vai no vetor. Fica no SQL.
```

## Query híbrida

```python
from qdrant_client.models import (
    Prefetch, NamedVector, NamedSparseVector,
    FusionQuery, Fusion, Filter, FieldCondition, MatchValue,
)

def hybrid_search(
    query: str,
    tenant_id: str,
    doc_type: str | None = None,
    top_k: int = 5,
) -> list:
    dense_vec = embed_dense_oai(query)
    sparse_vec = embed_sparse(query)

    must = [FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))]
    if doc_type:
        must.append(FieldCondition(key="type", match=MatchValue(value=doc_type)))
    filter_ = Filter(must=must)

    results = client.query_points(
        collection_name="catalogo_hibrido",
        prefetch=[
            Prefetch(
                query=NamedVector(name="dense", vector=dense_vec),
                using="dense",
                filter=filter_,
                limit=20,
            ),
            Prefetch(
                query=NamedSparseVector(
                    name="sparse",
                    vector=sparse_vec,
                ),
                using="sparse",
                filter=filter_,
                limit=20,
            ),
        ],
        query=FusionQuery(fusion=Fusion.RRF),  # RRF built-in
        limit=top_k,
        with_payload=True,
    )
    return results.points

# Uso:
chunks = hybrid_search(
    query="parafuso M8 aço inox cabeça sextavada",
    tenant_id="empresa-a",
    doc_type="produto",
)
# → recupera o produto exato com "M8" (BM25) + produtos similares (semântico)
#   na ordem certa graças ao RRF
```

## Integração com LEDGER (two-query)

Após encontrar o produto via busca híbrida, buscar o preço no SQL:

```python
async def search_produto_com_preco(query: str, tenant_id: str):
    # 1. Híbrido: acha o produto pelo texto
    chunks = hybrid_search(query, tenant_id, doc_type="produto")

    if not chunks:
        return None, "Produto não encontrado"

    produto_id = chunks[0].payload["produto_id"]

    # 2. LEDGER: extrai o preço exato
    result = await sql_ledger.lookup(
        table="produtos",
        where={"produto_id": produto_id, "tenant_id": tenant_id},
        select=["preco", "estoque", "ultima_atualizacao"],
    )

    context = f"Produto encontrado: {chunks[0].payload['content']}\n"
    if result.found:
        context += f"\nPreço atual: R$ {result.value['preco']:.2f} (fonte: {result.source})"
        context += f"\nEstoque: {result.value['estoque']} unidades"

    return chunks, context
```

## Referências
- `../concepts/hybrid-search.md` — conceitos de BM25 e RRF
- `../concepts/indexing-techniques.md` — sparse vectors no Qdrant
- `ledger-lookup.md` — two-query pattern para dados exatos
- `metadata-filtering.md` — pre-filter em queries híbridas
