# Query Optimization & Indexes

**Tags**: #sql #performance #optimization #indexing #senior #real-interview  
**Empresas**: Amazon, Google, Meta, Stripe  
**Dificultad**: Senior  
**Tiempo estimado**: 25 min

---

## TL;DR

Para optimizar queries lentos: (1) Analiza el EXPLAIN PLAN, (2) Agrega índices en columnas usadas en WHERE/JOIN, (3) Reescribe la lógica si es necesario, (4) Particiona datos si es posible. Índices aceleran búsquedas pero ralentizan INSERTs/UPDATEs.

---

## Concepto Core

- **Qué es**: Optimization es el arte de hacer queries correr más rápido sin cambiar el resultado. Indexes son estructuras que aceleran búsquedas
- **Por qué importa**: En data engineering, diferencia entre query que corre en 5s vs 5 minutos. Critical para prod
- **Principio clave**: Usa EXPLAIN PLAN para ver qué hace la BD. Agrega índices donde se hace Sequential Scan. Trade-off: lecturas rápidas, escrituras lentas

---

## Memory Trick

**"Libro vs Índice"** — Sin índice, BD lee todas las filas (Sequential Scan) como leer un libro línea por línea. Con índice (B-tree), salta directo como un índice de libro.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Si un query es lento, primero corro EXPLAIN PLAN para ver qué hace la BD"

**Paso 2**: "Busco Sequential Scans en columnas grandes. Eso significa que lee TODAS las filas. Ahí agrego un índice"

**Paso 3**: "Reescriatura: Evito `WHERE UPPER(name) = ...` (función desactiva índice). Uso `WHERE name = UPPER(...)` en la data, no en la BD"

**Paso 4**: "Valido que el query ahora usa el índice (Index Scan en EXPLAIN). Chequeo tiempo antes/después"

---

## Código/Query ejemplo

### Escenario: Query Lento en Tabla de 100M de filas

-- ❌ QUERY LENTO (sin índice o mal escrito)
SELECT
customer_id,
COUNT() as order_count,
SUM(amount) as total_spent
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2024
AND order_status = 'completed'
AND UPPER(customer_name) LIKE '%JOHN%'
GROUP BY customer_id
HAVING COUNT() > 10
ORDER BY total_spent DESC;

text

**EXPLAIN PLAN output (problema):**
Seq Scan on orders (cost=0.00..500000.00 rows=1000000)
Filter: (EXTRACT(YEAR, order_date) = 2024) AND (order_status = 'completed') AND (UPPER(customer_name) LIKE '%JOHN%')

text

⚠️ **Problemas**:

1. **Seq Scan**: Lee TODAS las 100M filas
2. **EXTRACT en WHERE**: Función desactiva índice en `order_date`
3. **UPPER()**: Función desactiva índice en `customer_name`
4. **LIKE '%JOHN%'**: Pattern matching lento (full table scan)

---

### Paso 1: Crear Índices

-- Índice en order_date (sin función)
CREATE INDEX idx_orders_order_date ON orders(order_date);

-- Índice en order_status
CREATE INDEX idx_orders_status ON orders(order_status);

-- Índice compuesto (mejor para esta query)
CREATE INDEX idx_orders_composite ON orders(order_date, order_status, customer_id);

-- Nota: UPPER(customer_name) no se puede indexar bien con LIKE '%...'
-- Mejor solución: full-text search o denormalization

text

---

### Paso 2: Reescribir Query (evitar funciones)

-- ✅ QUERY OPTIMIZADO
SELECT
customer_id,
COUNT() as order_count,
SUM(amount) as total_spent
FROM orders
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01' -- Sin EXTRACT()
AND order_status = 'completed'
AND customer_name ILIKE '%john%' -- ILIKE sin UPPER()
GROUP BY customer_id
HAVING COUNT() > 10
ORDER BY total_spent DESC;

text

**EXPLAIN PLAN output (mejorado):**
Index Scan using idx_orders_composite on orders (cost=100.00..5000.00 rows=50000)
Index Cond: (order_date >= '2024-01-01' AND order_date < '2025-01-01' AND order_status = 'completed')
Filter: (ILIKE '%john%')
Group Aggregate (cost=5000.00..6000.00 rows=100)

text

✅ **Mejoras**:

1. Index Scan en lugar de Seq Scan (100x más rápido)
2. Lee solo ~50k filas en lugar de 100M
3. Índice compuesto acelera el filtro

**Comparación de tiempo**:

- Antes: 45 segundos (Seq Scan)
- Después: 0.3 segundos (Index Scan)

---

## Tipos de Índices

| Tipo            | Uso                             | Ventaja                      | Desventaja                |
| --------------- | ------------------------------- | ---------------------------- | ------------------------- |
| **B-Tree**      | Por defecto, rango queries      | Rápido para =, <, >, BETWEEN | No help para LIKE '%...'  |
| **Hash**        | Igualdad (=)                    | Super rápido                 | Solo = operator           |
| **Full-Text**   | Búsqueda de texto               | Soporta LIKE, palabras       | Lento para datos no-texto |
| **Bitmap**      | Data warehouse, low cardinality | Comprimido                   | No para OLTP              |
| **Partitioned** | Tablas enormes                  | Elimina particiones          | Complejo de mantener      |

---

## Errores comunes en entrevista

- **Error**: Crear índice en cada columna → **Solución**: Índices ralentizan INSERTs. Crea solo donde se necesita (WHERE, JOIN, ORDER BY)

- **Error**: No usar EXPLAIN PLAN antes de optimizar → **Solución**: Siempre EXPLAIN primero. A veces el problema no es índices

- **Error**: Usar función en WHERE (EXTRACT, UPPER, etc.) → **Solución**: Evita funciones en columnas indexadas

- **Error**: Índice compuesto en orden incorrecto → **Solución**: Orden importa. `(col1, col2)` ≠ `(col2, col1)` para queries

---

## Preguntas de seguimiento típicas

1. **"¿Diferencia entre UNIQUE index y PRIMARY KEY?"**
   - PRIMARY KEY: Unique + NOT NULL + tabla solo tiene 1
   - UNIQUE index: Solo enforces uniqueness, puede haber NULLs

2. **"¿Cuándo NO deberías usar índices?"**
   - Columnas con baja selectividad (95% de filas = true)
   - Tablas muy pequeñas (<1M filas)
   - Columnas actualizadas frecuentemente (índice ralentiza UPDATE)

3. **"¿Cómo debuggearías un query lento?"**
   - EXPLAIN PLAN
   - ANALYZE (actualiza stats)
   - Chequea índices existentes
   - Mira si hay missing indexes

4. **"¿Spark vs PostgreSQL: ¿Cómo optimizas en Spark?"**
   - Spark: No hay índices. Usa **partitioning** y **bucketing**
   - Particiona por columna usada en WHERE
   - Bucketing para join keys

---

## Real-World: Data Lake Optimization

En Spark/Data Lake (sin índices):

-- ❌ SLOW: Sin partición, lee TODO
SELECT \* FROM sales_data WHERE country = 'US' AND year = 2024;

-- ✅ FAST: Datos particionados por country/year
-- Spark automáticamente prune partitions
CREATE TABLE sales_data (
order_id INT,
amount DECIMAL,
country STRING,
year INT
)
PARTITIONED BY (country, year);

-- Mismo query, pero Spark solo lee partición US/2024
SELECT \* FROM sales_data WHERE country = 'US' AND year = 2024;

text

---

## Checklist para Optimizar Query Lento

1. ✅ Corro EXPLAIN PLAN
2. ✅ Identifico Seq Scans
3. ✅ Chequeo si hay índices en esas columnas
4. ✅ Si no, creo índice compuesto (si es posible)
5. ✅ Reescrito query para evitar funciones en WHERE
6. ✅ Valido con EXPLAIN nuevamente
7. ✅ Comparo time antes/después

---

## Referencias

- [EXPLAIN PLAN - PostgreSQL Docs](https://www.postgresql.org/docs/current/sql-explain.html)
- [Index Types - Use The Index Luke](https://use-the-index-luke.com/)
- [Query Optimization - Mode Analytics](https://mode.com/sql-tutorial/sql-performance-tuning/)
- [Spark Partitioning & Bucketing](https://spark.apache.org/docs/latest/sql-data-sources-parquet.html)
