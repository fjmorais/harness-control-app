# Arquitetura Medallion

## O que é

Padrão de organização de dados em camadas progressivas de qualidade.
Cada camada tem uma responsabilidade única e imutável.

## As 4 camadas

### Raw (Zona de Pouso)
- **Responsabilidade**: cópia exata da fonte, imutável, nunca transformada
- **Nunca fazer**: transformar, filtrar, renomear, deletar
- **Formato**: preservar o original (CSV, JSON, Parquet, etc.)
- **Colunas obrigatórias adicionadas**: `_source_file`, `_ingest_ts`
- **Retenção**: permanente (é o backup de disaster recovery)

### Bronze (Dados Brutos Tipados)
- **Responsabilidade**: tipagem explícita + metadados de lineage. Sem lógica de negócio.
- **Nunca fazer**: aplicar regras de negócio, filtrar registros "ruins"
- **Formato**: Delta Lake (ou Parquet com schema enforcement)
- **Colunas obrigatórias**: `_ingested_at`, `_source`, `_run_id`, `_batch_id`
- **Schema evolution**: configurado aqui (merge/quarantine/fail)

### Silver (Dados Limpos e Validados)
- **Responsabilidade**: deduplicação, validação, limpeza, enriquecimento básico
- **Nunca fazer**: agregar, fazer join com tabelas de negócio externas ao domínio
- **Formato**: Delta Lake com schema enforcement estrito
- **Colunas obrigatórias**: `_processed_at`, `_pipeline_version`
- **Qualidade**: todas as validações de negócio passam aqui

### Gold (Agregações de Negócio)
- **Responsabilidade**: modelos finais para BI, ML ou APIs downstream
- **Nunca fazer**: lógica de limpeza (isso é Silver), transformações brutas
- **Formato**: Delta Lake ou tabelas otimizadas para o consumidor
- **Colunas obrigatórias**: `_updated_at`
- **Acesso**: somente leitura para o agente (nunca escrever via LLM)

## Regras invariantes (nunca quebrar)

1. **Raw é sagrado** — nunca modifique, nunca delete, nunca filtre
2. **Cada camada tem uma responsabilidade** — sem "atalhos" (ex: raw→gold direto)
3. **Lineage columns sempre** — toda camada tem suas colunas de rastreamento
4. **Schema enforcement desde Bronze** — dados sem schema = dados sem contrato
5. **Gold = somente leitura para o agente** — nunca `INSERT/UPDATE/DELETE` em Gold via LLM

## Exemplo de fluxo

```python
# Raw: copiar da fonte sem transformar
raw_df = spark.read.csv(source_path)
raw_df = raw_df.withColumn("_source_file", lit(source_path)) \
               .withColumn("_ingest_ts", current_timestamp())
raw_df.write.mode("append").parquet(config.raw_path())

# Bronze: tipar + adicionar lineage
bronze_df = raw_df.select(
    col("id").cast("string"),
    col("amount").cast("decimal(10,2)"),
    col("created_at").cast("timestamp"),
    # lineage
    lit(run_id).alias("_run_id"),
    lit(batch_id).alias("_batch_id"),
    current_timestamp().alias("_ingested_at"),
    lit(source_name).alias("_source"),
)
bronze_df.write.mode("append").saveAsTable(config.bronze_table("orders"))

# Silver: limpar + validar
silver_df = bronze_df \
    .dropDuplicates(["id"]) \
    .filter(col("amount") > 0) \
    .withColumn("_processed_at", current_timestamp()) \
    .withColumn("_pipeline_version", lit("1.0.0"))
silver_df.write.mode("overwrite").saveAsTable(config.silver_table("orders"))
```
