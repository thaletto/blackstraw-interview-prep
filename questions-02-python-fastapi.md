# Python / FastAPI / Pydantic Interview Questions

**Your Self-Assessment:** 7/10  
**Focus:** Decorators, generators, async, Python internals, FastAPI patterns (your weak areas)

---

## Question 1: Decorators Fundamentals

**Difficulty:** Intermediate  
**Category:** Gap Identification

What is a Python decorator? Write a `@timer` decorator that logs how long a function takes to execute.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

A **decorator** is a function that takes another function and extends its behavior without explicitly modifying it. It's syntactic sugar for function wrapping.

**Basic Implementation:**

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)  # Preserves func.__name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "Done"

slow_function()
# Output: slow_function took 1.0012s
```

**Why `@wraps` is Critical:**

```python
# Without @wraps - metadata is lost
def timer(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@timer
def my_function():
    """My docstring"""
    pass

print(my_function.__name__)  # 'wrapper' (wrong)
print(my_function.__doc__)   # None (wrong)

# With @wraps - metadata is preserved
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

**Decorator with Arguments (Decorator Factory):**

```python
def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")
# Output: Hello, Alice! (3 times)
```

**Key Concepts:**
- Decorators are higher-order functions
- `@wraps` preserves metadata (critical for debugging)
- Decorator factories accept arguments (returns a decorator)
- Can be stacked (applied bottom-up)
- Work on methods, classes, and functions

**Common Mistakes:**
- Forgetting `@wraps` (breaks introspection and debugging)
- Not handling `*args, **kwargs` (breaks functions with arguments)
- Confusing decorator vs. decorator factory
- Side effects in decorators (hard to test)

**Interview Tip:**
> "A decorator takes a function and returns a wrapped version. I always use `@functools.wraps` to preserve the original function's metadata. For parameterized decorators, I create a decorator factory—a function that returns a decorator. Decorators are powerful for cross-cutting concerns like logging, caching, and access control."

</details>

---

## Question 2: Generator Functions

**Difficulty:** Intermediate  
**Category:** Gap Identification

What's the difference between a regular function and a generator function? When would you use generators?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Regular Function:** Returns a single value and exits.

```python
def regular_function():
    return 1
    return 2  # Never executed
```

**Generator Function:** Uses `yield` to produce a sequence of values lazily.

```python
def generator_function():
    yield 1
    yield 2
    yield 3

gen = generator_function()  # Generator object
print(next(gen))  # 1
print(next(gen))  # 2
print(next(gen))  # 3
print(next(gen))  # StopIteration exception
```

**Key Characteristics:**

1. **Lazy Evaluation:** Values are produced on-demand
2. **State Preservation:** Local variables persist between `yield` calls
3. **Memory Efficient:** Doesn't store entire sequence in memory
4. **One-Time Use:** Exhausted after one iteration

**Practical Example: Reading Large Files**

```python
# Bad: Loads entire file into memory
def read_lines_bad(filename):
    with open(filename) as f:
        return f.readlines()  # List of all lines

# Good: Generator yields one line at a time
def read_lines_good(filename):
    with open(filename) as f:
        for line in f:
            yield line.strip()

# Usage
for line in read_lines_good('huge_file.txt'):
    process(line)  # Only one line in memory at a time
```

**Generator Expressions:**

```python
# List comprehension (eager - creates full list)
squares_list = [x**2 for x in range(1000000)]

# Generator expression (lazy - creates generator)
squares_gen = (x**2 for x in range(1000000))

import sys
print(sys.getsizeof(squares_list))  # ~8 MB
print(sys.getsizeof(squares_gen))   # ~200 bytes
```

**Use Cases:**
- Reading large files
- Streaming data
- Infinite sequences
- Pipeline processing
- Memory-constrained environments

**Key Concepts:**
- `yield` pauses function execution
- Generator objects are iterators
- `next()` advances to next `yield`
- Generator expressions use `(expr for x in iterable)`
- `StopIteration` signals exhaustion

**Common Mistakes:**
- Treating generator as list (can't index or get length)
- Exhausting generator by iterating twice
- Not understanding lazy evaluation
- Using generators when you need random access

**Interview Tip:**
> "Generators use `yield` to produce values lazily, one at a time. They're memory-efficient for large datasets—reading a 10GB file uses constant memory instead of loading it all. I use them for streaming data, ETL pipelines, and when I need to process sequences too large for memory. The key is lazy evaluation: values are computed only when requested."

</details>

---

## Question 3: Async/Await Deep Dive

**Difficulty:** Advanced  
**Category:** Gap Identification

Explain Python's asyncio event loop. How does `async`/`await` work under the hood?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Core Concept:**

`async`/`await` is **cooperative multitasking** in a single thread. The event loop manages coroutines and switches between them when they `await` I/O operations.

**Basic Syntax:**

```python
import asyncio

async def fetch_data(url: str) -> dict:
    print(f"Fetching {url}")
    await asyncio.sleep(1)  # Simulates I/O
    return {"data": url}

async def main():
    task1 = asyncio.create_task(fetch_data("url1"))
    task2 = asyncio.create_task(fetch_data("url2"))
    
    result1 = await task1
    result2 = await task2
    
    print(result1, result2)

asyncio.run(main())
# Both URLs fetched concurrently
```

**How It Works:**

1. **`async def`** creates a **coroutine function**
2. Calling it returns a **coroutine object** (not executed yet)
3. **`await`** suspends execution and yields control to event loop
4. Event loop runs other ready coroutines
5. When awaited operation completes, coroutine resumes

**Concurrency vs. Parallelism:**

```python
# Sequential (slow)
async def slow():
    await asyncio.sleep(1)
    await asyncio.sleep(1)
    # Takes 2 seconds

# Concurrent (fast)
async def fast():
    task1 = asyncio.sleep(1)
    task2 = asyncio.sleep(1)
    await asyncio.gather(task1, task2)
    # Takes 1 second
```

**Common Pitfalls:**

```python
# Forgetting await
async def main():
    result = fetch_data("url")  # Returns coroutine object, not data
    print(result)  # <coroutine object>

# Properly awaiting
async def main():
    result = await fetch_data("url")  # Actual data

# Blocking the event loop
async def main():
    time.sleep(1)  # Blocks entire event loop!
    await asyncio.sleep(1)  # Non-blocking
```

**Key Concepts:**
- Coroutines are cooperatively scheduled
- Single-threaded (no GIL bypass for I/O)
- `await` yields control to event loop
- Use `asyncio.gather()` for concurrent execution
- CPU-bound tasks need `ProcessPoolExecutor`, not asyncio

**Common Mistakes:**
- Using `time.sleep()` instead of `asyncio.sleep()`
- Mixing sync and async code incorrectly
- Not using `asyncio.gather()` for independent tasks
- Using asyncio for CPU-bound work (use multiprocessing)

**Interview Tip:**
> "Python's asyncio is cooperative multitasking on a single thread. The event loop runs coroutines until they `await` an I/O operation, then switches to another ready coroutine. It's perfect for I/O-bound work like API calls and database queries, but useless for CPU-bound tasks—those need multiprocessing. I use `asyncio.gather()` for parallel execution."

</details>

---

## Question 4: Pydantic Custom Validators

**Difficulty:** Intermediate  
**Category:** Learning

Create a Pydantic model with custom validators for a user registration system.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Basic Model:**

```python
from pydantic import BaseModel, EmailStr, Field, field_validator
from typing import Optional
from datetime import datetime
import re

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    email: EmailStr
    password: str = Field(..., min_length=8)
    age: int = Field(..., ge=18, le=120)
    bio: Optional[str] = Field(None, max_length=500)
    created_at: datetime = Field(default_factory=datetime.now)
    
    @field_validator('username')
    @classmethod
    def validate_username(cls, v):
        if not re.match(r'^[a-zA-Z0-9_]+$', v):
            raise ValueError('Username must be alphanumeric')
        if v.startswith('_'):
            raise ValueError('Username cannot start with underscore')
        return v.lower()
    
    @field_validator('password')
    @classmethod
    def validate_password(cls, v):
        if not re.search(r'[A-Z]', v):
            raise ValueError('Password must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Password must contain lowercase letter')
        if not re.search(r'\d', v):
            raise ValueError('Password must contain digit')
        if not re.search(r'[!@#$%^&*]', v):
            raise ValueError('Password must contain special character')
        return v
```

**Usage in FastAPI:**

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.post("/users")
async def create_user(user: UserCreate):
    # Pydantic automatically validates
    # If validation fails, returns 422 with error details
    return {"username": user.username, "email": user.email}
```

**Cross-Field Validation:**

```python
from pydantic import model_validator

class DateRange(BaseModel):
    start_date: datetime
    end_date: datetime
    
    @model_validator(mode='after')
    def validate_range(self):
        if self.start_date >= self.end_date:
            raise ValueError('start_date must be before end_date')
        return self
```

**Key Concepts:**
- `@field_validator` (Pydantic v2) for single-field validation
- `@model_validator` for cross-field validation
- `EmailStr` from `pydantic[email]`
- `Field()` for constraints and metadata
- `default_factory` for dynamic defaults
- Automatic validation in FastAPI

**Common Mistakes:**
- Using `@validator` in Pydantic v2 (use `@field_validator`)
- Not handling optional fields correctly
- Putting business logic in validators (should be in service layer)
- Not testing edge cases (empty strings, None)

**Interview Tip:**
> "Pydantic v2 uses `@field_validator` for single-field validation and `@model_validator` for cross-field checks. I use `Field()` for type constraints and `EmailStr` for email validation. FastAPI automatically validates request bodies, returning 422 errors with detailed messages. I keep validators simple—complex business logic goes in the service layer."

</details>

---

## Question 5: FastAPI Dependency Injection

**Difficulty:** Advanced  
**Category:** Learning

How does FastAPI's dependency injection work? Design a system for authentication and database access.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Core Concept:**

FastAPI's dependency injection system is built on Python's type hints. Dependencies are functions that can be injected into path operations, other dependencies, or global contexts.

**Database Session Dependency:**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from typing import Generator

DATABASE_URL = "postgresql://user:pass@localhost/dbname"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db  # Provide session to endpoint
    finally:
        db.close()  # Cleanup after request

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

**Authentication Dependency:**

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

security = HTTPBearer()

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db)
) -> User:
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        if not user_id:
            raise HTTPException(status_code=401, detail="Invalid token")
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user
```

**Role-Based Access Control:**

```python
def require_role(required_role: str):
    def role_checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role != required_role:
            raise HTTPException(
                status_code=403,
                detail=f"Requires {required_role} role"
            )
        return current_user
    return role_checker

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(require_role("admin"))
):
    return {"message": "User deleted"}
```

**Caching Dependencies:**

```python
from functools import lru_cache

@lru_cache()
def get_settings():
    return Settings()  # Singleton, cached

@app.get("/config")
async def get_config(settings: Settings = Depends(get_settings)):
    return settings
```

**Key Concepts:**
- Dependencies are callables that yield a value
- `Depends()` is evaluated per request
- Can be nested (dependency on dependency)
- `yield` enables cleanup logic
- `lru_cache` for singleton dependencies
- Global dependencies apply to all routes

**Common Mistakes:**
- Not using `yield` for cleanup (resource leaks)
- Heavy logic in dependencies (slow every request)
- Circular dependencies
- Not caching expensive dependencies

**Interview Tip:**
> "FastAPI's dependency injection uses type hints and `Depends()`. I use `yield` dependencies for resource management like database sessions—the `finally` block ensures cleanup. For authentication, I create a `get_current_user` dependency that other routes depend on. `lru_cache` is useful for singleton dependencies like settings."

</details>

---

## Question 6: Python Memory Management

**Difficulty:** Advanced  
**Category:** Learning

Explain Python's memory management. What is the GIL, and how does it affect multithreading?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Memory Management Components:**

1. **Reference Counting:** Primary mechanism
2. **Garbage Collector (GC):** Handles circular references
3. **Memory Pools:** Small object optimization (pymalloc)

**Reference Counting:**

```python
import sys

x = [1, 2, 3]
print(sys.getrefcount(x))  # 2 (x and getrefcount's argument)

y = x
print(sys.getrefcount(x))  # 3 (added y)

del y
print(sys.getrefcount(x))  # 2 (y removed)
# When refcount reaches 0, object is deallocated immediately
```

**Circular References:**

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

a = Node(1)
b = Node(2)
a.next = b
b.next = a  # Circular reference

del a, b
# Refcount won't reach 0, but GC will collect it
```

**The GIL (Global Interpreter Lock):**

The GIL allows only one thread to execute Python bytecode at a time. It's a mutex that protects Python's memory management.

```python
# Multithreading doesn't help CPU-bound work (GIL contention)
import threading

def cpu_bound_task():
    total = 0
    for i in range(10_000_000):
        total += i
    return total

threads = [threading.Thread(target=cpu_bound_task) for _ in range(4)]
for t in threads:
    t.start()
# Slower than sequential due to GIL overhead

# Multiprocessing bypasses GIL
from multiprocessing import Pool

with Pool(4) as pool:
    results = pool.map(cpu_bound_task, range(4))
# True parallelism
```

**When GIL is Released:**

The GIL is released during I/O operations:
- File operations
- Network requests
- Database queries
- C extensions (NumPy, etc.)

```python
# Threading works for I/O-bound tasks
import requests

def fetch_url(url):
    return requests.get(url)  # GIL released, threading helps

threads = [threading.Thread(target=fetch_url, args=(url,)) for url in urls]
```

**Memory Optimization:**

```python
# Use __slots__ for classes
class Point:
    __slots__ = ['x', 'y']  # No __dict__, saves memory
    def __init__(self, x, y):
        self.x = x
        self.y = y

# Use generators for large datasets
def read_large_file():
    with open('huge.txt') as f:
        for line in f:
            yield line
```

**Key Concepts:**
- Reference counting is primary, GC handles cycles
- GIL prevents true parallelism in threads
- Use multiprocessing for CPU-bound work
- Use threading/asyncio for I/O-bound work
- `__slots__` reduces memory for instances

**Common Mistakes:**
- Using threading for CPU-bound work (use multiprocessing)
- Not understanding GIL release during I/O
- Creating circular references (leaks memory)
- Not using generators for large files

**Interview Tip:**
> "Python uses reference counting as the primary memory management mechanism, with a generational garbage collector to handle circular references. The GIL prevents true parallelism for CPU-bound work—threading only helps for I/O-bound tasks. For CPU-intensive work, I use multiprocessing or C extensions like NumPy that release the GIL."

</details>

---

## Question 7: Context Managers

**Difficulty:** Intermediate  
**Category:** Learning

What is a context manager? Implement a custom context manager for timing code blocks.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

A **context manager** is an object that defines `__enter__` and `__exit__` methods, used with the `with` statement to ensure proper resource management (setup/teardown).

**Class-Based Context Manager:**

```python
import time
from contextlib import contextmanager

class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.end = time.perf_counter()
        self.elapsed = self.end - self.start
        print(f"Elapsed: {self.elapsed:.4f}s")
        return False  # Don't suppress exceptions

# Usage
with Timer() as t:
    time.sleep(1)
    print(f"Took {t.elapsed:.4f}s")
```

**Decorator-Based (Simpler):**

```python
from contextlib import contextmanager

@contextmanager
def timer(name: str = "Block"):
    start = time.perf_counter()
    try:
        yield  # Code block executes here
    finally:
        elapsed = time.perf_counter() - start
        print(f"{name} took {elapsed:.4f}s")

# Usage
with timer("Database query"):
    time.sleep(1)
```

**Database Transaction Manager:**

```python
@contextmanager
def transaction(db: Session):
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()

# Usage
with transaction(db) as session:
    session.add(user)
    session.add(profile)
    # If any operation fails, transaction is rolled back
```

**Async Context Manager:**

```python
from contextlib import asynccontextmanager
import aiohttp

@asynccontextmanager
async def http_session():
    session = aiohttp.ClientSession()
    try:
        yield session
    finally:
        await session.close()
```

**Key Concepts:**
- `__enter__` runs on `with` statement entry
- `__exit__` runs on exit (even with exceptions)
- `@contextmanager` decorator simplifies implementation
- `@asynccontextmanager` for async code
- `try/finally` ensures cleanup

**Common Mistakes:**
- Not handling exceptions in `__exit__`
- Forgetting to return `False` from `__exit__` (suppresses exceptions)
- Using context managers for non-resource code
- Not using `try/finally` in `@contextmanager` version

**Interview Tip:**
> "Context managers ensure proper resource cleanup using `__enter__` and `__exit__` methods. I use `@contextmanager` from `contextlib` for simpler implementations—it turns a generator with one `yield` into a context manager. The `finally` block guarantees cleanup even if exceptions occur. For async code, I use `@asynccontextmanager`."

</details>

---

## Question 8: Descriptors

**Difficulty:** Advanced  
**Category:** Learning

What is a Python descriptor? Implement a `@validated` descriptor for type checking.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

A **descriptor** is an object that defines `__get__`, `__set__`, or `__delete__`. It customizes attribute access on other objects. Properties, methods, and class-level attributes use descriptors.

**Basic Descriptor:**

```python
class Validator:
    def __init__(self, type_):
        self.type = type_
    
    def __set_name__(self, owner, name):
        self.name = name
    
    def __get__(self, obj, objtype=None):
        return obj.__dict__.get(self.name)
    
    def __set__(self, obj, value):
        if not isinstance(value, self.type):
            raise TypeError(f"Expected {self.type.__name__}, got {type(value).__name__}")
        obj.__dict__[self.name] = value

class Person:
    name = Validator(str)
    age = Validator(int)
    
    def __init__(self, name, age):
        self.name = name
        self.age = age

p = Person("Alice", 30)  # OK
p.age = "thirty"  # TypeError
```

**Property Descriptor (Built-in):**

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
        import math
        return math.pi * self._radius ** 2
```

**Key Concepts:**
- Descriptors control attribute access
- `__get__`, `__set__`, `__delete__` are descriptor methods
- Properties are descriptors
- `__set_name__` (Python 3.6+) auto-captures attribute name
- Enables validation, logging, lazy properties

**Common Mistakes:**
- Over-engineering with descriptors when properties suffice
- Storing descriptor state in the descriptor (should be in instance)
- Forgetting `__set_name__` for name capture

**Interview Tip:**
> "Descriptors customize attribute access by defining `__get__`, `__set__`, or `__delete__`. Python's `property` is a built-in descriptor. I use descriptors for validation, logging, or lazy loading. For most cases, plain properties are sufficient—descriptors are for reusable validation logic across multiple classes."

</details>

---

## Question 9: FastAPI Middleware

**Difficulty:** Advanced  
**Category:** Learning

How do you create custom middleware in FastAPI? Implement request logging and rate limiting.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Middleware Basics:**

Middleware in FastAPI is a function that processes every request before it reaches the endpoint, and every response before it's sent to the client.

**Request Logging Middleware:**

```python
from fastapi import FastAPI, Request
import time
import logging

app = FastAPI()
logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.perf_counter()
    
    logger.info(f"Request: {request.method} {request.url.path}")
    
    response = await call_next(request)
    
    process_time = time.perf_counter() - start_time
    logger.info(
        f"Response: {response.status_code} "
        f"({process_time:.4f}s)"
    )
    
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

**Rate Limiting Middleware:**

```python
from fastapi import HTTPException
from collections import defaultdict
from time import time

class RateLimiter:
    def __init__(self, requests_per_minute: int = 60):
        self.requests_per_minute = requests_per_minute
        self.requests = defaultdict(list)
    
    async def __call__(self, request: Request, call_next):
        client_ip = request.client.host
        now = time()
        minute_ago = now - 60
        
        # Clean old requests
        self.requests[client_ip] = [
            req_time for req_time in self.requests[client_ip]
            if req_time > minute_ago
        ]
        
        # Check limit
        if len(self.requests[client_ip]) >= self.requests_per_minute:
            raise HTTPException(
                status_code=429,
                detail="Rate limit exceeded"
            )
        
        self.requests[client_ip].append(now)
        return await call_next(request)

app.add_middleware(RateLimiter, requests_per_minute=100)
```

**CORS Middleware (Built-in):**

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Middleware Execution Order:**

```python
# Middleware executes in reverse order of addition
app.add_middleware(Middleware1)  # Executes 3rd
app.add_middleware(Middleware2)  # Executes 2nd
app.add_middleware(Middleware3)  # Executes 1st (closest to endpoint)

# Request flow: Client → M1 → M2 → M3 → Endpoint
# Response flow: Endpoint → M3 → M2 → M1 → Client
```

**Key Concepts:**
- Middleware uses `@app.middleware("http")` decorator
- `call_next` passes request to next handler
- Middleware executes in reverse order of addition
- Can modify request/response
- Can raise exceptions or return responses early

**Common Mistakes:**
- Heavy logic in middleware (slows every request)
- Not handling exceptions (crashes app)
- Adding too many middleware (performance overhead)
- Using sync functions (blocks event loop)

**Interview Tip:**
> "FastAPI middleware uses `app.middleware('http')` decorator. The function receives `request` and `call_next`. I use middleware for cross-cutting concerns: logging, CORS, rate limiting, authentication. Middleware executes in reverse order of addition, so the last added runs first. I avoid heavy logic in middleware—it's called for every request."

</details>

---

## Question 10: Async Generators and Streaming

**Difficulty:** Advanced  
**Category:** Learning

How do you stream responses in FastAPI? Implement Server-Sent Events (SSE) with async generators.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Streaming Response:**

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def generate_data():
    """Async generator for streaming"""
    for i in range(100):
        yield f"data: {i}\n\n"
        await asyncio.sleep(0.1)  # Simulate work

@app.get("/stream")
async def stream():
    return StreamingResponse(
        generate_data(),
        media_type="text/event-stream"
    )
```

**Server-Sent Events (SSE):**

```python
import json
from datetime import datetime

async def event_stream():
    """SSE with structured data"""
    for i in range(10):
        data = {
            "id": i,
            "timestamp": datetime.now().isoformat(),
            "message": f"Event {i}"
        }
        yield f"data: {json.dumps(data)}\n\n"
        await asyncio.sleep(1)

@app.get("/events")
async def events():
    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no"  # Disable Nginx buffering
        }
    )
```

**Streaming LLM Responses:**

```python
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def stream_llm_response(prompt: str):
    """Stream OpenAI responses"""
    stream = await client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )
    
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield f"data: {json.dumps({'text': chunk.choices[0].delta.content})}\n\n"

@app.post("/chat/stream")
async def chat_stream(prompt: str):
    return StreamingResponse(
        stream_llm_response(prompt),
        media_type="text/event-stream"
    )
```

**Client-Side Consumption (JavaScript):**

```javascript
const eventSource = new EventSource('/events');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log('Received:', data);
};
```

**Key Concepts:**
- `StreamingResponse` takes an async generator
- SSE uses `text/event-stream` media type
- `yield` sends chunks to client
- `await asyncio.sleep()` yields control to event loop
- Client uses `EventSource` or `fetch` with `ReadableStream`

**Common Mistakes:**
- Forgetting to set `X-Accel-Buffering: no` (Nginx buffers)
- Not handling client disconnection
- Blocking the event loop in the generator
- Not setting proper headers (CORS, Cache-Control)

**Interview Tip:**
> "I use `StreamingResponse` with async generators for streaming responses. For LLM outputs, I yield tokens as they arrive from the OpenAI stream. For SSE, I use `text/event-stream` media type and send `data: {json}\\n\\n` format. I always set `X-Accel-Buffering: no` to prevent proxy buffering."

</details>

---

## Question 11: Python `*args` and `**kwargs`

**Difficulty:** Beginner  
**Category:** Gap Identification

What are `*args` and `**kwargs`? When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definitions:**

- `*args`: Variable positional arguments (packed into a tuple)
- `**kwargs`: Variable keyword arguments (packed into a dict)

**`*args` Example:**

```python
def sum_all(*args):
    print(type(args))  # <class 'tuple'>
    return sum(args)

print(sum_all(1, 2, 3, 4))  # 10
print(sum_all(1, 2))         # 3
```

**`**kwargs` Example:**

```python
def print_info(**kwargs):
    print(type(kwargs))  # <class 'dict'>
    for key, value in kwargs.items():
        print(f"{key}: {value}")

print_info(name="Alice", age=30, city="NYC")
# Output:
# name: Alice
# age: 30
# city: NYC
```

**Combined Usage:**

```python
def func(required, *args, **kwargs):
    print(f"Required: {required}")
    print(f"Args: {args}")
    print(f"Kwargs: {kwargs}")

func("hello", 1, 2, 3, name="Alice", age=30)
```

**Unpacking Arguments:**

```python
def func(a, b, c):
    return a + b + c

# Unpacking list/tuple with *
args = [1, 2, 3]
print(func(*args))  # 6

# Unpacking dict with **
kwargs = {'a': 1, 'b': 2, 'c': 3}
print(func(**kwargs))  # 6
```

**Use Cases:**

```python
# 1. Decorators (most common)
def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

# 2. Class inheritance
class Child(Parent):
    def __init__(self, name, age, grade):
        super().__init__(name, age)
        self.grade = grade

# 3. Flexible APIs
def create_user(username, email, **optional):
    user = {'username': username, 'email': email}
    user.update(optional)
    return user
```

**Key Concepts:**
- `*args` packs positional args into tuple
- `**kwargs` packs keyword args into dict
- Order: positional, `*args`, keyword-only, `**kwargs`
- `*` and `**` unpack when calling functions
- Use in decorators, inheritance, flexible APIs

**Common Mistakes:**
- Wrong argument order (syntax error)
- Not using `*` to separate keyword-only args
- Using `*args`/`**kwargs` when specific args are clearer
- Not unpacking correctly when calling

**Interview Tip:**
> "`*args` packs positional arguments into a tuple, `**kwargs` packs keyword arguments into a dict. I use them in decorators to pass through any arguments, in class inheritance to forward to parent `__init__`, and in flexible APIs where users can pass optional fields. The order is: positional, `*args`, keyword-only, `**kwargs`."

</details>

---

## Question 12: Type Hints and `typing` Module

**Difficulty:** Intermediate  
**Category:** Learning

Explain advanced type hints: `Union`, `Optional`, `Generic`, `Protocol`, and `TypedDict`.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**`Union` and `Optional`:**

```python
from typing import Union, Optional

# Union: value can be one of several types
def process(data: Union[str, bytes, int]) -> str:
    if isinstance(data, str):
        return data.upper()
    elif isinstance(data, bytes):
        return data.decode()
    return str(data)

# Optional[X] is Union[X, None]
def find_user(user_id: int) -> Optional[User]:
    user = db.query(User).filter(User.id == user_id).first()
    return user  # User or None

# Python 3.10+ syntax
def process(data: str | bytes | int) -> str:
    pass

def find_user(user_id: int) -> User | None:
    pass
```

**`Generic`:**

```python
from typing import Generic, TypeVar

T = TypeVar('T')

class Repository(Generic[T]):
    def __init__(self, model: type[T]):
        self.model = model
    
    def get(self, id: int) -> T:
        return self.model.query.get(id)
    
    def save(self, entity: T) -> T:
        return entity

# Usage
user_repo = Repository[User](User)
user: User = user_repo.get(1)  # Type-checked as User
```

**`Protocol` (Structural Typing):**

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:
    def draw(self) -> None:
        print("Drawing circle")

class Square:
    def draw(self) -> None:
        print("Drawing square")

def render(shape: Drawable) -> None:
    shape.draw()

# Both work (duck typing with type safety)
render(Circle())
render(Square())
```

**`TypedDict`:**

```python
from typing import TypedDict, NotRequired

class UserDict(TypedDict):
    id: int
    name: str
    email: str
    age: NotRequired[int]  # Optional field

user: UserDict = {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com"
    # age is optional
}
```

**Key Concepts:**
- `Union[X, Y]` - value can be X or Y
- `Optional[X]` = `Union[X, None]`
- `Generic[T]` for type-parameterized classes
- `Protocol` for structural typing (duck typing)
- `TypedDict` for dict schemas
- `Literal` for specific values

**Common Mistakes:**
- Using `Union` when a single type is more specific
- Not using `Optional` for nullable returns
- Over-using `Any` instead of proper types
- Not using `Protocol` for duck typing

**Interview Tip:**
> "I use `Optional[T]` for nullable values, `Union[A, B]` for multiple types, and `Generic[T]` for reusable data structures. `Protocol` is great for structural typing—I can type-hint duck-typed functions without explicit inheritance. `TypedDict` is perfect for API responses and config dicts."

</details>

---

## Question 13: FastAPI Background Tasks

**Difficulty:** Intermediate  
**Category:** Learning

What are FastAPI BackgroundTasks? When would you use them vs. Celery?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

**BackgroundTasks** in FastAPI run functions after returning a response. They're part of the same process, not a separate worker.

**Basic Usage:**

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def send_email(email: str, message: str):
    # Send email (slow operation)
    print(f"Email sent to {email}")

@app.post("/register")
async def register(
    email: str,
    background_tasks: BackgroundTasks
):
    user = create_user(email)
    
    # Schedule email (runs after response)
    background_tasks.add_task(send_email, email, "Welcome!")
    
    return {"user_id": user.id, "message": "Registration successful"}
# Response is sent immediately
# Email is sent after response
```

**Multiple Tasks:**

```python
@app.post("/order")
async def create_order(
    items: list[str],
    background_tasks: BackgroundTasks
):
    order = create_order_in_db(items)
    
    background_tasks.add_task(send_confirmation_email, order.user_email)
    background_tasks.add_task(update_inventory, items)
    background_tasks.add_task(generate_invoice_pdf, order.id)
    background_tasks.add_task(notify_warehouse, order)
    
    return {"order_id": order.id, "status": "processing"}
```

**BackgroundTasks vs. Celery:**

| Aspect | BackgroundTasks | Celery |
|--------|-----------------|--------|
| Process | Same process | Separate workers |
| Reliability | Lost on crash | Persistent (with broker) |
| Scalability | Single server | Distributed |
| Use case | Fire-and-forget | Critical jobs |
| Retry logic | Manual | Built-in |
| Monitoring | Limited | Full-featured |

**When to Use BackgroundTasks:**

```python
# Good use cases
- Sending welcome emails
- Logging analytics events
- Generating thumbnails
- Cache invalidation
- Webhook delivery (low-stakes)

# Not suitable
- Payment processing (use Celery)
- Critical notifications (use Celery)
- Long-running tasks (use Celery)
- Tasks requiring retries (use Celery)
```

**Key Concepts:**
- `BackgroundTasks` runs after response is sent
- Same process (no separate worker)
- Lost on server crash
- Perfect for fire-and-forget tasks
- Use Celery for critical, persistent jobs

**Common Mistakes:**
- Using for critical operations (no persistence)
- Not handling errors (exceptions are silent)
- Using for long-running tasks (blocks worker)
- Not setting timeouts (can hang forever)

**Interview Tip:**
> "FastAPI's `BackgroundTasks` runs after the response is sent, in the same process. I use it for non-critical tasks like sending welcome emails, logging events, or generating thumbnails. For critical jobs that need persistence, retries, and monitoring, I use Celery with Redis or RabbitMQ."

</details>

---

## Question 14: Python `dataclasses` and Pydantic

**Difficulty:** Intermediate  
**Category:** Learning

Compare `dataclasses`, `attrs`, and `Pydantic` models. When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**`dataclasses` (Standard Library):**

```python
from dataclasses import dataclass, field
from typing import List, Optional
from datetime import datetime

@dataclass
class User:
    username: str
    email: str
    age: int
    roles: List[str] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    bio: Optional[str] = None
    
    def __post_init__(self):
        if self.age < 0:
            raise ValueError("Age cannot be negative")

user = User(username="alice", email="alice@example.com", age=30)
print(user.username)  # alice
```

**Pydantic:**

```python
from pydantic import BaseModel, Field, EmailStr, field_validator

class User(BaseModel):
    username: str = Field(..., min_length=3)
    email: EmailStr
    age: int = Field(..., ge=0, le=150)
    roles: List[str] = Field(default_factory=list)
    created_at: datetime = Field(default_factory=datetime.now)
    
    @field_validator('username')
    @classmethod
    def validate_username(cls, v):
        if not v.isalnum():
            raise ValueError("Username must be alphanumeric")
        return v.lower()

# Usage
user = User(username="alice", email="alice@example.com", age=30)

# JSON serialization
user_json = user.model_dump_json()
user_dict = user.model_dump()

# Validation errors
try:
    User(username="ab", email="invalid", age=-1)
except ValidationError as e:
    print(e.errors())
```

**Comparison Table:**

| Feature | `dataclasses` | Pydantic |
|---------|---------------|----------|
| Standard library | Yes | No |
| Validation | No | Yes (powerful) |
| Type coercion | No | Yes |
| JSON serialization | No | Built-in |
| Performance | Fast | Slower (validation overhead) |
| FastAPI integration | Manual | Native |

**When to Use Each:**

```python
# Use dataclasses for:
# - Internal data structures
# - DTOs without validation
# - Configuration objects
# - Performance-critical code

# Use Pydantic for:
# - API request/response models
# - External data validation
# - JSON parsing
# - FastAPI integration
# - Configuration with environment variables
```

**Pydantic Settings (Configuration):**

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str
    debug: bool = False
    
    class Config:
        env_file = ".env"
        case_sensitive = False

settings = Settings()  # Reads from environment or .env file
```

**Key Concepts:**
- `dataclasses`: Simple data containers, no validation
- Pydantic: Full validation, type coercion, JSON serialization
- FastAPI uses Pydantic by default
- Pydantic is slower but catches more errors

**Common Mistakes:**
- Using `dataclasses` for external data (no validation)
- Using Pydantic for internal data structures (overhead)
- Not using `frozen=True` for immutable data

**Interview Tip:**
> "I use `dataclasses` for internal data structures and DTOs—simple, fast, no dependencies. For API models in FastAPI, I use Pydantic—it provides validation, type coercion, and JSON serialization out of the box. The choice depends on validation needs: no validation → dataclasses, complex validation → Pydantic."

</details>

---

## Question 15: Python `itertools` and Functional Programming

**Difficulty:** Intermediate  
**Category:** Learning

Demonstrate useful `itertools` functions. How do you chain, filter, and transform data efficiently?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Chaining Iterables:**

```python
from itertools import chain

list1 = [1, 2, 3]
list2 = [4, 5, 6]

# chain: Combine iterables
combined = list(chain(list1, list2))
# [1, 2, 3, 4, 5, 6]

# chain.from_iterable: Flatten nested iterables
nested = [[1, 2], [3, 4], [5, 6]]
flat = list(chain.from_iterable(nested))
# [1, 2, 3, 4, 5, 6]
```

**Filtering and Selecting:**

```python
from itertools import filterfalse, dropwhile, takewhile, compress

data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# filterfalse: Opposite of filter
evens = list(filterfalse(lambda x: x % 2, data))
# [2, 4, 6, 8, 10]

# dropwhile: Drop items while condition is true
result = list(dropwhile(lambda x: x < 5, data))
# [5, 6, 7, 8, 9, 10]

# takewhile: Take items while condition is true
result = list(takewhile(lambda x: x < 5, data))
# [1, 2, 3, 4]

# compress: Filter using selectors
selectors = [1, 0, 1, 0, 1, 1, 0, 1, 0, 1]
result = list(compress(data, selectors))
# [1, 3, 5, 6, 8, 10]
```

**Grouping:**

```python
from itertools import groupby

# Group consecutive identical items
data = [1, 1, 1, 2, 2, 3, 3, 3, 3, 4]
groups = []
for key, group in groupby(data):
    groups.append((key, list(group)))
# [(1, [1, 1, 1]), (2, [2, 2]), (3, [3, 3, 3, 3]), (4, [4])]
```

**Combinatorics:**

```python
from itertools import product, permutations, combinations

# product: Cartesian product
colors = ['red', 'blue']
sizes = ['S', 'M', 'L']
for color, size in product(colors, sizes):
    print(f"{color} - {size}")

# permutations: All orderings
for p in permutations([1, 2, 3], 2):
    print(p)

# combinations: All subsets
for c in combinations([1, 2, 3, 4], 2):
    print(c)
```

**Infinite Iterators:**

```python
from itertools import count, cycle, repeat

# count: Infinite counter
for i in count(10, 2):  # Start at 10, step 2
    print(i)
    if i >= 20:
        break

# cycle: Cycle through iterable
counter = 0
for item in cycle(['A', 'B', 'C']):
    print(item)
    counter += 1
    if counter >= 7:
        break
```

**Key Concepts:**
- `chain`: Combine iterables
- `groupby`: Group consecutive items (requires sorted data)
- `product`, `permutations`, `combinations`: Combinatorics
- `islice`: Slice iterators
- `accumulate`: Running totals/products
- All return iterators (memory efficient)

**Common Mistakes:**
- Using `groupby` without sorting first
- Converting to list too early (defeats memory efficiency)
- Not understanding lazy evaluation
- Reinventing what itertools provides

**Interview Tip:**
> "`itertools` provides memory-efficient iterator operations. I use `chain` to combine iterables, `groupby` for grouping sorted data, `islice` to limit results, and `accumulate` for running totals. All functions return iterators, so they're perfect for large datasets."

</details>

---

## Question 16: Python `multiprocessing` vs `threading`

**Difficulty:** Advanced  
**Category:** Learning

When would you use `multiprocessing` vs `threading` vs `asyncio`?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Decision Tree:**

```
Is the task CPU-bound or I/O-bound?
├─ I/O-bound → Use asyncio or threading
│   ├─ Many concurrent I/O operations → asyncio (efficient)
│   └─ Simple parallel I/O → threading (simpler)
└─ CPU-bound → Use multiprocessing
```

**`threading` (I/O-bound, concurrent):**

```python
import threading
import requests

def fetch_url(url):
    response = requests.get(url)
    return len(response.content)

urls = ['https://example.com', 'https://google.com', 'https://github.com']

# Threading releases GIL during I/O
threads = []
for url in urls:
    t = threading.Thread(target=fetch_url, args=(url,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

**`multiprocessing` (CPU-bound, parallel):**

```python
import multiprocessing

def cpu_intensive_task(n):
    total = 0
    for i in range(n):
        total += i ** 2
    return total

if __name__ == '__main__':
    # Each process has its own Python interpreter and memory
    with multiprocessing.Pool(4) as pool:
        results = pool.map(cpu_intensive_task, [10**7] * 4)
    print(results)
```

**`asyncio` (I/O-bound, scalable):**

```python
import asyncio
import aiohttp

async def fetch_url(session, url):
    async with session.get(url) as response:
        return len(await response.text())

async def main():
    urls = ['https://example.com', 'https://google.com']
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    return results

asyncio.run(main())
```

**Comparison:**

| Aspect | `threading` | `multiprocessing` | `asyncio` |
|--------|-------------|-------------------|-----------|
| Parallelism | No (GIL) | Yes (separate processes) | No (single thread) |
| Concurrency | Yes | Yes | Yes |
| Memory | Shared | Separate per process | Shared |
| Overhead | Low | High (process creation) | Very low |
| Best for | I/O-bound | CPU-bound | Many I/O operations |

**Real-World Examples:**

```python
# Web scraping → asyncio
# Many HTTP requests, I/O-bound
async def scrape_urls(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# Image processing → multiprocessing
# CPU-intensive, parallel processing
from multiprocessing import Pool
def process_image(image_path):
    img = load_image(image_path)
    return apply_filter(img)

with Pool(8) as pool:
    results = pool.map(process_image, image_paths)
```

**Key Concepts:**
- `threading`: Concurrent I/O, GIL prevents CPU parallelism
- `multiprocessing`: True parallelism, separate memory
- `asyncio`: Scalable I/O concurrency, single thread
- CPU-bound → multiprocessing
- I/O-bound → asyncio or threading

**Common Mistakes:**
- Using threading for CPU-bound work (GIL bottleneck)
- Not using `if __name__ == '__main__'` in multiprocessing
- Sharing state between processes (use Queue or Manager)
- Mixing asyncio with blocking calls

**Interview Tip:**
> "For CPU-bound work, I use `multiprocessing` to bypass the GIL—each process has its own Python interpreter. For I/O-bound work with many concurrent operations, I prefer `asyncio` for its efficiency. For simple parallel I/O, `threading` works but has more overhead."

</details>

---

## Question 17: FastAPI WebSockets

**Difficulty:** Advanced  
**Category:** Learning

How do you implement WebSockets in FastAPI? Build a real-time chat application.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Basic WebSocket Endpoint:**

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Message received: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

**Real-Time Chat with Connection Manager:**

```python
from typing import List
from fastapi import WebSocket

class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
    
    async def send_personal_message(self, message: str, websocket: WebSocket):
        await websocket.send_text(message)
    
    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Client {client_id}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"Client {client_id} left")
```

**Chat with Multiple Rooms:**

```python
from collections import defaultdict

class RoomManager:
    def __init__(self):
        self.rooms: dict[str, List[WebSocket]] = defaultdict(list)
    
    async def connect(self, room: str, websocket: WebSocket):
        await websocket.accept()
        self.rooms[room].append(websocket)
    
    def disconnect(self, room: str, websocket: WebSocket):
        self.rooms[room].remove(websocket)
        if not self.rooms[room]:
            del self.rooms[room]
    
    async def send_to_room(self, room: str, message: dict, exclude: WebSocket = None):
        for connection in self.rooms[room]:
            if connection != exclude:
                await connection.send_json(message)

room_manager = RoomManager()

@app.websocket("/ws/{room}/{username}")
async def chat_room(
    websocket: WebSocket,
    room: str,
    username: str
):
    await room_manager.connect(room, websocket)
    try:
        while True:
            data = await websocket.receive_json()
            message = {
                "username": username,
                "text": data["text"],
                "timestamp": datetime.now().isoformat()
            }
            await room_manager.send_to_room(room, message, exclude=websocket)
    except WebSocketDisconnect:
        room_manager.disconnect(room, websocket)
```

**Key Concepts:**
- `WebSocket` for full-duplex communication
- `ConnectionManager` tracks active connections
- `receive_text()`, `receive_json()`, `send_text()`, `send_json()`
- `WebSocketDisconnect` exception on disconnection
- Use rooms for multi-user chat

**Common Mistakes:**
- Not handling `WebSocketDisconnect` (resource leak)
- Broadcasting without exclusion (echo to sender)
- No authentication (security risk)
- Not cleaning up on disconnect

**Interview Tip:**
> "FastAPI WebSockets enable real-time bidirectional communication. I use a `ConnectionManager` to track active connections and broadcast messages. For multi-room chat, I use a `RoomManager` with a dict of room → connections. WebSockets are perfect for chat, live notifications, and real-time dashboards."

</details>

---

## Question 18: Python `__init__.py` and Packages

**Difficulty:** Beginner  
**Category:** Gap Identification

How do Python packages work? What's the difference between a module and a package?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definitions:**

- **Module:** A single `.py` file
- **Package:** A directory containing `__init__.py` and modules

**Package Structure:**

```
mypackage/
├── __init__.py          # Makes it a package
├── module1.py
├── module2.py
└── subpackage/
    ├── __init__.py
    └── module3.py
```

**`__init__.py` Purpose:**

```python
# mypackage/__init__.py

# 1. Empty (just marks directory as package)

# 2. Package-level imports
from .module1 import MyClass
from .module2 import helper_function

# 3. Package metadata
__version__ = "1.0.0"
__author__ = "Alice"

# 4. Make submodules available
from . import subpackage
```

**Importing:**

```python
# Import entire module
import mypackage.module1
obj = mypackage.module1.MyClass()

# Import specific item
from mypackage.module1 import MyClass
obj = MyClass()

# Import from package (if re-exported in __init__.py)
from mypackage import MyClass

# Import subpackage
from mypackage.subpackage import module3
```

**Relative vs. Absolute Imports:**

```python
# mypackage/subpackage/module3.py

# Absolute import (recommended)
from mypackage.module1 import MyClass

# Relative import
from ..module1 import MyClass  # Go up two levels
from .sibling import other_function
```

**`__all__` Variable:**

```python
# mypackage/module1.py
__all__ = ['MyClass', 'helper_function']

def MyClass:
    pass

def helper_function():
    pass

def _private_function():  # Not in __all__
    pass
```

```python
# When using: from module1 import *
from mypackage.module1 import *  # Only imports MyClass and helper_function
```

**Key Concepts:**
- Module = single `.py` file
- Package = directory with `__init__.py`
- `__init__.py` runs on package import
- `__all__` controls `from package import *`
- Absolute imports are preferred over relative

**Common Mistakes:**
- Circular imports
- Using relative imports across packages
- Putting too much logic in `__init__.py`
- Not using `__all__` for public API
- Missing `__init__.py` (in older Python versions)

**Interview Tip:**
> "A module is a single Python file; a package is a directory with `__init__.py`. The `__init__.py` file runs when the package is imported—I use it for package-level imports and metadata. I prefer absolute imports (`from myproject.api import routes`) over relative ones for clarity."

</details>

---

## Question 19: FastAPI Testing with `pytest`

**Difficulty:** Advanced  
**Category:** Learning

How do you test FastAPI applications? Write tests for endpoints with database and authentication.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Basic Test Setup:**

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from myapp.main import app, get_db
from myapp.models import Base

# Test database
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False}
)
TestingSessionLocal = sessionmaker(bind=engine)

@pytest.fixture
def db():
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client(db):
    def override_get_db():
        try:
            yield db
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()
```

**Testing Endpoints:**

```python
# tests/test_users.py

def test_create_user(client):
    response = client.post(
        "/users",
        json={
            "username": "alice",
            "email": "alice@example.com",
            "password": "Secret123!"
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["username"] == "alice"
    assert "id" in data
    assert "password" not in data  # Should be excluded

def test_get_user(client, db):
    # Create user first
    user = create_test_user(db, username="bob")
    
    response = client.get(f"/users/{user.id}")
    assert response.status_code == 200
    assert response.json()["username"] == "bob"

def test_get_nonexistent_user(client):
    response = client.get("/users/999")
    assert response.status_code == 404

def test_invalid_email(client):
    response = client.post(
        "/users",
        json={
            "username": "alice",
            "email": "invalid-email",
            "password": "Secret123!"
        }
    )
    assert response.status_code == 422  # Validation error
```

**Testing with Authentication:**

```python
# tests/conftest.py
@pytest.fixture
def auth_token(client, db):
    user = create_test_user(
        db,
        username="testuser",
        email="test@example.com"
    )
    response = client.post(
        "/auth/login",
        json={"username": "testuser", "password": "testpass"}
    )
    return response.json()["access_token"]

@pytest.fixture
def auth_headers(auth_token):
    return {"Authorization": f"Bearer {auth_token}"}

# tests/test_protected.py
def test_protected_endpoint_requires_auth(client):
    response = client.get("/profile")
    assert response.status_code == 401

def test_protected_endpoint_with_auth(client, auth_headers):
    response = client.get("/profile", headers=auth_headers)
    assert response.status_code == 200
    assert response.json()["username"] == "testuser"
```

**Key Concepts:**
- `TestClient` for synchronous testing
- `httpx.AsyncClient` for async testing
- Fixtures for setup/teardown
- `dependency_overrides` for dependency injection
- Test database separate from production
- Parametrize for testing multiple cases

**Common Mistakes:**
- Using production database in tests
- Not cleaning up between tests (data leaks)
- Testing implementation details instead of behavior
- Not using fixtures (code duplication)

**Interview Tip:**
> "I use `TestClient` for synchronous tests and `httpx.AsyncClient` with `ASGITransport` for async tests. I create a separate test database and use `dependency_overrides` to inject test dependencies. Fixtures handle setup and teardown. For authentication, I create a fixture that generates a valid token."

</details>

---

## Question 20: Python `__init__`, `__new__`, and `__del__`

**Difficulty:** Advanced  
**Category:** Learning

What's the difference between `__init__` and `__new__`? When would you override each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**`__new__` - Object Creation:**

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        print(f"Creating instance of {cls.__name__}")
        instance = super().__new__(cls)
        return instance

obj = MyClass()  # "Creating instance of MyClass"
```

**`__init__` - Object Initialization:**

```python
class MyClass:
    def __init__(self, value):
        print(f"Initializing with {value}")
        self.value = value

obj = MyClass(42)  # "Initializing with 42"
```

**Key Differences:**

| Method | Purpose | Returns | When Called |
|--------|---------|---------|-------------|
| `__new__` | Create instance | Instance | Before `__init__` |
| `__init__` | Initialize instance | None | After `__new__` |
| `__del__` | Cleanup | None | When garbage collected |

**When to Override `__new__`:**

**1. Singleton Pattern:**

```python
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

a = Singleton()
b = Singleton()
print(a is b)  # True
```

**2. Immutable Subclass:**

```python
class UpperStr(str):
    def __new__(cls, value):
        return super().__new__(cls, value.upper())

s = UpperStr("hello")
print(s)  # "HELLO"
```

**3. Factory Pattern:**

```python
class Shape:
    def __new__(cls, shape_type, *args, **kwargs):
        if shape_type == "circle":
            return Circle(*args, **kwargs)
        elif shape_type == "square":
            return Square(*args, **kwargs)
        return super().__new__(cls)

# Usage
shape1 = Shape("circle", radius=5)
shape2 = Shape("square", side=4)
```

**When to Override `__init__`:**

```python
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email
        self.created_at = datetime.now()
        self._validate()
    
    def _validate(self):
        if not self.username:
            raise ValueError("Username required")
```

**`__del__` - Destructor (Use Sparingly):**

```python
# Better: Use context manager
class FileHandler:
    def __init__(self, filename):
        self.file = open(filename, 'w')
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self.file.close()
    
    def write(self, data):
        self.file.write(data)

with FileHandler("log.txt") as f:
    f.write("Hello")
# File is closed automatically
```

**Key Concepts:**
- `__new__` creates and returns instance (called first)
- `__init__` initializes instance (called after `__new__`)
- `__new__` is a classmethod (takes `cls` as first arg)
- `__init__` is an instance method (takes `self` as first arg)
- Use `__new__` for singletons, immutables, factory patterns
- Use `__init__` for regular initialization

**Common Mistakes:**
- Forgetting to return instance from `__new__`
- Overriding `__del__` for resource cleanup (use context managers)
- Not calling `super().__new__(cls)` in `__new__`

**Interview Tip:**
> "`__new__` creates the instance and is called first—it's a classmethod that must return an instance. `__init__` initializes the instance and is called after. I use `__new__` for the singleton pattern, immutable subclasses like `UpperStr`, and factory patterns. For regular initialization, I use `__init__`. I avoid `__del__`—it has unpredictable timing, so I use context managers for cleanup instead."

</details>

---

## Question 21: FastAPI Dependency Injection: Advanced

**Difficulty:** Advanced  
**Category:** Learning

Implement dependency injection with database sessions, caching, and complex authentication.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Layered Dependencies:**

```python
from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import Generator
import jwt
from datetime import datetime, timedelta

# Database dependency
def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Token utilities
def create_access_token(user_id: int) -> str:
    expire = datetime.utcnow() + timedelta(hours=24)
    payload = {"sub": user_id, "exp": expire}
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")

def decode_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expired")
    except jwt.PyJWTError:
        raise HTTPException(401, "Invalid token")

# Authentication dependency
def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    payload = decode_token(token)
    user_id = payload.get("sub")
    
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(401, "User not found")
    
    if not user.is_active:
        raise HTTPException(403, "User inactive")
    
    return user

# Role-based dependency factory
def require_role(*allowed_roles: str):
    def role_checker(
        current_user: User = Depends(get_current_user)
    ) -> User:
        if current_user.role not in allowed_roles:
            raise HTTPException(403, f"Requires one of: {allowed_roles}")
        return current_user
    return role_checker

# Usage
@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(require_role("admin", "superadmin"))
):
    pass
```

**Caching with Dependencies:**

```python
from functools import lru_cache

class Settings:
    def __init__(self):
        self.database_url = os.getenv("DATABASE_URL")
        self.api_key = os.getenv("API_KEY")
        self.debug = os.getenv("DEBUG", "False") == "True"

@lru_cache()
def get_settings() -> Settings:
    return Settings()  # Singleton, cached

@app.get("/config")
async def get_config(settings: Settings = Depends(get_settings)):
    return {"debug": settings.debug}
```

**Pagination Dependency:**

```python
from dataclasses import dataclass
from fastapi import Query

@dataclass
class Pagination:
    skip: int = 0
    limit: int = 10
    total: int = 0
    
    def paginate(self, query):
        self.total = query.count()
        return query.offset(self.skip).limit(self.limit).all()

def get_pagination(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
) -> Pagination:
    return Pagination(skip=skip, limit=limit)

@app.get("/items")
async def list_items(
    pagination: Pagination = Depends(get_pagination),
    db: Session = Depends(get_db)
):
    items = pagination.paginate(db.query(Item))
    return {
        "items": items,
        "total": pagination.total,
        "skip": pagination.skip,
        "limit": pagination.limit
    }
```

**Global Dependencies:**

```python
# Apply to all routes
app = FastAPI(dependencies=[Depends(verify_api_key)])

# Apply to router
admin_router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(require_role("admin"))]
)
```

**Override Dependencies in Tests:**

```python
# tests/conftest.py
def override_get_db():
    try:
        db = TestingSessionLocal()
        yield db
    finally:
        db.close()

def override_get_current_user():
    return User(id=1, username="test", email="test@example.com")

# In test setup
app.dependency_overrides[get_db] = override_get_db
app.dependency_overrides[get_current_user] = override_get_current_user
```

**Key Concepts:**
- Dependencies can be nested
- Use factories for parameterized dependencies
- `lru_cache` for singleton dependencies
- Global dependencies for cross-cutting concerns
- Override dependencies in tests

**Common Mistakes:**
- Not using factories for parameterized dependencies
- Heavy logic in dependencies
- Not caching expensive dependencies
- Circular dependencies

**Interview Tip:**
> "I use FastAPI's dependency injection for cross-cutting concerns: database sessions, authentication, authorization, pagination, and settings. For role-based access, I create a `require_role` factory that returns a dependency. I use `lru_cache` for singleton dependencies like settings. In tests, I use `dependency_overrides` to inject test versions."

</details>

---

## Question 22: Python `pathlib` vs `os.path`

**Difficulty:** Beginner  
**Category:** Learning

Compare `pathlib.Path` and `os.path`. Which is preferred in modern Python?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**`os.path` (String-based):**

```python
import os

# Joining paths
path = os.path.join("folder", "subfolder", "file.txt")
# "folder/subfolder/file.txt"

# Checking existence
exists = os.path.exists(path)
is_file = os.path.isfile(path)
is_dir = os.path.isdir(path)

# Getting components
dirname = os.path.dirname(path)
basename = os.path.basename(path)
name, ext = os.path.splitext(basename)

# Creating directories
os.mkdir("new_folder")
os.makedirs("nested/folders", exist_ok=True)

# Listing directory
files = os.listdir("folder")
```

**`pathlib.Path` (Object-based):**

```python
from pathlib import Path

# Creating paths
path = Path("folder") / "subfolder" / "file.txt"
# PosixPath('folder/subfolder/file.txt')

# Checking existence
exists = path.exists()
is_file = path.is_file()
is_dir = path.is_dir()

# Getting components
dirname = path.parent
basename = path.name
name = path.stem
ext = path.suffix
parent = path.parent

# Creating directories
Path("new_folder").mkdir()
Path("nested/folders").mkdir(parents=True, exist_ok=True)

# Listing directory
files = list(Path("folder").iterdir())
```

**Modern Python Prefers `pathlib`:**

```python
# os.path (less readable)
import os

config_path = os.path.join(
    os.path.expanduser("~"),
    ".config",
    "myapp",
    "config.json"
)
if os.path.exists(config_path) and os.path.isfile(config_path):
    with open(config_path) as f:
        config = f.read()

# pathlib (more readable)
from pathlib import Path

config_path = Path.home() / ".config" / "myapp" / "config.json"
if config_path.is_file():
    config = config_path.read_text()
```

**File Operations:**

```python
# pathlib makes file operations easier
path = Path("data.txt")

# Reading
content = path.read_text()
data = path.read_bytes()
lines = path.read_text().splitlines()

# Writing
path.write_text("Hello")
path.write_bytes(b"Binary data")

# Get file info
size = path.stat().st_size
modified = path.stat().st_mtime

# Path manipulation
abs_path = path.absolute()
resolved = path.resolve()  # Follows symlinks
relative = path.relative_to("/home/user")
```

**Glob and Pattern Matching:**

```python
# Find all Python files
for py_file in Path("src").rglob("*.py"):
    print(py_file)

# Find files matching pattern
for config in Path(".").glob("config*.json"):
    print(config)

# Match pattern
if path.match("*.txt"):
    print("Text file")
```

**Key Concepts:**
- `pathlib.Path` is object-oriented
- `/` operator for path joining
- Methods for file operations (read, write, etc.)
- Cross-platform (handles Windows/Linux/Mac)
- `rglob()` for recursive search

**Common Mistakes:**
- Mixing `os.path` and `pathlib` (use one consistently)
- Using `str(path)` when `Path` is expected
- Forgetting `parents=True` for nested directories
- Not using `exist_ok=True` (raises error if exists)

**Interview Tip:**
> "I prefer `pathlib.Path` over `os.path` in modern Python. It provides an object-oriented API with the `/` operator for path joining, methods for file I/O, and cross-platform handling. `path.read_text()` is cleaner than `with open() as f: f.read()`."

</details>

---

## Question 23: Python `logging` Best Practices

**Difficulty:** Intermediate  
**Category:** Learning

How do you set up structured logging in a Python application?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Basic Setup:**

```python
import logging

# Simple configuration
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)
logger.info("Application started")
```

**Advanced Configuration:**

```python
# logging_config.py
import logging
import logging.config
from logging.handlers import RotatingFileHandler
import json
from datetime import datetime

# JSON formatter for structured logging
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": datetime.utcfromtimestamp(record.created).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }
        
        if record.exc_info:
            log_entry["exception"] = self.formatException(record.exc_info)
        
        if hasattr(record, "user_id"):
            log_entry["user_id"] = record.user_id
        
        return json.dumps(log_entry)

# Configuration
LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {"()": JSONFormatter},
        "standard": {
            "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        }
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "standard",
            "level": "INFO"
        },
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "logs/app.log",
            "maxBytes": 10485760,  # 10MB
            "backupCount": 5,
            "formatter": "json",
            "level": "INFO"
        }
    },
    "loggers": {
        "": {  # Root logger
            "handlers": ["console", "file"],
            "level": "INFO"
        }
    }
}

logging.config.dictConfig(LOGGING_CONFIG)
logger = logging.getLogger("myapp")
```

**Usage in Application:**

```python
# myapp/services/user_service.py
import logging

logger = logging.getLogger(__name__)

class UserService:
    def create_user(self, username: str, email: str):
        logger.info(
            "Creating user",
            extra={"username": username, "email": email}
        )
        
        try:
            user = db.add(User(username=username, email=email))
            db.commit()
            logger.info(
                "User created successfully",
                extra={"user_id": user.id, "username": username}
            )
            return user
        except IntegrityError as e:
            logger.error(
                "Failed to create user: duplicate",
                extra={"username": username, "email": email},
                exc_info=True
            )
            raise
```

**Performance Optimization:**

```python
# Use lazy formatting
logger.debug("User %s with ID %d", username, user_id)  # Lazy
logger.debug(f"User {username} with ID {user_id}")  # Eager (always formatted)

# Check level before expensive operations
if logger.isEnabledFor(logging.DEBUG):
    expensive_data = compute_expensive_debug_info()
    logger.debug("Debug info: %s", expensive_data)
```

**Key Concepts:**
- Use `getLogger(__name__)` for module-specific loggers
- JSON formatter for structured logs
- Rotating file handlers for log rotation
- `extra` parameter for custom fields
- `exc_info=True` or `logger.exception()` for stack traces
- Lazy formatting with `%s` placeholders

**Common Mistakes:**
- Using `print()` instead of logging
- Not using `__name__` for logger names
- Logging sensitive data (passwords, tokens)
- Not rotating log files (disk fills up)
- Using eager string formatting (use `%s` placeholders)

**Interview Tip:**
> "I use Python's `logging` module with JSON formatting for structured logs. Each module gets its own logger with `getLogger(__name__)`. I add custom fields with `extra={}` and use context variables for request IDs. For production, I use rotating file handlers and ship logs to centralized logging (ELK, Datadog). I avoid logging sensitive data and use lazy `%s` formatting for performance."

</details>

---

## Question 24: Python `__slots__` and Memory Optimization

**Difficulty:** Intermediate  
**Category:** Learning

What is `__slots__`? When should you use it?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

`__slots__` is a class variable that tells Python to **not** create a `__dict__` for instances, saving memory by pre-defining which attributes can exist.

**Without `__slots__`:**

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p.__dict__)  # {'x': 1, 'y': 2}

import sys
print(sys.getsizeof(p))  # ~56 bytes + dict overhead
```

**With `__slots__`:**

```python
class Point:
    __slots__ = ['x', 'y']
    
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p.__dict__)  # AttributeError: no __dict__

import sys
print(sys.getsizeof(p))  # ~48 bytes, no dict
```

**Memory Comparison:**

```python
# Without slots
class WithoutSlots:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

# With slots
class WithSlots:
    __slots__ = ['x', 'y', 'z']
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

# Create 1 million instances
objects_without = [WithoutSlots(i, i, i) for i in range(1000000)]
objects_with = [WithSlots(i, i, i) for i in range(1000000)]

# With slots: ~3x memory savings
```

**Limitations of `__slots__`:**

```python
class Point:
    __slots__ = ['x', 'y']

p = Point(1, 2)
p.z = 3  # AttributeError: 'Point' object has no attribute 'z'

# Can't add new attributes not in __slots__
# No __dict__ or __weakref__ by default
```

**Inheritance with `__slots__`:**

```python
class Base:
    __slots__ = ['x']

class Child(Base):
    __slots__ = ['y']  # Must define __slots__ to maintain optimization
    # Child has both x and y

c = Child()
c.x = 1  # OK
c.y = 2  # OK
c.z = 3  # AttributeError
```

**When to Use `__slots__`:**

```python
# Use for:
# - Classes with many instances (millions+)
# - Data classes with fixed attributes
# - Performance-critical code
# - Memory-constrained environments

# Don't use for:
# - Classes with dynamic attributes
# - Classes that need __dict__ for serialization
# - Small number of instances
# - When using multiple inheritance with conflicting slots
```

**Key Concepts:**
- `__slots__` prevents `__dict__` creation
- Saves memory (no per-instance dict)
- Faster attribute access
- Can't add attributes not in `__slots__`
- Child classes need their own `__slots__`

**Common Mistakes:**
- Forgetting `__slots__` in child classes (loses optimization)
- Trying to add attributes not in `__slots__`
- Using with classes that need dynamic attributes
- Not measuring actual memory savings

**Interview Tip:**
> "`__slots__` is a class optimization that prevents the creation of `__dict__` for instances, saving significant memory for classes with many instances. I use it for data classes and ORM models with millions of instances—it can reduce memory by 3-5x. The trade-off is you can't add attributes dynamically. Child classes need their own `__slots__` to maintain the optimization."

</details>

---

## Question 25: Python `lru_cache` and Memoization

**Difficulty:** Intermediate  
**Category:** Learning

How does `functools.lru_cache` work? Implement custom memoization.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Built-in `lru_cache`:**

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# First call: slow
print(fibonacci(100))  # Fast despite recursive implementation
```

**How It Works:**

- **LRU = Least Recently Used**
- Stores results in a dict
- When cache is full, removes least recently used entry
- Thread-safe (uses internal locking)
- Cache key is argument tuple

**Cache Configuration:**

```python
# Max size
@lru_cache(maxsize=128)  # Default: 128
def expensive_func(x):
    return x ** 2

# Unlimited cache
@lru_cache(maxsize=None)
def expensive_func(x):
    return x ** 2

# Check cache info
print(expensive_func.cache_info())
# CacheInfo(hits=10, misses=5, maxsize=128, currsize=5)

# Clear cache
expensive_func.cache_clear()
```

**Custom Memoization:**

```python
def memoize(func):
    cache = {}
    
    def wrapper(*args, **kwargs):
        # Create cache key from args and kwargs
        key = (args, tuple(sorted(kwargs.items())))
        
        if key not in cache:
            cache[key] = func(*args, **kwargs)
        
        return cache[key]
    
    wrapper.cache = cache
    wrapper.cache_clear = lambda: cache.clear()
    return wrapper

@memoize
def expensive_function(x, y):
    return x ** y
```

**Time-Based Cache (TTL):**

```python
import time
from functools import wraps

def timed_cache(seconds=60):
    def decorator(func):
        cache = {}
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            key = (args, tuple(sorted(kwargs.items())))
            
            if key in cache:
                result, timestamp = cache[key]
                if time.time() - timestamp < seconds:
                    return result
            
            result = func(*args, **kwargs)
            cache[key] = (result, time.time())
            return result
        
        wrapper.cache_clear = lambda: cache.clear()
        return wrapper
    return decorator

@timed_cache(seconds=60)
def fetch_data(url):
    return requests.get(url).json()
```

**When to Use Memoization:**

```python
# Good use cases:
# - Expensive recursive functions (fibonacci, factorial)
# - API calls with same parameters
# - Database queries with same filters
# - Mathematical computations
# - File parsing with same input

# Bad use cases:
# - Functions with side effects
# - Functions returning mutable objects
# - Functions with unhashable arguments
# - When memory is limited
# - When results change frequently
```

**Key Concepts:**
- `lru_cache` memoizes function results
- LRU eviction when cache is full
- Cache key is argument tuple
- Thread-safe and fast
- `cache_info()` for statistics
- `cache_clear()` to reset
- TTL cache for time-based expiration

**Common Mistakes:**
- Using with mutable arguments
- Not setting appropriate `maxsize`
- Caching functions with side effects
- Memory leaks from large cache values
- Forgetting to clear cache in tests

**Interview Tip:**
> "I use `lru_cache` for expensive functions called repeatedly with the same arguments—API calls, recursive algorithms, database queries. The cache uses LRU eviction to prevent unbounded growth. For time-based expiration, I create a custom TTL cache. I always check `cache_info()` to measure hit rate and `cache_clear()` to reset in tests."

</details>

---

## Question 26: FastAPI Error Handling

**Difficulty:** Intermediate  
**Category:** Learning

How do you handle errors and exceptions in FastAPI?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**HTTPException:**

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = db.get(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found",
            headers={"X-Error": "User not found"}
        )
    return user
```

**Custom Exception Handlers:**

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id

class InsufficientFundsError(Exception):
    def __init__(self, balance: float, required: float):
        self.balance = balance
        self.required = required

# Register exception handlers
@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(
    request: Request,
    exc: UserNotFoundError
):
    return JSONResponse(
        status_code=404,
        content={
            "error": "UserNotFound",
            "message": f"User {exc.user_id} not found",
            "user_id": exc.user_id
        }
    )

@app.exception_handler(InsufficientFundsError)
async def insufficient_funds_handler(
    request: Request,
    exc: InsufficientFundsError
):
    return JSONResponse(
        status_code=402,
        content={
            "error": "InsufficientFunds",
            "message": f"Required {exc.required}, have {exc.balance}",
        }
    )

# Usage
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = db.get(user_id)
    if not user:
        raise UserNotFoundError(user_id)
    return user
```

**Validation Error Handling:**

```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(
    request: Request,
    exc: RequestValidationError
):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": ".".join(str(x) for x in error["loc"]),
            "message": error["msg"],
            "type": error["type"]
        })
    
    return JSONResponse(
        status_code=422,
        content={
            "error": "ValidationError",
            "message": "Request validation failed",
            "details": errors
        }
    )
```

**Global Exception Handler:**

```python
import logging
import traceback

logger = logging.getLogger(__name__)

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(
        f"Unhandled exception: {exc}",
        extra={
            "path": request.url.path,
            "method": request.method,
            "traceback": traceback.format_exc()
        },
        exc_info=True
    )
    
    return JSONResponse(
        status_code=500,
        content={
            "error": "InternalServerError",
            "message": "An unexpected error occurred"
        }
    )
```

**Database Error Handling:**

```python
from sqlalchemy.exc import IntegrityError, SQLAlchemyError

@app.post("/users")
async def create_user(user: UserCreate, db: Session = Depends(get_db)):
    try:
        db_user = User(**user.dict())
        db.add(db_user)
        db.commit()
        return db_user
    except IntegrityError as e:
        db.rollback()
        if "unique constraint" in str(e.orig).lower():
            raise HTTPException(
                status_code=409,
                detail="User already exists"
            )
        raise HTTPException(500, "Database error")
    except SQLAlchemyError as e:
        db.rollback()
        logger.exception("Database error")
        raise HTTPException(500, "Database error")
```

**Key Concepts:**
- `HTTPException` for standard HTTP errors
- Custom exception handlers with `@app.exception_handler`
- Validation errors handled by FastAPI (422)
- Global handler for unhandled exceptions
- Log errors with full context
- Return structured error responses

**Common Mistakes:**
- Not logging errors (silent failures)
- Exposing sensitive info in error messages
- Using 500 for client errors
- Not handling database errors (connection leaks)
- Inconsistent error response format

**Interview Tip:**
> "I use `HTTPException` for standard HTTP errors and create custom exception classes for business logic. Each exception has a handler that returns a structured error response with error code and message. I log all errors with full context (path, method, traceback). For database errors, I rollback the transaction and return a user-friendly message."

</details>

---

## Question 27: Python `asyncio` Advanced Patterns

**Difficulty:** Advanced  
**Category:** Learning

Demonstrate advanced asyncio patterns: tasks, queues, locks, and semaphores.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Tasks and Cancellation:**

```python
import asyncio

async def long_running_task():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("Task cancelled")
        raise  # Always re-raise CancelledError

async def main():
    task = asyncio.create_task(long_running_task())
    
    # Cancel after 2 seconds
    await asyncio.sleep(2)
    task.cancel()
    
    try:
        await task
    except asyncio.CancelledError:
        print("Task was cancelled")

# Timeout
async def with_timeout():
    try:
        result = await asyncio.wait_for(
            long_running_task(),
            timeout=5.0
        )
    except asyncio.TimeoutError:
        print("Task took too long")
```

**TaskGroup (Python 3.11+):**

```python
async def fetch_user(user_id: int):
    await asyncio.sleep(1)
    if user_id == 3:
        raise ValueError("User 3 not found")
    return {"id": user_id}

async def main():
    # Structured concurrency - all tasks run together
    # If one fails, all are cancelled
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch_user(1))
        task2 = tg.create_task(fetch_user(2))
        task3 = tg.create_task(fetch_user(3))  # This fails
    
    # All tasks cancelled, exception raised
```

**Queues for Producer-Consumer:**

```python
async def producer(queue: asyncio.Queue):
    for i in range(10):
        await queue.put(i)
        await asyncio.sleep(0.1)
    await queue.put(None)  # Signal end

async def consumer(queue: asyncio.Queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"Processing {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=5)  # Bounded queue
    
    prod_task = asyncio.create_task(producer(queue))
    cons_task = asyncio.create_task(consumer(queue))
    
    await prod_task
    await cons_task
```

**Locks for Mutual Exclusion:**

```python
import asyncio

balance_lock = asyncio.Lock()
balance = 0

async def deposit(amount: int):
    global balance
    async with balance_lock:
        temp = balance
        await asyncio.sleep(0.1)  # Simulate processing
        balance = temp + amount
        # Lock released automatically
```

**Semaphores for Rate Limiting:**

```python
import asyncio

# Limit to 3 concurrent requests
semaphore = asyncio.Semaphore(3)

async def fetch_url(url: str):
    async with semaphore:
        # Only 3 of these run concurrently
        print(f"Fetching {url}")
        await asyncio.sleep(1)
        return f"Result of {url}"

async def main():
    urls = [f"url{i}" for i in range(10)]
    tasks = [fetch_url(url) for url in urls]
    results = await asyncio.gather(*tasks)
```

**Events for Coordination:**

```python
import asyncio

# Event to signal readiness
ready_event = asyncio.Event()

async def waiter():
    print("Waiting for event...")
    await ready_event.wait()
    print("Event triggered!")

async def setter():
    await asyncio.sleep(2)
    print("Setting event")
    ready_event.set()

async def main():
    await asyncio.gather(waiter(), setter())
```

**Gather vs. Wait:**

```python
# gather: Returns results in order, raises first exception
async def gather_example():
    results = await asyncio.gather(
        fetch("url1"),
        fetch("url2"),
        return_exceptions=True  # Don't raise, return exceptions
    )

# wait: More control over completion
async def wait_example():
    tasks = [
        asyncio.create_task(fetch("url1")),
        asyncio.create_task(fetch("url2"))
    ]
    
    done, pending = await asyncio.wait(
        tasks,
        timeout=5.0,
        return_when=asyncio.FIRST_COMPLETED
    )
    
    # Process completed tasks
    for task in done:
        print(await task)
    
    # Cancel pending
    for task in pending:
        task.cancel()
```

**Key Concepts:**
- `create_task()` for fire-and-forget tasks
- `TaskGroup` for structured concurrency (3.11+)
- `asyncio.Queue` for producer-consumer
- `asyncio.Lock` for mutual exclusion
- `asyncio.Semaphore` for rate limiting
- `asyncio.Event` for coordination
- `asyncio.gather()` for parallel execution
- `asyncio.wait_for()` for timeouts

**Common Mistakes:**
- Forgetting to await tasks (coroutine warnings)
- Not handling `CancelledError` properly
- Using locks in single-threaded code (unnecessary)
- Not cleaning up resources on cancellation

**Interview Tip:**
> "I use `asyncio.TaskGroup` for structured concurrency—it ensures all tasks complete or are cancelled together. For producer-consumer patterns, I use `asyncio.Queue`. For rate limiting, I use `asyncio.Semaphore` to limit concurrent operations. I always handle `CancelledError` properly and clean up resources in `finally` blocks."

</details>

---

## Question 28: Python Type Checking with `mypy`

**Difficulty:** Intermediate  
**Category:** Learning

How do you set up static type checking in a Python project?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Installation:**

```bash
pip install mypy
# Or with strict checking
pip install mypy[strict]
```

**Basic Configuration:**

```ini
# mypy.ini or setup.cfg
[mypy]
python_version = 3.11
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
disallow_any_generics = True
disallow_untyped_calls = True
check_untyped_defs = True
no_implicit_optional = True
warn_redundant_casts = True
warn_unused_ignores = True
warn_no_return = True
strict_optional = True
```

**Running mypy:**

```bash
mypy mypackage/
mypy --strict mypackage/
mypy --show-error-codes mypackage/
```

**Type Annotations:**

```python
from typing import List, Dict, Optional, Union, Tuple

def process_items(
    items: List[str],
    config: Optional[Dict[str, str]] = None
) -> Tuple[int, List[str]]:
    if config is None:
        config = {}
    processed = [item.upper() for item in items]
    count = len(processed)
    return count, processed

# Python 3.9+ built-in generics
def process_items(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}
```

**Type Aliases:**

```python
from typing import TypeAlias

# Simple alias
UserId: TypeAlias = int
UserDict: TypeAlias = Dict[str, Union[str, int]]

# Complex alias
JsonValue: TypeAlias = Union[
    None, bool, int, float, str, list, dict
]

def parse_json(data: str) -> JsonValue:
    import json
    return json.loads(data)
```

**Generic Types:**

```python
from typing import Generic, TypeVar, Iterable

T = TypeVar('T')
K = TypeVar('K')
V = TypeVar('V')

class Repository(Generic[T]):
    def get(self, id: int) -> T:
        ...
    
    def save(self, entity: T) -> None:
        ...

class Cache(Generic[K, V]):
    def get(self, key: K) -> Optional[V]:
        ...
    
    def set(self, key: K, value: V) -> None:
        ...
```

**Protocols for Structural Typing:**

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

def render_all(items: Iterable[Drawable]) -> None:
    for item in items:
        item.draw()

# Any class with a draw() method works
class Circle:
    def draw(self) -> None:
        print("Drawing circle")

render_all([Circle()])  # OK
```

**Stub Files (`.pyi`):**

```python
# mypackage/untyped_module.py
import untyped_library

def process(data):  # No type hints
    return untyped_library.do_something(data)

# mypackage/untyped_module.pyi (stub file)
import untyped_library
from typing import Any

def process(data: str) -> dict[str, Any]: ...
```

**Ignoring Type Errors:**

```python
# Type: ignore comment
result = untyped_function()  # type: ignore

# Specific error code
result = untyped_function()  # type: ignore[no-untyped-call]
```

**Integrating with CI:**

```yaml
# .github/workflows/type-check.yml
name: Type Check
on: [push, pull_request]
jobs:
  mypy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
      - run: pip install mypy
      - run: mypy mypackage/
```

**Key Concepts:**
- `mypy` is the standard static type checker
- Type hints are optional but recommended
- Strict mode catches more errors
- Stub files (`.pyi`) for untyped libraries
- Type aliases improve readability
- Protocols for structural typing

**Common Mistakes:**
- Using `Any` too liberally
- Not handling `Optional` types properly
- Missing type hints in function signatures
- Not using strict mode in CI

**Interview Tip:**
> "I use `mypy` in strict mode for static type checking. It catches type errors before runtime, improves IDE autocomplete, and serves as documentation. I use `Protocol` for structural typing, `TypeAlias` for complex types, and stub files (`.pyi`) for untyped third-party libraries. I run mypy in CI to prevent type errors from being merged."

</details>

---

## Question 29: Python `concurrent.futures`

**Difficulty:** Advanced  
**Category:** Learning

How do you use `concurrent.futures` for parallel execution?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**`ThreadPoolExecutor` (I/O-bound):**

```python
from concurrent.futures import ThreadPoolExecutor
import requests

def fetch_url(url: str) -> str:
    response = requests.get(url)
    return response.text

urls = [
    "https://example.com",
    "https://google.com",
    "https://github.com"
]

# Execute concurrently with threads
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(fetch_url, url) for url in urls]
    
    # Get results as they complete
    for future in futures:
        result = future.result()  # Blocks until done
        print(f"Got {len(result)} bytes")

# Or use map for simpler cases
with ThreadPoolExecutor(max_workers=5) as executor:
    results = executor.map(fetch_url, urls)
    for result in results:
        print(f"Got {len(result)} bytes")
```

**`ProcessPoolExecutor` (CPU-bound):**

```python
from concurrent.futures import ProcessPoolExecutor

def cpu_intensive_task(n: int) -> int:
    total = 0
    for i in range(n):
        total += i ** 2
    return total

# Execute in parallel processes
with ProcessPoolExecutor(max_workers=4) as executor:
    futures = [
        executor.submit(cpu_intensive_task, 10**7)
        for _ in range(4)
    ]
    
    results = [f.result() for f in futures]
    print(results)
```

**Handling Exceptions:**

```python
from concurrent.futures import ThreadPoolExecutor

def might_fail(n: int) -> int:
    if n == 0:
        raise ValueError("Cannot process zero")
    return 100 // n

with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(might_fail, n) for n in [10, 0, 5]]
    
    for future in futures:
        try:
            result = future.result()
            print(f"Result: {result}")
        except ValueError as e:
            print(f"Error: {e}")
```

**Callbacks:**

```python
from concurrent.futures import ThreadPoolExecutor

def task(n: int) -> int:
    return n ** 2

def callback(future):
    try:
        result = future.result()
        print(f"Completed: {result}")
    except Exception as e:
        print(f"Failed: {e}")

with ThreadPoolExecutor(max_workers=3) as executor:
    for i in range(5):
        future = executor.submit(task, i)
        future.add_done_callback(callback)
```

**`as_completed` for Results as They Complete:**

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def fetch_with_delay(url: str, delay: float) -> str:
    time.sleep(delay)
    return f"Result of {url}"

urls = [
    ("url1", 0.1),
    ("url2", 0.5),
    ("url3", 0.2)
]

with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [
        executor.submit(fetch_with_delay, url, delay)
        for url, delay in urls
    ]
    
    # Process results as they complete (not in order)
    for future in as_completed(futures):
        result = future.result()
        print(result)
```

**Comparison with asyncio:**

```python
# concurrent.futures - simpler API, callback-based
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor() as executor:
    results = executor.map(fetch_url, urls)

# asyncio - more control, async/await syntax
import asyncio
import aiohttp

async def fetch_all():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks)

asyncio.run(fetch_all())
```

**Key Concepts:**
- `ThreadPoolExecutor` for I/O-bound work
- `ProcessPoolExecutor` for CPU-bound work
- `submit()` returns `Future` object
- `map()` for simple parallel mapping
- `as_completed()` for results as they finish
- `add_done_callback()` for notifications
- Context manager ensures cleanup

**Common Mistakes:**
- Using `ProcessPoolExecutor` for I/O-bound work (overhead)
- Not handling exceptions in futures
- Forgetting to call `result()` (future is lazy)
- Sharing state between processes (use Queue)

**Interview Tip:**
> "I use `ThreadPoolExecutor` for I/O-bound parallelism and `ProcessPoolExecutor` for CPU-bound work. The API is simpler than raw threading/multiprocessing—I use `submit()` for individual tasks and `map()` for simple parallel mapping. For results as they complete, I use `as_completed()`. Always use the context manager to ensure proper cleanup."

</details>

---

## Question 30: Python Packaging and Distribution

**Difficulty:** Intermediate  
**Category:** Learning

How do you package and distribute a Python library?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Modern Structure with `pyproject.toml`:**

```
myproject/
├── pyproject.toml
├── README.md
├── LICENSE
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── main.py
│       └── utils.py
└── tests/
    ├── __init__.py
    └── test_main.py
```

**`pyproject.toml`:**

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "myproject"
version = "0.1.0"
description = "A short description"
readme = "README.md"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "you@example.com"}
]
requires-python = ">=3.8"
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
]
dependencies = [
    "requests>=2.28.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "mypy>=1.0",
    "black>=22.0",
]

[project.urls]
Homepage = "https://github.com/you/myproject"
Issues = "https://github.com/you/myproject/issues"

[project.scripts]
myproject-cli = "myproject.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**Building the Package:**

```bash
# Install build tools
pip install build

# Build distribution
python -m build
# Creates dist/myproject-0.1.0.tar.gz (source)
# Creates dist/myproject-0.1.0-py3-none-any.whl (wheel)

# Install locally
pip install -e .
```

**Publishing to PyPI:**

```bash
# Install twine
pip install twine

# Upload to Test PyPI first
python -m twine upload --repository testpypi dist/*

# Upload to PyPI
python -m twine upload dist/*
```

**Version Management:**

```python
# myproject/__init__.py
__version__ = "0.1.0"
```

**Semantic Versioning:**

- **MAJOR:** Breaking changes (1.0.0)
- **MINOR:** New features, backward compatible (0.1.0)
- **PATCH:** Bug fixes (0.0.1)

**Dependencies:**

```toml
# Exact version (avoid)
dependencies = [
    "requests==2.28.0"
]

# Minimum version (preferred)
dependencies = [
    "requests>=2.28.0"
]

# Version range
dependencies = [
    "requests>=2.28.0,<3.0.0"
]

# Compatible release
dependencies = [
    "requests~=2.28.0"  # >=2.28.0, <2.29.0
]
```

**Entry Points (CLI):**

```toml
[project.scripts]
myproject-cli = "myproject.cli:main"
```

```python
# myproject/cli.py
def main():
    """Entry point for CLI"""
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--name", required=True)
    args = parser.parse_args()
    print(f"Hello, {args.name}!")

# After install: myproject-cli --name Alice
# Output: Hello, Alice!
```

**GitHub Actions for Automated Publishing:**

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI

on:
  release:
    types: [created]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
      - run: pip install build
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
```

**Key Concepts:**
- `pyproject.toml` is the modern standard (PEP 621)
- Use `src/` layout to avoid import conflicts
- Build with `python -m build`
- Upload with `twine`
- Semantic versioning (MAJOR.MINOR.PATCH)
- Pin minimum versions, not exact versions
- Entry points for CLI tools

**Common Mistakes:**
- Not using `src/` layout (causes import issues)
- Pinning exact versions (causes conflicts)
- Not testing on Test PyPI first
- Missing `__init__.py` files
- Forgetting to bump version before publishing
- Not including `LICENSE` file

**Interview Tip:**
> "I use the modern `pyproject.toml` format with the `src/` layout for Python packages. I build with `python -m build` and publish with `twine`. I follow semantic versioning and pin minimum versions to avoid conflicts. For CLI tools, I use entry points in `pyproject.toml`. I always test on Test PyPI before publishing to PyPI."

</details>

---

## Summary Checklist

- [x] Decorators fundamentals and patterns
- [x] Generator functions and expressions
- [x] Async/await and event loop
- [x] Pydantic custom validators
- [x] FastAPI dependency injection
- [x] Python memory management and GIL
- [x] Context managers
- [x] Descriptors
- [x] FastAPI middleware
- [x] Async generators and streaming
- [x] `*args` and `**kwargs`
- [x] Type hints and `typing` module
- [x] FastAPI BackgroundTasks
- [x] `dataclasses` vs Pydantic
- [x] `itertools` and functional programming
- [x] `multiprocessing` vs `threading` vs `asyncio`
- [x] FastAPI WebSockets
- [x] Python packages and `__init__.py`
- [x] FastAPI testing with `pytest`
- [x] `__init__` vs `__new__` vs `__del__`
- [x] Advanced FastAPI dependencies
- [x] `pathlib` vs `os.path`
- [x] Python `logging` best practices
- [x] `__slots__` and memory optimization
- [x] `lru_cache` and memoization
- [x] FastAPI error handling
- [x] Advanced asyncio patterns
- [x] `mypy` static type checking
- [x] `concurrent.futures`
- [x] Python packaging and distribution

**Total: 30 questions** covering Python internals, FastAPI patterns, and practical applications.