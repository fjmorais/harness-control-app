---
# Convenções do backend — adapte os paths: conforme sua estrutura.
paths:
  - "backend/app/**"
  - "src/app/**"
  - "src/**/*.py"
---

# Backend

A camada web é fina: **roteia, valida, delega.** A lógica de negócio vive em `services/` e no
código do agente/pipeline — nunca nos routers.

- **Routers finos:** um router só extrai/valida entrada (Pydantic), chama um service e formata
  a saída. Zero regra de negócio ou SQL no router.
- **`services/`** orquestra (chama o agente, persiste artefatos, monta payload). É onde mora a
  lógica de aplicação.
- **Config só via settings** (pydantic-settings ou equivalente). Nunca leia `os.environ` solto
  nem hardcode URL/senha/token. O container injeta as variáveis via `.env`.
- **Async de ponta a ponta:** handlers `async def`, clientes async, sem chamada bloqueante no
  event loop.
- **Erros:** traduza falha de domínio em resposta HTTP com status correto; não vaze stack trace
  nem SQL cru para o cliente.
- **Única porta exposta ao cliente é via gateway/proxy** (nginx ou equivalente) — não publique
  a porta da API direto ao host em produção.
- **Todo run de agente gravado no schema de observabilidade** (`harness/` ou equivalente).
  Persistência de trace é parte do contrato, não opcional.
