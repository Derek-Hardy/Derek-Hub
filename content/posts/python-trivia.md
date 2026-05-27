+++
date = '2026-03-07T16:58:35+08:00'
draft = false
title = 'Python Pitfalls: Know Them Once, Avoid Them Forever'
+++

Python is very intuitive to write but some behaviours are subtle enough that they can quietly turn into bugs if I don't keep them in mind.

Put down a summary here for the Python pitfalls that I find most practical to remember.
Most of them are not really "weird" once we understand the language rule behind it, but they are exactly the kind of details that can bite in debugging.

### Late binding in closures

Closure means a function can still refer to variables from the place where it is defined, even after that surrounding code has finished running.

In the below example, each `lambda` is a tiny function created inside the loop.
The variable `i` is not defined inside the `lambda` itself, so Python will look it up from the surrounding scope when the `lambda` runs.

The tricky part is that Python captures the variable name, not the value at the time the function is defined.

```python
funcs = []

for i in range(5):
    # each lambda looks up i only when it is called
    funcs.append(lambda: i)

print([f() for f in funcs])  # [4, 4, 4, 4, 4]
```

All lambdas end up using the final value of `i`, because the loop does not create a new scope for each iteration.

The usual fix is to force evaluation at definition time with a default argument:

```python
funcs = []

for i in range(5):
    funcs.append(lambda i=i: i)

print([f() for f in funcs])  # [0, 1, 2, 3, 4]
```

### Mutable default arguments

Function default arguments are evaluated once when the function is defined, not every time the function is called.

So if the default value is mutable, it can accidentally become shared state.

```python
def add(val, data=[]):
    data.append(val)
    return data

print(add(1))  # [1]
print(add(2))  # [1, 2]
```

This is almost never what I want from a helper function. The safer pattern is to use `None` as a sentinel value:

```python
def add(val, data=None):
    if data is None:
        data = []
    data.append(val)
    return data

print(add(1))  # [1]
print(add(2))  # [2]
```

### Class attribute shared across instances

Class attributes are shared by all instances unless an instance shadows the attribute.

That is perfectly fine for constants, but pretty dangerous for mutable containers.

```python
class RequestTracker:
    recent_paths = []

    def record(self, path):
        self.recent_paths.append(path)


user_api = RequestTracker()
payment_api = RequestTracker()

user_api.record("/users/123")
payment_api.record("/payments/abc")

print(user_api.recent_paths)     # ['/users/123', '/payments/abc']
print(payment_api.recent_paths)  # ['/users/123', '/payments/abc']
```

In most cases, the list should live on the instance instead:

```python
class RequestTracker:
    def __init__(self):
        self.recent_paths = []

    def record(self, path):
        self.recent_paths.append(path)
```

Dataclasses are quite helpful here because they explicitly reject mutable literal defaults.
The intended pattern is `default_factory`:

```python
from dataclasses import dataclass, field

@dataclass
class Test:
    data: list = field(default_factory=list)
```

### Assignment is alias, not copy

This one is probably the most important Python habit to internalise.
Assignment binds a name to an object; it does not copy the object.

Even when we copy a list with slicing, it is still only a shallow copy.

```python
x = [[1, 2], [3, 4]]
y = x[:]

y[0].append(999)

print(x)  # [[1, 2, 999], [3, 4]]
print(y)  # [[1, 2, 999], [3, 4]]
```

The outer list is copied, but the inner lists are still the same objects.
If we need the whole nested structure to be independent, use `copy.deepcopy`.

```python
import copy

x = [[1, 2], [3, 4]]
y = copy.deepcopy(x)

y[0].append(999)

print(x)  # [[1, 2], [3, 4]]
print(y)  # [[1, 2, 999], [3, 4]]
```

It is not that deep copy should be used everywhere.
It is more that I should be very clear whether I am copying the container, the elements, or both.

### List multiplication with nested lists

This is another version of the same reference issue.

```python
grid = [[0] * 3] * 3
grid[0][0] = 1

print(grid)
# [[1, 0, 0], [1, 0, 0], [1, 0, 0]]
```

The inner list is created once and reused three times.

The safer way is to create each row independently:

```python
grid = [[0] * 3 for _ in range(3)]
grid[0][0] = 1

print(grid)
# [[1, 0, 0], [0, 0, 0], [0, 0, 0]]
```

### Scope is decided at compile time

Python decides whether a name is local by looking at the whole function body.
If there is an assignment to the name anywhere inside the function, Python treats it as local unless we say otherwise.

```python
x = 10

def inc():
    x += 1
    return x

inc()  # UnboundLocalError: cannot access local variable 'x'
```

It may look like `x += 1` should read the global variable first, but the assignment makes `x` a local name for the whole function.

If we really want to modify the global binding, we can say it explicitly:

```python
x = 10

def inc():
    global x
    x += 1
    return x


print(inc())  # 11
print(inc())  # 12
```

For nested functions, `nonlocal` binds to the nearest enclosing function scope.

```python
def run():
    count = 0

    def inc():
        # without nonlocal, count += 1 would make count local to inc
        # and Python would complain before it can read the outer count
        nonlocal count
        count += 1
        return count

    print(inc())  # 1
    print(inc())  # 2


run()
```

### Loop variables leak

Regular `for` loops do not create their own scope.

```python
for j in range(3):
    pass

print(j)  # 2
```

The loop variable remains available after the loop finishes.

List comprehensions behave differently in Python 3:

```python
j = "outer value"
_ = [j for j in range(3)]

print(j)  # outer value
```

This difference is small, but it explains why loop-related closure bugs can be surprising if we mentally assume every iteration owns a fresh scope.

### Floating point comparison

Decimal values like `0.1` cannot always be represented exactly in binary floating point.

```python
print(0.1 + 0.1 + 0.1 == 0.3)  # False
```

For practical comparison, use `math.isclose`.

```python
import math

print(math.isclose(0.1 + 0.1 + 0.1, 0.3))  # True
```

### Empty any() and all()

The behaviour of `any` and `all` on empty iterables can be unintuitive at first.

```python
print(any([]))  # False
print(all([]))  # True
```

I find it easier to think of them as loops that stop early.

```python
def my_any(values):
    for v in values:
        if v:
            return True
    return False


def my_all(values):
    for v in values:
        if not v:
            return False
    return True
```

For `any([])`, the loop never finds a truthy value, so it returns `False`.
For `all([])`, the loop never finds a falsy value, so it returns `True`.

That is why an empty list can accidentally pass a validation like this:

```python
scores = []

print(all(score >= 60 for score in scores))  # True
```

This matters when filtering collections.
If an empty list should be treated as invalid input, it is better to check emptiness directly before using `all`.

### finally can override return and exceptions

`finally` is guaranteed to run, which makes it a good place for cleanup.
However, returning from `finally` can hide what happened in `try`.

```python
def f():
    try:
        1 / 0
    finally:
        return 42

print(f())  # 42
```

The `ZeroDivisionError` is discarded.

It can also override a normal return:

```python
def g():
    try:
        return "try"
    finally:
        return "finally"

print(g())  # finally
```

So my rule is simple: use `finally` for cleanup, not for deciding the function result.
If cleanup needs to be structured, a context manager is usually clearer.

### Unpacking is practical

Not all Python trivia is a trap.
Some features are just handy to remember.

Extended unpacking is one of them:

```python
a, *b = [1, 2, 3]

print(a)  # 1
print(b)  # [2, 3]
```

It also works nicely when I only care about the first or last few values:

```python
head, *middle, tail = [1, 2, 3, 4, 5]

print(head)    # 1
print(middle)  # [2, 3, 4]
print(tail)    # 5
```
