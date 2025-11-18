# Spark SQL: Queries SQL en PySpark

**Tags**: #pyspark #spark-sql #sql #real-interview

---

## TL;DR

**Spark SQL** = escribir SQL directamente en Spark. Registra DataFrame como tabla temporal con `.createOrReplaceTempView()`, luego usa `spark.sql("SELECT ...")`. Spark SQL y DataFrame API usan mismo optimizer (Catalyst). Elige SQL si SQL devs, elige API si Python devs. Performance igual.

---

## Concepto Core

- **Qué es**: Spark SQL permite escribir SQL en lugar de DataFrame API. Ambos se compilan al mismo plan de ejecución (Catalyst)
- **Por qué importa**: SQL devs pueden contribuir sin aprender Spark API. Algunos queries son más fáciles en SQL (joins complejos, window functions)
- **Principio clave**: SQL y API son equivalentes. Elige por preferencia/legibilidad, no performance

---

## Memory Trick

**"Dos idiomas, un motor"** — SQL y DataFrame API hablan diferente pero usan mismo motor (Catalyst). El result es idéntico.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Spark SQL permite escribir SQL en lugar de DataFrame transformations"

**Paso 2**: "Registra DataFrame como tabla temporal: `.createOrReplaceTempView()`"

**Paso 3**: "Luego `spark.sql('SELECT ...')` como si fuera base de datos SQL"

**Paso 4**: "Performance es igual a DataFrame API porque Catalyst optimiza ambos"

---

## Código/Query ejemplo

### DataFrame API vs Spark SQL

**Ambos hacen lo mismo:**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, avg

spark = SparkSession.builder.appName("Spark SQL Example").getOrCreate()

orders = spark.read.parquet("orders.parquet")
customers = spark.read.parquet("customers.parquet")

# ===== OPCIÓN 1: DataFrame API =====
result_api = (
    orders
    .join(customers, "customer_id")
    .filter(col("order_amount") > 100)
    .groupBy("customer_id", "name")
    .agg(
        sum("order_amount").alias("total_spent"),
        avg("order_amount").alias("avg_order")
    )
    .orderBy(col("total_spent").desc())
)

result_api.show()

# ===== OPCIÓN 2: Spark SQL =====
# Primero: registra DataFrames como tablas temporales
orders.createOrReplaceTempView("orders_temp")
customers.createOrReplaceTempView("customers_temp")

# Luego: SQL query
result_sql = spark.sql("""
SELECT
    o.customer_id,
    c.name,
    SUM(o.order_amount) as total_spent,
    AVG(o.order_amount) as avg_order
FROM orders_temp o
JOIN customers_temp c ON o.customer_id = c.customer_id
WHERE o.order_amount > 100
GROUP BY o.customer_id, c.name
ORDER BY total_spent DESC
""")

result_sql.show()

# Performance: IDÉNTICO (mismo optimizer)
# Elige por legibilidad/preferencia
```

---

## Registrar Tablas Temporales

### Temporary Views (sesión actual)

```python
# Scope: Solo esta sesión
df.createOrReplaceTempView("my_table")

# SQL access
result = spark.sql("SELECT * FROM my_table")
```

---

### Global Temporary Views (todos drivers)

```python
# Scope: Todos drivers conectados a cluster
df.createGlobalTempView("my_global_table")

# SQL access (prefix global_temp)
result = spark.sql("SELECT * FROM global_temp.my_global_table")
```

---

### Permanent Tables (persisten)

```python
# Scope: Persistente en metastore (incluso después de Spark stop)
df.write.mode("overwrite").saveAsTable("my_permanent_table")

# SQL access
result = spark.sql("SELECT * FROM my_permanent_table")

# Permanece después de:
spark.stop()

# ... restart Spark
spark.sql("SELECT * FROM my_permanent_table")  # Aún existe!
```

---

## Spark SQL vs DataFrame: Cuándo Cada Uno

### ✅ Usa Spark SQL si:

- SQL devs en tu team
- Query es complejo (múltiples joins, CTEs)
- Ya tienes SQL escrita (legacy)
- Te sientes más cómodo con SQL

```python
# SQL: CTEs, window functions, complejas joins
spark.sql("""
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) as rank
    FROM products
)
SELECT * FROM ranked WHERE rank <= 3
""")
```

---

### ✅ Usa DataFrame API si:

- Python devs puro
- Lógica con UDFs/custom functions
- Necesitas debuggear paso a paso
- Data engineering workflow (transformaciones complejas)

```python
# API: UDFs, custom logic, debuggeable
from pyspark.sql.functions import row_number, col, window

products = spark.read.parquet("products/")
window_spec = window.partitionBy("category").orderBy(col("sales").desc())

result = (
    products
    .withColumn("rank", row_number().over(window_spec))
    .filter(col("rank") <= 3)
)
```

---

## Complex Examples: SQL vs API

### Ejemplo 1: Window Functions + CTEs

**SQL (más legible):**

```sql
WITH ranked_sales AS (
    SELECT
        product_id,
        category,
        sales,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) as rank
    FROM products
)
SELECT * FROM ranked_sales WHERE rank <= 3
```

**API (más verboso):**

```python
from pyspark.sql.functions import row_number, col, window

window_spec = window.partitionBy("category").orderBy(col("sales").desc())
result = (
    products
    .withColumn("rank", row_number().over(window_spec))
    .filter(col("rank") <= 3)
    .select("product_id", "category", "sales", "rank")
)
```

---

### Ejemplo 2: Multiple Joins

**SQL (más claro):**

```sql
SELECT
    o.order_id,
    c.customer_name,
    p.product_name,
    s.supplier_name,
    o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN products p ON o.product_id = p.product_id
JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE o.date >= '2024-01-01'
```

**API (más pasos):**

```python
from pyspark.sql.functions import col

result = (
    orders
    .join(customers, "customer_id")
    .join(products, "product_id")
    .join(suppliers, "supplier_id")
    .filter(col("date") >= "2024-01-01")
    .select("order_id", "customer_name", "product_name", "supplier_name", "amount")
)
```

---

## Explain Plans: Validar Optimización

```python
# Spark SQL
spark.sql("""
    SELECT * FROM orders WHERE amount > 100
""").explain(mode="extended")

# DataFrame API (same result)
orders.filter(col("amount") > 100).explain(mode="extended")

# Output (identical):
# == Optimized Logical Plan ==
# Filter (amount#1 > 100)
# +- Relation[id#0, amount#1, ...]
# == Physical Plan ==
# Filter (amount#1 > 100)
# +- FileScan parquet [id#0, amount#1, ...]
```

Catalyst optimiza ambos idénticamente.

---

## Interoperabilidad: SQL ↔ DataFrame

```python
# Partir de SQL, retornar DataFrame
sql_result = spark.sql("SELECT * FROM orders WHERE amount > 100")

# Aplicar API transformations
final = sql_result.withColumn("tax", col("amount") * 0.1)

# Back to SQL
final.createOrReplaceTempView("orders_with_tax")
spark.sql("SELECT * FROM orders_with_tax")

# Mezclar libremente
```

---

## Errores comunes en entrevista

- **Error**: Pensar que SQL es más lento que API → **Solución**: Performance es idéntico (Catalyst optimiza ambos)

- **Error**: Usar `createTempView()` sin verificar existencia → **Solución**: Usa `createOrReplaceTempView()` para evitar errors

- **Error**: No limpiar tablas temporales → **Solución**: Se limpian automáticamente al cerrar sesión, pero buena práctica: `spark.sql("DROP TABLE IF EXISTS my_table")`

- **Error**: Mezclar temporary + permanent views sin entender scope → **Solución**: Temporary = sesión actual. Permanent = persistente. Elige según necesidad

---

## Preguntas de seguimiento típicas

1. **"¿Cuándo usarías Spark SQL vs DataFrame API?"**
   - SQL: SQL devs, queries complejas, legibilidad
   - API: Python devs, custom logic, debuggeable

2. **"¿Diferencia entre createTempView y createOrReplaceTempView?"**
   - `createTempView`: Error si tabla existe
   - `createOrReplaceTempView`: Reemplaza silenciosamente (safe)

3. **"¿Performance de Spark SQL vs DataFrame API?"**
   - Idéntico. Catalyst optimiza ambos

4. **"¿Cómo pasas resultados entre SQL y API?"**
   - SQL → DataFrame: `spark.sql(...)`
   - DataFrame → SQL: `.createOrReplaceTempView()`
   - Mezcla libremente

---

## Real-World: ETL Pipeline Mixta

```python
# Leer datos
raw_events = spark.read.parquet("raw_events/")

# Limpieza: API (más control)
cleaned = raw_events.filter(col("timestamp").isNotNull())

# Agregación: SQL (más legible)
cleaned.createOrReplaceTempView("cleaned_events")

result = spark.sql("""
    SELECT
        DATE(timestamp) as event_date,
        event_type,
        COUNT(*) as count,
        COUNT(DISTINCT user_id) as unique_users
    FROM cleaned_events
    WHERE event_type IN ('purchase', 'view')
    GROUP BY DATE(timestamp), event_type
    ORDER BY event_date DESC, count DESC
""")

# Agregación adicional: API
final = result.withColumn("daily_total",
    sum("count").over(Window.partitionBy("event_date")))

# Guardar
final.write.parquet("output/daily_summary")
```

---

## References

- [Spark SQL - PySpark Docs](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/index.html)
- [Temporary Views - Spark Docs](https://spark.apache.org/docs/latest/sql-data-sources.html#temporary-views)
- [Catalyst Optimizer - Spark Architecture](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
