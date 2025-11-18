# Normalización vs Desnormalización: Compensaciones

**Tags**: #data-modeling #normalization #denormalization #oltp #olap #design-decisions #real-interview

---

## TL;DR

**Normalization** (OLTP) = estructurada, elimina redundancia, previene anomalías, optimizada para escritura. **Denormalization** (OLAP) = datos duplicados, lecturas rápidas, optimizada para análisis. **Realidad**: OLTP = normalizado (bases de datos de producción), OLAP = desnormalizado (data warehouses). Elige según la carga de trabajo: transacciones → normaliza, analítica → desnormaliza.

---

## Concepto Core

- **Qué es**: Normalization = organizar para eliminar redundancia. Denormalization = duplicar estratégicamente para velocidad.
- **Por qué importa**: La elección equivocada produce escrituras lentas o consultas lentas (malo en cualquier caso).
- **Principio clave**: Optimiza para tu carga de trabajo (transaccional vs analítica).

---

## Normalization (OLTP - Online Transactional Processing)

### Normal Forms

**1NF (First Normal Form)**:  
├─ Valores atómicos (sin arrays/listas)  
├─ No grupos repetitivos  
└─ Cada fila única (PK definido)

**Ejemplo:**  
❌ MAL: `customer_id, name, phone_list=[555-1234, 555-5678]`  
✅ BIEN: `customer_id, name, phone_number` (filas separadas si hay varios)

**2NF (Second Normal Form)**:  
├─ Ya en 1NF  
├─ Sin dependencias parciales  
└─ Todos los atributos no‑clave dependen del PK completo

**Ejemplo (PK compuesto):**  
❌ MAL: `(student_id, course_id, professor_name)`  
└─ `professor_name` depende solo de `course_id`  
✅ BIEN: dividir en 2 tablas  
└─ `courses(course_id, professor_name)`  
└─ `enrollments(student_id, course_id FK)`

**3NF (Third Normal Form)**:  
├─ Ya en 2NF  
├─ Sin dependencias transitivas  
└─ Los atributos no‑clave dependen solo del PK

**Ejemplo:**  
❌ MAL: `customer(customer_id, name, city, state, zip_code)`  
└─ `zip_code` determina `state` (transitiva: `customer_id → zip_code → state`)  
✅ BIEN: dividir en 2 tablas  
└─ `customers(customer_id, name, zip_code FK)`  
└─ `zip_codes(zip_code, city, state)`

**BCNF (Boyce‑Codd Normal Form)**:  
├─ Más estricto que 3NF  
├─ Para cada FD, el LHS debe ser clave candidata  
└─ Rara vez necesario (3NF suele ser suficiente)

### Normalized Schema Example

```sql
-- NORMALIZED (3NF)
-- Multiple tables, no redundancy, write‑efficient

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100),
    zip_code INT FK
);

CREATE TABLE zip_codes (
    zip_code INT PRIMARY KEY,
    city VARCHAR(50),
    state VARCHAR(50)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT FK,
    order_date DATE,
    total_amount DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT FK,
    product_id INT FK,
    quantity INT,
    unit_price DECIMAL(10,2)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category_id INT FK
);

-- Query: "Get customer name, city, total orders"
SELECT
    c.name,
    z.city,
    COUNT(o.order_id) AS num_orders,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN zip_codes z ON c.zip_code = z.zip_code
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;

-- Requires 4 JOINs (performance trade‑off for normalization)
-- But: Insert/update/delete is clean (one place to change data)
```

### Beneficios de la Normalización

✓ Sin redundancia (actualizar precio una vez, se actualiza en todas partes)  
✓ Sin anomalías (inserción, actualización, eliminación seguras)  
✓ Integridad referencial forzada (FKs aseguran consistencia)  
✓ Eficiente para escritura (inserciones/actualizaciones rápidas)  
✓ Almacenamiento eficiente (sin duplicados)

✗ Consultas intensivas (muchos JOINs = lecturas lentas)  
✗ Consultas complejas (múltiples tablas)

---

## Denormalization (OLAP - Online Analytical Processing)

### Denormalized Schema Example

```sql
-- DENORMALIZED (Star Schema)
-- Duplicate data, fast reads, analysis‑optimized

CREATE TABLE fact_customer_orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),      -- DUPLICATE (from customers table)
    city VARCHAR(50),                -- DUPLICATE (from zip_codes table)
    state VARCHAR(50),               -- DUPLICATE
    product_id INT,
    product_name VARCHAR(100),       -- DUPLICATE
    product_price DECIMAL(10,2),     -- DUPLICATE
    category_id INT,
    category_name VARCHAR(50),       -- DUPLICATE
    quantity INT,
    unit_price DECIMAL(10,2),
    order_date DATE,
    total_amount DECIMAL(10,2)
);

-- Query: "Get customer name, city, total orders"
SELECT
    customer_name,
    city,
    COUNT(*) AS num_orders,
    SUM(total_amount) AS total_spent
FROM fact_customer_orders
GROUP BY customer_id;

-- Single table scan (no JOINs!)
-- Fast (queries run in seconds vs minutes)
-- But: Redundancy (customer_name repeated for every order)
```

### Beneficios de la Desnormalización

✓ Consultas rápidas (menos JOINs o ninguno)  
✓ Consultas más simples (parece una sola tabla)  
✓ Optimizado para lectura (consultas de análisis rápidas)  
✓ Pre-agregaciones (sumas parciales almacenadas)

✗ Redundancia (mismos datos en múltiples lugares)  
✗ Anomalías de actualización (cambiar precio en un lugar, roto en otro)  
✗ Inflación de almacenamiento (datos duplicados)  
✗ Más difícil de mantener (ETL complejo para desnormalizar)

---

## Decision Matrix

| Escenario                        | Normalizar             | Desnormalizar                |
| -------------------------------- | ---------------------- | ---------------------------- |
| **Base de datos de producción**  | ✅ SÍ                  | ❌ NO                        |
| **Data warehouse**               | ❌ NO                  | ✅ SÍ                        |
| **Transaccional en tiempo real** | ✅ SÍ                  | ❌ NO                        |
| **Analítica por lotes**          | ❌ NO                  | ✅ SÍ                        |
| **Bajo volumen de escritura**    | ❌ OK cualquiera       | ✅ SÍ (lecturas más rápidas) |
| **Alto volumen de escritura**    | ✅ SÍ                  | ❌ NO                        |
| **Conjunto pequeño**             | ✅ SÍ (más simple)     | ❌ OK cualquiera             |
| **Conjunto grande**              | ✅ SÍ (ahorra storage) | ✅ SÍ (velocidad)            |
| **Consultas complejas**          | ❌ Difícil             | ✅ Fácil                     |
| **Frecuencia de actualización**  | ✅ Mejor               | ❌ Pesadilla                 |

---

## Real‑World: E‑commerce Architecture

**PRODUCTION DATABASE (OLTP - NORMALIZED):**  
├─ `customers (customer_id, name, email)`  
├─ `products (product_id, name, price)`  
├─ `orders (order_id, customer_id, order_date)`  
├─ `order_items (order_item_id, order_id, product_id, quantity)`  
└─ Todo normalizado (3NF)

**Daily ETL Pipeline:**  
├─ Read from OLTP (normalized)  
├─ Transform: Denormalize into warehouse schema  
└─ Write to OLAP (Star Schema, denormalized)

**ANALYTICS WAREHOUSE (OLAP - DENORMALIZED):**  
├─ `fact_customer_orders` (una fila por orden, desnormalizada)  
├─ `dim_customer` (atributos del cliente duplicados en facts)  
├─ `dim_product` (atributos del producto duplicados en facts)  
└─ Todo desnormalizado (redundancia estratégica)

**Patrón de consultas:**  
❌ Consultas OLTP (app de producción):

```sql
SELECT * FROM orders WHERE customer_id = 123;
```

→ Normalizado, búsqueda PK rápida

✅ Consultas OLAP (analytics):

```sql
SELECT customer_name, product_name, SUM(amount)
FROM fact_customer_orders
GROUP BY customer_id, product_id;
```

→ Desnormalizado, agregación rápida

---

## Partial Denormalization (Middle Ground)

```sql
-- Not fully normalized, not fully denormalized

CREATE TABLE orders_enriched (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),   -- One‑level denorm
    customer_city VARCHAR(50),    -- One‑level denorm
    order_date DATE,
    total_amount DECIMAL(10,2)
);

CREATE TABLE order_items_enriched (
    order_item_id INT PRIMARY KEY,
    order_id INT FK,
    product_id INT,
    product_name VARCHAR(100),    -- One‑level denorm
    product_category VARCHAR(50), -- One‑level denorm
    quantity INT,
    unit_price DECIMAL(10,2)
);

-- Benefit: Faster queries than normalized, but not as much redundancy as full denorm
-- Use case: When full denormalization is overkill, but pure normalization too slow
```

---

## Materialized Views (Smart Denormalization)

```sql
-- Instead of storing denormalized table, use materialized view

CREATE MATERIALIZED VIEW customer_order_summary AS
SELECT
    c.customer_id,
    c.name AS customer_name,
    z.city,
    COUNT(o.order_id) AS num_orders,
    SUM(o.total_amount) AS total_spent,
    MAX(o.order_date) AS last_order_date
FROM customers c
JOIN zip_codes z ON c.zip_code = z.zip_code
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;

-- Query against materialized view (looks like table, executes fast)
SELECT * FROM customer_order_summary WHERE customer_id = 123;

-- Refresh strategy:
-- Option 1: Nightly REFRESH MATERIALIZED VIEW (full recompute)
-- Option 2: Incremental refresh (only changed rows)
-- Option 3: Query rewrite (transparent to user)

Benefits:
✓ Denormalization benefits (speed)
✓ Normalized source data (correct updates)
✓ Automatic refresh (can schedule)
✓ Transparent to users
```

---

## Performance Comparison

**NORMALIZED (OLTP):**

- Insert 1 order: 4 inserts (`customer`, `order`, `order_items`, `products`) → Tiempo ≈ 100 ms
- Query "Customer total spent": 4 JOINs → Tiempo 5‑10 s (con 1 B filas)

**DENORMALIZED (OLAP):**

- Insert 1 order: 1 insert (fact order, todos los atributos) → Tiempo ≈ 50 ms (más rápido, pero storage bloat)
- Query "Customer total spent": 1 scan de tabla, GROUP BY → Tiempo 1‑2 s (≈ 100× más rápido)

**Conclusión:**

- OLTP (producción): **Normalize** (correcto, escrituras eficientes)
- OLAP (warehouse): **Denormalize** (lecturas rápidas, aceptamos redundancia)

---

## Errores Comunes en Entrevista

- **Error**: "La normalización siempre es buena" → **Solución**: No para analytics (las consultas son demasiado lentas)
- **Error**: "La desnormalización desperdicia espacio" → **Solución**: Verdadero, pero la velocidad de consulta importa más en el warehouse
- **Error**: Desnormalizar la base de datos de producción → **Solución**: ¡Nunca! Mantén OLTP normalizado, desnormaliza en el warehouse
- **Error**: Sobre-normalizar warehouse (demasiados JOINs) → **Solución**: Desnormaliza para velocidad

---

## Preguntas de Seguimiento

1. **"¿3NF vs Desnormalizado: cuándo cada una?"**
   - 3NF: Producción (escrituras, consistencia)
   - Desnormalizado: Warehouse (lecturas, velocidad)

2. **"Materialized views vs tables?"**
   - Views: Lógicas (siempre frescas desde la fuente)
   - Materialized: Físicas (pre‑calculadas, refrescar manualmente)

3. **"¿Cómo desnormalizar de forma segura?"**
   - Proceso ETL (source normalizado → target desnormalizado)
   - Nunca actualizaciones directas en la tabla desnormalizada
   - Queries de validación (desnorm == original al agregar)

4. **"¿Actualizar una columna desnormalizada?"**
   - Peligroso (inconsistencia)
   - Solución: Reconstruir desde la fuente normalizada (ETL nocturno)

---

## References

- [Database Normalization - Wikipedia](https://en.wikipedia.org/wiki/Database_normalization)
- [OLTP vs OLAP - Microsoft](https://learn.microsoft.com/en-us/azure/architecture/data-guide/relational-data/online-analytical-processing)
