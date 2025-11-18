# PySpark Optimization & Caching Strategies

**Tags**: #pyspark #optimization #caching #performance #senior #real-interview

---

## TL;DR

Optimiza Spark con: (1) **Partitioning** — divide datos por clave, (2) **Caching** — reutiliza RDDs caros, (3) **Broadcasting** — reparte variables pequeñas a workers, (4) **Bucketing** — hash-basado, (5) **Predicate Pushdown** — filtra temprano. Monitorea con `.explain()` y Spark UI.

---

## Concepto Core

- **Qué es**: Optimization es hacer queries correr más rápido. Caching reutiliza datos costosos. Broadcasting minimiza network traffic
- **Por qué importa**: Diferencia entre query en 5min vs 1hr. Critical para data engineers. Demuestra madurez
- **Principio clave**: Identifica bottlenecks (shuffle, network), aplica técnica apropiada

---

## Memory Trick

**"Fábrica eficiente"**:

- Partitioning = producción distribuida (cada factory = parte de datos)
- Caching = almacén temporal (reutiliza products populares)
- Broadcasting = manuales en cada factory (información pequeña, no transmite)

---

## Cómo explicarlo en entrevista

**Paso 1**: "Spark es distribuido. El bottleneck = network (shuffle). Optimización = minimizar shuffle"

**Paso 2**: "Técnicas: Partitioning (menos shuffle), Caching (reutiliza), Broadcasting (info pequeña sin red)"

**Paso 3**: "Siempre usa `.explain()` para identificar DONDE está el problema. Luego aplica técnica"

**Paso 4**: "Monitor en Spark UI: busca long-running tasks. Eso = área a optimizar"

---

## Código/Query ejemplo

### Escenario: Join Gigante (Problema típico)

**Datos:**

- orders: 1 millón filas (1 GB) — distribuidao
- customers: 1 millón filas (500 MB) — pequeña
- products: 100k filas (50 MB) — muy pequeña

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, broadcast

spark = SparkSession.builder.appName("Optimization").getOrCreate()

orders = spark.read.parquet("orders/") # 1 GB
customers = spark.read.parquet("customers/") # 500 MB
products = spark.read.parquet("products/") # 50 MB

# ❌ PROBLEMA: Join sin optimización
slow_result = (
    orders
    .join(customers, "customer_id")  # Shuffle gigante: 1GB ↔ 500MB
    .join(products, "product_id")    # Shuffle de nuevo
)

print("Tiempo esperado: 15+ minutos (shuffle masivo)")
```

**Lo que pasa sin optimización:**

```
Shuffle orders por customer_id (1GB enviado por red)
↓
Shuffle customers por customer_id (500MB enviado por red)
↓
Match local (finalmente)
↓
Repeat para products
```

---

### ✅ Opción 1: Broadcasting (Recomendado para este caso)

```python
# products es pequeña (50 MB). Broadcast a todos workers
fast_result = (
    orders
    .join(broadcast(customers), "customer_id")  # ← Broadcast 500MB a cada worker
    .join(broadcast(products), "product_id")   # ← Broadcast 50MB a cada worker
)

# Tiempo: 1-2 minutos
# Qué pasó: Cada worker tiene copia de customers+products, local join
```

**Timeline:**

```
BROADCAST (instantáneo, < 1 GB cada):
customers (500MB) enviado 1 sola vez a cada worker
products (50MB) enviado 1 sola vez a cada worker

JOIN (local, rápido):
En cada worker: orders [local] ← join → customers [local] ← join → products [local]
```

---

### ✅ Opción 2: Partitioning (Para joins recurrentes)

```python
# Si haces este join muchas veces, particiona
customers_partitioned = customers.repartition(col("customer_id"))
products_partitioned = products.repartition(col("product_id"))

# Cacheamos particiones
customers_partitioned.cache()
products_partitioned.cache()

result = (
    orders
    .join(customers_partitioned, "customer_id")  # Menos shuffle
    .join(products_partitioned, "product_id")
)

# Primera ejecución: 5 minutos (partitioning + caching)
# Siguientes: 1 minuto (usa cache)
```

---

### ✅ Opción 3: Bucketing (Pre-partitioned en Storage)

```python
# Para queries recurrentes, guarda datos bucketed
# Writes ONE TIME:
customers.write.mode("overwrite").bucketBy(10, "customer_id").saveAsTable("customers_bucketed")
products.write.mode("overwrite").bucketBy(5, "product_id").saveAsTable("products_bucketed")

# Luego:
customers_b = spark.table("customers_bucketed")
products_b = spark.table("products_bucketed")

result = orders.join(customers_b, "customer_id").join(products_b, "product_id")

# Spark SABE que ya está bucketed, no hace shuffle (sorttable merge join)
# Tiempo: 1-2 minutos SIEMPRE (óptimo)
```

---

## Caching Strategies

### Cuándo Cachear

```python
# ❌ NO cachear todo
result.cache()  # Desperdicia memoria

# ✅ Cachear transformations CARAS
parsed_data = raw_data.flatMap(expensive_parsing_function)
parsed_data.cache()

# Usado múltiples veces:
result1 = parsed_data.filter(...)
result2 = parsed_data.groupBy(...)
result3 = parsed_data.join(...)
```

---

### Storage Levels

```python
from pyspark.storagelevel import StorageLevel

# MEMORY (rápido, puede faltar espacio)
df.cache()  # Equivalente a MEMORY
df.persist(StorageLevel.MEMORY_ONLY)

# MEMORY_AND_DISK (safe, más lento)
df.persist(StorageLevel.MEMORY_AND_DISK)

# DISK_ONLY (último recurso)
df.persist(StorageLevel.DISK_ONLY)

# MEMORY_SERIALIZED (comprimido, más lento pero compacto)
df.persist(StorageLevel.MEMORY_ONLY_SER)

# Recomendación: MEMORY_AND_DISK para data > 1GB
```

---

### Uncache

```python
# Liberar memoria después de usar
df.unpersist()

# Lazy unpersist (después de siguiente action)
df.unpersist(blocking=False)
```

---

## Broadcasting Variables

Para datos pequeños (< 2GB):

```python
# Diccionario de config (pequeño)
config = {"USD": 1.0, "EUR": 0.85, "GBP": 0.73}

# SIN broadcast (❌ ineficiente)
def convert_currency(row):
    # Accede a 'config' en cada worker (ineficiente)
    return config.get(row.currency)

result = df.rdd.map(convert_currency)

# CON broadcast (✅ eficiente)
broadcast_config = spark.broadcast(config)

def convert_currency_optimized(row):
    # Accede a copia local en worker
    return broadcast_config.value.get(row.currency)

result = df.rdd.map(convert_currency_optimized)
```

---

## Partitioning Strategies

### Repartition vs Coalesce

```python
# REPARTITION: Siempre rehash (shuffle)
df.repartition(100)  # Útil si aumentas particiones

# COALESCE: Merge particiones sin shuffle
df.coalesce(10)  # Útil si disminuyes particiones (10x más rápido)

# Regla: coalesce si menos particiones, repartition si más
```

---

### Partitioning Smart

```python
# ¿Cuántas particiones?
# Regla: 1 partition = 128 MB idealmente
# Para 1 GB de datos:
df.repartition(10)  # ≈ 100 MB por partición (óptimo)

# Para 10 GB:
df.repartition(100)  # ≈ 100 MB por partición

# Demasiadas particiones = overhead
# Pocas particiones = bajo paralelismo
```

---

## Identify Bottlenecks

### Via `.explain()`

```python
# Modo simple (básico)
df.explain(mode="simple")

# Modo extended (Catalyst optimizations)
df.explain(mode="extended")
```

Busca estos problemas:

- Exchange (SHUFFLE) - caro
- Cartesian Product - muy caro
- Broadcast Join vs Sort Merge Join

```python
result = orders.join(customers, "customer_id")
result.explain(mode="extended")
```

Si ves "Exchange" → Reduce con broadcast/partition
Si ves "Cartesian" → Hay bug en join condition

---

### Via Spark UI

**URL:** `http://localhost:4040` (durante ejecución)

**Qué ver:**

- **Stages**: Cuál es más lento?
- **Tasks**: Task que corre > 1 minuto?
- **Shuffle**: Cuántos bytes se movieron?
- **Skewed**: Un task mucho más lento que otros?

**Acción:**

- Task lento = aumento particiones en ese stage
- Shuffle grande = considera broadcast/bucketing

---

## Real-World: Full Optimization Pipeline

```python
# Escenario: Daily report, 10 GB orders, 1 GB customers
orders = spark.read.parquet("orders/daily")
customers = spark.read.parquet("customers/")

# Step 1: Broadcast small dimension
customers_bcast = broadcast(customers)

# Step 2: Partition orders by date (pre-filter)
orders_filtered = orders.filter(col("date") >= "2024-01-01")

# Step 3: Join with broadcast
joined = orders_filtered.join(customers_bcast, "customer_id")

# Step 4: Cache before multiple aggregations
joined.cache()

# Step 5: Multiple aggregations (usan cache)
by_customer = joined.groupBy("customer_id").agg(sum("amount"))
by_product = joined.groupBy("product").agg(count("*"))
by_date = joined.groupBy("date").agg(avg("amount"))

# Step 6: Write results
by_customer.write.parquet("output/by_customer")
by_product.write.parquet("output/by_product")
by_date.write.parquet("output/by_date")

# Performance:
# - Sin optimización: 30 minutos
# - Con broadcasts: 10 minutos
# - Con caching: 5 minutos (primero) + 1 min (otros)
```

---

## Errores comunes en entrevista

- **Error**: Cachear DataFrame gigante (no cabe en memoria) → **Solución**: Usa `StorageLevel.MEMORY_AND_DISK`

- **Error**: Usar `collect()` después de caching → **Solución**: `.show()` o `.write()` (evita OOM)

- **Error**: No usar broadcast para joins pequeños → **Solución**: 2GB → broadcast automático, < 2GB → manual broadcast

- **Error**: No uncache después de usar → **Solución**: Libera memoria: `df.unpersist()`

---

## Preguntas de seguimiento típicas

1. **"¿Cuándo usas broadcast vs partitioning?"**
   - Broadcast: 1 tabla pequeña, múltiples tables grandes
   - Partitioning: recurrente, > 10 GB total

2. **"¿Cómo detectas skewed tasks?"**
   - Spark UI: ver si un task toma 10x más tiempo
   - Solución: repartition data de manera más equitativa

3. **"¿Cuántas particiones debería usar?"**
   - Regla: `cores * 2` a `cores * 4`
   - O: ~100 MB por partición

4. **"Diferencia entre cache() y checkpoint()?"**
   - cache(): memoria, rápido, perdible si ejecutor falla
   - checkpoint(): disk, lento, persistente (para RDDs problematic)

---

## References

- [Caching & Persistence - Spark Docs](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)
- [Broadcasting - Spark Docs](https://spark.apache.org/docs/latest/rdd-programming-guide.html#broadcast-variables)
- [Bucketing - Spark SQL](https://spark.apache.org/docs/latest/sql-data-sources-parquet.html#partition-discovery)
