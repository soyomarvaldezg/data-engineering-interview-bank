# AWS Data Stack para Data Engineering

**Tags**: #cloud #aws #s3 #redshift #emr #lambda #real-interview

---

## TL;DR

AWS data stack: **S3 (storage)** → **EC2/EMR (compute)** → **Redshift (warehouse)** → **Lambda (functions)**  
Servicios clave: S3 (barato, duradero), EMR (Spark managed), Redshift (OLAP warehouse), Lambda (serverless), Glue (ETL), Kinesis (streaming)  
Trade-off: Costo vs Conveniencia vs Control.  
Mejor: S3 para storage, EMR para batch, Redshift para analytics.

---

## Concepto Core

- Qué es: AWS = infraestructura como servicio. Puedes armar cualquier pipeline.
- Por qué importa: AWS es industry standard. 40% de data engineers usan AWS.
- Principio clave: Storage (S3) ≠ Compute (EMR). Separados → flexible.

---

## Cómo explicarlo en entrevista

1. "AWS data stack = S3 (storage) + EMR (compute) + Redshift (warehouse)"
2. "S3 es data lake. EMR corre Spark. Redshift es warehouse. Separación = flexible"
3. "Alternativas: Glue (managed ETL), Lambda (serverless), Kinesis (streaming)"
4. "Trade-off: Control/Flexibilidad vs Conveniencia vs Costo"

---

## Arquitectura

```
┌──────────────────────────────────────┐
│ DATA SOURCES                        │
│ ├─ Databases (RDS)                  │
│ ├─ APIs                             │
│ ├─ On-prem systems                  │
│ └─ 3rd party SaaS                   │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ AWS INGESTION                       │
│ ├─ Direct Connect (fast)             │
│ ├─ S3 Transfer Acceleration          │
│ ├─ Kinesis (real-time)               │
│ └─ Database Migration Service (DMS)  │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ S3 DATA LAKE (Raw)                   │
│ ├─ s3://data-lake/raw/               │
│ ├─ Partitioned by source/date        │
│ └─ Lifecycle policies (archive old)  │
└───────────────┬──────────────────────┘
                │
       ┌────────┴───────────┐
       │                    │
       ▼                    ▼
┌─────────────┐     ┌──────────────┐
│ EMR Cluster │     │ AWS Glue     │
│ (Spark)     │     │ (Managed ETL)│
└──────┬──────┘     └──────┬───────┘
       │                    │
       └────────┬───────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ S3 DATA LAKE (Processed)             │
│ ├─ Parquet format                    │
│ ├─ Partitioned smart                 │
│ └─ Ready for analytics               │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ REDSHIFT WAREHOUSE                   │
│ ├─ Loaded via COPY command            │
│ ├─ Optimized for OLAP queries         │
│ └─ Ready for BI tools                 │
└───────────────┬──────────────────────┘
                │
       ┌────────┴───────────┐
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│ Dashboards   │     │ Analysts     │
│ (Tableau)    │     │ SQL queries  │
└──────────────┘     └──────────────┘
```

---

## Servicios Clave

### 1. S3 (Simple Storage Service)

**Qué:** Object storage (blobs, no files)  
**Por qué:** Barato ($0.023/GB/month), duradero (99.999999999%), escalable  
**Contras:** Eventually consistent, latencia ≈100ms

**Pricing:**

- Storage: $0.023/GB/month
- Transfer out: $0.09/GB
- API calls: $0.0004 per 1000 requests

**Architecture:**

```
s3://bucket/path/object
├─ Bucket: Namespace (único globalmente)
├─ Path: Estructura tipo folder (no folders reales)
└─ Object: Archivo real
```

**Optimización:**

- Partitioning: `s3://bucket/year=2024/month=01/day=15/`  
  → Permite el partition pruning.
- Formato: Parquet > CSV/JSON  
  → Parquet: columnar, comprimido, 10x más pequeño.
- Lifecycle: Mover datos antiguos a Glacier (archive, 90% más barato).

---

### 2. EMR (Elastic MapReduce)

**Qué:** Managed Hadoop/Spark cluster  
**Por qué:** No hay necesidad de gestionar la infraestructura, solo ejecutar Spark  
**Contras:** Más caro que raw EC2, overkill para jobs pequeños

**Example:**

```
# Launch 10-node Spark cluster
aws emr create-cluster \
--name "daily-etl" \
--release-label emr-6.9.0 \
--applications Name=Spark \
--instance-count 10 \
--instance-type m5.xlarge

# Submit Spark job
spark-submit s3://code/etl.py
```

El cluster auto-escalado y se apaga cuando termina.

---

### 3. Redshift (Data Warehouse)

**Qué:** Base de datos MPP (Massively Parallel Processing), OLAP optimized  
**Por qué:** Consulta 100GB en segundos, barato para analytics  
**Contras:** Necesita schema design, no real-time

**Ejemplo de arquitectura:**

- 4 nodes × 2TB cada uno = 8TB de capacidad
- Columnar storage → consultas rápidas
- Ejecución distribuida

**Pricing:**

- `dc2.large`: $0.43/hour per node
- 4 nodes × $0.43 = $1.72/hour
- ≈ $1200/month (24/7)

**Optimizaciones:**

- Compression: zstd (90% más pequeño)
- Sort keys: ordenar por columnas de filtro comunes
- Dist keys: distribuir por join key
- Vacuum: recuperar espacio de las eliminaciones

---

### 4. Lambda (Serverless Functions)

**Qué:** Ejecutar código sin gestionar servers  
**Por qué:** Auto-scales, paga por ejecución  
**Contras:** Cold start (~1s), timeout máximo 15 min

**Casos de uso:**

- Trigger ETL en subida a S3
- Procesar streaming data
- API backends

**Ejemplo:** Trigger Lambda en subida a S3

```
@lambda_handler
def handler(event, context):
    bucket = event['Records']['s3']['bucket']['name']
    key = event['Records']['s3']['object']['key']

    df = pd.read_csv(f"s3://{bucket}/{key}")
    result = process(df)

    result.to_parquet(f"s3://output/{key}.parquet")
    return {'statusCode': 200}
```

**Pricing:** $0.20 per 1M invocations + compute time.

---

### 5. Glue (Managed ETL)

**Qué:** ETL gestionado por AWS para serverless pipelines  
**Por qué:** Auto-scales, se integra con Redshift/Athena  
**Contras:** Control limitado vs pure Spark

**Características:**

- **Data Catalog:** registro de metadata
- **Crawlers:** auto-detectar schema desde S3
- **Jobs:** ejecutar PySpark/Scala
- **Triggers:** basados en schedule o eventos

**Example:**

```
import awsglue
from awsglue.context import GlueContext
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)

# Read from S3
dyf = glueContext.create_dynamic_frame.from_options(
    "s3",
    {"path": "s3://data-lake/raw/customers/"}
)

# Transform
transformed = dyf.map(lambda x: clean_data(x))

# Write to Redshift
glueContext.write_dynamic_frame.from_jdbc_conf(
    frame=transformed,
    catalog_connection="redshift-connection",
    connection_options={"dbtable": "public.customers"},
    transformation_ctx="to_redshift"
)
```

---

### 6. Kinesis (Streaming)

**Qué:** Alternativa managed a Kafka  
**Por qué:** Escala automáticamente, AWS-native  
**Contras:** Mayor costo, menos control

**Casos de uso:**

- Real-time ingestion
- Event streaming
- On-the-fly analytics

**Arquitectura:**

```
Data Source → Kinesis Streams → Lambda/Spark → S3/Redshift
```

**Especificaciones:**

- 1 shard = 1 MB/sec ingestion
- 10 shards = 10 MB/sec
- Auto-scaling disponible
- Pricing: $0.035/shard-hour

---

## Mundo Real: End-to-End Pipeline

**Escenario:** ETL diario de e-commerce

**Pasos:**

1. Lambda triggered por subida a S3
2. EMR Spark cluster se inicia
3. Leer raw S3 → Transformar → Escribir processed S3
4. Redshift COPY desde S3
5. Analytics en Redshift

**Architecture:**

```
AWS_ARCHITECTURE = {
    "ingestion": "S3 (raw data subida diariamente)",
    "processing": "EMR (Spark cluster, 5 nodes)",
    "storage": "S3 (processed Parquet)",
    "warehouse": "Redshift (4 nodes, 8TB)",
    "orchestration": "Lambda + EventBridge (cron triggers)",
    "monitoring": "CloudWatch (logs, metrics, alarms)",
    "cost": "$2000/month (S3 $100, EMR $500, Redshift $1200, Lambda $50, misc $150)"
}
```

**Alternativas:**

- AWS Glue only: más fácil pero menos control
- Databricks on EC2: mayor control, mayor complejidad

---

## Trade-offs: AWS vs Alternativas

| Criterios            | AWS      | GCP             | Azure        |
| -------------------- | -------- | --------------- | ------------ |
| Equivalente a S3     | S3       | GCS             | Blob Storage |
| Spark managed        | EMR      | Dataproc        | HDInsight    |
| Warehouse            | Redshift | BigQuery        | Synapse      |
| Serverless functions | Lambda   | Cloud Functions | Functions    |
| Streaming            | Kinesis  | Pub/Sub         | Event Hubs   |
| Costo                | Medio    | Bajo (BigQuery) | Medio        |
| Facilidad            | Medio    | Alto (BigQuery) | Medio        |
| Control              | Alto     | Medio           | Medio        |

---

## Errores Comunes en Entrevista

- **"EMR es siempre más barato que Glue"** → Depende: para pequeños jobs, Glue es más barato.
- **"S3 es ilimitado"** → Sí, pero el costo/performance importan. Haz partition smart.
- **Ignorar Data Transfer Out Cost** → Transferir fuera de AWS cuesta $0.09/GB.
- **Usar Redshift para datos real-time** → Redshift es batch-oriented. Usa Kinesis o Lambda.

---

## Preguntas de Seguimiento

1.  **¿Cuándo usas EMR vs Glue?**
    - EMR: Control, custom Spark, big clusters.
    - Glue: Managed, serverless, smaller jobs.

2.  **¿Redshift vs Athena?**
    - Redshift: Persistent warehouse, datos preloaded, complex queries.
    - Athena: Query S3 directamente, barato y ad-hoc.

3.  **¿Cómo optimizas S3 para costo?**
    - Partitioning (partition pruning).
    - Compression (Parquet).
    - Lifecycle (archivar data vieja).
    - Right format (Parquet > CSV).

4.  **¿Cold start de EMR clusters?**
    - 5–10 minutos para launch.
    - Solución: mantener warm clusters (más caro, más rápido).

---

## Referencias

- [AWS S3 - Official Docs](https://docs.aws.amazon.com/s3/)
- [EMR - Spark on AWS](https://docs.aws.amazon.com/emr/)
- [Redshift - Data Warehouse](https://docs.aws.amazon.com/redshift/)
- [AWS Glue - ETL Service](https://docs.aws.amazon.com/glue/)
- [Kinesis - Streaming](https://docs.aws.amazon.com/kinesis/)
