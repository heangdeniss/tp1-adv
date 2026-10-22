# Factory Pattern for Data Transformations

## Overview

This module implements the **Factory Pattern** for creating data transformation objects. It provides an abstract base class for transformations and concrete implementations for common data cleaning operations, with a factory to instantiate them by name.

---

## Architecture

```
DataTransform (ABC)
    │
    ├── NormalizeColumns
    ├── RemoveDuplicates
    └── StandardizeText
           ▲
           │
    TransformFactory.create("name")
```

---

## Abstract Base Class

### `DataTransform`

**`apply(df: pd.DataFrame) -> pd.DataFrame`** (abstract)

Defines the contract all transformations must follow. Returns a **transformed copy** of the input DataFrame—never mutates the original.

---

## Concrete Transformations

### `NormalizeColumns`

**`apply(df: pd.DataFrame) -> pd.DataFrame`**

Applies **min-max normalization** to numeric columns only:
```
(value - min) / (max - min)  →  scales to [0, 1]
```

| Behavior | Details |
|----------|---------|
| **Scope** | `select_dtypes(include='number')` — numeric columns only |
| **NaN handling** | Preserved (not normalized) |
| **Constant columns** | Set to `0.0` (avoids division by zero) |
| **Non-numeric** | Left unchanged |
| **Returns** | New DataFrame (copy) |

---

### `RemoveDuplicates`

**`apply(df: pd.DataFrame) -> pd.DataFrame`**

Removes duplicate rows using pandas' `drop_duplicates()` and resets the index:
```python
df.drop_duplicates().reset_index(drop=True)
```

- Returns new DataFrame (pandas copies by default)
- Clean sequential index starting at 0

---

### `StandardizeText`

**`apply(df: pd.DataFrame) -> pd.DataFrame`**

Standardizes **object/string columns**:
- `strip()` leading/trailing whitespace
- `lower()` converts to lowercase

| Behavior | Details |
|----------|---------|
| **Scope** | `select_dtypes(include=['object', 'string'])` |
| **Non-strings** | Preserved as-is (NaN, numbers, etc.) |
| **Non-text columns** | Unchanged |
| **Returns** | New DataFrame (copy) |

---

## Factory

### `TransformFactory`

Registry-based factory that maps string names to transformation classes.

#### `create(name: str) -> DataTransform`

Instantiates a transformation by name (case-insensitive, whitespace trimmed).

| Input | Returns |
|-------|---------|
| `"normalize"` | `NormalizeColumns` |
| `"remove_duplicates"` / `"duplicates"` | `RemoveDuplicates` |
| `"standardize_text"` / `"text"` | `StandardizeText` |

**Raises** `ValueError` with a helpful message listing all supported names for invalid input.

#### `supported_transformations() -> list[str]`

Returns sorted list of all supported transformation name strings.

---

## Usage

```python
from factory import TransformFactory
import pandas as pd

df = pd.read_csv("sample_data.csv")

# Explicit
df = TransformFactory.create("normalize").apply(df)
df = TransformFactory.create("remove_duplicates").apply(df)
df = TransformFactory.create("standardize_text").apply(df)

# Dynamic pipeline
for name in ["normalize", "duplicates", "text"]:
    df = TransformFactory.create(name).apply(df)
```

---

## Extending

Add a new transformation in two steps:

```python
# 1. Create subclass
class FillMissing(DataTransform):
    def apply(self, df: pd.DataFrame) -> pd.DataFrame:
        return df.fillna(df.mean(numeric_only=True))

# 2. Register (aliases allowed)
TransformFactory._TRANSFORMS['fill_missing'] = FillMissing
TransformFactory._TRANSFORMS['fill'] = FillMissing
```

Now `TransformFactory.create("fill_missing")` works immediately.