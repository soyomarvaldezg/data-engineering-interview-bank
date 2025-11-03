# Star Schema & Dimensional Modeling

**Tags**: #data-modeling #star-schema #dimensional #facts #dimensions #real-interview  
**Empresas**: Amazon, Google, Meta, Stripe, Airbnb  
**Dificultad**: Mid  
**Tiempo estimado**: 20 min  

---

## TL;DR

**Star Schema** = fact table (numbers) + dimension tables (descriptors). **Design principle**: Fact table is narrow, dimensions wide. **Benefit**: Queries are fast (pre-joined), dimensions are reusable (conformed). **Trade-off**: Denormalized (more storage) vs fast queries. Best practice: Fact grain = one row per transaction/event.

---

## Concepto Core

- **Qué es**: Star Schema = join of fact + dimensions (looks like star in diagram)
- **Por qué importa**: Analytics queries need this structure (queries 100x faster)
- **Principio clave**: Fact = what you measure (sales), Dimension = how you slice (time, store, product)

---

## Cómo explicarlo en entrevista

**Paso 1**: "Star Schema = fact + dimensions. Fact = numbers (sales, count), Dimension = descriptors (date, store, customer)"

**Paso 2**: "Fact table grain = granularity (one row per sale, not per product type)"

**Paso 3**: "Dimensions are conformed (reused across fact tables)"

**Paso 4**: "Performance: Pre-denormalized = fast queries, some storage overhead"

---

## Diagrama

text
             DIM_DATE
            ├─ date_id (PK)
            ├─ date
            ├─ month
            └─ year
                  ▲
                  │
DIMCUSTOMER ◄────── FACT_SALES ──────► DIM_PRODUCT
├─ customer_id (PK) ├─ sales_id (PK) ├─ product_id (PK)
├─ name ├─ customer_id (FK)├─ name
├─ segment ├─ product_id (FK) ├─ category
└─ country ├─ store_id (FK) └─ price
├─ date_id (FK)
├─ qty
├─ amount
└─ profit
▲
│
DIM_STORE
├─ store_id (PK)
├─ store_name
├─ city
└─ region

text

---

## SQL: Star Schema Tables

-- FACT TABLE (Core)
CREATE TABLE fact_sales (
sales_id BIGINT PRIMARY KEY,
customer_id INT NOT NULL,
product_id INT NOT NULL,
store_id INT NOT NULL,
date_id INT NOT NULL,
quantity INT,
sales_amount DECIMAL(10,2),
discount DECIMAL(10,2),
profit DECIMAL(10,2),
FOREIGN KEY (customer_id) REFERENCES dim_customer,
FOREIGN KEY (product_id) REFERENCES dim_product,
FOREIGN KEY (store_id) REFERENCES dim_store,
FOREIGN KEY (date_id) REFERENCES dim_date
);

-- DIMENSIONS (Descriptive)
CREATE TABLE dim_customer (
customer_id INT PRIMARY KEY,
customer_name VARCHAR(100),
segment VARCHAR(50), -- Premium, Standard
country VARCHAR(50),
created_at TIMESTAMP
);

CREATE TABLE dim_product (
product_id INT PRIMARY KEY,
product_name VARCHAR(100),
category VARCHAR(50),
subcategory VARCHAR(50),
price DECIMAL(10,2)
);

CREATE TABLE dim_store (
store_id INT PRIMARY KEY,
store_name VARCHAR(100),
city VARCHAR(50),
region VARCHAR(50),
country VARCHAR(50)
);

CREATE TABLE dim_date (
date_id INT PRIMARY KEY, -- 20240115
date DATE,
month INT,
month_name VARCHAR(10),
quarter INT,
year INT,
day_name VARCHAR(10)
);

-- INDEXES (Critical for query performance)
CREATE INDEX idx_fact_date ON fact_sales(date_id);
CREATE INDEX idx_fact_store ON fact_sales(store_id);
CREATE INDEX idx_fact_customer ON fact_sales(customer_id);
CREATE INDEX idx_fact_product ON fact_sales(product_id);

text

---

## Queries: Star Schema Benefits

-- Query 1: Sales by category, month (simple join)
SELECT
d.year,
d.month_name,
p.category,
SUM(f.quantity) as qty_sold,
SUM(f.sales_amount) as revenue,
AVG(f.profit) as avg_profit
FROM fact_sales f
JOIN dim_product p ON f.product_id = p.product_id
JOIN dim_date d ON f.date_id = d.date_id
WHERE d.year = 2024
GROUP BY d.year, d.month_name, p.category
ORDER BY revenue DESC;

-- Query 2: Top stores by region
SELECT
s.region,
s.store_name,
SUM(f.sales_amount) as total_sales,
COUNT(DISTINCT f.customer_id) as unique_customers
FROM fact_sales f
JOIN dim_store s ON f.store_id = s.store_id
WHERE f.date_id >= 20240101
GROUP BY s.region, s.store_name
ORDER BY total_sales DESC;

-- Performance: Index on fact_sales(date_id) helps (partition pruning if partitioned)
-- Execution time: <1 second (for 500M rows)

text

---

## Fact Table: Grain

**Grain** = level of detail (one row represents what?)

Example: Sales transactions

Grain Option 1: One row per TRANSACTION
├─ sales_id, customer_id, product_id, amount
├─ Pros: Most detailed, can answer any question
├─ Cons: Huge table (1B+ rows for 1 year)
├─ Best for: Detailed analytics

Grain Option 2: One row per PRODUCT PER TRANSACTION
├─ sales_id, customer_id, product_id, product_qty, product_amount
├─ Pros: Still detailed, slightly smaller
├─ Cons: Still large

Grain Option 3: One row per CUSTOMER PER DAY
├─ customer_id, date_id, total_qty, total_amount
├─ Pros: Smaller (1M rows instead of 1B)
├─ Cons: Lost product detail
├─ Best for: Customer analytics (churn, LTV)

Wrong grain = wrong answers!

Example error:

What if grain is PER DAY (one row per customer per day)?
SELECT customer_id, SUM(sales_amount)
FROM fact_sales
GROUP BY customer_id;

WRONG: This sums sales_amount multiple times if multiple products on same day
Correct: Define grain clearly
Grain = one row per (customer, product, date)
text

---

## Conformed Dimensions

**Conformed** = same dimension reused across multiple fact tables

Example: Customer dimension

Fact Table 1: fact_sales
├─ customer_id (FK to dim_customer)
├─ ...

Fact Table 2: fact_returns
├─ customer_id (FK to dim_customer)
├─ ...

Fact Table 3: fact_customer_support_calls
├─ customer_id (FK to dim_customer)
├─ ...

Benefit: "Conformed dimension"
├─ Same customer_id definition everywhere
├─ Query across fact tables is consistent
├─ Cannot have conflicting data
└─ One source of truth for customer attributes

vs Non-conformed:
├─ fact_sales uses "cust_id"
├─ fact_returns uses "customer_id"
├─ fact_calls uses "caller_id"
├─ Nightmare: Different definitions, cannot easily join

Conformed Dimension Rules:

PK is unique across all fact tables

Attributes are consistent

No fact-table-specific data

Updates to dimension affect all fact tables uniformly

text

---

## Degenerate Dimensions

**Degenerate** = dimension data stored in fact table (not separate table)

Example: Order number, Invoice number

Normal: Separate dimension table
├─ dim_order (order_id, order_number, order_date, order_status)
├─ fact_sales (order_id FK, amount)

Degenerate: In fact table directly
├─ fact_sales (order_number, order_date, order_status, amount)
├─ No separate dim_order table

When to use:
✓ If dimension has only 1-2 attributes
✓ If never referenced by other fact tables
✓ If unique per fact row

Example:
CREATE TABLE fact_orders_degenerate (
order_id INT PRIMARY KEY,
order_number VARCHAR(50), -- Degenerate (no separate table)
customer_id INT FK,
date_id INT FK,
amount DECIMAL
);

Benefit: Slightly smaller storage, simpler schema

text

---

## Slowly Changing Dimensions Preview (Covered Next)

Problem: Dimension attributes change over time
├─ Customer name changes (marriage)
├─ Store location changes (relocated)
├─ Product price changes
└─ How to track history?

Solutions (SCD Types):

Type 1: Overwrite (lose history)

Type 2: Add new row (keep history)

Type 3: Add columns (limited history)

text

---

## Real-World: E-commerce Data Warehouse

-- Fact: Orders
CREATE TABLE fact_orders (
order_id BIGINT PRIMARY KEY,
customer_id INT FK,
product_id INT FK,
store_id INT FK,
date_id INT FK,
quantity INT,
unit_price DECIMAL(10,2),
total_amount DECIMAL(10,2),
discount DECIMAL(10,2),
tax DECIMAL(10,2),
profit DECIMAL(10,2)
);

-- Fact: Order Line Items (separate grain)
CREATE TABLE fact_order_line_items (
line_item_id BIGINT PRIMARY KEY,
order_id BIGINT FK,
product_id INT FK,
quantity INT,
unit_price DECIMAL(10,2),
line_total DECIMAL(10,2)
);

-- Fact: Customer Activity
CREATE TABLE fact_customer_activity (
activity_id BIGINT PRIMARY KEY,
customer_id INT FK,
date_id INT FK,
num_page_views INT,
num_clicks INT,
session_duration_sec INT,
purchase_amount DECIMAL(10,2)
);

-- Queries:
SELECT
d.month_name,
SUM(o.total_amount) as revenue,
SUM(o.profit) as profit,
COUNT(DISTINCT o.customer_id) as unique_customers,
AVG(o.total_amount) as avg_order
FROM fact_orders o
JOIN dim_date d ON o.date_id = d.date_id
WHERE d.year = 2024
GROUP BY d.month_name;

-- All queries use same pattern: fact + dimensions (conformed)

text

---

## Errores Comunes en Entrevista

- **Error**: "Grain is ambiguous" → **Solución**: ALWAYS define grain explicitly

- **Error**: Mixing grains in one table → **Solución**: Wrong aggregations = wrong answers

- **Error**: No conformed dimensions → **Solución**: Different fact tables cannot be compared

- **Error**: Fact table too wide (1000+ columns)** → **Solución**: Break into multiple smaller facts

---

## Preguntas de Seguimiento

1. **"¿Cómo defines grain?"**
   - Look at one row: what does it represent?
   - One sale? One product in sale? One customer per day?

2. **"¿Denormalized vs normalized trade-offs?"**
   - Star (denormalized): Fast queries, more storage
   - Normalized: Slow queries, less storage
   - Data warehouse: Choose star (queries > storage cost)

3. **"¿Conformed dimensions cost?"**
   - Upfront: Design effort, coordination
   - Payoff: Faster development, consistent analytics

4. **"¿Fact table indexing?"**
   - Indexes on ALL foreign keys (date, store, customer, product)
   - Plus composite index on common filters

---

## References

- [Dimensional Modeling - Ralph Kimball](https://www.kimballgroup.com/)
- [Star Schema - Wikipedia](https://en.wikipedia.org/wiki/Star_schema)
- [Fact and Dimension Tables](https://en.wikipedia.org/wiki/Fact_table)

