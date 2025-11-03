# Apache Airflow: DAG Orchestration

**Tags**: #orchestration #airflow #dag #scheduling #error-handling #real-interview  
**Empresas**: Amazon, Google, Meta, Airbnb, Spotify  
**Dificultad**: Mid  
**Tiempo estimado**: 22 min

---

## TL;DR

**Airflow** = workflow orchestration platform. **Core concept**: DAG (Directed Acyclic Graph) = task dependency graph. **Tasks**: PythonOperator, BashOperator, Sensor, etc. **Scheduling**: Cron-like intervals or event-triggered. **Benefits**: Visibility, retry logic, error handling, monitoring. **Trade-off**: Setup complexity vs control gained.

---

## Concepto Core

- **Qué es**: Airflow = scheduler + executor + scheduler + database. Manages task dependencies + scheduling
- **Por qué importa**: Production pipelines need reliability. Airflow = proven, industry-standard
- **Principio clave**: DAG = code-as-infrastructure (pipeline is code, versioned, testable)

---

## Architecture

┌─────────────────────────────────────────────────────────┐
│ AIRFLOW CLUSTER │
├─────────────────────────────────────────────────────────┤
│ │
│ Scheduler │
│ ├─ Parses DAGs every N seconds │
│ ├─ Determines which tasks to run │
│ └─ Submits to executor │
│ │
│ Executor (Local, Celery, Kubernetes) │
│ ├─ Local: Run on same machine (dev only) │
│ ├─ Celery: Distributed (worker pools) │
│ └─ Kubernetes: Container-native (scalable) │
│ │
│ Metadata DB (PostgreSQL) │
│ ├─ DAG definitions │
│ ├─ Task instances (state, logs, retries) │
│ ├─ Variables, connections (encrypted) │
│ └─ XCom (cross-task communication) │
│ │
│ Web UI │
│ ├─ Monitor DAGs, tasks │
│ ├─ Trigger manual runs │
│ └─ View logs, metrics │
│ │
└─────────────────────────────────────────────────────────┘

text

---

## DAG Basics

from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

Define DAG
dag = DAG(
'daily_etl_pipeline', # DAG ID
description='Daily data processing',
schedule_interval='0 2 \* \* \*', # Every day at 2 AM UTC
start_date=datetime(2024, 1, 1),
catchup=True, # Backfill past runs
tags=['data-engineering', 'production']
)

Define tasks
def extract_data(\*\*context):
"""Extract from source"""
print(f"Extracting data for {context['ds']}")

# Code: connect to DB, extract

return 'extracted'

def transform_data(\*\*context):
"""Transform extracted data"""
print("Transforming...")

# Code: pandas/spark transform

def load_data(\*\*context):
"""Load to warehouse"""
print("Loading...")

# Code: COPY to Redshift

Create task instances
t1 = PythonOperator(
task_id='extract',
python_callable=extract_data,
dag=dag
)

t2 = PythonOperator(
task_id='transform',
python_callable=transform_data,
dag=dag
)

t3 = PythonOperator(
task_id='load',
python_callable=load_data,
dag=dag
)

Define dependencies (DAG structure)
t1 >> t2 >> t3

Means: t1 THEN t2 THEN t3 (sequential)
Alternative syntax:
t1.set_downstream(t2)
t2.set_downstream(t3)
text

---

## Task Dependencies & Patterns

from airflow.operators.empty import EmptyOperator

Pattern 1: Linear (sequential)
t1 >> t2 >> t3 >> t4

t1 finishes, then t2 starts, then t3, then t4
Pattern 2: Fan-out (parallel after t1)
t1 >> [t2, t3, t4]

t1 finishes, then t2, t3, t4 run in parallel
Pattern 3: Fan-in (merge parallel into one)
[t2, t3, t4] >> t5

All t2, t3, t4 must finish before t5 starts
Pattern 4: Complex DAG
t2 ──┐
/ ├─> t5
t1 ─────┤ ├─> t7
\ ├─> t6
t3 ──┘
t4 ────────> t8
t1 >> [t2, t3]
t2 >> t5
t3 >> t5
t5 >> [t6, t7]
t4 >> t8

Visualize in Airflow UI (see DAG structure)
text

---

## Operators & Sensors

from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.dummy import DummyOperator
from airflow.sensors.filesystem import FileSensor
from airflow.sensors.poke import PythonSensor
from airflow.operators.python import BranchPythonOperator

PythonOperator: Run Python function
t1 = PythonOperator(
task_id='my_python_task',
python_callable=my_function
)

BashOperator: Run bash command
t2 = BashOperator(
task_id='my_bash_task',
bash_command='python /path/to/script.py'
)

FileSensor: Wait for file to exist
t3 = FileSensor(
task_id='wait_for_file',
filepath='/tmp/data.csv',
poke_interval=60, # Check every 60 sec
timeout=3600 # Give up after 1 hour
)

PythonSensor: Custom sensor (polling)
t4 = PythonSensor(
task_id='wait_for_condition',
python_callable=check_if_ready,
poke_interval=30
)

BranchPythonOperator: Conditional branching
def decide_branch(\*\*context):
if some_condition:
return 'task_a'
else:
return 'task_b'

t5 = BranchPythonOperator(
task_id='branch_decision',
python_callable=decide_branch
)

Branching DAG
t5 >> [t_a, t_b]

Only one branch executes based on return value
text

---

## Error Handling & Retries

from airflow.utils.decorators import apply_defaults
from airflow.exceptions import AirflowException

dag = DAG(
'etl_with_retries',
schedule_interval='0 2 \* \* \*',
start_date=datetime(2024, 1, 1),

text

# Default retry policy for all tasks

default_args={
'retries': 3, # Retry 3 times on failure
'retry_delay': timedelta(minutes=5), # Wait 5 min between retries
'retry_exponential_backoff': True, # Double wait time each retry
'on_failure_callback': send_alert_on_failure, # Alert on fail
'on_success_callback': send_alert_on_success, # Alert on success
}
)

def risky_task(\*\*context):
try:

# Potentially failing code

result = call_api()
if not result:
raise AirflowException("API returned empty")
return result
except Exception as e:

# Log, but let Airflow retry

print(f"Error: {e}, will retry")
raise

t1 = PythonOperator(
task_id='risky_operation',
python_callable=risky_task,
retries=5, # Override default
retry_delay=timedelta(minutes=2),
dag=dag
)

Retry timeline:
Attempt 1: Fails at 02:00
Attempt 2: Retries at 02:02 (2 min delay)
Attempt 3: Retries at 02:06 (4 min delay, exponential)
Attempt 4: Retries at 02:14 (8 min delay)
Attempt 5: Retries at 02:30 (16 min delay)
If still fails: Mark as FAILED, trigger alert
text

---

## XCom (Cross-Task Communication)

def task_1(\*\*context):
"""Push data to XCom"""
result = "important_data_12345"
context['task_instance'].xcom_push(key='my_key', value=result)

# Stores in metadata DB

def task_2(\*\*context):
"""Pull data from XCom"""
task_instance = context['task_instance']
pushed_value = task_instance.xcom_pull(
task_ids='task_1', # Which task pushed it
key='my_key'
)
print(f"Received: {pushed_value}") # "important_data_12345"

t1 = PythonOperator(
task_id='task_1',
python_callable=task_1,
dag=dag
)

t2 = PythonOperator(
task_id='task_2',
python_callable=task_2,
dag=dag
)

t1 >> t2

Use case: Pass file paths, data counts, status flags between tasks
text

---

## Real-World: E-commerce Daily ETL

from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.sensors.filesystem import FileSensor
from datetime import datetime, timedelta

default_args = {
'owner': 'data-engineering',
'retries': 2,
'retry_delay': timedelta(minutes=5),
}

dag = DAG(
'ecommerce_daily_etl',
description='Daily order processing',
schedule_interval='0 1 \* \* \*', # 1 AM UTC
start_date=datetime(2024, 1, 1),
default_args=default_args,
tags=['production', 'ecommerce']
)

Task 1: Wait for source data file
sensor = FileSensor(
task*id='wait_for_orders_file',
filepath='/data/incoming/orders*{{ ds }}.csv',
timeout=3600,
poke_interval=60,
dag=dag
)

Task 2: Extract from source
def extract_orders(\*\*context):
ds = context['ds']
print(f"Extracting orders for {ds}")

# Read CSV, validate, upload to S3

row_count = 150000
context['task_instance'].xcom_push(key='row_count', value=row_count)

extract = PythonOperator(
task_id='extract_orders',
python_callable=extract_orders,
dag=dag
)

Task 3: Transform (Spark job)
transform = BashOperator(
task_id='transform_orders',
bash_command='spark-submit /path/to/etl.py {{ ds }}',
dag=dag
)

Task 4: Load to warehouse
def load_to_warehouse(\*\*context):
print("Loading to Redshift...")
row_count = context['task_instance'].xcom_pull(
task_ids='extract_orders',
key='row_count'
)
print(f"Loaded {row_count} rows")

# COPY command to Redshift

load = PythonOperator(
task_id='load_redshift',
python_callable=load_to_warehouse,
dag=dag
)

Task 5: Validation
def validate_data(\*\*context):
print("Validating...")

# Row count checks, data quality

# If fails: raise AirflowException

validate = PythonOperator(
task_id='validate_load',
python_callable=validate_data,
dag=dag
)

Task 6: Notify success
def notify_success(\*\*context):
print("Pipeline success! Notifying analytics team...")

# Send Slack message, email

notify = PythonOperator(
task_id='notify_slack',
python_callable=notify_success,
dag=dag
)

Define DAG
sensor >> extract >> transform >> load >> validate >> notify

If validate fails, retry entire from load (via default retry)
If all succeed, notify sent at end
text

---

## SLAs & Alerting

from datetime import timedelta
from airflow.exceptions import AirflowException

default_args = {
'owner': 'data-engineering',
'retries': 2,
'retry_delay': timedelta(minutes=5),
'sla': timedelta(hours=4), # DAG must complete in 4 hours
'sla_miss_callback': alert_on_sla_miss, # Alert if breached
}

def alert_on_sla_miss(dag, task_list, blocking_task_list):
"""Called if SLA is breached"""
print(f"SLA MISS! Tasks: {task_list}")

# Send urgent alert to on-call engineer

dag = DAG(
'critical_pipeline',
schedule_interval='0 1 \* \* \*',
start_date=datetime(2024, 1, 1),
default_args=default_args,
sla_miss_callback=alert_on_sla_miss,
)

Individual task SLA
t1 = PythonOperator(
task_id='critical_extract',
python_callable=extract,
sla=timedelta(minutes=30), # Must finish in 30 min
dag=dag
)

SLA timeline:
01:00 - DAG starts
01:30 - Task SLA for t1 (must finish by 01:30)
05:00 - DAG SLA (must finish by 05:00, else alert)
text

---

## Monitoring & Observability

Airflow Web UI:
├─ DAG view: Dependency graph
├─ Tree view: Historical runs (green=success, red=failed)
├─ Gantt chart: Task duration + timeline
├─ Task logs: Real-time output
└─ XCom: Data passed between tasks

Programmatic monitoring:
├─ Email on failure (on_failure_callback)
├─ Slack notifications (SlackOperator)
├─ Metrics to CloudWatch (custom code)
└─ Logs to centralized logging (ELK, Splunk)

Metrics to track:
├─ Task duration (is it slower lately?)
├─ Success rate (% failures)
├─ SLA compliance (% on-time)
└─ Retry frequency (is code unstable?)

text

---

## Errores Comunes en Entrevista

- **Error**: "Just use cron jobs" → **Solución**: Cron = no error handling, no visibility

- **Error**: Too many dependencies (spaghetti DAG)** → **Solución\*\*: Break into smaller DAGs

- **Error**: No SLA/alerting → **Solución**: Data engineering = production, need monitoring

- **Error**: Hardcoding dates (not using context['ds'])** → **Solución\*\*: Use Airflow variables for backfill support

---

## Preguntas de Seguimiento

1. **"¿Airflow vs cron vs managed services?"**
   - Cron: Simple, no visibility
   - Airflow: Full control, complexity
   - Managed (AWS Step Functions): Ease vs control trade-off

2. **"¿Parallel tasks?"**
   - Use fan-out: t1 >> [t2, t3, t4]
   - Executor parallelism controls how many run concurrently

3. **"¿Backfilling?"**
   - Start date = 2024-01-01, run date = 2024-03-15
   - Airflow will automatically run all missed intervals
   - Set catchup=True in DAG

4. **"¿Airflow for real-time?"**
   - No (designed for scheduled batches)
   - For real-time: Use Kafka/Spark Streaming/Flink

---

## References

- [Apache Airflow - Official Docs](https://airflow.apache.org/docs/)
- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [DAG Structure - Airflow Guide](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html)
