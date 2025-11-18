# Monitoreo de Calidad de Datos a Escala

**Tags**: #data-quality #monitoring #alerts #production #real-interview

---

## TL;DR

Monitoreo de calidad de datos = detectar automáticamente problemas (NULLs, duplicados, cambios de esquema, datos tardíos). Métricas: null_percentage, duplicate_count, row_count_delta, schema_validation, freshness. Alertas si las métricas exceden umbrales predefinidos. Herramientas: Great Expectations (automatiza tests), DataDog/Prometheus + Grafana, Slack/Email. Trade-offs: Strict (muchas alertas) vs Loose (menos alertas). Para producción: Balance entre detección temprana vs fatiga de alertas.

---

## Concepto

- **Qué es**: Monitoreo de calidad de datos = verificar que los datos cumplen con estándares de calidad
- **Por qué importa**: Los datos malos conducen a análisis incorrectos y decisiones empresariales malas. En producción, el 80% del tiempo se gasta en calidad de datos
- **Principio clave**: Calidad de datos = "Basura dentro, basura fuera" (GIGO). El monitoreo automático es esencial

---

## Framework de Monitoreo de Calidad de Datos

### Great Expectations (Automatización de Validaciones)

```python
from great_expectations import GreatExpectations
from pyspark.sql import SparkSession

class DataQualityChecks(GreatExpectations):
    def __init__(self, spark):
        self.spark = spark

    def validate_customer_schema(self, df):
        """Valida esquema de clientes"""
        return self.expect_dataframe_to_have_columns(df, ['customer_id', 'name', 'email'])
        return self.expect_column_values_to_be_unique('customer_id')
        return self.expect_column_values_to_not_be_null('email')

    def validate_orders_schema(self, df):
        """Valida esquema de pedidos"""
        return self.expect_column_values_to_be_positive('amount')
        return self.expect_column_values_to_be_between('quantity', 1, 1000)

    def validate_row_count(self, table, expected_range):
        """Valida que el conteo de filas esté en rango esperado"""
        actual_count = self.spark.sql(f"SELECT COUNT(*) FROM {table}")
        min_val, max_val = expected_range
        actual_count = actual_count.collect()[0][0]

        if not (min_val <= actual_count <= max_val):
            raise Exception(f"Row count fuera de rango: {actual_count} (esperado: {min_val}-{max_val}")

    def validate_freshness(self, table, max_hours):
        """Valida frescura de datos"""
        max_timestamp = self.spark.sql(f"SELECT MAX(updated_at) FROM {table}")
        hours_old = (datetime.now() - max_timestamp).total_seconds() / 3600

        if hours_old > max_hours:
            raise Exception(f"Datos demasiado viejos: {hours_old} horas")

    def validate_duplicates(self, table, key_cols):
        """Detecta duplicados por clave"""
        dups = self.spark.sql(f"""
            SELECT {', '.join(key_cols), COUNT(*) as count
            FROM {table}
            GROUP BY {', '.join(key_cols)
            HAVING COUNT(*) > 1
        """)

        if dups.count() > 0:
            raise Exception(f"Se detectaron {dups.count()} duplicados en {table}")

    def validate_null_percentage(self, table, column, threshold=5):
        """Verifica porcentaje de nulos"""
        null_pct = self.spark.sql(f"""
            SELECT (COUNT(*) - COUNT({column}) * 100 / COUNT(*))
            FROM {table}
        """).collect()[0][0]

        if null_pct > threshold:
            raise Exception(f"Porcentaje de nulos en {table}.{column}: {null_pct}%")

    def run_all_checks(self, tables_info):
        """Ejecuta todas las validaciones"""
        results = {}

        for table_name, config in tables_info.items():
            df = self.spark.table(table_name)

            # Validación de esquema
            self.validate_schema(table_name, df)

            # Validación de datos
            self.validate_row_count(table_name, config["expected_range"])

            # Validación de nulos
            self.validate_null_percentage(table_name, config["null_thresholds"])

            # Validación de duplicados
            for key_cols in config["key_columns"]:
                self.validate_duplicates(table_name, key_cols)

            results[table_name] = {
                "row_count": df.count(),
                "schema_validation": "PASS",
                "null_percentage": null_pct,
                "duplicate_count": dups.count()
            }

        return results
```

### Sistema de Alertas

```python
class AlertManager:
    def __init__(self, alert_webhook, email_smtp):
        self.alert_webhook = alert_webhook
        self.email_smtp = email_smtp

    def send_alert(self, alert_type, message, details):
        """Envía alerta según el tipo"""
        alert_data = {
            "type": alert_type,
            "message": message,
            "details": details,
            "timestamp": datetime.now()
        }

        self.alert_webhook(alert_data)

    def send_data_quality_alert(self, table, metric, value, threshold):
        """Envía alerta si la métrica excede el umbral"""
        if value > threshold:
            self.send_alert("DATA_QUALITY", f"{table}.{metric}: {value} (threshold: {threshold})")

    def send_fraud_alert(self, table, fraud_count):
        """Alerta si hay demasiados fraudes"""
        if fraud_count > threshold:
            self.send_alert("FRAUD", f"{table}: {fraud_count} fraudes detectados")

    def send_latency_alert(self, component, latency_ms, threshold_ms):
        """Alerta si la latencia excede el umbral"""
        if latency_ms > threshold_ms:
            self.send_alert("LATENCY", f"{component}: {latency_ms}ms (threshold: {threshold_ms}ms")
```

---

## Métricas Clave

### Métricas de Calidad de Datos

```python
class DataQualityMetrics:
    def calculate_null_percentage(self, table, column):
        null_count = self.spark.sql(f"""
            SELECT (COUNT(*) - COUNT({column}) * 100 / COUNT(*))
            FROM {table}
        """).collect()[0][0]

        return null_count / self.spark.sql(f"SELECT COUNT(*) FROM {table}").collect()[0][0] * 100

    def calculate_duplicate_count(self, table, key_cols):
        dups = self.spark.sql(f"""
            SELECT {', '.join(key_cols), COUNT(*) as count
            FROM {table}
            GROUP BY {', '.join(key_cols)
            HAVING COUNT(*) > 1
        """).collect()

        return dups

    def calculate_row_count_delta(self, table, hours=24):
        today_count = self.spark.sql(f"""
            SELECT COUNT(*) FROM {table}
            WHERE DATE(updated_at) >= DATE_SUB(CURRENT_DATE, INTERVAL '{hours} HOUR)
            """).collect()[0][0]

        yesterday_count = self.spark.sql(f"""
            SELECT COUNT(*) FROM {table}
            WHERE DATE(updated_at) BETWEEN DATE_SUB(CURRENT_DATE, INTERVAL '{hours+24} HOUR, DATE_SUB(CURRENT_DATE, INTERVAL '{hours} HOUR)
            """).collect()[0][0]

        delta = today_count - yesterday_count

        return {
            "table": table,
            "today_count": today_count,
            "yesterday_count": yesterday_count,
            "row_count_delta": delta,
            "pct_change": delta / yesterday_count * 100
        }
```

### Métricas de Rendimiento

```python
class PerformanceMetrics:
    def calculate_processing_time(self, table):
        """Calcula tiempo de procesamiento"""
        start_time = time.time()

        self.spark.sql(f"SELECT COUNT(*) FROM {table}").collect()[0][0]

        end_time = time.time()
        processing_time = end_time - start_time

        return processing_time

    def calculate_storage_size(self, table):
        """Calcula tamaño de almacenamiento"""
        table_size_mb = self.spark.sql(f"""
            SELECT SUM(size_bytes) / (1024 * 1024) AS size_mb
            FROM information_schema.tables
            WHERE table = '{table}'
            """).collect()[0][0]

        return table_size_mb

    def calculate_query_performance(self, query):
        """Calcula tiempo de consulta"""
        start_time = time.time()

        result = self.spark.sql(query)
        end_time = time.time()

        return end_time - start_time
```

---

## Implementación de Alertas

### Sistema de Alertas

```python
class AlertSystem:
    def __init__(self):
        self.alert_rules = {
            "row_count_delta": {"threshold": 10, "severity": "HIGH"},
            "null_percentage": {"threshold": 5, "severity": "MEDIUM"},
            "duplicate_count": {"threshold": 1, "severity": "HIGH"},
            "freshness": {"threshold": 2, "severity": "MEDIUM"},
            "processing_time": {"threshold": 500, "severity": "MEDIUM"},
            "query_performance": {"threshold": 1000, "severity": "HIGH"}
        }
        }

    def evaluate_metric(self, metric_name, table, value):
        rule = self.alert_rules.get(metric_name)

        if value > rule["threshold"]:
            severity = rule["severity"]
            message = f"{metric_name}: {value} (threshold: {rule['threshold']})"

            self.send_alert(severity, message, {
                "table": table,
                "metric": metric_name,
                "value": value
            })

# Uso
alert_system = AlertSystem()
alert_system.evaluate_metric("row_count_delta", "customers", 15)  # Excede umbral de 10%
# Alerta: "HIGH: row_count_delta: 15% (threshold: 10%)
```

---

## Errores Comunes en Entrevista

- **Error**: "Monitorear TODO" → **Solución**: Priorizar métricas críticas y establecer alertas automáticas

- **Error**: "Alertar demasiado" → **Solución**: Implementar umbrales de alerta para evitar fatiga de alertas

- **Error**: "No monitorear el rendimiento" → **Solución**: Incluir métricas de rendimiento y latencia

- **Error**: "No alertar cambios en el esquema" → **Solución**: Detectar cambios en el esquema y alertar

---

## Preguntas de Seguimiento Típicas

1. **"¿Cómo manejas el ruido en las métricas?"**
   - Suavizado de métricas
   - Filtros de ruido
   - Umbral adaptativo según la hora del día

2. **"¿Cómo manejas alertas falsos positivos?"**
   - Confirmación con datos históricos
   - Validación cruzada
   - Alertas en cascada con diferentes niveles de severidad

3. **"¿Cómo escalas a 1000+ pipelines?"**
   - Plataforma centralizada con estándares predefinidos
   - Automatización de alertas
   - Jerarquía de alertas

4. **"¿Cómo manejas cambios en el esquema?"**
   - Versionado de esquemas
   - Validación de compatibilidad hacia atrás
   - Alertas sobre cambios de esquema

---

## Referencias

- https://learn.microsoft.com/en-us/azure/databricks/data-quality-monitoring/
- https://www.ibm.com/think/topics/data-quality-monitoring-techniques
