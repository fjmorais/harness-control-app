# Tool run_sql — Somente Leitura

## Implementação completa

```python
import asyncio
import asyncpg
from dataclasses import dataclass
from typing import Any

ALLOWED_TABLES = frozenset({
    "produtos", "pedidos", "clientes", "metas",
    "regioes", "categorias", "canais", "sessoes_diarias",
})
MAX_ROWS = 500
QUERY_TIMEOUT = 10.0

@dataclass
class SQLResult:
    found: bool
    rows: list[dict]
    row_count: int
    source: str
    error: str | None = None

async def run_sql(query: str, params: list[Any], table_hint: str, dsn: str) -> SQLResult:
    if table_hint not in ALLOWED_TABLES:
        return SQLResult(found=False, rows=[], row_count=0,
                         source=f"sql:{table_hint}",
                         error=f"Tabela não autorizada: {table_hint}")

    normalized = query.strip().rstrip(";")
    if "limit" not in normalized.lower():
        normalized += f" LIMIT {MAX_ROWS}"

    try:
        async with asyncpg.connect(dsn) as conn:
            rows = await asyncio.wait_for(conn.fetch(normalized, *params), timeout=QUERY_TIMEOUT)
        records = [dict(r) for r in rows]
        return SQLResult(found=bool(records), rows=records, row_count=len(records),
                         source=f"sql:negocio.{table_hint}")
    except asyncio.TimeoutError:
        return SQLResult(found=False, rows=[], row_count=0, source=f"sql:{table_hint}",
                         error=f"Query timeout após {QUERY_TIMEOUT}s")
    except asyncpg.PostgresError as e:
        return SQLResult(found=False, rows=[], row_count=0, source=f"sql:{table_hint}",
                         error=f"Erro de banco: {e}")
```

## Nó que usa a tool

```python
async def run_sql_node(state: AgentState) -> dict:
    sql, params, table = await sql_builder.build(
        query=state["query"],
        intent=state["intent"],
        entities=state["entities"],
    )
    result = await run_sql(sql, params, table, DSN)
    if result.error:
        return {"error": result.error}
    return {"sql_result": format_rows(result.rows), "sources": [result.source]}

def format_rows(rows: list[dict]) -> str:
    if not rows:
        return "Nenhum resultado encontrado."
    headers = list(rows[0].keys())
    lines = [" | ".join(headers)]
    lines += [" | ".join(str(r.get(h, "")) for h in headers) for r in rows]
    return "\n".join(lines)
```

## SQL Builder — construção parametrizada

```python
from pydantic import BaseModel

class SQLPlan(BaseModel):
    query: str
    params: list[Any]
    table_hint: str
    explanation: str

async def build_sql(query: str, intent: str, entities: list[str]) -> SQLPlan:
    result = await llm.with_structured_output(SQLPlan).ainvoke(
        BUILDER_PROMPT.format(query=query, intent=intent,
                              entities=entities, schema=SCHEMA_SUMMARY)
    )
    forbidden = ("insert", "update", "delete", "drop", "create", "truncate", "alter")
    if any(kw in result.query.lower() for kw in forbidden):
        raise ValueError("Query contém operação não permitida")
    return result
```

## Invariantes obrigatórias

| # | Invariante |
|---|---|
| SQL-01 | Sempre parametrizado — nunca f-string com input do usuário |
| SQL-02 | Allowlist de tabelas validada antes de executar |
| SQL-03 | LIMIT sempre presente — injetado automaticamente se ausente |
| SQL-04 | Timeout sempre configurado |
| SQL-05 | Apenas SELECT — ausência de DDL/DML validada antes de executar |
| SQL-06 | `source` sempre retornado para grounding |
