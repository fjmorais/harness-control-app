# Design de Tools no LangGraph

## Princípio: tool parametrizada por nó, não pelo LLM

A mesma tool pode servir a múltiplas coleções/tabelas.
O **nó** decide os parâmetros; a tool executa. O LLM não escolhe a coleção.

```python
# ERRADO: LLM decide qual coleção usar
@tool
def search(query: str, collection: str) -> str:
    """Busca em qualquer coleção."""
    return qdrant.search(collection, query)

# CERTO: nó injeta o parâmetro de coleção
def search_factory(collection: str):
    """Cria uma tool específica para uma coleção — parâmetro fixo."""
    def search(query: str) -> str:
        return qdrant.search(collection, query)
    search.__name__ = f"search_{collection}"
    return search

# No grafo, cada nó usa sua própria instância:
search_docs   = search_factory("documentos_privados")
search_catalog = search_factory("catalogo_produtos")
```

## Contrato de retorno de tool

Toda tool deve retornar um objeto com:
- `result` / `data` — o conteúdo encontrado
- `source` — de onde veio (para grounding)
- `found` — bool indicando se encontrou algo

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolResult:
    found: bool
    data: Any
    source: str      # "sql:tabela_produtos" | "qdrant:col_docs:chunk_id"
    error: str | None = None

    def to_state_update(self) -> dict:
        if not self.found:
            return {"error": f"Não encontrado em {self.source}"}
        return {"sql_result": str(self.data), "sources": [self.source]}
```

## Tool SQL somente-leitura

```python
import asyncpg
from typing import Annotated

ALLOWED_TABLES = frozenset({"produtos", "pedidos", "clientes", "metas"})
MAX_ROWS = 1000
TIMEOUT_SECS = 10.0

async def run_sql(
    query: str,
    params: list,
    table_hint: str,  # para validação de allowlist
) -> ToolResult:
    if table_hint not in ALLOWED_TABLES:
        return ToolResult(found=False, data=None, source=f"sql:{table_hint}",
                         error=f"Tabela não permitida: {table_hint}")
    try:
        async with asyncpg.connect(DSN) as conn:
            rows = await asyncio.wait_for(
                conn.fetch(query, *params),
                timeout=TIMEOUT_SECS,
            )
        return ToolResult(found=bool(rows), data=rows, source=f"sql:{table_hint}")
    except asyncio.TimeoutError:
        return ToolResult(found=False, data=None, source=f"sql:{table_hint}",
                         error="Query timeout")
```

## Tool de busca vetorial

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue

def make_search_tool(collection: str, client: QdrantClient, embedder):
    async def search(
        query: str,
        tenant_id: str,
        periodo: str | None = None,
        top_k: int = 5,
    ) -> ToolResult:
        # pre-filter SEMPRE antes do semântico
        must = [FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))]
        if periodo:
            must.append(FieldCondition(key="periodo_referencia", match=MatchValue(value=periodo)))

        vector = await embedder.embed(query)
        hits = client.search(
            collection_name=collection,
            query_vector=vector,
            query_filter=Filter(must=must),
            limit=top_k,
        )
        if not hits:
            return ToolResult(found=False, data=None, source=f"qdrant:{collection}")

        chunks = [h.payload["content"] for h in hits]
        sources = [f"qdrant:{collection}:{h.id}" for h in hits]
        return ToolResult(found=True, data="\n\n".join(chunks), source="|".join(sources))

    return search
```

## Registrando tools no grafo

Tools são chamadas **dentro do nó**, não como `ToolNode` do LangGraph quando o fluxo é determinístico:

```python
# Nó determinístico — chama a tool diretamente, sem ToolNode
async def run_sql_node(state: AgentState) -> dict:
    sql, params, table = sql_builder.build(state["query"], state["entities"])
    result = await run_sql(sql, params, table)
    if result.error:
        return {"error": result.error}
    return {"sql_result": str(result.data), "sources": result.source.split("|")}

# ToolNode do LangGraph — usar só quando o LLM precisa escolher a tool (ReAct)
# Evitar em grafos determinísticos
```

## Checklist de tool

- [ ] Retorna `ToolResult` com `found`, `data`, `source`
- [ ] Query SQL sempre parametrizada (nunca f-string)
- [ ] Busca vetorial sempre com pre-filter de `tenant_id`
- [ ] Timeout configurável por tool
- [ ] Allowlist de tabelas/coleções validada no início
- [ ] Error capturado e retornado no `ToolResult.error` (sem raise)
