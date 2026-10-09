# Python Syntax: Full Revision Guide

*Revision guide for typing and testing by hand. Python 3.12+. Predict the output before you run each snippet.*

## How to use this guide

- Type every example by hand (no copy-paste), run it, and predict the output first.
- Keep a `revision/` folder with one file per section (`s01_basics.py`, `s02_numbers.py`, ...). Add your own `assert` lines.
- After each section, do the **Drill** from memory, then compare with the section.
- Run files with `python file.py`. Use the REPL (`python`) for quick probes. `help(obj)` and `dir(obj)` explain anything.
- Break each example on purpose and read the traceback. Errors teach faster than successes.

## 1. Basics

**Theory.** Python is dynamically typed: a name is bound to an object, and the type belongs to the object, not the name. Indentation defines blocks. Everything is an object.

```python
# a comment
x = 10            # int
pi = 3.14         # float
name = 'Amr'      # str
ok = True         # bool
nothing = None    # the single 'no value' object

a, b = 1, 2       # multiple assignment
a, b = b, a       # swap without a temp variable
x = y = 0         # chained assignment

print(type(x), id(x))   # every object has a type and an identity
print('a', 'b', sep='-', end='!\n')   # a-b!
```

Naming (PEP 8): `snake_case` for variables and functions, `PascalCase` for classes, `UPPER_CASE` for constants, a leading `_` for internal use.

Input and f-strings:

```python
n = int(input('n: '))          # input() always returns str
print(f'{n} squared is {n**2}')
s = 'x'
print(f'{3.14159:.2f} | {42:05d} | {1234567:,} | {0.256:.1%}')   # 3.14 | 00042 | 1,234,567 | 25.6%
print(f'{s:>5}|{s:<5}|{s:^5}')  # right, left, center alignment
print(f'{n = }')               # debug form, prints: n = 5
```

`==` compares values. `is` compares identity; use it only for `None`, `True`, `False`.

Truthiness: `False, None, 0, 0.0, '', [], (), {}, set()` are falsy. Everything else is truthy.

**Drill:** swap two variables; print a float with 3 decimals right-aligned in width 10; read two ints on one line with `a, b = map(int, input().split())`.

## 2. Numbers and operators

```python
7 / 2        # 3.5   true division always gives float
7 // 2       # 3     floor division rounds toward negative infinity
-7 // 2      # -4
7 % 3        # 1     result takes the sign of the divisor: -7 % 3 == 2
2 ** 10      # 1024  power; 2 ** -1 == 0.5
divmod(17, 5)          # (3, 2)
round(2.5)             # 2  banker's rounding: ties go to the even number
round(3.14159, 2)      # 3.14
10 ** 100              # ints have arbitrary precision, no overflow
0.1 + 0.2 == 0.3       # False: binary floating point
```

Compare floats with `math.isclose(0.1 + 0.2, 0.3)`.

- Chained comparison: `0 < x < 10`.
- `and`, `or`, `not` short-circuit and return an operand: `name = user_input or 'default'`.
- Bitwise: `& | ^ ~ << >>`. Odd test `x & 1`, power of two `1 << k`, halve `x >> 1`, popcount `x.bit_count()`.
- Augmented assignment: `+= -= *= //= %= **= <<= >>= &= |= ^=`.
- `int(-3.7)` is -3 (truncates), `math.floor(-3.7)` is -4.

```python
import math
math.sqrt(16); math.isqrt(17); math.floor(2.7); math.ceil(2.1)
math.gcd(12, 18); math.lcm(4, 6); math.comb(5, 2); math.factorial(5)
math.inf; float('inf'); int('42'); int('ff', 16); bin(5); hex(255)
```

**Drill:** predict `-7 // 2`, `-7 % 3`, `round(0.5)`, `round(1.5)`, `int(-0.9)` (check: -4, 2, 0, 2, 0). Then compute digit sum of an int without converting to str.

## 3. Strings

Immutable sequences of Unicode characters.

```python
s = 'hello world'
s[0]; s[-1]; s[1:4]; s[::2]; s[::-1]     # index, last, slice, step, reverse
len(s); 'lo' in s
s.find('o')       # 4, or -1 if absent
s.index('o')      # 4, raises ValueError if absent
s.upper(); s.title(); s.strip(); s.lstrip(); s.rstrip()
s.split(); 'a,b'.split(','); ','.join(['a', 'b'])   # join is called on the separator
s.replace('l', 'L', 1); s.startswith('he'); s.endswith('ld')
s.count('l'); '42'.isdigit(); 'ab'.isalpha(); 'a1'.isalnum()
'a-b-c'.partition('-')     # ('a', '-', 'b-c')
'x'.zfill(3); 'x'.center(5, '*'); 'abc'.ljust(5)
r'raw \n string'          # raw: backslashes are kept as typed
'''multi
line'''
ord('a'); chr(97)         # code point conversions
b = 'é'.encode('utf-8'); b.decode('utf-8')   # str <-> bytes
```

- Slicing never raises on out-of-range bounds; indexing does (`IndexError`).
- Build big strings with a list and `''.join(parts)`, not `+=` in a loop.
- `sorted('hello')` returns a list of characters.
- Comparison is lexicographic by code point: `'B' < 'a'` is True.

**Drill:** reverse the words of a sentence; palindrome check ignoring case and punctuation; count vowels; Caesar cipher using `ord`/`chr`.

## 4. Lists and tuples

List: mutable, ordered. Tuple: immutable, ordered (hashable if its items are).

```python
a = [3, 1, 2]
a.append(4)            # [3, 1, 2, 4]
a.extend([5, 6])       # [3, 1, 2, 4, 5, 6]
a.insert(0, 9)         # [9, 3, 1, 2, 4, 5, 6]
a.pop()                # removes and returns 6 (O(1))
a.pop(0)               # removes and returns 9 (O(n))
a.remove(1)            # removes first 1 -> [3, 2, 4, 5]
a.index(2)             # 1
a.count(2)             # 1
a.sort()               # in place -> [2, 3, 4, 5]; returns None
a.sort(reverse=True)   # [5, 4, 3, 2]
a.reverse()            # [2, 3, 4, 5]
sorted(a, key=lambda x: -x)   # NEW list [5, 4, 3, 2]
a[1:3] = [7, 8, 9]     # slice assignment may change length -> [2, 7, 8, 9, 5]
del a[0]               # [7, 8, 9, 5]
len(a), min(a), max(a), sum(a), 8 in a
a.clear()
```

Slicing is `a[start:stop:step]` with stop exclusive. Negative indices count from the end. `a[:]` is a shallow copy.

Copying:

```python
b = a            # alias: same object
b = a[:]         # shallow copy (also a.copy(), list(a))
import copy
c = copy.deepcopy(a)   # nested objects copied too
```

Classic bug:

```python
grid = [[0] * 3] * 3                   # WRONG: three references to one row
grid = [[0] * 3 for _ in range(3)]     # correct
```

Unpacking and tuples:

```python
first, *rest = [1, 2, 3, 4]    # first=1, rest=[2, 3, 4]
*init, last = [1, 2, 3]        # init=[1, 2], last=3
a, (b, c) = 1, (2, 3)
t = (1,)                       # a one-element tuple needs the comma
```

- Sort by several keys: `sorted(people, key=lambda p: (p.age, p.name))`. Sorting is stable.
- Transpose a matrix: `list(zip(*g))`. Dimensions: `rows, cols = len(g), len(g[0])`.

**Drill:** rotate a list by k; flatten a nested list; transpose a matrix; sort words by length then alphabetically; explain why `[[0]*3]*3` breaks.

## 5. Dicts, sets, and collections

```python
d = {'a': 1, 'b': 2}
d['c'] = 3
d.get('z'); d.get('z', 0)       # None, 0 (no KeyError)
d.pop('a'); d.pop('zz', None)   # default avoids KeyError
d.setdefault('k', []).append(1)
for k, v in d.items(): print(k, v)
list(d.keys()); list(d.values()); 'b' in d    # key check is O(1)
merged = d | {'x': 1}           # merge (3.9+); d |= other updates in place
{k: v for k, v in d.items() if k != 'b'}
dict(zip(['x', 'y'], [1, 2])); dict.fromkeys('abc', 0)
scores = {'ann': 3, 'bob': 5}
sorted(scores.items(), key=lambda kv: kv[1], reverse=True)   # [('bob', 5), ('ann', 3)]
```

Dicts keep insertion order (3.7+). Keys must be hashable: `str`, `int`, `tuple` of hashables. Not `list`, `dict`, or `set`.

Sets:

```python
s = {1, 2, 3}; e = set()          # {} is an empty DICT, not a set
s.add(4); s.discard(9); s.remove(1)   # remove raises KeyError if missing
a, b = {1, 2, 3}, {2, 3, 4}
a | b; a & b; a - b; a ^ b; a <= b    # union, intersection, difference, symmetric difference, subset
frozenset({1, 2})                     # hashable set
```

`collections`:

```python
from collections import Counter, defaultdict, deque
Counter('banana')                  # Counter({'a': 3, 'n': 2, 'b': 1})
Counter(['x', 'y', 'x']).most_common(1)   # [('x', 2)]
dd = defaultdict(list); dd['x'].append(1) # missing key auto-creates list()
dq = deque([1, 2, 3]); dq.appendleft(0); dq.popleft(); dq.rotate(1)   # O(1) at both ends
deque(maxlen=3)                    # fixed-size sliding window
```

**Drill:** word-frequency counter; group anagrams with `tuple(sorted(word))` as key; two-sum with a dict; dedupe a list preserving order with `list(dict.fromkeys(a))`.

## 6. Control flow

```python
if x > 0:
    ...
elif x == 0:
    ...
else:
    ...
label = 'pos' if x > 0 else 'non-pos'      # ternary

for i in range(5): ...              # 0..4
for i in range(2, 10, 3): ...       # 2, 5, 8
for i in range(10, 0, -1): ...      # countdown
for i, v in enumerate(items, start=1): ...
for a, b in zip(xs, ys): ...        # stops at the shortest; zip(xs, ys, strict=True) raises on mismatch
for x in reversed(xs): ...
while cond: ...

for x in xs:
    if x == target:
        break
else:
    print('not found')     # runs only if the loop did NOT break
```

`continue` skips to the next iteration. `pass` is a no-op placeholder.

Walrus: `if (n := len(a)) > 10: print(n)`.

Pattern matching (3.10+):

```python
def handle(cmd):
    match cmd.split():
        case ['go', direction]:
            return f'going {direction}'
        case ['quit' | 'exit']:
            return 'bye'
        case [x, y] if x == y:
            return 'twins'
        case _:
            return 'unknown'

def area(shape):
    match shape:
        case {'type': 'circle', 'r': r}:
            return 3.14159 * r * r
        case {'type': 'rect', 'w': w, 'h': h}:
            return w * h
        case int() | float() as n:
            return n
        case _:
            raise ValueError(shape)
```

- Never modify a list while iterating over it. Iterate over a copy (`for x in list(xs)`) or build a new list.
- `range` is lazy, and `in`, `len`, and indexing work on it in O(1).

**Drill:** FizzBuzz using `match`; find the first duplicate with `for/else`; print a multiplication table with aligned columns.

## 7. Functions

```python
def add(a, b=0, *args, scale=1, **kwargs):
    '''One-line summary (docstring).'''
    return (a + b + sum(args)) * scale

add(1, 2, 3, 4, scale=2, extra='x')   # a=1, b=2, args=(3, 4), kwargs={'extra': 'x'} -> 20
```

Parameter kinds: `def f(pos_only, /, normal, *, kw_only)`. Call-site unpacking: `f(*lst, **dct)`.

Return several values by returning a tuple: `return lo, hi`, then `lo, hi = f()`. No `return` means `None`.

**Mutable default pitfall.** Defaults are evaluated once, at definition time:

```python
def bad(x, acc=[]):
    acc.append(x)
    return acc            # bad(1) -> [1], bad(2) -> [1, 2]

def good(x, acc=None):
    if acc is None:
        acc = []
    acc.append(x)
    return acc
```

Scope follows LEGB: Local, Enclosing, Global, Built-in.

```python
count = 0
def inc():
    global count
    count += 1

def make_counter():
    n = 0
    def inner():
        nonlocal n
        n += 1
        return n
    return inner
```

Arguments are passed by object reference. Mutating a list inside a function affects the caller. Rebinding the parameter name does not.

Lambdas and first-class functions:

```python
sq = lambda x: x * x                      # a single expression only
list(map(sq, [1, 2, 3]))                  # [1, 4, 9]
list(filter(lambda x: x % 2, range(10)))  # [1, 3, 5, 7, 9]
ops = {'add': lambda a, b: a + b, 'mul': lambda a, b: a * b}
ops['mul'](3, 4)                          # 12
```

Type hints are not enforced at runtime, but tools and readers use them:

```python
def mean(xs: list[float]) -> float: ...
def find(xs: list[int], t: int) -> int | None: ...
from typing import Callable, Iterable, Optional, TypeVar
```

Recursion: the default limit is about 1000 frames and there is no tail-call optimization. Prefer iteration or an explicit stack for deep recursion, and memoize with `functools.cache`.

**Drill:** recursive `flatten(nested)`; a `make_counter()` closure; a function using `*args` to return the product; explain what `bad(1); bad(2)` returns.

## 8. Comprehensions, iterators, generators

```python
[x * x for x in range(10) if x % 2 == 0]       # [0, 4, 16, 36, 64]
{x: x * x for x in range(5)}
{c for c in 'hello'}
[(i, j) for i in range(3) for j in range(3) if i != j]   # outer loop first
[[0] * 4 for _ in range(3)]
gen = (x * x for x in range(10**9))            # generator expression: lazy, O(1) memory
sum(x * x for x in range(10))                  # no extra brackets in a single-argument call
```

Iterator protocol: an *iterable* has `__iter__`. An *iterator* has `__next__` and raises `StopIteration` when finished. A `for` loop calls `iter()` once, then `next()` repeatedly.

```python
it = iter([1, 2])
next(it); next(it); next(it, 'done')     # 1, 2, 'done'
```

Generators:

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

def lines(path):
    with open(path) as f:
        for line in f:
            yield line.rstrip()

def chain_two(a, b):
    yield from a        # delegate to another iterable
    yield from b
```

Generators are single-pass: `list(gen)` exhausts them.

Handy built-ins: `any`, `all`, `sum`, `min(xs, key=...)`, `max(xs, default=0)`, `sorted`, `reversed`, `enumerate`, `zip`, `map`, `filter`, `next`, `iter`.

`itertools`: `chain`, `product`, `permutations`, `combinations`, `accumulate`, `groupby` (needs sorted input), `islice`, `count`, `cycle`, `pairwise` (3.10+), `batched` (3.12+).

**Drill:** a lazy Fibonacci generator consumed with `islice`; running sums with `accumulate`; all pairs with `combinations`; rewrite three of your for-loops as comprehensions and one comprehension back into a loop.

## 9. Exceptions

```python
try:
    x = int(s)
    y = 10 / x
except ValueError as e:
    print('bad int:', e)
except (ZeroDivisionError, TypeError):
    print('math problem')
except Exception:           # broad net; never use a bare `except:`
    raise                   # re-raise after handling or logging
else:
    print('no exception')   # runs only if the try block succeeded
finally:
    print('always runs')    # cleanup, even after return or raise

raise ValueError('message')
# inside an except block: chain the cause explicitly
#     raise RuntimeError('wrapped') from e
assert x > 0, 'x must be positive'   # internal invariants only; removed with python -O

class AppError(Exception): ...

class NotFound(AppError):
    def __init__(self, key):
        super().__init__(f'{key!r} not found')
        self.key = key
```

- Python prefers **EAFP** (easier to ask forgiveness than permission): try the operation and handle the failure, instead of pre-checking everything (LBYL).
- Common built-ins: `ValueError`, `TypeError`, `KeyError`, `IndexError`, `AttributeError`, `ZeroDivisionError`, `StopIteration`, `FileNotFoundError`, `ImportError`, `RecursionError`.
- `ExceptionGroup` and `except*` (3.11+) exist for several simultaneous errors. Know they exist; learn them when you need async code.
- Read a traceback from the bottom: the last line is the error, the lines above show the call chain that led to it.

Debugging toolkit:

```python
print(f'{var = }')                 # quick inspection
breakpoint()                       # opens pdb: n (next), s (step), c (continue), p var, l (list), q (quit)
# python -m pdb script.py          # start under the debugger
import logging
logging.basicConfig(level=logging.INFO)
logging.info('loaded %d rows', 10)   # prefer logging over print in real code
```

**Drill:** `read_int(prompt)` that loops until the input is valid; a custom exception carrying a field; show that `finally` runs even when the `try` block returns.

## 10. Files, context managers, JSON

```python
with open('data.txt', 'w', encoding='utf-8') as f:
    f.write('line1\n')
    f.writelines(['a\n', 'b\n'])

with open('data.txt', encoding='utf-8') as f:    # mode 'r' is the default
    text = f.read()                              # whole file as one str

with open('data.txt', encoding='utf-8') as f:
    for line in f:                               # lazy, one line at a time
        print(line.rstrip())
```

Modes: `r` read, `w` write (truncates), `a` append, `x` create-only, plus `b` for binary and `+` for read/write. `with` guarantees the file closes even if an exception occurs.

```python
from pathlib import Path
p = Path('data') / 'file.txt'
p.parent.mkdir(parents=True, exist_ok=True)
p.write_text('hi', encoding='utf-8'); p.read_text(encoding='utf-8')
p.exists(); p.suffix; p.stem; p.name
list(Path('.').glob('*.py'))

import json
text = json.dumps({'a': 1}, indent=2)
data = json.loads(text)
with open('x.json', 'w') as f:
    json.dump(data, f, indent=2)
with open('x.json') as f:
    data = json.load(f)

import csv
with open('x.csv', newline='') as f:
    for row in csv.DictReader(f):
        print(row['col'])
```

Custom context manager:

```python
from contextlib import contextmanager
import time

@contextmanager
def timer(label):
    t = time.perf_counter()
    try:
        yield
    finally:
        print(label, time.perf_counter() - t)

with timer('work'):
    sum(range(10**6))
```

The class-based form implements `__enter__(self)` and `__exit__(self, exc_type, exc, tb)`. Returning `True` from `__exit__` swallows the exception.

**Drill:** count words in a text file; save and reload a dict as JSON; write `timer` yourself from memory.

## 11. Modules, packages, environments

```python
import math
from math import sqrt, pi
from math import sqrt as sq
import numpy as np            # alias convention
# from collections import *   # avoid wildcard imports
```

```python
# utils.py
def helper(): ...

def main(): ...

if __name__ == '__main__':    # runs only when executed directly, not when imported
    main()
```

A package is a folder of modules (usually with an `__init__.py`). Inside a package use relative imports: `from . import x`, `from .mod import y`.

```
project/
  pyproject.toml
  src/mypkg/__init__.py
  src/mypkg/core.py
  tests/test_core.py
```

Environments: `python -m venv .venv`, activate it, `pip install pkg`, `pip freeze > requirements.txt`. With `uv`: `uv init`, `uv add pkg`, `uv run file.py`, `uv sync`. Never install project dependencies into the global interpreter.

Testing with `pytest`: put functions named `test_*` in files named `test_*.py` and use plain `assert`.

Standard library worth knowing: `os`, `sys`, `pathlib`, `json`, `csv`, `re`, `datetime`, `time`, `random`, `math`, `statistics`, `itertools`, `functools`, `collections`, `heapq`, `bisect`, `dataclasses`, `enum`, `typing`, `logging`, `argparse`, `subprocess`, `threading`, `asyncio`, `copy`, `textwrap`.

**Drill:** split a script into a module plus a `main()` guarded by `__name__`; create a venv or `uv` project and run a `pytest` test.

## 12. Decorators and functools

A decorator is a function that takes a function and returns a function.

```python
import functools, time

def timed(func):
    @functools.wraps(func)               # keeps the original name and docstring
    def wrapper(*args, **kwargs):
        t = time.perf_counter()
        result = func(*args, **kwargs)
        print(f'{func.__name__}: {time.perf_counter() - t:.4f}s')
        return result
    return wrapper

@timed                                   # same as: slow = timed(slow)
def slow(n):
    return sum(range(n))

def retry(times):                        # decorator with arguments = three nested layers
    def deco(func):
        @functools.wraps(func)
        def wrapper(*a, **kw):
            for attempt in range(times):
                try:
                    return func(*a, **kw)
                except Exception:
                    if attempt == times - 1:
                        raise
        return wrapper
    return deco

@retry(3)
def flaky(): ...
```

Stacked decorators apply from the bottom up.

`functools` essentials:

```python
@functools.cache                # memoization; arguments must be hashable
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

from functools import partial, reduce
double = partial(pow, 2)        # pow(2, x)
reduce(lambda a, b: a * b, [1, 2, 3, 4])   # 24
```

Also know: `@lru_cache(maxsize=N)`, `@cached_property`, `@total_ordering`, `@singledispatch`.

**Drill:** write `@timed` and `@retry(3)` from memory; memoize a grid-paths function with `@cache`; explain why `wraps` matters.

## 13. Classes: a quick bridge

The full treatment is in the companion guide, *OOP and SOLID: Full Revision Guide*. This is just enough syntax.

```python
class Point:
    count = 0                              # class attribute, shared by all instances

    def __init__(self, x, y):
        self.x, self.y = x, y              # instance attributes
        Point.count += 1

    def __repr__(self):
        return f'Point({self.x}, {self.y})'

    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)

    def __hash__(self):
        return hash((self.x, self.y))

    def dist(self, other):
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5
```

Dataclasses and enums remove boilerplate:

```python
from dataclasses import dataclass, field
from enum import Enum, auto

@dataclass(frozen=True)
class P:
    x: float
    y: float = 0.0

@dataclass
class Item:
    name: str
    tags: list[str] = field(default_factory=list)   # mutable defaults need default_factory

class Color(Enum):
    RED = auto()
    GREEN = auto()

Color.RED.name; Color['RED']; Color(1)     # 'RED', Color.RED, Color.RED
```

## 14. Python for problem solving

Fast input and output:

```python
import sys
input = sys.stdin.readline                     # much faster than the built-in input()
n = int(input())
a = list(map(int, input().split()))
data = sys.stdin.read().split()                # or read everything at once
print('\n'.join(map(str, results)))             # one print call instead of thousands
```

Heaps (min-heap only; negate values for a max-heap):

```python
import heapq
h = []
heapq.heappush(h, (3, 'c')); heapq.heappush(h, (1, 'a'))
d, u = heapq.heappop(h)                        # (1, 'a')
heapq.heapify(lst); heapq.nlargest(3, lst); heapq.nsmallest(3, lst)
```

Binary search:

```python
from bisect import bisect_left, bisect_right, insort
a = [1, 2, 2, 2, 5]
i = bisect_left(a, 2)       # 1: first index with a[i] >= 2
j = bisect_right(a, 2)      # 4: first index with a[j] > 2
j - i                       # 3 occurrences
```

BFS and iterative DFS:

```python
from collections import deque

def bfs(graph, start):
    seen = {start}
    q = deque([start])
    while q:
        u = q.popleft()
        for v in graph[u]:
            if v not in seen:
                seen.add(v)
                q.append(v)
    return seen

def dfs(graph, start):
    seen, stack = set(), [start]
    while stack:
        u = stack.pop()
        if u in seen:
            continue
        seen.add(u)
        stack.extend(graph[u])
    return seen
```

Other tools: `defaultdict(list)` for adjacency lists, `Counter`, `math.gcd`, `itertools.accumulate` for prefix sums, `float('inf')`, `sorted(range(n), key=a.__getitem__)` for argsort, `functools.cache` for DP, `x.bit_count()`.

Performance notes:

- Put hot code inside a function (`def main():`); local variables are faster than globals.
- Comprehensions beat append loops. Avoid `list.pop(0)` and `insert(0, x)`; use `deque`.
- Avoid repeated string concatenation; join once.
- On Codeforces, submit under PyPy. Deep recursion is risky even with a raised limit; write iterative versions.

| Operation | Typical cost |
| --- | --- |
| list append, `pop()`, index | O(1) |
| `pop(0)`, `insert(0, x)`, `x in list` | O(n) |
| dict / set get, add, `in` | O(1) average |
| slicing `a[i:j]` | O(j - i) |
| `sorted`, `list.sort` | O(n log n) |
| `heappush`, `heappop` | O(log n) |
| `deque` append / popleft | O(1) |

**Drill:** implement BFS shortest path on a grid; Dijkstra with `heapq`; two-sum with a dict; count occurrences of x in a sorted array with `bisect`; solve five problems under a 30-minute timer without AI.

## 15. Pitfalls checklist

- Mutable default arguments (`def f(x, acc=[])`).
- `[[0] * n] * m` shares one row.
- Shallow copy versus `deepcopy`.
- `is` versus `==`.
- Modifying a collection while iterating over it.
- Late-binding closures: `[lambda: i for i in range(3)]` all return 2. Fix with `lambda i=i: i`.
- `list.sort()` returns `None`; `sorted()` returns a new list.
- `round()` uses banker's rounding; floats are not exact.
- `//` and `%` on negative numbers.
- Generators can be consumed only once.
- `{}` is an empty dict, not an empty set.
- A bare `except:` hides bugs, including `KeyboardInterrupt`.
- Shadowing built-ins (`list`, `sum`, `max`, `id`, `input`, `str`).
- `+=` on strings inside long loops.
- A one-element tuple needs a trailing comma.
- Forgetting `self` or `return`.

## 16. Revision checklist

Tick an item only when you can do it blind, without looking at this guide.

- [ ] Swap, unpack, star-unpack, and format output with f-strings
- [ ] Predict `//`, `%`, `round`, and `int` on negatives and ties
- [ ] Slice and reverse strings and lists; explain alias versus copy
- [ ] Use `Counter`, `defaultdict`, `deque`, and set operations
- [ ] Write `for/else`, a ternary, a `match` statement, and a comprehension
- [ ] Write a function with defaults, `*args`, `**kwargs`, and keyword-only arguments
- [ ] Explain LEGB scope, closures, and `nonlocal`
- [ ] Write a generator, and explain iterator versus iterable
- [ ] Use `try/except/else/finally`, define a custom exception, and read a traceback
- [ ] Read and write text, JSON, and CSV files with `with`
- [ ] Structure a module with a `__name__` guard and run a `pytest` test
- [ ] Write `@timed` and `@retry(3)` decorators from memory
- [ ] Implement BFS, DFS, Dijkstra, and binary search with the standard library
- [ ] Name the complexity of any list, dict, set, deque, or heap operation
