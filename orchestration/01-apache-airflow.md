# Apache Airflow: Orquestación de DAG

**Tags**: #orchestration #airflow #dag #scheduling #error-handling #real-interview

---

## TL;DR

**Airflow** = plataforma de orquestación de workflows. **Concepto clave**: DAG (Directed Acyclic Graph) = grafo de dependencias de tasks. **Tasks**: PythonOperator, BashOperator, Sensor, etc. **Scheduling**: intervalos tipo Cron o disparados por eventos. **Beneficios**: Visibilidad, lógica de reintentos, manejo de errores, monitoreo. **Compensación**: Complejidad de configuración vs control obtenido.

---

## Concepto Core

- **Qué es**: Airflow = scheduler + executor + scheduler + database. Gestiona las dependencias de tasks + scheduling
- **Por qué importa**: Los pipelines de producción necesitan confiabilidad. Airflow = probado, estándar de la industria
- **Principio clave**: DAG = code-as-infrastructure (el pipeline es código, versionado, testeable)

---

## Architecture

┌─────────────────────────────────────────────────────────┐
│ AIRFLOW CLUSTER │
├─────────────────────────────────────────────────────────┤
│ │
│ Scheduler │
│ ├─ Analiza DAGs cada N segundos │
│ ├─ Determina qué tasks ejecutar │
│ └─ Envía al executor │
│ │
│ Executor (Local, Celery, Kubernetes) │
│ ├─ Local: Se ejecuta en la misma máquina (solo para dev) │
│ ├─ Celery: Distribuido (worker pools) │
│ └─ Kubernetes: Nativo de contenedores (escalable) │
│ │
│ Metadata DB (PostgreSQL) │
│ ├─ Definiciones de DAG │
│ ├─ Task instances (estado, logs, reintentos) │
│ ├─ Variables, connections (encriptadas) │
│ └─ XCom (cross-task communication) │
│ │
│ Web UI │
│ ├─ Monitorea DAGs, tasks │
│ ├─ Dispara ejecuciones manuales │
│ └─ Ver logs, metrics │
│ │
└─────────────────────────────────────────────────────────┘

---

## DAG Basics

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

# Definir DAG
dag = DAG(
    'daily_etl_pipeline', # DAG ID
    description='Daily data processing',
    schedule_interval='0 2 * * *', # Every day at 2 AM UTC
    start_date=datetime(2024, 1, 1),
    catchup=True, # Backfill past runs
    tags=['data-engineering', 'production']
)

# Definir tasks
def extract_data(**context):
    """Extract from source"""
    print(f"Extracting data for {context['ds']}")
    # Code: connect to DB, extract
    return 'extracted'

def transform_data(**context):
    """Transform extracted data"""
    print("Transforming...")
    # Code: pandas/spark transform

def load_data(**context):
    """Load to warehouse"""
    print("Loading...")
    # Code: COPY to Redshift

# Crear task instances
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

# Definir dependencias (estructura del DAG)
t1 >> t2 >> t3

# Significa: t1 ENTONCES t2 ENTONCES t3 (secuencial)
# Sintaxis alternativa:
# t1.set_downstream(t2)
# t2.set_downstream(t3)
```

---

## Task Dependencies & Patterns

```python
from airflow.operators.empty import EmptyOperator

# Patrón 1: Lineal (secuencial)
# t1 >> t2 >> t3 >> t4
# t1 termina, luego t2 comienza, luego t3, luego t4

# Patrón 2: Fan-out (paralelo después de t1)
# t1 >> [t2, t3, t4]
# t1 termina, luego t2, t3, t4 se ejecutan en paralelo

# Patrón 3: Fan-in (fusionar paralelos en uno)
# [t2, t3, t4] >> t5
# Todos los t2, t3, t4 deben terminar antes de que t5 comience

# Patrón 4: DAG complejo
# t2 ──┐
# / ├─> t5
# t1 ─────┤ ├─> t7
# \ ├─> t6
# t3 ──┘
# t4 ────────> t8
# t1 >> [t2, t3]
# t2 >> t5
# t3 >> t5
# t5 >> [t6, t7]
# t4 >> t8

# Visualizar en la Airflow UI (ver estructura del DAG)
```

---

## Operators & Sensors

```python
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.dummy import DummyOperator
from airflow.sensors.filesystem import FileSensor
from airflow.sensors.poke import PythonSensor
from airflow.operators.python import BranchPythonOperator

PythonOperator: Ejecuta función Python
t1 = PythonOperator(
    task_id='my_python_task',
    python_callable=my_function
)

BashOperator: Ejecuta comando bash
t2 = BashOperator(
    task_id='my_bash_task',
    bash_command='python /path/to/script.py'
)

FileSensor: Espera a que un archivo exista
t3 = FileSensor(
    task_id='wait_for_file',
    filepath='/tmp/data.csv',
    poke_interval=60, # Check every 60 sec
    timeout=3600 # Give up after 1 hour
)

PythonSensor: Sensor personalizado (polling)
t4 = PythonSensor(
    task_id='wait_for_condition',
    python_callable=check_if_ready,
    poke_interval=30
)

BranchPythonOperator: Ramificación condicional
def decide_branch(**context):
    if some_condition:
        return 'task_a'
    else:
        return 'task_b'

t5 = BranchPythonOperator(
    task_id='branch_decision',
    python_callable=decide_branch
)

DAG de ramificación
t5 >> [t_a, t_b]
Solo una rama se ejecuta según el valor de retorno
```

---

## Error Handling & Retries

```python
from airflow.utils.decorators import apply_defaults
from airflow.exceptions import AirflowException

dag = DAG(
    'etl_with_retries',
    schedule_interval='0 2 * * *',
    start_date=datetime(2024, 1, 1),
```

```python
    # Política de reintentos predeterminada para todos los tasks
    default_args={
        'retries': 3, # Retry 3 times on failure
        'retry_delay': timedelta(minutes=5), # Wait 5 min between retries
        'retry_exponential_backoff': True, # Double wait time each retry
        'on_failure_callback': send_alert_on_failure, # Alert on fail
        'on_success_callback': send_alert_on_success, # Alert on success
    }
)

def risky_task(**context):
    try:
        # Código potencialmente fallido
        result = call_api()
        if not result:
            raise AirflowException("API returned empty")
        return result
    except Exception as e:
        # Registrar, pero dejar que Airflow reintente
        print(f"Error: {e}, will retry")
        raise

t1 = PythonOperator(
    task_id='risky_operation',
    python_callable=risky_task,
    retries=5, # Override default
    retry_delay=timedelta(minutes=2),
    dag=dag
)

# Cronograma de reintentos:
# Intento 1: Falla a las 02:00
# Intento 2: Reintenta a las 02:02 (retraso de 2 min)
# Intento 3: Reintenta a las 02:06 (retraso de 4 min, exponencial)
# Intento 4: Reintenta a las 02:14 (retraso de 8 min)
# Intento 5: Reintenta a las 02:30 (retraso de 16 min)
# Si aún falla: Marcar como FAILED, disparar alerta
```

---

## XCom (Cross-Task Communication)

```python
def task_1(**context):
    """Push data to XCom"""
    result = "important_data_12345"
    context['task_instance'].xcom_push(key='my_key', value=result)
    # Almacena en la Metadata DB

def task_2(**context):
    """Pull data from XCom"""
    task_instance = context['task_instance']
    pushed_value = task_instance.xcom_pull(
        task_ids='task_1', # Qué task lo empujó
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

# Caso de uso: Pasar rutas de archivo, conteos de datos, flags de estado entre tasks
```

---

## Real-World: E-commerce Daily ETL

```python
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
    schedule_interval='0 1 * * *', # 1 AM UTC
    start_date=datetime(2024, 1, 1),
    default_args=default_args,
    tags=['production', 'ecommerce']
)

# Task 1: Esperar por el archivo de datos fuente
sensor = FileSensor(
    task_id='wait_for_orders_file',
    filepath='/data/incoming/orders_{{ ds }}.csv',
    timeout=3600,
    poke_interval=60,
    dag=dag
)

# Task 2: Extraer de la fuente
def extract_orders(**context):
    ds = context['ds']
    print(f"Extracting orders for {ds}")
    # Leer CSV, validar, subir a S3
    row_count = 150000
    context['task_instance'].xcom_push(key='row_count', value=row_count)

extract = PythonOperator(
    task_id='extract_orders',
    python_callable=extract_orders,
    dag=dag
)

# Task 3: Transformar (Spark job)
transform = BashOperator(
    task_id='transform_orders',
    bash_command='spark-submit /path/to/etl.py {{ ds }}',
    dag=dag
)

# Task 4: Cargar al warehouse
def load_to_warehouse(**context):
    print("Loading to Redshift...")
    row_count = context['task_instance'].xcom_pull(
        task_ids='extract_orders',
        key='row_count'
    )
    print(f"Loaded {row_count} rows")
    # Comando COPY a Redshift

load = PythonOperator(
    task_id='load_redshift',
    python_callable=load_to_warehouse,
    dag=dag
)

# Task 5: Validación
def validate_data(**context):
    print("Validating...")
    # Chequeos de conteo de filas, calidad de datos
    # Si falla: lanzar AirflowException

validate = PythonOperator(
    task_id='validate_load',
    python_callable=validate_data,
    dag=dag
)

# Task 6: Notificar éxito
def notify_success(**context):
    print("Pipeline success! Notifying analytics team...")
    # Enviar mensaje de Slack, correo electrónico

notify = PythonOperator(
    task_id='notify_slack',
    python_callable=notify_success,
    dag=dag
)

# Definir DAG
sensor >> extract >> transform >> load >> validate >> notify

# Si la validación falla, reintentar todo desde la carga (mediante el retry predeterminado)
# Si todo tiene éxito, la notificación se envía al final
```

---

## SLAs & Alerting

```python
from datetime import timedelta
from airflow.exceptions import AirflowException

default_args = {
    'owner': 'data-engineering',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'sla': timedelta(hours=4), # El DAG debe completarse en 4 horas
    'sla_miss_callback': alert_on_sla_miss, # Alertar si se incumple
}

def alert_on_sla_miss(dag, task_list, blocking_task_list):
    """Llamado si se incumple el SLA"""
    print(f"SLA MISS! Tasks: {task_list}")
    # Enviar alerta urgente al ingeniero de guardia

dag = DAG(
    'critical_pipeline',
    schedule_interval='0 1 * * *',
    start_date=datetime(2024, 1, 1),
    default_args=default_args,
    sla_miss_callback=alert_on_sla_miss,
)

# SLA de task individual
t1 = PythonOperator(
    task_id='critical_extract',
    python_callable=extract,
    sla=timedelta(minutes=30), # Debe terminar en 30 min
    dag=dag
)

# Cronograma de SLA:
# 01:00 - El DAG comienza
# 01:30 - SLA de Task para t1 (debe terminar a las 01:30)
# 05:00 - SLA del DAG (debe terminar a las 05:00, de lo contrario, alerta)
```

---

## Monitoring & Observability

Airflow Web UI:
├─ Vista de DAG: Grafo de dependencias
├─ Vista de árbol: Ejecuciones históricas (verde=éxito, rojo=fallido)
├─ Diagrama de Gantt: Duración de tasks + cronograma
├─ Logs de tasks: Salida en tiempo real
└─ XCom: Datos pasados entre tasks

Monitoreo programático:
├─ Correo electrónico en caso de falla (on_failure_callback)
├─ Notificaciones de Slack (SlackOperator)
├─ Metrics a CloudWatch (código personalizado)
└─ Logs a logging centralizado (ELK, Splunk)

Metrics a rastrear:
├─ Duración de tasks (¿está más lento últimamente?)
├─ Tasa de éxito (% de fallas)
├─ Cumplimiento de SLA (% a tiempo)
└─ Frecuencia de reintentos (¿el código es inestable?)

---

## Errores Comunes en Entrevista

- **Error**: "Simplemente usa cron jobs" → **Solución**: Cron = sin manejo de errores, sin visibilidad

- **Error**: Demasiadas dependencias (spaghetti DAG) → **Solución**: Dividir en DAGs más pequeños

- **Error**: No hay SLA/alertas → **Solución**: Data engineering = producción, se necesita monitoreo

- **Error**: Hardcoding de fechas (no usar context['ds']) → **Solución**: Usar variables de Airflow para el soporte de backfill

---

## Preguntas de Seguimiento

1. **"¿Airflow vs cron vs managed services?"**
   - Cron: Simple, sin visibilidad
   - Airflow: Control total, complejidad
   - Managed (AWS Step Functions): Compensación entre facilidad y control

2. **"¿Parallel tasks?"**
   - Usar fan-out: t1 >> [t2, t3, t4]
   - El parallelism del Executor controla cuántos se ejecutan concurrentemente

3. **"¿Backfilling?"**
   - Start date = 2024-01-01, run date = 2024-03-15
   - Airflow ejecutará automáticamente todos los intervalos perdidos
   - Set catchup=True en DAG

4. **"¿Airflow for real-time?"**
   - No (diseñado para batches programados)
   - Para real-time: Usar Kafka/Spark Streaming/Flink

---

## References

- [Apache Airflow - Official Docs](https://airflow.apache.org/docs/)
- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [DAG Structure - Airflow Guide](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html)
