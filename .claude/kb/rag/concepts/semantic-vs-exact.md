# Busca Semântica vs Busca Exata (LEDGER)

## O problema fundamental

Existem dois tipos radicalmente diferentes de informação:

| Tipo | Exemplo | Característica |
|---|---|---|
| **Narrativa / texto corrido** | "Nossa política de reembolso permite até 30 dias..." | Significado emerge do contexto |
| **Dado exato / estruturado** | `refund_days = 30`, `customer_id = 7821` | Uma e apenas uma resposta correta |

Usar banco vetorial para dados exatos é como usar Google Translate para fazer aritmética — a ferramenta errada para o trabalho.

## Busca Semântica (RAG)

**Quando usar:**
- A resposta requer compreensão de contexto, explicação, comparação
- O usuário pode formular a pergunta de mil formas diferentes
- A informação existe em texto não-estruturado (manuais, políticas, relatórios, emails)
- Documentos privados que não existem nos dados de treino do LLM

**Como funciona:**
- Documento → chunks → embed (vetor) → índice HNSW
- Query → embed → similaridade coseno → top-k chunks → LLM gera resposta com grounding

**Exemplos:**
- "Explique a política de cancelamento"
- "O que os clientes dizem sobre o produto X?"
- "Qual o procedimento para onboarding de novo cliente?"

## Busca Exata — Padrão LEDGER

**Quando usar:**
- A resposta é única e verificável
- Campo tem tipo primitivo (number, date, enum, ID, CPF)
- A pergunta é "qual é o valor de X?" não "me explique X"

**Como funciona:**
- SQL: `SELECT preco FROM produtos WHERE produto_id = $1`
- KV: `redis.get(f"preco:{produto_id}")`
- Resposta é determinística, auditável, sem alucinação possível

**Exemplos:**
- "Qual o preço do produto 4521?" → SQL
- "Quantos pedidos o cliente 9910 fez?" → SQL
- "Qual o status do pedido #BR-2024-0055?" → SQL/KV
- "Qual o saldo atual?" → SQL

## Decision Tree

```
Pergunta recebida
       │
       ▼
É uma pergunta com resposta única e determinística?
       │
      SIM ──► LEDGER (SQL/KV)
       │         Exemplos: preço, ID, data, saldo, CPF, status enum
       │
      NÃO
       │
       ▼
A informação existe só em texto não-estruturado?
       │
      SIM ──► RAG (banco vetorial)
       │         Exemplos: política, manual, relato, narrativa
       │
      NÃO (misto)
       │
       ▼
Two-query pattern: RAG acha o doc, LEDGER extrai o dado exato
```

## Anti-padrões (nunca fazer)

```python
# ERRADO — dado exato em banco vetorial
qdrant.upsert(collection="produtos", points=[
    PointStruct(
        id=1,
        vector=embed("R$ 149,90"),  # preço vira vetor??
        payload={"produto_id": 4521, "preco": 149.90}
    )
])

# Por quê é errado:
# 1. "R$ 149,90" e "R$ 149,00" têm distância semântica quase zero — busca retorna ambos
# 2. Não há garantia de exatidão — pode retornar produto parecido, não o exato
# 3. Custo de embedding desnecessário para algo que SQL faz em < 1ms

# CERTO — preço no SQL, apenas a narrativa no vetor
cursor.execute("SELECT preco FROM produtos WHERE produto_id = %s", (4521,))
preco = cursor.fetchone()["preco"]  # R$ 149,90 — exato, determinístico
```

## O two-query pattern (busca mista)

Para perguntas que misturam os dois mundos:

```python
# Pergunta: "Qual a política de troca e qual o prazo vigente?"
# → "política de troca" = semântico | "prazo vigente" = LEDGER

# Step 1: RAG → acha o contexto narrativo
chunks = qdrant.search(
    collection="politicas",
    query_vector=embed("política de troca"),
    query_filter=Filter(must=[FieldCondition(key="tipo", match=MatchValue(value="troca"))]),
    limit=3,
)

# Step 2: LEDGER → extrai o valor exato
cursor.execute("SELECT prazo_dias FROM config_politicas WHERE tipo = 'troca'")
prazo = cursor.fetchone()["prazo_dias"]

# Step 3: LLM combina
context = format_chunks(chunks) + f"\n\nPrazo atual: {prazo} dias (fonte: config_politicas)"
resposta = llm.generate(prompt=context, question=pergunta)
```

## Referências
- `../patterns/ledger-lookup.md` — padrão LEDGER completo
- `../patterns/rag-pipeline.md` — pipeline RAG completo
- `vector-db-what-not-to-store.md` — lista completa do que não vai no vetor
