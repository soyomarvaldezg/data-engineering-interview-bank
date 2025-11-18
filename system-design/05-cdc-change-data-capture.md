# CDC: Change Data Capture Patterns

**Tags**: #system-design #cdc #change-tracking #incremental-load #real-interview

---

## TL;DR

**CDC** = capturar cambios (INSERT, UPDATE, DELETE) en fuente y replicar a destino. Problema: cómo saber qué cambió sin re-leer TODO? Soluciones: (1) Query-based (WHERE timestamp > last_run), (2) Log-based (binlog, WAL), (3) Polling-based (cambios detectados por app). Trade-off: Accuracy vs Complexity vs Cost.

---

## Problema Real

**Escenario:**

- Banco con 10M customer records en PostgreSQL
- OLTP database → Data Warehouse (Redshift)
- Replicación: Full load 1x día = 30 min (caro)
- Problema: Si customer actualiza perfil en 3pm, analytics ve cambio en 11pm (8h delay)
- Necesitan: Cambios reflejados en < 1 hora

**Preguntas:**

1. ¿Cómo detectas cambios sin full scan?
2. ¿Query-based vs Log-based vs Polling?
3. ¿Cómo mantienes PK consistency?
4. ¿Qué pasa si UPDATE falla a mitad?
5. ¿Cómo escalas a 100M records?

---

## Solución 1: Query-Based CDC (Simple)

### Concepto

Idea: WHERE updated_at >= last_run_time

Setup:
├─ Source table tiene "updated_at" column (TIMESTAMP)
├─ Cada noche: SELECT WHERE updated_at >= yesterday_11pm
├─ Load cambios a warehouse
└─ Update "last_run_time"

### Implementación

-- Source: PostgreSQL (OLTP)
CREATE TABLE customers (
customer_id BIGINT PRIMARY KEY,
name VARCHAR(100),
email VARCHAR(100),
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Trigger: Actualiza updated_at en cambios
CREATE TRIGGER update_timestamp
BEFORE UPDATE ON customers
FOR EACH ROW
SET NEW.updated_at = CURRENT_TIMESTAMP;

-- ===== INCREMENTAL LOAD =====
-- Warehouse query: Solo cambios desde ayer
SELECT \*
FROM customers
WHERE updated_at >= '2024-01-14 23:00:00' -- last_run_time
ORDER BY customer_id;

-- Load to Redshift:
-- UPSERT (si existe, update; si no, insert)
BEGIN TRANSACTION;
MERGE INTO redshift.customers t
USING (SELECT _ FROM staging) s
ON t.customer_id = s.customer_id
WHEN MATCHED THEN UPDATE SET t.email = s.email, t.updated_at = s.updated_at
WHEN NOT MATCHED THEN INSERT VALUES (s._);
COMMIT;

### Ventajas & Desventajas

✓ VENTAJAS:

Simple (sin overhead de aplicación)

Basado en timestamp = fácil de trackear

No requiere infraestructura especial

✗ DESVENTAJAS:

Requiere updated_at column (qué pasa si falta?)

Miss changes si batch fails mid-process

Requires manual re-query si discrepancia

No captura deletes (UPDATE sí, DELETE no)

Timezone issues si multi-region

---

## Solución 2: Log-Based CDC (Robusto)

### Concepto

Idea: Leer database transaction log (binlog, WAL)

Database internamente:
├─ Cada INSERT/UPDATE/DELETE va al log
├─ Log es FUENTE DE VERDAD
├─ No necesitas timestamp
└─ Captura TODO (INSERT, UPDATE, DELETE)

Extracto:
├─ Herramientas leen log en tiempo real
├─ Parsean cambios
└─ Stream a Kafka/S3

### Arquitectura

PostgreSQL (OLTP)
├─ Write-Ahead Log (WAL)
└─ Every transaction → log
│
▼
┌──────────────────────┐
│ Logical Decoder │
│ (pg_logical_slot) │
└──────────────────────┘
│
▼
┌──────────────────────┐
│ Debezium │
│ (reads WAL) │
└──────────────────────┘
│
▼
┌──────────────────────┐
│ Kafka Topic │
│ (cdc-customers) │
└──────────────────────┘
│
▼
┌──────────────────────┐
│ Spark Streaming │
│ (consume, aggregate) │
└──────────────────────┘
│
▼
┌──────────────────────┐
│ Redshift │
│ (data warehouse) │
└──────────────────────┘

### Setup: Debezium + Kafka

debezium-config.yaml
name: postgres-cdc-connector
config:
connector.class: io.debezium.connector.postgresql.PostgresConnector

Source database
database.hostname: postgres.prod.aws
database.port: 5432
database.user: replication_user
database.password: secret
database.dbname: production

Logical decoding
plugin.name: pgoutput
slot.name: debezium_slot

Capture mode
table.include.list: public.customers,public.orders,public.payments

Output
topic.prefix: cdc\_
transforms: unwrap
transforms.unwrap.type: io.debezium.transforms.ExtractNewRecordState

**Resultado en Kafka:**

// INSERT
{
"op": "c", // create/insert
"after": {"customer_id": 1001, "name": "Alice", "email": "alice@example.com"},
"ts_ms": 1705276800000
}

// UPDATE
{
"op": "u", // update
"before": {"customer_id": 1001, "email": "alice@old.com"},
"after": {"customer_id": 1001, "email": "alice@new.com"},
"ts_ms": 1705276850000
}

// DELETE
{
"op": "d", // delete
"before": {"customer_id": 1001, "name": "Alice"},
"ts_ms": 1705276900000
}

### Consumo en Spark Streaming

from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json, get_json_object

spark = SparkSession.builder.appName("CDC Consumer").getOrCreate()

Read CDC from Kafka
cdc_stream = spark
.readStream
.format("kafka")
.option("kafka.bootstrap.servers", "kafka:9092")
.option("subscribe", "cdc_customers")
.load()

Parse CDC events
cdc_parsed = cdc_stream.select(
get_json_object(col("value").cast("string"), "$.op").alias("operation"),
get_json_object(col("value").cast("string"), "$.after").alias("after_data"),
get_json_object(col("value").cast("string"), "$.ts_ms").alias("timestamp")
)

Apply changes to warehouse
def apply_cdc_to_warehouse(batch_df, batch_id):
for row in batch_df.collect():
op = row["operation"]
data = row["after_data"]

if op == "c": # INSERT
insert_to_redshift(data)
elif op == "u": # UPDATE
upsert_to_redshift(data)
elif op == "d": # DELETE
delete_from_redshift(data)
cdc_parsed.writeStream
.foreachBatch(apply_cdc_to_warehouse)
.option("checkpointLocation", "s3://checkpoints/cdc_customers")
.start()

### Ventajas & Desventajas

✓ VENTAJAS:

Captura TODO (INSERT, UPDATE, DELETE)

No requiere cambios a source app

Real-time (sin batch delay)

Accurate (fuente de verdad = log)

✗ DESVENTAJAS:

Infraestructura compleja (Debezium, Kafka)

Latency inicialmente (setup tiempo)

Overhead en database (logging)

Cost (Kafka cluster, Spark)

---

## Solución 3: Polling-Based CDC (App-Level)

### Concepto

Idea: Aplicación mantiene "change_log" tabla

Cada cambio:
├─ UPDATE customers SET email = ...
├─ INSERT INTO change_log (table_name, record_id, operation, timestamp)
└─ 2 transacciones (atomicity critical)

### Implementación

-- Change log table
CREATE TABLE change_log (
change_id BIGINT PRIMARY KEY AUTO_INCREMENT,
table_name VARCHAR(50),
record_id BIGINT,
operation VARCHAR(10), -- INSERT, UPDATE, DELETE
changed_columns JSON,
changed_at TIMESTAMP,
INDEX idx_table_time (table_name, changed_at)
);

-- ===== ON INSERT =====
BEGIN TRANSACTION;
INSERT INTO customers (name, email) VALUES ('Bob', 'bob@example.com');
INSERT INTO change_log (table_name, record_id, operation, changed_at)
VALUES ('customers', LAST_INSERT_ID(), 'INSERT', NOW());
COMMIT;

-- ===== EXTRACT CHANGES =====
SELECT \*
FROM change_log
WHERE changed_at >= '2024-01-14 23:00:00'
ORDER BY change_id;

-- ===== CLEANUP =====
DELETE FROM change_log WHERE changed_at < DATE_SUB(NOW(), INTERVAL 30 DAY);

### Ventajas & Desventajas

✓ VENTAJAS:

Simple (sin extra infrastructure)

Controlable (tu app lo maneja)

Queryable (simple SQL)

✗ DESVENTAJAS:

Doble write (customers + change_log)

App complexity (quien mantiene change_log?)

Inconsistency risk (si insert ok pero change_log falla)

Overhead (extra inserts)

---

## Comparación: Query vs Log vs Polling

| Criterio           | Query-Based        | Log-Based       | Polling-Based   |
| ------------------ | ------------------ | --------------- | --------------- |
| **Latency**        | Batch (hours)      | Real-time       | Batch (minutes) |
| **Accuracy**       | Good               | Excellent       | Good            |
| **DELETE support** | Requires tombstone | Native          | Native          |
| **Infrastructure** | None               | Kafka, Debezium | None            |
| **Cost**           | Low                | High            | Medium          |
| **Complexity**     | Low                | High            | Medium          |
| **Consistency**    | Eventually         | Strongly        | Eventually      |
| **Recomendación**  | Small tables       | Large, critical | Medium          |

---

## Handling Late/Out-of-Order Events

Problema: CDC events pueden llegar desordenados
Ej: UPDATE event llega ANTES que INSERT event
class CDCProcessor:

def process_cdc_event(self, event):
"""Maneja out-of-order, duplicates"""

    op = event["op"]
    data = event["data"]
    ts = event["ts"]

    # 1. Deduplication (mismo event, múltiples veces)
    if self.is_duplicate(event):
        return  # Skip

    # 2. Out-of-order handling
    if op == "u" and not self.record_exists(data["id"]):
        # UPDATE llega pero no existe registro
        # Opción A: Buffer y esperar INSERT (risky)
        # Opción B: Treat como UPSERT (mejor)
        self.upsert(data)

    # 3. Delete tracking (soft deletes)
    if op == "d":
        self.soft_delete(data["id"], ts)

    # 4. Latest-write-wins (si multiple events same record)
    existing_ts = self.get_record_timestamp(data["id"])
    if ts > existing_ts:
        self.apply_change(op, data)

---

## Real-World: Bank Customer Changes

Escenario: 10M customers, cambios al perfil (address, email, phone)
===== Option 1: Query-Based (Simple, pero 1h delay) =====
Cada hora: SELECT \* FROM customers WHERE updated_at >= 1h_ago
===== Option 2: Log-Based (Real-time, Kafka) =====
Debezium reads PostgreSQL WAL
Kafka topics: customer-inserts, customer-updates, customer-deletes
Spark Streaming consumes, applies to Redshift
Latency: < 5 segundos
===== Option 3: Hybrid (Best of both) =====

- Log-based para transaccional (< 5s)
- Query-based como fallback (nightly reconciliation)
- Si discrepancia: alert y investigate
  ===== IMPLEMENTATION =====
  pipeline = CDCPipeline(
  source_db="postgres",
  method="log-based", # Debezium
  target_warehouse="redshift",
  consistency_check="nightly" # Reconcile cada noche
  )

pipeline.start()

---

## Errores Comunes en Entrevista

- **Error**: "Query-based CDC es siempre suficiente" → **Solución**: Para grandes tables, log-based es más efficient

- **Error**: No manejar out-of-order events → **Solución**: UPSERT siempre, no INSERT

- **Error**: No tracking DELETEs → **Solución**: Log-based captura, query-based NO (requiere soft deletes)

- **Error**: Ignoring consistency (replication lag) → **Solución**: Nightly reconciliation contra source

---

## Preguntas de Seguimiento

1. **"¿Cuándo CDC vs Full Load?"**
   - Full: < 1M records, simples
   - CDC: > 1M records, time-sensitive

2. **"¿Cómo manejas schema changes?"**
   - Log-based: Debezium handles automáticamente
   - Query-based: Requiere manual intervention

3. **"¿Cost de CDC vs Full Load?"**
   - Full: 30 min batch, barato pero infrequent
   - Log-based: Continuous, más expensive pero real-time

4. **"¿Cómo validates CDC correctness?"**
   - Reconciliation queries (count, hash)
   - Hourly vs source comparison

---

## References

- [Debezium CDC - Apache](https://debezium.io/)
- [PostgreSQL WAL - Documentation](https://www.postgresql.org/docs/current/wal.html)
