# Window Functions en PySpark

**Tags**: #pyspark #window-functions #ranking #real-interview

---

## TL;DR

Window functions en Spark = window functions en SQL pero código Python. Usa `Window.partitionBy().orderBy()` para definir ventana, luego aplica `row_number()`, `rank()`, `dense_rank()`, `lag()`, `lead()`, etc. Similar a SQL pero más flexible. Muy usado en transformaciones de datos.

---

## Concepto Core

- **Qué es**: Window function calcula valor para cada fila basado en "ventana" de filas (grupo + orden)
- **Por qué importa**: Fundamental en transformaciones. Ranking, sumas acumulativas, lead/lag son muy comunes. Demuestra dominio de Spark
- **Principio clave**: Window = PARTITION BY + ORDER BY. Aplica función dentro de cada ventana

---

## Memory Trick

**"Ventanas deslizantes"** — Imagina ventana que se desliza sobre datos. Cada fila tiene su propia ventana (grupo + contexto). La función se aplica en cada ventana.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Window functions hacen cálculos 'sobre ventanas' de filas, no solo colapsando grupos"

**Paso 2**: "Defino ventana con `Window.partitionBy(col).orderBy(col)` — es como GROUP BY + ORDER BY"

**Paso 3**: "Aplico función window: `row_number()`, `rank()`, `lag()`, `sum().over(window)`"

**Paso 4**: "Resultado: cada fila original + valor calculado dentro de su ventana"

---

## Código/Query ejemplo

### Datos: Sales por empleado

| id_empleado | nombre  | departamento | monto | fecha      |
| ----------- | ------- | ------------ | ----- | ---------- |
| 1           | Alice   | Ventas       | 1000  | 2024-01-01 |
| 2           | Bob     | Ventas       | 1500  | 2024-01-02 |
| 3           | Charlie | TI           | 2000  | 2024-01-03 |
| 1           | Alice   | Ventas       | 800   | 2024-01-04 |
| 4           | David   | TI           | 2500  | 2024-01-05 |
| 2           | Bob     | Ventas       | 900   | 2024-01-06 |

---

### Problema 1: Ranking Dentro de Cada Departamento

```python
from pyspark.sql import Window
from pyspark.sql.functions import row_number, rank, dense_rank, col

spark = SparkSession.builder.appName("Window Functions").getOrCreate()

sales = spark.read.parquet("sales.parquet")

# Define la ventana: particiona por department, ordena por amount (descendente)
window = Window.partitionBy("departamento").orderBy(col("monto").desc())

# Aplica ranking dentro de cada ventana
result = (
    sales
    .withColumn("rank_en_dpto", rank().over(window))
    .withColumn("dense_rank_en_dpto", dense_rank().over(window))
    .withColumn("row_number_en_dpto", row_number().over(window))
    .select("id_empleado", "nombre", "departamento", "monto", "rank_en_dpto", "dense_rank_en_dpto", "row_number_en_dpto")
)

result.show()
```

**Resultado:**
id_empleado | nombre | departamento | monto | rank_en_dpto | dense_rank_en_dpto | row_number_en_dpto
---|---|---|---|---|---|---
4 | David | TI | 2500 | 1 | 1 | 1
3 | Charlie | TI | 2000 | 2 | 2 | 2
2 | Bob | Ventas | 1500 | 1 | 1 | 1
1 | Alice | Ventas | 1000 | 2 | 2 | 2
2 | Bob | Ventas | 900 | 3 | 3 | 3
1 | Alice | Ventas | 800 | 4 | 4 | 4

---

### Problema 2: Running Total (Suma Acumulativa)

```python
# Window: particiona por employee, ordena cronológicamente
window = Window.partitionBy("id_empleado").orderBy("fecha")

result = (
    sales
    .withColumn("total_acumulado", sum("monto").over(window))
    .select("id_empleado", "nombre", "fecha", "monto", "total_acumulado")
)

result.show()
```

**Resultado:**
id_empleado | nombre | fecha | monto | total_acumulado
---|---|---|---|---
1 | Alice | 2024-01-01 | 1000 | 1000
1 | Alice | 2024-01-04 | 800 | 1800
2 | Bob | 2024-01-02 | 1500 | 1500
2 | Bob | 2024-01-06 | 900 | 2400

---

### Problema 3: LAG y LEAD (Fila Anterior/Siguiente)

```python
# LAG: valor de fila anterior
# LEAD: valor de fila siguiente
window = Window.partitionBy("id_empleado").orderBy("fecha")

result = (
    sales
    .withColumn("monto_anterior", lag("monto").over(window))
    .withColumn("monto_siguiente", lead("monto").over(window))
    .withColumn("diferencia_monto", col("monto") - col("monto_anterior"))
    .select("id_empleado", "fecha", "monto", "monto_anterior", "monto_siguiente", "diferencia_monto")
)

result.show()
```

**Resultado:**
id_empleado | fecha | monto | monto_anterior | monto_siguiente | diferencia_monto
---|---|---|---|---|---
1 | 2024-01-01 | 1000 | NULL | 800 | NULL
1 | 2024-01-04 | 800 | 1000 | NULL | -200
2 | 2024-01-02 | 1500 | NULL | 900 | NULL
2 | 2024-01-06 | 900 | 1500 | NULL | -600

---

### Problema 4: Window con ROWS (Ventanas Específicas)

```python
# Window ROWS: especifica rango (últimas 2 filas, etc.)
# UNBOUNDED PRECEDING: desde inicio
# CURRENT ROW: fila actual
# n FOLLOWING: n filas después
# Última suma de 2 filas (incluida actual)
window_2rows = (
    Window
    .partitionBy("id_empleado")
    .orderBy("fecha")
    .rowsBetween(-1, 0) # 1 fila anterior + actual
)

result = (
    sales
    .withColumn("suma_ultimas_2", sum("monto").over(window_2rows))
    .select("id_empleado", "fecha", "monto", "suma_ultimas_2")
)

result.show()
```

**Resultado:**
id_empleado | fecha | monto | suma_ultimas_2
---|---|---|---
1 | 2024-01-01 | 1000 | 1000 (solo actual, no hay anterior)
1 | 2024-01-04 | 800 | 1800 (1000 + 800)
2 | 2024-01-02 | 1500 | 1500 (solo actual)
2 | 2024-01-06 | 900 | 2400 (1500 + 900)

---

### Problema 5: Top N por Grupo (Patrón Común)

```python
# Ranking + Filter = Top N por grupo
window = Window.partitionBy("departamento").orderBy(col("monto").desc())

result = (
    sales
    .withColumn("rank", rank().over(window))
    .filter(col("rank") <= 2) # Top 2 por departamento
    .select("id_empleado", "nombre", "departamento", "monto", "rank")
)

result.show()
```

**Resultado:**
id_empleado | nombre | departamento | monto | rank
---|---|---|---|---
4 | David | TI | 2500 | 1
3 | Charlie | TI | 2000 | 2
2 | Bob | Ventas | 1500 | 1
1 | Alice | Ventas | 1000 | 2

---

## Window Functions Comunes

| Función             | Uso               | Ejemplo                  |
| ------------------- | ----------------- | ------------------------ |
| `row_number()`      | Número secuencial | 1, 2, 3, 4               |
| `rank()`            | Rank con saltos   | 1, 2, 2, 4               |
| `dense_rank()`      | Rank sin saltos   | 1, 2, 2, 3               |
| `lag(col, offset)`  | Fila anterior     | Valor de -1 fila         |
| `lead(col, offset)` | Fila siguiente    | Valor de +1 fila         |
| `sum(col).over()`   | Suma acumulativa  | Suma hasta fila actual   |
| `avg(col).over()`   | Promedio ventana  | Promedio en ventana      |
| `max(col).over()`   | Máximo ventana    | Máximo en ventana        |
| `min(col).over()`   | Mínimo ventana    | Mínimo en ventana        |
| `count(col).over()` | Contar ventana    | Cuántas filas en ventana |
| `first(col).over()` | Primer valor      | Valor de primera fila    |
| `last(col).over()`  | Último valor      | Valor de última fila     |

---

## Window Specifications

```python
# Ventana básica: todo el dataset, sin partición
window_all = Window.orderBy("fecha")

# Partición única: grupo + orden
window_dept = Window.partitionBy("departamento").orderBy(col("monto").desc())

# Múltiples particiones: grupo por 2+ columnas
window_multi = Window.partitionBy("departamento", "region").orderBy("fecha")

# Sin orden: solo partición
window_no_order = Window.partitionBy("departamento")

# ROWS specification: cuántas filas antes/después
window_rows = (
    Window
    .partitionBy("departamento")
    .orderBy("fecha")
    .rowsBetween(-2, 1) # 2 filas antes + actual + 1 fila después
)

# RANGE specification: por valor, no filas
window_range = (
    Window
    .partitionBy("departamento")
    .orderBy("monto")
    .rangeBetween(-100, 100) # Rango ±100 de monto actual
)
```

---

## Errores comunes en entrevista

- **Error**: Olvidar `orderBy()` en window → **Solución**: `partitionBy()` sin `orderBy()` da orden arbitrario. Siempre especifica ambos

- **Error**: Usar `rank()` cuando necesitas `row_number()` → **Solución**: `rank()` salta (1,2,2,4), `row_number()` secuencial (1,2,3,4). Conoce diferencia

- **Error**: No filtrar después de window function → **Solución**: Top N requiere `rank().over() then filter(rank <= N)`

- **Error**: Aplicar window a columna que no existe → **Solución**: Asegúrate que columna está en select antes de window

---

## Preguntas de seguimiento típicas

1. **"¿Diferencia entre `rowsBetween` y `rangeBetween`?"**
   - `rowsBetween`: Contar filas (1 anterior, actual, 1 siguiente)
   - `rangeBetween`: Rango de valores (valores dentro de rango)

2. **"¿Cómo haces running total por mes?"**
   - Particiona por mes: `Window.partitionBy(month()).orderBy(fecha)`
   - Reset automático cada mes

3. **"¿Puedes aplicar múltiples window functions?"**
   - Sí: `.withColumn("rank", rank().over(w1)).withColumn("lag", lag().over(w2))`

4. **"¿Performance de window functions?"**
   - Shuffle ocurre en `partitionBy`. Minimiza particiones si posible
   - `rowsBetween` es más rápido que `rangeBetween`

---

## Real-World: Customer Lifetime Value

```python
# Cálculo: última compra + total gasto + ranking por valor
window_customer = Window.partitionBy("id_cliente").orderBy(col("fecha").desc())
window_all = Window.orderBy(col("total_gastado").desc())

result = (
    sales
    .groupBy("id_cliente")
    .agg(
        sum("monto").alias("total_gastado"),
        max("fecha").alias("ultima_compra"),
        count("*").alias("num_compras")
    )
    .withColumn("dias_desde_ultima_compra",
                 datediff(current_date(), col("ultima_compra")))
    .withColumn("rank_cliente", rank().over(window_all))
    .filter(col("rank_cliente") <= 100)
    .orderBy("rank_cliente")
)

result.show()
```

---

## References

- [Window Functions - PySpark Docs](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html#window-functions)
- [Window Class - API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/window.html)
