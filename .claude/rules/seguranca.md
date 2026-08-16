---
# Segurança da Informação — carrega SEMPRE (sem paths: — regra global)
---

# Segurança da Informação (SI)

Regra **transversal** — vigora em qualquer área do projeto. Quebrar um invariante aqui não é
"escolha de design"; é um risco real de destruição de dados, vazamento de PII ou violação de LGPD.

## Dados de produção — SOMENTE LEITURA por padrão

- O agente **nunca** executa `DELETE`, `DROP`, `TRUNCATE`, `ALTER` ou qualquer DDL destrutivo em
  banco de dados de produção sem aprovação humana explícita + ADR registrado em `docs/adr/`.
- Se o projeto precisar de escrita em produção, declare o role e o escopo no `CLAUDE.md` como
  invariante explícito — sem declaração, o padrão é leitura.
- Banco de desenvolvimento/teste: escrita livre. Banco de produção: gate humano obrigatório.

## PII — Dados Pessoais (LGPD)

Dados pessoais identificam ou podem identificar uma pessoa natural:
CPF, RG, passaporte, email, telefone, nome completo, endereço, IP, localização,
dados biométricos, dados de saúde, dados financeiros, dados de menores.

**Invariantes:**
- PII **nunca** aparece em logs (structured ou plain text).
- PII **nunca** é exposto em URL query strings.
- PII é exibido na UI **somente via função de mascaramento** (ex.: `***-456-***` para CPF).
- O banco armazena PII raw para filtragem — a exibição é sempre mascarada.
- Nunca retorne PII cru em output do LLM sem necessidade declarada.

**Compliance LGPD (Art. 6º):**
- **Finalidade:** o agente só processa PII para o propósito declarado em `CLAUDE.md` (seção SI).
- **Minimização:** use apenas os campos PII estritamente necessários para a tarefa.
- **Transparência:** se o usuário perguntar quais dados são usados, informe com precisão.

## Secrets — nunca hardcoded

- Credentials, API keys, tokens, connection strings: **sempre via variáveis de ambiente** ou
  secret manager. Nunca no código, nunca em arquivos versionados.
- `.env` está no `.gitignore` — nunca o remova do ignore.
- Se um secret vazar no código, trate como incidente: rotacione antes de commitar.

## Operações destrutivas — sempre confirmar com o usuário

Antes de qualquer operação com potencial destrutivo (mesmo que pareça inofensiva):
- `docker compose down -v` · `docker volume rm` · `rm -rf` em dados
- Migrate com `DROP TABLE` · `TRUNCATE` em produção
- Deletar arquivos de configuração ou datasets

**Pare e pergunte.** Não assuma que o usuário quer destruir. O custo de perguntar é zero; o custo
de um dado apagado pode ser irreversível.

## SI no fluxo de desenvolvimento

- O `/novo-projeto` faz um **SI Assessment** na primeira pergunta — responda com precisão.
- Se o projeto lida com PII, declare `[AVISO_LGPD]` no `CLAUDE.md` e crie `rules/pii.md` com as
  regras específicas de mascaramento para os campos do domínio.
- Todo ADR que envolva acesso a dados sensíveis deve ter uma seção "Impacto em SI/LGPD".
- O `revisor-codigo` verifica SI como check bloqueante — sem aprovação de SI, sem merge.
