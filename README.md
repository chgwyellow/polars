<div align="center">

# Polars Mastery Journey

![Polars](https://img.shields.io/badge/Polars-1.0+-CD792C?logo=polars&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.8+-3776AB?logo=python&logoColor=white)
![Tool](https://img.shields.io/badge/Tool-Jupyter-F37626?logo=jupyter&logoColor=white)
![Status](https://img.shields.io/badge/Status-In%20Progress-blue)
![Updated](https://img.shields.io/badge/Updated-Jan%202026-green)

</div>

---

## 📚 Table of Contents

- [What is Polars?](#what-is-polars)
- [Features](#features)
  - [Expression API](#expression-api)
  - [Selector API](#selector-api)
  - [Lazy Mode & Streaming](#lazy-mode--streaming)

---

## What is Polars?

Polars is a **blazingly fast** DataFrame library designed for high-performance data manipulation and analysis.

### Core Architecture

- **Written in Rust** - Built entirely in Rust for memory safety, performance, and parallel execution
- **Apache Arrow Columnar Format** - Uses Arrow's in-memory columnar data structure for:
  - Cache-efficient operations
  - Zero-copy data sharing with other Arrow-compatible tools (PyArrow, Spark, etc.)
  - Vectorized operations using SIMD (Single Instruction, Multiple Data)

### Interoperability with PyArrow & Pandas

Polars uses **Apache Arrow** as its internal memory format, enabling seamless data exchange:

```text
Polars ←→ Arrow Memory ←→ PyArrow ←→ Pandas (PyArrow-backed)
         (Zero-copy)              (Zero-copy)
```

**Key Concepts:**

- **PyArrow** - Python implementation of Apache Arrow, acts as a bridge between Polars and Pandas
- **Zero-copy conversion** - Data is shared in memory (not duplicated) when converting between compatible formats
- **PyArrow-backed Pandas** - Pandas DataFrames using Arrow arrays instead of NumPy arrays

**Conversion Comparison:**

| Method | Memory | Speed | Data Types |
| ------ | ------ | ----- | ---------- |
| `to_pandas()` | ❌ Copies data | Slower | NumPy types (`int64`, `object`) |
| `to_pandas(use_pyarrow_extension_array=True)` | ✅ Zero-copy | Faster | Arrow types (`int64[pyarrow]`, `large_string[pyarrow]`) |

**Example:**

```python
import polars as pl

df_polars = pl.read_csv("data.csv")

# Traditional conversion (copies data)
df_pandas = df_polars.to_pandas()

# Zero-copy conversion (shares memory)
df_pandas_arrow = df_polars.to_pandas(use_pyarrow_extension_array=True)

# Convert back to Polars
df_polars_again = pl.from_pandas(df_pandas_arrow)
```

**Use Case:** Quickly switch between Polars and Pandas when you need Pandas-specific functions, without memory overhead.

### Key Features

- **Multi-language API** - Python, Rust, Node.js, and R bindings
- **Lightning Fast** - Often 5-10x faster than Pandas due to Rust core and columnar memory
- **Lazy Evaluation** - Query optimization similar to SQL databases
- **Out-of-Core Processing** - Handle datasets larger than RAM with streaming engine
- **Rich File Format Support** - CSV, Parquet, JSON, Excel, Arrow IPC, and more
- **Expressive API** - Clean, declarative syntax for complex data transformations

### Why Polars?

| Feature | Polars | Pandas |
| ------- | ------ | ------ |
| **Speed** | ⚡ 5-10x faster | Baseline |
| **Memory** | 🔋 More efficient (columnar) | Row-based |
| **Parallelization** | ✅ Automatic (multi-threaded) | ❌ Single-threaded by default |
| **Query Optimization** | ✅ Built-in (lazy mode) | ❌ No optimization |
| **Large Datasets** | ✅ Streaming support | ❌ Limited by RAM |

---

## Features

### Expression API

The most powerful tool in Polars.

Expression API is a **declarative syntax** for data transformation that allows you to describe *what* you want to do with your data, rather than *how* to do it.

This design enables Polars to:

- **Optimize query plans** - Analyze and optimize the entire operation before execution
- **Parallelize automatically** - Distribute operations across multiple CPU cores
- **Lazy evaluation** - Execute only when results are actually needed

Expressions are composable, readable, and incredibly fast. They form the foundation of Polars' performance advantage over traditional dataframe libraries.

#### Example 1: Basic Operations

```python
import polars as pl

df = pl.DataFrame({
    "name": ["Alice", "Bob", "Charlie", "David"],
    "age": [25, 30, 35, 28],
    "salary": [50000, 60000, 70000, 55000]
})

# Using Expression API to transform data
result = df.select([
    pl.col("name"),
    pl.col("age") + 5,                           # Add 5 to age
    (pl.col("salary") * 1.1).alias("new_salary") # 10% raise
])
```

#### Example 2: Method Chaining with Filtering

```python
# Calculate average salary for employees over 25
avg_salary = (
    df.filter(pl.col("age") > 25)
      .select(pl.col("salary").mean())
)

# Conditional expressions
df_with_level = df.select([
    pl.col("name"),
    pl.col("age"),
    pl.when(pl.col("age") > 30)
      .then(pl.lit("Senior"))
      .otherwise(pl.lit("Junior"))
      .alias("level")
])
```

---

### Selector API

A **concise, semantic way to select multiple columns** using the `polars.selectors` module (commonly aliased as `cs`).

Selector API complements Expression API by providing more intuitive syntax for common column selection patterns, especially when working with many columns or complex selection logic.

#### Core Concept

Selectors make column selection more readable and less verbose:

| Task | Expression API | Selector API |
| ---- | -------------- | ------------ |
| All numeric columns | `pl.col([pl.Int64, pl.Float64])` | `cs.numeric()` |
| All string columns | `pl.col(pl.Utf8)` | `cs.string()` |
| Columns starting with "P" | `pl.col("^P.*$")` | `cs.starts_with("P")` |
| Exclude specific columns | `pl.exclude("Age", "Fare")` | `~cs.by_name("Age", "Fare")` |

#### Key Features

**1. Select by Data Type**

```python
import polars.selectors as cs

# All numeric columns (integers + floats)
df.select(cs.numeric())

# All string columns
df.select(cs.string())
```

**2. Select by Name Pattern**

```python
# Columns starting with "P"
df.select(cs.starts_with("P"))

# Columns containing "age"
df.select(cs.contains("age"))

# Columns ending with "ed"
df.select(cs.ends_with("ed"))

# Regex matching (no need for ^ and $)
df.select(cs.matches("Age|Fare"))
```

**3. Select by Position**

```python
# First column
df.select(cs.first())

# Last column
df.select(cs.last())
```

**4. Set Operations (Most Powerful Feature)**

```python
# Intersection (&): Both conditions must be true
df.select(cs.numeric() & cs.contains("A"))  # Numeric AND contains "A"

# Union (|): At least one condition must be true
df.select(cs.string() | cs.contains("P"))   # String OR contains "P"

# Difference (-): Exclude specific columns
df.select(cs.string() - cs.by_name("Ticket"))  # Strings except Ticket

# Complement (~): Invert selection
df.select(~cs.by_name("Age", "Fare"))  # All columns except Age and Fare
```

**5. Chain with Expressions**

Selectors output standard Polars expressions, so you can chain them:

```python
# Select all columns and get max values
df.select(cs.all().max())

# Select numeric columns and calculate mean
df.select(cs.numeric().mean())

# Select columns starting with "P" and add 10
df.select(cs.starts_with("P") + 10)
```

#### Practical Examples

```python
import polars as pl
import polars.selectors as cs

df = pl.read_csv("titanic.csv")

# Select all numeric columns except PassengerId
df.select(cs.numeric() - cs.by_name("PassengerId"))

# Select columns starting with "P" OR all string columns
df.select(cs.starts_with("P") | cs.string())

# Select numeric columns that contain "a" in their name
df.select(cs.numeric() & cs.contains("a"))

# Get mean of all numeric columns
df.select(cs.numeric().mean())
```

#### When to Use Selector API

✅ **Use Selectors when:**

- Selecting multiple columns by type or pattern
- Complex column selection logic with set operations
- Want more readable code for column selection

✅ **Use Expression API when:**

- Performing transformations on specific columns
- Need fine-grained control over expressions
- Working with single columns

**Best Practice:** Combine both for maximum clarity and power!

```python
# Use selectors for column selection, expressions for transformation
df.select([
    cs.by_name("Name", "Age"),           # Selector for selection
    (cs.numeric() * 2).name.suffix("_x2")  # Selector + Expression for transformation
])
```

---

### Lazy Mode & Streaming

**Lazy Mode** delays execution until you call `.collect()`, allowing Polars to optimize the entire query plan before running it - similar to how SQL databases work.

#### Key Benefits

- **Query Optimization** - Polars analyzes and optimizes your entire operation chain
- **Memory Efficiency** - Only loads data that's actually needed
- **Automatic Parallelization** - Distributes work across CPU cores
- **Predicate Pushdown** - Filters data as early as possible (e.g., during file reading)

#### Execution Modes Comparison

| Mode | When it executes | Memory Usage | Use case |
| ------ | ---------------- | ------------ | -------- |
| **Eager** | Immediately | Loads all data | Small datasets, quick exploration |
| **Lazy** | On `.collect()` | Loads result | Large files, complex queries |
| **Streaming** | On `.collect(engine='streaming')` | Batch processing | Datasets larger than RAM |

#### Lazy Mode + Streaming Engine

**Streaming** is an advanced execution mode that processes data in batches, enabling you to work with datasets **larger than your available RAM**. It's 3-7x faster than regular lazy mode for large datasets.

**When to use Streaming:**

- ✅ Dataset size > Available RAM
- ✅ Processing files 10GB+
- ✅ Need maximum memory efficiency
- ✅ Writing results directly to disk

#### Example 1: Lazy Mode (Standard)

```python
# For datasets that fit in memory
lf = pl.scan_csv("large_file.csv")
result = lf.filter(pl.col("age") > 25) 
           .select(["name", "salary"]) 
           .collect()  # Execute optimized plan
```

#### Example 2: Streaming Mode (Out-of-Core)

```python
# For datasets larger than RAM
lf = pl.scan_csv("100GB_file.csv")

# Option 1: Collect with streaming engine
result = lf.filter(pl.col("age") > 25) 
           .group_by("country") 
           .agg(pl.col("revenue").sum()) 
           .collect(engine='streaming')  # Process in batches

# Option 2: Write directly to disk (auto-streaming)
lf.filter(pl.col("age") > 25) 
  .group_by("country") 
  .agg(pl.col("revenue").sum()) 
  .sink_parquet("output.parquet")  # No memory overhead
```

#### Mode Conversion Reference

| Operation | Input | Output | Description |
| --------- | ----- | ------ | ----------- |
| `.lazy()` | DataFrame | LazyFrame | Convert to lazy mode |
| `.collect()` | LazyFrame | DataFrame | Execute and convert to eager mode |

**Common Patterns:**

```python
# Lazy → Eager
lf = pl.scan_csv("data.csv"); df = lf.collect()

# Eager → Lazy  
df = pl.read_csv("data.csv"); lf = df.lazy()

# ✅ Optimized: Preview without loading all data
lf.head(5).collect()

# ❌ Inefficient: Loads all data then converts back
lf.collect().lazy().filter(...)
```

**Pro Tips**:

- Use `pl.scan_csv()` instead of `pl.read_csv()` for automatic lazy evaluation
- Use `.sink_*()` methods to write large results directly to disk without loading into memory
- Do operations in Lazy mode → Only `.collect()` at the end
- Use `.head()` before `.collect()` to preview without loading all data
- Streaming engine was completely redesigned in Polars 1.31+ for better performance
