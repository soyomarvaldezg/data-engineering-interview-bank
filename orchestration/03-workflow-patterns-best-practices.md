# Patrones de Workflow y Mejores Prácticas

**Tags**: #orchestration #patterns #reliability #idempotency #monitoring #real-interview

---

## TL;DR

**Workflow patterns** = soluciones probadas a problemas comunes (fallas, `retries`, `monitoring`). **Patrones clave**: Idempotency (`retries` seguros), `exactly-once semantics` (sin duplicados), `SLAs` (garantías de tiempo), `backfills` (ejecuciones históricas). **Realidad**: La mayoría de las fallas de `pipeline` se pueden prevenir con un buen diseño. **Clave**: Diseñar para la recuperación de fallas, no solo para el `happy path`.

---

## Concepto Core

- **Qué es**: `Patterns` = diseños repetibles para `reliability` + `maintainability`
- **Por qué importa**: Los `production pipelines` fallan. Se necesita un manejo elegante, recuperación rápida
- **Principio clave**: "Fail fast, recover quickly" (no "prevenir todas las fallas")

---

## Pattern 1: Idempotency (¡Crítico!)

Definición: La misma operación ejecutada varias veces = el mismo resultado

Ejemplo: `LOAD daily orders`

❌ NON-IDEMPOTENT (ROTO):

```sql
INSERT INTO warehouse.orders SELECT * FROM raw_orders WHERE date = '2024-01-15';
-- Ejecución 1: Inserta 1000 rows
-- Ejecución 2 (retry): Inserta 1000 rows OTRA VEZ (ahora 2000 en total = ¡INCORRECTO!)
```

✅ IDEMPOTENT (CORRECTO):

```sql
DELETE FROM warehouse.orders WHERE date = '2024-01-15';
INSERT INTO warehouse.orders SELECT * FROM raw_orders WHERE date = '2024-01-15';
-- Ejecución 1: Elimina 0, inserta 1000 → Total 1000
-- Ejecución 2 (retry): Elimina 1000, inserta 1000 → Total 1000 (¡CORRECTO!)
```

Enfoque `idempotent` alternativo:

```sql
INSERT INTO warehouse.orders SELECT * FROM raw_orders WHERE date = '2024-01-15'
ON CONFLICT (order_id) DO UPDATE SET ... ; -- Upsert
```

Requisito en el mundo real:
-- CADA `data pipeline` debe ser `idempotent`
-- Suposición: Podría ejecutarse varias veces (`retries`, `backfills`)
-- Por lo tanto: Debe producir el mismo resultado independientemente del número de ejecuciones

### Idempotency Patterns

-- Pattern 1: `DELETE + INSERT` (`Full refresh` por día)

```sql
DELETE FROM dim_customers WHERE load_date = '2024-01-15';
INSERT INTO dim_customers
SELECT customer_id, name, email, '2024-01-15' as load_date
FROM raw.customers;
```

-- Pattern 2: `UPSERT` (Insertar o actualizar si existe)

```sql
MERGE INTO dim_customers t
USING (SELECT customer_id, name, email FROM raw.customers) s
ON t.customer_id = s.customer_id AND t.load_date = '2024-01-15'
WHEN MATCHED THEN UPDATE SET t.name = s.name, t.email = s.email
WHEN NOT MATCHED THEN INSERT VALUES (s.customer_id, s.name, s.email, '2024-01-15');
```

-- Pattern 3: `CREATE OR REPLACE` (`Views`)

```sql
CREATE OR REPLACE VIEW fct_sales AS
SELECT order_id, customer_id, amount FROM raw_orders WHERE date >= '2024-01-01';
-- La View siempre refleja los últimos raw data, idempotent por diseño
```

-- Pattern 4: `Incremental` con `checkpoint`

```sql
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
-- Idempotent: Si ya se cargó, ON CONFLICT previene duplicados
```

---

## Pattern 2: SLA & Monitoring

`SLA` = `Service Level Agreement` (garantía de tiempo)

```python
from airflow import DAG
from airflow.utils.dates import days_ago
from datetime import timedelta

dag = DAG(
    'critical_etl_with_sla',
    schedule_interval='0 2 * * *', # 2 AM UTC daily
    start_date=days_ago(30),
    sla=timedelta(hours=4), # Debe finalizar en 4 horas
    sla_miss_callback=notify_sla_miss, # Alerta si se incumple
    tags=['production', 'critical']
)

def notify_sla_miss(dag, task_list, blocking_task_list):
    """Se llama si el DAG incumple el SLA"""
    print(f"URGENTE: ¡SLA del Pipeline INCUMPLIDO!")
    print(f"Tasks involucradas: {task_list}")
    # Enviar alerta urgente de Slack/PagerDuty
    # Notificar al ingeniero de guardia inmediatamente

# SLAs de tareas individuales
t1 = PythonOperator(
    task_id='extract_data',
    python_callable=extract,
    sla=timedelta(minutes=30), # Debe finalizar a las 02:30
    sla_miss_callback=notify_task_sla_miss,
    dag=dag
)

t2 = PythonOperator(
    task_id='transform_data',
    python_callable=transform,
    sla=timedelta(minutes=30), # Debe finalizar a las 03:00
    dag=dag
)

t3 = PythonOperator(
    task_id='load_warehouse',
    python_callable=load,
    sla=timedelta(minutes=30), # Debe finalizar a las 03:30
    dag=dag
)

t1 >> t2 >> t3
```

Cronograma de `SLA`:
02:00 - Inicio
02:30 - t1 `SLA` (`extract` debe finalizar)
03:00 - t2 `SLA` (`transform` debe finalizar)
03:30 - t3 `SLA` (`load` debe finalizar)
06:00 - `SLA` general del `DAG` (todo terminado para esta hora)
Si alguno falla: Alerta inmediata

Métricas de `monitoring`

```python
dag_metrics = {
    "duration_minutes": 45, # Tiempo real
    "target_sla_minutes": 240, # 4 horas
    "sla_compliance": (240 - 45) / 240 * 100, # 81% de margen
    "status": "PASS"
}
```

---

## Pattern 3: Retry Strategies

```python
from airflow.utils.decorators import apply_defaults
from datetime import timedelta
```

Estrategia 1: `Linear backoff` (esperar el mismo tiempo entre `retries`)

```python
default_args_linear = {
    'retries': 3,
    'retry_delay': timedelta(minutes=5), # Esperar 5 min cada vez
}
```

Cronograma: Falla a las 02:00 → `Retry` 02:05 → `Retry` 02:10 → `Retry` 02:15 → Desistir 02:15
Estrategia 2: `Exponential backoff` (esperar más tiempo cada vez)

```python
default_args_exponential = {
    'retries': 3,
    'retry_delay': timedelta(minutes=1),
    'retry_exponential_backoff': True, # Duplicar la espera en cada retry
}
```

Cronograma: Falla a las 02:00 → `Retry` 02:01 → `Retry` 02:03 → `Retry` 02:07 → Desistir 02:07
Mejor para fallas transitorias (problemas de red)
Estrategia 3: Lógica de `retry` personalizada

```python
from airflow.exceptions import AirflowException

def custom_retry_logic():
    """Decidir si reintentar basándose en el tipo de error"""
    try:
        result = call_flaky_api()
        return result
    except ConnectionError as e:
        # Transitorio (red): Reintentar
        raise AirflowException(f"Network error (will retry): {e}")
    except ValueError as e:
        # Permanente (datos incorrectos): No reintentar
        raise Exception(f"Bad data (won't retry): {e}")
```

Estrategia 4: `Max retries` con `timeout`

```python
t1 = PythonOperator(
    task_id='api_call',
    python_callable=custom_retry_logic,
    retries=5,
    max_tries=7, # Timeout general
    execution_timeout=timedelta(minutes=10), # Máximo 10 min por intento
)
```

---

## Pattern 4: Backfilling (Re-ejecución Histórica)

```python
from datetime import datetime
```

Escenario: Nueva tabla añadida al `warehouse`
Necesidad de `backfill` de 1 año de datos históricos

```python
dag = DAG(
    'backfill_historical_orders',
    schedule_interval=None, # Disparador manual (no programado)
    start_date=datetime(2023, 1, 1), # Iniciar desde hace 1 año
    end_date=datetime(2024, 1, 1), # Terminar hoy
    catchup=True, # CRÍTICO: Ejecutar todos los intervalos perdidos
)
```

Cuando implementes este `DAG`:
Airflow ejecuta automáticamente los intervalos:
2023-01-01, 2023-01-02, ..., 2024-01-01
En paralelo o secuencialmente (depende de la configuración de `parallelism`)
Cada ejecución obtiene un `date context` único:

```python
def process_date(**context):
    ds = context['ds'] # Fecha de ejecución (ej., '2023-06-15')
    print(f"Processing data for {ds}")
    # Consulta WHERE date = {ds}
```

En `production`: Una vez finalizado el `backfill`, cambiar a la programación normal

```python
dag_prod = DAG(
    'orders_daily_etl',
    schedule_interval='0 2 * * *', # Ejecutar diariamente a las 2 AM
    start_date=datetime(2024, 1, 1),
    catchup=True, # Ponerse al día si el scheduler está caído
)
```

`Backfill` por línea de comandos:
`airflow dags backfill -s 2023-01-01 -e 2024-01-01 orders_daily_etl`
Realiza `backfills` de todos los días en el rango

---

## Pattern 5: Exactly-Once Semantics (Distribuido)

Desafío: `Distributed pipelines` (múltiples `workers`)
Garantía: Cada `record` procesado `EXACTLY ONCE` (no 0, no 2+)

Escenario: `Kafka consumer` → `Processing` → `Database`
Riesgo: Un `worker` falla a mitad del proceso, el mensaje se procesa dos veces

Solución 1: `Idempotent writes` (ya discutido)
Solución 2: `Distributed deduplication`

Implementación:

1.  Leer desde `Kafka offset X`
2.  Procesar mensaje
3.  Escribir en la base de datos con un ID único
4.  `COMMIT offset` (`Kafka` lo marca como procesado)
5.  Si hay un `crash` antes del `commit`: Reprocesar desde `offset X` (¡`idempotent`!)

Código (`Spark Structured Streaming`):

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("exactlyOnce").getOrCreate()

df = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "events") \
    .load()

result = df \
    .select("value") \
    .writeStream \
    .outputMode("append") \
    .option("checkpointLocation", "/tmp/checkpoint") \
    .option("idempotencyKey", "event_id") # Desduplicar
    .toTable("events_deduplicated")
```

Spark desduplica automáticamente dentro de una ventana de 1 hora
Combinar con `idempotent writes` = garantías de `exactly-once`

---

## Pattern 6: Data Freshness & Staleness Alerts

Monitorizar: ¿Están los `analytics` actualizados?

```python
def check_data_staleness(**context):
    """Alertar si los datos no se han actualizado en >2 horas"""
    query = """
    SELECT MAX(load_timestamp) as last_update
    FROM warehouse.fact_sales
    """
    result = execute_query(query)
    last_update = result['last_update']
    hours_old = (datetime.now() - last_update).total_seconds() / 3600

    if hours_old > 2:
        # STALE: ¡Alerta!
        alert(f"Los datos tienen {hours_old} horas, se esperaban fresh en 2h")
        raise AirflowException("SLA de data freshness incumplido")
    else:
        print(f"Los datos están fresh ({hours_old:.1f}h de antigüedad)")

t_freshness_check = PythonOperator(
    task_id='check_data_freshness',
    python_callable=check_data_staleness,
    dag=dag
)
```

Ejecutar al final del `pipeline`
`load_task >> t_freshness_check`

---

## Pattern 7: Failure Modes & Recovery

Modo de Falla 1: `Task` falla, reintenta, sigue fallando
├─ Qué: Error permanente (código incorrecto, datos inválidos)
├─ Recuperación: Alerta, investigación manual, corregir código, `redeploy`
└─ Acción: `PagerDuty` → El ingeniero investiga

Modo de Falla 2: `Task` agota el tiempo (tarda demasiado)
├─ Qué: Degradación del `performance`, bucle infinito
├─ Recuperación: Terminar `task`, escalar recursos, optimizar código
└─ Acción: Investigar por qué es lento, añadir `caching/indexes`

Modo de Falla 3: La dependencia `downstream` falla
├─ Qué: Alimentar datos a la `task`, pero la fuente no existe
├─ Recuperación: Esperar + `retry`, o saltar si es opcional
└─ Acción: Añadir `sensors` (esperar a `upstream`) o `skip_on_fail`

Modo de Falla 4: La verificación de `data quality` falla
├─ Qué: Datos fuera de rango, nulos donde no se esperan
├─ Recuperación: No cargar datos incorrectos al `warehouse`, alertar
└─ Acción: Poner en cuarentena los datos, notificar al equipo de origen

Estrategias de recuperación:

```python
@on_failure_callback
def alert_on_failure(context):
    task_instance = context['task_instance']
    print(f"Task {task_instance.task_id} FALLÓ")
    # Enviar mensaje de Slack
    # Crear ticket de JIRA
    # Notificar al de guardia si es crítico

@on_success_callback
def log_on_success(context):
    print(f"Task {context['task_instance'].task_id} tuvo éxito")
    # Registrar metrics
    # Incrementar contador de éxito
```

---

## Pattern 8: Monitoring & Observability

Métricas clave a `trackear`

```python
metrics = {
    "dag_run_duration": 45, # minutos
    "dag_sla_target": 240, # minutos
    "dag_success_rate": 0.98, # 98% de éxito
    "task_retry_count": 2, # necesitó 2 retries
    "data_row_count": 150000,
    "data_freshness": 15, # minutos de antigüedad
    "task_durations": {
        "extract": 5,
        "transform": 30,
        "load": 10,
    }
}
```

Umbrales de alerta

```python
alerts = {
    "dag_run_duration_exceeds": 300, # >5 horas = WARN
    "task_failure": True, # Cualquier falla = ALERT
    "data_freshness_exceeds": 120, # >2 horas de antigüedad = ALERT
    "row_count_delta": 0.5, # Cambio >50% = INVESTIGAR
}
```

Mejores prácticas de `logging`

```python
logging.info(f"Inicio del pipeline: {datetime.now()}")
logging.info(f"Rows extraídas: {row_count}")
logging.info(f"Rows transformadas: {transformed_count}")
logging.info(f"Rows cargadas: {loaded_count}")
logging.info(f"Fin del pipeline: {datetime.now()}, duración: {duration_min}m")
```

Si algo inesperado:

```python
logging.warning(f"Patrón inusual: {row_count} >> esperado {avg_row_count}")
logging.error(f"CRÍTICO: Falló la verificación de data quality: {error}")
```

---

## Errores Comunes en Entrevista

- **Error**: "Just retry forever" → **Solución**: Se necesita `exponential backoff` + `max retries` (no reintentar código roto)

- **Error**: No `idempotency` → **Solución**: `Retries` = garantizados, por lo tanto DEBE ser `idempotent`

- **Error**: Ignorar `SLAs` → **Solución**: `Production` = garantías de tiempo (o no lo llames "`production`")

- **Error**: No `monitoring` → **Solución**: "Funciona localmente" no significa que funcione en `production`

---

## Preguntas de Seguimiento

1.  **"¿Cómo aseguran `exactly-once`?"**
    - `Idempotent writes` (estrategia principal)
    - `Distributed deduplication` (para `Kafka`)
    - Combinado = garantías

2.  **"¿`Backfill strategy`?"**
    - Diario: `Small backfill` (datos de ayer)
    - Grande: Programar el `backfill job` por separado (no bloqueará el diario)

3.  **"¿`Task failure recovery`?"**
    - `Retries` (auto-recuperación)
    - `SLAs` (detectar problemas)
    - `Monitoring` (alertar a humanos)

4.  **"¿`How to test reliability`?"**
    - `Chaos testing` (inyectar fallas)
    - `Retry simulation` (forzar reintentos)
    - `Load testing` (¿qué falla bajo presión?)

---

## References

- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Exactly-Once Semantics - Kafka](https://kafka.apache.org/documentation/#semantics)
- [Distributed Systems Reliability - SRE Book](https://sre.google/books/)
