# Generators & Iterators en Python

**Tags**: #python #generators #iterators #lazy-evaluation #memory-efficiency #real-interview

---

## TL;DR

**Iterators** = objeto que puede iterarse (`.next()` método). **Generators** = función que usa `yield` (returns iterator). Key benefit: **Lazy evaluation** — datos generados on-demand, not all-at-once. Para big data: generators = "game changer". Memory: generator de 1M items = O(1), list de 1M items = O(n).

---

## Concepto Core

- **Qué es**: Generators son funciones que "pausa" y "resume". Genera valores uno a la vez (lazy)
- **Por qué importa**: Para big data, generators = must-know. Spark usa este concepto (RDD lazy evaluation)
- **Principio clave**: `yield` vs `return`. Yield = "next value, pause", Return = "done"

---

## Memory Trick

**"Fábrica de demanda"** — Lista = fabricas TODO de una vez (espacio). Generator = fabrica bajo demanda (cuando pides). Generator es más eficiente.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Iterators son objetos iterables. Generators son funciones que usan yield"

**Paso 2**: "Key benefit: Lazy evaluation. Los valores se generan under-demand, no pre-computed"

**Paso 3**: "Para big data: generator de 1M items = O(1) memory. List = O(n) memory"

**Paso 4**: "Spark RDDs usan este concepto. Datos no se cargan hasta que los necesitas"

---

## Código/Ejemplos

### Parte 1: Iterators Basics

```python
# ===== WHAT IS AN ITERATOR? =====
# Object with iter() and next() methods
my_list = [1, 2, 3]

# Create iterator
iterator = iter(my_list)

# Get next element
print(next(iterator))  # 1
print(next(iterator))  # 2
print(next(iterator))  # 3

# next() when done raises StopIteration
try:
    print(next(iterator))  # Raises StopIteration
except StopIteration:
    print("Iterator exhausted")
```

```python
# ===== FOR LOOP UNDER THE HOOD =====
# This:
for item in my_list:
    print(item)

# Is equivalent to:
iterator = iter(my_list)
while True:
    try:
        item = next(iterator)
        print(item)
    except StopIteration:
        break
```

---

### Parte 2: Generators (Simple)

```python
# ===== GENERATOR: Function with yield =====
def simple_generator():
    print("Start")
    yield 1
    print("After 1")
    yield 2
    print("After 2")
    yield 3
    print("After 3")

# Create generator (doesn't execute yet!)
gen = simple_generator()
print("Generator created (but not executed)")

# Get values
print(next(gen))  # Prints "Start", then yields 1
print(next(gen))  # Prints "After 1", then yields 2
print(next(gen))  # Prints "After 2", then yields 3
```

```python
# ===== GENERATOR vs LIST =====
# List: Computes ALL at once
def list_version():
    result = []
    for i in range(5):
        print(f"Computing {i}")
        result.append(i ** 2)
    return result

print("List version:")
list_result = list_version()  # Computes ALL 5 immediately
print(f"Result: {list_result}")

# Generator: Computes on-demand
def generator_version():
    for i in range(5):
        print(f"Computing {i}")
        yield i ** 2

print("\nGenerator version:")
gen_result = generator_version()  # Does NOTHING yet
print("Generator created")
print(next(gen_result))  # Computes only 1st
print(next(gen_result))  # Computes only 2nd
```

---

### Parte 3: Real-World: Reading Large Files

```python
# ❌ BAD: Load entire file into memory
def read_file_all(filepath):
    with open(filepath) as f:
        lines = f.readlines()  # Entire file in memory!
    return lines

# File: 10GB → Memory: 10GB ← Problem!

# ✅ GOOD: Generator (line by line)
def read_file_generator(filepath):
    with open(filepath) as f:
        for line in f:
            yield line.strip()

# Usage: Memory = 1 line only!
for line in read_file_generator("huge_file.log"):
    process(line)  # Process one line at a time
```

```python
# ===== PARSING CSV FROM GENERATOR =====
import csv

def read_csv_generator(filepath):
    with open(filepath) as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row

# Process millions of rows without loading all
for customer in read_csv_generator("customers.csv"):
    validate(customer)
    save_to_db(customer)

# Memory: O(1) regardless of file size!
```

---

### Parte 4: Generator Expressions (Concise)

```python
# ===== GENERATOR EXPRESSION vs LIST COMPREHENSION =====
# List comprehension: creates list in memory
squares_list = [x ** 2 for x in range(1000000)]
print(type(squares_list))  # <class 'list'>
# Memory: ~8MB (1M integers)

# Generator expression: lazy
squares_gen = (x ** 2 for x in range(1000000))
print(type(squares_gen))  # <class 'generator'>
# Memory: ~1KB (just the generator object)
```

```python
# ===== SUM EXAMPLE =====
# Bad: Create huge list, then sum
total = sum([x ** 2 for x in range(1000000)])

# Good: Sum directly from generator (no intermediate list)
total = sum(x ** 2 for x in range(1000000))
```

```python
# ===== CHAINING GENERATORS =====
def even_numbers():
    for i in range(10):
        if i % 2 == 0:
            yield i

def square(numbers):
    for n in numbers:
        yield n ** 2

# Chain them (no intermediate lists!)
result = square(even_numbers())  # Creates generator of generators

for value in result:
    print(value)  # 0, 4, 16, 36, 64
```

---

### Parte 5: Generator Advantages for Big Data

```python
# ===== SCENARIO: Process log file with 1B lines =====
# ❌ NAIVE APPROACH
def process_all_at_once(logfile):
    with open(logfile) as f:
        lines = f.readlines()  # ← OOM! 1B lines = 100GB memory
    errors = [l for l in lines if "ERROR" in l]
    return errors

# ✅ GENERATOR APPROACH
def process_streaming(logfile):
    with open(logfile) as f:
        for line in f:
            if "ERROR" in line:
                yield line.strip()

# Memory: O(1) — only 1 line at a time
```

```python
# ===== REAL SCENARIO: ETL Pipeline =====
def read_orders_from_db(query):
    """Generator: yields orders from database"""
    db = connect_to_db()
    cursor = db.execute(query)

    for row in cursor:
        yield {
            "order_id": row[0],
            "customer_id": row[1],
            "amount": row[2]
        }

def transform_order(orders):
    """Generator: transforms each order"""
    for order in orders:
        order["amount_with_tax"] = order["amount"] * 1.1
        order["processed_at"] = datetime.now()
        yield order

def load_to_warehouse(orders):
    """Generator consumer: loads to warehouse"""
    for order in orders:
        db.insert("fact_orders", order)

# ===== FULL PIPELINE: No intermediate storage! =====
orders = read_orders_from_db("SELECT * FROM orders")
transformed = transform_order(orders)
load_to_warehouse(transformed)

# Memory: O(1) regardless of # orders (even 1B rows)
# vs Traditional ETL: O(n) memory (huge!)
```

---

### Parte 6: Generator Methods

```python
# ===== SEND: Comunicate back to generator =====
def generator_with_send():
    total = 0
    while True:
        value = yield total
        if value is None:
            break
        total += value

gen = generator_with_send()
print(next(gen))  # 0 (first yield)
print(gen.send(10))  # 10 (adds 10, yields new total)
print(gen.send(5))  # 15 (adds 5, yields new total)
gen.send(None)  # Stops generator
```

```python
# ===== THROW: Raise exception inside generator =====
def generator_with_throw():
    try:
        yield 1
        yield 2
        yield 3
    except ValueError:
        print("Caught ValueError!")

gen = generator_with_throw()
print(next(gen))  # 1
gen.throw(ValueError, "Error message")  # Raises inside generator
```

```python
# ===== CLOSE: Stop generator =====
gen = (x for x in range(10))
next(gen)  # 0
gen.close()  # Stop
# next(gen) raises StopIteration
```

---

## Generator vs List Performance

```python
import sys
import time

# Compare memory
numbers_list = list(range(1000000))
numbers_gen = (x for x in range(1000000))

print(f"List memory: {sys.getsizeof(numbers_list) / 1024 / 1024:.2f} MB")
# List memory: 8.59 MB

print(f"Generator memory: {sys.getsizeof(numbers_gen)} bytes")
# Generator memory: 128 bytes (1000x smaller!)

# Compare time
start = time.time()
sum(numbers_list)
list_time = time.time() - start

start = time.time()
sum(x for x in range(1000000))
gen_time = time.time() - start

print(f"List sum: {list_time:.4f}s")  # ~0.08s
print(f"Generator sum: {gen_time:.4f}s")  # ~0.08s (similar, but memory!)
```

---

## Errores Comunes en Entrevista

- **Error**: "Generators son lento" → **Solución**: Generators son igual de rápido, pero MEMORY efficient

- **Error**: Intentar usar generator dos veces → **Solución**: Generators exhaust después de una pasada. Crea uno nuevo si necesitas iterar again

- **Error**: Olvidando que generator es lazy (no ejecuta hasta iterate) → **Solución**: Si necesitas realizar side effects, asegurate de iterar

- **Error**: Retornar list cuando generator es mejor → **Solución**: Para big data, siempre generator

---

## Preguntas de Seguimiento

1. **"¿Diferencia entre generator y coroutine?"**
   - Generator: yields valores, data flow
   - Coroutine: send/receive, bi-directional

2. **"¿Cómo debuggeas generator?"**
   - Difícil, values se generan lazy
   - Solución: Print en generator, o convert a list para debugging

3. **"¿Cuándo list vs generator?"**
   - Generator: Big data, streaming, unknown size
   - List: Necesitas acceso múltiple, random access

4. **"¿Spark RDDs usan generators?"**
   - Sí, RDD = distributed generator (lazy evaluation)
   - `.map()`, `.filter()` crean transformations, no ejecutan

---

## Real-World: ETL Pipeline with Generators

```python
from datetime import datetime
import csv

def read_csv_chunked(filepath, chunk_size=1000):
    """Read CSV in chunks, yield batches"""
    with open(filepath) as f:
        reader = csv.DictReader(f)
        chunk = []
        for row in reader:
            chunk.append(row)
            if len(chunk) == chunk_size:
                yield chunk
                chunk = []
        if chunk:
            yield chunk

def validate_records(record_batches):
    """Validate and filter"""
    for batch in record_batches:
        valid_batch = []
        for record in batch:
            if is_valid(record):
                valid_batch.append(record)
        if valid_batch:
            yield valid_batch

def load_to_warehouse(valid_batches):
    """Load to database"""
    total_loaded = 0
    for batch in valid_batches:
        db.insert_batch("fact_data", batch)
        total_loaded += len(batch)
    return total_loaded

# ===== FULL PIPELINE =====
batches = read_csv_chunked("huge_file.csv", chunk_size=5000)
valid_batches = validate_records(batches)
total = load_to_warehouse(valid_batches)

print(f"Loaded {total} records")

# Memory: Only 5000 rows in memory at a time!
# File: 1B rows, Memory: 5000 rows (0.0005%)
```

---

## References

- [Iterators & Generators - Python Docs](https://docs.python.org/3/tutorial/classes.html#iterators)
- [Generator Expressions - Python Docs](https://docs.python.org/3/reference/expressions.html#generator-expressions)
