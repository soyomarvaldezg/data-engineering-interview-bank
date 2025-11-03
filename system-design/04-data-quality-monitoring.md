# Data Quality Monitoring at Scale

**Tags**: #system-design #data-quality #monitoring #alerting #production #real-interview  
**Empresas**: Amazon, Google, Meta, Stripe, Uber  
**Dificultad**: Senior  
**Tiempo estimado**: 25 min  

---

## TL;DR

Data quality monitoring = automatizar detección de anomalías en datos (NULL %, duplicates %, schema changes, freshness). Métricas: null_percentage, duplicate_count, row_count_delta, schema_validation, latency. Alertas si métricas bajan vs baseline. Herramientas: Great Expectations, dbt tests, custom Spark jobs. Trade-off: Strict (alerta mucho, ruido) vs Loose (pierdes problemas).

---

## Problema Real

**Escenario:**

- 100+ data pipelines en producción
- 50+ fuentes de datos (databases, APIs, S3)
- Data team = 3 personas (no pueden monitorear manual)
- Descubren bugs DESPUÉS (dashboards dan números malos)
- Necesitan: Alerta ANTES que datos lleguen a analytics

**Preguntas:**

1. ¿Cómo detectas data quality issues automáticamente?
2. ¿Qué métricas monitoreas?
3. ¿Cuándo alertas vs ignoras?
4. ¿Cómo evitas "alert fatigue" (ruido)?
5. ¿Cómo escalas monitoreo a 100+ pipelines?

---

## Solución: Data Quality Framework

### Arquitectura

┌──────────────────────────┐
│ Data Pipeline │
│ (ETL job running) │
└────────────┬─────────────┘
│
▼
┌──────────────────────────────────────┐
│ DATA QUALITY CHECKS (Post-Load) │
├──────────────────────────────────────┤
│ ✓ Schema validation │
│ ✓ Row count checks │
│ ✓ NULL percentage checks │
│ ✓ Duplicate detection │
│ ✓ Freshness checks │
│ ✓ Statistical anomalies │
└────────────┬─────────────────────────┘
│
┌────────┴────────┐
│ │
▼ ▼
┌─────────┐ ┌──────────────┐
│ PASS │ │ FAIL │
│ Continue│ │ Stop + Alert │
└────────┘ └──────────────┘
│ │
▼ ▼
┌────────────┐ ┌──────────────────┐
│ Warehouse │ │ Quarantine Zone │
│ Ready │ │ (manual review) │
└────────────┘ └──────────────────┘

text

---

### Métricas de Calidad

Pseudocódigo: Checks ejecutados post-ingesta
class DataQualityChecks:

text
# 1. SCHEMA VALIDATION
def validate_schema(self, df, expected_schema):
    """Verifica columnas, tipos, orden"""
    actual_schema = df.schema
    if actual_schema != expected_schema:
        raise Exception(f"Schema mismatch: {actual_schema}")

# 2. ROW COUNT VALIDATION
def validate_row_count(self, df, yesterday_count):
    """Detecta anomalías: si sube/baja demasiado"""
    current_count = df.count()
    pct_change = abs((current_count - yesterday_count) / yesterday_count) * 100
    
    if pct_change > 15:  # Threshold: ±15%
        raise Exception(f"Row count variance: {pct_change}%")

# 3. NULL PERCENTAGE
def validate_nulls(self, df, column, threshold=5):
    """NULL % no puede exceder threshold"""
    null_count = df.filter(col(column).isNull()).count()
    null_pct = (null_count / df.count()) * 100
    
    if null_pct > threshold:
        raise Exception(f"{column}: {null_pct}% NULLs (threshold: {threshold}%)")

# 4. DUPLICATE DETECTION
def validate_duplicates(self, df, key_cols):
    """Detecta filas duplicadas por clave"""
    dups = df.groupBy(key_cols).count().filter(col("count") > 1).count()
    
    if dups > 0:
        raise Exception(f"Found {dups} duplicate rows")

# 5. FRESHNESS CHECK
def validate_freshness(self, df, timestamp_col, max_hours=2):
    """Datos no pueden ser muy viejos"""
    max_timestamp = df.agg(max(col(timestamp_col))).collect()
    hours_old = (datetime.now() - max_timestamp).total_seconds() / 3600
    
    if hours_old > max_hours:
        raise Exception(f"Data {hours_old} hours old (max: {max_hours})")

# 6. BUSINESS LOGIC VALIDATION
def validate_business_rules(self, df):
    """Verifica reglas específicas del negocio"""
    invalid_rows = df.filter(
        (col("amount") < 0) |  # No negative amounts
        (col("age") > 150) |   # Reasonable age
        ~col("email").rlike(r'^[\w\.-]+@[\w\.-]+\.\w+$')  # Valid email
    ).count()
    
    if invalid_rows > 0:
        raise Exception(f"Found {invalid_rows} invalid rows")

# 7. STATISTICAL ANOMALY DETECTION
def detect_statistical_anomalies(self, df, column):
    """Detecta outliers usando Z-score"""
    stats = df.select(
        avg(col(column)).alias("mean"),
        stddev(col(column)).alias("stddev")
    ).collect()
    
    anomalies = df.filter(
        abs((col(column) - stats.mean) / stats.stddev) > 3  # 3-sigma rule
    ).count()
    
    if anomalies > 0:
        return f"Statistical anomalies: {anomalies} outliers detected"
text

---

### Implementación: Great Expectations

import great_expectations as gx

===== SETUP =====
context = gx.get_context()

===== DEFINE EXPECTATIONS =====
validator = context.sources.pandas_default.read_pandas(df)

validator.expect_table_columns_to_match_ordered_list([
"customer_id", "name", "email", "age", "signup_date"
])

validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_not_be_null("name")

validator.expect_column_values_to_match_regex("email", r'^[\w.-]+@[\w.-]+.\w+$')

validator.expect_column_values_to_be_between("age", min_value=0, max_value=150)

validator.expect_column_values_to_be_in_set("country", ["USA", "CA", "MX"])

===== VALIDATE =====
checkpoint = context.add_or_update_checkpoint(
name="customer_data_checkpoint",
validator=validator
)

result = checkpoint.run()

if result.success:
print("✓ All checks passed")
else:
print("✗ Validation failed")
for check in result.results:
print(f" - {check.expectation_config.expectation_type}: FAILED")

text

---

### Baseline vs Threshold Strategy

===== DYNAMIC BASELINES =====
Usar histórico para detectar anomalías
class BaselineCalculator:

text
def calculate_baseline(self, metric_history):
    """Calcula baseline + thresholds del histórico"""
    import numpy as np
    
    values = metric_history[-30:]  # Últimos 30 días
    mean = np.mean(values)
    stddev = np.std(values)
    
    return {
        "mean": mean,
        "lower_bound": mean - 2*stddev,  # -2 sigma
        "upper_bound": mean + 2*stddev   # +2 sigma
    }

def is_anomaly(self, current_value, baseline):
    """Detecta si valor actual es anomalía"""
    if current_value < baseline["lower_bound"]:
        return True, "BELOW_EXPECTED"
    if current_value > baseline["upper_bound"]:
        return True, "ABOVE_EXPECTED"
    return False, "NORMAL"
===== USAGE =====
baseline = calculator.calculate_baseline(row_count_history)

current_row_count = 5000
is_anom, reason = calculator.is_anomaly(current_row_count, baseline)

if is_anom:
alert(f"Row count anomaly: {reason} (current: {current_row_count}, expected: {baseline['mean']})")

text

---

## Alert Strategy: Evitar Fatiga

### Severidad de Alertas

SEVERITY 1 (CRITICAL - Page on-call):
├─ Schema changed (columns missing)
├─ Zero rows in fact table
├─ Data > 24h old (freshness SLA broke)
└─ Action: Immediate investigation

SEVERITY 2 (HIGH - Alert but don't page):
├─ NULL % > 10%
├─ Duplicates detected
├─ Row count change > 20%
└─ Action: Review next morning

SEVERITY 3 (LOW - Log only):
├─ Metric changed slightly
├─ Warning level thresholds
└─ Action: Monitor, no action needed

text

---

### Smart Alerting Rules

❌ ALERT FATIGUE: Alertar por todo
if metric != yesterday:
alert() # Alerta TODOS los días

✅ SMART ALERTING: Solo anomalías reales
def should_alert(current, baseline, severity):

text
# Ignore pequeños cambios
if abs(current - baseline["mean"]) / baseline["mean"] < 0.05:
    return False  # < 5% change = ignore

# Severe anomalies siempre
if abs(current - baseline["mean"]) > 3 * baseline["stddev"]:
    return True, "CRITICAL"

# Moderate anomalies solo si persistent (2+ days)
if abs(current - baseline["mean"]) > 2 * baseline["stddev"]:
    if persistent_count >= 2:
        return True, "HIGH"

return False
text

---

## Monitoreo a Escala (100+ Pipelines)

### Centralized Monitoring Platform

┌─────────────────────────────────────────┐
│ DATA QUALITY PLATFORM (Central) │
├─────────────────────────────────────────┤
│ ├─ Metadata registry (todas las tables) │
│ ├─ Rules engine (checks automáticos) │
│ ├─ Execution engine (corre checks) │
│ ├─ Alerting service (notificaciones) │
│ └─ Dashboard (visualización) │
└─────────────────────────────────────────┘
▲ ▲ ▲
│ │ │
┌────┴────┬────┴────┬────┴────┐
│ │ │ │
Pipeline Pipeline Pipeline ...
1 2 3

text

---

### Declarative Rules (Código)

rules.yaml: Define checks sin escribir código
tables:
customers:
checks:
- type: schema
columns: [id, name, email, age]
- type: row_count
baseline: 1000000
tolerance: 15% # ±15%
- type: null_percentage
column: email
threshold: 1%
- type: duplicates
key_columns: [id]
- type: freshness
timestamp_column: updated_at
max_hours: 2

orders:
checks:
- type: row_count
baseline: 50000
tolerance: 20%
- type: null_percentage
column: customer_id
threshold: 0% # NO NULLs permitidos
- type: business_rule
rule: "amount > 0"

Ejecución automática: Platform lee yaml, corre checks
text

---

## Ejemplo Real: End-to-End

from great_expectations import ValidationOperator
from datetime import datetime

class DataQualityPipeline:

text
def run_quality_checks(self, table_name, df):
    """Ejecuta checks y genera reporte"""
    
    checks_passed = []
    checks_failed = []
    
    # Check 1: Schema
    try:
        self.validate_schema(df, table_name)
        checks_passed.append("Schema validation")
    except Exception as e:
        checks_failed.append(f"Schema: {str(e)}")
    
    # Check 2: Row count
    try:
        baseline = self.get_baseline(table_name)
        self.validate_row_count(df, baseline)
        checks_passed.append("Row count validation")
    except Exception as e:
        checks_failed.append(f"Row count: {str(e)}")
    
    # Check 3: NULLs
    try:
        self.validate_nulls(df, table_name)
        checks_passed.append("NULL validation")
    except Exception as e:
        checks_failed.append(f"NULLs: {str(e)}")
    
    # Generate report
    report = {
        "table": table_name,
        "timestamp": datetime.now(),
        "total_rows": df.count(),
        "passed_checks": len(checks_passed),
        "failed_checks": len(checks_failed),
        "failures": checks_failed
    }
    
    # Save to monitoring DB
    self.save_report(report)
    
    # Alert si failed
    if checks_failed:
        severity = "CRITICAL" if len(checks_failed) > 2 else "HIGH"
        self.send_alert(table_name, checks_failed, severity)
    
    return report
text

---

## Errores Comunes en Entrevista

- **Error**: Monitorear TODO (genera ruido) → **Solución**: Prioriza checks críticos, ignora ruido

- **Error**: Thresholds fijos (no adaptables)** → **Solución**: Baselines dinámicos (histórico)

- **Error**: Alertas sin acción (qué hace el data engineer?) → **Solución**: Alerta debe incluir: qué falló, por qué, acción recomendada

- **Error**: No monitorear latencia/freshness → **Solución**: Datos válidos pero viejos = inútil

---

## Preguntas de Seguimiento

1. **"¿Cómo balanceas sensitive vs noisy checks?"**
   - Usar z-score, percentiles
   - Dynamic baselines, no fixed thresholds

2. **"¿Cómo manejas cambios esperados?"**
   - Update baselines post-change
   - Documentar expected anomalies

3. **"¿Cost de data quality monitoring?"**
   - Run checks en paralelo (post-load)
   - ~10% overhead vs main pipeline

4. **"¿Cómo handlea test data vs production?"**
   - Separate check profiles
   - Different thresholds por environment

---

## References

- [Great Expectations - Data Validation](https://greatexpectations.io/)
- [dbt tests - Data Testing](https://docs.getdbt.com/docs/building-a-dbt-project/tests)
- [Monitoring Data Pipelines - Databricks](https://docs.databricks.com/data-quality/index.html)

