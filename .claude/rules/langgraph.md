---
paths:
  - "**/agent/**"
  - "**/graph/**"
  - "**/langgraph/**"
  - "**/nodes/**"
  - "**/tools/**"
---

# Regras de LangGraph — Invariantes de Grafo Determinístico

Esta rule carrega ao tocar qualquer arquivo de agente, grafo, nó ou tool LangGraph.
São invariantes — nunca flexibilizar sem ADR e aprovação humana explícita.

## Os 5 invariantes

### LG-01 — Grafo determinístico: LLM decide dentro de nós, nunca o fluxo

O LLM pode classificar, extrair e gerar **dentro de um nó**.
O LLM **não** decide qual tool usar em laço livre (ReAct) nem qual nó chamar a seguir.

```python
# ERRADO: LLM decide a tool em loop
agent = create_react_agent(llm, tools=[search, run_sql, send_email])

# CERTO: nó classifica, lógica decide o próximo nó
def classify_node(state): return {"intent": classifier.run(state["query"])}
def route(state) -> str: return INTENT_TO_NODE[state["intent"]]
graph.add_conditional_edges("classify", route, INTENT_TO_NODE)
```

### LG-02 — State schema sempre TypedDict tipado

Nunca usar `dict` genérico como state. TypedDict garante type-checking e documenta o contrato.

```python
# ERRADO
state = {}  # ou dict genérico

# CERTO
from typing import TypedDict
class AgentState(TypedDict):
    query: str
    intent: str
    result: str
    sources: list[str]
    error: str | None
```

### LG-03 — Tools parametrizadas pelo nó, não pelo LLM

A mesma tool serve múltiplas coleções/tabelas. O **nó** injeta os parâmetros fixos.
O LLM não escolhe `collection` nem `table` como parâmetro livre.

```python
# ERRADO: LLM escolhe a coleção
@tool
def search(query: str, collection: str) -> str: ...  # LLM pode passar qualquer coleção

# CERTO: nó fixa a coleção via factory
search_docs    = make_search_tool("documentos_privados", client, embedder)
search_catalog = make_search_tool("catalogo_produtos", client, embedder)
```

### LG-04 — Grounding obrigatório em nós geradores

Todo nó que gera resposta ao usuário deve incluir `sources` no state e citá-los na resposta.
Resposta sem fonte = alucinação não auditável.

```python
# OBRIGATÓRIO no nó de geração
GENERATE_PROMPT = """
Responda com base APENAS nas fontes abaixo.
Se não houver informação suficiente, diga que não encontrou.
NÃO invente dados.

Contexto: {context}
Fontes: {sources}
Pergunta: {query}
"""

def generate_node(state: AgentState) -> dict:
    answer = llm.invoke(GENERATE_PROMPT.format(
        context=state["search_result"] or state["sql_result"],
        sources="\n".join(state["sources"]),
        query=state["query"],
    ))
    return {"answer": answer.content}
```

### LG-05 — Human-in-the-loop via interrupt(), nunca via prompt

Para ações que requerem aprovação humana, usar `interrupt_before` + checkpointer.
Nunca simular aprovação com "o usuário aprovou?" no prompt.

```python
# ERRADO: pseudo-aprovação via prompt
def bad_node(state):
    llm.invoke("O usuário aprovou a ação? Responda sim/não")

# CERTO: interrupt real com checkpointer
app = graph.compile(checkpointer=MemorySaver(), interrupt_before=["execute_action"])
# retomar após aprovação real:
result = await app.ainvoke(None, config={"configurable": {"thread_id": thread_id}})
```

---

## Regras de segurança para tools

```python
# SEMPRE: query SQL parametrizada + allowlist + LIMIT + timeout
ALLOWED_TABLES = frozenset({"produtos", "pedidos", "clientes"})

async def run_sql(query: str, params: list, table: str) -> ToolResult:
    assert table in ALLOWED_TABLES, f"Tabela não autorizada: {table}"
    # nunca: f"SELECT * FROM {table} WHERE {user_input}"
```

```python
# SEMPRE: busca vetorial com pre-filter de tenant
must = [FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))]
# nunca confiar apenas na similaridade semântica para isolamento
```

## Referências

- `.claude/kb/langgraph/index.md` — visão geral e quick reference
- `.claude/kb/langgraph/concepts/graph-anatomy.md` — LG-01 e LG-02
- `.claude/kb/langgraph/concepts/tool-design.md` — LG-03
- `.claude/kb/langgraph/concepts/human-in-the-loop.md` — LG-05
- `.claude/kb/langgraph/patterns/deterministic-routing.md` — LG-01 em código
