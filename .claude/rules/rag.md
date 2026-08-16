---
paths:
  - "**/rag/**"
  - "**/retrieval/**"
  - "**/search/**"
  - "**/vector*/**"
  - "**/embedding*/**"
  - "**/qdrant*/**"
  - "**/ledger*/**"
---

# Regras de RAG e Busca — Invariantes

Esta rule carrega ao tocar qualquer arquivo de busca, RAG, retrieval ou embedding.
São invariantes — nunca flexibilizar sem ADR e aprovação humana explícita.

## Os 10 invariantes

### RAG-01 — Banco vetorial = apenas texto corrido

Nunca indexar no banco vetorial: IDs, preços, CPF, CNPJ, datas, saldos, contadores, enums, tokens de autenticação, qualquer campo com resposta determinística única.

Banco vetorial armazena **similaridade de significado**. Dado exato não tem similaridade — tem exatidão. Misturar degrada a qualidade de ambos.

### RAG-02 — Dado exato → SQL/KV (padrão LEDGER)

Qualquer campo com resposta única e verificável vai para SQL ou KV.
Nunca usar busca semântica como substituto de `WHERE campo = $1`.

```python
# ERRADO
results = qdrant.search(query_vector=embed("preço do produto 4521"))

# CERTO
cursor.execute("SELECT preco FROM produtos WHERE produto_id = $1", [4521])
```

### RAG-03 — Pre-filter SEMPRE antes da busca semântica

Filtrar por `tenant_id`, `type`, `status` **antes** da busca vetorial — nunca depois.

Razão dupla: (1) performance — reduz espaço de busca; (2) segurança — isolamento de tenant via semântica é inseguro (documentos similares podem vazar entre tenants).

```python
# Obrigatório
query_filter = Filter(must=[
    FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))
])
```

### RAG-04 — Grounding obrigatório

Toda resposta gerada a partir de RAG DEVE citar o(s) chunk(s) fonte(s) recuperado(s).
Resposta sem grounding = alucinação não auditável = falha de qualidade.

O campo `content` no payload e o campo `source` são obrigatórios em todo chunk indexado.

### RAG-05 — Collection escolhida pelo nó (determinístico)

Em grafos LangGraph, o nó do grafo decide qual collection usar com base na intenção classificada.
O LLM **não** escolhe a collection em laço livre — isso introduz não-determinismo e risco de segurança.

```python
# ERRADO: LLM decide collection
collection = llm.decide("qual collection usar para essa query?")

# CERTO: mapeamento determinístico no nó
INTENT_TO_COLLECTION = {"politica": "docs_privados", "produto": "catalogo"}
collection = INTENT_TO_COLLECTION[state["intent"]]
```

### RAG-06 — Chunk size máximo 512 tokens

Chunk acima de 512 tokens dilui o sinal semântico — o vetor representa a média de muitos conceitos e nenhum representa bem.
Overlap de 10-15% para não perder contexto na borda.

### RAG-07 — Metadata schema versionado

Tratar o schema de metadata dos chunks como contrato de dados (igual ao `data-contracts.md`).
Campos obrigatórios em todo chunk: `source`, `section`, `date`, `type`, `tenant_id`, `content`, `chunk_index`.
Mudança de schema = ADR + migração dos pontos existentes.

### RAG-08 — Tenant isolation via pre-filter (SI crítico)

Em sistemas multi-tenant, **nunca** depender só da similaridade semântica para isolamento.
Se documento do Tenant A e documento do Tenant B são semanticamente próximos, a busca pode cruzar tenants sem pre-filter.

Payload index em `tenant_id` é obrigatório. Criar no momento de criação da collection.

### RAG-09 — Reranking para coleções > 10k documentos

Top-k retrieval sem reranking em coleções grandes = precisão degradada.
Usar Cohere Rerank API (produção) ou cross-encoder local (self-hosted) para coleções acima de 10k docs.
Buscar mais candidatos (ex: 20) e rerankar para top-k final (ex: 3).

### RAG-10 — HyDE ou multi-query para queries ambíguas

Não assumir que a pergunta literal do usuário é o melhor vetor de busca.
Queries curtas, informais ou ambíguas se beneficiam de expansão.
Usar HyDE (resposta hipotética) ou multi-query (N reformulações) + RRF para merge.

Custo: 1 chamada LLM extra. Vale o custo quando recall atual < 70%.

---

## Query SQL no LEDGER: segurança obrigatória

```python
# SEMPRE parametrizado — nunca interpolação de string (SQL injection)
cursor.execute("SELECT preco FROM produtos WHERE id = %s", [user_input])

# SEMPRE allowlist de tabelas e colunas
ALLOWED_TABLES = {"produtos", "politicas", "config"}
assert table in ALLOWED_TABLES, f"Tabela não permitida: {table}"
```

## Payload indexes obrigatórios ao criar collection

```python
for field in ["tenant_id", "type", "status", "date"]:
    client.create_payload_index(collection_name=..., field_name=field, ...)
```

Sem índice de payload: filtro = scan completo = performance O(n). Com índice: O(log n).

## Referências

- `.claude/kb/rag/index.md` — índice completo
- `.claude/kb/rag/concepts/semantic-vs-exact.md` — RAG-01 e RAG-02
- `.claude/kb/rag/patterns/metadata-filtering.md` — RAG-03 e RAG-08
- `.claude/kb/rag/patterns/ledger-lookup.md` — RAG-02
