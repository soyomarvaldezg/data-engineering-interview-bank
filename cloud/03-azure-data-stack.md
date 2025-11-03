# Azure Data Stack para Data Engineering

**Tags**: #cloud #azure #synapse #data-factory #blob-storage #event-hubs #purview #real-interview  
**Empresas**: Microsoft, JPMorgan, Goldman Sachs, Enterprise (Fortune 500)  
**Dificultad**: Mid  
**Tiempo estimado**: 20 min

---

## TL;DR

Azure data stack: **Blob Storage (ADLS Gen2, S3 equivalent)** → **Data Factory (managed ETL/orchestration)** → **Synapse Analytics (warehouse + Spark + SQL)**.  
Ventaja clave: Integración enterprise con el ecosistema Microsoft (Azure AD, Power BI, Office, Teams, Purview) y fuerte postura de governance/compliance.  
Trade-off: Muy potente pero más complejo; destaca en organizaciones grandes con requisitos de seguridad y gobierno estrictos.  
Ideal para: Corporaciones, “Microsoft shops”, industrias reguladas con data governance first.

---

## Concepto Core

- Qué es: Azure es la nube de Microsoft con enfoque enterprise, security/compliance y governance integrados desde el inicio.
- Por qué importa: Amplia adopción en empresas grandes, integración nativa con Microsoft 365 y Azure AD.
- Principio clave: “Enterprise data platform” con data governance, RBAC, private endpoints, y lineage como pilares.

---

## Arquitectura

```
┌──────────────────────────────────────┐
│ DATA SOURCES                        │
│ ├─ SQL Server (on-prem)             │
│ ├─ Dynamics 365                     │
│ ├─ SharePoint                       │
│ └─ External APIs                    │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ AZURE INGESTION                     │
│ ├─ Data Factory (pipelines)         │
│ ├─ Event Hubs (streaming)           │
│ ├─ Data Share (partner data)        │
│ └─ Synapse Pipelines                │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ AZURE DATA LAKE (ADLS Gen2)         │
│ ├─ Blob Storage (hierarchical ns)   │
│ ├─ Fine-grained ACLs                │
│ └─ Lifecycle policies               │
└───────────────┬──────────────────────┘
                │
       ┌────────┴───────────┐
       │                    │
       ▼                    ▼
┌─────────────┐     ┌──────────────┐
│ Data Factory│     │ Synapse      │
│ (Managed ETL)     │ Spark pools  │
└──────┬──────┘     └──────┬───────┘
       │                    │
       └────────┬───────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ SYNAPSE ANALYTICS (Warehouse)        │
│ ├─ Dedicated SQL pools (MPP)         │
│ ├─ Serverless SQL pools              │
│ └─ Notebooks/Data Science            │
└───────────────┬──────────────────────┘
                │
       ┌────────┴───────────┐
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│ Power BI     │     │ Analysts     │
│ (MS native)  │     │ SQL queries  │
└──────────────┘     └──────────────┘
```

---

## Key Services

### 1) Azure Blob Storage (ADLS Gen2)

- What: Object storage con hierarchical namespace (directorios “reales”) y ACLs granulares para data lake.
- Why: Integración estrecha con Data Factory, Synapse y Azure AD; seguridad y control a nivel enterprise.
- Cons: Diseño de permisos puede ser más detallado/estricto; gobernanza requiere buenas prácticas.

**URI (DFS/ABFSS):**

```
abfss://<container>@<storageaccount>.dfs.core.windows.net/path/file
```

**Features:**

- Hierarchical namespace (carpetas y operaciones a nivel de directorio).
- ACLs finos por carpeta/archivo; integración con Azure AD (RBAC).
- Lifecycle policies para archivar a Cool/Archive y optimizar costo.

---

### 2) Data Factory (Managed ETL / Orchestration)

- What: Orquestación visual de ETL/ELT con conectores a +100 fuentes; scheduling y triggers.
- Why: Ideal para integraciones enterprise (SQL Server, SAP, Dynamics, etc.), mover/transformar datos sin gestionar infra.
- Cons: Menos “code-first”; más “UI-first”. Para lógica compleja, conviene combinar con notebooks/Spark.

**Estructura:**

```
Pipeline
├─ Activities (Copy, Notebooks, SQL scripts)
├─ Triggers (Schedule, Event, Manual)
└─ Linked Services (conexiones a Blob, SQL, etc.)
```

**Ejemplo (copy + transform + load):**

- Activity 1: Copy SQL Server (on-prem) → ADLS
- Activity 2: Run Synapse Spark notebook (transform)
- Activity 3: Execute SQL en Synapse (load final)

---

### 3) Synapse Analytics

- What: Plataforma unificada de analytics: Dedicated SQL pools (MPP), Serverless SQL, Spark pools, pipelines, notebooks.
- Why: Un solo entorno para batch/stream, SQL analytics, y notebooks; integración nativa con Power BI y ADLS.
- Cons: Complejidad; muchos componentes y decisiones de arquitectura.

**Componentes:**

- Dedicated SQL pools (MPP): warehouse provisionado con DWUs.
- Serverless SQL: pay-as-you-query sobre data lake.
- Spark pools: PySpark/Scala para transformaciones.
- Pipelines: orquestación integrada (similares a Data Factory).
- Data Wrangler/Studio: herramientas visuales para preparación.

**Ejemplo: Query en Dedicated SQL Pool**

```
SELECT
  customer_id,
  SUM(amount) AS total,
  COUNT(*)    AS num_orders
FROM fact_sales
WHERE [date] >= '2024-01-01'
GROUP BY customer_id
ORDER BY total DESC;
```

**Ejemplo: Spark en Synapse**

```
from pyspark.sql import functions as F

df = spark.read.parquet("abfss://data@lake.dfs.core.windows.net/sales/")
transformed = (
    df.filter(F.col("amount") > 0)
      .groupBy("customer_id")
      .agg(F.sum("amount").alias("total_amount"))
)
transformed.write.mode("overwrite").parquet(
    "abfss://data@lake.dfs.core.windows.net/processed/sales_daily/"
)
```

---

### 4) Event Hubs (Streaming)

- What: Servicio de ingesta de eventos de alta escala (alternativa a Kafka fully managed).
- Why: Integración con Spark, Functions y Synapse; útil para telemetría, IoT y eventos financieros.
- Cons: Diferencias de API y operación frente a Kafka tradicional; patrones de consumo propios.

**Patrón:**

```
Producers → Event Hubs → Consumers (Spark/Functions) → ADLS/Synapse
```

---

### 5) Microsoft Purview (Governance)

- What: Data governance nativo (catalog, lineage, clasificación, data policy).
- Why: Lineage de extremo a extremo, data catalog central, etiquetas de sensibilidad, cumplimiento regulatorio.
- Cons: Requiere configuración y gobierno operativo; conviene definir ownership/roles desde el inicio.

**Capacidades:**

- Data Map, Data Catalog, Data Estate Insights, Data Policy, Data Sharing.
- Integración con Power BI, SQL Server, ADLS, Synapse, y fuentes multi-cloud.

---

## Why Azure for Enterprise

### Azure AD Integration

- Identidad centralizada y RBAC; usuarios/grupos de Azure AD se reflejan en permisos sobre ADLS/Synapse.
- Menos fricción en provisión de accesos, auditorías y revocación.

### Microsoft Stack Integration

- Power BI con Synapse/ADLS de forma nativa; Single Sign-On y governance alineado.
- Teams para alertas/notificaciones de pipelines; Dynamics 365 y M365 como fuentes frecuentes.

### Governance & Compliance

- Azure Policy, Blueprints y Purview para enforce/audit de políticas y cumplimiento (ISO, GDPR, HIPAA, PCI).
- VNet integration, Private Endpoints y cifrado estándar para reducir superficie de ataque.

---

## Real-World: Financial Company

**Scenario:** Réplica de operaciones (trades) a plataforma analítica con controles de compliance.

**Arquitectura (resumen):**

1. SQL Server (on-prem) → Data Factory (incremental).
2. ADLS (zona raw/bronze) con ACLs y carpetas jerárquicas.
3. Synapse Spark: validación, enrichment y agregaciones (silver/gold).
4. Synapse Dedicated SQL Pool: modelado relacional/estrella para BI.
5. Power BI: dashboards auditables con etiquetas de sensibilidad.

**Synapse Spark (transform ejemplo)**

```
from pyspark.sql.functions import col

df = spark.read.parquet(
    "abfss://data@lake.dfs.core.windows.net/raw/trades/2024-01-15/"
)
df_clean = df.filter(col("amount") > 0).dropna()
df_enriched = df_clean.withColumn("trade_value", col("quantity") * col("price"))
df_enriched.write.mode("overwrite").parquet(
    "abfss://data@lake.dfs.core.windows.net/processed/trades/2024-01-15/"
)
```

**Synapse SQL (reporting ejemplo)**

```
SELECT
  trader_id,
  SUM(trade_value) AS daily_volume,
  COUNT(*)         AS num_trades,
  AVG(trade_value) AS avg_trade
FROM fact_trades
WHERE [date] = '2024-01-15'
GROUP BY trader_id;
```

---

## Azure vs AWS vs GCP

| Feature                   | Azure            | AWS      | GCP          |
| ------------------------- | ---------------- | -------- | ------------ |
| Warehouse                 | Synapse SQL      | Redshift | BigQuery     |
| Managed ETL/Orchestration | Data Factory     | Glue     | Dataflow     |
| Enterprise Integration    | Best (Microsoft) | Good     | Developing   |
| Governance/Compliance     | Strong (Purview) | Strong   | Improving    |
| Ease (operar)             | Mid              | Mid-Low  | High         |
| Cost (típico)             | Mid              | Mid      | Low (pay/TB) |

---

## Errores Comunes en Entrevista

- “Synapse es siempre más caro”: Depende del patrón; workloads frecuentes y estables aprovechan Dedicated SQL, ad-hoc va mejor en Serverless.
- “Serverless para todo”: En cargas pesadas y recurrentes puede salir más caro; evalúa Dedicated SQL con DWUs adecuados.
- Ignorar Power BI/Purview: Perderás ventajas clave de integración y governance; son diferenciales en Azure.
- Permisos al azar en ADLS: Define RBAC/ACLs consistentes, naming conventions y zones (raw/silver/gold) desde el diseño.

---

## Preguntas de Seguimiento

1. ¿Cuándo Synapse Dedicated SQL vs Serverless?

- Dedicated: cargas frecuentes y predecibles, SLAs de performance, modelos estrella.
- Serverless: exploratorio, ad-hoc, picos esporádicos, data lake-first.

2. ¿Data Factory vs Synapse Pipelines?

- Data Factory: producto maduro, amplio set de conectores; ideal si no usas Synapse Studio.
- Synapse Pipelines: dentro de Synapse Studio; buena experiencia si ya concentras tu trabajo en Synapse.

3. ¿Purview vs Collibra?

- Purview: nativo Azure, excelente integración con el stack MS y costo/operación menores en Azure-first.
- Collibra: multi-cloud/agnóstico, robusto para organizaciones heterogéneas.

4. ¿Migración desde SQL Server on-prem?

- Data Factory (self-hosted IR) + Synapse; apoyarse en Azure Database Migration Service para lift-and-shift o modernización.

---

## Referencias

1.  Azure Synapse Analytics (documentación oficial) - 1 https://azure.microsoft.
    com/en-us/resources/cloud-computing-dictionary/what-is-a-data-governance
2.  Azure Data Factory (documentación oficial) - 2 https://azure.microsoft.com/en-us/solutions/governance
3.  Azure Data Lake Storage Gen2 (introducción) - 3 https://www.youtube.com/watch?v=jH37kLkWCRk
4.  Microsoft Purview (documentación oficial) - 4 https://learn.microsoft.
    com/en-us/azure/architecture/guide/management-governance/management-governance-start-here
5.  Azure Governance (Azure Policy, Blueprints, RBAC) - 5 https://atlan.com/azure-data-governance-benefits/
6.  6 https://dynatechconsultancy.com/blog/microsoft-azure-is-changing-enterprise-data-governance
7.  7 https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/govern
8.  8 https://www.informatica.com/lp/best-practices-azure-data-governance_4237.html
9.  9 https://www.atmosera.com/blog/data-governance-basics/
