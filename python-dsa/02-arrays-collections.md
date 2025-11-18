# Arrays & Collections en Python

**Tags**: #python #collections #lists #dicts #sets #performance #real-interview

---

## TL;DR

**Collections**: Lists (ordered, mutable), Tuples (ordered, immutable), Sets (unordered, unique), Dicts (key-value, mutable). Performance: List append O(1), search O(n). Set/Dict O(1) average. Use Set para "is it here?", Dict para mappings. For data: lists para arrays, dicts para records (JSON-like), sets para deduplication.

---

## Concepto Core

- **Qué es**: Collections = contenedores de datos. Python tiene 4 principales: list, tuple, set, dict
- **Por qué importa**: Eligir bien = performance. List vs Set para búsqueda = 100x diferencia
- **Principio clave**: List para orden, Dict para lookup, Set para unique, Tuple para immutable

---

## Memory Trick

**"Cajas de datos"** — List = caja ordenada (puedes agregar/remover). Dict = fichas indexadas (lookup rápido). Set = bolsa de únicos (sin orden). Tuple = caja sellada (no cambios).

---

## Cómo explicarlo en entrevista

**Paso 1**: "Python tiene 4 collections: list, tuple, dict, set. Cada una tiene use case"

**Paso 2**: "List: ordenada, mutable. Dict: lookup rápido. Set: unique, sin orden"

**Paso 3**: "Performance importa: búsqueda en list O(n), en set O(1)"

**Paso 4**: "Para data: lists para arrays, dicts para records, sets para dedup"

---

## Código/Ejemplos

### Parte 1: Lists

```python
# ===== BASICS =====
numbers = [1, 2, 3, 4, 5]

# Access
print(numbers[0])  # 1 (0-indexed)
print(numbers[-1])  # 5 (last element)
print(numbers[1:4])  # [2, 3, 4] (slice)

# Modify
numbers.append(6)  # [1, 2, 3, 4, 5, 6]
numbers.pop()  # Remove last: [1, 2, 3, 4, 5]
numbers.insert(0, 0)  # Insert at index: [0, 1, 2, 3, 4, 5]
numbers.remove(3)  # Remove by value: [0, 1, 2, 4, 5]

# Iteration
for num in numbers:
    print(num)

# List comprehension (elegant)
squared = [x**2 for x in numbers]
print(squared)  # [0, 1, 4, 16, 25]

# Filtering
evens = [x for x in numbers if x % 2 == 0]
print(evens)  # [0, 2, 4]
```

```python
# ===== PERFORMANCE =====
# Append: O(1) amortized
numbers.append(100)  # Fast

# Search: O(n) - linear!
if 3 in numbers:  # Slow for large lists
    print("Found")

# Access by index: O(1)
first = numbers[0]  # Fast

# Insert at beginning: O(n) - slow!
numbers.insert(0, -1)  # Slow, shifts all elements
```

---

### Parte 2: Tuples (Immutable)

```python
# ===== BASICS =====
point = (10, 20)  # Coordinates
rgb = (255, 128, 0)  # Color

# Access (like list)
print(point[0])  # 10
print(point[1])  # 20

# ✗ Cannot modify
point[0] = 15  # ERROR: tuples immutable

# Unpacking
x, y = point
print(f"x={x}, y={y}")  # x=10, y=20

# Return multiple values
def get_user():
    return ("Alice", 28, "alice@example.com")

name, age, email = get_user()
```

```python
# ===== WHEN TO USE =====

# 1. Function returns (can't modify accidentally)
# 2. Dictionary keys (tuples hashable, lists not)
# 3. Data is constant (semantic clarity)
locations = {
    (0, 0): "Origin",
    (10, 20): "Point A",
    (30, 40): "Point B"
}

print(locations[(0, 0)])  # "Origin"
```

---

### Parte 3: Dictionaries (Lookup)

```python
# ===== BASICS =====
user = {
    "name": "Alice",
    "age": 28,
    "email": "alice@example.com",
    "location": "NYC"
}

# Access
print(user["name"])  # "Alice"
print(user.get("age", 0))  # 28 (safe, returns default if missing)
print(user.get("phone", None))  # None (key doesn't exist)

# Modify
user["age"] = 29  # Update
user["phone"] = "555-123-4567"  # Add new key

# Iterate
for key in user:
    print(f"{key}: {user[key]}")

for key, value in user.items():
    print(f"{key}: {value}")
```

```python
# ===== JSON-LIKE STRUCTURES =====
customers = [
    {"id": 1, "name": "Alice", "email": "alice@example.com"},
    {"id": 2, "name": "Bob", "email": "bob@example.com"},
]

# Find by id
def find_customer(customers, customer_id):
    for customer in customers:
        if customer["id"] == customer_id:
            return customer
    return None

alice = find_customer(customers, 1)
print(alice)  # {"id": 1, "name": "Alice", ...}
```

```python
# ===== PERFORMANCE: LOOKUP =====
# Using list
users_list = [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"},
    # ... 1 million more
]

# Search: O(n) - slow!
for user in users_list:
    if user["id"] == 500000:  # Takes ~500k iterations
        break

# Using dict (better)
users_dict = {
    1: {"name": "Alice"},
    2: {"name": "Bob"},
    # ... 1 million more
}

# Lookup: O(1) - fast!
user = users_dict[500000]  # Instant!
```

```python
# ===== DEFAULTDICT (elegant) =====
from collections import defaultdict

# Count occurrences
words = ["apple", "banana", "apple", "cherry", "apple"]

# Old way
word_count = {}
for word in words:
    if word in word_count:
        word_count[word] += 1
    else:
        word_count[word] = 1

# Better: defaultdict
word_count = defaultdict(int)
for word in words:
    word_count[word] += 1

print(word_count)  # {'apple': 3, 'banana': 1, 'cherry': 1}
```

---

### Parte 4: Sets (Unique, Fast Lookup)

```python
# ===== BASICS =====
fruits = {"apple", "banana", "cherry", "banana"}
print(fruits)  # {'apple', 'banana', 'cherry'} - duplicates gone!

# Operations
fruits.add("orange")
fruits.remove("banana")
print(fruits)  # {'apple', 'cherry', 'orange'}
```

```python
# ===== SET OPERATIONS =====
set_a = {1, 2, 3, 4, 5}
set_b = {4, 5, 6, 7, 8}

# Union (all elements)
union = set_a | set_b  # {1, 2, 3, 4, 5, 6, 7, 8}

# Intersection (common)
intersection = set_a & set_b  # {4, 5}

# Difference (only in A)
difference = set_a - set_b  # {1, 2, 3}
```

```python
# ===== USE CASE 1: DEDUPLICATION =====
emails = ["alice@example.com", "bob@example.com", "alice@example.com", "charlie@example.com"]

unique_emails = list(set(emails))
print(unique_emails)
# ['alice@example.com', 'bob@example.com', 'charlie@example.com']
```

```python
# ===== USE CASE 2: FAST MEMBERSHIP TEST =====
# Scenario: Check if customer is VIP
# Slow approach (list)
vip_list = ["Alice", "Bob", "Charlie", "David"]  # ...1 million more
if "Alice" in vip_list:  # O(n) - slow
    print("VIP")

# Fast approach (set)
vip_set = {"Alice", "Bob", "Charlie", "David"}  # ...1 million more
if "Alice" in vip_set:  # O(1) - fast!
    print("VIP")
```

```python
# ===== PERFORMANCE =====
# List membership: O(n)
# Set membership: O(1)
import time

large_list = list(range(1000000))
large_set = set(range(1000000))

# Search for last element (worst case)
start = time.time()
_ = 999999 in large_list
list_time = time.time() - start

start = time.time()
_ = 999999 in large_set
set_time = time.time() - start

print(f"List: {list_time:.6f}s")  # ~0.05s
print(f"Set: {set_time:.6f}s")  # ~0.000001s (10,000x faster!)
```

---

### Parte 5: Collections Advanced

```python
from collections import Counter, defaultdict, OrderedDict

# ===== COUNTER: Count occurrences =====
words = ["apple", "banana", "apple", "cherry", "apple", "banana"]

counter = Counter(words)
print(counter)  # Counter({'apple': 3, 'banana': 2, 'cherry': 1})

# Top 2 most common
print(counter.most_common(2))  # [('apple', 3), ('banana', 2)]
```

```python
# ===== DEFAULTDICT: Grouping =====
# Group customers by city
customers = [
    {"name": "Alice", "city": "NYC"},
    {"name": "Bob", "city": "LA"},
    {"name": "Charlie", "city": "NYC"},
    {"name": "David", "city": "LA"},
]

from collections import defaultdict
customers_by_city = defaultdict(list)

for customer in customers:
    customers_by_city[customer["city"]].append(customer["name"])

print(customers_by_city)
# {'NYC': ['Alice', 'Charlie'], 'LA': ['Bob', 'David']}
```

```python
# ===== NESTED DICT: Multi-level lookup =====
# structure: region -> city -> count
data = defaultdict(lambda: defaultdict(int))

data["US"]["NYC"] = 100
data["US"]["LA"] = 50
data["EU"]["Paris"] = 30

print(data["US"]["NYC"])  # 100
```

---

## Comparación: Cuando Usar Cada Una

| Collection | Ordered | Mutable | Lookup | Use Case             |
| ---------- | ------- | ------- | ------ | -------------------- |
| **List**   | ✓       | ✓       | O(n)   | Arrays, sequences    |
| **Tuple**  | ✓       | ✗       | O(n)   | Immutable, dict keys |
| **Dict**   | ✗\*     | ✓       | O(1)   | Mappings, records    |
| **Set**    | ✗       | ✓       | O(1)   | Unique, membership   |

\*Python 3.7+: dicts mantienen insertion order, pero no es guaranteed

---

## Errores Comunes en Entrevista

- **Error**: Usar list cuando necesitas set (membership O(n) vs O(1)) → **Solución**: Para "is X in Y?", usa set

- **Error**: Usar dict ineficientemente (múltiples loops) → **Solución**: Indexa por clave, O(1) lookup

- **Error**: Olvidando que dicts son referencias (shallow copy) → **Solución**: `.copy()` o `deepcopy()` si necesitas independencia

- **Error**: Modificar collection mientras iteras → **Solución**: Itera sobre copia o crea lista nueva

---

## Preguntas de Seguimiento

1. **"¿Diferencia entre list.copy() y deepcopy()?"**
   - Shallow: Copia superficial (nested objs compartidos)
   - Deep: Copia profunda (todo independiente)

2. **"¿Performance: list vs deque?"**
   - List: append O(1), pop(0) O(n)
   - Deque: append O(1), pop(0) O(1)

3. **"¿Cómo ordenan dicts?"**
   - Python 3.7+: Insertion order
   - Explícito: `sorted(dict.items())`

4. **"¿Memory de sets vs lists?"**
   - Set: ~35% más memory overhead
   - Pero worth it para lookup performance

---

## Real-World: Data Processing Pipeline

```python
def deduplicate_and_validate(records):
    """Dedup by email, validate, group by city"""
    seen_emails = set()
    valid_records = []
    by_city = defaultdict(list)

    for record in records:  # Deduplicate by email
        if record["email"] in seen_emails:
            continue  # Skip duplicates

        seen_emails.add(record["email"])

        # Validate
        if not is_valid_email(record["email"]):
            continue

        valid_records.append(record)

        # Group by city
        by_city[record["city"]].append(record["name"])

    return valid_records, dict(by_city)

# Usage
records = [
    {"name": "Alice", "email": "alice@example.com", "city": "NYC"},
    {"name": "Bob", "email": "bob@example.com", "city": "LA"},
    {"name": "Alice", "email": "alice@example.com", "city": "NYC"},  # Duplicate
    {"name": "Charlie", "email": "invalid", "city": "NYC"},  # Invalid
]

valid, by_city = deduplicate_and_validate(records)
print(f"Valid: {len(valid)}")
print(f"By city: {by_city}")
# Valid: 2
# By city: {'NYC': ['Alice'], 'LA': ['Bob']}
```

---

## References

- [Python Collections - Official Docs](https://docs.python.org/3/tutorial/datastructures.html)
- [Collections Module - Advanced](https://docs.python.org/3/library/collections.html)
- [Big-O Complexity - Python Data Structures](https://wiki.python.org/moin/TimeComplexity)
