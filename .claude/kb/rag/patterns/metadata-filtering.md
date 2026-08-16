# Metadata Filtering — Pre-filter antes do Semântico

## Por que pre-filter é crítico

Sem pre-filter, a busca semântica varre **toda** a coleção. Com pre-filter:

1. **Performance:** reduz drasticamente o espaço de busca
2. **Precisão:** elimina ruído de outros tenants/tipos/períodos
3. **Segurança (SI):** **isolamento de tenant via semântica é INSEGURO** — a busca pode cruzar tenants se os documentos forem semanticamente próximos

## Campos obrigatórios no metadata

```python
MANDATORY_FIELDS = {
    "tenant_id": str,          # Isolamento multi-tenant — CRÍTICO
    "type": str,               # Tipo de documento para filtro por contexto
    "date": str,               # Data do documento (não de ingestão)
    "source": str,             # Origem para grounding
    "content": str,            # Texto original para grounding
}

RECOMMENDED_FIELDS = {
    "section": str,            # Seção/capítulo para contexto de grounding
    "doc_version": str,        # Versão do documento (schema versionado)
    "language": str,           # "pt-BR", "en-US"
    "status": str,             # "ativo" | "obsoleto" | "rascunho"
    "periodo_referencia": str, # "2024-Q1" — para análises temporais
}
```

## Pre-filter no Qdrant

```python
from qdrant_client.models import (
    Filter, FieldCondition, MatchValue, MatchAny,
    Range, DatetimeRange,
)
from datetime import datetime

# Filtro básico (tenant + tipo)
basic_filter = Filter(must=[
    FieldCondition(key="tenant_id", match=MatchValue(value="empresa-a")),
    FieldCondition(key="type", match=MatchValue(value="manual")),
])

# Filtro por período (documentos de 2024)
period_filter = Filter(must=[
    FieldCondition(key="tenant_id", match=MatchValue(value="empresa-a")),
    FieldCondition(
        key="date",
        range=Range(gte="2024-01-01", lte="2024-12-31"),
    ),
])

# Filtro múltiplos tipos (OR)
multi_type_filter = Filter(must=[
    FieldCondition(key="tenant_id", match=MatchValue(value="empresa-a")),
    FieldCondition(
        key="type",
        match=MatchAny(any=["manual", "politica", "faq"]),
    ),
])

# Filtro composto (must AND should)
composite_filter = Filter(
    must=[
        FieldCondition(key="tenant_id", match=MatchValue(value="empresa-a")),
    ],
    should=[
        FieldCondition(key="type", match=MatchValue(value="manual")),
        FieldCondition(key="type", match=MatchValue(value="politica")),
    ],
    must_not=[
        FieldCondition(key="status", match=MatchValue(value="obsoleto")),
    ],
)
```

## Payload indexes — performance crítica

Sem índice de payload, cada filtro faz scan completo. **Criar índices no momento de criação da collection.**

```python
from qdrant_client.models import PayloadSchemaType

def setup_collection(client, collection_name: str, vector_size: int):
    client.create_collection(
        collection_name=collection_name,
        vectors_config=VectorParams(size=vector_size, distance=Distance.COSINE),
    )

    # Campos de filtro frequente → índice obrigatório
    indexed_fields = [
        ("tenant_id", PayloadSchemaType.KEYWORD),
        ("type", PayloadSchemaType.KEYWORD),
        ("status", PayloadSchemaType.KEYWORD),
        ("date", PayloadSchemaType.DATETIME),
        ("language", PayloadSchemaType.KEYWORD),
    ]
    for field_name, field_schema in indexed_fields:
        client.create_payload_index(
            collection_name=collection_name,
            field_name=field_name,
            field_schema=field_schema,
        )
```

## Padrão de uso no grafo LangGraph

O nó de busca **decide** qual filtro aplicar (determinístico, não LLM):

```python
class SearchNode:
    """Nó do grafo — escolhe collection e filtros de forma determinística."""

    def __call__(self, state: AgentState) -> AgentState:
        intent = state["intent"]  # classificado pelo nó anterior

        # Mapeamento determinístico: intenção → collection + filtros
        collection, doc_types = self._resolve_collection(intent)

        filter_ = Filter(must=[
            FieldCondition(key="tenant_id", match=MatchValue(value=state["tenant_id"])),
            FieldCondition(key="type", match=MatchAny(any=doc_types)),
            FieldCondition(key="status", match=MatchValue(value="ativo")),
        ])

        results = qdrant.search(
            collection_name=collection,
            query_vector=embed(state["query"]),
            query_filter=filter_,
            limit=state["config"].top_k_retrieve,
        )

        return {**state, "retrieved_chunks": results}

    def _resolve_collection(self, intent: str) -> tuple[str, list[str]]:
        # Collection escolhida pelo nó — não pelo LLM em laço livre (invariante)
        mapping = {
            "politica": ("docs_privados", ["politica", "manual"]),
            "produto": ("catalogo", ["descricao", "faq"]),
            "suporte": ("tickets", ["email", "transcricao"]),
        }
        return mapping.get(intent, ("docs_privados", ["manual"]))
```

## Anti-padrões de isolamento de tenant

```python
# ERRADO — depender só do semântico para isolamento
results = qdrant.search(
    collection_name="docs_todos_tenants",
    query_vector=embed("política de cancelamento"),
    limit=5,
    # Sem filtro de tenant_id!
    # → pode retornar docs de outros tenants se forem semanticamente próximos
)

# CERTO — pre-filter obrigatório
results = qdrant.search(
    collection_name="docs_todos_tenants",
    query_vector=embed("política de cancelamento"),
    query_filter=Filter(must=[
        FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))
    ]),
    limit=5,
)
```

## Referências
- `rag-pipeline.md` — integração no pipeline de query
- `../concepts/rag-architecture.md` — posição do pre-filter no fluxo
- `../rules/rag.md` — invariantes de SI para multi-tenant
