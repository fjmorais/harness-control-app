# Query Expansion — HyDE, Multi-Query e Rewriting

## Por que expandir a query

O maior problema do RAG: a pergunta do usuário raramente usa as mesmas palavras que o documento.

```
Usuário pergunta: "como cancelo meu plano?"
Documento diz:   "Procedimento de rescisão contratual..."

→ "cancelo meu plano" e "rescisão contratual" são semanticamente próximos,
  mas a distância pode ser suficiente para não recuperar o documento certo em
  coleções com muito "ruído semântico"
```

## HyDE — Hypothetical Document Embeddings

**Ideia:** Em vez de embedar a pergunta (que é curta e informal), pedir ao LLM para gerar uma **resposta fictícia** e embedar essa resposta. Respostas se parecem mais com documentos.

```python
from openai import OpenAI

client = OpenAI()

def hyde_expand(question: str) -> str:
    """Gera hipotética resposta para embedar no lugar da pergunta."""
    response = client.chat.completions.create(
        model="claude-haiku-4-5-20251001",  # modelo barato — só geração
        messages=[
            {
                "role": "system",
                "content": (
                    "Você é um assistente técnico. Gere uma resposta curta e factual "
                    "para a pergunta abaixo, como se fosse extraída de um manual. "
                    "Não mencione que é uma resposta hipotética. "
                    "Responda em 2-3 frases, em português."
                ),
            },
            {"role": "user", "content": question},
        ],
        max_tokens=150,
    )
    return response.choices[0].message.content

# Uso:
question = "como cancelo meu plano?"
hypothetical = hyde_expand(question)
# → "Para cancelar seu plano, acesse Configurações > Assinatura > Cancelar.
#    O cancelamento é efetivo no fim do período de cobrança vigente..."

q_vec = embed(hypothetical)  # vetor da resposta hipotética, não da pergunta
results = qdrant.search(collection_name=..., query_vector=q_vec, ...)
```

**Quando HyDE vale o custo (1 chamada LLM extra):**
- Queries muito curtas ou informais ("como cancelo?")
- Coleção com linguagem técnica/formal diferente do usuário
- Recall atual < 70% nas queries de teste

**Quando NÃO usar:**
- Queries já longas e específicas ("procedimento de rescisão contratual para planos premium")
- Latência crítica < 100ms (HyDE adiciona ~300-500ms)
- Coleção pequena (< 500 docs) — sem problemas de recall

## Multi-Query

**Ideia:** Gerar N reformulações da pergunta, buscar com cada uma, fundir com RRF.

```python
def multi_query_expand(question: str, n: int = 3) -> list[str]:
    """Gera N variações da pergunta para busca."""
    response = client.chat.completions.create(
        model="claude-haiku-4-5-20251001",
        messages=[
            {
                "role": "system",
                "content": (
                    f"Gere {n} reformulações diferentes da pergunta do usuário. "
                    "Cada reformulação deve capturar a mesma intenção com palavras diferentes. "
                    f"Retorne apenas as {n} perguntas, uma por linha, sem numeração."
                ),
            },
            {"role": "user", "content": question},
        ],
        max_tokens=200,
    )
    lines = response.choices[0].message.content.strip().split("\n")
    return [q.strip() for q in lines if q.strip()][:n]

def multi_query_search(
    question: str,
    tenant_id: str,
    collection: str,
    n_queries: int = 3,
    limit_per_query: int = 10,
) -> list:
    """Busca com múltiplas queries e funde com RRF."""
    queries = [question] + multi_query_expand(question, n_queries - 1)

    all_results: list[list] = []
    seen_ids: set = set()

    for q in queries:
        q_vec = embed(q)
        results = qdrant.search(
            collection_name=collection,
            query_vector=q_vec,
            query_filter=Filter(must=[
                FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))
            ]),
            limit=limit_per_query,
        )
        ids = [r.id for r in results]
        all_results.append(ids)
        seen_ids.update(ids)

    # RRF fusion
    rrf_scores: dict[str, float] = {}
    for result_list in all_results:
        for rank, doc_id in enumerate(result_list, start=1):
            rrf_scores[doc_id] = rrf_scores.get(doc_id, 0) + 1.0 / (60 + rank)

    # Ordena por score RRF
    sorted_ids = sorted(rrf_scores, key=rrf_scores.get, reverse=True)

    # Recupera os pontos na ordem correta
    return qdrant.retrieve(
        collection_name=collection,
        ids=sorted_ids[:limit_per_query],
        with_payload=True,
    )
```

**Quando multi-query vale:**
- Pergunta ambígua com múltiplas interpretações possíveis
- Coleção muito grande onde uma busca não é suficiente
- Queries de usuários com vocabulário diverso

## Query Rewriting (pré-processamento simples)

Antes de qualquer expansão, limpar a pergunta:

```python
import re

def clean_query(question: str) -> str:
    """Remove ruído antes de embedar."""
    # Remove saudações
    question = re.sub(
        r'^(olá|oi|bom dia|boa tarde|boa noite|tudo bem)[,!.\s]+',
        '',
        question,
        flags=re.IGNORECASE,
    )
    # Remove pontuação excessiva
    question = re.sub(r'[!?]+', '?', question)
    # Colapsa espaços
    question = re.sub(r'\s+', ' ', question).strip()
    return question

# Uso:
raw = "Oi! Como eu cancelo meu plano??? Urgente!!!"
clean = clean_query(raw)
# → "Como eu cancelo meu plano?"
```

## Decisão: qual técnica usar?

| Situação | Técnica |
|---|---|
| Query curta e informal | **HyDE** |
| Query ambígua / múltiplas interpretações | **Multi-query** |
| Coleção formal, query informal | **HyDE** |
| Query com saudação ou ruído | **Rewriting** (sempre) |
| Alto recall necessário, latência tolerada | **Multi-query** |
| Latência < 200ms | Nenhuma expansão |
| Combinação ótima | Rewriting → HyDE → Reranking |

## Referências
- `rag-pipeline.md` — onde inserir a expansão no pipeline
- `../concepts/reranking.md` — combinar com reranking para máximo ganho
- `../concepts/hybrid-search.md` — alternativa via BM25 para queries com termos exatos
