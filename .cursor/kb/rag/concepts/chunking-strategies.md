# Estratégias de Chunking

## Por que chunking importa

Um chunk muito grande dilui o sinal semântico — o vetor média muitos conceitos e nenhum representa bem.
Um chunk muito pequeno perde contexto — a resposta precisa de mais do que um parágrafo isolado.

**Regra: nunca exceder 512 tokens por chunk.** Overlap de 10-15% para não perder contexto na borda.

## Estratégias

### 1. Fixed-size (baseline)

Divide por número fixo de caracteres, independente de fronteiras de parágrafo.

```python
from langchain.text_splitter import CharacterTextSplitter

splitter = CharacterTextSplitter(
    chunk_size=2000,    # caracteres (~500 tokens)
    chunk_overlap=200,  # sobreposição para contexto na borda
    separator="\n",
)
```

**Quando usar:** Dados homogêneos sem estrutura clara (logs, texto plano).
**Problema:** Corta frases no meio — contexto perdido na borda.

### 2. Recursive Character Splitter (padrão para a maioria dos casos)

Tenta dividir em hierarquia de separadores: `\n\n` → `\n` → ` ` → `""`.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,      # tokens
    chunk_overlap=64,    # ~10% de sobreposição
    separators=["\n\n", "\n", " ", ""],
    length_function=len,
)
chunks = splitter.split_documents(docs)
```

**Quando usar:** Texto em prosa, manuais, artigos, e-mails.
**Vantagem:** Respeita parágrafos, raramente corta frases.

### 3. Document-aware (respeita estrutura markdown/HTML)

Para documentos com estrutura hierárquica (headers, seções).

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

headers_to_split = [
    ("#", "H1"),
    ("##", "H2"),
    ("###", "H3"),
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split)
chunks = splitter.split_text(markdown_text)

# Cada chunk herda o header como metadata:
# chunk.metadata = {"H1": "Políticas", "H2": "Reembolso"}
```

**Quando usar:** Documentação técnica, wikis, bases de conhecimento em markdown.
**Vantagem:** Metadata de seção automático → melhor pre-filtering e grounding.

### 4. Semantic Chunking

Usa embeddings para identificar fronteiras de mudança de tema.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    OpenAIEmbeddings(model="text-embedding-3-small"),  # small = mais rápido
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95,
)
chunks = splitter.split_text(text)
```

**Quando usar:** Documentos longos com múltiplos temas misturados.
**Custo:** Mais lento (chama embedding durante a ingestão para cada sentença).
**Vantagem:** Chunks coesos por tema — melhor recall.

## Guidelines por tipo de documento

| Tipo de documento | Estratégia recomendada | Chunk size | Overlap |
|---|---|---|---|
| FAQ | Document-aware (H2 = pergunta) | 256-512 tokens | 0-32 |
| Manual técnico | Document-aware | 512 tokens | 64 |
| Artigo / narrativa | Recursive + Semantic | 384-512 tokens | 64 |
| E-mail / suporte | Recursive | 256-384 tokens | 32 |
| Contrato / jurídico | Recursive (parágrafo como unidade) | 512 tokens | 64 |
| Código-fonte | Recursive (separador `\nclass\|def`) | 512 tokens | 0 |
| Transcrição de call | Recursive (por turno de fala) | 256 tokens | 32 |

## Metadata obrigatório em todo chunk

```python
{
    "source": "docs/manual-produto.pdf",   # origem
    "section": "Capítulo 3 / Devolução",   # seção (header)
    "date": "2024-06-01",                  # data do documento
    "type": "manual",                      # tipo para pre-filter
    "tenant_id": "empresa-a",              # isolamento multi-tenant
    "chunk_index": 12,                     # posição no doc original
    "content": "texto original do chunk",  # para grounding
}
```

## Anti-padrões

```python
# ERRADO: chunk gigante (> 512 tokens)
splitter = CharacterTextSplitter(chunk_size=8000)
# → Vetor representa média de muitos conceitos → busca imprecisa

# ERRADO: chunk sem overlap em texto narrativo
splitter = CharacterTextSplitter(chunk_overlap=0)
# → Frases cortadas na borda perdem contexto → resposta incompleta

# ERRADO: ignorar metadata de seção
# → Sem "section" no payload, grounding fica sem referência de onde veio
```

## Referências
- `rag-architecture.md` — onde chunking se encaixa no pipeline
- `embedding-selection.md` — modelo de embedding alinhado com chunk size
