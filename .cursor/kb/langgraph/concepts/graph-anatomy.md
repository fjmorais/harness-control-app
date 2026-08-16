# Anatomia do Grafo LangGraph

## Componentes fundamentais

### StateGraph

Contêiner do grafo. Recebe o `TypedDict` de state no construtor.

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # reducer de mensagens
    query: str
    intent: str
    result: str
    sources: list[str]
    error: str | None

graph = StateGraph(AgentState)
```

### Nós (Nodes)

Funções que recebem state e retornam **dict parcial** (só os campos que mudam).

```python
def classify_node(state: AgentState) -> dict:
    intent = classify_intent(state["query"])
    return {"intent": intent}  # retorna apenas o que mudou

def generate_node(state: AgentState) -> dict:
    answer = llm.invoke(build_prompt(state["result"], state["sources"]))
    return {"messages": [AIMessage(content=answer.content)]}
```

Regras para nós:
- Retornar apenas os campos que o nó modifica — não o state completo
- Sem efeitos colaterais externos (I/O, DB) nos nós de decisão — só nos nós de ação
- Nome do nó deve descrever a ação: `classify`, `search_docs`, `run_sql`, `generate`

### Edges (Arestas)

**Diretas** — fluxo linear sem decisão:
```python
graph.add_edge("search", "generate")
graph.add_edge("generate", END)
```

**Condicionais** — roteamento baseado em dados do state:
```python
def route_by_intent(state: AgentState) -> str:
    """Retorna o nome do próximo nó — determinístico, sem LLM."""
    match state["intent"]:
        case "sql_query":   return "run_sql"
        case "doc_search":  return "search_docs"
        case "hybrid":      return "search_docs"  # começa pelo semântico
        case _:             return "generate"      # fallback

graph.add_conditional_edges(
    "classify",
    route_by_intent,
    {
        "run_sql":     "run_sql",
        "search_docs": "search_docs",
        "generate":    "generate",
    }
)
```

### Entry point e compilação

```python
graph.set_entry_point("classify")  # nó inicial

# Sem checkpointer (stateless)
app = graph.compile()

# Com checkpointer (stateful — necessário para human-in-the-loop)
from langgraph.checkpoint.memory import MemorySaver
app = graph.compile(checkpointer=MemorySaver())
```

### Invocação

```python
# Síncrono
result = app.invoke({"query": "quais produtos mais venderam?", "intent": "", "result": "", "sources": [], "error": None})

# Streaming por evento
for event in app.stream({"query": "..."}, stream_mode="values"):
    print(event)

# Com thread (stateful)
config = {"configurable": {"thread_id": "sessao-123"}}
result = app.invoke({"query": "..."}, config=config)
```

## Anti-padrões

```python
# ERRADO: LLM decide qual tool usar
def bad_node(state):
    tool_choice = llm.invoke("qual ferramenta usar para: " + state["query"])
    return run_tool(tool_choice)  # não-determinístico, inseguro

# CERTO: nó classifica, lógica decide
def classify_node(state):
    intent = classifier.classify(state["query"])  # determinístico
    return {"intent": intent}

def route(state) -> str:
    return INTENT_MAP[state["intent"]]  # mapeamento fixo
```

## Diagrama de fluxo típico

```
entrada → [classify] → route_by_intent
                           ├── "sql_query"   → [run_sql]    → [generate] → END
                           ├── "doc_search"  → [search_docs] → [generate] → END
                           └── "hybrid"      → [search_docs] → [run_sql]  → [generate] → END
```
