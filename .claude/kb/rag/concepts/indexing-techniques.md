# Técnicas de Indexação Vetorial

## Tipos de índice

| Índice | Recall | Velocidade | Memória | Quando usar |
|---|---|---|---|---|
| **HNSW** | Alto (~95%) | Rápido | Médio-alto | **Padrão** — melhor trade-off |
| **IVF** | Médio (~85%) | Muito rápido | Baixo | Coleções > 10M vetores |
| **Flat / Brute-force** | 100% (exato) | Lento | Alto | Coleções < 10k, testes |
| **ScaNN** | Alto | Rápido | Médio | Google Cloud, coleções grandes |

**Padrão do harness:** HNSW com Qdrant. Use IVF só se HNSW não caber em memória.

## HNSW — Hierarchical Navigable Small World

Grafos de múltiplas camadas com atalhos de "longo alcance" nas camadas superiores. Busca navega do geral para o específico.

```python
from qdrant_client.models import HnswConfigDiff

# Configuração padrão (Qdrant usa HNSW automaticamente)
client.create_collection(
    collection_name="docs_privados",
    vectors_config=VectorParams(
        size=3072,
        distance=Distance.COSINE,
    ),
    hnsw_config=HnswConfigDiff(
        m=16,               # conexões por nó (default 16 — não mudar sem benchmark)
        ef_construct=100,   # qualidade da construção (padrão 100)
        # full_scan_threshold: abaixo de N pontos, usa brute-force (padrão)
    ),
)

# Tempo de busca: ajustar ef na query (maior ef = mais recall, mais lento)
results = client.search(
    collection_name="docs_privados",
    query_vector=query_vec,
    limit=10,
    search_params=SearchParams(hnsw_ef=128, exact=False),
)
```

**Parâmetros:**
- `m=16`: número de conexões bidirecionais. Aumentar → melhor recall, mais memória. Default é ótimo.
- `ef_construct=100`: qualidade da construção do índice. Aumentar → melhor recall, indexação mais lenta.
- `ef` na query: padrão é `max(ef_construct, limit)`. Aumentar para recall máximo.

## IVF — Inverted File Index

Divide o espaço vetorial em clusters (Voronoi cells). Busca só nos clusters mais próximos.

```python
# Não nativo no Qdrant — disponível no Faiss
import faiss

# IVF com 256 centroides
quantizer = faiss.IndexFlatL2(3072)
index = faiss.IndexIVFFlat(quantizer, 3072, 256)

# Treino obrigatório (requer amostra representativa)
index.train(training_vectors)
index.add(all_vectors)

# nprobe: quantos clusters buscar (mais = melhor recall, mais lento)
index.nprobe = 10
distances, indices = index.search(query_vec, k=10)
```

**Quando usar IVF:**
- > 1M vetores (HNSW fica pesado em memória)
- Budget de memória restrito
- Latência < 10ms necessária com coleção muito grande

## Scalar Quantization — reduzir memória

Comprime vetores float32 (4 bytes) para int8 (1 byte) → 4x redução de memória.

```python
from qdrant_client.models import ScalarQuantizationConfig, ScalarType, QuantizationSearchParams

# Na criação da collection
client.create_collection(
    collection_name="docs_privados",
    vectors_config=VectorParams(size=3072, distance=Distance.COSINE),
    quantization_config=ScalarQuantizationConfig(
        scalar=ScalarType.INT8,
        quantile=0.99,      # percentil para clamp (0.99 = ignora 1% dos outliers)
        always_ram=True,    # mantém em RAM mesmo com quantização
    ),
)

# Na busca: rescore = True recomputa scores exatos para os candidatos finais
results = client.search(
    collection_name="docs_privados",
    query_vector=query_vec,
    limit=10,
    search_params=SearchParams(
        quantization=QuantizationSearchParams(
            ignore=False,
            rescore=True,   # ~1% perda de recall sem rescore; ~0% com rescore
        )
    ),
)
```

**Trade-off:** ~1% perda de recall com rescore=True, 4x menos memória. Vale quase sempre.

## Criação de payload indexes (para pre-filter eficiente)

```python
from qdrant_client.models import PayloadSchemaType

# Criar índices nos campos de filtro mais usados
client.create_payload_index(
    collection_name="docs_privados",
    field_name="tenant_id",
    field_schema=PayloadSchemaType.KEYWORD,
)

client.create_payload_index(
    collection_name="docs_privados",
    field_name="type",
    field_schema=PayloadSchemaType.KEYWORD,
)

client.create_payload_index(
    collection_name="docs_privados",
    field_name="date",
    field_schema=PayloadSchemaType.DATETIME,
)
```

**Sem índice de payload:** pre-filter faz scan completo da coleção.
**Com índice:** pre-filter em O(log n) antes da busca vetorial.

## Decisão de índice — fluxo

```
Volume da coleção
       │
       ├── < 10k docs     → Flat/brute-force (teste) ou HNSW (default)
       │
       ├── 10k – 1M docs  → HNSW (padrão Qdrant)
       │                     + Scalar Quantization se memória for restrita
       │
       └── > 1M docs      → HNSW + Quantização (primeiro)
                             se ainda não couber: IVF (Faiss)
```

## Referências
- `embedding-selection.md` — dimensão do vetor determina custo do índice
- `hybrid-search.md` — sparse index para BM25
- `../patterns/rag-pipeline.md` — criação do índice no pipeline de ingestão
