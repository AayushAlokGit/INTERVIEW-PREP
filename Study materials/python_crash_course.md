# Python Crash Course — Advanced Concepts

> Assumes you know: variables, loops, conditionals, functions, lists, dicts, basic OOP.
> Focus: things that come up in interviews and production backend code.

---

## 1. Comprehensions

```python
# List comprehension
squares = [x**2 for x in range(10) if x % 2 == 0]

# Dict comprehension
word_len = {word: len(word) for word in ["apple", "fig", "banana"]}

# Set comprehension
unique_lengths = {len(word) for word in ["apple", "fig", "apple"]}

# Nested list comprehension (flatten a 2D list)
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [x for row in matrix for x in row]  # [1, 2, 3, 4, 5, 6]
```

---

## 2. Generators

Lazy evaluation — computes values one at a time. Memory-efficient for large datasets.

```python
# Generator function (uses yield instead of return)
def count_up(n):
    for i in range(n):
        yield i

gen = count_up(5)
next(gen)  # 0
next(gen)  # 1

# Generator expression (like list comprehension but lazy)
gen = (x**2 for x in range(1000000))  # doesn't compute all at once

# Use in for loop
for val in gen:
    print(val)
```

**Key difference:** A list stores everything in memory. A generator produces values on demand.

---

## 3. *args and **kwargs

```python
def func(*args, **kwargs):
    print(args)    # tuple of positional args
    print(kwargs)  # dict of keyword args

func(1, 2, 3, name="Aayush", role="SWE")
# (1, 2, 3)
# {'name': 'Aayush', 'role': 'SWE'}

# Unpacking into a function call
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
add(*nums)  # same as add(1, 2, 3)

config = {"a": 1, "b": 2, "c": 3}
add(**config)  # same as add(a=1, b=2, c=3)
```

---

## 4. Decorators

A decorator wraps a function to add behavior before/after it runs.

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)

slow_function()  # prints: slow_function took 1.0001s
```

```python
# Decorator with arguments
def repeat(n):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                func(*args, **kwargs)
        return wrapper
    return decorator

@repeat(3)
def greet():
    print("Hello")

greet()  # prints Hello three times
```

**Use `functools.wraps`** to preserve the original function's metadata:

```python
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        ...
    return wrapper
```

---

## 5. Context Managers

Used with the `with` statement — guarantees cleanup (file close, DB connection release, lock release).

```python
# Built-in example
with open("file.txt", "r") as f:
    data = f.read()
# file is closed automatically even if an exception occurs

# Custom context manager using a class
class ManagedResource:
    def __enter__(self):
        print("Acquiring resource")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Releasing resource")
        return False  # False = don't suppress exceptions

with ManagedResource() as r:
    print("Using resource")

# Custom context manager using contextlib
from contextlib import contextmanager

@contextmanager
def managed_resource():
    print("Acquiring")
    try:
        yield "the resource"
    finally:
        print("Releasing")

with managed_resource() as r:
    print(f"Using {r}")
```

---

## 6. Closures

A function that captures variables from its enclosing scope.

```python
def make_multiplier(factor):
    def multiply(x):
        return x * factor  # 'factor' is captured from outer scope
    return multiply

double = make_multiplier(2)
triple = make_multiplier(3)

double(5)  # 10
triple(5)  # 15
```

Closures are the mechanism behind decorators.

---

## 7. Lambda, map, filter, reduce

```python
# Lambda: anonymous one-liner function
square = lambda x: x**2
square(4)  # 16

# map: apply a function to every element
nums = [1, 2, 3, 4]
list(map(lambda x: x**2, nums))  # [1, 4, 9, 16]

# filter: keep elements where function returns True
list(filter(lambda x: x % 2 == 0, nums))  # [2, 4]

# reduce: fold a list into a single value
from functools import reduce
reduce(lambda acc, x: acc + x, nums)  # 10 (sum)
```

> In modern Python, prefer list comprehensions over map/filter for readability. Use reduce sparingly.

---

## 8. OOP — Advanced

### `@classmethod` vs `@staticmethod`

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    @classmethod
    def from_dict(cls, data):     # alternative constructor
        return cls(data["name"], data["age"])

    @staticmethod
    def validate_age(age):        # utility — doesn't need self or cls
        return age > 0

u = User.from_dict({"name": "Aayush", "age": 25})
User.validate_age(25)  # True
```

### `@property`

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @property
    def area(self):
        return 3.14 * self._radius ** 2

c = Circle(5)
c.radius = 10    # uses setter
c.area           # computed property, no ()
```

### Dataclasses

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float
    label: str = "origin"
    tags: list = field(default_factory=list)  # mutable default

p = Point(1.0, 2.0)
# Auto-generates __init__, __repr__, __eq__
```

---

## 9. Type Hints

```python
from typing import Optional, Union, List, Dict, Tuple, Any, Callable

def greet(name: str) -> str:
    return f"Hello, {name}"

def process(data: List[Dict[str, Any]]) -> Optional[str]:
    ...

# Union (Python 3.10+: use X | Y)
def parse(val: Union[str, int]) -> str:
    return str(val)

# Callable
def apply(func: Callable[[int], int], val: int) -> int:
    return func(val)
```

Type hints are not enforced at runtime — they're for static analysis (mypy) and readability.

---

## 10. Async / Await

Used for I/O-bound concurrency (HTTP calls, DB queries) — critical for backend services.

```python
import asyncio

async def fetch_data(url: str) -> str:
    await asyncio.sleep(1)  # simulates I/O wait
    return f"data from {url}"

async def main():
    # Sequential — total 2s
    r1 = await fetch_data("url1")
    r2 = await fetch_data("url2")

    # Concurrent — total ~1s
    r1, r2 = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2")
    )

asyncio.run(main())
```

```python
# Async context manager
async with aiofiles.open("file.txt") as f:
    content = await f.read()

# Async generator
async def stream_data():
    for i in range(10):
        await asyncio.sleep(0.1)
        yield i

async for chunk in stream_data():
    print(chunk)
```

**Key rule:** `await` can only be used inside an `async` function. Use `asyncio.gather` to run coroutines concurrently.

---

## 11. Slots (Memory Optimization)

```python
class Point:
    __slots__ = ["x", "y"]  # prevents __dict__, reduces memory

    def __init__(self, x, y):
        self.x = x
        self.y = y
```

Use `__slots__` in classes that will be instantiated millions of times.

---

## 12. Python Internals

### The GIL (Global Interpreter Lock)

The GIL is a mutex in CPython that allows only **one thread to execute Python bytecode at a time**, even on multi-core machines.

**Why does it exist?**
CPython manages memory with reference counting. Without a lock, two threads could simultaneously modify an object's reference count and corrupt memory. The GIL is the simple solution to this.

```python
import threading

counter = 0

def increment():
    global counter
    for _ in range(1_000_000):
        counter += 1  # NOT thread-safe — read-modify-write is 3 bytecode ops

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()

print(counter)  # likely < 2,000,000 — race condition
```

**Consequences:**
- `threading` does NOT give you true parallelism for CPU-bound tasks
- `threading` IS useful for I/O-bound tasks (GIL is released during I/O waits)
- For true CPU parallelism, use `multiprocessing` (separate processes, each with own GIL)

```
CPU-bound work  →  multiprocessing (or C extensions like NumPy that release GIL)
I/O-bound work  →  threading OR async/await
```

> **Python 3.13+**: introduces an experimental "free-threaded" mode (no GIL), opt-in via build flag.

---

### Reference Counting & Garbage Collection

CPython tracks object lifetimes using **reference counts**.

```python
import sys

a = [1, 2, 3]
print(sys.getrefcount(a))  # 2 (one for 'a', one for getrefcount arg)

b = a
print(sys.getrefcount(a))  # 3

del b
print(sys.getrefcount(a))  # back to 2
```

When refcount hits 0, memory is freed immediately.

**The problem — reference cycles:**
```python
a = []
a.append(a)  # a refers to itself — refcount never hits 0
```

CPython has a **cyclic garbage collector** (`gc` module) that periodically detects and cleans up reference cycles. It runs in three generations (0, 1, 2) — short-lived objects are collected most frequently.

```python
import gc
gc.collect()        # force a collection
gc.disable()        # disable (if you manage manually)
gc.get_threshold()  # (700, 10, 10) — default generation thresholds
```

---

### Memory Model: Everything is an Object

In Python, **everything is a heap-allocated object** — integers, functions, classes, modules. Variables are just names (references) pointing to objects.

```python
a = 300
b = 300
a is b  # False — two separate objects on the heap

# BUT: small integers (-5 to 256) are cached (interned)
a = 100
b = 100
a is b  # True — same object from the integer cache
```

```python
# String interning
a = "hello"
b = "hello"
a is b  # True — CPython interns short strings automatically

a = "hello world"
b = "hello world"
a is b  # may be False — longer strings not always interned
```

**`is` vs `==`:**
- `is` checks **identity** (same object in memory)
- `==` checks **equality** (same value)
- Never use `is` to compare values — only use it for `None`, `True`, `False`

---

### How Python Executes Code

```
Source (.py)
    ↓ compile
Bytecode (.pyc, stored in __pycache__)
    ↓ interpret
CPython VM (stack-based interpreter)
```

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# LOAD_FAST  0 (a)
# LOAD_FAST  1 (b)
# BINARY_OP  0 (+)
# RETURN_VALUE
```

The CPython VM is **stack-based** — each operation pushes/pops values from an evaluation stack.

---

### The `multiprocessing` Module

Use when you need true CPU parallelism.

```python
from multiprocessing import Pool

def square(n):
    return n ** 2

with Pool(processes=4) as pool:
    results = pool.map(square, range(10))
    # Each worker is a separate OS process with its own GIL
```

**Gotcha:** Processes don't share memory — data is serialized (pickled) when passed between them. Large objects = expensive IPC.

---

### Threading vs Multiprocessing vs Asyncio

| | `threading` | `multiprocessing` | `asyncio` |
|---|---|---|---|
| **Best for** | I/O-bound | CPU-bound | I/O-bound |
| **Parallelism** | No (GIL) | Yes | No (single thread) |
| **Overhead** | Low | High (process fork) | Very low |
| **Shared memory** | Yes | No (pickling) | Yes |
| **Complexity** | Medium | Medium | Medium-High |

---

### Python Memory Optimization Tools

```python
# __slots__: skips per-instance __dict__, saves ~50-100 bytes/object
class Point:
    __slots__ = ["x", "y"]

# sys.getsizeof: check object size in bytes
import sys
sys.getsizeof([])        # 56
sys.getsizeof([1,2,3])   # 88
sys.getsizeof({})        # 64

# tracemalloc: track memory allocations
import tracemalloc
tracemalloc.start()
# ... your code ...
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics("lineno")[:3]:
    print(stat)
```

---

### Name Lookup Order: LEGB Rule

When Python resolves a variable name, it searches in this order:

```
L — Local (inside current function)
E — Enclosing (outer function, for closures)
G — Global (module level)
B — Built-in (print, len, etc.)
```

```python
x = "global"

def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)  # "local"
    inner()
    print(x)  # "enclosing"

outer()
print(x)  # "global"
```

```python
# global and nonlocal keywords
def counter():
    count = 0
    def increment():
        nonlocal count  # modify enclosing scope variable
        count += 1
    increment()
    increment()
    return count  # 2
```

---

### Metaclasses (Know What They Are)

A metaclass is the class of a class — it controls how classes are created.

```python
# type is the default metaclass
MyClass = type("MyClass", (object,), {"greet": lambda self: "hello"})

# Custom metaclass
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    pass

db1 = Database()
db2 = Database()
db1 is db2  # True
```

You likely won't write metaclasses often, but frameworks like Django ORM and SQLAlchemy use them heavily.

---

## Quick Reference: When to Use What

| Need | Use |
|---|---|
| Lazy large sequence | Generator |
| Count frequencies | `Counter` |
| Graph adjacency list | `defaultdict(list)` |
| Fast queue | `deque` |
| Cache recursive results | `@lru_cache` |
| Concurrent I/O | `async/await` + `asyncio.gather` |
| Guaranteed cleanup | Context manager (`with`) |
| Reusable behavior across functions | Decorator |
| Lightweight struct | `dataclass` or `namedtuple` |
| CPU-bound parallelism | `multiprocessing` |
| I/O-bound concurrency | `asyncio` or `threading` |
| Avoid GIL for CPU work | `multiprocessing` or C extension (NumPy) |
| Reduce memory per object | `__slots__` |
| Debug memory usage | `tracemalloc` + `sys.getsizeof` |
| Singleton / ORM magic | Metaclasses |
