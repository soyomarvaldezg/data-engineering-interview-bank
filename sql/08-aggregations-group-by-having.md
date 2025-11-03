# Aggregations Complejas: GROUP BY & HAVING

**Tags**: #sql #aggregations #group-by #having #data-analysis #real-interview  
**Empresas**: Amazon, Walmart, Google, Adobe  
**Dificultad**: Mid  
**Tiempo estimado**: 18 min  

---

## TL;DR

`GROUP BY` agrupa filas. `HAVING` filtra grupos (como WHERE pero para agregados). Usa `GROUP_CONCAT` (MySQL) / `STRING_AGG` (PostgreSQL) / `LISTAGG` (Oracle) para concatenar valores dentro de grupos. Siempre: SELECT solo columnas que están en GROUP BY o funciones agregadas.

---

## Concepto Core

- **Qué es**: GROUP BY divide datos en grupos. HAVING filtra esos grupos basado en agregados. Funciones como SUM, COUNT, STRING_AGG combinen valores dentro de cada grupo
- **Por qué importa**: Fundamental en reporting y análisis. Demuestra comprensión de agregaciones y cómo pensar en "grupos" vs "filas"
- **Principio clave**: WHERE filtra filas ANTES de GROUP BY. HAVING filtra grupos DESPUÉS de GROUP BY

---

## Memory Trick

**"Dividir, agregar, filtrar"** — GROUP BY divide, SUM/COUNT agregan, HAVING filtra el resultado agregado.

---

## Cómo explicarlo en entrevista

**Paso 1**: "GROUP BY divide filas en grupos basado en una columna (ej: por categoría)"

**Paso 2**: "Dentro de cada grupo, aplico funciones agregadas: COUNT(), SUM(), AVG(), etc."

**Paso 3**: "HAVING filtra esos grupos. Si necesito solo grupos con suma > 1000, uso `HAVING SUM(amount) > 1000`"

**Paso 4**: "Diferencia: WHERE filtra filas individuales. HAVING filtra grupos agregados"

---

## Código/Query ejemplo

### Tabla: sales

order_id | customer_id | product | category | amount | order_date
1 | 101 | Laptop | Electronics| 1000 | 2024-01-15
2 | 101 | Mouse | Electronics| 25 | 2024-01-20
3 | 102 | Desk | Furniture | 300 | 2024-02-01
4 | 102 | Chair | Furniture | 150 | 2024-02-10
5 | 103 | Monitor | Electronics| 400 | 2024-01-05
6 | 101 | Keyboard | Electronics| 80 | 2024-03-01
7 | 104 | Lamp | Furniture | 50 | 2024-02-15

text

---

### Problema 1: Total de Ventas por Categoría (Básico)

SELECT
category,
COUNT(*) as total_orders,
SUM(amount) as total_revenue,
AVG(amount) as avg_order,
MIN(amount) as min_order,
MAX(amount) as max_order
FROM sales
GROUP BY category
ORDER BY total_revenue DESC;

text

**Resultado:**
category | total_orders | total_revenue | avg_order | min_order | max_order
Electronics | 4 | 1505 | 376.25 | 25 | 1000
Furniture | 3 | 500 | 166.67 | 50 | 300

text

✅ **Nota**: SELECT contiene SOLO lo que está en GROUP BY (category) o agregados (COUNT, SUM)

---

### Problema 2: HAVING - Filtrar Grupos

-- ¿Categorías con ingresos > 500?
SELECT
category,
COUNT(*) as total_orders,
SUM(amount) as total_revenue
FROM sales
GROUP BY category
HAVING SUM(amount) > 500 -- Filtra GRUPOS, no filas
ORDER BY total_revenue DESC;

text

**Resultado:**
category | total_orders | total_revenue
Electronics | 4 | 1505

text

⚠️ **¿Dónde va cada filtro?**
WHERE → Filtra filas ANTES de GROUP BY
HAVING → Filtra grupos DESPUÉS de GROUP BY

SELECT category, SUM(amount) as rev
FROM sales
WHERE amount > 100 -- Filtra filas: solo ordenes > 100
GROUP BY category
HAVING SUM(amount) > 500 -- Filtra grupos: solo categorías con suma > 500

text

---

### Problema 3: Concatenar Valores en un Grupo

-- Todos los productos vendidos en cada categoría, en una lista
SELECT
category,
COUNT(*) as total_products,
STRING_AGG(product, ', ') as product_list, -- PostgreSQL
-- GROUP_CONCAT(product) as product_list, -- MySQL
-- LISTAGG(product, ', ') WITHIN GROUP (ORDER BY product) -- Oracle
SUM(amount) as total_revenue
FROM sales
GROUP BY category
ORDER BY total_revenue DESC;

text

**Resultado:**
category | total_products | product_list | total_revenue
Electronics | 4 | Laptop, Mouse, Monitor, Keyboard | 1505
Furniture | 3 | Chair, Desk, Lamp | 500

text

---

### Problema 4: GROUP BY con Múltiples Columnas

-- Ventas por categoría Y mes
SELECT
category,
DATE_TRUNC('month', order_date) as month,
COUNT(*) as order_count,
SUM(amount) as monthly_revenue
FROM sales
GROUP BY category, DATE_TRUNC('month', order_date)
ORDER BY category, month;

text

**Resultado:**
category | month | order_count | monthly_revenue
Electronics | 2024-01-01 | 3 | 1425
Electronics | 2024-03-01 | 1 | 80
Furniture | 2024-02-01 | 3 | 500

text

---

### Problema 5: Clientes con > 2 Órdenes (HAVING con COUNT)

SELECT
customer_id,
COUNT() as order_count,
SUM(amount) as total_spent,
STRING_AGG(product, ', ' ORDER BY product) as products_purchased
FROM sales
GROUP BY customer_id
HAVING COUNT() > 1 -- Solo clientes con más de 1 orden
ORDER BY order_count DESC;

text

**Resultado:**
customer_id | order_count | total_spent | products_purchased
101 | 3 | 1105 | Keyboard, Laptop, Mouse
102 | 2 | 450 | Chair, Desk

text

---

### Problema 6: WHERE + GROUP BY + HAVING (Combinados)

-- Categorías que tienen > 1 orden DE ELECTRONICS, con total > 300
SELECT
category,
COUNT() as order_count,
SUM(amount) as total_revenue
FROM sales
WHERE amount > 50 -- Filtra filas: ordenes > 50
GROUP BY category
HAVING SUM(amount) > 300 AND COUNT() > 1 -- Filtra grupos
ORDER BY total_revenue DESC;

text

**Resultado:**
category | order_count | total_revenue
Electronics | 4 | 1505

text

**Orden de ejecución:**
1. WHERE filtra (amount > 50)
2. GROUP BY agrupa
3. Agregados se calculan
4. HAVING filtra grupos

---

## Diferencias: WHERE vs HAVING vs GROUP BY

| Concepto | Cuándo | Ejemplo | Filtra |
|----------|--------|---------|--------|
| **WHERE** | Antes de agrupar | `WHERE amount > 100` | Filas individuales |
| **GROUP BY** | Agrupa filas | `GROUP BY category` | N/A (agrupa) |
| **HAVING** | Después de agrupar | `HAVING COUNT(*) > 5` | Grupos |
| **ORDER BY** | Después de todo | `ORDER BY total DESC` | Orden resultado |

---

## STRING_AGG vs GROUP_CONCAT vs LISTAGG

| BD | Función | Sintaxis |
|----|---------|----------|
| PostgreSQL | STRING_AGG | `STRING_AGG(col, ', ' ORDER BY col)` |
| MySQL | GROUP_CONCAT | `GROUP_CONCAT(col ORDER BY col SEPARATOR ', ')` |
| Oracle | LISTAGG | `LISTAGG(col, ', ') WITHIN GROUP (ORDER BY col)` |
| SQL Server | STRING_AGG | `STRING_AGG(col, ', ') WITHIN GROUP (ORDER BY col)` |

---

## Errores comunes en entrevista

- **Error**: Poner columna en SELECT sin estar en GROUP BY → **Solución**: SELECT solo: columnas de GROUP BY + agregados

- **Error**: Usar WHERE cuando necesitas HAVING → **Solución**: WHERE es pre-aggregation, HAVING es post-aggregation

- **Error**: ORDER BY posición incorrecta → **Solución**: ORDER BY va DESPUÉS de HAVING

- **Error**: Olvidar que agregados necesitan ALL datos del grupo → **Solución**: SUM() necesita acceso a todas las filas del grupo

---

## Preguntas de seguimiento típicas

1. **"¿Puedo usar HAVING sin GROUP BY?"**
   - Técnicamente sí, pero es raro. Sin GROUP BY, toda tabla = 1 grupo

2. **"¿Diferencia entre ORDER BY col vs ORDER BY 1?"**
   - `ORDER BY col` es explícito (mejor)
   - `ORDER BY 1` es posición de columna (less clear, evita)

3. **"¿Cómo hago un GROUP BY en 2+ columnas?"**
   - `GROUP BY col1, col2` — Crea grupos únicos por combinación

4. **"¿Puede GROUP BY ser en columna sin estar en SELECT?"**
   - Sí, válido: `SELECT category FROM sales GROUP BY category, customer_id` — Agrupa pero no muestra customer_id

---

## Real-World: E-Commerce Analytics

-- Top productos por categoría (últimos 30 días) con más de 10 órdenes
SELECT
category,
product,
DATE_TRUNC('month', order_date)::date as month,
COUNT() as order_count,
SUM(amount) as revenue,
AVG(amount) as avg_order_value,
STRING_AGG(DISTINCT customer_id::text, ', ') as unique_customers
FROM orders
WHERE order_date >= NOW() - INTERVAL '30 days'
GROUP BY category, product, DATE_TRUNC('month', order_date)
HAVING COUNT() > 10
ORDER BY revenue DESC
LIMIT 20;

text

---

## References

- [GROUP BY - PostgreSQL Docs](https://www.postgresql.org/docs/current/sql-select.html#SQL-GROUPBY)
- [HAVING Clause - W3Schools](https://www.w3schools.com/sql/sql_having.asp)
- [STRING_AGG - PostgreSQL](https://www.postgresql.org/docs/current/functions-aggregate.html)
- [GROUP_CONCAT - MySQL](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html#function_group-concat)

