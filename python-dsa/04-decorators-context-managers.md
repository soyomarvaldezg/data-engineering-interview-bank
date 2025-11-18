# Decorators & Context Managers en Python

**Tags**: #python #decorators #context-managers #code-quality #production #real-interview

---

## TL;DR

**Decorators** = funciones que modifican otras funciones (wrapping). Usa `@decorator` syntax. Casos: logging, timing, caching, validation. **Context Managers** = setup/cleanup pattern (`with` statement). Usa `__enter__`/`__exit__` o `@contextmanager`. Casos: file handling, database connections, transactions. Ambos = production-grade code.

---

## Concepto Core

- **Qué es**: Decorators "decoran" funciones. Context managers manejan setup/cleanup
- **Por qué importa**: Production code necesita logging, error handling, resource management. Decorators/CMs hacen esto limpio
- **Principio clave**: DRY (Don't Repeat Yourself) — decorators evitan repetir boilerplate

---

## Memory Trick

**"Decorador y Guardián"** — Decorador envuelve función (add features). Context manager = guardián de recursos (setup/cleanup).

---

## Cómo explicarlo en entrevista

**Paso 1**: "Decorators son wrappers alrededor de funciones. Antes/después behavior"

**Paso 2**: "Context managers manejan setup/cleanup (files, connections). `with` statement"

**Paso 3**: "Ambos son patrones para clean, production-grade code"

**Paso 4**: "Ejemplos: logging decorator, timing decorator, database connection manager"

---

## Código/Ejemplos

### Parte 1: Decorators Básicos

```python
# ===== WHAT IS A DECORATOR? =====
# Function that takes another function and modifies it
# Simple decorator
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print(f"Before {func.__name__}")
        result = func(*args, **kwargs)
        print(f"After {func.__name__}")
        return result
    return wrapper

# Apply decorator
@my_decorator
def greet(name):
    print(f"Hello, {name}!")
    return f"Greeted {name}"

# Call it
result = greet("Alice")

# Prints:
# Before greet
# Hello, Alice!
# After greet
```

```python
# ===== UNDER THE HOOD =====
# @my_decorator is equivalent to:
greet = my_decorator(greet)
```

---

### Parte 2: Practical Decorators

```python
import time
from functools import wraps

# ===== DECORATOR 1: TIMING =====
def timer(func):
    @wraps(func)  # Preserve function metadata
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "Done"

slow_function()
# Output: slow_function took 1.0010s
```

```python
# ===== DECORATOR 2: LOGGING =====
import logging

def log_calls(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        logging.info(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        try:
            result = func(*args, **kwargs)
            logging.info(f"{func.__name__} returned {result}")
            return result
        except Exception as e:
            logging.error(f"{func.__name__} raised {type(e).__name__}: {e}")
            raise
    return wrapper

@log_calls
def divide(a, b):
    return a / b

divide(10, 2)  # Logged: success
divide(10, 0)  # Logged: error
```

```python
# ===== DECORATOR 3: RETRY (Production!) =====
def retry(max_attempts=3, delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
                    print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2)
def flaky_api_call():
    import random
    if random.random() < 0.7:
        raise ConnectionError("Network error")
    return "Success"

flaky_api_call()  # Retries up to 3 times
```

```python
# ===== DECORATOR 4: CACHING (Memoization) =====
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# Without cache: O(2^n)
# With cache: O(n)
print(fibonacci(100))  # Instant (cached)
```

---

### Parte 3: Decorators with Arguments

```python
# ===== DECORATOR WITH ARGUMENTS =====
# Structure: @decorator(arg) → another function → decorator
def repeat(times):
    """Decorator that repeats function execution"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                result = func(*args, **kwargs)
                results.append(result)
            return results
        return wrapper
    return decorator

@repeat(times=3)
def say_hello():
    return "Hello"

print(say_hello())  # ['Hello', 'Hello', 'Hello']
```

```python
# ===== EXAMPLE: RATE LIMITER =====
import time
from collections import defaultdict

def rate_limit(calls_per_second):
    """Limit function calls to N per second"""
    min_interval = 1.0 / calls_per_second
    last_called = [0]  # Use list to make it mutable in nested function

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            last_called[0] = time.time()
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def api_call():
    print(f"API called at {time.time()}")

for _ in range(5):
    api_call()  # Prints every 0.5s (2 calls/sec)
```

---

### Parte 4: Context Managers

```python
# ===== CONTEXT MANAGER: File Handling =====
# Without context manager (❌ can leak if error)
f = open("file.txt")
data = f.read()
f.close()  # If error before this, file not closed

# With context manager (✅ safe, automatic cleanup)
with open("file.txt") as f:
    data = f.read()

# File automatically closed, even if error
```

```python
# ===== BUILDING A CONTEXT MANAGER =====
class DatabaseConnection:
    def __init__(self, host, user, password):
        self.host = host
        self.user = user
        self.password = password
        self.connection = None

    def __enter__(self):
        """Setup: called when entering 'with' block"""
        print(f"Connecting to {self.host}...")
        self.connection = self._connect()
        return self.connection

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Cleanup: called when exiting 'with' block"""
        if self.connection:
            print("Disconnecting...")
            self.connection.close()

        # Handle exceptions if any
        if exc_type is not None:
            print(f"Error occurred: {exc_type.__name__}: {exc_val}")
            return False  # Re-raise exception
        return True

    def _connect(self):  # Mock connection
        return {"connected": True}

# Usage
with DatabaseConnection("localhost", "user", "pass") as conn:
    print("Inside context")
    # conn is the return value from __enter__

# Automatically disconnects (calls __exit__)
```

```python
# ===== DECORATOR-BASED CONTEXT MANAGER =====
from contextlib import contextmanager

@contextmanager
def database_connection(host, user, password):
    """Context manager using decorator"""
    print(f"Connecting to {host}...")

    # Setup
    conn = {"connected": True}

    try:
        yield conn  # This is what 'as' gets
    finally:  # Cleanup (always runs)
        print("Disconnecting...")
        conn["connected"] = False

# Usage
with database_connection("localhost", "user", "pass") as conn:
    print("Inside context")
```

---

### Parte 5: Real-World Production Code

```python
# ===== PRODUCTION: Database Transaction Manager =====
@contextmanager
def db_transaction():
    """Automatic commit/rollback"""
    db = connect_to_db()
    db.begin_transaction()

    try:
        yield db
        db.commit()
        print("Transaction committed")
    except Exception as e:
        db.rollback()
        print(f"Transaction rolled back due to {e}")
        raise
    finally:
        db.close()

# Usage
def transfer_money(from_account, to_account, amount):
    with db_transaction() as db:
        db.update("accounts", {"balance": -amount}, where={"id": from_account})
        db.update("accounts", {"balance": +amount}, where={"id": to_account})

    # If any error: automatic rollback
    # If success: automatic commit
```

```python
# ===== PRODUCTION: Resource Timer =====
@contextmanager
def timed_block(name):
    """Time a block of code"""
    start = time.time()
    print(f"Starting {name}...")

    try:
        yield
    finally:
        elapsed = time.time() - start
        print(f"{name} took {elapsed:.2f}s")

# Usage
with timed_block("Data Processing"):
    process_data()

# Output: Data Processing took 2.34s
```

```python
# ===== PRODUCTION: Temp Directory =====
import tempfile
import shutil

@contextmanager
def temp_directory():
    """Create temporary directory, cleanup automatically"""
    tmpdir = tempfile.mkdtemp()
    print(f"Created temp dir: {tmpdir}")

    try:
        yield tmpdir
    finally:
        shutil.rmtree(tmpdir)
        print(f"Cleaned up {tmpdir}")

# Usage
with temp_directory() as tmpdir:
    # Write temp files
    with open(f"{tmpdir}/data.csv", "w") as f:
        f.write("data...")

# Directory and all files automatically deleted
```

```python
# ===== COMBINING DECORATORS + CONTEXT MANAGERS =====
@log_calls
@retry(max_attempts=3)
@timer
def complex_operation():
    """Logged, retried, timed"""
    with db_transaction() as db:
        with timed_block("Query"):
            results = db.query("SELECT ...")
            return results
```

---

## Errores Comunes en Entrevista

- **Error**: Decorador olvida `@wraps` → **Solución**: Pierde `__name__`, `__doc__` metadata

- **Error**: Context manager exception no manejada → **Solución**: Siempre `try/finally` o usa `__exit__`

- **Error**: Decorador aplicado en orden incorrecto → **Solución**: `@a @b func` = `a(b(func))`

- **Error**: Context manager no cerrando recursos → **Solución**: Test con exceptions adentro

---

## Preguntas de Seguimiento

1. **"¿Diferencia entre decorator y wrapper?"**
   - Decorator: Pattern, puede modificar signature
   - Wrapper: Más general, envuelve cualquier cosa

2. **"¿Cómo debuggeas decorators?"**
   - Usa `@wraps` para preservar metadata
   - Test decorator + function por separado

3. **"¿Múltiples decorators: orden importa?"**
   - Sí, stack en orden inverso
   - `@a @b func` = `a(b(func))`

4. **"¿Context manager vs try/finally?"**
   - CM es más limpio, Pythonic
   - try/finally es más explicit

---

## References

- [Decorators - Python Docs](https://docs.python.org/3/glossary.html#term-decorator)
- [Context Managers - PEP 343](https://www.python.org/dev/peps/pep-0343/)
- [functools.wraps - Standard Library](https://docs.python.org/3/library/functools.html#functools.wraps)
- [contextlib - Standard Library](https://docs.python.org/3/library/contextlib.html)
