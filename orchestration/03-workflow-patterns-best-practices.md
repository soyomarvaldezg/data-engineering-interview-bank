# Workflow Patterns & Best Practices

**Tags**: #orchestration #patterns #reliability #idempotency #monitoring #real-interview  
**Empresas**: Amazon, Google, Meta, Netflix, Uber  
**Dificultad**: Senior  
**Tiempo estimado**: 22 min  

---

## TL;DR

**Workflow patterns** = proven solutions to common problems (failures, retries, monitoring). **Core patterns**: Idempotency (safe retries), exactly-once semantics (no dupes), SLAs (time guarantees), backfills (historical runs). **Reality**: Most pipeline failures are preventable with good design. **Key**: Design for failure recovery, not just happy path.

---

## Concepto Core

- **Qué es**: Patterns = repeatable designs for reliability + maintainability
- **Por qué importa**: Production pipelines fail. Need graceful handling, quick recovery
- **Principio clave**: "Fail fast, recover quickly" (not "prevent all failures")

---

## Pattern 1: Idempotency (Critical!)

Definition: Same operation run multiple times = same result

Example: LOAD daily orders

❌ NON-IDEMPOTENT (BROKEN):
INSERT INTO warehouse.orders SELECT * FROM raw_orders WHERE date = '2024-01-15';
-- Run 1: Inserts 1000 rows
-- Run 2 (retry): Inserts 1000 rows AGAIN (now 2000 total = WRONG!)

✅ IDEMPOTENT (CORRECT):
DELETE FROM warehouse.orders WHERE date = '2024-01-15';
INSERT INTO warehouse.orders SELECT * FROM raw_orders WHERE date = '2024-01-15';
-- Run 1: Delete 0, insert 1000 → Total 1000
-- Run 2 (retry): Delete 1000, insert 1000 → Total 1000 (CORRECT!)

Alternative idempotent approach:
INSERT INTO warehouse.orders SELECT * FROM raw_orders WHERE date = '2024-01-15'
ON CONFLICT (order_id) DO UPDATE SET ... ; -- Upsert

Real-world requirement:
-- EVERY data pipeline must be idempotent
-- Assumption: Might run multiple times (retries, backfills)
-- So: Must produce same result regardless of run count

text

### Idempotency Patterns

-- Pattern 1: DELETE + INSERT (Full refresh per day)
DELETE FROM dim_customers WHERE load_date = '2024-01-15';
INSERT INTO dim_customers
SELECT customer_id, name, email, '2024-01-15' as load_date
FROM raw.customers;

-- Pattern 2: UPSERT (Insert or update if exists)
MERGE INTO dim_customers t
USING (SELECT customer_id, name, email FROM raw.customers) s
ON t.customer_id = s.customer_id AND t.load_date = '2024-01-15'
WHEN MATCHED THEN UPDATE SET t.name = s.name, t.email = s.email
WHEN NOT MATCHED THEN INSERT VALUES (s.customer_id, s.name, s.email, '2024-01-15');

-- Pattern 3: CREATE OR REPLACE (Views)
CREATE OR REPLACE VIEW fct_sales AS
SELECT order_id, customer_id, amount FROM raw_orders WHERE date >= '2024-01-01';
-- View always reflects latest raw data, idempotent by design

-- Pattern 4: Incremental with checkpoint
CREATE TABLE IF NOT EXISTS fact_events (
event_id INT,
timestamp TIMESTAMP,
data JSON,
load_timestamp TIMESTAMP,
load_date DATE,
UNIQUE(event_id, load_date)
);

INSERT INTO fact_events
SELECT event_id, timestamp, data, NOW(), '2024-01-15'
FROM raw_events
WHERE date = '2024-01-15'
ON CONFLICT (event_id, load_date) DO NOTHING;
-- Idempotent: If already loaded, ON CONFLICT prevents duplicates

text

---

## Pattern 2: SLA & Monitoring

SLA = Service Level Agreement (time guarantee)
from airflow import DAG
from airflow.utils.dates import days_ago
from datetime import timedelta

dag = DAG(
'critical_etl_with_sla',
schedule_interval='0 2 * * *', # 2 AM UTC daily
start_date=days_ago(30),
sla=timedelta(hours=4), # Must finish within 4 hours
sla_miss_callback=notify_sla_miss, # Alert if breached
tags=['production', 'critical']
)

def notify_sla_miss(dag, task_list, blocking_task_list):
"""Called if DAG misses SLA"""
print(f"URGENT: Pipeline SLA MISSED!")
print(f"Tasks involved: {task_list}")
# Send urgent Slack/PagerDuty alert
# Page on-call engineer immediately

Individual task SLAs
t1 = PythonOperator(
task_id='extract_data',
python_callable=extract,
sla=timedelta(minutes=30), # Must finish by 02:30
sla_miss_callback=notify_task_sla_miss,
dag=dag
)

t2 = PythonOperator(
task_id='transform_data',
python_callable=transform,
sla=timedelta(minutes=30), # Must finish by 03:00
dag=dag
)

t3 = PythonOperator(
task_id='load_warehouse',
python_callable=load,
sla=timedelta(minutes=30), # Must finish by 03:30
dag=dag
)

t1 >> t2 >> t3

SLA timeline:
02:00 - Start
02:30 - t1 SLA (extract must finish)
03:00 - t2 SLA (transform must finish)
03:30 - t3 SLA (load must finish)
06:00 - Overall DAG SLA (all done by here)
If any misses: Immediate alert
Monitoring metrics
dag_metrics = {
"duration_minutes": 45, # Actual time
"target_sla_minutes": 240, # 4 hours
"sla_compliance": (240 - 45) / 240 * 100, # 81% buffer
"status": "PASS"
}

text

---

## Pattern 3: Retry Strategies

from airflow.utils.decorators import apply_defaults
from datetime import timedelta

Strategy 1: Linear backoff (wait same time between retries)
default_args_linear = {
'retries': 3,
'retry_delay': timedelta(minutes=5), # Wait 5 min each time
}

Timeline: Fail at 02:00 → Retry 02:05 → Retry 02:10 → Retry 02:15 → Give up 02:15
Strategy 2: Exponential backoff (wait longer each time)
default_args_exponential = {
'retries': 3,
'retry_delay': timedelta(minutes=1),
'retry_exponential_backoff': True, # Double wait each retry
}

Timeline: Fail at 02:00 → Retry 02:01 → Retry 02:03 → Retry 02:07 → Give up 02:07
Better for transient failures (network glitches)
Strategy 3: Custom retry logic
from airflow.exceptions import AirflowException

def custom_retry_logic():
"""Decide whether to retry based on error type"""
try:
result = call_flaky_api()
return result
except ConnectionError as e:
# Transient (network): Retry
raise AirflowException(f"Network error (will retry): {e}")
except ValueError as e:
# Permanent (bad data): Don't retry
raise Exception(f"Bad data (won't retry): {e}")

Strategy 4: Max retries with timeout
t1 = PythonOperator(
task_id='api_call',
python_callable=custom_retry_logic,
retries=5,
max_tries=7, # Overall timeout
execution_timeout=timedelta(minutes=10), # 10 min per attempt max
)

text

---

## Pattern 4: Backfilling (Re-run Historical)

from datetime import datetime

Scenario: New table added to warehouse
Need to backfill 1 year of historical data
dag = DAG(
'backfill_historical_orders',
schedule_interval=None, # Manual trigger (not scheduled)
start_date=datetime(2023, 1, 1), # Start from 1 year ago
end_date=datetime(2024, 1, 1), # End at today
catchup=True, # CRITICAL: Run all missed intervals
)

When you deploy this DAG:
Airflow automatically runs intervals:
2023-01-01, 2023-01-02, ..., 2024-01-01
In parallel or sequentially (depends on parallelism setting)
Each run gets unique date context:
def process_date(**context):
ds = context['ds'] # Execution date (e.g., '2023-06-15')
print(f"Processing data for {ds}")
# Query WHERE date = {ds}

In production: Once backfill done, switch to normal scheduling
dag_prod = DAG(
'orders_daily_etl',
schedule_interval='0 2 * * *', # Run daily at 2 AM
start_date=datetime(2024, 1, 1),
catchup=True, # Catch up if scheduler down
)

Command-line backfill:
airflow dags backfill -s 2023-01-01 -e 2024-01-01 orders_daily_etl
Backfills all days in range
text

---

## Pattern 5: Exactly-Once Semantics (Distributed)

Challenge: Distributed pipelines (multiple workers)
Guarantee: Each record processed EXACTLY ONCE (not 0, not 2+)

Scenario: Kafka consumer → Processing → Database
Risk: Worker crashes mid-process, message processed twice

Solution 1: Idempotent writes (already discussed)
Solution 2: Distributed deduplication

Implementation:

Read from Kafka offset X

Process message

Write to database with unique ID

COMMIT offset (Kafka marks as processed)

If crash before commit: Reprocess from offset X (idempotent!)

Code (Spark Structured Streaming):
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("exactlyOnce").getOrCreate()

df = spark
.readStream
.format("kafka")
.option("kafka.bootstrap.servers", "localhost:9092")
.option("subscribe", "events")
.load()

result = df
.select("value")
.writeStream
.outputMode("append")
.option("checkpointLocation", "/tmp/checkpoint")
.option("idempotencyKey", "event_id") # Deduplicate
.toTable("events_deduplicated")

Spark automatically deduplicates within 1-hour window
Combining with idempotent writes = exactly-once guarantees
text

---

## Pattern 6: Data Freshness & Staleness Alerts

Monitor: Are analytics up-to-date?
def check_data_staleness(**context):
"""Alert if data hasn't updated in >2 hours"""
query = """
SELECT MAX(load_timestamp) as last_update
FROM warehouse.fact_sales
"""
result = execute_query(query)
last_update = result['last_update']
hours_old = (datetime.now() - last_update).total_seconds() / 3600

text
if hours_old > 2:
    # STALE: Alert!
    alert(f"Data is {hours_old} hours old, expected fresh within 2h")
    raise AirflowException("Data freshness SLA breached")
else:
    print(f"Data is fresh ({hours_old:.1f}h old)")
t_freshness_check = PythonOperator(
task_id='check_data_freshness',
python_callable=check_data_staleness,
dag=dag
)

Run at end of pipeline
load_task >> t_freshness_check

text

---

## Pattern 7: Failure Modes & Recovery

Failure Mode 1: Task fails, retries, still fails
├─ What: Permanent error (bad code, invalid data)
├─ Recovery: Alert, manual investigation, fix code, redeploy
└─ Action: PagerDuty → Engineer investigates

Failure Mode 2: Task times out (takes too long)
├─ What: Performance degradation, infinite loop
├─ Recovery: Kill task, scale up resources, optimize code
└─ Action: Investigate why slow, add caching/indexes

Failure Mode 3: Downstream dependency fails
├─ What: Feed data to task, but source doesn't exist
├─ Recovery: Wait + retry, or skip if optional
└─ Action: Add sensors (wait for upstream) or skip_on_fail

Failure Mode 4: Data quality check fails
├─ What: Data out of range, nulls where unexpected
├─ Recovery: Don't load bad data to warehouse, alert
└─ Action: Quarantine data, notify source team

Recovery strategies:

@on_failure_callback
def alert_on_failure(context):
task_instance = context['task_instance']
print(f"Task {task_instance.task_id} FAILED")
# Send Slack message
# Create JIRA ticket
# Page on-call if critical

@on_success_callback
def log_on_success(context):
print(f"Task {context['task_instance'].task_id} succeeded")
# Log metrics
# Increment success counter

text

---

## Pattern 8: Monitoring & Observability

Key metrics to track
metrics = {
"dag_run_duration": 45, # minutes
"dag_sla_target": 240, # minutes
"dag_success_rate": 0.98, # 98% success
"task_retry_count": 2, # needed 2 retries
"data_row_count": 150000,
"data_freshness": 15, # minutes old
"task_durations": {
"extract": 5,
"transform": 30,
"load": 10,
}
}

Alert thresholds
alerts = {
"dag_run_duration_exceeds": 300, # >5 hours = WARN
"task_failure": True, # Any failure = ALERT
"data_freshness_exceeds": 120, # >2 hours old = ALERT
"row_count_delta": 0.5, # Change >50% = INVESTIGATE
}

Logging best practices
logging.info(f"Pipeline start: {datetime.now()}")
logging.info(f"Rows extracted: {row_count}")
logging.info(f"Rows transformed: {transformed_count}")
logging.info(f"Rows loaded: {loaded_count}")
logging.info(f"Pipeline end: {datetime.now()}, duration: {duration_min}m")

If anything unexpected:
logging.warning(f"Unusual pattern: {row_count} >> expected {avg_row_count}")
logging.error(f"CRITICAL: Data quality check failed: {error}")

text

---

## Errores Comunes en Entrevista

- **Error**: "Just retry forever" → **Solución**: Need exponential backoff + max retries (don't retry broken code)

- **Error**: No idempotency → **Solución**: Retries = guaranteed, so MUST be idempotent

- **Error**: Ignoring SLAs → **Solución**: Production = time guarantees (or don't call it "production")

- **Error**: No monitoring → **Solución**: "Works locally" doesn't mean works in production

---

## Preguntas de Seguimiento

1. **"¿Cómo aseguran exactly-once?"**
   - Idempotent writes (main strategy)
   - Distributed deduplication (for Kafka)
   - Combined = guarantees

2. **"¿Backfill strategy?"**
   - Nightly: Small backfill (yesterday's data)
   - Large: Schedule backfill job separately (won't block daily)

3. **"¿Task failure recovery?"**
   - Retries (auto-recover)
   - SLAs (catch issues)
   - Monitoring (alert humans)

4. **"¿How to test reliability?"**
   - Chaos testing (inject failures)
   - Retry simulation (force retries)
   - Load testing (what breaks under pressure?)

---

## References

- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Exactly-Once Semantics - Kafka](https://kafka.apache.org/documentation/#semantics)
- [Distributed Systems Reliability - SRE Book](https://sre.google/books/)

