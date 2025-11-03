# AWS Data Stack para Data Engineering

**Tags**: #cloud #aws #s3 #redshift #emr #lambda #real-interview  
**Empresas**: Amazon, Google, Meta, Netflix, Stripe  
**Dificultad**: Mid  
**Tiempo estimado**: 22 min

---

## TL;DR

AWS data stack: **S3 (storage)** → **EC2/EMR (compute)** → **Redshift (warehouse)** → **Lambda (functions)**  
Key services: S3 (cheap, durable), EMR (Spark managed), Redshift (OLAP warehouse), Lambda (serverless), Glue (ETL), Kinesis (streaming)  
Trade-off: Cost vs Convenience vs Control.  
Best: S3 for storage, EMR for batch, Redshift for analytics.

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
4. "Trade-off: Control/Flexibility vs Convenience vs Cost"

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

## Key Services

### 1. S3 (Simple Storage Service)

**What:** Object storage (blobs, not files)  
**Why:** Cheap ($0.023/GB/month), durable (99.999999999%), scalable  
**Cons:** Eventually consistent, latency ≈100ms

**Pricing:**

- Storage: $0.023/GB/month
- Transfer out: $0.09/GB
- API calls: $0.0004 per 1000 requests

**Architecture:**

```
s3://bucket/path/object
├─ Bucket: Namespace (unique globally)
├─ Path: Folder-like structure (not real folders)
└─ Object: Actual file
```

**Optimization:**

- Partitioning: `s3://bucket/year=2024/month=01/day=15/`  
  → Enables partition pruning.
- Format: Parquet > CSV/JSON  
  → Parquet: columnar, compressed, 10x smaller.
- Lifecycle: Move old data to Glacier (archive, 90% cheaper).

---

### 2. EMR (Elastic MapReduce)

**What:** Managed Hadoop/Spark cluster  
**Why:** No need to manage infrastructure, just run Spark  
**Cons:** More expensive than raw EC2, overkill for small jobs

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

Cluster auto-scales and shuts down when done.

---

### 3. Redshift (Data Warehouse)

**What:** MPP (Massively Parallel Processing) database, OLAP optimized  
**Why:** Query 100GB in seconds, cheap for analytics  
**Cons:** Needs schema design, not real-time

**Example architecture:**

- 4 nodes × 2TB each = 8TB capacity
- Columnar storage → fast queries
- Distributed execution

**Pricing:**

- `dc2.large`: $0.43/hour per node
- 4 nodes × $0.43 = $1.72/hour
- ≈ $1200/month (24/7)

**Optimizations:**

- Compression: zstd (90% smaller)
- Sort keys: order by common filter columns
- Dist keys: distribute by join key
- Vacuum: reclaim space from deletes

---

### 4. Lambda (Serverless Functions)

**What:** Run code without managing servers  
**Why:** Auto-scales, pay per execution  
**Cons:** Cold start (~1s), max timeout 15 min

**Use cases:**

- Trigger ETL on S3 upload
- Process streaming data
- API backends

**Example:** Trigger Lambda on S3 upload

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

**What:** AWS-managed ETL for serverless pipelines  
**Why:** Auto-scales, integrates with Redshift/Athena  
**Cons:** Limited control vs pure Spark

**Features:**

- **Data Catalog:** metadata registry
- **Crawlers:** auto-detect schema from S3
- **Jobs:** run PySpark/Scala
- **Triggers:** schedule or event-based

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

**What:** Managed Kafka alternative  
**Why:** Scales automatically, AWS-native  
**Cons:** Higher cost, less control

**Use cases:**

- Real-time ingestion
- Event streaming
- On-the-fly analytics

**Architecture:**

```
Data Source → Kinesis Streams → Lambda/Spark → S3/Redshift
```

**Specs:**

- 1 shard = 1 MB/sec ingestion
- 10 shards = 10 MB/sec
- Auto-scaling available
- Pricing: $0.035/shard-hour

---

## Real-World: End-to-End Pipeline

**Scenario:** E-commerce daily ETL

**Steps:**

1. Lambda triggered by S3 upload
2. EMR Spark cluster starts
3. Read raw S3 → Transform → Write processed S3
4. Redshift COPY from S3
5. Analytics on Redshift

**Architecture:**

```
AWS_ARCHITECTURE = {
    "ingestion": "S3 (raw data uploaded daily)",
    "processing": "EMR (Spark cluster, 5 nodes)",
    "storage": "S3 (processed Parquet)",
    "warehouse": "Redshift (4 nodes, 8TB)",
    "orchestration": "Lambda + EventBridge (cron triggers)",
    "monitoring": "CloudWatch (logs, metrics, alarms)",
    "cost": "$2000/month (S3 $100, EMR $500, Redshift $1200, Lambda $50, misc $150)"
}
```

**Alternatives:**

- AWS Glue only: easier but less control
- Databricks on EC2: higher control, higher complexity

---

## Trade-offs: AWS vs Alternatives

| Criteria             | AWS      | GCP             | Azure        |
| -------------------- | -------- | --------------- | ------------ |
| S3 equivalent        | S3       | GCS             | Blob Storage |
| Spark managed        | EMR      | Dataproc        | HDInsight    |
| Warehouse            | Redshift | BigQuery        | Synapse      |
| Serverless functions | Lambda   | Cloud Functions | Functions    |
| Streaming            | Kinesis  | Pub/Sub         | Event Hubs   |
| Cost                 | Mid      | Low (BigQuery)  | Mid          |
| Ease                 | Mid      | High (BigQuery) | Mid          |
| Control              | High     | Mid             | Mid          |

---

## Errores Comunes en Entrevista

- **"EMR es siempre más barato que Glue"** → Depende: para pequeños jobs, Glue es más cheap.
- **"S3 es ilimitado"** → Sí, pero el costo/performance importan. Haz partition smart.
- **Ignorar Data Transfer Out Cost** → Transferir fuera de AWS cuesta $0.09/GB.
- **Usar Redshift para datos real-time** → Redshift es batch-oriented. Usa Kinesis o Lambda.

---

## Preguntas de Seguimiento

1. **¿Cuándo usas EMR vs Glue?**
   - EMR: Control, custom Spark, big clusters.
   - Glue: Managed, serverless, smaller jobs.

2. **¿Redshift vs Athena?**
   - Redshift: Persistent warehouse, preloaded data, complex queries.
   - Athena: Query S3 directamente, cheap y ad-hoc.

3. **¿Cómo optimizas S3 para costo?**
   - Partitioning (partition pruning).
   - Compression (Parquet).
   - Lifecycle (archivar data vieja).
   - Right format (Parquet > CSV).

4. **¿Cold start de EMR clusters?**
   - 5–10 minutos para launch.
   - Solución: mantener warm clusters (más caro, más rápido).

---

## References

1. [AWS S3 - Official Docs](https://docs.aws.amazon.com/s3/)
2. [EMR - Spark on AWS](https://docs.aws.amazon.com/emr/)
3. [Redshift - Data Warehouse](https://docs.aws.amazon.com/redshift/)
4. [AWS Glue - ETL Service](https://docs.aws.amazon.com/glue/)
5. [Kinesis - Streaming](https://docs.aws.amazon.com/kinesis/)
