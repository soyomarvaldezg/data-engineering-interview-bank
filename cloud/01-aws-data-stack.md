# AWS Data Stack para Data Engineering

**Tags**: #cloud #aws #s3 #redshift #emr #lambda #real-interview  
**Empresas**: Amazon, Google, Meta, Netflix, Stripe  
**Dificultad**: Mid  
**Tiempo estimado**: 22 min  

---

## TL;DR

**AWS data stack**: S3 (storage) → EC2/EMR (compute) → Redshift (warehouse) → Lambda (functions). **Key services**: S3 (cheap, durable), EMR (Spark managed), Redshift (OLAP warehouse), Lambda (serverless), Glue (ETL), Kinesis (streaming). Trade-off: Cost vs Convenience vs Control. Best: S3 for storage, EMR for batch, Redshift for analytics.

---

## Concepto Core

- **Qué es**: AWS = infrastructure as a service. Puedes armar cualquier pipeline
- **Por qué importa**: AWS es industry standard. 40% de data engineers usan AWS
- **Principio clave**: Storage (S3) ≠ Compute (EMR). Separados = flexible

---

## Cómo explicarlo en entrevista

**Paso 1**: "AWS data stack = S3 (storage) + EMR (compute) + Redshift (warehouse)"

**Paso 2**: "S3 es data lake. EMR corre Spark. Redshift es warehouse. Separación = flexible"

**Paso 3**: "Alternativas: Glue (managed ETL), Lambda (serverless), Kinesis (streaming)"

**Paso 4**: "Trade-off: Control/Flexibility vs Convenience vs Cost"

---

## Arquitectura

┌──────────────────────────────────────┐
│ DATA SOURCES │
│ ├─ Databases (RDS) │
│ ├─ APIs │
│ ├─ On-prem systems │
│ └─ 3rd party SaaS │
└───────────────┬──────────────────────┘
│
▼
┌──────────────────────────────────────┐
│ AWS INGESTION │
│ ├─ Direct Connect (fast) │
│ ├─ S3 Transfer Acceleration │
│ ├─ Kinesis (real-time) │
│ └─ Database Migration Service (DMS) │
└───────────────┬──────────────────────┘
│
▼
┌──────────────────────────────────────┐
│ S3 DATA LAKE (Raw) │
│ ├─ s3://data-lake/raw/ │
│ ├─ Partitioned by source/date │
│ └─ Lifecycle policies (archive old) │
└───────────────┬──────────────────────┘
│
┌───────────┴───────────┐
│ │
▼ ▼
┌─────────────┐ ┌──────────────┐
│ EMR Cluster │ │ AWS Glue │
│ (Spark) │ │ (Managed ETL)│
│ Process │ │ Alternative │
│ Transform │ │ │
└──────┬──────┘ └──────┬───────┘
│ │
└────────┬───────────┘
│
▼
┌──────────────────────────────────────┐
│ S3 DATA LAKE (Processed) │
│ ├─ Parquet format │
│ ├─ Partitioned smart │
│ └─ Ready for analytics │
└───────────────┬──────────────────────┘
│
▼
┌──────────────────────────────────────┐
│ REDSHIFT WAREHOUSE │
│ ├─ Loaded via COPY command │
│ ├─ Optimized for OLAP queries │
│ └─ Ready for BI tools │
└───────────────┬──────────────────────┘
│
┌───────────┴───────────┐
│ │
▼ ▼
┌──────────────┐ ┌──────────────┐
│ Dashboards │ │ Analysts │
│ (Tableau) │ │ SQL queries │
└──────────────┘ └──────────────┘

text

---

## Key Services

### 1. S3 (Simple Storage Service)

What: Object storage (blobs, not files)
Why: Cheap ($0.023/GB/month), durable (99.999999999%), scalable
Cons: Eventually consistent, latency ~100ms

Pricing:

Storage: $0.023/GB/month

Transfer out: $0.09/GB

API calls: $0.0004 per 1000 requests

Architecture:
s3://bucket/path/object
├─ Bucket: Namespace (unique globally)
├─ Path: Folder-like structure (not real folders)
└─ Object: Actual file

Optimization:

Partitioning: s3://bucket/year=2024/month=01/day=15/
→ Enables partition pruning (queries scan only relevant partitions)

Format: Parquet > CSV/JSON
→ Parquet: columnar, compressed, 10x smaller

Lifecycle: Move old data to Glacier (archive, 90% cheaper)

text

### 2. EMR (Elastic MapReduce)

What: Managed Hadoop/Spark cluster
Why: Don't manage infrastructure, just use Spark
Cons: More expensive than raw EC2, overkill for small jobs

Example:

Launch 10-node Spark cluster
aws emr create-cluster
--name "daily-etl"
--release-label emr-6.9.0
--applications Name=Spark
--instance-count 10
--instance-type m5.xlarge

Submit Spark job
spark-submit s3://code/etl.py

Cluster auto-scales, shuts down when done
text

### 3. Redshift (Data Warehouse)

What: MPP (Massively Parallel Processing) database, OLAP optimized
Why: Query 100GB in seconds, cheap for analytics
Cons: Needs schema design, not real-time

Example architecture:

4 nodes × 2TB each = 8TB capacity

Columnar storage (queries fast)

Distributed execution

Pricing:

dc2.large: $0.43/hour per node

4 nodes × $0.43 = $1.72/hour

≈ $1200/month (24/7)

Optimizations:

Compression: zstd (90% smaller)

Sort keys: Data ordered by common filter columns

Dist keys: Distribute by join key (less shuffling)

Vacuum: Reclaim space from deletes

text

### 4. Lambda (Serverless Functions)

What: Run code without managing servers
Why: Auto-scales, pay per execution, no idle cost
Cons: Cold start (~1s), timeout limit (15 min)

Use cases:

Trigger ETL on S3 file upload

Process streaming data

API backends

Example: Trigger Lambda on S3 upload

S3 event → Lambda function → Process → Save results
@lambda_handler
def handler(event, context):
bucket = event['Records']['s3']['bucket']['name']
key = event['Records']['s3']['object']['key']

text
# Process file
df = pd.read_csv(f"s3://{bucket}/{key}")
result = process(df)

# Save
result.to_parquet(f"s3://output/{key}.parquet")

return {'statusCode': 200}
Pricing: $0.20 per 1M invocations + compute

text

### 5. Glue (Managed ETL)

What: AWS's answer to "manage data pipelines"
Why: Serverless, auto-scales, integrates with Redshift/Athena
Cons: Limited control vs Spark directly

Features:

Data Catalog: Metadata registry (tables, schemas)

Crawlers: Auto-detect schema from S3 files

Jobs: Run PySpark/Scala without managing clusters

Triggers: Schedule or event-based

Example: Glue job
import awsglue
from awsglue.context import GlueContext

sc = SparkContext()
glueContext = GlueContext(sc)

Read from S3
dyf = glueContext.create_dynamic_frame.from_options(
"s3",
{"path": "s3://data-lake/raw/customers/"}
)

Transform
transformed = dyf.map(lambda x: clean_data(x))

Write to Redshift
glueContext.write_dynamic_frame.from_jdbc_conf(
frame=transformed,
catalog_connection="redshift-connection",
connection_options={"dbtable": "public.customers"},
transformation_ctx="to_redshift"
)

text

### 6. Kinesis (Streaming)

What: Managed Kafka alternative
Why: Scales automatically, AWS-native
Cons: More expensive than self-managed Kafka, less flexible

Use cases:

Real-time data ingestion

Event streaming

Analytics on-the-fly

Architecture:
Data Source → Kinesis Streams → Consumer Lambda/Spark → S3/Redshift

Shards: Parallelize data

1 shard = 1MB/sec ingestion

10 shards = 10MB/sec

Auto-scaling available

Pricing: $0.035/shard-hour

text

---

## Real-World: End-to-End Pipeline

Scenario: E-commerce daily ETL
Step 1: Lambda triggered by S3 upload (new daily file)
Step 2: EMR Spark cluster starts
Step 3: Read raw S3 → Transform → Write processed S3
Step 4: Redshift COPY from S3
Step 5: Analytics queries on Redshift
Architecture:
AWS_ARCHITECTURE = {
"ingestion": "S3 (raw data uploaded daily)",
"processing": "EMR (Spark cluster, 5 nodes)",
"storage": "S3 (processed Parquet)",
"warehouse": "Redshift (4 nodes, 8TB)",
"orchestration": "Lambda + EventBridge (cron triggers)",
"monitoring": "CloudWatch (logs, metrics, alarms)",
"cost": "$2000/month (S3 $100, EMR $500, Redshift $1200, Lambda $50, misc $150)"
}

vs Alternatives:
AWS Glue only: Easier but less control, similar cost
Databricks on EC2: More control, more complex
text

---

## Trade-offs: AWS vs Alternatives

| Criteria | AWS | GCP | Azure |
|----------|-----|-----|-------|
| **S3 equivalent** | S3 | GCS | Blob Storage |
| **Spark managed** | EMR | Dataproc | HDInsight |
| **Warehouse** | Redshift | BigQuery | Synapse |
| **Serverless functions** | Lambda | Cloud Functions | Functions |
| **Streaming** | Kinesis | Pub/Sub | Event Hubs |
| **Cost** | Mid | Low (BigQuery) | Mid |
| **Ease** | Mid | High (BigQuery) | Mid |
| **Control** | High | Mid | Mid |

---

## Errores Comunes en Entrevista

- **Error**: "EMR es siempre más barato que Glue" → **Solución**: Depende. Para pequeños jobs, Glue es más cheap

- **Error**: "S3 es ilimitado" → **Solución**: Sí, pero costo/performance importan. Partition smart

- **Error**: No pensar en Data Transfer out cost → **Solución**: Transfer fuera de AWS = $0.09/GB (caro)

- **Error**: Using Redshift para datos real-time → **Solución**: Redshift es batch-oriented. Usa Kinesis/Lambda para real-time

---

## Preguntas de Seguimiento

1. **"¿Cuándo usas EMR vs Glue?"**
   - EMR: Control, custom Spark, big clusters
   - Glue: Managed, serverless, smaller jobs

2. **"¿Redshift vs Athena?"**
   - Redshift: Persistent, pre-loaded, complex queries
   - Athena: Query S3 directly, cheaper for ad-hoc

3. **"¿Cómo optimizas S3 para cost?"**
   - Partitioning (partition pruning)
   - Compression (Parquet)
   - Lifecycle (archive old)
   - Right format (Parquet vs CSV)

4. **"¿Cold start de EMR clusters?"**
   - 5-10 min to launch
   - Solución: Keep warm clusters (más caro pero faster)

---

## References

- [AWS S3 - Official Docs](https://docs.aws.amazon.com/s3/)
- [EMR - Spark on AWS](https://docs.aws.amazon.com/emr/)
- [Redshift - Data Warehouse](https://docs.aws.amazon.com/redshift/)
- [AWS Glue - ETL Service](https://docs.aws.amazon.com/glue/)
- [Kinesis - Streaming](https://docs.aws.amazon.com/kinesis/)

