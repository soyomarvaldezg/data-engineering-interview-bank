# Calcular Pasivo Total de Préstamos por Cliente

**Tags**: #sql #joins #aggregation #banking #real-interview

---

## TL;DR

Haz `JOIN` entre `Loans` y `Accounts` usando `customer_id`, luego `SUM()` el saldo pendiente (`outstanding_balance`) agrupando por cliente.

---

## Concepto Core

- **Qué es**: Necesitas consolidar datos de múltiples tablas (préstamos e información de cuentas) para un cálculo por cliente
- **Por qué importa**: JOINs son fundamentales en data engineering. Demuestra que entiendes relaciones entre entidades
- **Principio clave**: Usa `INNER JOIN` si ambas tablas tienen el customer, `LEFT JOIN` si algunos clientes podrían no tener préstamos

---

## Memory Trick

**"Juntando información"** — Loans y Accounts tienen customer_id. Lo juntas con JOIN (como vincular dos archivos por ID) y luego agrupas.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Tengo dos tablas: `Loans` (con préstamos y saldos) y `Accounts` (info de cliente). Ambas tienen `customer_id`"

**Paso 2**: "Las uno con `JOIN` en customer_id. Si necesito solo clientes CON préstamos, INNER JOIN. Si todos los clientes, LEFT JOIN a partir de Accounts"

**Paso 3**: "Agrupo por `customer_id` y sumo `outstanding_balance`. El resultado: cada cliente con su pasivo total"

---

## Código/Query ejemplo

**Tablas disponibles:**

Loans:

```
loan_id | customer_id | loan_type | outstanding_balance | status
1 | 101 | Mortgage | 250000 | Active
2 | 101 | Auto | 25000 | Active
3 | 102 | Personal | 5000 | Active
4 | 103 | Mortgage | 300000 | Defaulted
```

Accounts:

```
account_id | customer_id | account_type | balance
1001 | 101 | Checking | 5000
1002 | 101 | Savings | 20000
1003 | 102 | Checking | 2000
1004 | 104 | Savings | 50000
```

### Solución 1: INNER JOIN (solo clientes con préstamos)

```sql
SELECT
  l.customer_id,
  a.account_type,
  COUNT(l.loan_id) as num_loans,
  SUM(l.outstanding_balance) as total_outstanding,
  SUM(a.balance) as total_account_balance,
  SUM(l.outstanding_balance) - SUM(a.balance) as net_liability
FROM Loans l
INNER JOIN Accounts a ON l.customer_id = a.customer_id
GROUP BY l.customer_id, a.account_type
ORDER BY total_outstanding DESC;
```

**Resultado:**

```
customer_id | account_type | num_loans | total_outstanding | total_account_balance | net_liability
101 | Checking | 2 | 275000 | 5000 | 270000
101 | Savings | 2 | 275000 | 20000 | 255000
102 | Checking | 1 | 5000 | 2000 | 3000
```

⚠️ **Nota**: Si un cliente tiene múltiples cuentas, aparece en múltiples filas. Para consolidado, ver Solución 2.

### Solución 2: Consolidado por Cliente (más limpio)

```sql
SELECT
  l.customer_id,
  COUNT(DISTINCT l.loan_id) as num_loans,
  SUM(l.outstanding_balance) as total_outstanding_liability,
  COUNT(DISTINCT a.account_id) as num_accounts,
  SUM(a.balance) as total_account_balance
FROM Loans l
INNER JOIN Accounts a ON l.customer_id = a.customer_id
GROUP BY l.customer_id
ORDER BY total_outstanding_liability DESC;
```

**Resultado:**

```
customer_id | num_loans | total_outstanding_liability | num_accounts | total_account_balance
101 | 2 | 275000 | 2 | 25000
102 | 1 | 5000 | 1 | 2000
```

### Solución 3: LEFT JOIN (todos los clientes)

Si quieres incluir clientes SIN préstamos:

```sql
SELECT
  a.customer_id,
  COUNT(DISTINCT l.loan_id) as num_loans,
  COALESCE(SUM(l.outstanding_balance), 0) as total_outstanding_liability,
  COUNT(DISTINCT a.account_id) as num_accounts,
  SUM(a.balance) as total_account_balance
FROM Accounts a
LEFT JOIN Loans l ON a.customer_id = l.customer_id
GROUP BY a.customer_id
ORDER BY total_outstanding_liability DESC;
```

**Resultado incluye cliente 104 (sin préstamos):**

```
customer_id | num_loans | total_outstanding_liability | num_accounts | total_account_balance
101 | 2 | 275000 | 2 | 25000
102 | 1 | 5000 | 1 | 2000
104 | 0 | 0 | 1 | 50000
```

---

## Errores comunes en entrevista

- **Error**: Hacer CROSS JOIN en lugar de JOIN (sin ON clause) → **Solución**: Siempre especifica `ON l.customer_id = a.customer_id`

- **Error**: Usar INNER JOIN cuando necesitas LEFT JOIN → **Solución**: Si clientes sin préstamos deben aparecer, usa LEFT JOIN y COALESCE

- **Error**: Olvidar DISTINCT en COUNT dentro del GROUP BY → **Solución**: Si un cliente aparece múltiples veces (múltiples cuentas), COUNT sin DISTINCT da números inflados

- **Error**: No usar COALESCE con LEFT JOIN → **Solución**: Con LEFT JOIN, NULL es común. Usa COALESCE(suma, 0) para evitar NULLs

---

## Preguntas de seguimiento típicas

1. **"¿Diferencia entre INNER JOIN, LEFT JOIN y FULL OUTER JOIN?"**
   - INNER: Solo filas que existen en ambas tablas
   - LEFT: Todas las filas de la izquierda, NULLs en la derecha si no coinciden
   - FULL OUTER: Todas las filas de ambas

2. **"¿Qué pasa si un cliente tiene 1 préstamo pero 3 cuentas?"**
   - Con INNER JOIN: 3 filas (1 préstamo x 3 cuentas)
   - Con COUNT DISTINCT: 1 fila, conteo correcto

3. **"¿Cómo optimizarías esto para 100M de clientes?"**
   - Índice en `Loans(customer_id, outstanding_balance)`
   - Índice en `Accounts(customer_id, balance)`
   - Spark: Repartición por customer_id antes del JOIN

4. **"¿Y si quiero incluir préstamos pagados (status = 'Closed')?"**
   - Agrega `WHERE l.status IN ('Active', 'Closed')` o deja la lógica de negocio abierta

---

## Alternativa: Subqueries (menos óptimo pero válido)

```sql
SELECT
  customer_id,
  (SELECT SUM(outstanding_balance) FROM Loans WHERE Loans.customer_id = Accounts.customer_id) as total_outstanding
FROM Accounts
GROUP BY customer_id;
```

⚠️ **Nota**: Esto es más lento porque ejecuta subquery por cada fila. Usa JOIN cuando sea posible.

---

## Contexto Bancario

En banking, este query es común para:

- **Risk Assessment**: ¿Cuánto debe cada cliente?
- **Credit Scoring**: Cliente con alto pasivo = riesgo mayor
- **Collections**: Identificar clientes con deuda importante
- **Compliance**: Reportes de exposición

---

## Referencias

- [SQL JOINs - W3Schools](https://www.w3schools.com/sql/sql_join.asp)
- [GROUP BY y Agregaciones - PostgreSQL Docs](https://www.postgresql.org/docs/current/queries-table-expressions.html)
