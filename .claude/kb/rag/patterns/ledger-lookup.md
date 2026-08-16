# Padrão LEDGER — Busca Exata

## Conceito

LEDGER = livro-razão. Cada entrada é precisa, única e verificável.
O padrão LEDGER trata dados estruturados como o que são: fatos determinísticos que têm exatamente uma resposta correta.

## Quando usar SQL vs KV

| Critério | SQL (Postgres, SQLite) | KV (Redis, DynamoDB) |
|---|---|---|
| Query por campo | `WHERE produto_id = $1` | `GET produto:4521` |
| Joins necessários | SIM | Não (desnormalize) |
| Escrita frequente | Qualquer | Alta frequência (cache) |
| Latência alvo | < 50ms | < 5ms |
| Dados relacionados | SIM | Não (preferência) |
| Padrão no harness | **Padrão** | Cache / lookup de alta freq. |

## Implementação SQL (Postgres)

```python
import asyncpg
from dataclasses import dataclass

@dataclass
class LedgerResult:
    found: bool
    value: dict | None
    source: str  # "sql:tabela.campo" — para grounding

class SQLLedger:
    def __init__(self, dsn: str):
        self._dsn = dsn
        self._pool: asyncpg.Pool | None = None

    async def connect(self):
        self._pool = await asyncpg.create_pool(self._dsn)

    async def lookup(
        self,
        table: str,
        where: dict,
        select: list[str],
    ) -> LedgerResult:
        cols = ", ".join(select)
        conditions = " AND ".join(f"{k} = ${i+1}" for i, k in enumerate(where))
        query = f"SELECT {cols} FROM {table} WHERE {conditions} LIMIT 1"

        async with self._pool.acquire() as conn:
            row = await conn.fetchrow(query, *where.values())

        if row is None:
            return LedgerResult(found=False, value=None, source=f"sql:{table}")

        return LedgerResult(
            found=True,
            value=dict(row),
            source=f"sql:{table}.{','.join(select)}",
        )

# Uso:
ledger = SQLLedger(dsn=os.getenv("DATABASE_URL"))
await ledger.connect()

result = await ledger.lookup(
    table="produtos",
    where={"produto_id": 4521},
    select=["preco", "estoque", "nome"],
)
# result.value = {"preco": 149.90, "estoque": 42, "nome": "Tênis XR-7000"}
# result.source = "sql:produtos.preco,estoque,nome"
```

## Implementação KV (Redis)

```python
import redis.asyncio as redis
import json

class RedisLedger:
    def __init__(self, url: str):
        self._client = redis.from_url(url)

    async def lookup(self, key: str) -> LedgerResult:
        raw = await self._client.get(key)
        if raw is None:
            return LedgerResult(found=False, value=None, source=f"kv:{key}")
        return LedgerResult(
            found=True,
            value=json.loads(raw),
            source=f"kv:{key}",
        )

    async def set(self, key: str, value: dict, ttl: int = 3600):
        await self._client.setex(key, ttl, json.dumps(value))

# Cache de preços com TTL
await redis_ledger.set(f"preco:{produto_id}", {"preco": 149.90}, ttl=300)
result = await redis_ledger.lookup(f"preco:{produto_id}")
```

## Two-Query Pattern (RAG + LEDGER)

Para perguntas que misturam narrativa com dado exato:

```python
async def query_hybrid(question: str, tenant_id: str) -> str:
    """
    Pergunta: "Qual a política de devolução e qual o prazo atual?"
    → RAG: acha a narrativa da política
    → LEDGER: extrai o prazo exato da tabela de config
    """
    # --- Query 1: RAG (parte narrativa) ---
    chunks = await rag_search(
        query="política de devolução",
        tenant_id=tenant_id,
        collection="politicas",
        top_k=2,
    )

    # --- Query 2: LEDGER (dado exato) ---
    prazo = await sql_ledger.lookup(
        table="config_politicas",
        where={"tipo": "devolucao", "tenant_id": tenant_id},
        select=["prazo_dias", "ultima_atualizacao"],
    )

    # --- Combina no context ---
    context = format_chunks(chunks)
    if prazo.found:
        context += f"\n\nDado exato (fonte: {prazo.source}):\n"
        context += f"Prazo de devolução: {prazo.value['prazo_dias']} dias\n"
        context += f"Atualizado em: {prazo.value['ultima_atualizacao']}"

    # --- LLM gera com grounding de AMBAS as fontes ---
    return await llm.generate(context=context, question=question)
```

## Grounding no LEDGER

O LEDGER também precisa de grounding — citar a fonte da resposta exata.

```python
# Template de resposta com grounding duplo:
RESPONSE_TEMPLATE = """
Com base na política documentada [1] e nos dados cadastrais [2]:

{resposta}

Fontes:
[1] {chunk_source} — {chunk_section}
[2] {ledger_source} — consultado em {timestamp}
"""
```

## Segurança (invariante SI)

```python
# SEMPRE usar query parametrizada — nunca interpolação de string
# ERRADO (SQL injection):
query = f"SELECT preco FROM produtos WHERE nome = '{user_input}'"

# CERTO:
query = "SELECT preco FROM produtos WHERE nome = $1"
result = await conn.fetchrow(query, user_input)

# Allowlist de tabelas e colunas — nunca deixar o LLM escolher tabela livre
ALLOWED_TABLES = {"produtos", "politicas", "config_politicas"}
if table not in ALLOWED_TABLES:
    raise ValueError(f"Tabela não permitida: {table}")
```

## Referências
- `../concepts/semantic-vs-exact.md` — quando escolher LEDGER vs RAG
- `../concepts/vector-db-what-not-to-store.md` — o que vai no SQL e não no vetor
- `rag-pipeline.md` — integração RAG + LEDGER no pipeline completo
