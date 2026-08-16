---
# Regras de pipeline de dados — carrega ao tocar código de pipeline.
# Remova este arquivo se o projeto não for de pipeline de dados.
paths:
  - "src/pipeline/**"
  - "pipelines/**"
  - "dags/**"
  - "notebooks/**"
---

# Pipeline de Dados — Medallion + SOLID + Contratos

Regras para projetos de engenharia de dados. Gerado automaticamente pelo `/harness-architect`
quando o tipo de projeto for `pipeline`. Adapte conforme a stack real do projeto.

## Arquitetura Medallion — invariantes por camada

| Camada | Responsabilidade | NUNCA fazer | Colunas obrigatórias |
|---|---|---|---|
| **Raw** | Cópia exata da fonte, imutável | Transformar, deletar, sobrescrever | `_source_file`, `_ingest_ts` |
| **Bronze** | Dados brutos tipados + metadata | Filtros de negócio, agregações | `_ingested_at`, `_source`, `_run_id`, `_batch_id` |
| **Silver** | Limpos, tipados, deduplicados, validados | Agregações de negócio | `_processed_at`, `_pipeline_version` |
| **Gold** | Agregações e modelos de negócio | Lógica de limpeza | `_updated_at` |

**Raw é sagrado:** nunca modifique ou delete dados da camada Raw.

## SOLID para pipelines

- **S** — Cada classe de transform faz UMA coisa: `BronzeIngestor`, `SilverTransformer`, `GoldAggregator`.
- **O** — `BaseTransform` é extensível sem modificar: herdar e implementar `transform()`.
- **L** — Qualquer `BaseTransform` é substituível sem quebrar `Pipeline.run()`.
- **I** — `Readable` e `Writable` são interfaces separadas; não force implementar os dois.
- **D** — `Pipeline(reader, transform, writer)` depende de abstrações, não de Spark/Delta concreto.

## Data contracts

- Cada fonte de dados para o pipeline tem um contrato em `data-contracts/` (YAML ou Pydantic).
- O contrato declara: schema, SLA de freshness, producer/consumer, regras de qualidade.
- **Violação de contrato = não passar silenciosamente.** Registre na quarantine e notifique o owner.

## Schema evolution

- **mergeSchema:** aceite apenas para adições não-breaking (novas colunas opcionais).
- **quarantine + notify:** para mudanças potencialmente breaking (tipo alterado, coluna removida).
- **fail:** para mudanças que violam o contrato declarado.
- Toda evolução aceita gera um ADR em `docs/adr/`.

## Quarantine

- Registros com schema inválido ou regras de qualidade violadas vão para
  `{catalog}.bronze.quarantine` (ou equivalente na stack local).
- Quarantine inclui: dados originais + `_quarantine_reason` + `_quarantine_ts`.
- Taxa de quarantine > [THRESHOLD]% dispara notificação automática ao owner.

## Configuração centralizada

- **NUNCA hardcode** catalog, schema, workspace URL, table names.
- Tudo via `PipelineConfig` carregado de `config/environments/{PIPELINE_ENV}.yaml`.
- Mudar de DEV para PRD = mudar `PIPELINE_ENV=prd`. Zero edição de código.

## Observabilidade obrigatória

- Log estruturado por step: `pipeline`, `layer`, `run_id`, `records_in`, `records_out`,
  `records_quarantined`, `duration_ms`. Sem PII em logs. (Ver `rules/seguranca.md`.)
- Notificação ao owner via webhook (Slack/Teams/email) para: schema mismatch, alta taxa de
  quarantine, SLA breach, falha de step.

## Linhagem

- Toda tabela gerada pelo pipeline contém as colunas de lineage obrigatórias (ver Medallion acima).
- Em Databricks: Unity Catalog rastreia lineage automaticamente — não desabilite.
- Em local/Airflow: adicione `_pipeline_version` e `_run_id` para rastrear manualmente.
