# Padrão interrupt — Aprovação Humana

## Estrutura mínima

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict

class AgentState(TypedDict):
    query: str
    action_plan: str
    result: str
    approved: bool
    error: str | None

# Nós
def plan_node(state: AgentState) -> dict:
    plan = build_plan(state["query"])
    return {"action_plan": plan}

def execute_node(state: AgentState) -> dict:
    result = execute(state["action_plan"])
    return {"result": result}

def generate_node(state: AgentState) -> dict:
    answer = llm.invoke(PROMPT.format(result=state["result"]))
    return {"answer": answer.content}

# Grafo com interrupt_before no nó de execução
graph = StateGraph(AgentState)
graph.add_node("plan", plan_node)
graph.add_node("execute", execute_node)
graph.add_node("generate", generate_node)
graph.set_entry_point("plan")
graph.add_edge("plan", "execute")
graph.add_edge("execute", "generate")
graph.add_edge("generate", END)

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer, interrupt_before=["execute"])
```

## Uso em API (FastAPI)

```python
from fastapi import FastAPI
import uuid

api = FastAPI()
pending: dict[str, dict] = {}  # em produção: Redis ou DB

@api.post("/chat")
async def chat(body: dict):
    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}

    # Primeira invocação — roda até o interrupt
    await app.ainvoke(
        {"query": body["query"], "action_plan": "", "result": "", "approved": False, "error": None},
        config=config,
    )

    # Verificar se pausou
    snapshot = app.get_state(config)
    if snapshot.next:
        plan = snapshot.values["action_plan"]
        pending[thread_id] = config
        return {"status": "awaiting_approval", "thread_id": thread_id, "plan": plan}

    return {"status": "done", "answer": snapshot.values.get("answer")}


@api.post("/approve/{thread_id}")
async def approve(thread_id: str, body: dict):
    config = pending.get(thread_id)
    if not config:
        return {"error": "Thread não encontrado"}

    if not body.get("approved"):
        app.update_state(config, {"error": "Ação cancelada pelo operador"})
        del pending[thread_id]
        return {"status": "cancelled"}

    # Retomar — passa None como input quando retomando
    result = await app.ainvoke(None, config=config)
    del pending[thread_id]
    return {"status": "done", "answer": result.get("answer")}
```

## Múltiplos pontos de interrupt

```python
# Pausar em dois nós distintos
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute_write"],   # antes de escrever
    interrupt_after=["generate"],         # após gerar (para revisão humana do output)
)
```

## Checklist

- [ ] `checkpointer` configurado antes de compilar
- [ ] `interrupt_before` ou `interrupt_after` declarado na compilação
- [ ] `thread_id` único por sessão/conversa
- [ ] `invoke(None, config=config)` para retomar (não passar state novamente)
- [ ] `get_state(config).next` para verificar se o grafo pausou
- [ ] Timeout para aprovação: limpar `pending` após X minutos sem resposta
