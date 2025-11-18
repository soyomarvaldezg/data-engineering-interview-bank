# Calidad de Datos en PySpark: Manejo de NULLs y Duplicados

**Tags**: #pyspark #data-quality #validation #null-handling #duplicates #real-interview

---

## TL;DR

Data quality en Spark = manejar NULLs, detectar duplicados, validar reglas negocio. Usa `.isNull()`, `.isNotNull()`, `coalesce()` para NULLs. Para duplicados: `drop_duplicates()` o `Window + row_number()`. Valida con `.filter()` + condicionales. En producción: crea métricas de calidad (null %, duplicates %, valid %) y alerta si baja

---

## Concepto Core

- **Qué es**: Data quality = asegurar datos sean correctos, completos, sin duplicados. Fundamental en data pipelines
- **Por qué importa**: Datos sucios = análisis sucio = decisiones malas. 80% del tiempo en pipelines es validación/limpieza
- **Principio clave**: Valida TEMPRANO. Detecta problemas en ingesta, no después en analytics

---

## Memory Trick

**"Basura dentro, basura fuera"** (GIGO: Garbage in, garbage out) — Si data está sucia, output es basura. Validación es inversión que paga

---

## Cómo explicarlo en entrevista

**Paso 1**: "Data quality = manejar anomalías (NULLs, duplicados, valores inválidos)"

**Paso 2**: "Para NULLs: `.isNull()`, `.isNotNull()`, `coalesce()`. Para duplicados: `drop_duplicates()` o window ranking"

**Paso 3**: "Valida reglas de negocio con `.filter()` + condicionales. Registra métricas de calidad"

**Paso 4**: "En prod: alertas si calidad baja (ej: null % sube, duplicates %, invalid records %)"

---

## Código/Query ejemplo

### Datos: Customers con problemas

| customer_id | name  | email               | age  | signup_date             |
| ----------- | ----- | ------------------- | ---- | ----------------------- |
| 1           | Alice | alice@example.com   | 28   | 2024-01-15              |
| 2           | Bob   | bob@example.com     | NULL | 2024-01-20              |
| 3           | NULL  | charlie@example.com | 35   | 2024-02-01              |
| 4           | Alice | alice@example.com   | 28   | 2024-01-15 (DUPLICATE!) |
| 5           | David | invalid-email       | -5   | 2024-03-01 (INVALID!)   |
| 6           | Eve   | eve@example.com     | 45   | NULL                    |
| 7           | Frank | frank@example.com   | 32   | 2024-02-15              |

---

### Problema 1: Detectar NULLs

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, isNull, when, count

spark = SparkSession.builder.appName("Data Quality").getOrCreate()

customers = spark.read.parquet("customers.parquet")

# ===== DETECTAR NULLs =====
nulls_report = customers.select(
    count(when(col("customer_id").isNull(), 1)).alias("null_customer_id"),
    count(when(col("name").isNull(), 1)).alias("null_name"),
    count(when(col("email").isNull(), 1)).alias("null_email"),
    count(when(col("age").isNull(), 1)).alias("null_age"),
    count(when(col("signup_date").isNull(), 1)).alias("null_signup_date")
)

nulls_report.show()
```

**Resultado:**

```
null_customer_id=0, null_name=1, null_email=0, null_age=1, null_signup_date=1
```

---

### Problema 2: Manejar NULLs

```python
# ===== OPCIÓN 1: Filtrar (eliminar filas con NULL) =====
cleaned = customers.filter(
    col("name").isNotNull() &
    col("age").isNotNull() &
    col("signup_date").isNotNull()
)

# ===== OPCIÓN 2: COALESCE (reemplazar con default) =====
from pyspark.sql.functions import coalesce

imputed = customers.select(
    col("customer_id"),
    coalesce(col("name"), lit("Unknown")).alias("name"),
    col("email"),
    coalesce(col("age"), lit(0)).alias("age"),
    coalesce(col("signup_date"), lit("1900-01-01")).alias("signup_date")
)

# ===== OPCIÓN 3: Mark como flag =====
flagged = customers.withColumn(
    "data_quality_issues",
    when(col("name").isNull(), "Missing name").otherwise("") +
    when(col("age").isNull(), " Missing age").otherwise("") +
    when(col("signup_date").isNull(), " Missing signup_date").otherwise("")
)

flagged.show()
```

**Resultado:**

```
customer_id | name | age | signup_date | data_quality_issues
1 | Alice | 28 | 2024-01-15 | (empty)
2 | Bob | NULL| 2024-01-20 | Missing age
3 | NULL | 35 | 2024-02-01 | Missing name
```

---

### Problema 3: Detectar Duplicados

```python
# ===== DETECTAR duplicados exactos =====
duplicates = customers.groupBy("customer_id", "name", "email", "age", "signup_date").count().filter(col("count") > 1)

print(f"Exact duplicates: {duplicates.count()}")
duplicates.show()

# ===== DETECTAR duplicados por clave (customer_id) =====
dup_by_key = customers.groupBy("customer_id").count().filter(col("count") > 1)

print(f"Duplicates by customer_id: {dup_by_key.count()}")

# ===== DETECTAR duplicados por email (lógico) =====
dup_by_email = customers.groupBy("email").count().filter(col("count") > 1)

print(f"Duplicates by email: {dup_by_email.count()}")
```

---

### Problema 4: Eliminar Duplicados

```python
from pyspark.sql.functions import row_number
from pyspark.sql import Window

# ===== OPCIÓN 1: Drop exactos (DISTINCT) =====
deduped_exact = customers.drop_duplicates()

# ===== OPCIÓN 2: Drop por clave, mantén primero =====
window_spec = Window.partitionBy("customer_id").orderBy("signup_date")

deduped_key = (
    customers
    .withColumn("rn", row_number().over(window_spec))
    .filter(col("rn") == 1)
    .drop("rn")
)

# ===== OPCIÓN 3: Drop por email, mantén con edad más alta =====
window_email = Window.partitionBy("email").orderBy(col("age").desc())

deduped_email = (
    customers
    .withColumn("rn", row_number().over(window_email))
    .filter(col("rn") == 1)
    .drop("rn")
)

print(f"Original: {customers.count()}")
print(f"After dedup: {deduped_key.count()}")
```

---

### Problema 5: Validar Reglas de Negocio

```python
from pyspark.sql.functions import regexp_replace, length

# ===== VALIDAR y MARCAR problemas =====
validated = customers.withColumn(
    "validation_status",
    when(col("age") < 0 | col("age") > 150, "Invalid: age out of range")
    .when(col("age") < 18, "Warning: underage")
    .when(~col("email").rlike(r'^[\w.-]+@[\w.-]+.\w+$'), "Invalid: email format")
    .when(col("name").rlike(r'^\s*$'), "Invalid: empty name")
    .when(col("signup_date").isNull(), "Invalid: no signup date")
    .otherwise("Valid")
)

# Mostrar problemas
issues = validated.filter(col("validation_status") != "Valid")
issues.select("customer_id", "name", "email", "age", "validation_status").show()

# Contar problemas por tipo
validated.groupBy("validation_status").count().show()
```

**Resultado:**

```
validation_status | count
Valid | 4
Invalid: age out of range | 1
Warning: underage | 0
Invalid: email format | 1
Missing signup date | 1
```

---

### Problema 6: Data Quality Report (Métrica)

```python
from pyspark.sql.functions import sum, avg, max, min

# ===== GENERAR reporte de calidad =====
total_rows = customers.count()

quality_report = customers.select(
    # NULL metrics
    (count(when(col("customer_id").isNull(), 1)) / total_rows * 100).alias("null_customer_id%"),
    (count(when(col("name").isNull(), 1)) / total_rows * 100).alias("null_name%"),
    (count(when(col("email").isNull(), 1)) / total_rows * 100).alias("null_email%"),
    (count(when(col("age").isNull(), 1)) / total_rows * 100).alias("null_age%"),

    # Validation metrics
    (count(when((col("age") < 0) | (col("age") > 150), 1)) / total_rows * 100).alias("invalid_age%"),
    (count(when(~col("email").rlike(r'^[\w\.-]+@[\w\.-]+\.\w+$'), 1)) / total_rows * 100).alias("invalid_email%")
)

quality_report.show()
```

**Resultado:**

```
null_customer_id%=0.0, null_name%=14.3, null_email%=0.0,
null_age%=14.3, invalid_age%=14.3, invalid_email%=14.3
```

---

### Problema 7: Data Quality Monitoring (Production)

```python
from datetime import datetime

# ===== GUARDAR métricas históricas =====
quality_metrics = quality_report.withColumn("check_date", lit(datetime.now()))

# Append a histórico
quality_metrics.write.mode("append").parquet("quality_history/")

# ===== ALERTAR si baja calidad =====
current = quality_metrics.select("null_name%").collect()[0][0]
threshold = 5.0  # Alerta si > 5% nulls

if current > threshold:
    print(f"⚠️ ALERT: null_name % = {current}% (threshold: {threshold}%)")
    send_alert(f"Data quality degradation: {current}%")
else:
    print(f"✓ OK: null_name % = {current}%")
```

---

## Common Quality Patterns

```python
# ===== Pattern 1: Clean + Validate =====
cleaned = (
    customers
    .filter(col("name").isNotNull())  # Remove NULLs
    .filter(col("age") >= 0)  # Validate logic
    .filter(col("email").rlike(r'^[\w.-]+@[\w.-]+.\w+$'))  # Validate format
)

# ===== Pattern 2: Flag + Separate =====
flagged = customers.withColumn(
    "quality_flag",
    when(
        col("name").isNull() |
        (col("age") < 0) |
        ~col("email").rlike(pattern),
        "INVALID"
    ).otherwise("VALID")
)

valid = flagged.filter(col("quality_flag") == "VALID")
invalid = flagged.filter(col("quality_flag") == "INVALID")

valid.write.parquet("valid/")
invalid.write.parquet("quarantine/")

# ===== Pattern 3: Dedup + Quality =====
result = (
    customers
    .drop_duplicates(["customer_id"])
    .filter(col("age") >= 0)
    .filter(col("name").isNotNull())
)
```

---

## Errores comunes en entrevista

- **Error**: No validar data al leer → **Solución**: Valida inmediatamente. Problemas temprano = fácil de fijar

- **Error**: Eliminar todas filas con NULLs (pierde data) → **Solución**: Depende de importancia. A veces coalesce es mejor

- **Error**: No medir calidad (no sabes si mejora/empeora) → **Solución**: Crea métricas. Compara histórico. Alerta si cae

- **Error**: Ignorar duplicados (analytics incorrectos) → **Solución**: Detecta + resuelve. Dedup es parte de pipeline

---

## Preguntas de seguimiento típicas

1. **"¿Cuándo eliminas vs alertas sobre data sucia?"**
   - Elimina: si < 5% (insignificante)
   - Alerta: si > 5% (problema real, investigar)

2. **"¿Cómo debuggeas data quality issues?"**
   - `.show()` primero, explora
   - Agrupa por `validation_status`, cuenta problemas
   - Guarda problemas en `quarantine/` para investigación

3. **"¿Performance de drop_duplicates() vs Window?"**
   - `drop_duplicates()`: Simple, rápido si pocos
   - Window: Más control (ej: qué row mantener), requiere sorting

4. **"¿Cómo monitoreas en producción?"**
   - Calcula métricas cada día
   - Compara vs histórico
   - Alert si % null sube, % invalid sube, etc.

---

## Real-World: Production ETL with Quality Gates

```python
# Lectura
raw = spark.read.parquet("raw_data/")

# Step 1: Validación inmediata
initial_count = raw.count()
print(f"Initial rows: {initial_count}")

# Step 2: Calcular métricas
quality = raw.select(
    count(when(col("id").isNull(), 1)).alias("null_ids"),
    count(when(col("amount") < 0, 1)).alias("negative_amounts")
).collect()

null_pct = quality[0]["null_ids"] / initial_count * 100
negative_pct = quality[0]["negative_amounts"] / initial_count * 100

print(f"Null IDs: {null_pct}%")
print(f"Negative amounts: {negative_pct}%")

# Step 3: Quality gates
if null_pct > 1:
    raise Exception(f"Too many NULL ids: {null_pct}%")
if negative_pct > 0.1:
    raise Exception(f"Too many negative amounts: {negative_pct}%")

# Step 4: Limpieza (si pasó gates)
cleaned = raw.filter(col("id").isNotNull()).filter(col("amount") >= 0)

# Step 5: Escribir
cleaned.write.mode("overwrite").parquet("processed/")

print(f"Final rows: {cleaned.count()} (removed {initial_count - cleaned.count()})")
```

---

## References

- [PySpark SQL Functions - Null Handling](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html#null)
- [Drop Duplicates - PySpark API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/dataframe.html#drop-duplicates)
