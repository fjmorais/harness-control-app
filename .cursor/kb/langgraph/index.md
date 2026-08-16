---
domain: langgraph
description: Grafos determinísticos com LangGraph — state, nós, tools, human-in-the-loop, multi-agent
mcp_validated: "2026-06-27"
confidence: 0.92
---

# KB: LangGraph

Base de conhecimento para construção de agentes com grafos **determinísticos** usando LangGraph.
O princípio central: o grafo decide o fluxo; o LLM decide apenas dentro dos nós — nunca o fluxo.

## Conceitos

| Arquivo | Tópico |
|---|---|
| [graph-anatomy.md](concepts/graph-anatomy.md) | StateGraph, nós, edges, entry point, compilação |
| [state-management.md](concepts/state-management.md) | TypedDict de state, reducers, schema de mensagens |
| [tool-design.md](concepts/tool-design.md) | Tools parametrizadas por coleção/tabela, contrato de retorno |
| [human-in-the-loop.md](concepts/human-in-the-loop.md) | interrupt(), checkpointers, aprovação humana |

## Padrões

| Arquivo | Tópico |
|---|---|
| [deterministic-routing.md](patterns/deterministic-routing.md) | Roteamento por intent classificado — sem loop livre do LLM |
| [run-sql-tool.md](patterns/run-sql-tool.md) | Tool somente-leitura com allowlist + LIMIT + timeout |
| [search-tool.md](patterns/search-tool.md) | Tool de busca vetorial com pre-filter de metadados |
| [interrupt-pattern.md](patterns/interrupt-pattern.md) | Pausa, coleta aprovação, retoma — com MemorySaver |

## Quick Reference

Ver [quick-reference.md](quick-reference.md) — anatomia de grafo mínimo, decision tree
determinístico vs ReAct, invariantes (LG-01…LG-05). Ler só se a tarefa exigir esse nível de detalhe.
