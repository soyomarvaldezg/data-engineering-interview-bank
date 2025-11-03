# Diseñar Sistema Real-Time: Kafka + Streaming

**Tags**: #system-design #streaming #kafka #real-time #architecture #real-interview  
**Empresas**: Amazon, Google, Meta, Netflix  
**Dificultad**: Senior  
**Tiempo estimado**: 30 min  

---

## TL;DR

Real-time streaming = datos procesados milisegundos después de ocurrir (vs batch = horas). Kafka = message broker (durabilidad, ordenamiento). Spark Streaming o Flink = procesamiento. Trade-off: Simple (Batch) vs Complex (Real-time), pero algunas apps NECESITAN real-time (fraud detection, live dashboards, recommendations).

---

## Problema Real

**Escenario:**

- Fintech: Detección de fraude en transacciones
- Necesita: Detectar fraude en < 500ms (bloquear transacción)
- Volumen: 1M transacciones/min (16.7k/seg)
- Datos históricos: 1 año de transacciones (para modelos ML)
- Business: Reducir fraude en 50%, cero falsos positivos

**Preguntas:**

1. ¿Cómo diseñas pipeline real-time?
2. ¿Kafka es necesario? ¿Alternativas?
3. ¿Cómo mantienes low latency?
4. ¿Cómo manejas errores sin perder datos?
5. ¿Cómo escalas a 10M transacciones/min?

---

## Solución: Real-Time Fraud Detection Pipeline

### Arquitectura

┌─────────────────────────────────────────────────────────────┐
│ SOURCES (Transacciones) │
│ ├─ Credit Card API │
│ ├─ Mobile App │
│ └─ Web Platform │
└─────────────┬───────────────────────────────────────────────┘
│ (1M tx/min, ~16.7k/seg)
▼
┌─────────────────────────────────────────────────────────────┐
│ KAFKA CLUSTER (Message Broker) │
│ Topic: transactions (partitions=100, replication=3) │
│ ├─ Durability: Persist 7 days │
│ ├─ Retention: 1 week (para replay) │
│ └─ Latency: < 10ms end-to-end │
└─────────────┬───────────────────────────────────────────────┘
│
┌─────────┴─────────┐
│ │
▼ ▼
┌─────────────────┐ ┌──────────────────────────┐
│ Stream Processor│ │ Historical Data (Parquet)│
│ (Spark/Flink) │ │ S3: 1 year transactions │
├─────────────────┤ └──────────────────────────┘
│ Enrich datos │ │
│ + ML inference │ │
│ Detect fraud │ │
│ Latency: 200ms │ │
└────────┬────────┘ │
│ │
▼ ▼
┌─────────────────────────────────────────────────────────────┐
│ DECISION ENGINE (Low Latency KV Store) │
│ ├─ Redis: user profiles, ML model cache │
│ ├─ Lookup: 1-5ms │
│ └─ Decision: APPROVE/BLOCK/REVIEW │
└─────────────┬───────────────────────────────────────────────┘
│
┌─────────┴─────────┐
│ │
▼ ▼
┌──────────────┐ ┌──────────────┐
│ ACTION: APPROVE
│ Back to │ │ ACTION: BLOCK
│ Merchant │ │ Send to QA │
└──────────────┘ └──────────────┘
│
▼
┌──────────────────┐
│ KAFKA Topic: │
│ blocked-txns │
│ (for review) │
└──────────────────┘

text

---

### Stack Tecnológico

| Componente | Tech | Razón | Costo |
|-----------|------|-------|-------|
| Message Bus | Kafka | Durabilidad, replayability | $5k/mes |
| Streaming | Spark Streaming | Flexible, integrado | $3k/mes |
| ML Model Store | S3 + Redis | Fast lookups | $1k/mes |
| Decision Store | Redis | < 5ms latency | $2k/mes |
| Monitoring | Datadog | Real-time metrics | $2k/mes |
| **Total** | | | ~$13k/mes |

---

### Pipeline Detallado

T=0ms: Transaction arrives at API
└─ Merchant: "Is this legit?"

T=1ms: Event published to Kafka
├─ Partition: hash(user_id) → consistencia
└─ Replication: 3 copies (durabilidad)

T=10ms: Stream processor reads from Kafka
├─ Lookup user profile (Redis): "Has this user done this before?"
├─ Enrich: amount, merchant category, location
└─ Prepare features for ML

T=50ms: ML Model inference
├─ Load model (cached in memory)
├─ Score: fraud_probability = 0.85
├─ Decision: IF > 0.8 THEN REVIEW
└─ Result: Send to decision engine

T=100ms: Decision engine query Redis
├─ User risk profile: "HIGH"
├─ Similar fraud patterns: YES
└─ FINAL DECISION: BLOCK

T=150ms: Return to merchant
├─ Response: DECLINED
└─ Reason: "Suspicious activity detected"

T=151ms: Log to Kafka topic "blocked-transactions"
└─ For human review team

T=200ms: End-to-end latency = 200ms ✓ (< 500ms SLA)

text

---

### Code: Spark Streaming Processor

from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json, to_json
from pyspark.sql.types import StructType, StructField, StringType, DoubleType
import redis
import pickle

spark = SparkSession.builder
.appName("FraudDetection")
.getOrCreate()

===== READ from Kafka =====
transactions_df = spark
.readStream
.format("kafka")
.option("kafka.bootstrap.servers", "kafka:9092")
.option("subscribe", "transactions")
.option("startingOffsets", "latest")
.load()

===== PARSE JSON =====
schema = StructType([
StructField("transaction_id", StringType()),
StructField("user_id", StringType()),
StructField("amount", DoubleType()),
StructField("merchant_id", StringType()),
StructField("timestamp", StringType())
])

parsed_df = transactions_df.select(
from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

===== ENRICH with Redis lookups =====
def get_user_profile(user_id):
redis_client = redis.Redis(host='redis', port=6379)
profile = redis_client.get(f"user:{user_id}")
return pickle.loads(profile) if profile else {}

UDF for Redis lookup
from pyspark.sql.functions import udf
get_profile_udf = udf(get_user_profile, StringType())

enriched_df = parsed_df.withColumn(
"user_profile",
get_profile_udf(col("user_id"))
)

===== ML INFERENCE =====
from pyspark.ml import PipelineModel

model = PipelineModel.load("s3://models/fraud_detection_v1")

predictions_df = model.transform(enriched_df).select(
"*",
col("prediction").alias("fraud_score")
)

===== DECISION LOGIC =====
decisions_df = predictions_df.withColumn(
"decision",
when(col("fraud_score") > 0.8, "BLOCK")
.when(col("fraud_score") > 0.5, "REVIEW")
.otherwise("APPROVE")
)

===== OUTPUT =====
Approved transactions back to Kafka
approved = decisions_df.filter(col("decision") == "APPROVE").select("")
approved.select(to_json(struct("")).alias("value"))
.writeStream
.format("kafka")
.option("kafka.bootstrap.servers", "kafka:9092")
.option("topic", "approved-transactions")
.option("checkpointLocation", "/tmp/checkpoint_approved")
.start()

Blocked transactions to separate topic (for review)
blocked = decisions_df.filter(col("decision") != "APPROVE").select("")
blocked.select(to_json(struct("")).alias("value"))
.writeStream
.format("kafka")
.option("kafka.bootstrap.servers", "kafka:9092")
.option("topic", "blocked-transactions")
.option("checkpointLocation", "/tmp/checkpoint_blocked")
.start()

spark.streams.awaitAnyTermination()

text

---

## Trade-offs Analizados

### Real-Time vs Batch

| Aspecto | Batch | Real-Time | Elegida |
|---------|-------|-----------|---------|
| **Latency** | Hours | Milliseconds | Real-Time |
| **Complexity** | Simple | Complex | Real-Time |
| **Cost** | Low | High | Real-Time |
| **Accuracy** | Good | Excellent | Real-Time |
| **Use Case** | Daily reports | Fraud detection | Real-Time |

---

### Kafka vs Alternatives

❌ RabbitMQ:

Lower throughput (< 100k msg/sec)

No durability guarantee

Uso: Traditional queues

❌ AWS Kinesis:

Higher latency (~ 1 sec)

Vendor lock-in

Uso: AWS-only shops

✅ KAFKA:

High throughput (millions msg/sec)

Durability + replayability

Multi-language support

Uso: Data engineering standard

text

---

## Manejo de Errores & Durability

===== CHECKPOINT: Fault Tolerance =====
Si spark job falla, retoma desde último checkpoint
.writeStream
.option("checkpointLocation", "s3://checkpoints/fraud")
.start()

Spark guarda offset = "leyó hasta aquí"
Si falla: restart lee desde último offset → NO pierde datos
===== EXACTLY-ONCE SEMANTICS =====
Garantía: Cada mensaje procesado exactamente 1 vez
Kafka offset management:
1. Read from Kafka
2. Process
3. Write result
4. Commit offset
Si falla en step 3: retry desde step 1 (duplicate prevention)
===== DEAD LETTER QUEUE =====
Mensajes que fallan 3x van a quarantine
def safe_process(msg):
try:
return enrich_and_score(msg)
except Exception as e:
if msg["retry_count"] >= 3:
send_to_deadletter(msg) # quarantine
else:
msg["retry_count"] += 1
return retry(msg)

text

---

## Scaling: 1M → 10M Transacciones/Min

KAFKA:

Brokers: 3 → 10 (más capacidad)

Partitions: 100 → 500 (paralelismo)

Replication: 3 (mantén)

SPARK STREAMING:

Executors: 10 → 50 (más workers)

Batch interval: 500ms → 100ms (más frecuente)

REDIS (ML Cache):

Nodes: 1 → Redis Cluster (3 nodes)

Replication: Enable (HA)

COST:

$13k/mes → $40k/mes (~3x)

text

---

## Monitoring Real-Time

===== METRICS =====
metrics = {
"end_to_end_latency": 150, # ms (target: < 500ms)
"fraud_detection_rate": 0.94, # % (target: > 90%)
"false_positive_rate": 0.02, # % (target: < 5%)
"throughput": "16.7k msg/sec", # (actual vs expected)
"lag": "5 sec", # Kafka consumer lag (target: < 10s)
}

ALERTS:
if end_to_end_latency > 500:
alert("CRITICAL: Fraud detection latency > 500ms")
if false_positive_rate > 0.05:
alert("WARNING: Too many false positives")
if consumer_lag > 30:
alert("WARNING: Processing falling behind")

text

---

## Alternativas Consideradas

### Alternativa 1: Batch Every Hour

❌ Latency: 1 hour (SLA required < 500ms)
❌ Fraud happens NOW, blocked después
✗ No viable

text

### Alternativa 2: Microservices (Direct API)

❌ No durability (si service cae, pierdes mensajes)
❌ Coupling (cada service debe conocer otros)
✗ Complex, error-prone

text

### Alternativa 3: Lambda Architecture (Batch + Real-Time)

✓ Best of both worlds
✗ Complexity + cost
└─ Usa si: necesitas offline re-computation

text

---

## Errores Comunes en Entrevista

- **Error**: "Voy a procesar TODO en memory" → **Solución**: Stream es infinito. Necesitas stateless o mini-batches

- **Error**: No manejar out-of-order events → **Solución**: Events llegan desorden (network). Necesitas watermarks

- **Error**: No pensar en estado (user profile lookup)** → **Solución**: State stores (Redis, RocksDB) son críticos

- **Error**: "Real-time siempre es mejor" → **Solución**: Batch es más simple, barato. Real-time solo si NECESARIO

---

## Preguntas de Seguimiento

1. **"¿Cómo manejas time-series data (eventos tardíos)?"**
   - Watermarks: eventos > 1h tardío son descartados
   - State stores: guardar estado por X tiempo
   - Late data handling: ALLOWED_LATENESS option

2. **"¿Cómo debuggeas streaming bugs?"**
   - Logs (difícil, datos fluyen rápido)
   - Mini-batch debugging (pausar stream)
   - Replayability: Kafka guarda data, puedes replay

3. **"¿Cómo evitas data loss?"**
   - Kafka replication (3 copies)
   - Spark checkpoints (offset tracking)
   - Exactly-once semantics

4. **"¿Cómo backfillas datos si model cambia?"**
   - Re-procesar histórico desde Kafka
   - Spark batch job (lee Kafka desde T0, vuelve a procesar)

---

## References

- [Kafka Architecture - Apache Docs](https://kafka.apache.org/documentation/)
- [Spark Structured Streaming - Databricks](https://docs.databricks.com/en/structured-streaming/index.html)
- [Real-Time Fraud Detection - Netflix](https://netflixtechblog.com/)

