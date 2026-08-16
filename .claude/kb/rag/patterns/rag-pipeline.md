# Pipeline RAG Completo — Código Python

## Estrutura SOLID do pipeline

```
rag/
├── ingest/
│   ├── loader.py          ← DocumentLoader (lê fonte)
│   ├── chunker.py         ← TextChunker (divide)
│   ├── embedder.py        ← Embedder (OpenAI)
│   └── indexer.py         ← VectorIndexer (Qdrant)
├── query/
│   ├── expander.py        ← QueryExpander (HyDE, multi-query)
│   ├── searcher.py        ← VectorSearch + pre-filter
│   ├── reranker.py        ← Reranker (Cohere ou local)
│   └── generator.py       ← ContextBuilder + LLM call
├── config.py              ← RAGConfig dataclass
└── pipeline.py            ← RAGPipeline (orquestra)
```

## RAGConfig

```python
from dataclasses import dataclass, field
import os

@dataclass
class RAGConfig:
    qdrant_url: str = field(default_factory=lambda: os.getenv("QDRANT_URL", "http://localhost:6333"))
    collection_name: str = "docs_privados"
    embedding_model: str = "text-embedding-3-large"
    embedding_dim: int = 3072
    chunk_size: int = 512       # tokens — invariante: nunca > 512
    chunk_overlap: int = 64
    top_k_retrieve: int = 20    # candidatos para reranking
    top_k_final: int = 3        # resultado final
    use_reranking: bool = True
    use_hybrid: bool = False    # sparse + dense
    openai_api_key: str = field(default_factory=lambda: os.getenv("OPENAI_API_KEY", ""))
    cohere_api_key: str = field(default_factory=lambda: os.getenv("COHERE_API_KEY", ""))

    @classmethod
    def from_env(cls) -> "RAGConfig":
        return cls()
```

## Fase 1 — Ingestão

```python
# rag/ingest/pipeline.py
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, Distance, PointStruct,
    PayloadSchemaType,
)
from langchain.text_splitter import RecursiveCharacterTextSplitter
from openai import OpenAI
import uuid

class RAGIngestor:
    def __init__(self, config: RAGConfig):
        self.config = config
        self.qdrant = QdrantClient(url=config.qdrant_url)
        self.openai = OpenAI(api_key=config.openai_api_key)
        self.splitter = RecursiveCharacterTextSplitter(
            chunk_size=config.chunk_size,
            chunk_overlap=config.chunk_overlap,
        )

    def ensure_collection(self):
        existing = [c.name for c in self.qdrant.get_collections().collections]
        if self.config.collection_name not in existing:
            self.qdrant.create_collection(
                collection_name=self.config.collection_name,
                vectors_config=VectorParams(
                    size=self.config.embedding_dim,
                    distance=Distance.COSINE,
                ),
            )
            # Indexes obrigatórios para pre-filter eficiente
            for field in ["tenant_id", "type", "doc_type"]:
                self.qdrant.create_payload_index(
                    collection_name=self.config.collection_name,
                    field_name=field,
                    field_schema=PayloadSchemaType.KEYWORD,
                )

    def embed(self, text: str) -> list[float]:
        resp = self.openai.embeddings.create(
            model=self.config.embedding_model,
            input=text[:8000],  # limite de segurança
        )
        return resp.data[0].embedding

    def ingest_document(
        self,
        text: str,
        metadata: dict,  # obrigatório: source, type, tenant_id, date
    ) -> int:
        """Retorna número de chunks indexados."""
        self.ensure_collection()

        chunks = self.splitter.split_text(text)
        points = []

        for i, chunk in enumerate(chunks):
            point_id = str(uuid.uuid4())
            vector = self.embed(chunk)
            points.append(PointStruct(
                id=point_id,
                vector=vector,
                payload={
                    **metadata,
                    "content": chunk,
                    "chunk_index": i,
                    "total_chunks": len(chunks),
                },
            ))

        self.qdrant.upsert(
            collection_name=self.config.collection_name,
            points=points,
        )
        return len(chunks)
```

## Metadata schema obrigatório

```python
# Todo chunk indexado DEVE ter:
REQUIRED_METADATA = {
    "source": str,       # "docs/manual.pdf", "email://msg-001", "wiki://pagina"
    "section": str,      # "Capítulo 3 / Devolução" (vazio "" se não houver)
    "date": str,         # "2024-06-01" — data do documento (não de ingestão)
    "type": str,         # "manual" | "politica" | "faq" | "email" | "contrato"
    "tenant_id": str,    # "empresa-a" — obrigatório em multi-tenant
    "content": str,      # texto original do chunk (para grounding)
    "chunk_index": int,  # posição no documento original
}
```

## Fase 2 — Query

```python
# rag/query/pipeline.py
from qdrant_client.models import Filter, FieldCondition, MatchValue

class RAGQuerier:
    def __init__(self, config: RAGConfig):
        self.config = config
        self.qdrant = QdrantClient(url=config.qdrant_url)
        self.openai = OpenAI(api_key=config.openai_api_key)

    def embed(self, text: str) -> list[float]:
        resp = self.openai.embeddings.create(
            model=self.config.embedding_model,
            input=text,
        )
        return resp.data[0].embedding

    def search(
        self,
        query: str,
        tenant_id: str,
        doc_type: str | None = None,
    ) -> list:
        q_vec = self.embed(query)

        # Pre-filter: SEMPRE antes do semântico
        must_conditions = [
            FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))
        ]
        if doc_type:
            must_conditions.append(
                FieldCondition(key="type", match=MatchValue(value=doc_type))
            )

        results = self.qdrant.search(
            collection_name=self.config.collection_name,
            query_vector=q_vec,
            query_filter=Filter(must=must_conditions),
            limit=self.config.top_k_retrieve,
            with_payload=True,
        )
        return results

    def build_context(self, chunks: list) -> tuple[str, list[str]]:
        parts = []
        sources = []
        for i, chunk in enumerate(chunks, 1):
            p = chunk.payload
            parts.append(f"[{i}] {p['content']}")
            src = p.get("source", "desconhecido")
            sec = p.get("section", "")
            sources.append(f"[{i}] {src}" + (f" — {sec}" if sec else ""))
        return "\n\n".join(parts), sources

    def query(
        self,
        question: str,
        tenant_id: str,
        doc_type: str | None = None,
    ) -> tuple[str, list[str]]:
        """Retorna (context, sources) para passar ao LLM."""
        candidates = self.search(question, tenant_id, doc_type)

        if self.config.use_reranking and len(candidates) > self.config.top_k_final:
            from rag.query.reranker import cohere_rerank
            top = cohere_rerank(
                question, candidates,
                top_n=self.config.top_k_final,
                api_key=self.config.cohere_api_key,
            )
        else:
            top = candidates[:self.config.top_k_final]

        return self.build_context(top)
```

## Uso completo

```python
config = RAGConfig.from_env()
ingestor = RAGIngestor(config)
querier = RAGQuerier(config)

# Ingestão
n = ingestor.ingest_document(
    text=manual_text,
    metadata={
        "source": "docs/manual-produto.pdf",
        "section": "Políticas",
        "date": "2024-01-15",
        "type": "manual",
        "tenant_id": "empresa-a",
    },
)

# Query
context, sources = querier.query(
    question="Qual o procedimento de devolução?",
    tenant_id="empresa-a",
    doc_type="manual",
)

resposta = llm.generate(
    system="Responda com base no contexto. Cite as fontes com [n].",
    context=context,
    question=question,
)
```

## Referências
- `../concepts/rag-architecture.md` — visão de componentes
- `../concepts/chunking-strategies.md` — estratégias de chunking
- `../concepts/reranking.md` — reranking detalhado
- `metadata-filtering.md` — pre-filter avançado
- `query-expansion.md` — HyDE e multi-query
