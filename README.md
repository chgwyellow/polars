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
- [Feature](#functions)
  - [Expression API](#expression-api)
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

Expression API is a **declarative syntax** for data transformation that allows you to describe *what* you want to do with your data, rather than *how* to do it. This design enables Polars to:

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
