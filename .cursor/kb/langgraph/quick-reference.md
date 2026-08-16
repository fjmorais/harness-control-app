---
domain: langgraph
topic: quick-reference
---

# LangGraph — Quick Reference

### Anatomia de um grafo mínimo

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    query: str
    intent: str
    result: str
    sources: list[str]

graph = StateGraph(AgentState)
graph.add_node("classify", classify_node)
graph.add_node("search", search_node)
graph.add_node("generate", generate_node)

graph.set_entry_point("classify")
graph.add_conditional_edges("classify", route_by_intent, {
    "sql": "run_sql",
    "search": "search",
})
graph.add_edge("search", "generate")
graph.add_edge("generate", END)

app = graph.compile()
```

### Decision tree: grafo determinístico vs ReAct

```
O LLM precisa escolher qual ferramenta usar em loop?
    ├── SIM → ReAct (mas documente o risco: não-determinismo, custo)
    └── NÃO → Grafo determinístico (preferido)
           ├── O fluxo tem desvios baseados em dados?
           │   └── SIM → conditional_edges com função de roteamento
           └── O fluxo é linear?
               └── SIM → add_edge direto
```

### Invariantes (nunca quebrar)

| # | Invariante |
|---|---|
| LG-01 | LLM **não** escolhe tool em loop livre — nó faz a escolha com lógica determinística |
| LG-02 | State schema sempre `TypedDict` tipado — sem dicts genéricos |
| LG-03 | Tools parametrizadas por nó — mesma tool, coleção/tabela diferente por intent |
| LG-04 | Grounding obrigatório em nós geradores — `sources` no state |
| LG-05 | Human-in-the-loop via `interrupt()` — nunca via prompt "espere confirmação" |
