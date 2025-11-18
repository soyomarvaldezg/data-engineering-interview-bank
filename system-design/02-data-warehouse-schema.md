# Diseñar Data Warehouse: Esquema Estrella y Modelado Dimensional

**Tags**: #system-design #data-warehouse #star-schema #dimensional-modeling #real-interview

---

## TL;DR

Data Warehouse = optimizado para OLAP (analíticas), no OLTP (transaccional). Esquema: **Star Schema** (hechos + dimensiones) es estándar. Fact = números (ventas, conteos, duración). Dimensiones = descripciones (cliente, producto, tienda, fecha). Compromiso: Esquema Estrella (rápido, más almacenamiento) vs. Normalizado (lento, menos almacenamiento). Para analítica: Esquema Estrella es ganador.

---

## Concepto Core

- **Qué es**: Data Warehouse = optimizado para consultas analíticas, no transacciones
- **Por qué importa**: Fundamental para BI y reporting. Demuestra comprensión de modelado dimensional
- **Principio clave**: Star Schema = denormalizado por diseño (para velocidad de consulta)

---

## Problema Real

**Escenario**:

- Retailer con 1000 tiendas, 100k productos, 50M transacciones/año
- Necesita responder: "Ventas por tienda, categoría, mes en último año"
- Consulta debe ser < 1 segundo (dashboard en tiempo real)
- Equipo de BI: 10 personas con herramientas de visualización

**Preguntas**:

1. ¿Cómo estructuras el esquema?
2. ¿Qué tablas de hechos y dimensiones necesitas?
3. ¿Cómo optimizas para consultas rápidas?
4. ¿Cómo manejas cambios en los datos (históricos de precios)?
5. ¿Cuándo usar Snowflake vs. Star?

---

## Solución Propuesta

### Esquema Estrella

#### Tablas de Hechos (Datos Numéricos)

```sql
-- Tabla de hechos de ventas
CREATE TABLE fact_sales (
    sales_id BIGINT PRIMARY KEY,
    date_id INTEGER NOT NULL REFERENCES dim_date(date_id),
    store_id INTEGER NOT NULL REFERENCES dim_store(store_id),
    product_id INTEGER NOT NULL REFERENCES dim_product(product_id),
    customer_id INTEGER NOT NULL REFERENCES dim_customer(customer_id),
    employee_id INTEGER NOT NULL REFERENCES dim_employee(employee_id),
    quantity_sold INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    sales_amount DECIMAL(12,2) NOT NULL,
    profit_amount DECIMAL(12,2) NOT NULL,
    sales_timestamp TIMESTAMP NOT NULL
);

-- Tabla de hechos de inventario
CREATE TABLE fact_inventory (
    inventory_id BIGINT PRIMARY KEY,
    date_id INTEGER NOT NULL REFERENCES dim_date(date_id),
    store_id INTEGER NOT NULL REFERENCES dim_store(store_id),
    product_id INTEGER NOT NULL REFERENCES dim_product(product_id),
    units_on_hand INTEGER NOT NULL,
    unit_cost DECIMAL(10,2) NOT NULL,
    inventory_date TIMESTAMP NOT NULL
);
```

#### Tablas de Dimensiones (Datos Descriptivos)

```sql
-- Dimensión de Tiempo
CREATE TABLE dim_date (
    date_id INTEGER PRIMARY KEY,
    date DATE NOT NULL,
    day_of_week INTEGER NOT NULL,
    day_of_month INTEGER NOT NULL,
    month INTEGER NOT NULL,
    quarter INTEGER NOT NULL,
    year INTEGER NOT NULL,
    is_weekend BOOLEAN NOT NULL,
    is_holiday BOOLEAN NOT NULL
);

-- Dimensión de Producto
CREATE TABLE dim_product (
    product_id INTEGER PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    product_category VARCHAR(50) NOT NULL,
    product_subcategory VARCHAR(50),
    brand VARCHAR(50),
    supplier_id INTEGER REFERENCES dim_supplier(supplier_id),
    current_price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

-- Dimensión de Tienda
CREATE TABLE dim_store (
    store_id INTEGER PRIMARY KEY,
    store_name VARCHAR(100) NOT NULL,
    store_type VARCHAR(50) NOT NULL,
    address VARCHAR(200),
    city VARCHAR(100),
    state VARCHAR(50),
    country VARCHAR(50),
    opened_date DATE,
    closed_date DATE,
    is_active BOOLEAN
);

-- Dimensión de Cliente
CREATE TABLE dim_customer (
    customer_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20),
    gender CHAR(1),
    birth_date DATE,
    join_date DATE,
    customer_segment VARCHAR(50),
    loyalty_tier VARCHAR(20)
);

-- Dimensión de Empleado
CREATE TABLE dim_employee (
    employee_id INTEGER PRIMARY KEY,
    employee_name VARCHAR(100) NOT NULL,
    store_id INTEGER REFERENCES dim_store(store_id),
    position VARCHAR(50),
    hire_date DATE,
    salary DECIMAL(10,2),
    is_active BOOLEAN
);
```

### Visualización del Esquema

```
┌─────────────────────────────────────────────────────┐
│                   DIMENSIONES                   │
├─────────────────────────────────────────────────────┤
│  dim_date  │  dim_product  │  dim_store  │  dim_customer  │  dim_employee │
└─────────────────────────────────────────────────────┘
                    ▼
┌─────────────────────────────────────────────────────┐
│                  FACT TABLES                      │
├─────────────────────────────────────────────────────┤
│  fact_sales (ventas)  │  fact_inventory (inventario) │
└─────────────────────────────────────────────────────┘
```

### Consultas Típicas

```sql
-- Consulta 1: Ventas por tienda, categoría, mes
SELECT
    s.store_name,
    p.product_category,
    d.month_name,
    SUM(f.sales_amount) AS total_sales,
    SUM(f.profit_amount) AS total_profit,
    COUNT(DISTINCT f.sales_id) AS transaction_count
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_store s ON f.store_id = s.store_id
JOIN dim_product p ON f.product_id = p.product_id
WHERE d.year = 2023
GROUP BY s.store_name, p.product_category, d.month_name
ORDER BY total_sales DESC;

-- Consulta 2: Productos más vendidos por mes
SELECT
    p.product_name,
    d.month_name,
    SUM(f.quantity_sold) AS total_quantity,
    SUM(f.sales_amount) AS total_sales
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_product p ON f.product_id = p.product_id
WHERE d.year = 2023
GROUP BY p.product_name, d.month_name
ORDER BY total_quantity DESC;

-- Consulta 3: Segmentación de clientes
SELECT
    c.customer_segment,
    COUNT(DISTINCT f.customer_id) AS customer_count,
    SUM(f.sales_amount) AS total_spent,
    AVG(f.sales_amount) AS avg_order_value
FROM fact_sales f
JOIN dim_customer c ON f.customer_id = c.customer_id
GROUP BY c.customer_segment
ORDER BY total_spent DESC;
```

---

## Decisiones de Diseño Clave

### 1. Grano de la Tabla de Hechos

**Opción A**: Grano Fino (una fila por transacción)

- Ventajas: Máximo detalle
- Desventajas: Tabla enorme, consultas más lentas

**Opción B**: Grano Grueso (agregado por día)

- Ventajas: Tabla más pequeña, consultas más rápidas
- Desventajas: Menor detalle, no puede analizar transacciones individuales

**Decisión**: **Grano Fino** para el caso de uso. El requisito de análisis de transacciones individuales justifica el mayor tamaño de la tabla.

### 2. Tipos de Datos en Dimensiones

**Atributos de Cambio Rápido** (deben estar en la tabla de hechos):

- Precio actual
- Nivel de inventario
- Estado del pedido

**Atributos de Cambio Lento** (pueden estar en dimensiones):

- Nombre del producto
- Categoría del producto
- Dirección de la tienda

**Decisión**: Separar atributos según su tasa de cambio. Los que cambian con frecuencia (precio, inventario) van en la tabla de hechos.

### 3. Tablas de Hechos Múltiples

**Opción A**: Tabla única de hechos (ventas + devoluciones)

- Ventajas: Más simple
- Desventajas: Consultas complejas para devoluciones

**Opción B**: Tablas separadas (hechos de ventas, hechos de devoluciones)

- Ventajas: Consultas más simples para cada tipo de evento
- Desventajas: Más complejo de mantener

**Decisión**: **Tablas separadas**. El caso de uso incluye análisis de devoluciones, que justifica la separación.

---

## Manejo de Cambios en los Datos

### Cambios en los Precios de los Productos

```sql
-- Estrategia: Rango de Fechas Efectivas (SCD Type 2)
CREATE TABLE dim_product (
    product_id INTEGER PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    product_category VARCHAR(50) NOT NULL,
    -- Columnas de historial de precios
    current_price DECIMAL(10,2) NOT NULL,
    valid_from DATE NOT NULL,
    valid_to DATE,
    is_current BOOLEAN DEFAULT TRUE
);

-- Proceso de SCD
-- 1. Nuevo precio
INSERT INTO dim_product (product_id, product_name, product_category, new_price, CURRENT_DATE, '9999-12-31', TRUE);

-- 2. Marcar precio anterior como obsoleto
UPDATE dim_product
SET valid_to = CURRENT_DATE - INTERVAL '1 day',
    is_current = FALSE
WHERE product_id = 123;

-- 3. Actualizar precio actual
UPDATE dim_product
SET current_price = new_price,
    valid_from = CURRENT_DATE,
    valid_to = '9999-12-31'
WHERE product_id = 123;
```

### Uso de Rangos de Fechas en Consultas

```sql
-- Consulta: Ventas por producto con precio histórico
SELECT
    p.product_name,
    f.sales_timestamp,
    f.sales_amount,
    COALESCE(h.price, p.current_price) AS price_at_sale
FROM fact_sales f
JOIN dim_product p ON f.product_id = p.product_id
LEFT JOIN dim_product_history h ON f.product_id = h.product_id
WHERE f.sales_timestamp BETWEEN h.valid_from AND h.valid_to
ORDER BY f.sales_timestamp;
```

---

## Comparación: Esquema Estrella vs. Snowflake

| Característica      | Estrella                          | Snowflake                           |
| ------------------- | --------------------------------- | ----------------------------------- |
| **Estructura**      | Hechos conectados a dimensiones   | Dimensiones normalizadas            |
| **Consultas**       | Simples, menos joins              | Más joins, más complejas            |
| **Rendimiento**     | Más rápido para consultas simples | Más rápido para consultas complejas |
| **Almacenamiento**  | Más redundancia                   | Menos redundancia                   |
| **Mantenimiento**   | Más simple                        | Más complejo                        |
| **Uso recomendado** | Dashboards, informes estándar     | Data warehouses muy grandes         |

**Decisión**: **Esquema Estrella**. Para el caso de uso, la simplicidad del mantenimiento y las consultas más rápidas para los patrones de consulta típicicos justifican el mayor almacenamiento.

---

## Errores Comunes en Entrevista

- **Error**: "Usar esquema normalizado para warehouse" → **Solución**: El esquema normalizado es lento para analítica. El esquema estrella está diseñado para velocidad de consulta.

- **Error**: "Todos los atributos en la tabla de hechos" → **Solución**: Separar atributos descriptivos en dimensiones.

- **Error**: "No considerar el grano de la tabla de hechos" → **Solución**: El grano debe coincidir con el nivel de análisis requerido.

- **Error**: "No manejar cambios en los datos" → **Solución**: Implementar SCD para cambios de precios y otros atributos que cambian con el tiempo.

---

## Preguntas de Seguimiento Típicas

1. **"¿Cuándo usarías Snowflake en lugar de Star?"**
   - Snowflake para warehouses muy grandes donde el almacenamiento es una preocupación principal
   - Star para la mayoría de los casos de uso general

2. **"¿Cómo manejas relaciones muchos-a-muchos en Star Schema?"**
   - Puente de unión (bridge table) para relaciones muchos-a-muchos
   - O normalizar si la consulta lo requiere

3. **"¿Cómo optimizas consultas lentas?"**
   - Índices en claves foráneas
   - Particionamiento adecuado
   - Vistas materializadas para consultas frecuentes

4. **"¿Cómo manejas datos lentamente cambiantes?"**
   - Rangos de fechas efectivas
   - Vistas materializadas actualizadas periódicamente
   - Considerar Snowflake para reducir redundancia

---

## Referencias

- https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/
