# Arquitetura RAG — Do Naive ao Modular

## Evolução dos padrões RAG

### Naive RAG (baseline)

```
Ingestão:  Documento → Split → Embed → Store
Query:     Pergunta → Embed → Search → Gerar resposta
```

Problemas: chunking fixo ignora estrutura, sem reranking, sem filtragem.

### Advanced RAG

Adiciona pré-processamento da query e pós-processamento dos resultados:

```
Ingestão:  Documento → Limpeza → Chunking inteligente → Embed → Store + Metadata
Query:     Pergunta → Rewrite/Expand → Embed → Search + Pre-filter → Rerank → Gerar com grounding
```

### Modular RAG (padrão atual)

Cada etapa é um módulo substituível. O nó do grafo escolhe a estratégia.

```
┌──────────────── FASE DE INGESTÃO ────────────────┐
│                                                    │
│  Fonte → Loader → Chunker → Embedder → VectorDB  │
│                                    ↓               │
│                              Metadata Store        │
│                         (source, section, date,   │
│                          type, tenant_id, version) │
└────────────────────────────────────────────────────┘

┌──────────────── FASE DE QUERY ───────────────────┐
│                                                    │
│  Pergunta → [Query Expansion?] → Embed            │
│                ↓                                   │
│         Pre-filter metadata → Search              │
│                ↓                                   │
│         [Rerank?] → Top-k chunks                 │
│                ↓                                   │
│         Context Builder → LLM → Resposta          │
│                              ↓                     │
│                         + Source citations        │
└────────────────────────────────────────────────────┘
```

## Componentes da Ingestão

### 1. Loader — lê a fonte

```python
from langchain.document_loaders import PyPDFLoader, DirectoryLoader, WebBaseLoader

# PDF
loader = PyPDFLoader("docs/manual.pdf")
docs = loader.load()

# Diretório de markdowns
loader = DirectoryLoader("docs/", glob="**/*.md")
docs = loader.load()
```

### 2. Chunker — divide o documento

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,       # tokens (nunca > 512 — invariante)
    chunk_overlap=64,     # sobreposição para não perder contexto na borda
    separators=["\n\n", "\n", " ", ""],
)
chunks = splitter.split_documents(docs)
```

### 3. Embedder — transforma em vetor

```python
from openai import OpenAI

client = OpenAI()

def embed(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-large",  # 3072d — padrão de qualidade
        input=text,
    )
    return response.data[0].embedding
```

### 4. VectorIndexer — armazena com metadata

```python
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct, VectorParams, Distance

client = QdrantClient(url="http://localhost:6333")

# Schema de metadata obrigatório
def build_point(chunk, doc_id: str, chunk_idx: int) -> PointStruct:
    return PointStruct(
        id=f"{doc_id}-{chunk_idx}",
        vector=embed(chunk.page_content),
        payload={
            "source": chunk.metadata.get("source", "unknown"),
            "section": chunk.metadata.get("section", ""),
            "date": chunk.metadata.get("date", ""),
            "type": chunk.metadata.get("type", "document"),
            "tenant_id": chunk.metadata.get("tenant_id", "default"),
            "content": chunk.page_content,      # texto original para grounding
            "chunk_index": chunk_idx,
        }
    )
```

## Componentes da Query

### 1. Pre-filter (sempre antes do semântico)

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

# Filtra antes de buscar — performance + isolamento de tenant
query_filter = Filter(must=[
    FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id)),
    FieldCondition(key="type", match=MatchValue(value="politica")),
    # Opcional: período
    # FieldCondition(key="date", range=Range(gte="2024-01-01")),
])
```

### 2. Search

```python
results = client.search(
    collection_name="docs_privados",
    query_vector=embed(query),
    query_filter=query_filter,
    limit=10,    # busca mais para rerankar depois
    with_payload=True,
)
```

### 3. Reranker (opcional, para coleções > 10k)

```python
import cohere

co = cohere.Client(api_key=os.getenv("COHERE_API_KEY"))

def rerank(query: str, results: list, top_n: int = 3) -> list:
    docs = [r.payload["content"] for r in results]
    reranked = co.rerank(
        query=query,
        documents=docs,
        model="rerank-multilingual-v3.0",
        top_n=top_n,
    )
    return [results[r.index] for r in reranked.results]
```

### 4. Context Builder + grounding obrigatório

```python
def build_context(chunks: list) -> tuple[str, list[str]]:
    context_parts = []
    sources = []
    for i, chunk in enumerate(chunks, 1):
        context_parts.append(f"[{i}] {chunk.payload['content']}")
        sources.append(f"[{i}] {chunk.payload['source']} — {chunk.payload['section']}")
    return "\n\n".join(context_parts), sources

context, sources = build_context(top_chunks)
# LLM recebe context + pergunta → responde citando [1], [2], etc.
```

## Referências
- `chunking-strategies.md` — estratégias de chunking
- `embedding-selection.md` — escolha do modelo
- `indexing-techniques.md` — HNSW, IVF, quantização
- `../patterns/rag-pipeline.md` — pipeline completo com código
- `../patterns/metadata-filtering.md` — pre-filter detalhado
- `reranking.md` — quando e como rerankar
