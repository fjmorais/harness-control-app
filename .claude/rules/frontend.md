---
# Convenções do frontend — adapte os paths: conforme sua estrutura.
paths:
  - "frontend/**"
  - "web/**"
  - "app/**/*.tsx"
  - "app/**/*.ts"
---

# Frontend

UI que consome o backend via proxy/gateway. Simples, tipada, com estados explícitos.

- **Componentes funcionais + hooks.** Sem class components.
- **Tipos explícitos** nas props e no contrato da API. Nada de `any`.
- **Streaming:** se o backend emite SSE, renderize **incrementalmente** conforme os eventos
  chegam; não espere a resposta inteira para exibir.
- **Estados de UI sempre tratados:** carregando, erro, vazio e sucesso — cada um com seu
  componente ou branch de renderização. Nunca deixe a tela "morta" durante uma requisição.
- **Chamadas à API isoladas** em um módulo próprio (`src/api/` ou equivalente), não espalhadas
  pelos componentes.
- **PII na UI:** sempre mascarado via função de display. Nunca exiba dado sensível cru.
  (Ver `rules/seguranca.md`.)
- **Sem secret no bundle.** A única origem de dados é o proxy/gateway na mesma origem; não
  aponte para serviços internos direto no cliente.
