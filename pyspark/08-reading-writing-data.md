# Lectura y Escritura de Datos en PySpark

**Tags**: #pyspark #io #formats #parquet #csv #json #jdbc #real-interview

---

## TL;DR

Leer datos: `.read.format("fmt").load()` o `.read.csv/parquet/json()`. Escribir: `.write.format("fmt").mode().save()`. Formatos comunes: **Parquet** (default, comprimido), **CSV** (legible pero lento), **JSON** (flexible), **JDBC** (databases). Modos write: `overwrite`, `append`, `ignore`, `error`. Para performance: usa Parquet, particiona datos, comprime.

---

## Concepto Core

- **Qué es**: I/O en Spark = leer datos desde storage (S3, HDFS, DB) y escribir resultados. Cada formato tiene trade-offs
- **Por qué importa**: 80% del tiempo en data pipelines es I/O. Elegir mal = pipeline lento. Demuestra comprensión práctica
- **Principio clave**: Parquet > CSV/JSON siempre (comprimido, columnar, rápido). JSON/CSV solo para legibilidad

---

## Memory Trick

**"Formato de caja"** — Parquet = caja comprimida (rápida, pequeña). CSV = sobre de papel (legible, grande, lento). JDBC = base de datos (transaccional, lento para analytics)

---

## Cómo explicarlo en entrevista

**Paso 1**: "Data I/O es bottleneck en pipelines. Elige formato basado en use case"

**Paso 2**: "Parquet: comprimido, columnar, rápido. CSV/JSON: legible pero lento. JDBC: databases"

**Paso 3**: "Para lectura: `.read.parquet()` o `.read.format().load()`. Para escritura: `.write.mode().parquet()`"

**Paso 4**: "Modos: `overwrite` (replace), `append` (add), `ignore` (skip si existe), `error` (fail si existe)"

---

## Código/Query ejemplo

### Lectura: Múltiples Formatos

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("I/O Example").getOrCreate()

# ===== LECTURA: Parquet =====
df_parquet = spark.read.parquet("s3://bucket/data.parquet")

# ===== LECTURA: CSV =====
df_csv = (
    spark.read
    .option("header", "true")  # Primera fila = nombres columnas
    .option("inferSchema", "true")  # Inferir tipos (lento pero útil exploración)
    .csv("s3://bucket/data.csv")
)

# ===== LECTURA: CSV con Schema Explícito (Recomendado) =====
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("id", IntegerType(), False),
    StructField("name", StringType(), False),
    StructField("amount", IntegerType(), True)
])

df_csv_explicit = (
    spark.read
    .schema(schema)
    .option("header", "true")
    .csv("s3://bucket/data.csv")
)

# ===== LECTURA: JSON =====
df_json = spark.read.json("s3://bucket/data.json")

# ===== LECTURA: JDBC (Database) =====
df_jdbc = (
    spark.read
    .format("jdbc")
    .option("url", "jdbc:postgresql://host:5432/database")
    .option("dbtable", "public.customers")
    .option("user", "username")
    .option("password", "password")
    .load()
)

# ===== LECTURA: Multiple Files (Wildcards) =====
df_multi = spark.read.parquet("s3://bucket/data-2024-*.parquet")
```

---

### Escritura: Modos y Particionamiento

```python
# ===== ESCRITURA: Parquet (Recomendado) =====
# Modo: OVERWRITE (replace everything)
df.write.mode("overwrite").parquet("s3://bucket/output/data.parquet")

# Modo: APPEND (agregar a existentes)
df.write.mode("append").parquet("s3://bucket/output/data.parquet")

# Modo: IGNORE (skip si existe)
df.write.mode("ignore").parquet("s3://bucket/output/data.parquet")

# Modo: ERROR (fail si existe, default)
df.write.mode("error").parquet("s3://bucket/output/data.parquet")

# ===== ESCRITURA: Particionada (Importante!) =====
# Particiona por fecha para queries rápidas
df.write
    .mode("overwrite")
    .partitionBy("year", "month", "day")
    .parquet("s3://bucket/output/partitioned/")

# Resulta en:
# s3://bucket/output/partitioned/year=2024/month=01/day=15/part-xxx.parquet
# s3://bucket/output/partitioned/year=2024/month=01/day=16/part-xxx.parquet

# ===== ESCRITURA: CSV =====
df.write
    .mode("overwrite")
    .option("header", "true")
    .csv("s3://bucket/output/data.csv")

# ===== ESCRITURA: JSON =====
df.write.mode("overwrite").json("s3://bucket/output/data.json")

# ===== ESCRITURA: JDBC (Database) =====
df.write
    .format("jdbc")
    .option("url", "jdbc:postgresql://host:5432/database")
    .option("dbtable", "public.results")
    .option("user", "username")
    .option("password", "password")
    .mode("overwrite")
    .save()
```

---

## Formatos: Comparación

| Formato     | Compresión       | Velocidad  | Legible | Casos Uso                      |
| ----------- | ---------------- | ---------- | ------- | ------------------------------ |
| **Parquet** | Sí (Snappy/Gzip) | Rápido     | No      | Data lakes, analytics, default |
| **CSV**     | No               | Lento      | Sí      | Legibilidad, Excel, exports    |
| **JSON**    | No               | Lento      | Sí      | APIs, flexible schema          |
| **JDBC**    | N/A              | Lento      | N/A     | Databases, transaccional       |
| **ORC**     | Sí               | Muy rápido | No      | Hive, Presto (legacy)          |
| **Avro**    | Sí               | Rápido     | No      | Kafka, event streaming         |

---

## Parquet: Deep Dive

```python
# ===== PARQUET: Opciones Comunes =====
# Comprensión: Snappy (fast) vs Gzip (compact)
df.write
    .option("compression", "snappy")
    .parquet("output/")

# Modo compresión: Control de número de files
df.coalesce(10).write
    .mode("overwrite")
    .parquet("output/")  # Genera 10 files en lugar de muchos

# Particionamiento inteligente
df.write
    .mode("overwrite")
    .partitionBy("date")
    .bucketBy(10, "user_id")
    .parquet("output/")

# Resulta en:
# output/date=2024-01-15/user_id=0/part-xxx.parquet
# output/date=2024-01-15/user_id=1/part-xxx.parquet
# ... (10 buckets por fecha)
```

---

## CSV: Opciones Comunes

```python
# ===== CSV: Lectura =====
df = (
    spark.read
    .option("header", "true")  # Primera fila = columnas
    .option("sep", ",")  # Separador (defecto: coma)
    .option("inferSchema", "true")  # Inferir tipos (lento)
    .option("nullValue", "")  # Tratar "" como NULL
    .option("mode", "PERMISSIVE")  # o FAILFAST
    .csv("data.csv")
)

# ===== CSV: Escritura =====
df.write
    .option("header", "true")
    .option("sep", ",")
    .mode("overwrite")
    .csv("output/")

# Resulta en multiple CSV files (no 1 solo)
```

---

## JSON: Opciones

```python
# ===== JSON: Lectura (Una línea = un JSON object) =====
df = spark.read.json("data.json")  # newline-delimited JSON

# ===== JSON: Escritura =====
df.write.mode("overwrite").json("output/")  # Newline-delimited
```

---

## JDBC: Database Connections

```python
# ===== JDBC: Lectura =====
df = (
    spark.read
    .format("jdbc")
    .option("url", "jdbc:postgresql://localhost:5432/mydb")
    .option("dbtable", "public.users")  # o query completa
    .option("user", "postgres")
    .option("password", "password")
    .option("numPartitions", 4)  # Parallelismo al leer
    .load()
)

# ===== JDBC: Lectura con Query Custom =====
df = (
    spark.read
    .format("jdbc")
    .option("url", "jdbc:postgresql://localhost:5432/mydb")
    .option("query", "SELECT * FROM users WHERE age > 25")
    .option("user", "postgres")
    .option("password", "password")
    .load()
)

# ===== JDBC: Escritura =====
df.write
    .format("jdbc")
    .option("url", "jdbc:postgresql://localhost:5432/mydb")
    .option("dbtable", "public.results")
    .option("user", "postgres")
    .option("password", "password")
    .mode("overwrite")
    .save()
```

---

## Performance: Lectura

```python
# ❌ LENTO: Sin schema, CSV con inferencia
df = spark.read.option("inferSchema", "true").csv("large_file.csv")
# 2 scans: inferir schema + leer datos

# ✅ RÁPIDO: Schema explícito, Parquet
df = spark.read.schema(schema).parquet("large_file.parquet")
# 1 scan, comprimido

# ❌ LENTO: Leer archivo gigante completo, luego filtrar
df = spark.read.csv("huge_file.csv").filter(col("year") == 2024)

# ✅ RÁPIDO: Particionar, Spark poda particiones
df = spark.read.parquet("partitioned/year=2024/")
# Spark lee solo 2024, ignora otros años
```

---

## Performance: Escritura

```python
# ❌ LENTO: Muchos archivos pequeños (default)
df.write.mode("overwrite").csv("output/")
# Genera 1000s de small files

# ✅ RÁPIDO: Limitar number of partitions
df.coalesce(10).write.mode("overwrite").csv("output/")
# Genera 10 archivos, más eficiente

# ✅ RÁPIDO: Particionar + Parquet
df.write
    .mode("overwrite")
    .partitionBy("year", "month")
    .parquet("output/")
# Crea estructura: output/year=2024/month=01/... (queryable)
```

---

## Errores comunes en entrevista

- **Error**: Usar CSV para big data → **Solución**: CSV es lento, no comprimido. Usa Parquet

- **Error**: No especificar schema en CSV → **Solución**: Inferencia = 2 scans + puede ser incorrecto. Siempre schema explícito

- **Error**: No particionar al escribir → **Solución**: Datos sin partición = full scan después. Particiona por columnas de filtro común

- **Error**: Usar modo "error" (default) sin manejar overwrite → **Solución**: Usa `mode("overwrite")` para actualizar, o `mode("append")` para agregar

---

## Preguntas de seguimiento típicas

1. **"¿Cuándo usarías Parquet vs ORC?"**
   - Parquet: Default, comprimido, columnar, rápido
   - ORC: Legacy (Hive), muy comprimido

2. **"¿Cómo manejas archivos pequeños (small files problem)?"**
   - Lectura: No hay problema, Spark combina automáticamente
   - Escritura: `coalesce(n)` antes de write para reducir archivos

3. **"¿JDBC es más rápido que Parquet?"**
   - No. JDBC = transaccional, lento para analytics. Parquet = rápido, columnar

4. **"¿Cómo incrementalmente escribes datos (append)?"**
   - `.write.mode("append")` (añade a existentes)
   - O: `.write.insertInto(table)` (SQL insert)

---

## Real-World: Daily Data Pipeline

```python
from pyspark.sql import SparkSession
from datetime import datetime

spark = SparkSession.builder.appName("Daily Pipeline").getOrCreate()

# Leer datos del día actual
today = datetime.now().strftime("%Y-%m-%d")
raw = spark.read.parquet(f"s3://raw/events/{today}/")

# Transformar
processed = raw.filter(col("valid") == True)

# Escribir con particiones
processed.write
    .mode("overwrite")
    .partitionBy("year", "month", "day")
    .parquet("s3://processed/events/")

# Resultado:
# s3://processed/events/year=2024/month=01/day=15/part-xxx.parquet

# Luego: queries son fast (partition pruning)
result = spark.read.parquet("s3://processed/events/year=2024/month=01/")
# Solo lee enero 2024, ignora otros meses
```

---

## References

- [DataFrameReader - PySpark](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/io.html)
- [Parquet Format - Apache Parquet](https://parquet.apache.org/)
- [JDBC Data Source - Spark Docs](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)
