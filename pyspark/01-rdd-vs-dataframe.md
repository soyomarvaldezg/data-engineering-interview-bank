# RDD vs DataFrame: Cuándo Usar Cada Uno

**Tags**: #pyspark #rdd #dataframe #architecture #fundamental #real-interview

---

## TL;DR

**RDD** (Resilient Distributed Dataset) = bajo nivel, unstructured, flexible. **DataFrame** = alto nivel, structured (SQL), optimizado. En 99% de casos: **usa DataFrame**. RDD solo si datos son realmente unstructured o necesitas transformaciones muy específicas.

---

## Concepto Core

- **Qué es**: RDD es la abstracción más baja en Spark (colecciones distribuidas). DataFrame es RDD con schema (estructura conocida)
- **Por qué importa**: Elegir mal entre RDD y DataFrame = performance disaster. Demuestra comprensión arquitectónico
- **Principio clave**: DataFrame > RDD siempre (a menos que tengas buena razón). Spark Catalyst optimizer trabaja con DataFrames

---

## Memory Trick

**"Escalera de abstracción"**:

- RDD = Lego blocks sueltos (máxima flexibilidad, máximo trabajo manual)
- DataFrame = Casas pre-construidas (menos flexible, súper rápido)
- Spark SQL = Ciudades enteras (SQL puro, máxima optimización)

---

## Cómo explicarlo en entrevista

**Paso 1**: "RDD es la abstracción más baja. Cada elemento es un objeto Python/Java, zero schema"

**Paso 2**: "DataFrame es RDD estructurado. Tiene columnas nombradas, tipos conocidos, schema"

**Paso 3**: "DataFrame usa Catalyst optimizer (Spark entiende qué haces y optimiza). RDD no"

**Paso 4**: "Conclusión: Usa DataFrame 99% del tiempo. RDD solo si datos son realmente unstructured"

---

## Código/Query ejemplo

### Escenario: Leer y procesar logs

**Logs sin estructura:**
2024-01-15 10:23:45 ERROR User 123 failed login attempt
2024-01-15 10:24:12 INFO User 456 logged in successfully
2024-01-15 10:25:03 WARNING High memory usage detected

---

### ❌ Opción 1: RDD (Flexible pero lento)

```python
from pyspark import SparkContext

sc = SparkContext("local", "RDD Example")

# Leer como RDD (cada línea es un string)
logs_rdd = sc.textFile("logs.txt")

# Transformación 1: Parse log line
def parse_log(line):
    parts = line.split()
    return {
        'timestamp': f"{parts[0]} {parts[1]}",
        'level': parts[2],
        'message': ' '.join(parts[3:])
    }

parsed_rdd = logs_rdd.map(parse_log)

# Transformación 2: Filter errores
errors_rdd = parsed_rdd.filter(lambda log: log['level'] == 'ERROR')

# Acción: Collect
errors_list = errors_rdd.collect()
print(errors_list)
```

**Problemas**:

- Cada `map()` = conversión Python → objeto genérico → Python (overhead)
- Spark NO SABE que hay columnas "level", "message"
- NO hay optimización automática
- `collect()` retorna a driver (puede ser huge)

---

### ✅ Opción 2: DataFrame (Recomendado)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, split, concat_ws
from pyspark.sql.types import StructType, StructField, StringType

spark = SparkSession.builder.appName("DataFrame Example").getOrCreate()

# Define schema explícito
schema = StructType([
    StructField("timestamp", StringType(), True),
    StructField("level", StringType(), True),
    StructField("message", StringType(), True)
])

# Leer y parsear con schema
logs_df = (
    spark.read.text("logs.txt")
    .select(
        split(col("value"), r"\s+").getItem(0).alias("date"),
        split(col("value"), r"\s+").getItem(1).alias("time"),
        split(col("value"), r"\s+").getItem(2).alias("level"),
        concat_ws(" ", split(col("value"), r"\s+").getItem(3),
                  split(col("value"), r"\s+").getItem(4),
                  split(col("value"), r"\s+").getItem(5),
                  split(col("value"), r"\s+").getItem(6)).alias("message")
    )
    .withColumn("timestamp", concat_ws(" ", col("date"), col("time")))
    .select("timestamp", "level", "message")
)

# Filter: Spark optimiza automáticamente
errors_df = logs_df.filter(col("level") == "ERROR")

# Show (no colapsas a driver como collect)
errors_df.show()
```

**Ventajas**:

- Spark SABE qué columnas tienes
- Catalyst optimizer puede predicate pushdown
- `.show()` es lazy (no trae TODO a driver)
- Interoperable con Spark SQL

---

### Comparación: RDD vs DataFrame (Mismo resultado, diferente performance)

❌ RDD: 1000 segundos (sin optimización)

```python
errors_count_rdd = parsed_rdd.filter(
    lambda log: log['level'] == 'ERROR'
).count()
```

✅ DataFrame: 50 segundos (con optimizer)

```python
errors_count_df = logs_df.filter(
    col("level") == "ERROR"
).count()
```

Catalyst optimizer hizo 20x más rápido

---

## Diferencias Clave: RDD vs DataFrame

| Aspecto               | RDD          | DataFrame          | Ganador   |
| --------------------- | ------------ | ------------------ | --------- |
| **Schema**            | No           | Sí                 | DataFrame |
| **Performance**       | Lento        | Rápido (optimizer) | DataFrame |
| **SQL**               | No           | Sí                 | DataFrame |
| **Type Safety**       | Débil        | Fuerte             | DataFrame |
| **Flexibilidad**      | Máxima       | Media              | RDD       |
| **Casos de Uso**      | Unstructured | Structured         | Depende   |
| **Curva aprendizaje** | Difícil      | Fácil              | DataFrame |

---

## ¿Cuándo Usar RDD?

**Usa RDD solo si:**

1. **Datos realmente unstructured** (binary, irregular)
   Ejemplo: Imágenes, audio, binarios

   ```python
   images_rdd = sc.binaryFiles("images/").map(lambda x: x)
   ```

2. **Necesitas transformaciones muy específicas**
   Ejemplo: Custom encoding/decoding

   ```python
   encoded_rdd = data_rdd.map(custom_encryption_function)
   ```

3. **Optimización extrema de memoria** (rare)
   Ejemplo: Comprimir antes de serializar
   ```python
   compressed_rdd = data_rdd.map(lambda x: zlib.compress(str(x)))
   ```

**En 99% de casos: Esto NO aplica.** Usa DataFrame.

---

## ¿Cuándo Usar DataFrame?

**Usa DataFrame SIEMPRE que:**

- ✅ Datos tienen estructura (CSV, Parquet, JSON, SQL)
- ✅ Necesitas filtrado/agregación
- ✅ Quieres usar Spark SQL
- ✅ Performance importa
- ✅ Team necesita mantenibilidad

**Resumen: Casi siempre usa DataFrame.**

---

## Performance: Catalyst Optimizer

Cuando usas DataFrame, Spark ejecuta estas optimizaciones automáticamente:

Tu código:

```python
df.filter(col("age") > 25).select("name", "salary").groupBy("department").avg("salary")
```

Lo que Spark hace internamente (Catalyst):

- Predicate Pushdown: Filtra ANTES de select (menos datos)
- Column Pruning: Solo lee columnas necesarias
- Constant Folding: Pre-calcula constantes
- Join Reordering: Ordena joins para minimizar shuffle
- Result en Parquet: Serializa eficientemente

**RDD hace NINGUNA de estas optimizaciones.**

---

## Hybrid: Conversiones RDD ↔ DataFrame

A veces tienes RDD y necesitas DataFrame (o viceversa):

```python
# RDD → DataFrame
rdd = sc.parallelize([("Alice", 28), ("Bob", 35)])
df = rdd.toDF(["name", "age"])

# DataFrame → RDD (perdes schema, metadata)
df_back_rdd = df.rdd  # df.rdd es RDD[Row]

# Útil para: operaciones específicas que necesitan RDD flexibility
processed_rdd = df.rdd.map(lambda row: (row.name.upper(), row.age * 2))
result_df = processed_rdd.toDF(["name_upper", "age_doubled"])
```

---

## Errores comunes en entrevista

- **Error**: "RDD es siempre mejor porque es más flexible" → **Solución**: Flexibilidad ≠ velocidad. DataFrame gana en performance

- **Error**: "Usa RDD porque entiendo mejor la lógica" → **Solución**: DataFrame es más intuitivo. Si RDD te parece fácil, no entiendes Spark

- **Error**: No pensar en schema → **Solución**: Schema explícito (no inferencia) es production-ready

- **Error**: Usar `collect()` en RDD grande → **Solución**: Causa out-of-memory. Usa `.take()` o `.first()`

---

## Preguntas de seguimiento típicas

1. **"¿Qué es Catalyst Optimizer?"**
   - Componente de Spark que optimiza queries DataFrame automáticamente
   - Analiza lógica, reordena operaciones, predicate pushdown
   - RDD bypassa completamente esto

2. **"¿Cómo eliges schema: explícito o inferencia?"**
   - Explícito: Mejor para production, conoces tipos
   - Inferencia: Rápido para exploración, más lento (2 scans)

3. **"¿Spark SQL vs DataFrame API?"**
   - SQL: Más legible para SQL devs
   - API: Más flexible, se integra con código Python
   - Mismo motor abajo (Catalyst optimiza ambos)

4. **"¿Cuándo RDD es más rápido que DataFrame?"**
   - Casi nunca. DataFrame siempre gana
   - Excepción: si datos son tiny (< 1GB), overhead es negligible

---

## Real-World: Decisión en Proyecto Actual

**Escenario: Pipeline de E-Commerce**

Datos: clicks de usuario, compras, reviews
Estructura: CSV con columnas conocidas
✅ CORRECTO: DataFrame

```python
events_df = spark.read.schema(my_schema).csv("events.csv")
purchase_df = events_df.filter(col("event_type") == "purchase")
by_user = purchase_df.groupBy("user_id").agg(sum("amount"))
by_user.write.parquet("output/")
```

❌ INCORRECTO: RDD

```python
events_rdd = sc.textFile("events.csv").map(parse_csv)
purchase_rdd = events_rdd.filter(lambda x: x['event_type'] == 'purchase')
by_user_rdd = purchase_rdd.groupByKey().mapValues(lambda v: sum([x['amount'] for x in v]))
```

Lento, difícil de mantener, zero optimización

**La regla**: Si tienes datos estructurados, siempre DataFrame.

---

## Referencias

- [Catalyst Optimizer - Apache Spark](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [DataFrame API - PySpark Docs](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/dataframe.html)
