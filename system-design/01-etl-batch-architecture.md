# Diseñar ETL Pipeline Batch: Arquitectura a Escala

**Tags**: #system-design #etl #batch #architecture #real-interview  
**Empresas**: Amazon, Google, Meta, Databricks  
**Dificultad**: Senior  
**Tiempo estimado**: 30 min  

---

## TL;DR

ETL batch = Extract (fuente) → Transform (proceso) → Load (destino), ejecuta en horarios (diario, horario). Arquitectura: Data Lake (raw) → Processing (Spark) → Data Warehouse (analytics). Key decisions: Spark vs SQL, Partitioning, Incremental vs Full, Scheduling (Airflow/Cron), Monitoring. Trade-off: Simplicidad vs Performance vs Costo.

---

## Problema Real

**Escenario:**

- E-commerce con 1 millón de transacciones/día
- 50+ fuentes de datos (databases, APIs, S3, Kafka)
- Necesitas reporte consolidado cada mañana
- Analytics team (5 personas) + 2 data engineers
- Budget: $10k/mes cloud infrastructure

**Preguntas:**

1. ¿Cómo diseñas el pipeline?
2. ¿Qué tecnologías usas (Spark? SQL? Serverless?)?
3. ¿Cómo manejas freshness de datos?
4. ¿Cómo monitoreás si algo falla?
5. ¿Cómo escalas a 10M transacciones/día?

---

## Solución Propuesta

### Arquitectura (3 Capas)

┌─────────────────────────────────────────────────────────────┐
│ SOURCES (50+ systems) │
│ ├─ Databases (MySQL, PostgreSQL) │
│ ├─ APIs (3rd party services) │
│ ├─ S3 (batch files) │
│ └─ Kafka (real-time streams, pero batch consume) │
└─────────────────────┬───────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ RAW LAYER (Data Lake - S3) │
│ ├─ s3://data-lake/raw/mysql/customers/date=2024-01-15/ │
│ ├─ s3://data-lake/raw/api/payments/date=2024-01-15/ │
│ ├─ s3://data-lake/raw/s3/events/date=2024-01-15/ │
│ └─ Landing zone (sin procesamiento) │
└─────────────────────┬───────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ PROCESSING LAYER (Spark on EMR/Databricks) │
│ ├─ Validación (NULL handling, duplicates) │
│ ├─ Transformación (joins, aggregations) │
│ ├─ Enriquecimiento (add business logic) │
│ └─ Output → Processed (s3://data-lake/processed/) │
└─────────────────────┬───────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ WAREHOUSE LAYER (Redshift/BigQuery) │
│ ├─ Dimensional tables (customers, products, etc.) │
│ ├─ Fact tables (transactions, events) │
│ └─ Ready for analytics/BI tools │
└─────────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ ANALYTICS LAYER │
│ ├─ Dashboards (Tableau, Looker) │
│ ├─ Reports (automated daily) │
│ └─ Self-service queries (analysts) │
└─────────────────────────────────────────────────────────────┘

text

---

### Stack Tecnológico

| Componente | Tecnología | Por Qué | Costo |
|-----------|-----------|--------|--------|
| **Orquestación** | Apache Airflow | DAG visibility, retries, monitoring | $1k/mes |
| **Processing** | Spark (EMR) | Distribuido, flexible, costo-efectivo | $3k/mes |
| **Storage (Raw)** | S3 | Económico, escalable, durable | $1k/mes |
| **Warehouse** | Redshift | Integrado, queries rápidas, comodin | $4k/mes |
| **Monitoring** | CloudWatch + DataDog | Alertas, logs, metrics | $1k/mes |
| **Total** | | | ~$10k/mes ✓ |

---

### Pipeline Diario: Ejecución

05:00 AM - START: Airflow scheduler despierta
├─ Task 1: Extraer de MySQL (incremental: Δ últimas 24h)
├─ Task 2: Extraer de APIs (retry 3x si falla)
├─ Task 3: Descargar S3 files (con validation)
└─ Pause en Task 4 si Task 1-3 fallan

06:00 AM - TRANSFORM: Spark EMR cluster
├─ Spark job 1: Validar data (NULLs, duplicates, schema)
├─ Spark job 2: Join customers + orders + payments
├─ Spark job 3: Aggregate hourly → daily metrics
└─ Spark job 4: Write to Processed (partitioned)

07:00 AM - LOAD: Transfer a Redshift
├─ COPY customers from s3://processed/customers/
├─ COPY orders from s3://processed/orders/
├─ COPY transactions from s3://processed/transactions/
└─ Refresh warehouse views

08:00 AM - VALIDATE: Data quality checks
├─ Row count validations (vs yesterday ±5%)
├─ NULL % checks
├─ Freshness checks (arrival time)
└─ Alert si falla algo

09:00 AM - READY: Dashboards refresh
└─ Analytics team accede a datos frescos

text

---

### Key Design Decisions

#### Decision 1: Incremental vs Full Load

❌ FULL LOAD (No recomendado para este caso)

Re-procesar TODO cada día (50M rows + 50 sources)

Tiempo: 4+ horas (demasiado)

Costo: Alto (mucho compute)

✅ INCREMENTAL (Recomendado)

Solo procesar cambios de últimas 24h

MySQL: WHERE updated_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)

APIs: Pedir "last modified after X timestamp"

Tiempo: 30-45 min (factible)

Costo: Bajo (menos datos)

text

**Trade-off:** Incremental = más complejo (manejar updates/deletes), pero Performance >> Full Load.

---

#### Decision 2: Spark Cluster Setup

❌ On-Demand (payg por hora)

Flexible pero caro si spike

Ideal: testing, exploratorio

✅ Reserved Instances (RI, prepaid)

1 año commitment → 30% discount

Cluster siempre activo

Ideal: pipeline predecible (diario)

✅ Spot Instances (mezcla)

Nodo master: On-Demand (estable)

Nodos worker: Spot (50% discount)

Riesgo: Spot puede terminar, pero OK si retry

text

**Para este caso:** Reserved (master) + Spot (workers) = balance costo/reliability.

---

#### Decision 3: Retry Strategy

Pipeline failures = inevitables (network, source outage, etc.)

Task 1 (Extract MySQL):
├─ Retry 1: Esperar 5 min, reintentar
├─ Retry 2: Esperar 15 min, reintentar
├─ Retry 3: Esperar 60 min, reintentar
└─ Si falla 3x: Alert + manual intervention

Task 2-4 (Transform/Load):
├─ Solo retry inmediato (no retry tardío)
└─ Si falla: Rollback warehouse changes

Filosofía: Tolera failures transitorios (network blip), fail-fast en lógica

text

---

### Monitoring & Alerting

Pseudocódigo: checks ejecutados post-pipeline
class DataQualityChecks:
def row_count_validation(self, table, yesterday_count):
today_count = get_count(table)
pct_change = abs((today_count - yesterday_count) / yesterday_count) * 100

text
    if pct_change > 10:  # +/- 10% es threshold
        alert("Row count variance", f"{table}: {pct_change}%")

def freshness_check(self, table):
    max_timestamp = get_max_timestamp(table)
    hours_old = (now() - max_timestamp).hours
    
    if hours_old > 2:  # Datos no frescos
        alert("Data freshness", f"{table}: {hours_old} hours old")

def null_percentage(self, table, column, threshold=5):
    null_pct = get_null_percent(table, column)
    
    if null_pct > threshold:
        alert("Null percentage", f"{table}.{column}: {null_pct}%")

def duplicate_check(self, table, key_cols):
    dups = count_duplicates(table, key_cols)
    
    if dups > 0:
        alert("Duplicates detected", f"{table}: {dups} duplicate rows")
text

---

### Scaling: 1M → 10M Transacciones/Día

**Cambios necesarios:**

COMPUTE:

Spark cluster: 8 workers → 20 workers (parallelismo ↑)

Memory per worker: 8GB → 16GB

STORAGE:

S3 → S3 + Glacier (archive old data)

Redshift: 128GB → 512GB (más nodos)

PROCESSING:

Pipeline time: 30 min → 1.5h (OK, < 24h window)

Batch size: Reducir para memory

COST:

$10k/mes → $25k/mes (2.5x, pero lineal no exponencial)

text

**Architectural changes:**

Si crecer a 100M/día (imposible con batch diario):

Cambiar a HYBRID: Batch + Streaming

Raw → Kafka (real-time ingest)

Late-arriving updates: Upsert con CDC pattern

Warehouse: Usar partitioning + clustering

text

---

## Trade-offs Analizados

| Decision | Opción A | Opción B | Elegida | Por Qué |
|----------|----------|----------|---------|---------|
| **Batch vs Real-Time** | Batch | Real-Time | Batch | Simpler, 1M/día = OK |
| **Full vs Incremental** | Full Load | Incremental | Incremental | Performance |
| **Spark vs SQL** | Spark | SQL | Spark | Flexibility, UDFs |
| **Cloud vs On-Prem** | Cloud | On-Prem | Cloud | Cost, scalability |
| **Airflow vs Cron** | Airflow | Cron | Airflow | Monitoring, DAG |
| **Redshift vs BigQuery** | Redshift | BigQuery | Redshift | Cost at this scale |

---

## Alternativas Consideradas

### Alternativa 1: Serverless (AWS Glue)

✓ No manage clusters
✓ Auto-scaling
✗ Menos control
✗ Más caro para este volumen
✗ Cold start latency

Recomendación: Solo si <100GB/día

text

### Alternativa 2: Pure Real-Time (Kafka → Flink)

✓ Data freshness: Milliseconds
✓ No batch delays
✗ Mucho más complejo
✗ 3-5x más caro
✗ 24/7 infrastructure

Recomendación: Solo si "live" dashboard necesario

text

### Alternativa 3: Data Warehouse as Source (no Data Lake)

✓ Simpler (no S3)
✗ Perdés raw data (compliance issue)
✗ Less flexible

Recomendación: NO, Data Lake pattern es mejor

text

---

## Errores Comunes en Entrevista

- **Error**: "Voy a usar Hadoop/MapReduce" → **Solución**: Spark es newer, faster, simpler. Hadoop es legacy

- **Error**: "Voy a procesar TODO cada día" → **Solución**: Incremental es más eficiente. Entiende trade-off

- **Error**: No pensar en monitoring/alerting → **Solución**: Production pipeline SIN alerts = disaster. Data quality es parte critical

- **Error**: "Real-time es siempre mejor" → **Solución**: Real-time = complejo, caro. Batch es simple, sufficient para 1M/día

---

## Preguntas de Seguimiento

1. **"¿Cómo manejas late-arriving data?"**
   - Ventana de retrazo (aceptar updates 48h tardío)
   - Upsert en warehouse (CDC pattern)
   - Trigger recompute de aggregations

2. **"¿Qué pasa si source database falla?"**
   - Retry strategy (exponential backoff)
   - Fallback a snapshot anterior si available
   - Alert: Manual intervention

3. **"¿Cómo coordinas con 50+ source teams?"**
   - SLA contract (cuándo datos disponibles)
   - Ownership matrix (quién es responsible)
   - Feedback loop (si break, ellos saben

4. **"¿Cómo presupuestas/monitoreás costo?"**
   - Budget alert si exceed $10k
   - Tag resources (cost center)
   - Reserved instances para committed spend

---

## References

- [AWS EMR + Spark Architecture](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-architecture.html)
- [Apache Airflow DAG Scheduling](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html)
- [ETL Best Practices - Databricks](https://docs.databricks.com/solutions/etl.html)
- [Data Lake Architecture - AWS](https://aws.amazon.com/architecture/datalake/)

