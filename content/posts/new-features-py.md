+++
date = '2025-11-02T15:18:31+08:00'
draft = false
title = 'New Python Features 3.10 - 3.14'
+++

Starting from Python 3.10 (released Oct 2021) to Python 3.14 (released last month), there are quite a few new features
added to the language that made it more intuitive to use.

I want to put a summary here to remind myself and always try to make full use of the modern Pythonic idioms. 
Starting with the one I'm most excited about:

### Max heap! (3.14)

For a long time, Python only natively supports min heap. Users are sort of forced to play the trick 
to negate the value whenever we find max heap is needed for the problem. This is very error-prone
in that it's super easy to forget about the '-' sign on the way in or out.

Finally, we have the native max heap:

```python
from heapq import heapify_max, heappush_max, heappop_max

arr = [5,7,2,4,8]
heapify_max(arr)
print(arr) # [8, 7, 2, 4, 5]

points = []
heappush_max(points, (2, 'b'))
heappush_max(points, (1, 'a'))
heappush_max(points, (3, 'c'))
print(heappop_max(points)) # (3, 'c')
```

### Deferred evaluation of annotation (3.14)

Ever since Python 3 introduced type annotation, it is always evaluated immediately. In some cases, when we want to reference
the type that is not yet defined, we will have to do some trickery by quoting it.

```python
def make_pair(x: "T", y: "T") -> "Pair[T]":
    print(f"we will combine x and y into a Pair...")
```

Alternatively, we need to manually defer the evaluation:

```python
from __future__ import annotations

def make_pair(x: T, y: T) -> Pair[T]:
    print(f"we will combine x and y into a Pair...")
```

Now, in 3.14 we can forget about all the hassles to simply define the function to reference what we need.

```python
# no problem here
def make_pair(x: T, y: T) -> Pair[T]:
    return Pair(x, y)

# defined later
class Pair[T]:
    def __init__(self, a: T, b: T):
        self.a, self.b = a, b
```

### t-string (3.14)

It builds a Template object that defers the evaluation of the formatted string. In contrast, f-string does eager evaluation.

I realised that this feature is merged later the dev cycle this year and not really active in the official 3.14 release when I tried.

```python
# what to look out for in Python 3.15
from string import Template

log_template = t"[{level}] received {count} messages"
print(log_template.format(level="INFO", count=1))
```

### f-string revamp (3.12)

Previously the allowed expressions in f-string format is pretty restrictive (in fact built with a custom mini-parser)

In 3.12, we can do more things about it:

```python
# multi-line
name = "Derek"
print(f"""
Hi, this is {name.upper()}!
This is another line.
""")

# support comprehensions/assignments
arr = [1, 2, 3]
print(f"{[n+1 for n in arr]}") # output: [2, 3, 4]

# {expr =} auto-expand print to both expression text and evaluated value
# pretty useful as a debugging shortcut
print(f"{[n+1 for n in arr] = }") # output: [n+1 for n in arr] = [2, 3, 4]
```

### easier type alias (3.12)

This is used when we want to give a more descriptive name to an existing data type that makes
the code more readable in the given context.

It is pretty much built-in for Golang:
```go
type Frequency map[string]int
```

In older Python, however, it was somewhat verbose:
```python
from typing import TypeAlias
Frequency: TypeAlias = dict[str, int]

# for generics
from typing import TypeVar
T = TypeVar("T")
Count: TypeAlias = dict[T, int]
```

Now, the new release supports minimalist style just like in Golang:
```python
type Frequency = dict[str, int]
# generics
type Count = dict[T, int]
```

### Pattern matching (3.10)

We can now forgo the long and repetitive if-clauses when switch-case is clearly a more readable option

```python
from enum import Enum

class Command(Enum):
    RUN = 1
    STOP = 2
    KILL = 3
    CANCEL = 4

def execute(cmd: Command) -> str:
    match cmd:
        case Command.RUN:
            return "execute RUN command"
        case Command.STOP | Command.KILL: # match multiple values
            return "execute STOP/KILL command"
        case _: # default case
            return "not implemented"

# we can also match on the structure
def evaluate(expr: str) -> int:
    match expr:
        case ("ADD", a, b):
            return evaluate(a) + evaluate(b)
        case ("MUL", a, b):
            return evaluate(a) * evaluate(b)
        case int(n):
            return n
        # simply exits if there is no matching case and no wildcard case
```

### Type unions (3.10)

We can easily define multiple allowed input/output types in the typing annotation

```python
def generate(n: int) -> int | str:
    return 0 if n % 2 == 0 else "odd"
```







