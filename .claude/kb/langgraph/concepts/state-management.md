# Gerenciamento de State no LangGraph

## TypedDict — o contrato do state

O state é um `TypedDict` — tipado, versionado, documentado.
Nunca use `dict` genérico: perde type-checking e documentação implícita.

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # --- Input (imutável após entrada) ---
    query: str              # pergunta original do usuário
    session_id: str         # identificador da sessão

    # --- Classificação (preenchido por classify_node) ---
    intent: str             # "sql_query" | "doc_search" | "hybrid"
    entities: list[str]     # entidades extraídas da query

    # --- Execução (preenchido pelos nós de ação) ---
    sql_result: str         # resultado bruto da query SQL
    search_result: str      # trechos recuperados do Qdrant
    sources: list[str]      # [{"source": "...", "section": "..."}]

    # --- Output (preenchido por generate_node) ---
    answer: str             # resposta final ao usuário
    messages: Annotated[list, add_messages]  # histórico de mensagens

    # --- Controle ---
    error: str | None       # erro capturado (None = sem erro)
    requires_approval: bool # flag para human-in-the-loop
```

## Reducers

Para campos que **acumulam** valores em vez de sobrescrever, use `Annotated` com reducer.

```python
from operator import add

class AgentState(TypedDict):
    # add_messages: acumula, deduplica por id, mantém ordem
    messages: Annotated[list, add_messages]

    # add: concatena listas (ex: múltiplos resultados de busca)
    search_chunks: Annotated[list[str], add]

    # sem reducer: sobrescreve (comportamento padrão)
    intent: str
    answer: str
```

Quando usar cada um:
| Campo | Reducer | Razão |
|---|---|---|
| `messages` | `add_messages` | histórico de chat acumulado |
| `search_chunks` | `add` | múltiplos nós de busca paralelos |
| `intent`, `answer`, `error` | nenhum (sobrescreve) | valor único por run |

## Updates parciais

Nós retornam **somente o que mudou** — o LangGraph faz merge automático:

```python
# Classify só retorna intent e entities — não toca nos outros campos
def classify_node(state: AgentState) -> dict:
    intent, entities = classifier.run(state["query"])
    return {
        "intent": intent,
        "entities": entities,
    }

# Search só retorna search_result e sources
def search_node(state: AgentState) -> dict:
    chunks, sources = qdrant_search(state["query"], state["intent"])
    return {
        "search_result": "\n\n".join(chunks),
        "sources": sources,
    }
```

## State por thread (stateful)

Com checkpointer, cada `thread_id` tem seu próprio state persistido:

```python
from langgraph.checkpoint.memory import MemorySaver

app = graph.compile(checkpointer=MemorySaver())

# Thread A — conversa do usuário 1
config_a = {"configurable": {"thread_id": "user-1-session-abc"}}
app.invoke({"query": "..."}, config=config_a)

# Thread B — conversa do usuário 2 (state independente)
config_b = {"configurable": {"thread_id": "user-2-session-xyz"}}
app.invoke({"query": "..."}, config=config_b)

# Recuperar state atual de um thread
snapshot = app.get_state(config_a)
print(snapshot.values["messages"])
```

## Inicialização do state

Sempre forneça todos os campos no invoke inicial — mesmo os que são `None` ou `[]`:

```python
initial_state = {
    "query": user_query,
    "session_id": session_id,
    "intent": "",
    "entities": [],
    "sql_result": "",
    "search_result": "",
    "sources": [],
    "answer": "",
    "messages": [],
    "error": None,
    "requires_approval": False,
}
result = app.invoke(initial_state, config=config)
```

## Campos de controle recomendados

| Campo | Tipo | Uso |
|---|---|---|
| `error` | `str \| None` | propaga erros entre nós sem exceção |
| `requires_approval` | `bool` | sinaliza para o interrupt pattern |
| `retry_count` | `int` | contador de retentativas (para retry com backoff) |
| `debug_trace` | `list[str]` | log interno de decisões (desativar em produção) |
