# Self Joins & Datos Jerárquicos

**Tags**: #sql #self-join #hierarchies #relationships #tricky #real-interview

---

## TL;DR

Un **Self Join** es cuando juntas una tabla CONSIGO MISMA. Útil para relaciones jerárquicas (manager-employee, parent-child). Alias la tabla 2 veces con nombres distintos: `FROM employees e1 JOIN employees e2 ON e1.manager_id = e2.id`.

---

## Concepto Core

- **Qué es**: Juntar una tabla consigo misma usando aliases distintos. Permite encontrar relaciones dentro de los mismos datos
- **Por qué importa**: Jerarquías, reportes de línea, relaciones recursivas. Muy común en entrevistas. Demuestra comprensión profundo de JOINs
- **Principio clave**: Necesitas 2+ aliases de la misma tabla. Cada alias representa un "rol" diferente (ej: manager vs employee)

---

## Memory Trick

**"Espejo de tabla"** — Imagina la tabla como espejo. Un lado es "managers", el otro "employees". Self Join los conecta.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Tengo una tabla employees donde cada fila tiene `manager_id` que apunta a otro empleado"

**Paso 2**: "Para encontrar quién es el manager de cada employee, hago JOIN de employees CONSIGO MISMA: `FROM employees e1 (empleado) JOIN employees e2 (manager) ON e1.manager_id = e2.id`"

**Paso 3**: "El resultado: cada fila tiene el empleado Y su manager en la misma fila"

---

## Código/Query ejemplo

### Escenario: Estructura Organizacional

**Tabla: employees**

```
id | name | manager_id | salary | department
1 | Alice | NULL | 150000 | Engineering (CEO)
2 | Bob | 1 | 120000 | Engineering
3 | Charlie | 1 | 100000 | Engineering
4 | David | 2 | 80000 | Engineering
5 | Eve | 2 | 85000 | Engineering
6 | Frank | NULL | 140000 | Sales (Head of Sales)
7 | Grace | 6 | 90000 | Sales
```

### Problema 1: ¿Quién es el Manager de cada Empleado?

```sql
-- ❌ Intento sin Self Join (no funciona)
SELECT
  id,
  name,
  manager_id,
  (SELECT name FROM employees WHERE id = manager_id) as manager_name
FROM employees;

-- ✅ Self Join (correcto)
SELECT
  e1.id as employee_id,
  e1.name as employee_name,
  e1.department,
  e1.salary as employee_salary,
  e2.id as manager_id,
  e2.name as manager_name,
  e2.salary as manager_salary,
  e1.salary - e2.salary as salary_diff
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id
ORDER BY e1.department, e1.name;
```

**Resultado:**

```
employee_id | employee_name | department | employee_salary | manager_id | manager_name | manager_salary | salary_diff
2 | Bob | Engineering | 120000 | 1 | Alice | 150000 | -30000
3 | Charlie | Engineering | 100000 | 1 | Alice | 150000 | -50000
4 | David | Engineering | 80000 | 2 | Bob | 120000 | -40000
5 | Eve | Engineering | 85000 | 2 | Bob | 120000 | -35000
7 | Grace | Sales | 90000 | 6 | Frank | 140000 | -50000
1 | Alice | Engineering | 150000 | NULL | NULL | NULL | NULL
6 | Frank | Sales | 140000 | NULL | NULL | NULL | NULL
```

**Nota**: LEFT JOIN porque algunos (Alice, Frank) no tienen manager

---

### Problema 2: Cadena Jerárquica (3+ niveles)

```sql
-- Manager -> Director -> VP
SELECT
  e1.id as employee_id,
  e1.name as employee_name,
  e1.department,

  e2.name as direct_manager,

  e3.name as director,

  e4.name as vp
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id
LEFT JOIN employees e3 ON e2.manager_id = e3.id
LEFT JOIN employees e4 ON e3.manager_id = e4.id
ORDER BY e1.department, e1.name;
```

**Problema**: Si tienes 10 niveles, necesitas 10 JOINs. Mejor solución: Recursive CTE

---

### Problema 3: Todos los Reportes Directos de un Manager

```sql
-- ¿Quiénes reportan a Alice?
SELECT
  manager.id as manager_id,
  manager.name as manager_name,
  employee.id as report_id,
  employee.name as report_name,
  employee.salary
FROM employees employee
JOIN employees manager ON employee.manager_id = manager.id
WHERE manager.name = 'Alice'
ORDER BY employee.salary DESC;
```

**Resultado:**

```
manager_id | manager_name | report_id | report_name | salary
1 | Alice | 2 | Bob | 120000
1 | Alice | 3 | Charlie | 100000
```

---

### Problema 4: Todos los Subordinados (Recursivo - Cadena Completa)

Para encontrar TODOS los subordinados de Alice (no solo directos):

```sql
-- Recursive CTE (mejor que múltiples Self Joins)
WITH RECURSIVE org_hierarchy AS (
  -- Base: Alice (manager)
  SELECT
    id,
    name,
    manager_id,
    1 as level
  FROM employees
  WHERE id = 1 -- Alice's ID

  UNION ALL

  -- Recursive: Sus subordinados
  SELECT
    e.id,
    e.name,
    e.manager_id,
    oh.level + 1
  FROM employees e
  INNER JOIN org_hierarchy oh ON e.manager_id = oh.id
  WHERE oh.level < 10 -- Limita profundidad para evitar loops
)
SELECT * FROM org_hierarchy
ORDER BY level, name;
```

**Resultado (todos bajo Alice):**

```
id | name | manager_id | level
1 | Alice | NULL | 1
2 | Bob | 1 | 2
3 | Charlie | 1 | 2
4 | David | 2 | 3
5 | Eve | 2 | 3
```

---

### Problema 5: Encontrar Empleados con Mismo Manager

```sql
-- ¿Quiénes trabajan bajo el mismo manager?
SELECT
  e1.name as employee_1,
  e2.name as employee_2,
  e1.department,
  e1.manager_id
FROM employees e1
JOIN employees e2 ON e1.manager_id = e2.manager_id AND e1.id < e2.id
WHERE e1.manager_id IS NOT NULL
ORDER BY e1.manager_id, e1.name;
```

**Resultado:**

```
employee_1 | employee_2 | department | manager_id
Bob | Charlie | Engineering | 1
David | Eve | Engineering | 2
```

**Nota**: `e1.id < e2.id` evita duplicados (Bob-Charlie vs Charlie-Bob)

---

## Errores comunes en entrevista

- **Error**: Olvidar aliases en Self Join → **Solución**: SIEMPRE usa `FROM table t1 JOIN table t2`. Sin aliases, ambos son "table"

- **Error**: Usar INNER JOIN cuando debería LEFT JOIN (pierde CEOs sin manager) → **Solución**: Piensa en la lógica: ¿todos deben tener match?

- **Error**: No evitar duplicados en `e1.id < e2.id` → **Solución**: Sin esto, obtienes cada par 2 veces

- **Error**: Crear Recursive CTE sin condición STOP → **Solución**: Siempre limita profundidad para evitar loop infinito

---

## Preguntas de seguimiento típicas

1. **"¿Diferencia entre Self Join y Recursive CTE?"**
   - Self Join: Niveles fijos (ej: solo employee + manager)
   - Recursive CTE: Cadenas profundas y desconocidas

2. **"¿Cómo optimizarías un Self Join lento?"**
   - Índice en `manager_id`
   - Índice en jerarquía si es muy profunda
   - Considerar denormalización (guardar toda la cadena en una columna)

3. **"¿Cómo manejarías ciclos?"** (ej: A → B → A)
   - Recursive CTE con LIMIT en level
   - Agregar lógica para detectar: `WHERE id NOT IN (path_so_far)`

4. **"¿Real-World Use Cases?"**
   - Org charts, jerarquías
   - Categorías de productos (parent-child)
   - Rutas de tránsito (station → next station)
   - Threads de comentarios (reply-to)

---

## Comparación: Self Join vs Recursive vs Denormalization

| Escenario                              | Solución                       | Ventaja        | Desventaja          |
| -------------------------------------- | ------------------------------ | -------------- | ------------------- |
| **2 niveles (manager-employee)**       | Self Join                      | Simple, rápido | Solo 2 niveles      |
| **N niveles desconocidos**             | Recursive CTE                  | Flexible       | Más lento, complejo |
| **Acceso frecuente a cadena completa** | Denormalization (guardar path) | Super rápido   | Caro mantener       |
| **Queries exploratorios**              | Self Join + manual             | Flexible       | Múltiples queries   |

---

## Real-World: LinkedIn Org Chart

```sql
-- Todos bajo VP de Engineering
WITH RECURSIVE reporting_chain AS (
  SELECT id, name, manager_id, 1 as depth
  FROM employees
  WHERE id = (SELECT id FROM employees WHERE title = 'VP Engineering')

  UNION ALL

  SELECT e.id, e.name, e.manager_id, rc.depth + 1
  FROM employees e
  INNER JOIN reporting_chain rc ON e.manager_id = rc.id
  WHERE rc.depth < 5
)
SELECT * FROM reporting_chain
ORDER BY depth, name;
```

---

## Referencias

- [Self Join Examples - W3Schools](https://www.w3schools.com/sql/sql_join_self.asp)
- [Recursive CTEs - PostgreSQL](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE)
- [Hierarchical Data in SQL - Use The Index Luke](https://use-the-index-luke.com/)
