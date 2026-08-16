---
# Definição de Pronto (DoD) — carrega sempre (sem paths:)
---

# Definição de Pronto (DoD) + critérios de aceite

Regra **transversal**: define o que conta como "entregue e correto". É o contrato que torna
a execução do agente mensurável — sem ele, "está bom?" é inauditável. O `/scorecard` mede
a entrega contra esta definição.

## Toda task declara critérios de aceite

Critérios **testáveis**, escritos na task antes de implementar. Ex.: "POST /chat responde em
SSE", não "melhorar o chat". O que não é verificável não é critério de aceite.

## Uma task só está PRONTA quando (todos):

1. **Gate verde** — `/validar` (ruff + mypy + pytest) passa sem exceção. Não se comita vermelho.
2. **Rules respeitadas** — nenhum invariante violado (SI, somente-leitura onde declarado, grafo
   determinístico, grounding com fonte, runtime puro).
3. **Revisor aprovou** — `revisor-codigo` com veredito `aprovado` (ou `aprovado com ressalvas`);
   **nenhum bloqueante** em aberto.
4. **Critérios de aceite atendidos** — cada um com evidência (teste que prova, ou demonstração).
5. **ADR registrado** se houve decisão contestável (`docs/adr/`).
6. **Delivery record gravado** em `metrics/entregas.jsonl` (1 linha JSON por task — schema em
   `metrics/README.md`). É o que alimenta o `/scorecard`.

## Sinais lagging (medidos depois)

- **Change-failure:** task reaberta, bug aberto contra task já concluída, ou mudança pedida em
  revisão humana. Cada um conta contra a entrega; registre, não esconda.
- **Autonomia:** task concluída sem edição humana = entrega autônoma. A meta é essa taxa subir.

> Não persiga vaidade (linhas de código, nº de commits). O gate é alvo legítimo (testes reais);
> métricas de revisão humana não entram no que o agente otimiza.
