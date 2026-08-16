# pgvector — RAG sobre Supabase/Postgres

## Quando usar pgvector vs Qdrant

| Critério | pgvector (Supabase) | Qdrant |
|---|---|---|
| Já usa Supabase | ✅ Preferir | — |
| Volume > 1M vetores | — | ✅ Preferir |
| Filtros complexos por metadados | — | ✅ Preferir |
| Projeto novo sem infra extra | ✅ Simples | — |
| Alta throughput de busca | — | ✅ Preferir |

## Setup

```sql
-- Habilitar extensão
CREATE EXTENSION IF NOT EXISTS vector;

-- Tabela de documentos com vetor
CREATE TABLE documentos (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID REFERENCES auth.users(id) NOT NULL,
  content     TEXT NOT NULL,
  embedding   vector(1536),   -- text-embedding-3-small
  -- ou vector(3072)          -- text-embedding-3-large
  metadata    JSONB DEFAULT '{}'::jsonb,
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- Índice HNSW para busca aproximada eficiente
CREATE INDEX ON documentos USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- RLS obrigatório
ALTER TABLE documentos ENABLE ROW LEVEL SECURITY;
CREATE POLICY "usuario_ve_proprios_docs"
ON documentos FOR SELECT USING (user_id = auth.uid());
CREATE POLICY "usuario_insere_proprios_docs"
ON documentos FOR INSERT WITH CHECK (user_id = auth.uid());
```

## Função de busca (match_documents)

```sql
CREATE OR REPLACE FUNCTION match_documents(
  query_embedding vector(1536),
  match_threshold FLOAT DEFAULT 0.7,
  match_count     INT DEFAULT 5,
  filter_user_id  UUID DEFAULT auth.uid()  -- sempre filtra por usuário
)
RETURNS TABLE (
  id        UUID,
  content   TEXT,
  metadata  JSONB,
  similarity FLOAT
)
LANGUAGE sql STABLE
AS $$
  SELECT
    id,
    content,
    metadata,
    1 - (embedding <=> query_embedding) AS similarity
  FROM documentos
  WHERE
    user_id = filter_user_id
    AND 1 - (embedding <=> query_embedding) > match_threshold
  ORDER BY embedding <=> query_embedding
  LIMIT match_count;
$$;
```

## Ingestão de documento (Python)

```python
from openai import AsyncOpenAI
from supabase import create_client

openai = AsyncOpenAI()
supabase = create_client(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)

async def ingest_document(content: str, user_id: str, metadata: dict):
    response = await openai.embeddings.create(
        model="text-embedding-3-small",
        input=content,
    )
    embedding = response.data[0].embedding

    supabase.table("documentos").insert({
        "user_id": user_id,
        "content": content,
        "embedding": embedding,
        "metadata": metadata,
    }).execute()
```

## Query RAG (TypeScript — frontend)

```typescript
const { data: { session } } = await supabase.auth.getSession()

async function searchDocuments(query: string, topK = 5) {
  // 1. Embedar a query (via Edge Function para não expor API key)
  const { data: embedding } = await supabase.functions.invoke("embed", {
    body: { text: query },
  })

  // 2. Buscar documentos similares (RLS garante isolamento por usuário)
  const { data, error } = await supabase.rpc("match_documents", {
    query_embedding: embedding,
    match_threshold: 0.7,
    match_count: topK,
  })

  return data ?? []
}
```

## Edge Function para embedding (protege a API key)

```typescript
// supabase/functions/embed/index.ts
serve(async (req) => {
  const { text } = await req.json()

  const response = await fetch("https://api.openai.com/v1/embeddings", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${Deno.env.get("OPENAI_API_KEY")}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ model: "text-embedding-3-small", input: text }),
  })

  const { data } = await response.json()
  return new Response(JSON.stringify(data[0].embedding), {
    headers: { "Content-Type": "application/json" },
  })
})
```

## Invariantes pgvector

| # | Invariante |
|---|---|
| PGV-01 | RLS habilitado na tabela de vetores — pré-filtro por `user_id` na função |
| PGV-02 | Mesmo modelo de embedding na ingestão e na query |
| PGV-03 | Índice HNSW criado antes de popular (ou recriar após população) |
| PGV-04 | `match_threshold` definido — nunca retornar resultados irrelevantes |
| PGV-05 | API key de embedding nunca exposta no frontend — usar Edge Function |
