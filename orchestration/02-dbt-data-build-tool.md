# dbt: Data Build Tool & SQL Transformations

**Tags**: #dbt #sql-transformations #testing #lineage #analytics-engineering #real-interview

---

## TL;DR

**dbt** = "SQL-first transformation tool". **Philosophy**: Las Transformations = SQL code (versioned, tested, documented). **Core**: Models (archivos SQL), Tests (data quality), Lineage (qué depende de qué). **Benefit**: El Data modeling se vuelve version-controlled, documented, trustworthy. **Trade-off**: SQL-only (no Python), requiere un data warehouse.

---

## Concepto Core

- **Qué es**: dbt = version-control + testing + documentation para SQL transformations
- **Por qué importa**: La mayoría de las transformations son SQL. dbt las convierte en professional-grade
- **Principio clave**: "Transform in the warehouse" (no en Python ETL tools)

---

## Architecture

dbt Project Structure:
├─ dbt_project.yml (config)
├─ models/
│ ├─ staging/ (raw data, minimal transforms)
│ ├─ intermediate/ (business logic)
│ └─ marts/ (final tables, customer-facing)
├─ tests/
│ └─ data quality checks
├─ macros/
│ └─ reusable SQL functions
└─ seeds/
└─ static data (CSV → table)

Workflow:

Escribir SQL model files

dbt run: Execute models en el warehouse

dbt test: Run data quality tests

dbt docs: Generate lineage documentation

Deploy: Version control, CI/CD pipeline

---

## dbt Models

-- dbt model: models/staging/stg_customers.sql
-- (Staging layer: minimal transforms, 1:1 con la source)

{{ config(
materialized='table' -- o 'view', 'incremental'
) }}

SELECT
customer_id,
customer_name,
email,
created_at,
CURRENT_TIMESTAMP as dbt_loaded_at
FROM {{ source('raw', 'customers') }} -- Reference a la raw table
WHERE deleted_at IS NULL

-- Note: {{ }} = Jinja templating (dbt feature)
-- source('raw', 'customers') = definido en models/sources.yml

-- dbt model: models/intermediate/int_customer_orders.sql
-- (Intermediate: business logic, enrichment)

WITH customers AS (
SELECT \* FROM {{ ref('stg_customers') }}
-- ref() = reference a otro model (crea dependency)
),

orders AS (
SELECT \* FROM {{ ref('stg_orders') }}
),

customer_orders AS (
SELECT
c.customer_id,
c.customer_name,
COUNT(o.order_id) as num_orders,
SUM(o.amount) as total_spent,
MAX(o.order_date) as last_order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
)

SELECT \* FROM customer_orders

-- dbt model: models/marts/customers.sql
-- (Mart layer: final, customer-facing)

{{ config(
materialized='table',
unique_id='customer_id' -- Primary key
) }}

SELECT
customer_id,
customer_name,
email,
num_orders,
total_spent,
CASE
WHEN total_spent > 1000 THEN 'High'
WHEN total_spent > 100 THEN 'Medium'
ELSE 'Low'
END as customer_segment,
last_order_date
FROM {{ ref('int_customer_orders') }}

---

## Materialization Types

VIEW (default)
├─ SQL es ejecutado en cada query
├─ Zero storage
├─ Siempre los Latest data
└─ Uso: Intermediate joins, transformations

TABLE (materialized)
├─ SQL ejecutado una vez, resultado almacenado
├─ Storage cost
├─ Fast queries (no computation)
└─ Uso: Final marts, frequently queried

INCREMENTAL
├─ Primera ejecución: Full table
├─ Después: Solo new/updated rows
├─ Storage + speed optimized
└─ Uso: Large fact tables (daily updates)

Example:
{{ config(
materialized='incremental',
unique_id=['order_id', 'date'],
on_schema_change='fail'
) }}

SELECT
order_id,
customer_id,
amount,
order_date
FROM {{ source('raw', 'orders') }}
{% if execute %}
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
-- Solo cargar NEW orders
{% endif %}

---

## Tests & Data Quality

tests/sources.yml
version: 2

sources:

name: raw
tables:

name: customers
columns:

name: customer_id
tests:

not_null # Cannot be NULL

unique # Must be unique

name: email
tests:

not_null

unique

models/marts/schema.yml
version: 2

models:

name: customers
tests:

dbt_expectations.expect_table_row_count_to_be_between:
min_value: 100
max_value: 1000000

columns:

name: customer_id
tests:

not_null

unique

relationships:
to: ref('stg_customers')
field: customer_id

name: total_spent
tests:

accepted_values:
values: # Valid ranges

-- Custom test: tests/assert_no_null_emails.sql
-- (Test que los emails no son null)

SELECT COUNT(\*)
FROM {{ ref('customers') }}
WHERE email IS NULL

-- Si esto retorna > 0, el test FAILS

**Running tests:**
dbt test

Output:
✓ unique constraint on customers.customer_id: PASS
✓ not_null constraint on customers.email: PASS
✗ assert_no_null_emails: FAIL (10 null emails found)

---

## Lineage & Dependency Management

Dependency graph (auto-generated por dbt):

raw.customers ────┐
├──> stg_customers ──┐
raw.orders ───────┤ ├──> int_customer_orders ──┐
└──> stg_orders ─────┤ ├──> customers (MART)
└─────────────────────────┘

dbt sabe:
├─ Si la source cambia → stg_customers debe refresh
├─ Si stg_customers cambia → int_customer_orders debe refresh
├─ Si int_customer_orders cambia → customers debe refresh
└─ Auto-selects el execution order correcto

Commands:
dbt run # Run todos los models
dbt run --select customers # Run solo customers + dependencies
dbt run --select +customers # Run customers + todos los upstream
dbt run --select customers+ # Run customers + todos los downstream
dbt test # Test todos los models
dbt docs generate # Generate lineage documentation

---

## Real-World: E-commerce Data Transformation

dbt_project.yml
name: 'ecommerce_analytics'
version: '1.0.0'
config-version: 2

profile: 'ecommerce'
model-paths: ['models']
analysis-paths: ['analysis']
test-paths: ['tests']
seed-paths: ['data']

models:
ecommerce_analytics:
materialized: 'view'
staging:
materialized: 'view' # Staging son views (ephemeral)
marts:
materialized: 'table' # Marts son tables (persistent)

-- models/staging/stg_orders.sql
SELECT
order_id,
customer_id,
order_date,
total_amount as order_amount,
order_status,
created_at as order_created_at
FROM {{ source('ecommerce', 'raw_orders') }}
WHERE order_status != 'cancelled'

-- models/staging/stg_order_items.sql
SELECT
order_item_id,
order_id,
product_id,
quantity,
unit_price,
quantity \* unit_price as line_total
FROM {{ source('ecommerce', 'raw_order_items') }}

-- models/intermediate/int_order_details.sql
WITH orders AS (
SELECT \* FROM {{ ref('stg_orders') }}
),

order_items AS (
SELECT \* FROM {{ ref('stg_order_items') }}
),

products AS (
SELECT \* FROM {{ ref('stg_products') }}
)

SELECT
o.order_id,
o.customer_id,
oi.order_item_id,
oi.product_id,
p.product_name,
p.category,
oi.quantity,
oi.unit_price,
oi.line_total,
o.order_date
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id

-- models/marts/fct_orders.sql (Fact table)
{{ config(
materialized='table',
unique_id='order_id'
) }}

SELECT
order_id,
customer_id,
SUM(line_total) as order_total,
COUNT(order_item_id) as num_items,
MIN(order_date) as order_date
FROM {{ ref('int_order_details') }}
GROUP BY order_id, customer_id, order_date

-- models/marts/dim_customers.sql (Dimension table)
{{ config(
materialized='table',
unique_id='customer_id'
) }}

WITH customers AS (
SELECT \* FROM {{ ref('stg_customers') }}
),

order_summary AS (
SELECT
customer_id,
COUNT(order_id) as lifetime_orders,
SUM(order_total) as lifetime_value
FROM {{ ref('fct_orders') }}
GROUP BY customer_id
)

SELECT
c.customer_id,
c.customer_name,
c.email,
COALESCE(o.lifetime_orders, 0) as lifetime_orders,
COALESCE(o.lifetime_value, 0) as lifetime_value
FROM customers c
LEFT JOIN order_summary o ON c.customer_id = o.customer_id

**Execution:**
dbt run

Runs en dependency order:

1. stg_customers, stg_orders, stg_order_items, stg_products (staging views)
2. int_order_details (intermediate)
3. fct_orders, dim_customers (marts/tables)
   dbt test

Validates: No nulls en PKs, relationships intact, row counts healthy
dbt docs generate

Creates lineage diagram + documentation portal

---

## dbt + Airflow Integration

Airflow DAG that runs dbt
from airflow import DAG
from cosmos import DbtTaskGroup
from datetime import datetime

with DAG(
'dbt_orchestration',
schedule_interval='0 1 \* \* \*',
start_date=datetime(2024, 1, 1)
) as dag:

# Run dbt project en Airflow

dbt_run = DbtTaskGroup(
group_id='transform',
dbt_project_path='/path/to/dbt_project',
dbt_profiles_dir='/path/to/.dbt',
dbt_command='dbt run --select +fct_orders'
)

# Test después del transform

dbt_test = DbtTaskGroup(
group_id='test',
dbt_project_path='/path/to/dbt_project',
dbt_command='dbt test'
)

dbt_run >> dbt_test
Airflow orchestrates dbt models (task scheduling + error handling)
dbt handles SQL transformation + testing

---

## Errores Comunes en Entrevista

- **Error**: "dbt is for SQL only" → **Solución**: True, pero el 80% de los transforms SON SQL

- **Error**: Usar dbt sin version control → **Solución**: dbt brilla con Git (CI/CD, code review)

- **Error**: No data quality tests → **Solución**: dbt tests detectan bugs ANTES de que lleguen a analytics

- **Error**: Mezclar Python ETL + dbt → **Solución**: Usar dbt para SQL transforms, Python solo para complex logic

---

## Preguntas de Seguimiento

1.  **"¿dbt vs Spark SQL?"**
    - dbt: SQL-first, testing, docs, warehouse-native
    - Spark: Distributed, complex logic, no SQL-only

2.  **"¿View vs Table en dbt?"**
    - View: No storage, latest always
    - Table: Storage cost, fast queries
    - Elegir basado en size + query frequency

3.  **"¿Incremental models?"**
    - Full table en la primera ejecución
    - Append solo new rows en cada refresh
    - Critical para large tables (billions of rows)

4.  **"¿dbt project structure?"**
    - staging/ (minimal transforms)
    - intermediate/ (business logic)
    - marts/ (final tables)
    - Mantiene el code organized + debuggable

---

## References

- [dbt Official Docs](https://docs.getdbt.com/)
- [dbt Best Practices](https://docs.getdbt.com/guides/best-practices)
- [dbt + Airflow Integration](https://cosmos.astronomer.io/)
