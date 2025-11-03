# Transformations & Actions: Lazy Evaluation

**Tags**: #pyspark #transformations #actions #lazy-evaluation #fundamental #real-interview  
**Empresas**: Amazon, Google, Databricks, Uber  
**Dificultad**: Mid  
**Tiempo estimado**: 22 min

---

## TL;DR

**Transformations** = operaciones lazy (no se ejecutan hasta acción). Devuelven nuevo DataFrame/RDD. **Actions** = operaciones eager (ejecutan TODO). Devuelven resultado a driver o escriben. Lazy evaluation es lo que hace Spark eficiente: puede optimizar todo el plan antes de ejecutar.

---

## Concepto Core

- **Qué es**: Lazy evaluation significa Spark no ejecuta transformations hasta que le pidas un resultado (action)
- **Por qué importa**: Spark puede optimizar el plan completo antes de ejecutar. Sin lazy evaluation, sería 100x más lento
- **Principio clave**: Transf. = "qué hacer". Actions = "hazlo ahora". Transformations se stackean, actions las ejecutan todas juntas

---

## Memory Trick

**"Director y actores"**:

- Transformations = Director dice "éstas van a ser las escenas"
- Actions = "¡Rodenlo AHORA!" (se ejecuta todo de verdad)
- Sin actions, no hay película (transformations nunca corren)

---

## Cómo explicarlo en entrevista

**Paso 1**: "Spark es lazy: no ejecuta nada hasta que pides un resultado"

**Paso 2**: "Transformations (map, filter, join) = 'planes'. Actions (collect, write, count) = 'ejecuta'"

**Paso 3**: "Ventaja: Spark ve el plan completo, optimiza, luego ejecuta. Sin lazy, ejecutaría cada transformación por separado"

**Paso 4**: "Eso es por qué Spark es rápido: optimización global, no paso a paso"

---

## Código/Query ejemplo

### Conceptual: Lazy vs Eager

from pyspark.sql import SparkSession
from pyspark.sql.functions import col

spark = SparkSession.builder.appName("Lazy Evaluation").getOrCreate()

Leer datos
df = spark.read.parquet("data.parquet")

===== TRANSFORMATIONS (Lazy) =====
Ninguna de estas ejecuta. Spark solo PLANEA qué hacer
Transf 1: Filter
filtered = df.filter(col("age") > 25)

Transf 2: Select
selected = filtered.select("name", "salary")

Transf 3: GroupBy
grouped = selected.groupBy("department").avg("salary")

En este punto: NADA se ha ejecutado. Spark tiene el plan pero no corre
print("Plan lógico (no ejecutado):")
grouped.explain(mode="simple")

===== ACTION: AHORA se ejecuta TODO =====
result = grouped.collect() # ← AQUÍ Spark ejecuta transformations 1-3

print("Resultado:", result)

text

**Timeline:**
Línea 1-12: Spark construye plan (instantáneo)
Línea 15: explain() muestra plan (no ejecuta)
Línea 18: collect() ejecuta TODO (toma tiempo)

text

---

## Transformations Comunes (Lazy)

===== Transformations que DEVUELVEN DataFrame/RDD =====

1. FILTER: Filas que cumplen condición
   filtered = df.filter(col("salary") > 50000)

2. SELECT: Columnas específicas
   selected = df.select("name", "salary")

3. WITHCOLUMN: Agregar/modificar columna
   with_bonus = df.withColumn("bonus", col("salary") \* 0.1)

4. MAP: Transformación custom (RDD)
   squared_rdd = numbers_rdd.map(lambda x: x \*\* 2)

5. FLATMAP: Map + flatten
   words = text_rdd.flatMap(lambda line: line.split())

6. JOIN: Combina DataFrames
   joined = df1.join(df2, "id")

7. UNION: Combina verticalmente
   combined = df1.union(df2)

8. GROUPBY: Agrupa (devuelve GroupedData, aún lazy)
   grouped = df.groupBy("department")

9. SORT: Ordena
   sorted_df = df.sort(col("salary").desc())

10. DISTINCT: Únicos
    unique = df.select("department").distinct()

11. DROP: Elimina columna
    dropped = df.drop("unnecessary_column")

NINGUNA DE ESTAS EJECUTÓ NADA. Son planes.
text

---

## Actions Comunes (Eager)

===== ACTIONS: Estos sí ejecutan TODO =====

1. COLLECT: Trae TODO a driver (⚠️ cuidado con datasets grandes)
   all_data = df.collect()

2. TAKE: Primeras N filas
   first_10 = df.take(10)

3. FIRST: Primera fila
   first_row = df.first()

4. COUNT: Número de filas
   row_count = df.count()

5. SHOW: Muestra primeras 20 filas en consola
   df.show()

6. WRITE: Guarda a storage (S3, HDFS, Parquet)
   df.write.parquet("output/result.parquet")

7. FOREACH: Aplica función a cada fila
   def process_row(row):
   print(row)
   df.foreach(process_row)

8. REDUCE: Reduce RDD a 1 valor (RDD only)
   sum_rdd = numbers_rdd.reduce(lambda a, b: a + b)

9. SAVEASTEXTFILE: Guarda como texto (RDD)
   rdd.saveAsTextFile("output/")

TODAS ESTAS EJECUTAN. Buscan el plan, lo optimizan, lo corren.
text

---

## Lazy Evaluation en Acción

### Escenario: Pipeline de Ventas

Datos: millones de transacciones
sales = spark.read.parquet("sales.parquet")

Plan (no ejecuta):
monthly_report = (
sales
.filter(col("status") == "completed") # Transf 1
.filter(col("amount") > 0) # Transf 2
.withColumn("month", to_date(col("date"))) # Transf 3
.groupBy("month", "product") # Transf 4
.agg(sum("amount").alias("revenue")) # Transf 5
.sort(col("revenue").desc()) # Transf 6
)

print("Plan construido (instantáneo)")
print(monthly_report.explain(mode="simple"))

===== PRIMER ACTION =====
print("COLLECT (action 1): Trae TODO a driver")
result = monthly_report.collect()

Aquí: Spark ejecuta transf 1-6 JUNTAS (no uno por uno)
Catalyst optimizó: predicate pushdown, column pruning, etc.
===== SEGUNDO ACTION (diferente) =====
print("WRITE (action 2): Guarda a Parquet")
monthly_report.write.parquet("output/sales_report")

Aquí: Spark ejecuta transf 1-6 de nuevo (cada action es ejecución nueva)
PERO el plan se re-optimiza
text

⚠️ **Importante**: Cada action re-ejecuta transformations anteriores. Si usas resultado múltiples veces, **cachea**:

Cachear antes de múltiples actions
monthly_report.cache()

result1 = monthly_report.collect() # Ejecuta, cachea
result2 = monthly_report.count() # Usa cache (rápido)
result3 = monthly_report.write.parquet(...) # Usa cache (rápido)

Sin cache, ejecutaría 3 veces la pipeline
text

---

## Cadena de Transformations (DAG)

Spark crea un DAG (Directed Acyclic Graph):

Tu código:
result = (
df.filter(col("age") > 25)
.select("name", "salary")
.groupBy("department")
.avg("salary")
)

Spark internamente:
Read Parquet
↓
Filter (age > 25) ← Predicate pushdown aquí (Catalyst)
↓
Select (name, salary) ← Column pruning aquí
↓
GroupBy + Agg
↓
Output
Plan COMPLETO se optimiza antes de ejecutar
text

**Sin lazy evaluation (❌ ineficiente):**
Read → (ejecuta) → Filter → (ejecuta) → Select → (ejecuta) → GroupBy → (ejecuta)

text

**Con lazy evaluation (✅ eficiente):**
Read + Filter + Select + GroupBy → (optimiza TODO junto) → (ejecuta una sola vez)

text

---

## Errores comunes en entrevista

- **Error**: Pensar que `.filter()` ejecuta inmediatamente → **Solución**: Es lazy. Espera action

- **Error**: Usar `collect()` en DataFrame gigante → **Solución**: Causa out-of-memory. Usa `.write()` directo

- **Error**: No cachear cuando usas resultado múltiples veces → **Solución**: `.cache()` después del transformation costoso

- **Error**: No entender por qué 2 actions son lentos (repiten trabajo) → **Solución**: Cada action = nueva ejecución. Si reutilizas, cachea

---

## Preguntas de seguimiento típicas

1. **"¿Qué es un DAG?"**
   - Directed Acyclic Graph: representación del plan de ejecución
   - Spark lo usa para optimizar (Catalyst)
   - Puedes verlo con `.explain()`

2. **"¿Por qué `collect()` es peligroso?"**
   - Trae TODO a driver memory (un servidor)
   - Si DataFrame = 100GB, crash
   - Mejor: `.write()` directo a storage

3. **"¿Diferencia entre `cache()` y `persist()`?"**
   - `cache()` = guardar en memoria (por defecto)
   - `persist()` = guardar en memoria/disk/hybrid (configurable)
   - `persist(StorageLevel.MEMORY_AND_DISK)` es safe para datos grandes

4. **"¿Cómo debuggear plan lento?"**
   - `.explain(mode="extended")` muestra optimizaciones
   - Busca "Shuffle" (costoso)
   - Busca "Cartesian Product" (muy costoso)
   - Optimiza: partitioning, predicate pushdown, joins

---

## Patrón Common: Pipeline Real

Lectura
orders = spark.read.parquet("orders.parquet")
customers = spark.read.parquet("customers.parquet")

Transformations (planea, no ejecuta)
pipeline = (
orders
.filter(col("year") == 2024) # Predicate pushdown
.join(customers, "customer_id") # Broadcast si customers es pequeño
.select("name", "order_amount", "category")
.groupBy("category")
.agg(sum("order_amount").alias("total"))
)

Cachea si lo usarás múltiples veces
pipeline.cache()

Action 1
pipeline.show()

Action 2
pipeline.write.parquet("output/summary")

Action 3
count = pipeline.count()

Sin cache, ejecutaría 3 veces. Con cache, 1 ejecución + 2 lecturas de cache.
text

---

## Referencias

- [Lazy Evaluation - Spark Docs](https://spark.apache.org/docs/latest/rdd-programming-guide.html#lazy-evaluation)
- [Transformations & Actions - PySpark](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/dataframe.html)
- [Explain Plans - Databricks](https://docs.databricks.com/en/spark/latest/spark-sql/query-optimizer.html)
