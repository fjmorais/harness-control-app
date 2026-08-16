# Tool search — Busca Vetorial com Pre-Filter

## Implementação completa

```python
from dataclasses import dataclass
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue

@dataclass
class SearchResult:
    found: bool
    chunks: list[str]
    sources: list[str]
    error: str | None = None

def make_search_tool(collection: str, client: QdrantClient, embedder, top_k: int = 5):
    """Fábrica: retorna tool fixada a uma coleção específica."""

    async def search(
        query: str,
        tenant_id: str,
        periodo_referencia: str | None = None,
        doc_type: str | None = None,
    ) -> SearchResult:
        # Pre-filter SEMPRE antes do semântico
        must = [FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))]
        if periodo_referencia:
            must.append(FieldCondition(key="periodo_referencia",
                                       match=MatchValue(value=periodo_referencia)))
        if doc_type:
            must.append(FieldCondition(key="type", match=MatchValue(value=doc_type)))

        try:
            vector = await embedder.aembed_query(query)
            hits = client.search(
                collection_name=collection,
                query_vector=vector,
                query_filter=Filter(must=must),
                limit=top_k,
                with_payload=True,
            )
        except Exception as e:
            return SearchResult(found=False, chunks=[], sources=[],
                                error=f"Erro na busca: {e}")

        if not hits:
            return SearchResult(found=False, chunks=[], sources=[])

        chunks = [h.payload.get("content", "") for h in hits]
        sources = [
            f"qdrant:{collection}:{h.id} (score={h.score:.3f})"
            for h in hits
        ]
        return SearchResult(found=True, chunks=chunks, sources=sources)

    search.__name__ = f"search_{collection}"
    return search
```

## Nó que usa a tool

```python
# Instâncias fixas por coleção — decididas pelo grafo, não pelo LLM
search_docs    = make_search_tool("documentos_privados", qdrant, embedder)
search_catalog = make_search_tool("catalogo_produtos", qdrant, embedder)

async def search_docs_node(state: AgentState) -> dict:
    result = await search_docs(
        query=state["query"],
        tenant_id=state["session_id"],  # ou tenant extraído do JWT
        periodo_referencia=extract_period(state["entities"]),
    )
    if result.error:
        return {"error": result.error}
    if not result.found:
        return {"search_result": "Nenhum documento encontrado.", "sources": []}
    return {
        "search_result": "\n\n---\n\n".join(result.chunks),
        "sources": result.sources,
    }
```

## Roteamento por intent para a coleção correta

```python
INTENT_TO_COLLECTION = {
    "politica":   "documentos_privados",
    "produto":    "catalogo_produtos",
    "relatorio":  "relatorios_analiticos",
}

def route_to_search(state: AgentState) -> str:
    collection = INTENT_TO_COLLECTION.get(state["intent"])
    if not collection:
        return "generate"  # intent sem coleção → gera diretamente
    return f"search_{state['intent']}"  # nó específico por coleção
```

## Context builder com grounding

```python
def build_context(state: AgentState) -> str:
    """Monta o contexto para o LLM — sempre com sources."""
    context_parts = []

    if state.get("search_result"):
        context_parts.append(f"Documentos recuperados:\n{state['search_result']}")

    if state.get("sql_result"):
        context_parts.append(f"Dados do banco:\n{state['sql_result']}")

    if not context_parts:
        return ""

    sources_str = "\n".join(f"- {s}" for s in state.get("sources", []))
    return "\n\n".join(context_parts) + f"\n\nFontes:\n{sources_str}"
```

## Invariantes

| # | Invariante |
|---|---|
| SE-01 | Pre-filter por `tenant_id` SEMPRE presente — nunca busca cross-tenant |
| SE-02 | Coleção fixada na instância da tool — não como parâmetro da query |
| SE-03 | `content` e `source` obrigatórios no payload de todo chunk indexado |
| SE-04 | Sources retornados ao state para grounding no nó generate |
| SE-05 | Filtro por `periodo_referencia`, nunca por `data_ingestao` |
