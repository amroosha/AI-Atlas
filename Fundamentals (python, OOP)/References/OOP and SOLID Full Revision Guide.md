# OOP and SOLID: Full Revision Guide

*Theory first, then code to type by hand. Python 3.12+. Companion to the Python Syntax guide.*

## How to use this guide

- Read the **theory** paragraph of a section, close the guide, and try to write the code from it. Then compare.
- Type every example, run it, then add `assert` lines or `pytest` tests to prove it behaves as claimed.
- The SOLID section has a *violation* and a *fix* for each letter. Type the violation first, feel the pain, then refactor.

## 1. Theory first: what OOP is for

An object bundles **state** (data) with **behavior** (functions that act on that data). The four classic ideas:

- **Encapsulation:** keep data and the rules that protect it together; expose a small public surface.
- **Abstraction:** callers use *what* an object does, not *how*.
- **Inheritance:** a subclass reuses and specializes a base class (an is-a relationship).
- **Polymorphism:** the same call (`shape.area()`) behaves differently depending on the object.

The real goal is managing change: keep things that change together in one place, and things that change independently apart. Everything else (SOLID, patterns) serves that goal.

Python is not Java. Functions are first-class, modules are namespaces, and duck typing means you often do not need a class hierarchy. If there is no state to carry, write a function.

## 2. Classes and objects in Python

```python
class Account:
    interest_rate = 0.02                    # class attribute, shared by all instances

    def __init__(self, owner, balance=0):
        self.owner = owner                  # instance attributes
        self.balance = balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError('amount must be positive')
        self.balance += amount

    @classmethod
    def from_dict(cls, d):                  # alternative constructor; cls also works for subclasses
        return cls(d['owner'], d.get('balance', 0))

    @staticmethod
    def is_valid_amount(x):                 # no self or cls: a namespaced plain function
        return x > 0

    def __repr__(self):                     # for developers; unambiguous
        return f'Account({self.owner!r}, {self.balance})'

    def __str__(self):                      # for users; falls back to __repr__ if missing
        return f'{self.owner}: {self.balance}'

acct = Account('Ann', 10)
acct.deposit(5)
Account.deposit(acct, 5)    # identical to acct.deposit(5): self is just the first argument
```

- Attribute lookup order: the instance, then its class, then parent classes (the MRO).
- Assigning `obj.x = 1` creates an *instance* attribute that shadows a class attribute of the same name.
- Pitfall: a mutable class attribute (`tags = []` in the class body) is shared by every instance. Create mutable state in `__init__`.

## 3. Encapsulation

Conventions: `_name` means internal, do not touch from outside. `__name` is name-mangled to `_Class__name`, which prevents accidental clashes in subclasses (it is not security).

Properties let you start with plain attributes and add validation later without changing any caller:

```python
class Temperature:
    def __init__(self, celsius):
        self.celsius = celsius              # goes through the setter below

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError('below absolute zero')
        self._celsius = value

    @property
    def fahrenheit(self):                   # computed, read-only
        return self._celsius * 9 / 5 + 32
```

Core rule: an object must not be able to reach an invalid state through its public API. Prefer **tell, don't ask**: call `account.withdraw(50)` and let the object enforce the rule, instead of reading `balance` and deciding outside.

`__slots__ = ('x', 'y')` restricts attributes and saves memory for many small objects.

## 4. Inheritance and the MRO

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        raise NotImplementedError

    def describe(self):
        return f'{self.name} says {self.speak()}'

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)              # always initialise the parent
        self.breed = breed

    def speak(self):                        # override
        return 'woof'

d = Dog('Rex', 'lab')
d.describe()                                # 'Rex says woof'
isinstance(d, Animal); issubclass(Dog, Animal)   # True, True
```

Multiple inheritance uses the **MRO** (C3 linearization). `super()` follows the MRO, not just the direct parent:

```python
class A:
    def hi(self): return 'A'
class B(A):
    def hi(self): return 'B>' + super().hi()
class C(A):
    def hi(self): return 'C>' + super().hi()
class D(B, C):
    def hi(self): return 'D>' + super().hi()

D().hi()        # 'D>B>C>A'
D.__mro__       # D, B, C, A, object
```

**Mixins** are small classes that add one behavior and hold no state:

```python
import json

class JsonMixin:
    def to_json(self):
        return json.dumps(self.__dict__)    # a JSON string of the instance fields

class User(JsonMixin):
    def __init__(self, name):
        self.name = name
```

Guidance: inheritance models *is-a*. If you only want code reuse, use composition (section 6). Hierarchies deeper than two or three levels are a smell.

## 5. Polymorphism, duck typing, ABCs, protocols

**Duck typing:** if it has the method, it works. No shared base class needed.

```python
class Duck:
    def speak(self): return 'quack'
class Robot:
    def speak(self): return 'beep'

for thing in (Duck(), Robot()):
    print(thing.speak())
```

**ABC** (nominal typing: enforced at instantiation):

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Circle(Shape):
    def __init__(self, r):
        self.r = r
    def area(self):
        return 3.14159 * self.r ** 2

Shape()         # TypeError: cannot instantiate an abstract class
Circle(2).area()
```

**Protocol** (structural typing: describes what a caller needs; checked by mypy or pyright, not at runtime):

```python
from typing import Protocol

class SupportsArea(Protocol):
    def area(self) -> float: ...

def total_area(shapes: list[SupportsArea]) -> float:
    return sum(s.area() for s in shapes)    # any object with .area() qualifies, no inheritance
```

Choose an ABC when you want enforcement and shared code in a base class. Choose a Protocol when you want to state what you need from the outside.

**Dunder methods** (the data model) let your objects behave like built-ins:

| Method | Enables |
| --- | --- |
| `__repr__`, `__str__` | `repr(x)`, `print(x)` |
| `__eq__`, `__hash__` | `==`, use in sets and dict keys |
| `__lt__`, `__le__`, ... (or `@total_ordering`) | `<`, `sorted` |
| `__add__`, `__mul__`, `__neg__` | `+`, `*`, unary `-` |
| `__len__`, `__getitem__`, `__contains__` | `len(x)`, `x[i]`, `in` |
| `__iter__`, `__next__` | `for` loops |
| `__bool__` | truthiness |
| `__call__` | `x(...)` |
| `__enter__`, `__exit__` | `with x:` |

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, o):
        if not isinstance(o, Vector):
            return NotImplemented           # lets Python try the other operand
        return Vector(self.x + o.x, self.y + o.y)

    def __mul__(self, k):
        return Vector(self.x * k, self.y * k)

    def __eq__(self, o):
        return isinstance(o, Vector) and (self.x, self.y) == (o.x, o.y)

    def __hash__(self):
        return hash((self.x, self.y))       # equal objects MUST hash equal

    def __repr__(self):
        return f'Vector({self.x}, {self.y})'

class Deck:
    def __init__(self, cards):
        self._cards = list(cards)
    def __len__(self):
        return len(self._cards)
    def __getitem__(self, i):
        return self._cards[i]               # also gives iteration, slicing, reversed()
```

Defining `__eq__` without `__hash__` makes a class unhashable.

## 6. Composition, delegation, dependency injection

Prefer **has-a** over **is-a**. A car *has* an engine; it is not a kind of engine.

```python
class Engine:
    def start(self):
        return 'vroom'

class ElectricEngine:
    def start(self):
        return 'hummm'

class Car:
    def __init__(self, engine):             # collaborator is injected
        self.engine = engine
    def start(self):
        return self.engine.start()          # delegation

Car(Engine()).start(); Car(ElectricEngine()).start()
```

Benefits: parts are swappable, easy to test with fakes, and you avoid the fragile base class problem (a change in a parent silently breaking children).

## 7. Dataclasses, NamedTuple, enums

```python
from dataclasses import dataclass, field, replace

@dataclass(frozen=True, slots=True, order=True)
class Money:
    amount: int
    currency: str = 'USD'

    def __post_init__(self):                # validation hook
        if self.amount < 0:
            raise ValueError('amount cannot be negative')

m = Money(5)
replace(m, amount=7)        # a NEW Money(7, 'USD')
m.amount = 1                # FrozenInstanceError

@dataclass
class Basket:
    items: list[str] = field(default_factory=list)   # mutable defaults need default_factory

from typing import NamedTuple
class Pt(NamedTuple):
    x: float
    y: float

from enum import Enum, auto
class Status(Enum):
    NEW = auto()
    PAID = auto()
```

**Value object vs entity.** A value object (Money, Point) is compared by value and is best immutable. An entity (User, Order) has an identity that persists while its data changes. Use enums for closed sets instead of string flags.

## 8. Class design heuristics

- **Cohesion:** a class's methods should use the same data. **Coupling:** depend on as little as possible.
- **Tell, don't ask:** `order.apply_discount()` beats pulling fields out and computing elsewhere.
- **Law of Demeter:** talk to your direct collaborators; avoid `a.b.c.d` chains.
- Prefer small classes, immutability where practical, and functions where there is no state.
- A class that has only static methods is usually just a module.
- Smells: God class, feature envy, primitive obsession, shotgun surgery (one change touches many classes), deep inheritance.

## 9. SOLID

**Theory.** Five heuristics for classes that stay easy to change. They are lenses for refactoring, not laws. Each targets a specific pain: S (changes ripple across unrelated code), O (constant edits to tested code), L (subclasses that surprise callers), I (depending on things you do not use), D (rigid wiring, untestable code).

### S: Single Responsibility

A class should have **one reason to change** (one stakeholder or actor). Violation:

```python
class Report:
    def __init__(self, rows):
        self.rows = rows
    def total(self):                        # business rule
        return sum(r['amount'] for r in self.rows)
    def to_html(self):                      # presentation
        return f'<p>{self.total()}</p>'
    def save(self, path):                   # persistence
        with open(path, 'w') as f:
            f.write(self.to_html())
```

Three reasons to change: the math, the format, the storage. Fix:

```python
class Report:
    def __init__(self, rows):
        self.rows = rows
    def total(self):
        return sum(r['amount'] for r in self.rows)

class HtmlFormatter:
    def format(self, report):
        return f'<p>{report.total()}</p>'

class FileStore:
    def save(self, path, text):
        with open(path, 'w') as f:
            f.write(text)
```

Test: describe the class in one sentence without the word *and*. Pitfall: SRP is about reasons to change, not the number of methods. Do not shatter code into dozens of one-method classes.

### O: Open/Closed

**Open for extension, closed for modification:** add behavior by adding code, not by editing tested code. Violation:

```python
def price(total, kind):
    if kind == 'regular':
        return total
    elif kind == 'vip':
        return total * 0.9
    elif kind == 'student':
        return total * 0.8                  # every new customer type edits this function
```

Fix with polymorphism (the Strategy pattern):

```python
from typing import Protocol

class Discount(Protocol):
    def apply(self, total: float) -> float: ...

class NoDiscount:
    def apply(self, total):
        return total

class Percent:
    def __init__(self, pct):
        self.pct = pct
    def apply(self, total):
        return total * (1 - self.pct)

def price(total: float, discount: Discount) -> float:
    return discount.apply(total)

price(100, Percent(0.1))     # 90.0
```

Pythonic variants: pass a plain function, or keep a dict of callables (`DISCOUNTS = {'vip': lambda t: t * 0.9}`). An `if` chain is fine while the cases are few and stable. Apply OCP after the second or third change (rule of three), not before.

### L: Liskov Substitution

A subclass must be usable anywhere its base class is expected **without breaking correctness**: do not strengthen preconditions, do not weaken postconditions, keep the base class invariants, and do not raise surprising exceptions. Classic violation:

```python
class Rectangle:
    def __init__(self, w, h):
        self.w, self.h = w, h
    def set_width(self, w):
        self.w = w
    def area(self):
        return self.w * self.h

class Square(Rectangle):
    def __init__(self, s):
        super().__init__(s, s)
    def set_width(self, w):
        self.w = self.h = w                 # changes height too: surprise

def widen(r: Rectangle):
    old_h = r.h
    r.set_width(10)
    assert r.area() == 10 * old_h           # True for Rectangle, False for Square(3)
```

Fix: do not make `Square` a `Rectangle`. Use immutable siblings under a common `Shape` ABC with `area()`. Another smell: a subclass that overrides a method just to raise `NotImplementedError` or do nothing (the `Penguin.fly()` problem). Split the capabilities instead (`Bird`, `FlyingBird`). Also watch for overrides that change the signature or return type.

### I: Interface Segregation

Clients should not depend on methods they do not use. Violation: one fat interface.

```python
from abc import ABC, abstractmethod

class Machine(ABC):
    @abstractmethod
    def print_doc(self, doc): ...
    @abstractmethod
    def scan(self): ...
    @abstractmethod
    def fax(self, doc): ...
# BasicPrinter(Machine) is forced to implement scan and fax it cannot do
```

Fix: small, role-based protocols:

```python
from typing import Protocol

class Printer(Protocol):
    def print_doc(self, doc: str) -> None: ...

class Scanner(Protocol):
    def scan(self) -> str: ...

class BasicPrinter:
    def print_doc(self, doc):
        print(doc)

class OfficeMachine:
    def print_doc(self, doc):
        print(doc)
    def scan(self):
        return 'scanned'

def run_report(p: Printer):                 # asks for only what it needs
    p.print_doc('report')
```

In Python, ISP is nearly free: type-hint the smallest capability a function needs (`Iterable[float]` rather than `list[float]`, `Mapping` rather than `dict`).

### D: Dependency Inversion

High-level policy should not depend on low-level details. **Both depend on abstractions.** Violation:

```python
class SmtpMailer:
    def send(self, to, body): ...           # talks to a real mail server

class SignupService:
    def __init__(self):
        self.mailer = SmtpMailer()          # hard-wired: cannot swap it, cannot test without a server
    def register(self, email):
        self.mailer.send(email, 'welcome')
```

Fix: depend on an abstraction and inject the concrete class:

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, to: str, body: str) -> None: ...

class SignupService:
    def __init__(self, notifier: Notifier):
        self.notifier = notifier
    def register(self, email):
        self.notifier.send(email, 'welcome')

class FakeNotifier:
    def __init__(self):
        self.sent = []
    def send(self, to, body):
        self.sent.append((to, body))

def test_register_sends_welcome():
    fake = FakeNotifier()
    SignupService(fake).register('a@b.c')
    assert fake.sent == [('a@b.c', 'welcome')]
```

DIP is the design rule (depend on abstractions). **Dependency injection** is the technique (pass collaborators in). A single composition root, usually `main()`, creates the concrete objects and wires them together. In Python, constructor arguments are enough; you do not need a DI framework.

## 10. SOLID in an AI or RAG codebase

The same ideas applied to the track you are building next:

```python
from typing import Protocol

class Retriever(Protocol):
    def retrieve(self, query: str, k: int) -> list[str]: ...

class LLM(Protocol):
    def complete(self, prompt: str) -> str: ...

class RagPipeline:
    def __init__(self, retriever: Retriever, llm: LLM):
        self.retriever, self.llm = retriever, llm

    def answer(self, question: str) -> str:
        context = '\n'.join(self.retriever.retrieve(question, k=3))
        return self.llm.complete(f'Context:\n{context}\n\nQuestion: {question}')
```

| Principle | Example in an AI codebase |
| --- | --- |
| S | Separate retrieval, generation, and evaluation into their own classes or modules |
| O | A tool registry: adding a tool never edits the agent loop |
| L | Every `Retriever` returns `list[str]` and never raises for an empty result |
| I | A `Tool` protocol with just `name`, `description`, `run(**kwargs)` |
| D | `RagPipeline` depends on `Retriever` and `LLM` protocols; tests pass a fake LLM instead of calling a real API |

With this shape you can swap vector stores or model providers without touching the pipeline, and unit-test the pipeline offline.

## 11. Patterns you will meet (quick map)

| Pattern | Idea | Python form |
| --- | --- | --- |
| Strategy | Swap an algorithm at runtime | A protocol, a function, or a dict of callables |
| Factory | Create objects by key | `make_retriever('faiss')` returning an instance |
| Observer | Notify subscribers of events | A list of callbacks |
| Decorator | Wrap to add behavior | The `@decorator` syntax, or wrapper classes |
| Adapter | Make an incompatible API fit your interface | A thin class wrapping a vendor SDK to satisfy `LLM` |
| Singleton | One shared instance | A module-level object |
| Repository | Hide how data is stored | A class with `get`, `add`, `list` methods |

## 12. Anti-patterns and judgment

- Do not write a class where a function will do. Modules are namespaces.
- Do not use inheritance just to reuse code; use composition.
- Do not add an interface for a single implementation with no test need. Wait for the second use.
- Beware God classes, mutable global state, and Singletons used as globals.
- SOLID is a lens for refactoring. Write simple code first and refactor when change starts to hurt.
- A warning sign of over-engineering is a hierarchy of factories, managers, and handlers for a 200-line problem.

## 13. Practice projects (type, then test with pytest)

1. **Bank accounts:** encapsulation, properties, custom exceptions, `__repr__`.
2. **Shapes:** an ABC with `area()` and `perimeter()`, and a `total_area` function taking a Protocol.
3. **Vector and Matrix:** operator overloading, `__eq__` and `__hash__`, `__iter__`.
4. **Refactor drill:** write a 100-line script with one God class. Split it by SRP, add a new discount type via OCP, add a subclass that obeys LSP, give each client a small Protocol (ISP), and inject storage so you can test with a fake (DIP).
5. **Mini RAG pipeline:** `Retriever` and `LLM` protocols with fakes. Add a second retriever without editing the pipeline.

## 14. Self-test questions

- `@classmethod` versus `@staticmethod`: when do you need each?
- Why does `super()` follow the MRO instead of going to the direct parent?
- ABC versus Protocol: when do you pick each?
- Why does defining `__eq__` alone make a class unhashable, and how do you fix it?
- For each SOLID letter, give the smell in one sentence and the fix in one sentence.
- How is Dependency Inversion different from Dependency Injection?
- Why is `Square(Rectangle)` a Liskov violation even though a square is a rectangle in geometry?
