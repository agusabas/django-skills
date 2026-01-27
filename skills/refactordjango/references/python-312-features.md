# Python 3.12+ Best Practices

Apply these modern Python features and improvements when refactoring.

## Type Parameter Syntax (PEP 695)

Use the new type parameter syntax for generics instead of `TypeVar`:

```python
# OLD (Python 3.11 and earlier)
from typing import TypeVar, Generic

T = TypeVar('T')

class Stack(Generic[T]):
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...

# NEW (Python 3.12+)
class Stack[T]:
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...

# Type aliases with constraints
type HashableSequence[T: Hashable] = Sequence[T]
type IntOrStrSequence[T: (int, str)] = Sequence[T]

# Generic functions
def first[T](items: list[T]) -> T:
    return items[0]
```

## @override Decorator

Use `@override` to mark methods that override parent class methods. This helps catch refactoring bugs when parent methods are renamed or removed:

```python
from typing import override

class BaseProcessor:
    def process(self, data: str) -> str:
        return data

class CustomProcessor(BaseProcessor):
    @override
    def process(self, data: str) -> str:
        # Type checker will warn if BaseProcessor.process is renamed/removed
        return data.upper()
```

## Better Error Messages

Python 3.12 provides improved error messages with suggestions:
- NameError now suggests standard library imports: `Did you forget to import 'sys'?`
- Instance attribute suggestions: `Did you mean: 'self.blech'?`
- Import syntax corrections: `Did you mean to use 'from ... import ...' instead?`
- Misspelled import suggestions: `Did you mean: 'Path'?`

## Other Python 3.12+ Features

- **Inlined Comprehensions (PEP 709)**: List, dict, and set comprehensions are now inlined for up to 2x performance improvement
- **Improved f-strings**: Many previous limitations removed; nested quotes and backslashes now work
- **Per-interpreter GIL**: Foundation for better concurrency in future versions

## Python-Specific Best Practices

### Type Hints

Add comprehensive type annotations (PEP 484, 585):

```python
def process_order(order_id: int, items: list[Item]) -> OrderResult:
    ...
```

### Dataclasses & Named Tuples

Use `@dataclass` for data containers:

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class OrderItem:
    product_id: int
    quantity: int
    price: Decimal
```

### Enums

Replace magic strings/numbers with `Enum`, `StrEnum`, or `IntEnum`:

```python
from enum import StrEnum

class OrderStatus(StrEnum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
```

### List Comprehensions & Generators

Replace verbose loops when readable:

```python
# Instead of verbose loop
result = [item.name for item in items if item.is_active]
```

### Context Managers

Use `with` statements for resource management:

```python
with open(path) as f, transaction.atomic():
    process_data(f.read())
```

### Walrus Operator

Use `:=` to reduce redundant computations:

```python
if (match := pattern.search(text)) is not None:
    process_match(match)
```

### Match Statements (Python 3.10+)

Use structural pattern matching:

```python
match command:
    case {"action": "create", "data": data}:
        create_item(data)
    case {"action": "delete", "id": item_id}:
        delete_item(item_id)
    case _:
        raise ValueError("Unknown command")
```

### Additional Best Practices

- **f-strings**: Use f-strings for string formatting
- **Pathlib**: Use `pathlib.Path` instead of `os.path`
- **Exception Handling**: Be specific about caught exceptions
- **Protocols**: Use `typing.Protocol` for structural subtyping
- **`__slots__`**: Use for classes with many instances to reduce memory
- **`functools`**: Use `lru_cache`, `cached_property`, `partial`
- **Itertools**: Leverage `chain`, `groupby`, `islice`, `combinations`
- **PEP 8**: Follow style guidelines; use `ruff`, `black`, `isort`
