# Human-in-the-Loop no LangGraph

## Por que e quando usar

Human-in-the-loop (HITL) pausa o grafo para aprovação humana antes de ações irreversíveis:
- Executar query que modifica dados
- Enviar comunicação externa (e-mail, Slack, webhook)
- Tomar decisão de negócio de alto impacto
- Qualquer ação além da capacidade de leitura/análise do agente

## Pré-requisito: checkpointer

Sem checkpointer, o state não persiste entre invocações — o interrupt não funciona.

```python
from langgraph.checkpoint.memory import MemorySaver  # dev/test
# Em produção: usar SqliteSaver ou PostgresSaver
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer, interrupt_before=["execute_action"])
```

`interrupt_before`: lista de nós que devem pausar *antes* de executar.
`interrupt_after`: lista de nós que devem pausar *após* executar (para revisão do output).

## Padrão com interrupt_before

```python
from langgraph.types import interrupt

def plan_action_node(state: AgentState) -> dict:
    """Planeja a ação e sinaliza para aprovação."""
    action_plan = build_action_plan(state["query"], state["sql_result"])
    return {
        "action_plan": action_plan,
        "requires_approval": True,
    }

def execute_action_node(state: AgentState) -> dict:
    """Executa só após aprovação — nunca chamado diretamente."""
    result = execute(state["action_plan"])
    return {"execution_result": str(result), "requires_approval": False}

# Grafo com interrupt
graph.add_node("plan_action", plan_action_node)
graph.add_node("execute_action", execute_action_node)
graph.add_edge("plan_action", "execute_action")

app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute_action"],  # pausa aqui
)
```

## Fluxo completo de aprovação

```python
import asyncio

async def run_with_approval(query: str, thread_id: str):
    config = {"configurable": {"thread_id": thread_id}}

    # 1. Rodar até o interrupt
    result = await app.ainvoke({"query": query, ...}, config=config)

    # 2. Verificar se pausou (result será None ou parcial dependendo da versão)
    snapshot = app.get_state(config)
    if snapshot.next:  # há nós pendentes
        # Mostrar o plano ao usuário
        plan = snapshot.values.get("action_plan", "")
        print(f"Plano proposto:\n{plan}")

        # 3. Coletar aprovação
        approved = await collect_human_approval()  # sua lógica de UI/webhook

        if approved:
            # 4a. Retomar a execução
            final = await app.ainvoke(None, config=config)
            return final
        else:
            # 4b. Cancelar — atualizar state e encerrar
            app.update_state(config, {"error": "Ação cancelada pelo operador"})
            return {"error": "Cancelado"}

    return result
```

## update_state — edição manual do state

Permite ao humano corrigir valores antes de retomar:

```python
# Corrigir o plano antes de executar
app.update_state(
    config,
    {"action_plan": plano_corrigido_pelo_humano},
    as_node="plan_action",  # atribui ao nó correto no histórico
)

# Retomar após correção
result = app.invoke(None, config=config)
```

## Streaming com interrupt

```python
async for event in app.astream(initial_state, config=config, stream_mode="values"):
    if "__interrupt__" in event:
        # O grafo pausou — mostrar ao usuário e aguardar input
        interruption = event["__interrupt__"][0]
        print(f"Aguardando aprovação: {interruption.value}")
        break
    # processar evento normal
    process_event(event)
```

## Tabela de decisão

| Situação | Estratégia |
|---|---|
| Ação irreversível (write, send) | `interrupt_before=["execute_node"]` |
| Revisão do output antes de mostrar | `interrupt_after=["generate_node"]` |
| Aprovação condicional (só para valores altos) | `interrupt()` dentro do nó com condicional |
| Multi-step com aprovação em cada etapa | `interrupt_before` em múltiplos nós |

## Anti-padrões

```python
# ERRADO: usar prompt como "aprovação"
def bad_node(state):
    answer = llm.invoke("O usuário aprovou? Responda sim/não")
    if "sim" in answer.lower():
        execute()  # não é aprovação real

# CERTO: interrupt + update_state real
app = graph.compile(checkpointer=checkpointer, interrupt_before=["execute"])
# ... fluxo com get_state / invoke(None) documentado acima
```
