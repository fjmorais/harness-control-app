# O Que NUNCA Guardar no Banco Vetorial

## A regra

> Banco vetorial armazena **similaridade de significado**.
> Se a pergunta tem uma resposta **única e exata**, a similaridade é irrelevante — e o vetor vai te dar a resposta **errada ou aproximada**.

## Lista de tipos que NÃO vão no vetor

| Tipo | Exemplo | Por quê não funciona no vetor |
|---|---|---|
| **IDs / chaves primárias** | `customer_id = 7821`, `order_id = BR-0055` | `"7821"` e `"7820"` têm distância semântica quase zero |
| **Preços / valores monetários** | `R$ 149,90`, `$ 1.234,00` | `"149,90"` e `"149,00"` são semanticamente idênticos |
| **CPF / CNPJ / documentos** | `123.456.789-00` | Distância semântica entre números não tem significado |
| **Datas e timestamps** | `2024-03-15`, `2024-03-16` | Duas datas próximas são semanticamente próximas também |
| **Saldos / contadores** | `saldo = 1.000,00`, `qtd_pedidos = 42` | Valor exato vs aproximado — o vetor não distingue |
| **Status / enums** | `status = "pending"`, `status = "confirmed"` | Use SQL: `WHERE status = 'pending'` |
| **Flags booleanas** | `is_premium = true` | Filtre no SQL, não no semântico |
| **Versões** | `v2.1.3`, `v2.1.4` | Versões próximas ≠ semanticamente próximas |
| **Dados de autenticação** | tokens, hashes, senhas | Nunca — SI crítico |
| **Dados PII estruturados** | nome + CPF + endereço em tupla | Fragmentar narrativas de PII = risco de re-identificação |

## Por que o vetor falha com dados exatos

```python
# Embedding de valores monetários:
embed("R$ 149,90") → [0.12, -0.34, 0.88, ...]
embed("R$ 149,00") → [0.11, -0.34, 0.88, ...]  # quase idêntico!

# Busca por "R$ 149,90":
# top-1: "R$ 149,00"   distância: 0.001  ← ERRADO mas muito próximo
# top-2: "R$ 149,90"   distância: 0.002  ← correto mas perdeu para o errado

# SQL faz isso em < 1ms sem ambiguidade:
SELECT preco FROM produtos WHERE produto_id = 4521
# Resultado: 149.90  ← exato, determinístico, auditável
```

## O que SIM vai no banco vetorial

| Tipo | Exemplo |
|---|---|
| **Narrativas / explicações** | "Nossa política de devolução estabelece que..." |
| **Manuais e documentação** | Parágrafos de manual técnico |
| **E-mails e comunicados** | Corpo de e-mail de suporte |
| **Avaliações e reviews** | "O produto chegou danificado e o suporte foi..." |
| **Artigos e posts** | Conteúdo editorial não-estruturado |
| **Transcrições** | Transcrição de chamada de suporte |
| **Notas de reunião** | Ata narrativa (não a pauta com IDs) |
| **Descrições de produto** | "Tênis de corrida com amortecimento reativo..." |
| **Perguntas frequentes** | FAQ em formato texto corrido |

## Armadilha comum: metadata vs conteúdo

Metadata no payload do vetor é OK — é só um filtro, não é buscado semanticamente.

```python
# CORRETO: metadata de identificação no payload (não vira vetor)
PointStruct(
    vector=embed("Nossa política de reembolso..."),   # ← texto corrido → OK
    payload={
        "tenant_id": "empresa-a",    # ← ID no payload → OK (só para filtrar)
        "doc_id": "POL-001",         # ← ID no payload → OK
        "date": "2024-01-15",        # ← data no payload → OK (filtrar por período)
        "content": "Nossa política...",  # ← texto original → OK (para grounding)
    }
)

# ERRADO: dado exato como conteúdo do chunk que vai ser embedado
PointStruct(
    vector=embed("preço: R$ 149,90"),  # ← NUNCA! dado exato vira vetor
    payload={"preco": 149.90}
)
```

## Checklist antes de indexar

- [ ] O dado tem uma resposta única e exata? → **SQL/KV, não vetor**
- [ ] É um número, ID, data ou status? → **SQL/KV, não vetor**
- [ ] A busca precisa de exatidão (não aproximação)? → **SQL/KV, não vetor**
- [ ] É texto narrativo com significado contextual? → **Vetor, OK**
- [ ] A mesma informação pode ser expressa de formas diferentes? → **Vetor, OK**

## Referências
- `semantic-vs-exact.md` — decision tree completo
- `../patterns/ledger-lookup.md` — padrão LEDGER com código
