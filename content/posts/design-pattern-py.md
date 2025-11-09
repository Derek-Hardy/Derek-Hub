+++
date = '2025-09-23T19:00:53+08:00'
draft = false
title = 'Design Patterns in Python'
+++

As a super flexible language, Python is blazing fast to build working prototypes and get the steam going. 
At the same time, it can also end up becoming a messy codebase and a pain to read later if being careless in the coding style.

Therefore, I would always prefer to look out for idiomatic patterns to keep the Python code maintainable and easy to work with (both for me and others).

### Singleton

It means only one instance of the class should exist during a program's lifetime.

A refresher on the Python object creation process:
* `__new__(cls)`: the very first step to allocate memory for the raw instance
* `__init__(self)`: fills in the details (attributes) for the new instance object returned by the above step

Now, we want to have a shared cache object:

```python3
import time

class SingleCache:
    _instance = None # class-level attribute
    
    def __new__(cls):
        # allocate new instance only when there isn't one; otherwise it reuses the same one
        if not cls._instance:
            cls._instance = super().__new__(cls) # all class inherits from 'object'
            cls._instance._cache = {}
        return cls._instance

    def set(self, k, v, ttl=10):
        expiry = time.time() + ttl
        self._cache[k] = (v, expiry) # self = cls._instance = obj returned by __new__()

    def get(self, k):
        if k not in self._cache:
            return None
        v, expiry = self._cache[k]
        if time.time() > expiry:
            del self._cache[k]
            return None
        return v
```

Another thing to note, in Python, modules are imported once per interpreter. Every import just reuses the same module object.
So we can actually define the singleton object at the module level.

```python
# single_cache.py
class SingleCache:
    def __init__(self):
        print("initialise all attributes...")
        self._cache = {}

cache = SingleCache()

# and then use it from other places
from single_cache import SingleCache
```

### Decorator

It is arguably the most Python-native design pattern more than anything else. 
The decorator wraps around the core function to add behaviours immediately before and after its execution.

From my experience, it is most useful to add observability, caching and recovery mechanism around functions in the least disturbing way possible.

It looks self-explanatory in its minimal form:

```python
def decorator(func):
    def wrapper(*args, **kwargs): # just pass in positional & keyword arguments that func accepts
        print("set up before the func")
        result = func(*args, **kwargs)
        print("modifications after the func")
        return result
    return wrapper
```

A practical example if we want to profile compute-heavy operations in the time dimension is to have a timer decorator:

```python
import time
from functools import wraps

def time_it(func):
    # preserve original func metadata (__name__, __doc__) instead of being overridden by wrapper function
    # in order to keep a less confusing call stack
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} completed in {elapsed:.2f} seconds")
        return result
    return wrapper

# now we use it
# '@time_it' is actually a syntactic sugar for the full form: 
# slow_compute = time_it(slow_compute)
@time_it
def slow_compute():
    time.sleep(20)
    return "done"
```

In fact, there are a lot of pre-built decorators we can just plug in and use:
* `@timing`: measure function execution time
* `@log_exceptions`: log exception with the function name
* `@functools.lru_cache(maxsize=16)`: cache function results to be re-used for the same calling arguments
* `@retry(times=3, delay=1)`: re-execute the function that might fail due to transient errors (e.g. rate limits)
* `@singleton`: decorate around class definition to enforce only one instance is ever created

### Iterator

It allows us to traverse through a collection of elements sequentially without worrying about how the internal data structure looks like in order to do it.

The pattern boils down to a protocol of 2 APIs:
* `__iter__()`: return an iterator
* `__next()`: return the next item, or raise `StopIteration` when we exhaust the collection

Any object that implements the above two is an iterator. Any object that implements only `__iter__()` is an iterable.

It might look natural that we traverse through a list using for-loop:
```python
nums = [1, 2, 3]
for n in nums:
    print(n) # 1, 2, 3
```

And the for-loop syntax is actually interpreted as:
```python
it = iter(nums)
while True:
    try:
        n = next(it)
    except StopIteration:
        break
```

A practical use is when we want to process a huge log file, instead of loading everything in memory at once, we can design an iterator
to load the data bit by bit for more efficient memory footprint.

```python
class BigFileIterator:
    def __init__(self, path, chunk_size=1024):
        self.path = path
        self.chunk_size = chunk_size
        self.buffer = ""
        self.file = None

    def __iter__(self):
        self.file = open(self.path, "r")
        return self

    def __next__(self):
        while True:
            # only return one line at a time
            newline = self.buffer.find("\n")
            if newline != -1:
                line = self.buffer[:newline]
                self.buffer = self.buffer[newline + 1:]
                return line

            # only load 'chunk_size' amount of data into memory buffer at a time
            chunk = self.file.read(self.chunk_size)
            if not chunk:
                if self.buffer:
                    return self._flush()  # EOF - flush the remaining buffer
                self.file.close()
                raise StopIteration
            self.buffer += chunk

    def _flush(self):
        _line = self.buffer
        self.buffer = ""
        return _line
```

This is just an illustration of how lazy loading can be effective in dealing with large data using iterator. 
In reality, it would be better to design such usage as a **context manager** where file handle is properly opened and closed upon enter and exit.


### Builder

It allows us to construct complex objects with many optional parameters step by step instead of having a gigantic constructor or assigning attributes all over the place.

The builder uses method chaining to make the same task look cleaner and more extensible, and encourages immutability after creation.

For instance, we want to build a custom HTTP client. It can be done in a very modular way:

```python
from dataclasses import dataclass

@dataclass # automatically generate __init__ method
class HttpClientConfig:
    base_url: str
    time_out: int
    retries: int
    headers: dict
    
class HttpClient:
    def __init__(self, config: HttpClientConfig):
        self._config = config
        self._init_session()

    def _init_session(self):
        print("set up the request session...")

    def get(self, path: str):
        print(f"GET {self._config.base_url}/{path}")

class HttpClientBuilder:
    def __init__(self, base_url: str):
        self._base_url = base_url
        self._time_out = None
        self._retries = None
        self._headers = None
    
    def with_timeout(self, seconds: int):
        self._time_out = seconds
    
    def with_retries(self, retries: int):
        self._retries = retries
    
    def with_headers(self, headers: dict):
        self._headers = headers

    def build(self):
        cfg = HttpClientConfig(
            base_url=self._base_url,
            time_out=self._time_out,
            retries=self._retries,
            headers=self._headers
        )
        return HttpClient(cfg)
```

### Observer

Because functions are first-class in Python, it is easy to create a simple observer pattern to broadcast events to all interested components.

```python
class Notification:
    def __init__(self):
        self.subscribers = []

    def subscribe(self, call_back):
        self.subscribers.append(call_back)

    def notify(self, event):
        for fn in self.subscribers:
            fn(event)
```

