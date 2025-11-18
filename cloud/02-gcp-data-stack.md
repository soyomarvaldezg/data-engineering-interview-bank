# GCP Data Stack para Data Engineering

**Tags**: #cloud #gcp #bigquery #dataflow #cloud-storage #pub-sub #real-interview

---

## TL;DR

GCP data stack: **Cloud Storage (S3 equivalent)** → **Dataflow (managed Beam/Spark)** → **BigQuery (OLAP + ML)**.
Ventaja clave: **BigQuery** = **serverless warehouse** + **SQL** + **ML** en uno; muchas veces no necesitas **data lake** separado.
Trade-off: Más fácil de operar vs menos control fino.
Ideal para: **Analytics-first**, **SQL-first**, equipos con poco overhead de infraestructura.

---

## Concepto Core

- Qué es: **GCP** es la nube de Google con filosofía **managed-first** (menos DIY, más **serverless**).
- Por qué importa: Adopción creciente, especialmente **BigQuery** en **analytics**/**ML**.
- Principio clave: Modelo “**analytics warehouse**” (**BigQuery**) vs modelo “**data lake**” (**Cloud Storage** + **engines**).

---

## Arquitectura

```
┌──────────────────────────────────────┐
│ DATA SOURCES                        │
│ ├─ Firestore / Datastore            │
│ ├─ Cloud SQL (MySQL, PostgreSQL)    │
│ ├─ APIs                             │
│ └─ On-prem databases                │
└───────────────┬──────────────────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ GCP INGESTION                       │
│ ├─ Cloud Pub/Sub (real-time)        │
│ ├─ Datastream (CDC replication)     │
│ ├─ Cloud Transfer Service           │
│ └─ Dataflow (ETL)                   │
└───────────────┬──────────────────────┘
                │
       ┌────────┴───────────┐
       │                    │
       ▼                    ▼
┌─────────────┐     ┌──────────────┐
│ Cloud       │     │ Dataflow     │
│ Storage     │     │ (Beam/Spark) │
│ (Data Lake) │     │ Transform    │
└──────┬──────┘     └──────┬───────┘
       │                    │
       └────────┬───────────┘
                │
                ▼
┌──────────────────────────────────────┐
│ BIGQUERY                             │
│ ├─ Serverless warehouse              │
│ ├─ SQL analytics                     │
│ ├─ BQML (ML integrado)               │
│ └─ BI Engine (in-memory cache)       │
└───────────────┬──────────────────────┘
                │
       ┌────────┴───────────┐
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│ Looker       │     │ Looker Studio│
│ (BI)         │     │ (Free viz)   │
└──────────────┘     └──────────────┘
```

---

## Key Services

### 1) Cloud Storage (GCS)

- What: **Object storage** (similar a **AWS S3**).
- Why: Barato, alta durabilidad, integra bien con **BigQuery** y **Dataflow**.
- Cons: Mismas consideraciones de **S3** (consistencia eventual en listados, **egress costs**).

**Features:**

- **Multi-regional replication** (**high availability**).
- **Object Lifecycle** (mover a **Coldline**/**Archive** para ahorro).
- **Signed URLs** (acceso temporal).

**URI pattern:**

```
gs://bucket/path/object
```

---

### 2) BigQuery (Game Changer)

- What: **Serverless data warehouse** + **SQL** + **ML** (**BQML**).
- Why: **Query** escala **TB**–**PB** sin gestionar **clusters**; **autoscaling**; **pay-per-query** u opciones de **flat-rate**.
- Diferencia vs **Redshift**: En **BigQuery** pagas por bytes escaneados y no dimensionas **clusters**; **Redshift** requiere **sizing** y pagar por **nodos**.

**Features:**

- **SQL dialect** estándar (con extensiones analíticas).
- **BQML** para entrenar modelos sin salir de **SQL**.
- **Federated queries** (consultar datos en **Cloud Storage** o **Bigtable**).
- **BI Engine** (**in-memory cache** para latencias de **BI** muy bajas).

**Example: SQL de agregación**

```
SELECT
  customer_id,
  SUM(amount) AS total_spent,
  COUNT(*)    AS num_purchases
FROM `project.dataset.transactions`
WHERE date >= '2024-01-01'
GROUP BY customer_id
ORDER BY total_spent DESC;
```

**Example: Entrenar un modelo con BQML**

```
CREATE OR REPLACE MODEL `project.dataset.fraud_model`
OPTIONS(model_type = 'linear_reg') AS
SELECT
  amount,
  customer_age,
  is_fraud AS label
FROM `project.dataset.transactions`
WHERE date >= '2024-01-01';
```

**Example: Predicción con el modelo**

```
SELECT
  amount,
  customer_age,
  ml.predict_value(MODEL `project.dataset.fraud_model`, t) AS fraud_prob
FROM `project.dataset.transactions_new` AS t;
```

---

### 3) Dataflow (Managed Beam)

- What: **Managed Apache Beam** para **batch** y **streaming**; alternativa a **Spark**/**EMR**.
- Why: **Serverless autoscaling**, integración nativa con **Pub/Sub**, **GCS**, **BigQuery**.
- Cons: Ecosistema **Beam** menor que **Spark**; curva de aprendizaje en **SDKs** (**Python**/**Java**).

**Use cases:**

- **Batch ETL** sobre **GCS** → **BigQuery** (**Parquet**/**CSV**/**JSON**).
- **Streaming pipelines** (**Pub/Sub** → **Dataflow** → **BigQuery**/**Sink**).

**Example: Batch pipeline (Beam)**

```
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

with beam.Pipeline(runner='DataflowRunner', options=PipelineOptions()) as p:
    (
        p
        | 'Read from GCS' >> beam.io.ReadFromText('gs://bucket/input.json')
        | 'Parse JSON'   >> beam.Map(parse_json)
        | 'Filter valid' >> beam.Filter(is_valid)
        | 'Write to BQ'  >> beam.io.WriteToBigQuery('project:dataset.table')
    )
```

---

### 4) Pub/Sub (Streaming)

- What: **Managed messaging** (alternativa a **Kafka** para **event streaming**).
- Why: **Fully managed**, **auto-scale**, bajo overhead operativo.
- Cons: **API** diferente a **Kafka**; menor flexibilidad para ciertos patrones.

**Use cases:**

- **Real-time ingestion** (**event-driven**).
- **Decoupling producers**/**consumers**.
- **Stream processing** con **Dataflow**.

**Example: Publisher**

```
from google.cloud import pubsub_v1
import json

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path('project-id', 'transactions')

for tx in incoming_transactions:
    publisher.publish(topic_path, json.dumps(tx).encode('utf-8'))
```

---

### 5) Datastream (CDC Replication)

- What: **Serverless CDC** para replicar cambios de **MySQL**/**PostgreSQL**/**Oracle**.
- Why: **Near real-time replication** hacia **BigQuery**/**GCS** sin **ETL** manual.
- Cons: Soporte limitado a fuentes/destinos compatibles.

**Flujo típico:**

- **Source**: **MySQL prod** → **Datastream** → **GCS staging** → **BigQuery external**/**federated** o **load incremental**.

---

## Why BigQuery Changes Everything

### Old Way (estilo AWS)

- Extraer de bases → **S3** (**data lake**) → **EMR Spark** (**ETL**) → **Redshift** (**load**) → **Query** → **ML** aparte.
- Costo: **infraestructura** + **clusters** dedicados.
- Latencia: horas en **batch** tradicionales.

### New Way (GCP con BigQuery)

- Extraer/**stream** a **BigQuery** (o a **GCS**).
- **Query** directo en **BigQuery**.
- Entrenar modelos con **BQML** en el mismo **warehouse**.
- Costo: **pay-as-you-go** por bytes escaneados o **flat-rate**; menor **ops**.
- Latencia: segundos a minutos, especialmente con **streaming** y **BI Engine**.

---

## Real-World: E-commerce Analytics

**Scenario:** Análisis de una tienda Shopify en tiempo real/**light-batch**.

**Arquitectura:**

1. Shopify **API** → **Cloud Functions** (**pull**/**push events**).
2. Publicar eventos a **Pub/Sub** (**topic orders**).
3. **Dataflow** (**stream**) consume, valida y enriquece.
4. **Sink** en **BigQuery** (**dataset ecommerce**).
5. **Looker**/**Looker Studio** sobre **BigQuery** para **dashboards**.

**Step 1: Cloud Function (publisher)**

```
from google.cloud import pubsub_v1
import functions_framework, json

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path('my-project', 'orders')

@functions_framework.http
def process_order(request):
    order = request.get_json()
    publisher.publish(topic_path, json.dumps(order).encode('utf-8'))
    return 'Published'
```

**Step 2: Dataflow job (streaming consumer)**

```
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
import json
from datetime import datetime

def enrich_order(order):
    order['processed_at'] = datetime.utcnow().isoformat()
    return order

with beam.Pipeline(runner='DataflowRunner', options=PipelineOptions(streaming=True)) as p:
    (
        p
        | 'Read from Pub/Sub' >> beam.io.gcp.pubsub.ReadFromPubSub(
              topic='projects/my-project/topics/orders')
        | 'Parse'  >> beam.Map(lambda b: json.loads(b))
        | 'Enrich' >> beam.Map(enrich_order)
        | 'Write'  >> beam.io.WriteToBigQuery(
              'my-project:ecommerce.orders',
              write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND)
    )
```

**Step 3: BigQuery queries (reporting)**

```
SELECT
  DATE(processed_at) AS date,
  COUNT(*)          AS total_orders,
  SUM(total_price)  AS revenue,
  AVG(total_price)  AS avg_order_value
FROM `my-project.ecommerce.orders`
GROUP BY date
ORDER BY date DESC;
```

---

## GCP vs AWS vs Azure

| Feature        | GCP          | AWS                | Azure                |
| -------------- | ------------ | ------------------ | -------------------- |
| Data Warehouse | BigQuery     | Redshift           | Synapse              |
| Ease           | Highest      | Mid                | Mid                  |
| Cost           | Low (pay/TB) | Mid                | Mid                  |
| ML Built-in    | Yes (BQML)   | Limited            | Limited              |
| Serverless     | Yes (mostly) | Mixed              | Mixed                |
| SQL            | Standard     | Proprietary mix    | T-SQL                |
| Best for       | Analytics/ML | Enterprise/Complex | Enterprise Microsoft |

---

## Errores Comunes en Entrevista

- “**BigQuery** siempre es más barato” → Depende: **on-demand** vs **flat-rate**; **bad queries** escanean más bytes y suben costo.
- “**BigQuery** reemplaza **OLTP**” → **BigQuery** es **OLAP**; para **transactional** usa **Cloud SQL**/**Spanner**/**Firestore**.
- No usar **BI Engine** → Pierdes latencia ultra-baja en **dashboards**; habilítalo cuando convenga.
- Consultar **raw files** en **GCS** sin gobernanza → Prefiere **particionar**/**clusterizar** tablas en **BigQuery** o **federated** con criterio.

---

## Preguntas de Seguimiento

1. ¿Cuándo **BigQuery** vs **Redshift**?

- **BigQuery**: **serverless**, **pay-per-query**, **BQML**, menos **ops**.
- **Redshift**: más control de **compute**, **tuning** detallado, **ecosistemas** existentes en **AWS**.

2. ¿**Pub/Sub** vs **Kafka**?

- **Pub/Sub**: **fully managed**, sencillo, **GCP-native**.
- **Kafka**: más flexible, gran **ecosistema**; requiere más **ops** si **self-managed**.

3. ¿**BQML** vs **custom ML**?

- **BQML**: rápido para modelos estándar en **SQL**.
- **Custom**: **TensorFlow**/**Vertex AI** para control avanzado y **pipelines MLOps**.

4. ¿**BigQuery pricing on-demand** vs **flat-rate**?

- **On-demand**: paga por **TB escaneado**; ideal **workloads** variables.
- **Flat-rate**: **costo fijo mensual**; ideal **heavy**/**estable** con **SLAs de costos**.

---

## References

- [Data Engineering in the Cloud: Comparing AWS, Azure and Google Cloud Platform](https://www.geeksforgeeks.org/data-engineering/data-engineering-in-the-cloud-comparing-aws-azure-and-google-cloud-platform/)
- [Data Engineering in the Cloud: Comparing AWS, Azure and GCP](https://sunscrapers.com/blog/data-engineering-in-the-cloud-comparing-aws-azure-and-gcp/)
- [Which is better? GCP data engineering vs AWS data engineering](https://visualpathblogs.com/gcp-data-engineering/which-is-better-gcp-data-engineering-and-aws-data-engineering/)
- [AWS vs Azure vs GCP as Data Engineer](https://www.reddit.com/r/dataengineersindia/comments/1gmdd86/aws_vs_azure_vs_gcp_as_data_engineer/)
- [Cloud Comparison Guide for Data Engineers | LinkedIn Post by Pooja Jain](https://www.linkedin.com/posts/pooja-jain-898253106_cloud-comparison-guide-for-data-engineers-activity-7357379682721304576-X6HA)
- [AWS vs Azure vs GCP - Which cloud should you learn?](https://dataengineeracademy.com/module/aws-vs-azure-vs-gcp-which-cloud-should-you-learn/)
- [Google Cloud vs AWS](https://www.netcomlearning.com/blog/google-cloud-vs-aws)
- [AWS, Azure and GCP Service Comparison for Data Science and AI | Cheat Sheet](https://www.datacamp.com/cheat-sheet/aws-azure-and-gcp-service-comparison-for-data-science-and-ai)
