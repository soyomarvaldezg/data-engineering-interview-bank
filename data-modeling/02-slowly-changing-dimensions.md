# Slowly Changing Dimensions (SCD)

**Tags**: #data-modeling #scd #dimensions #history #tracking #real-interview  
**Empresas**: Amazon, Google, Meta, JPMorgan, Walmart  
**Dificultad**: Senior  
**Tiempo estimado**: 25 min  

---

## TL;DR

**SCD** = how to track changes in dimensions over time. **Types**: (1) Overwrite (no history), (2) Add row (full history), (3) Add columns (recent history), (4) Hybrid. **Reality**: Most analytics = Type 2 (track everything). **Challenge**: Without SCD = wrong historical analysis ("price was $100" but history says $50).

---

## Concepto Core

- **Qué es**: Dimension attributes change (price, location, name). Need to track history
- **Por qué importa**: If you ignore changes, historical analysis is WRONG
- **Principio clave**: Design SCD upfront, not after data corrupts

---

## Problem: Why SCD Matters

Example: Product dimension

Scenario: Nike Air Max
├─ 2024-01-01: Price = $120, Color = Red
├─ 2024-06-01: Price = $99 (sale)
├─ 2024-08-01: Color = Blue (restocked)

Question: "How much did we sell Air Max for in Q2 2024?"

Without SCD (overwrite):
├─ dim_product: product_id=1, name="Air Max", price=$99, color=Blue
├─ Old prices/colors: LOST
├─ Answer: "We sold for $99" (WRONG! It was $120 in Q2)

With SCD Type 2:
├─ Row 1: product_id=1, valid_from=2024-01-01, valid_to=2024-06-01, price=$120
├─ Row 2: product_id=1, valid_from=2024-06-01, valid_to=2024-08-01, price=$99
├─ Row 3: product_id=1, valid_from=2024-08-01, valid_to=9999-12-31, price=$99
├─ Old data: PRESERVED
├─ Answer: "Q2 sales were at $120" (CORRECT!)

text

---

## SCD Type 1: Overwrite (Simple, Lossy)

CREATE TABLE dim_product_type1 (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
updated_at TIMESTAMP
);

-- Initial insert
INSERT INTO dim_product_type1
VALUES (1, 'Nike Air Max', 'Shoes', 120.00, NOW());

-- Price changes
UPDATE dim_product_type1 SET price = 99.00, updated_at = NOW() WHERE product_id = 1;
-- Old price (120) is GONE

-- Query: "What was price in Q2?"
SELECT price FROM dim_product_type1 WHERE product_id = 1;
-- Returns: 99.00 (WRONG! Q2 price was 120)

Pros:
✓ Simple (just UPDATE)
✓ Small storage

Cons:
✗ No history (cannot answer "was the price different before?")
✗ Analytics wrong
✗ Compliance nightmare

Use cases:

Only non-critical attributes (rarely used attributes)

Correctness not important

text

---

## SCD Type 2: Add Row (Full History, Recommended)

CREATE TABLE dim_product_type2 (
product_key INT PRIMARY KEY, -- Surrogate key (auto-increment)
product_id INT, -- Business key (what customers know)
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
valid_from DATE,
valid_to DATE,
is_current BOOLEAN,
INDEX idx_product_id (product_id),
INDEX idx_current (is_current)
);

-- Initial insert
INSERT INTO dim_product_type2
VALUES (
NULL, -- Auto-increment
1, -- Business key
'Nike Air Max',
'Shoes',
120.00,
'2024-01-01',
'9999-12-31',
TRUE
);
-- Result: product_key=1000, product_id=1

-- Price changes on 2024-06-01
UPDATE dim_product_type2 SET valid_to = '2024-05-31', is_current = FALSE WHERE product_key = 1000;

INSERT INTO dim_product_type2
VALUES (
NULL,
1,
'Nike Air Max',
'Shoes',
99.00,
'2024-06-01',
'9999-12-31',
TRUE
);
-- Result: product_key=1001, product_id=1

-- Query: "What was price in Q2 2024?"
SELECT price FROM dim_product_type2
WHERE product_id = 1
AND valid_from <= '2024-06-15'
AND valid_to >= '2024-06-15';
-- Returns: 120.00 (CORRECT!)

-- Query: "Current price"
SELECT price FROM dim_product_type2
WHERE product_id = 1 AND is_current = TRUE;
-- Returns: 99.00 (CORRECT!)

-- Query: "Price history"
SELECT price, valid_from, valid_to FROM dim_product_type2
WHERE product_id = 1
ORDER BY valid_from;
-- Returns: 120 (Jan-May), 99 (June+) with dates

Pros:
✓ Full history (can answer any time question)
✓ Correct analytics
✓ Compliance-friendly (audit trail)

Cons:
✗ More rows (storage overhead)
✗ Requires surrogate key (business key != PK)
✗ Queries more complex (date conditions)

Use cases:

Most analytics (default choice)

Audited industries (HIPAA, PCI, GDPR)

Price/cost sensitive analysis

text

---

## SCD Type 3: Add Columns (Recent History)

CREATE TABLE dim_product_type3 (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
price_current DECIMAL(10,2),
price_previous DECIMAL(10,2),
price_updated_date DATE,
color_current VARCHAR(50),
color_previous VARCHAR(50),
color_updated_date DATE,
updated_at TIMESTAMP
);

-- Initial
INSERT INTO dim_product_type3
VALUES (1, 'Nike Air Max', 'Shoes', 120.00, NULL, '2024-01-01', 'Red', NULL, NULL, NOW());

-- Price changes
UPDATE dim_product_type3
SET price_previous = price_current,
price_current = 99.00,
price_updated_date = NOW(),
updated_at = NOW()
WHERE product_id = 1;
-- Result: price_current=99, price_previous=120

-- Query: "Current and previous price"
SELECT price_current, price_previous FROM dim_product_type3 WHERE product_id = 1;
-- Returns: current=99, previous=120 (CORRECT for recent)

-- Query: "What was price 3 months ago?"
-- Cannot answer! Only have current + 1 previous

Pros:
✓ Balance between Type 1 and Type 2
✓ Slightly more history than Type 1
✓ Less storage than Type 2

Cons:
✗ Cannot answer "what if" for old dates
✗ Limited history (only 2 versions)
✗ Complex schema (many [field]_current, [field]_previous columns)

Use cases:

Only need recent previous value

Storage constrained

Rare

text

---

## SCD Type 4: History Table (Hybrid)

-- Current table (Type 1)
CREATE TABLE dim_product (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
updated_at TIMESTAMP
);

-- History table (Type 2 style)
CREATE TABLE dim_product_history (
product_history_id INT PRIMARY KEY,
product_id INT FK,
product_name VARCHAR(100),
category VARCHAR(50),
price DECIMAL(10,2),
valid_from DATE,
valid_to DATE
);

-- Insert/update pattern:
-- 1. Record current values to history table (before change)
-- 2. Update current table

-- Query current:
SELECT price FROM dim_product WHERE product_id = 1;

-- Query historical:
SELECT price FROM dim_product_history WHERE product_id = 1 AND valid_to >= date;

Pros:
✓ Current queries fast (no filtering on dates)
✓ History preserved (separate table)

Cons:
✗ More complex (2 tables to maintain)
✗ Application logic (need to manage history)

Use cases:

Need both fast current + historical queries

Large dimensions

text

---

## Implementation Example: Retail

-- Dimension: Store
CREATE TABLE dim_store_type2 (
store_key INT PRIMARY KEY,
store_id INT,
store_name VARCHAR(100),
city VARCHAR(50),
region VARCHAR(50),
manager_name VARCHAR(100),
square_feet INT,
valid_from DATE,
valid_to DATE,
is_current BOOLEAN,
INDEX idx_store_id (store_id),
INDEX idx_current (is_current)
);

-- Scenario: Store relocates, name changes, manager changes
-- 2024-01-01: Store opened, manager = John
-- 2024-06-01: Manager = Sarah
-- 2024-09-01: Store relocated, new city/region

-- Insert 1: Initial
INSERT INTO dim_store_type2 VALUES (
1000, 1, 'Downtown Mall Store', 'Boston', 'Northeast', 'John', 5000,
'2024-01-01', '9999-12-31', TRUE
);

-- Insert 2: Manager change (2024-06-01)
UPDATE dim_store_type2 SET valid_to = '2024-05-31', is_current = FALSE WHERE store_key = 1000;
INSERT INTO dim_store_type2 VALUES (
1001, 1, 'Downtown Mall Store', 'Boston', 'Northeast', 'Sarah', 5000,
'2024-06-01', '9999-12-31', TRUE
);

-- Insert 3: Relocation (2024-09-01)
UPDATE dim_store_type2 SET valid_to = '2024-08-31', is_current = FALSE WHERE store_key = 1001;
INSERT INTO dim_store_type2 VALUES (
1002, 1, 'Harbor Plaza Store', 'Salem', 'Northeast', 'Sarah', 7000,
'2024-09-01', '9999-12-31', TRUE
);

-- Analytics query: "Sales by manager, 2024"
SELECT
ds.manager_name,
SUM(fs.sales_amount) as total_sales
FROM fact_sales fs
JOIN dim_store_type2 ds ON fs.store_id = ds.store_id AND fs.date_id >= ds.valid_from AND fs.date_id <= ds.valid_to
WHERE YEAR(ds.valid_from) = 2024
GROUP BY ds.manager_name;
-- Correctly attributes sales to manager who was there at time

text

---

## Performance: SCD Type 2 Query Optimization

-- Slow: Multiple conditions
SELECT price FROM dim_product_type2
WHERE product_id = 1
AND valid_from <= '2024-06-15'
AND valid_to >= '2024-06-15';
-- Date filtering can be slow

-- Fast: Use is_current flag for most queries
SELECT price FROM dim_product_type2
WHERE product_id = 1 AND is_current = TRUE;
-- Index on is_current makes this fast

-- Fast: Partition by date
CREATE TABLE dim_product_type2_2024_q2 AS (
SELECT * FROM dim_product_type2
WHERE valid_from >= '2024-04-01' AND valid_to <= '2024-06-30'
);
-- Query partitioned table (smaller scans)

text

---

## Real-World: E-commerce Pricing History

Scenario: Amazon tracks product prices for analytics

Question: "Elasticity analysis: when we drop price from $100 to $80, how many more units sell?"

Without SCD: Cannot answer (only have current price)
With SCD Type 2: Full price history → can correlate price changes with sales volume

Implementation:

Nightly: Check if price changed

If changed: Close old dim_product row, open new row with new price

Analytics: JOIN fact_sales with dim_product on date range

Result: Accurate price per transaction date

text

---

## Errores Comunes en Entrevista

- **Error**: "Type 1 is sufficient" → **Solución**: Wrong! Historical analysis breaks

- **Error**: Forgetting surrogate key (use business key as PK)** → **Solución**: Business key can change, need stable PK

- **Error**: Not filtering dates in joins** → **Solución**: Join without date range = cartesian product nightmare

- **Error**: Too many Type 2 dimensions** → **Solución**: Storage explodes. Use Type 1 for rarely-changing attrs

---

## Preguntas de Seguimiento

1. **"¿Cuándo Type 1 vs Type 2?"**
   - Type 1: Non-critical (description, color)
   - Type 2: Audited (price, location, status)

2. **"¿Performance of Type 2 queries?"**
   - Slower (date filtering)
   - Mitigate: is_current flag, partitioning, indexing

3. **"¿Deletes in SCD Type 2?"**
   - Don't delete, just set valid_to to yesterday
   - Preserves history, soft delete

4. **"¿SCD for both dimensions and facts?"**
   - Dimension: Always SCD (track attribute changes)
   - Fact: Never SCD (facts don't change, only added)

---

## References

- [SCD Types - Ralph Kimball](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/slowly-changing-dimensions/)
- [SCD Implementation - Databricks](https://docs.databricks.com/solutions/data-warehousing.html)

