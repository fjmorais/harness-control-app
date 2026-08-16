# Roteamento Determinístico

## O padrão

Classificar a intent **antes** de entrar nos nós de ação.
A função de roteamento é código puro — sem LLM, sem I/O.

```python
# Mapeamento fixo: intent → nó
INTENT_ROUTING: dict[str, str] = {
    "sql_aggregate":  "run_sql",
    "sql_lookup":     "run_sql",
    "doc_search":     "search_docs",
    "hybrid":         "search_docs",   # começa pelo semântico
    "greeting":       "generate",      # resposta direta
    "out_of_scope":   "generate",      # recusa educada
}

def route_by_intent(state: AgentState) -> str:
    intent = state.get("intent", "out_of_scope")
    return INTENT_ROUTING.get(intent, "generate")  # fallback seguro

graph.add_conditional_edges("classify", route_by_intent, INTENT_ROUTING)
```

## Nó de classificação

O classificador pode usar LLM, mas a decisão de roteamento é código puro.

```python
from pydantic import BaseModel

class IntentResult(BaseModel):
    intent: str
    confidence: float
    entities: list[str]

VALID_INTENTS = frozenset(INTENT_ROUTING.keys())

def classify_node(state: AgentState) -> dict:
    result: IntentResult = llm.with_structured_output(IntentResult).invoke(
        CLASSIFY_PROMPT.format(query=state["query"])
    )
    # Forçar fallback se intent desconhecido
    intent = result.intent if result.intent in VALID_INTENTS else "out_of_scope"
    return {"intent": intent, "entities": result.entities}
```

## Roteamento multi-nível

Para grafos com sub-rotas:

```python
def route_sql_type(state: AgentState) -> str:
    """Segunda camada de roteamento dentro do fluxo SQL."""
    if "ranking" in state["query"].lower() or "top" in state["query"].lower():
        return "sql_ranking"
    return "sql_aggregate"

graph.add_conditional_edges(
    "run_sql",
    route_sql_type,
    {"sql_ranking": "format_ranking", "sql_aggregate": "format_table"},
)
```

## Roteamento com fallback de erro

```python
def route_after_sql(state: AgentState) -> str:
    if state.get("error"):
        return "handle_error"
    return "generate"

graph.add_conditional_edges(
    "run_sql",
    route_after_sql,
    {"generate": "generate", "handle_error": "handle_error"},
)

def handle_error_node(state: AgentState) -> dict:
    """Gera resposta amigável para o usuário quando a query falhou."""
    return {"answer": f"Não foi possível obter o resultado. Tente reformular a pergunta."}
```

## Visualização do grafo

```python
# Gerar imagem PNG do grafo para documentação
from IPython.display import Image, display
display(Image(app.get_graph().draw_mermaid_png()))

# Ou imprimir em texto
print(app.get_graph().draw_ascii())
```

## Checklist de roteamento determinístico

- [ ] `INTENT_ROUTING` é dict fixo de código — sem LLM decidindo a chave
- [ ] Há fallback explícito para intent desconhecido
- [ ] A função de routing não tem I/O nem side effects
- [ ] Cada intent mapeia para exatamente 1 nó próximo
- [ ] Erros têm rota própria (`handle_error`)
