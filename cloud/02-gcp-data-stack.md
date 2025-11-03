# GCP Data Stack para Data Engineering

**Tags**: #cloud #gcp #bigquery #dataflow #cloud-storage #pub-sub #real-interview  
**Empresas**: Google, Meta, Airbnb, Spotify, Shopify  
**Dificultad**: Mid  
**Tiempo estimado**: 20 min  

---

## TL;DR

**GCP data stack**: Cloud Storage (S3 equivalent) → Dataflow (managed Beam/Spark) → BigQuery (OLAP + ML). **Key advantage**: BigQuery = serverless warehouse + SQL + ML in one. No need separate data lake. **Trade-off**: Easier vs Less control. Best for: Analytics companies, SQL-first, less infrastructure.

---

## Concepto Core

- **Qué es**: GCP = Google's cloud. Diferente filosofía vs AWS (managed > DIY)
- **Por qué importa**: Growing adoption, especially BigQuery (40% of analytics shops)
- **Principio clave**: "Analytics warehouse" model (BigQuery) vs "Data Lake" model (AWS S3)

---

## Arquitectura

┌──────────────────────────────────────┐
│ DATA SOURCES │
│ ├─ Firestore / Datastore │
│ ├─ Cloud SQL (MySQL, PostgreSQL) │
│ ├─ APIs │
│ └─ On-prem databases │
└───────────────┬──────────────────────┘
│
▼
┌──────────────────────────────────────┐
│ GCP INGESTION │
│ ├─ Cloud Pub/Sub (real-time) │
│ ├─ Datastream (CDC replication) │
│ ├─ Cloud Transfer Service │
│ └─ Dataflow (ETL) │
└───────────────┬──────────────────────┘
│
┌───────────┴───────────┐
│ │
▼ ▼
┌─────────────┐ ┌──────────────┐
│ Cloud │ │ Dataflow │
│ Storage │ │ (Beam/Spark) │
│ (Data Lake) │ │ Transform │
└──────┬──────┘ └──────┬───────┘
│ │
└────────┬───────────┘
│
▼
┌──────────────────────────────────────┐
│ BIGQUERY │
│ ├─ Serverless warehouse │
│ ├─ SQL analytics │
│ ├─ ML (BigQuery ML) │
│ └─ BI Engine (caching) │
└───────────────┬──────────────────────┘
│
┌───────────┴───────────┐
│ │
▼ ▼
┌──────────────┐ ┌──────────────┐
│ Looker │ │ Data Studio │
│ (BI Tool) │ │ (Free viz) │
└──────────────┘ └──────────────┘

text

---

## Key Services

### 1. Cloud Storage (GCS)

What: Object storage (like AWS S3)
Why: Cheap, integrates with BigQuery
Cons: Similar to S3

Pricing:

Storage: $0.020/GB/month (cheaper than S3)

Transfer out: $0.12/GB (more than AWS)

Features:

Multi-regional replication (HA)

Object Lifecycle (archive to Coldline, Archive)

Signed URLs (time-limited access)

Architecture: gs://bucket/path/object

text

### 2. BigQuery (Game Changer)

What: Serverless data warehouse + SQL + ML
Why: Query 1TB in seconds, no clusters to manage
Cost: $6.25 per TB queried (or flat rate $2000/month)

Key Difference from Redshift:

Redshift: You manage compute (pay for nodes)

BigQuery: You query data, pay per byte scanned

BigQuery: Auto-scales (no cluster sizing)

Features:

SQL dialect (mostly standard SQL)

ML capabilities built-in (BQML)

Federated queries (query Cloud Storage directly)

BI Engine (in-memory cache, 100x faster)

Example: Simple query
SELECT
customer_id,
SUM(amount) as total_spent,
COUNT(*) as num_purchases
FROM project.dataset.transactions
WHERE date >= '2024-01-01'
GROUP BY customer_id
ORDER BY total_spent DESC;

Cost: ~1GB scanned = $6.25

Example: ML model (without code)
CREATE OR REPLACE MODEL project.dataset.fraud_model
OPTIONS(
model_type='linear_reg'
) AS
SELECT
amount,
customer_age,
is_fraud as label
FROM project.dataset.transactions
WHERE date >= '2024-01-01';

Now predict
SELECT
amount,
customer_age,
ml.predict_value(MODEL project.dataset.fraud_model, *) as fraud_prob
FROM project.dataset.transactions_new;

text

### 3. Dataflow (Managed Beam)

What: Managed Apache Beam (Spark alternative)
Why: Serverless, auto-scales, integrated with Pub/Sub
Cons: Beam is less popular than Spark, less ecosystem

Use cases:

Batch ETL (like Spark)

Streaming (like Kafka consumers)

Pricing:

$0.07 per vCPU-hour (cheaper than EMR)

No cluster overhead

Example: Batch pipeline
import apache_beam as beam

with beam.Pipeline(runner='DataflowRunner') as p:
(p
| 'Read from GCS' >> beam.io.ReadFromText('gs://bucket/input.txt')
| 'Parse JSON' >> beam.Map(parse_json)
| 'Filter invalid' >> beam.Filter(is_valid)
| 'Write to BigQuery' >> beam.io.WriteToBigQuery('project:dataset.table')
)

text

### 4. Pub/Sub (Streaming)

What: Kafka alternative (message broker)
Why: Fully managed, no ops, auto-scales
Cons: Different API than Kafka, less flexible

Use cases:

Real-time data pipeline

Event streaming

Decoupling producers/consumers

Pricing: $0.04 per GB

Architecture:
Publisher → Pub/Sub Topic → Subscriber (Lambda-like)

Example: Real-time transaction processing

Publisher (credit card service)
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path('project-id', 'transactions')

for transaction in incoming_transactions:
publisher.publish(topic_path, json.dumps(transaction).encode('utf-8'))

Subscriber (Dataflow job)
Consume from Pub/Sub, enrich, write to BigQuery in real-time
text

### 5. Datastream (CDC Replication)

What: Serverless CDC (Change Data Capture)
Why: Replicate from MySQL/PostgreSQL/Oracle to BigQuery
Cons: Only supports certain databases

Example: Replicate customers from MySQL to BigQuery

Set up Datastream connection
Source: MySQL (production database)
Destination: BigQuery (analytics warehouse)
Result: Real-time replication, no manual ETL
text

---

## Why BigQuery Changes Everything

### Old Way (AWS-style)
Extract from databases

S3 (data lake)

EMR Spark job (ETL)

Redshift (load)

Query Redshift

Build separate ML models
Cost: $2000+/month
Latency: Hours

text

### New Way (GCP BigQuery)
Extract from databases

Stream to BigQuery (or GCS)

Query directly in BigQuery

Train ML in BigQuery (BQML)
Cost: $500-1000/month (pay-as-you-go)
Latency: Seconds

text

**BigQuery = one system for everything (warehouse + ML + BI)**

---

## Real-World: E-commerce Analytics

Scenario: Shopify store analysis
Architecture:
1. Shopify API → Cloud Functions
2. Write to Pub/Sub
3. Dataflow consumer (real-time transformation)
4. Write to BigQuery
5. Looker dashboards on BigQuery
Step 1: Cloud Function (triggered by API)
from google.cloud import pubsub_v1
import functions_framework

@functions_framework.http
def process_order(request):
order = request.get_json()

text
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path('my-project', 'orders')

publisher.publish(topic_path, json.dumps(order).encode('utf-8'))

return 'Published'
Step 2: Dataflow job (streaming consumer)
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

def enrich_order(order):
# Add timestamp, validate, enrich
order['processed_at'] = datetime.now()
return order

with beam.Pipeline(runner='DataflowRunner', options=PipelineOptions()) as p:
(p
| 'Read from Pub/Sub' >> beam.io.gcp.pubsub.ReadFromPubSub(topic='projects/my-project/topics/orders')
| 'Parse' >> beam.Map(lambda x: json.loads(x))
| 'Enrich' >> beam.Map(enrich_order)
| 'Write to BigQuery' >> beam.io.WriteToBigQuery('my-project:ecommerce.orders')
)

Step 3: BigQuery queries
SELECT
DATE(processed_at) as date,
COUNT(*) as total_orders,
SUM(total_price) as revenue,
AVG(total_price) as avg_order_value
FROM my-project.ecommerce.orders
GROUP BY date
ORDER BY date DESC;

Cost: BigQuery ($100-200/month) + Dataflow ($50/month) = $150/month
vs AWS EMR: $500-1000/month
text

---

## GCP vs AWS vs Azure

| Feature | GCP | AWS | Azure |
|---------|-----|-----|-------|
| **Data Warehouse** | BigQuery | Redshift | Synapse |
| **Ease** | Highest | Mid | Mid |
| **Cost** | Low | Mid | Mid |
| **ML Built-in** | Yes (BQML) | Limited | Limited |
| **Serverless** | Yes (mostly) | Limited | Limited |
| **SQL** | Standard | Proprietary | T-SQL |
| **Best for** | Analytics/ML | Enterprise/Complex | Enterprise Microsoft |

---

## Errores Comunes en Entrevista

- **Error**: "BigQuery es siempre más barato" → **Solución**: Depende. Flat rate puede ser más caro si queries small

- **Error**: Pensando que BigQuery puede reemplazar OLTP (transactional) → **Solución**: BigQuery = OLAP (analytics), not transactional

- **Error**: No usando BI Engine cache → **Solución**: BI Engine = 100x faster for BI queries, almost free

- **Error**: Querying raw GCS files sin proyectos** → **Solución**: Load a BigQuery primero o usa federated queries (más caro)

---

## Preguntas de Seguimiento

1. **"¿Cuándo BigQuery vs Redshift?"**
   - BigQuery: Easier, serverless, built-in ML
   - Redshift: More control, established

2. **"¿Pub/Sub vs Kafka?"**
   - Pub/Sub: Managed, simpler, GCP-native
   - Kafka: More flexible, larger ecosystem

3. **"¿BQML vs custom ML?"**
   - BQML: Quick, simple, good for standard models
   - Custom: TensorFlow, more control

4. **"¿BigQuery pricing: on-demand vs flat-rate?"**
   - On-demand: $6.25/TB, good for variable workloads
   - Flat-rate: $2000/month, good for heavy users

---

## References

- [BigQuery - Official Docs](https://cloud.google.com/bigquery/docs)
- [Dataflow - Beam on GCP](https://cloud.google.com/dataflow)
- [Cloud Storage - Object Storage](https://cloud.google.com/storage/docs)
- [Pub/Sub - Messaging](https://cloud.google.com/pubsub/docs)
- [BQML - ML in BigQuery](https://cloud.google.com/bigquery/docs/bqml-intro)

