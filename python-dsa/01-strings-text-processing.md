# Strings & Text Processing en Python

**Tags**: #python #strings #regex #text-processing #data-cleaning #real-interview  
**Empresas**: Amazon, Google, Databricks, Stripe  
**Dificultad**: Mid  
**Tiempo estimado**: 18 min  

---

## TL;DR

Strings en Python: immutable, indexable, iterable. Métodos útiles: `.split()`, `.join()`, `.strip()`, `.replace()`. Regex: `re.findall()`, `re.sub()`, `.match()` para patrones complejos. Para data: validación (emails, phones), parsing (logs, CSVs), normalizacion (lowercase, trim, remove special chars). Performance: strings son O(n), usa `.join()` en loops, no `+`.

---

## Concepto Core

- **Qué es**: Strings son secuencias de caracteres. En data: logs, emails, nombres, IDs → todos strings
- **Por qué importa**: 80% de data cleaning es string manipulation. Regex salva horas. Demuestra idiomaticity en Python
- **Principio clave**: Strings inmutables → siempre creas nuevos. Para loops, `.join()` > `+`

---

## Memory Trick

**"Inspector de datos"** — Strings son tus herramientas para inspeccionar, validar, limpiar datos. Regex es tu lupa.

---

## Cómo explicarlo en entrevista

**Paso 1**: "Strings son fundamentales. 90% de data es strings inicialmente (logs, names, emails)"

**Paso 2**: "Python tiene métodos built-in (split, strip, replace) y regex para patterns"

**Paso 3**: "Para performance: join() es 100x más rápido que + en loops"

**Paso 4**: "Validación: regex para emails, phones. Parsing: regex para logs, CSVs"

---

## Código/Ejemplos

### Parte 1: String Basics

===== BASICS =====
s = "Hello, World!"

Indexing (0-indexed)
print(s) # 'H'
print(s[-1]) # '!'
print(s[0:5]) # 'Hello'

Methods: Built-in
print(s.lower()) # 'hello, world!'
print(s.upper()) # 'HELLO, WORLD!'
print(s.strip()) # Remove leading/trailing whitespace
print(s.replace("World", "Python")) # 'Hello, Python!'
print(s.split(", ")) # ['Hello', 'World!']

===== f-STRINGS (Modern Python) =====
name = "Alice"
age = 28
print(f"{name} is {age} years old") # 'Alice is 28 years old'
print(f"{name.upper()} is {age * 2} in two years") # 'ALICE is 56 in two years'

===== STRING OPERATIONS =====
Concatenation (slow in loops!)
result = ""
for word in ["Python", "is", "awesome"]:
result += word + " " # ❌ BAD: Creates new string each iteration
print(result)

Better: join()
result = " ".join(["Python", "is", "awesome"]) # ✅ GOOD: One allocation
print(result)

text

---

### Parte 2: Validation & Parsing

import re

===== EMAIL VALIDATION =====
def is_valid_email(email):
pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+.[a-zA-Z]{2,}$'
return bool(re.match(pattern, email))

print(is_valid_email("alice@example.com")) # True
print(is_valid_email("alice@example")) # False
print(is_valid_email("alice.smith@example.co.uk")) # True

===== PHONE VALIDATION =====
def is_valid_phone(phone):
pattern = r'^(+1)?[-.\s]?
?
[
0
−
9
]
3
?[0−9]3?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}$'
return bool(re.match(pattern, phone))

print(is_valid_phone("555-1234-5678")) # True
print(is_valid_phone("+1 (555) 1234-5678")) # True
print(is_valid_phone("555")) # False

===== IP ADDRESS VALIDATION =====
def is_valid_ipv4(ip):
pattern = r'^(\d{1,3}.){3}\d{1,3}$'
if not re.match(pattern, ip):
return False
parts = ip.split('.')
return all(0 <= int(p) <= 255 for p in parts)

print(is_valid_ipv4("192.168.1.1")) # True
print(is_valid_ipv4("192.168.1.256")) # False

text

---

### Parte 3: Data Cleaning

===== CLEANING LOGS =====
log_line = " [ERROR] 2024-01-15 10:30:45 Database connection failed "

Step 1: Strip whitespace
cleaned = log_line.strip()

Step 2: Extract level
level_match = re.search(r'
(
\w
+
)
(\w+)', cleaned)
level = level_match.group(1) if level_match else "UNKNOWN"

Step 3: Extract timestamp
timestamp_match = re.search(r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})', cleaned)
timestamp = timestamp_match.group(1) if timestamp_match else None

Step 4: Extract message
message_match = re.search(r'$$ (.+)$', cleaned)
message = message_match.group(1).strip() if message_match else ""

print(f"Level: {level}, Time: {timestamp}, Message: {message}")

Output: Level: ERROR, Time: 2024-01-15 10:30:45, Message: Database connection failed
===== PARSING CSV (poor man's approach) =====
csv_line = 'Alice,"123 Main St",alice@example.com,28'

Split by comma (naive approach, fails if comma in quoted fields)
fields_naive = csv_line.split(',') # ❌ Wrong if commas in data

Better: Use csv module or handle quotes
import csv
import io

reader = csv.reader(io.StringIO(csv_line))
fields = next(reader)
print(fields) # ['Alice', '123 Main St', 'alice@example.com', '28']

===== NORMALIZE NAMES =====
names = [" JOHN SMITH ", "jane_doe", "robert.johnson", "Maria-Garcia"]

def normalize_name(name):
# Strip whitespace
name = name.strip()
# Convert to title case
name = name.title()
# Remove underscores, dots, hyphens
name = re.sub(r'[_.\-]', ' ', name)
# Remove multiple spaces
name = re.sub(r'\s+', ' ', name)
return name

normalized = [normalize_name(n) for n in names]
print(normalized)

['John Smith', 'Jane Doe', 'Robert Johnson', 'Maria Garcia']
text

---

### Parte 4: Regex Patterns (Common)

import re

===== FINDALL: Encontrar todas ocurrencias =====
text = "Emails: alice@example.com, bob@test.org, charlie@company.co.uk"

emails = re.findall(r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+.[a-zA-Z]{2,}', text)
print(emails)

['alice@example.com', 'bob@test.org', 'charlie@company.co.uk']
===== SUB: Reemplazar patrones =====
phone = "Call me at 555-123-4567 or 555.987.6543"

Replace all phone formats with [REDACTED]
redacted = re.sub(r'\d{3}[-.]?\d{3}[-.]?\d{4}', '[PHONE]', phone)
print(redacted)

'Call me at [PHONE] or [PHONE]'
===== SPLIT con regex =====
text = "apple, banana; orange | grape"

Split by comma, semicolon, or pipe
fruits = re.split(r'[,;|]', text)
print(fruits)

['apple', ' banana', ' orange ', ' grape']
Clean up
fruits = [f.strip() for f in fruits]
print(fruits)

['apple', 'banana', 'orange', 'grape']
===== GROUPS: Extraer partes =====
date_str = "2024-01-15"

match = re.match(r'(\d{4})-(\d{2})-(\d{2})', date_str)
if match:
year, month, day = match.groups()
print(f"Year: {year}, Month: {month}, Day: {day}")
# Year: 2024, Month: 01, Day: 15

text

---

### Parte 5: Performance (Importante!)

import time

❌ SLOW: String concatenation in loop
def slow_join():
result = ""
for i in range(100000):
result += f"Item {i}, "
return result

✅ FAST: Using join()
def fast_join():
items = [f"Item {i}" for i in range(100000)]
return ", ".join(items)

Benchmark
start = time.time()
slow_join()
slow_time = time.time() - start

start = time.time()
fast_join()
fast_time = time.time() - start

print(f"Slow: {slow_time:.3f}s") # ~0.5s (slow!)
print(f"Fast: {fast_time:.3f}s") # ~0.01s (100x faster!)

===== RULE =====
For loops with string concat: ALWAYS use list + join(), not +=
text

---

## Regex Patterns Útiles (Copy-Paste)

patterns = {
"email": r'^[a-zA-Z0-9.%+-]+@[a-zA-Z0-9.-]+.[a-zA-Z]{2,}$',
"phone_us": r'^(+1)?[-.\s]?
?
[
0
−
9
]
3
?[0−9]3?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}$',
"ipv4": r'^(\d{1,3}.){3}\d{1,3}$',
"url": r'https?://[^\s]+',
"date_iso": r'\d{4}-\d{2}-\d{2}',
"hex_color": r'^#[0-9A-Fa-f]{6}$',
"credit_card": r'^\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}$',
"ssn": r'^\d{3}-\d{2}-\d{4}$',
"uuid": r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$',
"username": r'^[a-zA-Z0-9]{3,16}$',
}

text

---

## Errores Comunes en Entrevista

- **Error**: Usar `+` en loops para strings → **Solución**: `.join()` siempre

- **Error**: Forgetting about regex groups** → **Solución**: `match.group(1)`, `match.groups()` son potentes

- **Error**: Not handling edge cases (None, empty, special chars)** → **Solución**: Valida input primero

- **Error**: Regex pattern es demasiado permisivo → **Solución**: Test con casos edge (edge@example, +++, etc)

---

## Preguntas de Seguimiento

1. **"¿Cuándo usas regex vs built-in methods?"**
   - `.split()`, `.replace()` para casos simples
   - Regex para patrones complejos

2. **"¿Performance de regex?"**
   - Regex es lento vs built-in
   - Pero correctness > speed (en validation)

3. **"¿Cómo debuggeas regex?"**
   - regex101.com (online tester)
   - Test strings systematically

4. **"¿Encoding issues (UTF-8, Unicode)?"**
   - Python 3 = Unicode por defecto (safe)
   - Python 2 = `.decode('utf-8')` (legacy)

---

## Real-World: Data Cleaning Pipeline

def clean_customer_data(raw_data):
"""Clean y validate customer records"""

text
cleaned = []
errors = []

for record in raw_data:
    try:
        name = record['name'].strip().title()
        
        # Validate email
        email = record['email'].lower().strip()
        if not is_valid_email(email):
            raise ValueError(f"Invalid email: {email}")
        
        # Validate phone
        phone = record['phone']
        if phone and not is_valid_phone(phone):
            raise ValueError(f"Invalid phone: {phone}")
        
        # Clean address (remove extra spaces)
        address = re.sub(r'\s+', ' ', record['address'].strip())
        
        cleaned.append({
            'name': name,
            'email': email,
            'phone': phone,
            'address': address
        })
    
    except ValueError as e:
        errors.append(f"Record {record['id']}: {str(e)}")

return cleaned, errors
Usage
raw_records = [
{'id': 1, 'name': ' ALICE SMITH ', 'email': 'ALICE@EXAMPLE.COM', 'phone': '555-123-4567', 'address': '123 Main St'},
{'id': 2, 'name': 'bob jones', 'email': 'invalid-email', 'phone': '555.987.6543', 'address': '456 Oak Ave'},
]

cleaned, errors = clean_customer_data(raw_records)
print(f"Cleaned: {len(cleaned)}, Errors: {len(errors)}")

Cleaned: 1, Errors: 1
text

---

## References

- [Python Strings - Official Docs](https://docs.python.org/3/tutorial/datastructures.html#strings)
- [Regex Module - re documentation](https://docs.python.org/3/library/re.html)
- [Regex 101 - Online Tester](https://regex101.com/)

