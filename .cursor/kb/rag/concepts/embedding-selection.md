# Seleção de Modelo de Embedding

## Modelos OpenAI (padrão do harness)

| Modelo | Dimensões | Custo | Melhor para |
|---|---|---|---|
| `text-embedding-3-large` | 3072d | Alto | Produção, máxima qualidade |
| `text-embedding-3-small` | 1536d | Baixo | Dev, chunking semântico, prototipagem |
| `text-embedding-ada-002` | 1536d | Médio | Legacy — preferir 3-small |

**Padrão do harness:** `text-embedding-3-large` para indexação. `text-embedding-3-small` para chunking semântico (chamado muitas vezes durante ingestão).

## Trade-offs

```
Qualidade  ──────────────────────────────────► alta
           ada-002    3-small    3-large

Custo      ──────────────────────────────────► caro
           3-small    ada-002    3-large

Velocidade ──────────────────────────────────► lento
           3-small    3-large    ada-002

Armazenamento ───────────────────────────────► maior
           3-small(1536d)    3-large(3072d)
```

## Configuração no Qdrant

```python
from qdrant_client.models import VectorParams, Distance

# Para text-embedding-3-large
client.create_collection(
    collection_name="docs_privados",
    vectors_config=VectorParams(
        size=3072,
        distance=Distance.COSINE,  # padrão para OpenAI embeddings
    ),
)

# Para text-embedding-3-small
client.create_collection(
    collection_name="docs_privados",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE,
    ),
)
```

## Considerações pt-BR e multilíngue

Os modelos `text-embedding-3-*` são **nativamente multilíngues** — funcionam bem em português sem ajuste.

Pontos de atenção:
- Termos técnicos em português podem ter representação diferente de inglês → testar recuperação
- Siglas e acrônimos brasileiros (CPF, CNPJ, CRM, etc.) podem não ser bem representados → considerar expansão no preprocessing

## Modelos alternativos (open-source)

| Modelo | Dimensões | Quando considerar |
|---|---|---|
| `BAAI/bge-large-en-v1.5` | 1024d | Se não puder chamar API OpenAI |
| `intfloat/multilingual-e5-large` | 1024d | Multilíngue, melhor pt-BR |
| `sentence-transformers/paraphrase-multilingual-mpnet-base-v2` | 768d | Leve, on-premise |

Para self-hosted com Ollama:
```bash
ollama pull nomic-embed-text  # 768d, rápido
```

## Consistência ingestão ↔ query (invariante crítico)

**O mesmo modelo DEVE ser usado na ingestão e na query.**

```python
# CERTO: mesmo modelo nos dois lados
EMBEDDING_MODEL = "text-embedding-3-large"

# Ingestão
chunk_vector = openai.embeddings.create(model=EMBEDDING_MODEL, input=chunk_text)

# Query
query_vector = openai.embeddings.create(model=EMBEDDING_MODEL, input=user_question)

# ERRADO: modelos diferentes (dimensões incompatíveis, distâncias inválidas)
# Ingestão: text-embedding-3-large (3072d)
# Query:    text-embedding-3-small (1536d) ← erro de dimensão!
```

## Scalar Quantization (reduz memória, pequena perda de recall)

```python
from qdrant_client.models import ScalarQuantizationConfig, ScalarType

client.create_collection(
    collection_name="docs_privados",
    vectors_config=VectorParams(size=3072, distance=Distance.COSINE),
    quantization_config=ScalarQuantizationConfig(
        scalar=ScalarType.INT8,  # 4x redução de memória, ~1% perda de recall
        quantile=0.99,
        always_ram=True,
    ),
)
```

Use quando: coleção > 1M vetores ou memória é restrita.

## Referências
- `chunking-strategies.md` — chunk size alinhado ao modelo
- `indexing-techniques.md` — configuração do índice Qdrant
- `rag-architecture.md` — onde embedder se encaixa
