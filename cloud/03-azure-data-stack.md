# Azure Data Stack para Data Engineering

**Tags**: #cloud #azure #synapse #data-factory #blob-storage #event-hubs #purview #real-interview

---

## TL;DR

Stack de datos de Azure: **Blob Storage (ADLS Gen2, equivalente a S3)** → **Data Factory (ETL/orchestration gestionado)** → **Synapse Analytics (warehouse + Spark + SQL)**.  
Ventaja clave: Integración enterprise con el ecosistema Microsoft (Azure AD, Power BI, Office, Teams, Purview) y una sólida postura de governance/compliance.  
Trade-off: Muy potente pero más complejo; destaca en organizaciones grandes con requisitos estrictos de seguridad y gobierno.  
Ideal para: Corporaciones, “Microsoft shops”, industrias reguladas con data governance como prioridad.

---

## Concepto Core

- Qué es: Azure es la nube de Microsoft con un enfoque enterprise, security/compliance y governance integrados desde el inicio.
- Por qué importa: Amplia adopción en empresas grandes, integración nativa con Microsoft 365 y Azure AD.
- Principio clave: “Enterprise data platform” con data governance, RBAC, private endpoints y lineage como pilares.

---

## Arquitectura

```
┌──────────────────────────────────────┐
│ FUENTES DE DATOS                    │
│ ├─ SQL Server (local)               │
│ ├─ Dynamics 365                     │
│ ├─ SharePoint                       │
│ └─ External APIs                    │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ INGESTA EN AZURE                    │
│ ├─ Data Factory (pipelines)         │
│ ├─ Event Hubs (streaming)           │
│ ├─ Data Share (datos de socios)     │
│ └─ Synapse Pipelines                │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ AZURE DATA LAKE (ADLS Gen2)         │
│ ├─ Blob Storage (namespace jerárquico)│
│ ├─ ACLs de grano fino               │
│ └─ Políticas de ciclo de vida       │
└───────────────┬──────────────────────┘
                │
       ┌────────┴───────────┐
       │                    │
       ▼                    ▼
┌─────────────┐     ┌──────────────┐
│ Data Factory│     │ Synapse      │
│ (ETL gestionado)  │ Spark pools  │
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
│ Power BI     │     │ Analistas    │
│ (nativo de MS)│     │ SQL queries  │
└──────────────┘     └──────────────┘
```

---

## Key Services

### 1) Azure Blob Storage (ADLS Gen2)

- Qué: Object storage con hierarchical namespace (directorios “reales”) y ACLs granulares para data lake.
- Por qué: Integración estrecha con Data Factory, Synapse y Azure AD; seguridad y control a nivel enterprise.
- Contras: El diseño de permisos puede ser más detallado/estricto; el governance requiere buenas prácticas.

**URI (DFS/ABFSS):**

```
abfss://<container>@<storageaccount>.dfs.core.windows.net/path/file
```

**Features:**

- Hierarchical namespace (carpetas y operaciones a nivel de directorio).
- ACLs finos por carpeta/archivo; integración con Azure AD (RBAC).
- Lifecycle policies para archivar a Cool/Archive y optimizar el costo.

---

### 2) Data Factory (Managed ETL / Orchestration)

- Qué: Orquestación visual de ETL/ELT con conectores a +100 fuentes; scheduling y triggers.
- Por qué: Ideal para integraciones enterprise (SQL Server, SAP, Dynamics, etc.), para mover/transformar datos sin gestionar infra.
- Contras: Menos “code-first”; más “UI-first”. Para lógica compleja, conviene combinar con notebooks/Spark.

**Estructura:**

```
Pipeline
├─ Activities (Copy, Notebooks, SQL scripts)
├─ Triggers (Schedule, Event, Manual)
└─ Linked Services (conexiones a Blob, SQL, etc.)
```

**Ejemplo (copy + transform + load):**

- Activity 1: Copy SQL Server (local) → ADLS
- Activity 2: Ejecutar Synapse Spark notebook (transform)
- Activity 3: Ejecutar SQL en Synapse (load final)

---

### 3) Synapse Analytics

- Qué: Plataforma unificada de analytics: Dedicated SQL pools (MPP), Serverless SQL, Spark pools, pipelines, notebooks.
- Por qué: Un solo entorno para batch/stream, SQL analytics y notebooks; integración nativa con Power BI y ADLS.
- Contras: Complejidad; muchos componentes y decisiones de arquitectura.

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

- Qué: Servicio de ingesta de eventos de alta escala (alternativa a Kafka fully managed).
- Por qué: Integración con Spark, Functions y Synapse; útil para telemetría, IoT y eventos financieros.
- Contras: Diferencias de API y operación frente a Kafka tradicional; patrones de consumo propios.

**Patrón:**

```
Producers → Event Hubs → Consumers (Spark/Functions) → ADLS/Synapse
```

---

### 5) Microsoft Purview (Governance)

- Qué: Data governance nativo (catalog, lineage, clasificación, data policy).
- Por qué: Lineage de extremo a extremo, data catalog central, etiquetas de sensibilidad, cumplimiento regulatorio.
- Contras: Requiere configuración y gobierno operativo; conviene definir ownership/roles desde el inicio.

**Capacidades:**

- Data Map, Data Catalog, Data Estate Insights, Data Policy, Data Sharing.
- Integración con Power BI, SQL Server, ADLS, Synapse y fuentes multi-cloud.

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
- VNet integration, Private Endpoints y cifrado estándar para reducir la superficie de ataque.

---

## Real-World: Financial Company

**Escenario:** Réplica de operaciones (trades) a plataforma analítica con controles de compliance.

**Arquitectura (resumen):**

1.  SQL Server (local) → Data Factory (incremental).
2.  ADLS (zona raw/bronze) con ACLs y carpetas jerárquicas.
3.  Synapse Spark: validación, enrichment y agregaciones (silver/gold).
4.  Synapse Dedicated SQL Pool: modelado relacional/estrella para BI.
5.  Power BI: dashboards auditables con etiquetas de sensibilidad.

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

| Feature                      | Azure             | AWS        | GCP                |
| :--------------------------- | :---------------- | :--------- | :----------------- |
| Warehouse                    | Synapse SQL       | Redshift   | BigQuery           |
| ETL/Orchestration Gestionado | Data Factory      | Glue       | Dataflow           |
| Integración Enterprise       | Mejor (Microsoft) | Bueno      | En desarrollo      |
| Governance/Compliance        | Fuerte (Purview)  | Fuerte     | Mejorando          |
| Facilidad (operar)           | Medio             | Medio-Bajo | Alto               |
| Costo (típico)               | Medio             | Medio      | Bajo (pago por TB) |

---

## Errores Comunes en Entrevista

- “Synapse es siempre más caro”: Depende del patrón; workloads frecuentes y estables aprovechan Dedicated SQL, ad-hoc funciona mejor en Serverless.
- “Serverless para todo”: En cargas pesadas y recurrentes puede resultar más caro; evalúa Dedicated SQL con DWUs adecuados.
- Ignorar Power BI/Purview: Perderás ventajas clave de integración y governance; son diferenciales en Azure.
- Permisos al azar en ADLS: Define RBAC/ACLs consistentes, naming conventions y zones (raw/silver/gold) desde el diseño.

---

## Preguntas de Seguimiento

1.  ¿Cuándo usar Synapse Dedicated SQL vs Serverless?
    - Dedicated: cargas frecuentes y predecibles, SLAs de performance, modelos estrella.
    - Serverless: exploratorio, ad-hoc, picos esporádicos, data lake-first.

2.  ¿Data Factory vs Synapse Pipelines?
    - Data Factory: producto maduro, amplio set de conectores; ideal si no usas Synapse Studio.
    - Synapse Pipelines: dentro de Synapse Studio; buena experiencia si ya concentras tu trabajo en Synapse.

3.  ¿Purview vs Collibra?
    - Purview: nativo de Azure, excelente integración con el stack MS y costo/operación menores en un entorno Azure-first.
    - Collibra: multi-cloud/agnóstico, robusto para organizaciones heterogéneas.

4.  ¿Migración desde SQL Server local?
    - Data Factory (self-hosted IR) + Synapse; apoyarse en Azure Database Migration Service para lift-and-shift o modernización.

---

## Referencias

- Azure Data Lake Storage Gen2 (introducción) - https://www.youtube.com/watch?v=jH37kLkWCRk
- Azure Governance (Azure Policy, Blueprints, RBAC) - https://atlan.com/azure-data-governance-benefits/
- https://learn.microsoft.com/en-us/azure/dms/
- https://azure.microsoft.com/en-us/products/data-factory/
