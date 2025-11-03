# Normalization vs Denormalization: Trade-offs

**Tags**: #data-modeling #normalization #denormalization #oltp #olap #design-decisions #real-interview  
**Empresas**: Amazon, Google, Meta, Stripe, JPMorgan  
**Dificultad**: Senior  
**Tiempo estimado**: 18 min  

---

## TL;DR

**Normalization** (OLTP) = structured, eliminate redundancy, prevent anomalies, write-optimized. **Denormalization** (OLAP) = duplicate data, fast reads, analysis-optimized. **Reality**: OLTP = normalized (production databases), OLAP = denormalized (data warehouses). Choose based on workload: transactions → normalize, analytics → denormalize.

---

## Concepto Core

- **Qué es**: Normalization = organize to eliminate redundancy. Denormalization = duplicate strategically for speed
- **Por qué importa**: Wrong choice = either slow writes or slow queries (bad either way)
- **Principio clave**: Optimize for your workload (transactional vs analytical)

---

## Normalization (OLTP - Online Transactional Processing)

### Normal Forms

1NF (First Normal Form):
├─ Atomic values (no arrays/lists)
├─ No repeating groups
└─ Each row unique (PK defined)

Example:
❌ BAD: customer_id, name, phone_list=[555-1234, 555-5678]
✅ GOOD: customer_id, name, phone_number (separate rows if multiple)

2NF (Second Normal Form):
├─ Already 1NF
├─ No partial dependencies
└─ All non-key attrs depend on entire PK

Example (Composite PK):
❌ BAD: (student_id, course_id, professor_name)
└─ professor_name depends on course_id only, not (student_id, course_id)
✅ GOOD: Split into 2 tables
└─ courses(course_id, professor_name)
└─ enrollments(student_id, course_id FK)

3NF (Third Normal Form):
├─ Already 2NF
├─ No transitive dependencies
└─ Non-key attrs depend only on PK

Example:
❌ BAD: customer(customer_id, name, city, state, zip_code)
└─ zip_code determines state (transitive: customer_id → zip_code → state)
✅ GOOD: Split into 2 tables
└─ customers(customer_id, name, zip_code FK)
└─ zip_codes(zip_code, city, state)

BCNF (Boyce-Codd Normal Form):
├─ Even stricter than 3NF
├─ For every FD: LHS must be candidate key
└─ Rare to need (3NF usually sufficient)

text

### Normalized Schema Example

-- NORMALIZED (3NF)
-- Multiple tables, no redundancy, write-efficient

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
COUNT(o.order_id) as num_orders,
SUM(o.total_amount) as total_spent
FROM customers c
JOIN zip_codes z ON c.zip_code = z.zip_code
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id;

-- Requires 4 JOINs (performance trade-off for normalization)
-- But: Insert/update/delete is clean (one place to change data)

text

### Benefits of Normalization

✓ No redundancy (update price once, everywhere updates)
✓ No anomalies (insertion, update, deletion safe)
✓ Enforced referential integrity (FKs ensure consistency)
✓ Write-efficient (inserts/updates fast)
✓ Storage efficient (no duplicates)

✗ Query-heavy (many JOINs = slow reads)
✗ Complex queries (multiple tables)

text

---

## Denormalization (OLAP - Online Analytical Processing)

### Denormalized Schema Example

-- DENORMALIZED (Star Schema)
-- Duplicate data, fast reads, analysis-optimized

CREATE TABLE fact_customer_orders (
order_id INT PRIMARY KEY,
customer_id INT,
customer_name VARCHAR(100), -- DUPLICATE (from customers table)
city VARCHAR(50), -- DUPLICATE (from zip_codes table)
state VARCHAR(50), -- DUPLICATE
product_id INT,
product_name VARCHAR(100), -- DUPLICATE
product_price DECIMAL(10,2), -- DUPLICATE
category_id INT,
category_name VARCHAR(50), -- DUPLICATE
quantity INT,
unit_price DECIMAL(10,2),
order_date DATE,
total_amount DECIMAL(10,2)
);

-- Query: "Get customer name, city, total orders"
SELECT
customer_name,
city,
COUNT(*) as num_orders,
SUM(total_amount) as total_spent
FROM fact_customer_orders
GROUP BY customer_id;

-- Single table scan (no JOINs!)
-- Fast (queries run in seconds vs minutes)
-- But: Redundancy (customer_name repeated for every order)

text

### Benefits of Denormalization

✓ Fast queries (fewer JOINs or none)
✓ Simpler queries (looks like single table)
✓ Read-optimized (analysis queries fast)
✓ Pre-aggregations (partial sums stored)

✗ Redundancy (same data multiple places)
✗ Update anomalies (change price in one place, broken elsewhere)
✗ Storage bloat (duplicated data)
✗ Harder to maintain (complex ETL to denormalize)

text

---

## Decision Matrix

| Scenario | Normalize | Denormalize |
|----------|-----------|-------------|
| **Production database** | ✅ YES | ❌ NO |
| **Data warehouse** | ❌ NO | ✅ YES |
| **Real-time transactional** | ✅ YES | ❌ NO |
| **Batch analytics** | ❌ NO | ✅ YES |
| **Low write volume** | ❌ OK either | ✅ YES (faster reads) |
| **High write volume** | ✅ YES | ❌ NO |
| **Small dataset** | ✅ YES (simpler) | ❌ OK either |
| **Large dataset** | ✅ YES (storage) | ✅ YES (speed) |
| **Complex queries** | ❌ Hard | ✅ Easy |
| **Update frequency** | ✅ Better | ❌ Nightmare |

---

## Real-World: E-commerce Architecture

PRODUCTION DATABASE (OLTP - NORMALIZED):
├─ customers (customer_id, name, email)
├─ products (product_id, name, price)
├─ orders (order_id, customer_id, order_date)
├─ order_items (order_item_id, order_id, product_id, quantity)
└─ All normalized (3NF)

Daily ETL Pipeline:
├─ Read from OLTP (normalized)
├─ Transform: Denormalize into warehouse schema
└─ Write to OLAP (Star Schema, denormalized)

ANALYTICS WAREHOUSE (OLAP - DENORMALIZED):
├─ fact_customer_orders (one-row-per-order, denormalized)
├─ dim_customer (customer attrs duplicated in facts)
├─ dim_product (product attrs duplicated in facts)
└─ All denormalized (strategic redundancy)

Query Pattern:
❌ OLTP queries (production app):
SELECT * FROM orders WHERE customer_id = 123;
→ Normalized, fast PK lookup

✅ OLAP queries (analytics):
SELECT customer_name, product_name, SUM(amount)
FROM fact_customer_orders
GROUP BY customer_id, product_id;
→ Denormalized, fast aggregation

text

---

## Partial Denormalization (Middle Ground)

-- Not fully normalized, not fully denormalized

CREATE TABLE orders_enriched (
order_id INT PRIMARY KEY,
customer_id INT,
customer_name VARCHAR(100), -- One-level denorm
customer_city VARCHAR(50), -- One-level denorm
order_date DATE,
total_amount DECIMAL(10,2)
);

CREATE TABLE order_items_enriched (
order_item_id INT PRIMARY KEY,
order_id INT FK,
product_id INT,
product_name VARCHAR(100), -- One-level denorm
product_category VARCHAR(50), -- One-level denorm
quantity INT,
unit_price DECIMAL(10,2)
);

-- Benefit: Faster queries than normalized, but not as much redundancy as full denorm
-- Use case: When full denormalization is overkill, but pure normalization too slow

text

---

## Materialized Views (Smart Denormalization)

-- Instead of storing denormalized table, use materialized view

CREATE MATERIALIZED VIEW customer_order_summary AS
SELECT
c.customer_id,
c.name as customer_name,
z.city,
COUNT(o.order_id) as num_orders,
SUM(o.total_amount) as total_spent,
MAX(o.order_date) as last_order_date
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

text

---

## Performance Comparison

NORMALIZED (OLTP):

Insert 1 order: 4 inserts (customer, order, order_items, products)
└─ Time: ~100ms

Query "Customer total spent": 4 JOINs
└─ Time: 5-10 seconds (if 1B rows)

DENORMALIZED (OLAP):

Insert 1 order: 1 insert (fact_order, all attrs)
└─ Time: ~50ms (faster, but storage bloat)

Query "Customer total spent": 1 table scan, GROUP BY
└─ Time: 1-2 seconds (100x faster!)

Conclusion:

OLTP (production): Normalize (correct, efficient writes)

OLAP (warehouse): Denormalize (fast reads, accepts redundancy)

text

---

## Errores Comunes en Entrevista

- **Error**: "Normalization is always good" → **Solución**: Not for analytics (queries too slow)

- **Error**: "Denormalization wastes space" → **Solución**: True, but query speed matters more in warehouse

- **Error**: Denormalizing production database → **Solución**: Never! Keep OLTP normalized, denormalize in warehouse

- **Error**: Over-normalizing warehouse (too many JOINs)** → **Solución**: Denormalize for speed

---

## Preguntas de Seguimiento

1. **"¿3NF vs Denormalized: when each?"**
   - 3NF: Production (writes, correctness)
   - Denormalized: Warehouse (reads, speed)

2. **"¿Materialized views vs tables?"**
   - Views: Logical (always fresh from source)
   - Materialized: Physical (pre-computed, refresh manually)

3. **"¿How to denormalize safely?"**
   - ETL process (normalize source → denormalize target)
   - Never direct updates
   - Validation queries (denorm == original when aggregated)

4. **"¿Update a denormalized column?"**
   - Dangerous (inconsistency)
   - Solution: Rebuild from normalized source (ETL, nightly)

---

## References

- [Database Normalization - Wikipedia](https://en.wikipedia.org/wiki/Database_normalization)
- [OLTP vs OLAP - Microsoft](https://learn.microsoft.com/en-us/azure/architecture/data-guide/relational-data/online-analytical-processing)
- [Denormalization for Analytics - Kimball](https://www.kimballgroup.com/)

