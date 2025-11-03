# CTEs (Common Table Expressions) - WITH Clause

**Tags**: #sql #cte #readability #best-practices #real-interview  
**Empresas**: Amazon, Adobe, Google, Stripe  
**Dificultad**: Mid  
**Tiempo estimado**: 18 min

---

## TL;DR

Un **CTE (WITH clause)** es una tabla temporal dentro del query. Te permite escribir código limpio y legible dividiendo queries complejas en partes reutilizables. Sintaxis: `WITH cte_name AS (SELECT ...) SELECT ... FROM cte_name`.

---

## Concepto Core

- **Qué es**: Una tabla temporal que existe solo durante la ejecución del query. La defines con `WITH nombre AS (...)` y luego la usas como si fuera una tabla real
- **Por qué importa**: Hace queries complejos legibles. Es mejor que subqueries anidadas. Permite reutilizar lógica. Fundamental en data engineering
- **Principio clave**: CTEs se evalúan una sola vez (si es no-recursive). Son como "variables temporales" en SQL

---

## Memory Trick

**"Tabla prestada"** — WITH te deja crear una tabla temporal que existe solo mientras el query corre. Usas como tabla normal, pero luego desaparece.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Para queries complejos, en lugar de anidar subqueries (difícil de leer), uso CTEs"

**Paso 2**: "Defino la CTE con `WITH nombre AS (SELECT ...)`. Dentro va la lógica compleja"

**Paso 3**: "Luego en el SELECT principal, uso la CTE como si fuera una tabla: `SELECT ... FROM nombre`. Mucho más limpio"

---

## Código/Query ejemplo

### Problema: Query Anidado (Difícil de leer)

SELECT
customer_id,
(SELECT AVG(order_amount) FROM orders o2 WHERE o2.customer_id = o1.customer_id) as avg_order,
(SELECT COUNT(\*) FROM orders o3 WHERE o3.customer_id = o1.customer_id AND o3.status = 'completed') as completed_orders
FROM orders o1
WHERE customer_id IN (
SELECT customer_id FROM orders
GROUP BY customer_id
HAVING SUM(order_amount) > 1000
)
GROUP BY customer_id;

text

❌ **Problema**: Anidado, difícil de seguir, subqueries ejecutadas múltiples veces

### Solución: Con CTEs (Limpio y legible)

-- Paso 1: Clientes con compras > 1000
WITH high_value_customers AS (
SELECT
customer_id,
SUM(order_amount) as total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(order_amount) > 1000
),

-- Paso 2: Estadísticas por cliente
customer_stats AS (
SELECT
customer_id,
COUNT(\*) as total_orders,
AVG(order_amount) as avg_order_amount,
MAX(order_amount) as max_order,
MIN(order_amount) as min_order
FROM orders
GROUP BY customer_id
)

-- Paso 3: Join y resultado final
SELECT
h.customer_id,
h.total_spent,
s.total_orders,
s.avg_order_amount,
s.max_order,
s.min_order,
ROUND(h.total_spent / s.total_orders, 2) as avg_per_order
FROM high_value_customers h
JOIN customer_stats s ON h.customer_id = s.customer_id
ORDER BY h.total_spent DESC;

text

✅ **Ventajas**:

- Cada paso es claro
- Fácil de debuggear
- Lógica reutilizable
- SQL se lee como prosa

---

## Multiple CTEs (Encadenadas)

WITH
-- CTE 1: Base de datos limpia
cleaned_orders AS (
SELECT
order_id,
customer_id,
order_date,
order_amount,
CASE
WHEN order_amount < 0 THEN 0
ELSE order_amount
END as adjusted_amount
FROM raw_orders
WHERE order_date >= '2024-01-01'
),

-- CTE 2: Agregación
monthly_summary AS (
SELECT
customer_id,
DATE_TRUNC('month', order_date) as month,
COUNT(\*) as orders_that_month,
SUM(adjusted_amount) as revenue_that_month
FROM cleaned_orders
GROUP BY customer_id, DATE_TRUNC('month', order_date)
),

-- CTE 3: Ranking
ranked_customers AS (
SELECT
customer_id,
month,
revenue_that_month,
RANK() OVER (PARTITION BY customer_id ORDER BY revenue_that_month DESC) as rank_in_customer_history
FROM monthly_summary
)

-- Query principal
SELECT
customer_id,
month,
revenue_that_month
FROM ranked_customers
WHERE rank_in_customer_history <= 3
ORDER BY customer_id, revenue_that_month DESC;

text

---

## Recursive CTEs (Avanzado)

Para problemas con jerarquías o secuencias:

-- Ejemplo: Generar números del 1 al 10
WITH RECURSIVE numbers AS (
-- Anchor (caso base)
SELECT 1 as num

UNION ALL

-- Recursive (suma 1 hasta llegar a 10)
SELECT num + 1
FROM numbers
WHERE num < 10
)
SELECT \* FROM numbers;

text

⚠️ **Cuidado**: Recursive CTEs pueden ser lentas. Usa con moderación.

---

## Errores comunes en entrevista

- **Error**: Definir la CTE pero no usarla en el SELECT principal → **Solución**: La CTE debe ser usada, sino es código muerto

- **Error**: Intentar usar una CTE que no está definida → **Solución**: Verifica que la CTE está en el `WITH` y antes de usarla

- **Error**: No separar múltiples CTEs con comas → **Solución**: Sintaxis correcta: `WITH cte1 AS (...), cte2 AS (...), cte3 AS (...)`

- **Error**: Recursive CTE sin condición STOP → **Solución**: Siempre incluye `WHERE` para parar la recursión, sino loop infinito

---

## Preguntas de seguimiento típicas

1. **"¿Diferencia entre CTE y subquery?"**
   - CTE: Más legible, reutilizable, se define una vez
   - Subquery: Anidado dentro, menos legible, puede ejecutarse múltiples veces
   - Performance: Similar, pero CTE es mejor para legibilidad

2. **"¿Cuando usarías CTE vs Temporary Table?"**
   - CTE: Query único, desaparece después
   - Temp Table: Persiste sesión, puedes acceder múltiples veces
   - CTE es preferido por limpieza

3. **"¿Puedes reutilizar una CTE múltiples veces en el mismo query?"**
   - Sí, `SELECT * FROM cte UNION ALL SELECT * FROM cte` es válido
   - Pero se define UNA sola vez en el WITH

4. **"¿Cómo optimizarías un CTE lento?"**
   - Agrega índices en las columnas usadas
   - Particiona datos si es posible
   - Usa MATERIALIZED hint (en algunos motores)

---

## Comparación: Subquery vs CTE vs Temp Table

| Aspecto           | Subquery                         | CTE                | Temp Table                     |
| ----------------- | -------------------------------- | ------------------ | ------------------------------ |
| **Sintaxis**      | Anidada dentro de SELECT         | WITH...AS          | CREATE TABLE                   |
| **Legibilidad**   | Difícil (nidación profunda)      | Fácil (top-down)   | Clara pero verbose             |
| **Performance**   | Puede ser lento (repite cálculo) | Similar a subquery | Mejor si reutilizada           |
| **Reutilización** | Cada uso = evaluación            | Define una vez     | Persistente                    |
| **Cuándo usar**   | Simple, una vez                  | Complejo, modular  | Datos grandes, varias sesiones |

---

## Real-World Scenario: Data Warehouse

En data engineering, CTEs se usan constantemente:

WITH
-- Stage 1: Source data
source_data AS (
SELECT \* FROM raw_events WHERE date >= '2024-01-01'
),

-- Stage 2: Transformations
transformed_data AS (
SELECT
user_id,
event_type,
TIMESTAMP as event_time,
CASE WHEN event_type = 'purchase' THEN amount ELSE 0 END as revenue
FROM source_data
),

-- Stage 3: Aggregation
daily_summary AS (
SELECT
DATE(event_time) as event_date,
COUNT(\*) as total_events,
SUM(revenue) as daily_revenue
FROM transformed_data
GROUP BY DATE(event_time)
)

-- Final: Load to warehouse
INSERT INTO analytics.daily_events
SELECT \* FROM daily_summary;

text

---

## Referencias

- [CTEs (WITH clause) - PostgreSQL Docs](https://www.postgresql.org/docs/current/queries-with.html)
- [CTEs Explained - Mode Analytics](https://mode.com/sql-tutorial/sql-cte/)
- [Recursive CTEs - Advanced SQL](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE)
