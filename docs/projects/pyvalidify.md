# Runtime Type Validation for Python

## Introduction

### Problem

Python's type hints are powerful, but they stop at static analysis. At runtime, they are not enforced. This creates a gap:

- Developers rely on type hints for correctness
- But invalid data can still silently pass through at runtime
- Built-in tools like `isinstance()` don't scale well for complex, nested types

For example:

```python
list[dict[str, tuple[str, str, bool]]]
```

Validating structures like this quickly becomes repetitive and error-prone.

### Goal

Explore whether Python's type hint system can be used to provide **lightweight runtime validation**, without introducing heavy schemas or external frameworks.

### Approach

Validify is built around a simple idea:

> Convert type hints into a structured internal representation, then validate runtime data against it.

The library introduces a `Descriptor` abstraction that:

- Represents primitive and generic types in the same format
- Makes validation of nested structures possible
- Supports unions and combinations of types

On top of that core, I added:

- A manual validation function (`isvalid`)
- Decorators for functions and classes so checks can run automatically

## Implementation

### How It Works

1. **Type Parsing**

  - Type hints are transformed into `Descriptor` objects
  - Handles generics (`list[str]`), unions (`str | int`), and nesting

2. **Validation Engine**

  - Recursively compares runtime values against descriptors
  - Applies rules based on structure and type relationships

3. **Integration Layer**

  - Function decorators validate arguments at call time
  - Class decorators validate attributes and method inputs

### Example Usage

#### Manual Validation

```python
from validify import isvalid

data = [
    {"name": "John", "email": "john@example.com"},
    {"name": "Dan", "email": None}
]

isvalid(data, list[dict[str, str | None]])  # True
```

#### Function Validation

```python
from validify import func

@func.validate
def process(items: list[int]) -> None:
    ...

process([1, 2, 3])   # OK
process(["a", "b"])  # TypeError
```

#### Class Validation

```python
from validify import cls

@cls.validify
class Basket:
    items: list[tuple[str, int, float]]

Basket([("apple", 2, 1.5)])  # OK
Basket([("apple", "2", 1.5)])  # TypeError
```

### Key Design Decisions

1. **Descriptor abstraction**

  - I needed one representation for both simple and nested types
  - This let me separate *type parsing* from *validation logic*

2. **Decorators over metaprogramming**

  - I chose decorators for readability and explicit usage
  - I avoided deeper magic (metaclasses, proxies) to keep behavior predictable

3. **Layered structure**

  - Three layers: core type logic, validation rules, integration (decorators)
  - This made the code easier to reason about and extend

## Evaluation

### Trade-offs

- Nested unions are not fully propagated across all combinations
- Recursive validation introduces performance cost
- Does not fully support advanced constructs like `TypedDict` or `Annotated`

These were intentional trade-offs so I could keep the implementation focused and understandable.

### Known Limitation

For a type like:

```python
list[tuple[int | str]]
```

The system does not fully expand equivalent representations such as:

```python
list[tuple[int] | tuple[str]]
```

This is where the hard part became clear: checking one nested union shape is manageable, but handling equivalent nested forms consistently gets complicated quickly.

### What I Would Improve

- Extend support for modern typing features (`TypedDict`, `Annotated`)
- Improve how unions propagate through nested structures
- Add performance benchmarks and targeted optimization passes
- Make error messages more actionable for developers
- Explore tighter integration with static type checkers

## Reflection

### Takeaways

This project was less about shipping a production-ready library and more about exploring:

- How far Python's type system can be pushed at runtime
- The complexity of mapping static types to dynamic validation
- Trade-offs between correctness, usability, and performance

It also reinforced a few engineering lessons for me:

- Clear abstractions when dealing with complex structures
- Making trade-offs explicit in system design
- Designing tools around developer experience, not just raw capability

### Retrospective

At the time of building Validify, I saw it as a strong general-purpose solution for runtime type validation. Since then, my perspective has shifted.

Today, I would approach this differently, likely leaning more on established patterns (for example schema-based validation, or stricter boundaries between parsing and validation) depending on the use case.

My coding practices have also evolved. If I revisited this project, I would start by:

- Simplifying parts of the architecture
- Improving consistency and naming
- Putting more emphasis on testability and developer experience

Even so, this project still reflects an important phase in my growth, especially in how I think about type systems, abstractions, and trade-offs in Python.

## Links

[Repo](https://github.com/JPatryk13/pyvalidify)<br>
[Package](https://pypi.org/project/pyvalidify/1.0.1/)