<div align="center">

# Polars Mastery Journey

![Polars](https://img.shields.io/badge/Polars-1.37+-CD792C?logo=polars&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.13+-3776AB?logo=python&logoColor=white)
![Tool](https://img.shields.io/badge/Tool-Jupyter-F37626?logo=jupyter&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success)
![Updated](https://img.shields.io/badge/Updated-Jan%202026-green)

</div>

---

## 📚 Table of Contents

- [What is Polars?](#what-is-polars)
- [Features](#features) *(8 collapsible sections)*
- [Project Structure](#project-structure) *(click to expand)*

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

<details>
<summary><b>⚡ Lazy Mode & Streaming</b></summary>

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

</details>

---

<details>
<summary><b>📌 Expression API</b></summary>

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

</details>

---

<details>
<summary><b>📊 `select()` vs `with_columns()`</b></summary>

Two fundamental methods that beginners often confuse.

Understanding their difference is crucial for effective data manipulation.

#### Core Difference

| Method | Behavior | Analogy |
| ------ | -------- | ------- |
| `select()` | **Keeps only** selected columns | Picking fruits - only take what you select |
| `with_columns()` | **Keeps all** columns + adds/modifies | Adding toppings - keep everything + add more |

#### Side-by-Side Comparison

```python
import polars as pl

# Original DataFrame
df = pl.DataFrame({
    "a": [1, 2, 3],
    "b": [4, 5, 6],
    "c": [7, 8, 9]
})

# Using select() - Only keeps selected columns
result1 = df.select(
    pl.col("a"),
    (pl.col("b") * 2).alias("b_doubled")
)
# Result: Only columns "a" and "b_doubled"
# ❌ Column "c" is gone!

# Using with_columns() - Keeps all columns
result2 = df.with_columns(
    (pl.col("b") * 2).alias("b_doubled")
)
# Result: Columns "a", "b", "c", and "b_doubled"
# ✅ All original columns preserved!
```

#### When to Use Each

**Use `select()` when:**

```python
# 1. You only need specific columns
df.select("name", "age")

# 2. Creating new computed columns (projection)
df.select(
    (pl.col("price") * pl.col("quantity")).alias("total")
)

# 3. Reordering columns
df.select("id", "name", "email")  # Explicit order
```

**Use `with_columns()` when:**

```python
# 1. Adding new computed columns
df.with_columns(
    (pl.col("price") * 1.1).alias("price_with_tax")
)

# 2. Modifying existing columns while keeping others
df.with_columns(
    pl.col("date").str.to_datetime(),  # Convert date
    pl.col("amount").cast(pl.Float64)   # Cast amount
)

# 3. Batch transformations
df.with_columns(
    pl.all().fill_null(0)  # Fill nulls in all columns
)
```

#### Quick Decision Rule

Ask yourself: **"Do I need to keep other columns?"**

- ✅ **Yes** → Use `with_columns()`
- ❌ **No** → Use `select()`

#### Common Pattern: Combine Both

```python
# First add computed columns, then select what you need
df.with_columns(
    (pl.col("a") + pl.col("b")).alias("sum"),
    (pl.col("a") * pl.col("b")).alias("product")
).select(
    "sum", "product", "c"  # Only keep these
)
```

</details>

---

<details>
<summary><b>🔗 Joins & Combining Data</b></summary>

Joining dataframes is one of the most common operations in data analysis. Polars provides efficient implementations of all standard SQL join types.

#### Sample Data

Let's create two example dataframes to demonstrate different join types:

```python
import polars as pl

# Customers table
customers = pl.DataFrame({
    "customer_id": [1, 2, 3, 4, 5],
    "name": ["Alice", "Bob", "Charlie", "David", "Eve"],
    "city": ["NYC", "LA", "Chicago", "Houston", "Phoenix"]
})

# Orders table
orders = pl.DataFrame({
    "order_id": [101, 102, 103, 104, 105],
    "customer_id": [1, 1, 2, 3, 6],  # Note: customer_id 6 doesn't exist in customers
    "amount": [250, 180, 420, 350, 290]
})
```

#### Join Types Comparison

| Join Type | Description | Use Case |
| --------- | ----------- | -------- |
| **Inner** | Only matching rows from both tables | Most common, excludes unmatched data |
| **Left** | All rows from left + matching from right | Keep all original data, add related info |
| **Outer** | All rows from both tables | Complete picture, includes all data |
| **Cross** | Cartesian product (all combinations) | Generate all possible pairs |
| **Semi** | Rows from left that have matches in right | Filter based on existence |
| **Anti** | Rows from left that DON'T have matches | Find missing relationships |

#### 1. Inner Join (Default)

Returns only rows where the join key exists in **both** dataframes.

```python
# Inner join - only customers who have orders
result = customers.join(orders, on="customer_id", how="inner")

# Result: 3 rows (customers 1, 2, 3)
# ┌─────────────┬─────────┬─────────┬──────────┬────────┐
# │ customer_id │ name    │ city    │ order_id │ amount │
# ├─────────────┼─────────┼─────────┼──────────┼────────┤
# │ 1           │ Alice   │ NYC     │ 101      │ 250    │
# │ 1           │ Alice   │ NYC     │ 102      │ 180    │
# │ 2           │ Bob     │ LA      │ 103      │ 420    │
# │ 3           │ Charlie │ Chicago │ 104      │ 350    │
# └─────────────┴─────────┴─────────┴──────────┴────────┘
```

#### 2. Left Join

Returns **all rows from the left** dataframe, with matching data from right (nulls if no match).

```python
# Left join - all customers, with their orders if they exist
result = customers.join(orders, on="customer_id", how="left")

# Result: 6 rows (all 5 customers, Alice appears twice)
# ┌─────────────┬─────────┬─────────┬──────────┬────────┐
# │ customer_id │ name    │ city    │ order_id │ amount │
# ├─────────────┼─────────┼─────────┼──────────┼────────┤
# │ 1           │ Alice   │ NYC     │ 101      │ 250    │
# │ 1           │ Alice   │ NYC     │ 102      │ 180    │
# │ 2           │ Bob     │ LA      │ 103      │ 420    │
# │ 3           │ Charlie │ Chicago │ 104      │ 350    │
# │ 4           │ David   │ Houston │ null     │ null   │  ← No orders
# │ 5           │ Eve     │ Phoenix │ null     │ null   │  ← No orders
# └─────────────┴─────────┴─────────┴──────────┴────────┘
```

#### 3. Outer Join (Full Outer)

Returns **all rows from both** dataframes, with nulls where data doesn't match.

```python
# Outer join - all customers AND all orders
result = customers.join(orders, on="customer_id", how="outer")

# Result: 7 rows (all customers + orphaned order)
# Includes customer_id 6 from orders (doesn't exist in customers)
```

#### 4. Cross Join

Creates a **Cartesian product** - every row from left combined with every row from right.

```python
# Cross join - all possible customer-order combinations
result = customers.join(orders, how="cross")

# Result: 5 × 5 = 25 rows (every customer paired with every order)
# ⚠️ Warning: Can create huge datasets! Use with caution.
```

#### 5. Semi Join

Returns rows from left that **have a match** in right (but doesn't include right's columns).

```python
# Semi join - customers who have placed orders
result = customers.join(orders, on="customer_id", how="semi")

# Result: 3 rows (only customers 1, 2, 3)
# ┌─────────────┬─────────┬─────────┐
# │ customer_id │ name    │ city    │
# ├─────────────┼─────────┼─────────┤
# │ 1           │ Alice   │ NYC     │
# │ 2           │ Bob     │ LA      │
# │ 3           │ Charlie │ Chicago │
# └─────────────┴─────────┴─────────┘
```

#### 6. Anti Join

Returns rows from left that **DON'T have a match** in right.

```python
# Anti join - customers who haven't placed any orders
result = customers.join(orders, on="customer_id", how="anti")

# Result: 2 rows (customers 4, 5)
# ┌─────────────┬───────┬─────────┐
# │ customer_id │ name  │ city    │
# ├─────────────┼───────┼─────────┤
# │ 4           │ David │ Houston │
# │ 5           │ Eve   │ Phoenix │
# └─────────────┴───────┴─────────┘
```

#### Advanced: Joining on Different Column Names

```python
# When join keys have different names
customers_alt = customers.rename({"customer_id": "cust_id"})

result = customers_alt.join(
    orders,
    left_on="cust_id",
    right_on="customer_id",
    how="inner"
)
```

#### Advanced: Multiple Join Keys

```python
# Join on multiple columns
df1 = pl.DataFrame({
    "year": [2023, 2023, 2024],
    "month": [1, 2, 1],
    "sales": [100, 150, 120]
})

df2 = pl.DataFrame({
    "year": [2023, 2024],
    "month": [1, 1],
    "target": [90, 110]
})

result = df1.join(df2, on=["year", "month"], how="left")
```

#### Performance Tips

- ✅ **Use inner joins** when possible (fastest)
- ✅ **Join on integers** rather than strings (faster)
- ✅ **Filter before joining** to reduce data size
- ✅ **Use semi/anti joins** instead of left join + filter for existence checks
- ⚠️ **Avoid cross joins** on large datasets (exponential growth)

#### Quick Decision Guide

**Need to:**

- ✅ Combine matching data → **Inner join**
- ✅ Keep all left data, add related info → **Left join**
- ✅ See everything from both sides → **Outer join**
- ✅ Filter by existence in another table → **Semi join**
- ✅ Find missing relationships → **Anti join**
- ⚠️ Generate all combinations → **Cross join** (use carefully!)

</details>

---

<details>
<summary><b>🎯 Selector API</b></summary>

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

</details>

---

<details>
<summary><b>🔄 `group_by()` vs `group_by_dynamic()`</b></summary>

Two methods for grouping data that serve different purposes.

Understanding when to use each is crucial for efficient time series analysis.

#### Core Difference

| Method | Purpose | Best For |
| ------ | ------- | -------- |
| `group_by()` | General-purpose grouping | Any grouping operation |
| `group_by_dynamic()` | **Time series grouping** | Fixed time interval windows |

#### Key Distinctions

| Feature | `group_by()` | `group_by_dynamic()` |
| ------- | ------------ | -------------------- |
| **Use Case** | General grouping | Time-based windowing |
| **Time Handling** | Manual extraction (`.dt.date()`) | Automatic window creation |
| **Sorting Requirement** | ❌ None | ✅ **Must be sorted ascending** |
| **Syntax** | More verbose for time | Concise with `every` parameter |
| **Performance** | Standard | Optimized for sorted time data |
| **Window Intervals** | Manual definition | Built-in (`"1d"`, `"3h"`, `"6h"`) |

#### Side-by-Side Comparison

```python
import polars as pl

df = pl.read_csv("trips.csv", try_parse_dates=True)

# ─────────────────────────────────────────────────────────
# Using group_by() - Manual time extraction
# ─────────────────────────────────────────────────────────
result1 = df.group_by(
    pl.col("pickup").dt.date().alias("date")  # Extract date component
).agg(
    pl.col("trip_distance").mean().round(1)
).sort("date")  # Manual sorting

# ─────────────────────────────────────────────────────────
# Using group_by_dynamic() - Automatic windowing
# ─────────────────────────────────────────────────────────
result2 = df.group_by_dynamic(
    "pickup",      # Time column (must be sorted!)
    every="1d"     # Window size: 1 day
).agg(
    pl.col("trip_distance").mean().round(1)
)  # Already sorted by time
```

#### Critical Requirement for `group_by_dynamic()`

⚠️ **The time column MUST be sorted in ascending order:**

```python
# Check if sorted
df["pickup"].is_sorted()  # Must return True

# If not sorted, sort first
df = df.sort("pickup")

# Or mark as sorted (if you know it's already sorted)
df = df.with_columns(pl.col("pickup").set_sorted())
```

#### Flexible Time Intervals

`group_by_dynamic()` supports intuitive interval strings:

```python
# Hourly windows
df.group_by_dynamic("pickup", every="1h")

# 6-hour windows
df.group_by_dynamic("pickup", every="6h")

# Daily windows
df.group_by_dynamic("pickup", every="1d")

# Weekly windows
df.group_by_dynamic("pickup", every="1w")

# Monthly windows
df.group_by_dynamic("pickup", every="1mo")
```

#### Combining with Regular Grouping

You can group by other columns **before** applying dynamic time windows:

```python
# Group by VendorID, then apply 3-hour windows
df.sort("VendorID", "pickup").group_by_dynamic(
    "pickup",
    every="3h",
    group_by="VendorID"  # First group by this column
).agg(
    pl.col("tip_amount").mean().round(1)
)
```

#### When to Use Each

**Use `group_by()` when:**

```python
# 1. Grouping by non-time columns
df.group_by("category").agg(pl.col("sales").sum())

# 2. Grouping by multiple non-time columns
df.group_by(["region", "product"]).agg(pl.col("revenue").mean())

# 3. Custom time grouping (not fixed intervals)
df.group_by(
    pl.col("date").dt.year(),
    pl.col("date").dt.quarter()
).agg(pl.col("sales").sum())
```

**Use `group_by_dynamic()` when:**

```python
# 1. Fixed time interval windows
df.group_by_dynamic("timestamp", every="1h")

# 2. Time series resampling
df.group_by_dynamic("date", every="1w").agg(pl.col("value").mean())

# 3. Rolling time windows with grouping
df.group_by_dynamic(
    "timestamp",
    every="5min",
    group_by="sensor_id"
)
```

#### Quick Decision Rule

Ask yourself: **"Am I grouping by fixed time intervals?"**

- ✅ **Yes** → Use `group_by_dynamic()` (faster, cleaner)
- ❌ **No** → Use `group_by()` (more flexible)

#### Performance Note

`group_by_dynamic()` uses a fast-track algorithm that requires sorted data. For large time series datasets, it's significantly faster than manually extracting time components with `group_by()`.

</details>

---

<details>
<summary><b>📦 Parquet File Format</b></summary>

**Parquet** is a **columnar binary file format** that has become the de facto standard for data analytics and big data processing. It's one of the most important file formats to understand when working with Polars.

#### Why Parquet?

Beyond being compact and fast, Parquet offers several critical advantages:

| Feature | Parquet | CSV |
| ------- | ------- | --- |
| **Storage Type** | Columnar (binary) | Row-based (text) |
| **File Size** | 🟢 Small (compressed) | 🔴 Large (uncompressed text) |
| **Read Speed** | 🟢 Very Fast | 🟡 Moderate |
| **Write Speed** | 🟢 Fast | 🟢 Fast |
| **Schema Preservation** | ✅ Yes (embedded metadata) | ❌ No (inferred on read) |
| **Data Types** | ✅ Rich types (nested, complex) | ❌ Limited (strings/numbers) |
| **Compression** | ✅ Built-in (Snappy, GZIP, ZSTD) | ❌ None (external only) |
| **Partial Reading** | ✅ Column pruning | ❌ Must read all columns |
| **Query Optimization** | ✅ Predicate pushdown | ❌ No optimization |

#### Core Features

**1. Columnar Storage**

- Data is stored by column, not by row
- Enables reading only the columns you need (column pruning)
- Perfect for analytical queries that typically access a subset of columns

**2. Built-in Compression**

- Supports multiple compression algorithms: Snappy (default), GZIP, LZO, Brotli, ZSTD
- Columnar layout improves compression ratios (similar data types compress better)
- Typical compression: 5-10x smaller than CSV

**3. Schema Evolution**

- Schema is embedded in the file (no need to specify data types on read)
- Supports adding, removing, or modifying columns over time
- Backward and forward compatibility

**4. Complex Data Types**

- Supports nested structures: arrays, maps, structs
- Can represent hierarchical data without flattening

**5. Data Partitioning**

- Organize data into directories by column values (e.g., `year=2024/month=01/`)
- Query only relevant partitions for massive speedups

**6. Statistics & Indexing**

- Stores min/max/null count for each column chunk
- Enables **predicate pushdown**: filter data during file reading, not after
- Skip entire row groups that don't match filter conditions

**7. Cross-Language Support**

- Apache project with standardized format
- Works seamlessly across Python, Java, C++, R, Rust, etc.

#### Parquet in Polars

Parquet is a **first-class citizen** in Polars. The columnar storage format aligns perfectly with Polars' columnar processing engine.

**Basic Usage:**

```python
import polars as pl

# Write DataFrame to Parquet
df.write_parquet("data.parquet")

# Read Parquet file (eager mode)
df = pl.read_parquet("data.parquet")

# Read Parquet file (lazy mode) - RECOMMENDED
lf = pl.scan_parquet("data.parquet")
result = lf.filter(pl.col("age") > 30).collect()
```

**Advanced Features:**

```python
# 1. Compression options
df.write_parquet("data.parquet", compression="zstd")  # Better compression
df.write_parquet("data.parquet", compression="snappy")  # Faster (default)

# 2. Predicate pushdown (filter during read)
# Only reads rows where age > 30 - MUCH faster!
df = pl.scan_parquet("data.parquet") \
       .filter(pl.col("age") > 30) \
       .collect()

# 3. Column pruning (read only specific columns)
# Only reads "name" and "age" columns - saves memory!
df = pl.scan_parquet("data.parquet") \
       .select(["name", "age"]) \
       .collect()

# 4. Partitioned datasets
# Write partitioned by year and month
df.write_parquet("data", partition_by=["year", "month"])

# Read partitioned dataset (auto-discovers partitions)
df = pl.scan_parquet("data/**/*.parquet").collect()

# 5. Streaming for large files
lf = pl.scan_parquet("huge_file.parquet")
result = lf.filter(pl.col("revenue") > 1000) \
           .group_by("country") \
           .agg(pl.col("revenue").sum()) \
           .collect(streaming=True)  # Process in batches
```

#### Parquet in PySpark

Parquet is also the **recommended default format** in PySpark:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("example").getOrCreate()

# Write DataFrame to Parquet
df.write.parquet("data.parquet")

# Read Parquet file
df = spark.read.parquet("data.parquet")

# Partitioned write (common pattern in data lakes)
df.write.partitionBy("year", "month").parquet("partitioned_data")

# Predicate pushdown (automatic in Spark)
df = spark.read.parquet("data.parquet") \
          .filter("age > 30")  # Filter pushed to file read
```

#### When to Use Parquet

✅ **Use Parquet when:**

- Working with analytical workloads (read-heavy, column-focused queries)
- Need to store data long-term (schema preservation)
- File size matters (compression)
- Working with Polars, Spark, or other big data tools
- Need fast query performance on large datasets
- Want to leverage predicate pushdown and column pruning

❌ **Use CSV when:**

- Need human-readable format for debugging
- Sharing data with non-technical users
- Working with simple, small datasets
- Need maximum compatibility with legacy systems

#### Best Practices

1. **Always use `scan_parquet()` instead of `read_parquet()`** for lazy evaluation
2. **Apply filters early** to leverage predicate pushdown
3. **Select only needed columns** to benefit from column pruning
4. **Use partitioning** for large datasets organized by time or category
5. **Choose compression wisely**:
   - `snappy` (default): Balanced speed and compression
   - `zstd`: Best compression ratio, slightly slower
   - `gzip`: Good compression, slower than snappy
6. **Use streaming mode** for files larger than RAM

#### Real-World Example

```python
import polars as pl

# ❌ Inefficient: Eager read, loads everything
df = pl.read_parquet("sales_100GB.parquet")
result = df.filter(pl.col("year") == 2024) \
           .select(["product", "revenue"]) \
           .group_by("product") \
           .agg(pl.col("revenue").sum())

# ✅ Efficient: Lazy + predicate pushdown + column pruning
result = pl.scan_parquet("sales_100GB.parquet") \
           .filter(pl.col("year") == 2024)  # Pushed to file read
           .select(["product", "revenue"])   # Only reads these columns
           .group_by("product") \
           .agg(pl.col("revenue").sum()) \
           .collect(streaming=True)  # Batch processing
```

**Performance difference:** The efficient version can be **10-100x faster** and use **90% less memory**!

</details>

---

<details>
<summary><b>⚠️ Common Pitfalls & Best Practices</b></summary>

Learn from common mistakes to write better, faster Polars code. These pitfalls catch even experienced users!

#### 1. 🚫 Collecting Too Early in Lazy Mode

**❌ Bad - Defeats the purpose of lazy evaluation:**

```python
# Loads ALL data into memory, then filters
df = pl.scan_csv("huge_file.csv").collect()  # ❌ Too early!
result = df.filter(pl.col("age") > 25)
```

**✅ Good - Let Polars optimize:**

```python
# Polars pushes filter to file reading (only loads needed rows)
result = pl.scan_csv("huge_file.csv") \
           .filter(pl.col("age") > 25) \
           .collect()  # ✅ Collect at the end
```

**Why it matters:** The bad version might load 10GB, the good version might only load 2GB!

---

#### 2. 🚫 Using `.apply()` Instead of Expressions

**❌ Bad - Slow Python loop:**

```python
# Runs Python function on each row (SLOW!)
df = df.with_columns(
    pl.col("price").apply(lambda x: x * 1.1).alias("price_with_tax")  # ❌
)
```

**✅ Good - Vectorized operation:**

```python
# Runs in parallel Rust code (FAST!)
df = df.with_columns(
    (pl.col("price") * 1.1).alias("price_with_tax")  # ✅
)
```

**Performance difference:** 100-1000x faster! Use expressions whenever possible.

---

#### 3. 🚫 Forgetting to Alias New Columns

**❌ Bad - Overwrites original column:**

```python
df = df.with_columns(
    pl.col("salary") * 1.1  # ❌ Replaces "salary" column!
)
```

**✅ Good - Creates new column:**

```python
df = df.with_columns(
    (pl.col("salary") * 1.1).alias("salary_increased")  # ✅ New column
)
```

---

#### 4. 🚫 Using `select()` When You Mean `with_columns()`

**❌ Bad - Loses other columns:**

```python
df = df.select(
    (pl.col("a") + pl.col("b")).alias("sum")
)
# ❌ Now you only have "sum" column! Lost "a", "b", and all others
```

**✅ Good - Keeps all columns:**

```python
df = df.with_columns(
    (pl.col("a") + pl.col("b")).alias("sum")
)
# ✅ Still have "a", "b", and all other columns + new "sum"
```

**Remember:** `select()` = "keep only these", `with_columns()` = "add/modify these"

---

#### 5. 🚫 Chaining Multiple `collect()` Calls

**❌ Bad - Inefficient back-and-forth:**

```python
lf = pl.scan_csv("data.csv")
df1 = lf.filter(pl.col("age") > 25).collect()  # ❌ First collect
df2 = df1.lazy().select(["name", "age"]).collect()  # ❌ Second collect
```

**✅ Good - Single optimized execution:**

```python
result = pl.scan_csv("data.csv") \
           .filter(pl.col("age") > 25) \
           .select(["name", "age"]) \
           .collect()  # ✅ One collect at the end
```

---

#### 6. 🚫 Not Handling Nulls Properly

**❌ Bad - Nulls propagate unexpectedly:**

```python
# If any value is null, result is null!
df.with_columns(
    (pl.col("a") + pl.col("b")).alias("sum")  # ❌ null + 5 = null
)
```

**✅ Good - Explicit null handling:**

```python
# Option 1: Fill nulls first
df.with_columns(
    (pl.col("a").fill_null(0) + pl.col("b").fill_null(0)).alias("sum")
)

# Option 2: Use coalesce
df.with_columns(
    pl.coalesce([pl.col("a") + pl.col("b"), pl.lit(0)]).alias("sum")
)
```

---

#### 7. 🚫 Using Pandas-style Iteration

**❌ Bad - Extremely slow:**

```python
# DON'T iterate over rows like Pandas!
for row in df.iter_rows():  # ❌ VERY SLOW
    # process row...
    pass
```

**✅ Good - Use expressions:**

```python
# Let Polars handle it in parallel
df = df.with_columns(
    pl.when(pl.col("age") > 30)
      .then(pl.lit("Senior"))
      .otherwise(pl.lit("Junior"))
      .alias("level")
)
```

**When you MUST iterate:** Use `iter_rows()` only as last resort, prefer expressions!

---

#### 8. 🚫 Ignoring Data Types

**❌ Bad - Implicit type conversions:**

```python
# String "123" + Integer 45 = ???
df = pl.DataFrame({
    "id": ["123", "456"],  # ❌ String!
    "value": [45, 67]
})
result = df.filter(pl.col("id") == 123)  # ❌ Won't match!
```

**✅ Good - Explicit type handling:**

```python
# Cast to correct type
result = df.filter(pl.col("id").cast(pl.Int64) == 123)

# Or fix at source
df = pl.DataFrame({
    "id": [123, 456],  # ✅ Integer from start
    "value": [45, 67]
})
```

---

#### 9. 🚫 Not Using Categorical for Repeated Strings

**❌ Bad - Wastes memory:**

```python
# Stores "New York" string millions of times
df = pl.DataFrame({
    "city": ["New York"] * 1_000_000  # ❌ Huge memory usage
})
```

**✅ Good - Use Categorical:**

```python
# Stores "New York" once, uses integer references
df = pl.DataFrame({
    "city": ["New York"] * 1_000_000
}).with_columns(
    pl.col("city").cast(pl.Categorical)  # ✅ Much less memory
)
```

**Memory savings:** Can be 10-50x less memory for repeated strings!

---

#### 10. 🚫 Reading CSV Without Schema Hints

**❌ Bad - Polars guesses types (might be wrong):**

```python
df = pl.read_csv("data.csv")  # ❌ Might infer "001" as integer 1
```

**✅ Good - Specify important types:**

```python
df = pl.read_csv(
    "data.csv",
    dtypes={"id": pl.Utf8, "amount": pl.Float64}  # ✅ Explicit types
)
```

---

#### 🎯 Quick Best Practices Checklist

**Do:**

- ✅ Use lazy mode for large datasets (`scan_*` instead of `read_*`)
- ✅ Collect only once, at the very end
- ✅ Use expressions instead of `.apply()`
- ✅ Specify data types explicitly when reading files
- ✅ Use `.alias()` to name new columns
- ✅ Handle nulls explicitly
- ✅ Use Categorical for repeated strings
- ✅ Filter and select early to reduce data size

**Don't:**

- ❌ Collect multiple times in a pipeline
- ❌ Use `.apply()` with Python functions (unless absolutely necessary)
- ❌ Iterate over rows (use expressions instead)
- ❌ Mix up `select()` and `with_columns()`
- ❌ Ignore null values
- ❌ Let Polars guess important data types

---

#### 💡 Performance Debugging Tips

If your Polars code is slow:

1. **Check if you're using lazy mode** - Add `.lazy()` and `.collect()`
2. **Look for `.apply()`** - Replace with expressions
3. **Check for multiple `.collect()`** - Combine into one
4. **Profile with `.explain()`** - See the query plan
5. **Use `.describe_optimized_plan()`** - See what Polars actually runs

```python
# See what Polars will actually execute
lf = pl.scan_csv("data.csv").filter(pl.col("age") > 25)
print(lf.explain())  # Shows the optimized plan
```

</details>

---

## Project Structure

<details>
<summary><b> View Full Project Structure (62 Notebooks)</b></summary>

```text
polars/
├── data/                         
├── notebook/                     
│   ├── 01_intro/                 
│   │   ├── 01_intro.ipynb
│   │   ├── 02_lazy_mode.ipynb
│   │   ├── 03_Lazy_model_evaluation.ipynb
│   │   ├── 04_data_types.ipynb
│   │   ├── 05_series_dataframe.ipynb
│   │   ├── 06_conversion_pandas_numpy.ipynb
│   │   └── 07_visualization.ipynb
│   │
│   ├── 02_filter_rows/          
│   │   ├── 01_select_rows.ipynb
│   │   ├── 02_select_filter.ipynb
│   │   └── 03_filter_in_lazy_mode.ipynb
│   │
│   ├── 03_selecting_trasformations/ 
│   │   ├── 01_select_columns.ipynb
│   │   ├── 02_select_method.ipynb
│   │   ├── 03_select_multi_columns.ipynb
│   │   ├── 04_add_new_column.ipynb
│   │   ├── 05_add_multi_columns.ipynb
│   │   ├── 06_add_column_conditionally.ipynb
│   │   ├── 07_sorting.ipynb
│   │   ├── 08_transform_columns.ipynb
│   │   └── 09_iterating.ipynb
│   │
│   ├── 04_dtype_missing_value/  
│   │   ├── 01_missing_value.ipynb
│   │   ├── 02_replace_missing_value.ipynb
│   │   ├── 03_replace_missing_value_expression.ipynb
│   │   ├── 04_numerical_dtype.ipynb
│   │   ├── 05_categorical.ipynb
│   │   ├── 06_categorical_cache.ipynb
│   │   ├── 07_nested_dtype.ipynb
│   │   ├── 08_list_dtype.ipynb
│   │   ├── 09_list_operation.ipynb
│   │   └── 10_text_manipulation.ipynb
│   │
│   ├── 05_grouping_&_agg/       
│   │   ├── 01_statistics.ipynb
│   │   ├── 02_value_counts.ipynb
│   │   ├── 03_groupby_object.ipynb
│   │   ├── 04_groupby_aggregation.ipynb
│   │   ├── 05_quantile_and_histogram.ipynb
│   │   ├── 06_group_operation.ipynb
│   │   └── 07_pivot.ipynb
│   │
│   ├── 06_combine_dataframe/    
│   │   ├── 01_concat.ipynb
│   │   ├── 02_join.ipynb
│   │   ├── 03_join_string_categorical.ipynb
│   │   ├── 04_filter_by_dataframe.ipynb
│   │   └── 05_inequality_joins.ipynb
│   │
│   ├── 07_io/              
│   │   ├── 01_single_csv.ipynb
│   │   ├── 02_excel.ipynb
│   │   ├── 03_json_and_serialization.ipynb
│   │   ├── 04_parquet.ipynb
│   │   ├── 05_batch_csv.ipynb
│   │   ├── 06_stream_csv.ipynb
│   │   ├── 07_multiple_csv.ipynb
│   │   └── 08_db_connection.ipynb
│   │
│   ├── 08_time_series/
│   │   ├── 01_preliminary.ipynb
│   │   ├── 02_time_series_intro.ipynb
│   │   ├── 03_date_time_range.ipynb
│   │   ├── 04_timezone.ipynb
│   │   ├── 05_datetime_string.ipynb
│   │   ├── 06_adjust_datetime.ipynb
│   │   ├── 07_time_series_fearures.ipynb
│   │   ├── 08_filter_time.ipynb
│   │   ├── 09_dynamic_group_by.ipynb
│   │   ├── 10_dynamic_group_by_window.ipynb
│   │   └── 11_rolling.ipynb
│   │
│   └── 09_machine_learning/ 
│       ├── 01_sklearn.ipynb
│       └── 02_polars_ols.ipynb
│
├── pyproject.toml        
├── poetry.lock         
└── README.md              
```

</details>
