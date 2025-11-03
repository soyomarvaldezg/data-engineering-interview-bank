# Azure Data Stack para Data Engineering

**Tags**: #cloud #azure #synapse #data-factory #blob-storage #real-interview  
**Empresas**: Microsoft, JPMorgan, Goldman Sachs, Enterprise (Fortune 500)  
**Dificultad**: Mid  
**Tiempo estimado**: 20 min  

---

## TL;DR

**Azure data stack**: Blob Storage (S3 equivalent) → Data Factory (managed ETL) → Synapse Analytics (warehouse). **Key advantage**: Enterprise integration (Microsoft Stack: Office, Teams, Active Directory). **Trade-off**: Powerful but complex, best for enterprise. Best for: Large corporations, Microsoft shops, strict governance.

---

## Concepto Core

- **Qué es**: Azure = Microsoft's cloud. Diferencia: empresa-oriented, security/compliance strong
- **Por qué importa**: Growing in enterprises (50% Fortune 500 use Azure)
- **Principio clave**: "Enterprise data platform" (governance, security first)

---

## Arquitectura

┌──────────────────────────────────────┐
│ DATA SOURCES │
│ ├─ SQL Server (on-prem) │
│ ├─ Dynamics 365 │
│ ├─ SharePoint │
│ └─ External APIs │
└───────────────┬──────────────────────┘
│
▼
┌──────────────────────────────────────┐
│ AZURE INGESTION │
│ ├─ Data Factory pipelines │
│ ├─ Event Hubs (streaming) │
│ ├─ Azure Data Share (partner data) │
│ └─ Synapse Pipelines │
└───────────────┬──────────────────────┘
│
▼
┌──────────────────────────────────────┐
│ AZURE DATA LAKE (ADLS) │
│ ├─ Blob Storage (gen2) │
│ ├─ Hierarchical namespace │
│ └─ ACL security (granular) │
└───────────────┬──────────────────────┘
│
┌───────────┴───────────┐
│ │
▼ ▼
┌─────────────┐ ┌──────────────┐
│ Data Factory│ │ Synapse │
│ (Managed │ │ Spark pools │
│ ETL) │ │ (Spark/PySpark
└──────┬──────┘ └──────┬───────┘
│ │
└────────┬───────────┘
│
▼
┌──────────────────────────────────────┐
│ SYNAPSE ANALYTICS (Warehouse) │
│ ├─ SQL pools (MPP warehouse) │
│ ├─ Serverless SQL pools │
│ └─ Data Science (notebooks) │
└───────────────┬──────────────────────┘
│
┌───────────┴───────────┐
│ │
▼ ▼
┌──────────────┐ ┌──────────────┐
│ Power BI │ │ Analysts │
│ (MS native) │ │ SQL queries │
└──────────────┘ └──────────────┘

text

---

## Key Services

### 1. Azure Blob Storage (Data Lake Storage Gen2)

What: Object storage with hierarchical namespace
Why: ACL security, integrates with Data Factory/Synapse
Cost: Similar to AWS S3

Pricing:

Storage: $0.018/GB/month (cheap)

Transactions: $0.004 per 10k operations

Data transfer: $0.08/GB out (mid)

Architecture:
abfss://container@storageaccount.dfs.core.windows.net/path/file

Features:

Hierarchical namespace (actual folders)

ACL (granular permissions)

Lifecycle policies (archive)

Benefit vs AWS S3:

Real folders (not just key prefixes)

Better for data lake (hierarchical)

Cheaper for enterprise

text

### 2. Data Factory (Managed ETL)

What: ETL orchestration (like AWS Glue or Airflow)
Why: Visual pipelines, connectors to 100+ sources, schedule/trigger
Cons: Less code control, more UI clicking

Architecture:
Pipeline
├─ Activities (Copy data, Run notebooks, SQL scripts)
├─ Triggers (Schedule, Event-based, Manual)
└─ Linked Services (Connections to Blob, SQL, etc)

Example: Copy from SQL Server to Data Lake
Pipeline: "DailyETL"
├─ Activity 1: Copy from SQL Server → Blob Storage
│ ├─ Source: SQL Server (on-prem)
│ ├─ Sink: Blob Storage (ADLS)
│ └─ Schedule: Daily 1am
├─ Activity 2: Run Spark notebook (transform)
│ └─ Path: /notebooks/transform.ipynb
└─ Activity 3: Run SQL stored procedure (load to Synapse)
└─ SQL: EXEC LoadToSynapse

Pricing: $1 per pipeline run (for first 50k runs free)

text

### 3. Synapse Analytics

What: Unified analytics platform (warehouse + Spark + SQL)
Why: One system for batch + streaming + data science
Cons: Complex, many moving parts

Components:
a) SQL Pools (MPP warehouse, like Redshift)

Provisioned: You choose DWU (Data Warehouse Units)

Serverless: Pay-as-you-query (cheaper for ad-hoc)

b) Spark Pools (Managed Spark)

Run Spark jobs without EMR setup

Language: PySpark, Scala, SQL

c) Pipelines (Orchestration, like Data Factory)

Integrated in Synapse Studio

d) Data Wrangler (Visual data prep)

Clean/transform data without code

Example: Query Synapse SQL Pool
SELECT
customer_id,
SUM(amount) as total,
COUNT(*) as num_orders
FROM fact_sales
WHERE date >= '2024-01-01'
GROUP BY customer_id
ORDER BY total DESC;

Example: Run Spark in Synapse
df = spark.read.parquet("abfss://data@lake.dfs.core.windows.net/sales/")
transformed = df.filter(df.amount > 0).groupBy("customer_id").agg(F.sum("amount"))
transformed.write.parquet("abfss://data@lake.dfs.core.windows.net/processed/")

Pricing:

SQL Pool: $1.71/DWU-hour (100 DWU = $171/hour)

Spark Pool: $0.49/vCore-hour (similar to GCP Dataflow)

text

### 4. Event Hubs (Streaming)

What: Kafka alternative, fully managed
Why: Integrates with Synapse, auto-scales
Cons: Kafka ecosystem smaller

Use case: Real-time data ingestion
Event Source → Event Hubs → Consumer (Spark/Function) → Synapse

Pricing: $0.015 per million events

text

---

## Why Azure for Enterprise

### Active Directory Integration
Users in Azure AD → Auto-authorized in Data Lake/Synapse
No need separate identity management
GDPR/Compliance: Audit trails built-in
text

### Microsoft Stack Integration
Power BI → Synapse (same admin console)
Teams → Data Factory alerts (notifications in Teams)
Dynamics 365 → Real-time sync to Synapse
Office 365 → Data governance (sensitivity labels)

text

### Governance & Compliance
Purview: Data lineage, cataloging (what changed, who accessed)

Security: VNet integration, private endpoints

Compliance: Built-in HIPAA, PCI, GDPR templates

text

---

## Real-World: Financial Company

Scenario: JPMorgan wants to replicate trades to analytics
Architecture:
1. SQL Server (on-prem) → Data Factory
2. Incremental load to Blob Storage (ADLS)
3. Synapse Spark (transform)
4. Synapse SQL Pool (warehouse)
5. Power BI (trading dashboards)
Step 1: Data Factory Pipeline
Pipeline: "DailyTradeETL"
├─ Linked Service: SQL Server (on-prem, connection string)
├─ Activity: Copy from SQL Server
│ └─ Query: SELECT * FROM trades WHERE date = '2024-01-15'
│ └─ Sink: Blob/ADLS
├─ Activity: Run Spark notebook
│ └─ Transform: Validate, enrich, aggregate
└─ Activity: Execute SQL script (Synapse)
└─ COPY INTO fact_trades FROM ADLS

Step 2: Synapse Spark notebook (transform)
df = spark.read.parquet("abfss://data@lake.dfs.core.windows.net/raw/trades/2024-01-15/")
df_clean = df.filter(df.amount > 0).dropna()
df_enriched = df_clean.withColumn("trade_value", col("quantity") * col("price"))
df_enriched.write.parquet("abfss://data@lake.dfs.core.windows.net/processed/trades/2024-01-15/")

Step 3: Synapse SQL Pool (reporting)
SELECT
trader_id,
SUM(trade_value) as daily_volume,
COUNT(*) as num_trades,
AVG(trade_value) as avg_trade
FROM fact_trades
WHERE date = '2024-01-15'
GROUP BY trader_id;

Step 4: Power BI (connected to Synapse)
Dashboards auto-refresh, compliance audit trails
Cost:
Data Factory: $500/month
Synapse SQL: $1500/month (100 DWU)
Synapse Spark: $200/month
Purview: $200/month
Total: ~$2400/month
text

---

## Azure vs AWS vs GCP

| Feature | Azure | AWS | GCP |
|---------|-------|-----|-----|
| **Warehouse** | Synapse SQL | Redshift | BigQuery |
| **Managed ETL** | Data Factory | Glue | Dataflow |
| **Enterprise** | Best | Good | Developing |
| **Microsoft Integration** | Native | Limited | Limited |
| **Ease** | Mid | Low-Mid | High |
| **Cost** | Mid | Mid | Low |
| **Compliance** | Strong | Strong | Developing |

---

## Errores Comunes en Entrevista

- **Error**: "Synapse es más expensive" → **Solución**: Comparable, but better value for enterprise

- **Error**: Usando Synapse Serverless para everything → **Solución**: Serverless más caro si queries big, use provisioned para frequent

- **Error**: No integrating with Power BI → **Solución**: Power BI + Synapse = seamless (main Azure advantage)

- **Error**: Forgetting Purview for compliance** → **Solución**: Enterprise needs data lineage/governance

---

## Preguntas de Seguimiento

1. **"¿Cuándo Synapse SQL vs Serverless?"**
   - SQL Pool: Frequent queries, predictable workload
   - Serverless: Ad-hoc, variable queries

2. **"¿Data Factory vs Synapse Pipelines?"**
   - Data Factory: More matured, more connectors
   - Synapse Pipelines: Newer, integrated ecosystem

3. **"¿Purview vs Collibra?"**
   - Purview: Native Azure, governance
   - Collibra: Multi-cloud, mature

4. **"¿Migrando from on-prem SQL Server?"**
   - Data Factory + Synapse = smooth migration path
   - Azure Database Migration Service (DMS) helps

---

## References

- [Azure Synapse - Official Docs](https://docs.microsoft.com/en-us/azure/synapse-analytics/)
- [Data Factory - ETL Service](https://docs.microsoft.com/en-us/azure/data-factory/)
- [Azure Data Lake - Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
- [Purview - Data Governance](https://docs.microsoft.com/en-us/azure/purview/)

