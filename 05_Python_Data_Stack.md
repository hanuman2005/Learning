# The Python Data Stack for Backend Engineers

A working guide to NumPy, Pandas and scikit-learn, with brief coverage of matplotlib and Jupyter, written for engineers who build and operate backend services. It teaches the mental model behind each library so you can read, review, debug and productionise data code, not just recall function names.

**Audience:** backend developers who know Python and want to be credible with data and machine-learning code in a product-company role.
**Prerequisites:** comfortable Python (functions, classes, generators, virtual environments). Basic SQL helps for the Pandas chapters.
**Scope:** the single-machine Python data stack, plus how a backend engineer serves or schedules such work. Deep learning, distributed engines (Spark and similar) and statistics theory are out of scope.
**Accuracy status:** Reviewed against official documentation on 2026-11-06; items still unverified are marked.
**Note on accuracy:** the concepts and APIs described are long-stable. Defaults and deprecations change between releases (Pandas in particular changed copy semantics and several defaults across 2.x and 3.x). Confirm exact signatures and behaviour in the official documentation for the versions you pin. Code snippets are short and illustrative. The FastAPI snippet in chapter 14 follows the official lifespan pattern.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 15 lists hands-on labs. Chapter 16 collects interview questions with model answers. Chapter 17 lists primary references. Chapter 18 is a glossary of hard terms. Each chapter starts with an "In plain words" summary.

**What "good enough" means for a backend role:** you can explain why a vectorised operation is fast, read a Pandas transformation and spot a silent bug (a join that multiplies rows, a leaky split, a chained assignment), load and aggregate a dataset without exhausting memory, and wrap a trained model in a reliable service or batch job with versioning. You are not expected to invent models. Chapters 2, 3, 6, 11 and 14 matter most for backend interviews.

**What interviewers realistically ask.** For a backend role, this stack is usually a secondary topic. Expect questions about data handling (joins, aggregation, memory), correctness traps (leakage, nulls, time zones), and integration (how you would serve a model, schedule a job, version artefacts, monitor drift). Rarely will you be asked to derive an algorithm. If the role is "ML platform" or "data-intensive backend", the depth increases toward chapters 11 to 14. Chapter 16 marks which questions are common for which level.

## Contents

1. The mental model: where the stack fits
2. NumPy
3. Pandas: data structures and indexing
4. Pandas: groupby, joins and reshaping
5. Pandas: missing data, time and categories
6. Pandas: I/O, memory, performance and its limits
7. scikit-learn: the estimator API and preprocessing
8. Pipelines and ColumnTransformer
9. Common models and when to use them
10. Metrics
11. Validation: cross-validation, leakage, fit and imbalance
12. Hyperparameter search and model persistence
13. matplotlib and the Jupyter workflow
14. Serving and scheduling: batch versus online
15. Practice labs
16. Interview questions
17. References
18. Glossary

---

## 1. The mental model: where the stack fits

**In plain words:** Data work in Python is built in layers. NumPy holds the numbers, Pandas adds labelled tables, and scikit-learn learns from them. Knowing the layers helps you find where a wrong number came from.

### Understand

**Layers.** The stack is built bottom-up, and knowing the layering explains most behaviour.

| Layer | Library | Role |
|---|---|---|
| Memory and arithmetic | NumPy | typed, contiguous n-dimensional arrays and compiled loops |
| Labelled tabular data | Pandas | columns of mixed types with an index, built on NumPy (and Arrow in newer versions) |
| Learning algorithms | scikit-learn | uniform API over models and preprocessing, consuming arrays or DataFrames |
| Columnar storage | pyarrow, Parquet | efficient on-disk and in-memory columnar format |
| Presentation | matplotlib, Jupyter | plots and interactive exploration |
| Persistence and parallelism | joblib | saving Python objects, simple parallel loops |

**Why this stack is fast.** Plain Python loops are slow because each element is a boxed object and each operation is dispatched dynamically. NumPy stores raw numbers in one contiguous block and runs compiled loops over them. Pandas and scikit-learn push work into those compiled loops. The rule that follows: **stay inside the library's operations and avoid Python-level loops over rows.**

**Why it matters to a backend engineer.** Data code differs from request-handling code in three ways: it is memory-bound rather than I/O-bound, its bugs are often silent (wrong numbers, not exceptions), and its state is large (datasets, fitted models). Reviewing it well means checking shapes, dtypes, nulls, key uniqueness and what data a model saw at training time.

**Exploration versus production.** A notebook is for exploring. Production code is a tested module, a pinned environment, a scheduled job or a service. Much of this guide is about the crossing from the first to the second.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Environment isolation | `venv`, `pip`, `uv`, `pip freeze`, lock files | Reproduce an exact environment on another machine |
| Inspecting versions | `python -c "import pandas; print(pandas.__version__)"`, `pip list` | Know which versions behaviour depends on |
| Interactive probing | `python -i`, IPython, `help()`, `?` in Jupyter | Read a function's signature and docstring quickly |

### Common mistakes

- Looping over DataFrame rows in Python when a column operation exists.
- Not pinning library versions, so a retrained model or a job behaves differently after an upgrade.
- Treating a notebook as the deliverable.

### Self-check

- Why is `sum(my_list)` slower than `my_array.sum()` for large data?
- Which layer would you inspect first if a number looks wrong after a join?

---

## 2. NumPy

**In plain words:** NumPy stores many numbers of the same type in one block of memory. Because of this, it can do maths on all of them at once (this is called vectorisation, meaning one operation on a whole array). Learn when a result shares memory with the original and when it is a separate copy.

### Understand

**The ndarray memory model.** An `ndarray` is a block of raw memory plus metadata: `dtype` (element type), `shape`, and `strides` (bytes to step along each axis). Elements are all the same type, which is what makes compiled loops possible. Indexing and reshaping usually change only the metadata, not the data.

```python
import numpy as np

a = np.arange(12).reshape(3, 4)
a.shape, a.dtype, a.strides      # (3, 4), int64 on most 64-bit Linux builds, (32, 8)
a.T.strides                      # transposed view: same memory, swapped strides
```

**Dtypes.** The dtype decides memory use and numeric behaviour.
- Common: `int64`, `int32`, `float64`, `float32`, `bool`, `datetime64`, and `object` (array of Python references, slow and not vectorised).
- Integers have fixed width and **wrap around on overflow** silently in array arithmetic. Floats have rounding error: never compare with `==`, use `np.isclose`.
- `float32` halves memory against `float64` at the cost of precision. Many ML workloads accept it.
- `NaN` is a float concept. An integer array cannot hold NaN, which is why Pandas upcasts integer columns with missing values (chapter 5).
- Default integer size is platform-dependent on some systems; state dtypes explicitly when they matter.

**Views versus copies.** This is the most important NumPy concept for avoiding bugs.
- **Basic slicing** (`a[1:3]`, `a[:, 0]`, `a.T`, `a.reshape(...)` when possible) returns a **view**: new metadata over the same memory. Writing through a view changes the original.
- **Fancy indexing** (an integer array or list as index) and **boolean masks** return a **copy**.
- `reshape` returns a view when it can and a copy when it must. Do not rely on either; call `.copy()` when you need independence.
- Check with `np.shares_memory(x, y)` or `x.base is not None`.

```python
a = np.arange(5)
v = a[1:4]          # view
v[0] = 99           # a[1] is now 99
c = a[[1, 2, 3]]    # copy
c[0] = -1           # a is unchanged
```

**Broadcasting.** Rules for combining arrays of different shapes without copying. Align shapes from the right. Two dimensions are compatible if they are equal or one of them is 1. A dimension of 1 is stretched.

```python
x = np.array([[1.0], [2.0], [3.0]])   # shape (3, 1)
y = np.array([10.0, 20.0, 30.0, 40.0]) # shape (4,)
x + y                                  # shape (3, 4)
```

Typical use: centre each column by subtracting `X.mean(axis=0)`, which has shape `(n_features,)` and broadcasts against `(n_samples, n_features)`. Shape mismatches produce a `ValueError`; an unintended broadcast (for example `(n,)` with `(n, 1)` yielding `(n, n)`) produces a silently huge wrong result.

**Axis semantics.** `axis=k` means "collapse axis k". For a 2-D array of shape `(rows, cols)`:
- `a.sum(axis=0)` collapses rows and returns one value per column.
- `a.sum(axis=1)` collapses columns and returns one value per row.
- `keepdims=True` retains the collapsed axis with length 1, which makes later broadcasting straightforward.

**Vectorisation.** Express the computation as whole-array operations: arithmetic, comparison, `np.where`, reductions (`sum`, `mean`, `std`, `min`, `max`, `argmax`), `np.dot`/`@`, `np.cumsum`. Each runs a compiled loop. Vectorising typically yields order-of-magnitude speedups over Python loops. Trade-off: intermediate arrays use memory, so a very long expression on huge data can double or triple peak memory.

**Other essentials.**
- Random numbers: `rng = np.random.default_rng(seed)` then `rng.normal(...)`, `rng.integers(...)`. The modern Generator API is preferred over the legacy global `np.random.seed`.
- Sorting and selection: `np.sort`, `np.argsort`, `np.unique(..., return_counts=True)`, `np.searchsorted`.
- Linear algebra: `@`, `np.linalg.solve`, `np.linalg.norm`. Prefer `solve` over explicit inverse.
- Memory-mapped files: `np.memmap` or `np.load(..., mmap_mode="r")` to work with arrays larger than RAM.
- `np.nan*` functions (`nanmean`, `nansum`) ignore NaN; plain reductions propagate it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Array creation | `np.array`, `zeros`, `ones`, `arange`, `linspace`, `np.empty`, `np.full` | Create with an explicit dtype |
| Shape manipulation | `reshape`, `ravel`, `T`, `np.newaxis`, `concatenate`, `stack`, `split` | Predict the output shape before running |
| Memory model | `.dtype`, `.strides`, `.flags`, `.base`, `np.shares_memory`, `.nbytes` | Prove whether an operation copied |
| Indexing | slices, boolean masks, `np.where`, `np.take`, `np.argwhere` | Select without Python loops |
| Reductions | `sum`, `mean`, `std`, `argmax`, `axis=`, `keepdims=` | Reduce along the intended axis |
| Floating-point care | `np.isclose`, `np.allclose`, `np.errstate`, `np.nan*` functions | Compare floats and handle NaN safely |
| Persistence | `np.save`, `np.load`, `np.savez`, `np.memmap` | Store and re-load arrays exactly |

### Common mistakes

- Writing into a slice and corrupting the original array (view aliasing).
- Using `==` on floats.
- Integer overflow in `int32` accumulation, or an `int` counter in a small dtype.
- Appending in a loop (`np.append` copies the whole array each time). Collect in a list, then convert once.
- Confusing `axis=0` and `axis=1`, particularly on Pandas and NumPy interchangeably.
- Leaving `object` dtype arrays when numeric data was intended.

### Self-check

- Which of slicing, boolean masking and fancy indexing return views, and how do you confirm it?
- What shape results from adding arrays of shape `(5, 1)` and `(5,)`? Is that usually what you want?
- What does `a.sum(axis=0, keepdims=True)` return for `a.shape == (3, 4)`?
- Why is `np.append` in a loop slow?

---

## 3. Pandas: data structures and indexing

**In plain words:** A Pandas table is a set of columns that share one list of row labels. Pandas matches data by label, not by position. Use `loc` and `iloc` on purpose, and change data in one step so the change reaches the real table.

### Understand

**Series and DataFrame.** A **Series** is a one-dimensional labelled array: values plus an **index**. A **DataFrame** is a table: an ordered collection of Series (columns) sharing one row index, each column with its own dtype. Think of it as a dict of columns, not a list of rows. This is why column operations are fast and row-wise Python loops are slow.

**The index.** The index labels rows and drives **alignment**: arithmetic between two Series or DataFrames matches by label, not by position, and produces NaN where labels do not match. This is powerful and a frequent source of surprise.

```python
import pandas as pd

s1 = pd.Series([1, 2, 3], index=["a", "b", "c"])
s2 = pd.Series([10, 20, 30], index=["b", "c", "d"])
s1 + s2     # a NaN, b 12, c 23, d NaN
```

Index uses: fast lookup by label, join keys, time-series slicing. A non-unique index is allowed and can cause duplicated rows in joins and assignments. After filtering, the index keeps the original labels; `reset_index(drop=True)` restores a 0..n-1 range.

**Selection: `loc`, `iloc`, `[]`.**
- `df.loc[row_labels, col_labels]` selects by **label**. Slices with labels are **inclusive** of the end.
- `df.iloc[row_positions, col_positions]` selects by **integer position**. Slices are end-exclusive, like Python.
- `df["col"]` returns a Series; `df[["a", "b"]]` returns a DataFrame; `df[mask]` filters rows.
- Use `loc`/`iloc` when assigning; they express intent clearly.

**Boolean masks.** Comparisons produce a boolean Series; combine with `&`, `|`, `~` and **parenthesise each condition** because of operator precedence. Python's `and`/`or` do not work elementwise.

```python
active_big = df.loc[(df["status"] == "active") & (df["amount"] > 100), ["id", "amount"]]
```

`query` offers a string form. `isin`, `between`, `str.contains` and `notna` cover many mask needs.

**Chained assignment and SettingWithCopy.** Consider:

```python
df[df["amount"] > 100]["flag"] = True     # chained: may modify a temporary copy
```

`df[mask]` may return a copy or a view depending on internals, so the assignment may or may not reach `df`. Historically Pandas emitted `SettingWithCopyWarning` for suspected cases. The safe pattern is a **single `loc` call**:

```python
df.loc[df["amount"] > 100, "flag"] = True
```

Pandas introduced **Copy-on-Write** semantics: any derived object behaves as an independent copy, with the copy deferred until a write, and chained assignment never updates the original. It is opt-in in 2.x (`pd.options.mode.copy_on_write = True`) and the default and only mode in 3.0 (verified in the pandas 3.0 migration guide). Write code that is correct under both: assign through `loc`, and call `.copy()` explicitly when you mean to modify a subset independently.

**Creating columns.** `df["new"] = df["a"] * df["b"]`, or `df.assign(new=lambda d: d["a"] * d["b"])` for method chaining. `df.apply(func, axis=1)` runs Python per row and is slow; prefer vectorised expressions, `np.where`, `np.select`, or `Series.map` with a dict.

**Inspection.** First look at any new dataset: `df.shape`, `df.dtypes`, `df.head()`, `df.info()`, `df.describe(include="all")`, `df.isna().sum()`, `df.duplicated().sum()`, `df["col"].value_counts(dropna=False)`, and `df.memory_usage(deep=True)`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Construction | `pd.DataFrame(dict)`, `pd.Series`, `pd.read_*`, `pd.concat` | Build frames from dicts, records and files |
| Selection | `loc`, `iloc`, `[]`, `query`, `isin`, `between`, `.at`/`.iat` | Predict the result type and shape of any selection |
| Assignment | `df.loc[mask, col] = v`, `assign`, `rename`, `drop`, `insert` | Modify without chained assignment |
| Index handling | `set_index`, `reset_index`, `reindex`, `sort_index`, `MultiIndex` basics | Explain alignment and when to reset |
| Inspection | `info`, `describe`, `dtypes`, `value_counts`, `nunique`, `duplicated` | Profile an unfamiliar table in five minutes |
| Row-wise logic | `np.where`, `np.select`, `map`, `str.*` accessors | Replace `apply(axis=1)` with vector operations |

### Common mistakes

- Chained indexing for assignment (`df[a][b] = v`).
- Forgetting parentheses around each condition in a combined mask.
- Misusing `loc` with integer labels after filtering, expecting positional behaviour.
- Assuming operations are in place. Most return a new object; `inplace=True` does not generally save memory and is discouraged.
- Ignoring index alignment, producing unexpected NaN rows.
- Iterating with `iterrows()` on large frames.

### Self-check

- What is the difference between `loc[1:3]` and `iloc[1:3]` on a frame with a default index?
- Why can `df[df.x > 0]["y"] = 1` fail to change `df`, and what is the correct form?
- What does `s1 + s2` return when the indexes differ?

---

## 4. Pandas: groupby, joins and reshaping

**In plain words:** Grouping lets you summarise rows by a key, like SQL GROUP BY. A join combines two tables, and repeated keys can silently multiply rows. Always check the row count after a join.

### Understand

**groupby: split, apply, combine.** `df.groupby(keys)` splits rows into groups by key, applies a function per group, and combines the results. Three shapes of result matter:

| Operation | Output shape | Use |
|---|---|---|
| `agg` | one row per group | summaries (sum, mean, count, custom) |
| `transform` | same length and index as the input | per-row group statistics (for example the group mean, a z-score within group) |
| `apply` / `filter` | variable | custom logic; `filter` keeps or drops whole groups |

```python
summary = (
    df.groupby("customer_id")
      .agg(orders=("order_id", "count"),
           revenue=("amount", "sum"),
           last_seen=("created_at", "max"))
      .reset_index()
)

df["share_of_customer"] = df["amount"] / df.groupby("customer_id")["amount"].transform("sum")
```

Named aggregation (`name=(column, function)`) keeps output columns clear. Know `count` (non-null values) versus `size` (rows including nulls). By default rows with a null group key are dropped; `dropna=False` keeps them. Choose `as_index=False` or `reset_index()` depending on whether you want keys as columns.

Other grouped tools: `rank`, `cumsum`, `shift`, `diff`, `rolling`, `expanding`, `nlargest`, `first`/`last`. `shift` within groups gives "previous value per entity", a core feature-engineering idiom. These are the Pandas analogue of SQL window functions.

**Joins.** `pd.merge(left, right, on=..., how=...)` is the analogue of SQL join.
- `how`: `inner`, `left`, `right`, `outer`, and `cross`.
- Different key names: `left_on=` and `right_on=`. Index joins: `left_index=True` or `df.join(other)`.
- **Key uniqueness decides row count.** If the key is duplicated in both inputs you get a many-to-many join, multiplying rows. This silently inflates sums downstream. Guard with `validate="one_to_one"`, `"one_to_many"`, `"many_to_one"`, or `"many_to_many"`, which raises if the assumption is false.
- `indicator=True` adds a `_merge` column showing which side each row came from, useful to audit unmatched rows.
- Overlapping non-key column names get suffixes (`_x`, `_y` by default); set `suffixes=` explicitly.
- Null keys match each other in Pandas merges, unlike SQL. (Not re-verified against the docs; test it on your version.) Drop null keys deliberately.
- `pd.merge_asof` joins on the nearest key (time-series "latest value as of"), and `pd.concat` stacks frames vertically or horizontally.

```python
merged = orders.merge(customers, on="customer_id", how="left", validate="many_to_one")
assert len(merged) == len(orders)
```

**Reshaping.**
- **Wide versus long.** Long (tidy) format has one observation per row; most grouping, plotting and modelling tools prefer it.
- `melt` converts wide to long. `pivot` and `pivot_table` convert long to wide (`pivot_table` aggregates duplicates, `pivot` raises on them). `stack`/`unstack` move index levels between rows and columns.
- `pd.crosstab` counts combinations. `explode` expands list-valued cells into rows. `pd.get_dummies` one-hot encodes (in a modelling pipeline prefer scikit-learn's encoder so training and serving match).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Aggregation | `groupby`, `agg` with named aggregation, `size`, `count`, `nunique` | Produce any per-entity summary in one expression |
| Group-aligned values | `transform`, `rank`, `shift`, `cumsum`, `rolling` | Add group statistics and lag features without a join |
| Joining | `merge`, `join`, `concat`, `merge_asof`, `validate=`, `indicator=` | Join safely and prove row counts |
| Reshaping | `melt`, `pivot`, `pivot_table`, `stack`, `unstack`, `explode`, `crosstab` | Convert between wide and long deliberately |
| Deduplication | `drop_duplicates`, `duplicated`, `nlargest`, `sort_values` | Keep the latest record per key |

### Common mistakes

- Joining on a non-unique key and not noticing the row count grew.
- Using `apply` with a lambda per group when a built-in aggregation exists.
- Confusing `agg` and `transform` and getting a shape error.
- Computing group means then merging them back, when `transform` does it in one step.
- Not sorting before `drop_duplicates(keep="last")` or `shift`, so "latest" is arbitrary.

### Self-check

- Write the Pandas equivalent of `SELECT customer_id, SUM(amount) FROM orders GROUP BY customer_id HAVING SUM(amount) > 1000`.
- A left join returned more rows than the left table. What happened and how do you prevent it?
- When do you use `transform` instead of `agg`?

---

## 5. Pandas: missing data, time and categories

**In plain words:** Real data has gaps, dates and repeated text values. This chapter shows how to handle missing values, time zones and category columns without changing the meaning of the data. The main rule is to decide what a gap means before you fill it.

### Understand

**Missing data.**
- Missing values appear as `NaN` (float), `None` (object), `NaT` (datetime) and, with nullable dtypes, `pd.NA`. `df.isna()` detects them all.
- A classic NumPy-backed integer column cannot hold NaN, so a missing value forces the column to float. **Nullable dtypes** (`Int64`, `boolean`, `string`) hold `pd.NA` without changing the type. Confirm current behaviour in the docs, because newer versions are moving defaults.
- Comparisons with NaN are false; `NaN != NaN`. Reductions skip NaN by default (`skipna=True`), whereas NumPy's do not.
- Options: `dropna` (by row or column, with `how`, `subset`, `thresh`), `fillna` (constant, or `ffill`/`bfill`), interpolation, or model-based imputation (`SimpleImputer` in scikit-learn).
- **Missingness is information.** Decide why data is missing (not collected, not applicable, error). Sometimes an `is_missing` indicator feature is more useful than imputing. For a model, impute inside the pipeline using training data only (chapter 11).
- Never fill silently with 0 where 0 is a valid value.

**Datetime handling.**
- Parse with `pd.to_datetime(col, errors="coerce", utc=True)`. Explicit `format=` is faster and unambiguous. Ambiguous strings such as `03/04/2024` depend on `dayfirst`.
- The `.dt` accessor exposes components: `.dt.year`, `.dt.dayofweek`, `.dt.hour`, `.dt.floor("D")`, `.dt.tz_convert(...)`.
- **Time zones.** Store and compute in UTC; convert for display. Mixing timezone-aware and naive timestamps raises errors or yields wrong comparisons. Daylight-saving transitions produce ambiguous or non-existent local times.
- `Timedelta` is a duration; subtracting two datetimes gives one. `.dt.total_seconds()` converts.
- With a `DatetimeIndex`: slice by string (`df.loc["2024-01"]`), `resample("D").sum()` to aggregate to a time grid, `rolling("7D")` for time-based windows, `shift` for lags. `pd.Grouper(key="ts", freq="W")` resamples inside a groupby.
- Time-series validation needs time-ordered splits (chapter 11).

**Categorical data.**
- `astype("category")` stores a column as integer codes plus a table of categories. Memory drops sharply for low-cardinality strings (status, country, plan) and groupby speeds up.
- Beware: operations that combine categoricals with different category sets can fall back to `object`, and groupby on categoricals may include unobserved categories. The `observed` parameter now defaults to `True` (changed in pandas 3.0); in 2.x the default was `False`, so set it explicitly.
- For a model, a category is not a number. Use an encoder (one-hot, ordinal, target encoding with care) inside the pipeline.

**Strings.** `.str` accessor methods are vectorised in interface but loop in Python internally for object dtype, so they are convenient rather than fast. Normalise (strip, lower) before joins.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Null handling | `isna`, `notna`, `dropna`, `fillna`, `ffill`, `interpolate`, nullable dtypes | Choose a policy per column and justify it |
| Datetime parsing | `to_datetime`, `.dt` accessor, `tz_localize`, `tz_convert`, `Timedelta` | Parse, normalise to UTC and extract features |
| Time series | `set_index`, `resample`, `rolling`, `shift`, `merge_asof`, `pd.Grouper` | Aggregate events to daily or weekly series |
| Categories | `astype("category")`, `.cat` accessor, `pd.Categorical` | Cut memory on low-cardinality columns |
| Text cleanup | `.str.strip`, `.str.lower`, `.str.replace`, `.str.extract` | Normalise join keys |

### Common mistakes

- `fillna(0)` on a column where zero has meaning.
- Mixing naive and aware datetimes, or doing arithmetic in local time across a DST change.
- Parsing dates without a format and getting day/month swapped.
- Checking `x == np.nan` (always false). Use `isna`.
- Assuming a column of integers stays integer after a left join introduces nulls.

### Self-check

- Why does an integer column become float when one value is missing, and how can you avoid it?
- How do you compute a seven-day rolling sum per customer?
- Why store timestamps in UTC?

---

## 6. Pandas: I/O, memory, performance and its limits

**In plain words:** This chapter is about reading data, saving memory and knowing when Pandas is too small for the job. Pandas keeps the whole table in memory in one process. For large data or a database question, use another tool.

### Understand

**Reading data.**
- `pd.read_csv(path, usecols=[...], dtype={...}, parse_dates=[...], na_values=[...], nrows=..., chunksize=...)`. Specify `usecols` and `dtype` up front: type inference on a huge file costs memory and time and can guess wrongly (an ID with leading zeros becomes an integer; a mixed column becomes `object`).
- `chunksize` returns an iterator of frames so you can aggregate a file larger than memory, keeping only partial results.
- **Parquet** (`pd.read_parquet`, `df.to_parquet`, using the `pyarrow` engine) is a typed, compressed, columnar format. It preserves dtypes, reads only requested columns (`columns=[...]`), supports predicate filtering and partitioned directories. For analytics data it is almost always better than CSV: smaller, faster, and no parsing ambiguity. CSV remains the interchange format for humans and simple tools.
- `pd.read_json`, `read_excel`, `read_feather` exist; `json_normalize` flattens nested API JSON.
- **SQL.** `pd.read_sql(query, connection)` or `read_sql_query`, using a SQLAlchemy connectable, a database URI string, an ADBC connection (pandas 2.2 and later) or a `sqlite3` connection. Other raw DBAPI connections (for example from psycopg2) are not officially supported. Push filtering, joining and aggregation **into the database**, and pull only the result. Use parameterised queries, never string-formatted SQL, to avoid injection. `df.to_sql(name, engine, if_exists=..., method=..., chunksize=...)` writes back; for large loads the database's bulk loader (for example `COPY` in PostgreSQL) is much faster.

```python
import pandas as pd
from sqlalchemy import create_engine, text

engine = create_engine(DATABASE_URL)   # from configuration, never hard-coded
df = pd.read_sql(
    text("SELECT id, amount, created_at FROM orders WHERE created_at >= :since"),
    engine,
    params={"since": "2024-01-01"},
)
```

**Memory.** A DataFrame is often several times larger in memory than the file it came from.
- Measure: `df.memory_usage(deep=True)`; `deep=True` counts the string contents of `object` columns.
- Reduce: read only needed columns; downcast numerics (`pd.to_numeric(..., downcast="integer")`, `float32` where precision allows); convert low-cardinality strings to `category`; use Arrow-backed or `string` dtypes where supported; drop intermediates (`del`, and let garbage collection run).
- Peak memory during an operation (a merge, a sort, a copy) can be two to three times the frame size. Plan for the peak.

**Performance habits.**
1. Vectorise; avoid `iterrows` and row-wise `apply`.
2. Filter and select columns early, before joins.
3. Use `merge` with indexed or sorted keys; avoid repeated concatenation in a loop (collect pieces in a list, then one `pd.concat`).
4. Prefer built-in groupby aggregations to `apply`.
5. Profile before optimising: `%timeit` and `%prun` in Jupyter, `cProfile`, and memory measurement. Optimise the measured bottleneck.
6. Method chaining with `assign`, `pipe` and `query` keeps transformations readable and testable.

**When Pandas is the wrong tool.** Pandas is a single-process, in-memory, eager library. Reach for something else when:
- **The data lives in a database and the question is a filter, join or aggregate.** Write SQL. The database has indexes and a query planner.
- **The data exceeds comfortable RAM** (a rule of thumb: when frames approach a third to a half of memory). Options include chunked processing, DuckDB or Polars (single-machine engines with out-of-core or lazy execution), or a distributed engine. Name them and the trade-off; do not claim one is universally better.
- **You need low-latency per-request logic.** Pandas has per-call overhead of milliseconds. In a request path, use plain Python or NumPy on small inputs, never a DataFrame built per request.
- **You need concurrent, transactional state.** That is a database's job.
- **The pipeline needs scheduling, retries and lineage at scale.** Use an orchestrator around it (chapter 14).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| CSV | `read_csv` with `usecols`, `dtype`, `parse_dates`, `chunksize`; `to_csv(index=False)` | Load a large file with controlled types and memory |
| Parquet | `read_parquet`, `to_parquet`, `pyarrow`, partitioned datasets | Choose and justify Parquet over CSV |
| SQL | `read_sql`, SQLAlchemy engine, parameterised queries, `to_sql` | Pull a result set safely and write back in batches |
| Memory | `memory_usage(deep=True)`, `info(memory_usage="deep")`, `astype`, `category`, downcasting | Cut a frame's footprint by half or more |
| Profiling | `%timeit`, `%prun`, `cProfile`, `tracemalloc` | Find the real bottleneck before changing code |
| Alternatives | DuckDB, Polars, SQL in the database, chunked processing | Know when to leave Pandas and what to move to |

### Common mistakes

- `SELECT *` into a DataFrame, then filtering in Pandas.
- Loading a whole CSV with default inference, then discovering dtype errors in production.
- Building a DataFrame inside a request handler.
- Calling `concat` or `append`-like operations in a loop.
- Storing intermediate data as CSV and losing dtypes and time zones on the round trip.
- Formatting SQL with f-strings.

### Self-check

- Why is Parquet usually preferable to CSV for a data pipeline?
- A 2 GB CSV exhausts memory on an 8 GB machine. List three techniques to process it.
- Why does `memory_usage()` without `deep=True` understate a frame with string columns?
- Give two situations where you would not use Pandas.

---

## 7. scikit-learn: the estimator API and preprocessing

**In plain words:** A model learns a rule from examples. In scikit-learn every learning step has the same methods: `fit` learns, `transform` or `predict` applies. Learn from the training data only, then apply the result to new data.

### Understand

**What machine learning is, for a backend engineer.** A model is a function learned from examples. **Supervised learning** maps features `X` to a target `y`: **classification** predicts a category (fraud or not), **regression** predicts a number (delivery time). **Unsupervised learning** (clustering, dimensionality reduction) has no target. The deliverable of training is a **fitted object**: parameters learned from data, which you then apply to new data.

**Data conventions.** `X` is a 2-D array or DataFrame of shape `(n_samples, n_features)`. `y` is 1-D of shape `(n_samples,)`. Features must be numeric by the time they reach a model, so categorical and text columns need encoding.

**The estimator API.** Consistency is scikit-learn's strength. Learn these three roles:
- **Estimator**: has `fit(X, y)`. It learns from data and stores learned attributes with a trailing underscore (`coef_`, `classes_`). Constructor arguments are **hyperparameters** (set by you, not learned).
- **Predictor**: has `predict(X)`; classifiers also `predict_proba(X)` and/or `decision_function(X)`; all have `score(X, y)`.
- **Transformer**: has `fit(X)` and `transform(X)` (and `fit_transform`). It learns statistics (a mean, a vocabulary) in `fit` and applies them in `transform`.

```python
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

scaler = StandardScaler().fit(X_train)       # learn mean and std from training data only
model = LogisticRegression(max_iter=1000).fit(scaler.transform(X_train), y_train)
pred = model.predict(scaler.transform(X_test))
```

The critical rule: **fit on training data only; apply (`transform`) to validation, test and production data.** Fitting a scaler on all the data lets test-set information leak into training (chapter 11). Pipelines (chapter 8) enforce this automatically.

**Train/test split.** Hold out a test set the model never influences; it estimates performance on unseen data. `stratify=y` preserves class proportions in imbalanced problems. `random_state` makes the split reproducible. For time-dependent data, split by time instead of randomly. A further **validation** set, or cross-validation, is used for choosing models and hyperparameters, so the test set is touched once at the end.

**Preprocessing.**

| Need | Transformer | Notes |
|---|---|---|
| Missing values | `SimpleImputer` (mean, median, most_frequent, constant) | add `add_indicator=True` to keep a missingness flag |
| Scale numbers | `StandardScaler`, `MinMaxScaler`, `RobustScaler` | needed for distance-based and regularised linear models and neural nets; unnecessary for trees |
| Encode categories | `OneHotEncoder(handle_unknown="ignore")`, `OrdinalEncoder` | one-hot for nominal data; handle unseen categories at serving time |
| Text | `TfidfVectorizer`, `CountVectorizer` | produce sparse matrices |
| Features | `PolynomialFeatures`, `FunctionTransformer`, `KBinsDiscretizer` | custom logic via `FunctionTransformer` |
| Dimensionality | `PCA` | scale first |

High-cardinality categoricals (user IDs, free-text keys) make one-hot encoding enormous; consider hashing, frequency encoding, target encoding done inside cross-validation, or a model that handles categoricals natively. Confirm current options in the docs.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Estimator API | `fit`, `predict`, `predict_proba`, `transform`, `get_params`, `set_params` | Explain what each call does and what state it creates |
| Splitting | `train_test_split` (`stratify`, `random_state`), `StratifiedKFold`, `TimeSeriesSplit` | Choose the right split for the data |
| Imputation | `SimpleImputer`, `IterativeImputer` (still experimental; needs `from sklearn.experimental import enable_iterative_imputer` first) | Impute within a pipeline |
| Scaling | `StandardScaler`, `MinMaxScaler`, `RobustScaler` | Say which models need scaling and why |
| Encoding | `OneHotEncoder`, `OrdinalEncoder`, `TfidfVectorizer` | Encode categoricals and text without train/serve skew |
| Reproducibility | `random_state`, pinned versions, saved split indices | Reproduce a result exactly |

### Common mistakes

- Fitting preprocessing on the full dataset before splitting.
- Using `pd.get_dummies` separately on training and serving data, giving mismatched columns.
- Forgetting `stratify` on imbalanced targets.
- Scaling features for tree models (harmless but pointless) while skipping it for a linear or distance model.
- Looking at the test set repeatedly while tuning, which turns it into a validation set.
- Not setting `random_state` and being unable to reproduce results.

### Self-check

- What do `fit`, `transform` and `fit_transform` each do, and on which data do you call them?
- Why does the order "scale, then split" leak information?
- Which model families need feature scaling and which do not?

---

## 8. Pipelines and ColumnTransformer

**In plain words:** A Pipeline links all steps, from cleaning to the model, into one object. A ColumnTransformer applies different steps to different columns. This keeps training and production the same and stops information leaking between data sets.

### Understand

**Pipeline.** A `Pipeline` chains transformers and a final estimator into one object with the same API. Calling `fit` fits each step in order on the training data, and `predict` applies the learned transforms and then the model. Benefits:
1. **No leakage**: preprocessing is fit only on whatever data `fit` receives, including inside cross-validation folds.
2. **One artefact to ship**: the same object preprocesses and predicts in production, which prevents training/serving skew.
3. **Tunable as a unit**: hyperparameters of any step are searched together.

**ColumnTransformer.** Applies different transformations to different columns, then concatenates the results. Numeric columns get impute-and-scale; categorical columns get impute-and-encode.

```python
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.ensemble import HistGradientBoostingClassifier

numeric = ["age", "amount", "days_since_signup"]
categorical = ["country", "plan"]

preprocess = ColumnTransformer([
    ("num", Pipeline([
        ("impute", SimpleImputer(strategy="median")),
        ("scale", StandardScaler()),
    ]), numeric),
    ("cat", Pipeline([
        ("impute", SimpleImputer(strategy="most_frequent")),
        ("onehot", OneHotEncoder(handle_unknown="ignore")),
    ]), categorical),
])

clf = Pipeline([
    ("prep", preprocess),
    ("model", HistGradientBoostingClassifier(random_state=0)),
])

clf.fit(X_train, y_train)
proba = clf.predict_proba(X_test)[:, 1]
```

Details to know:
- Columns not listed are dropped by default (`remainder="drop"`); use `remainder="passthrough"` to keep them.
- Columns are selected by name when `X` is a DataFrame, so **the serving input must have the same column names**. This is a contract to validate at the service boundary.
- Hyperparameters of nested steps are addressed with double underscores: `model__learning_rate`, `prep__num__impute__strategy`.
- `make_pipeline` and `make_column_selector` reduce boilerplate. `set_output(transform="pandas")` returns DataFrames from transformers and pipelines (added in scikit-learn 1.2, per the release notes; I could not confirm the version on the docs page itself).
- Custom steps: subclass `BaseEstimator` and `TransformerMixin`, or use `FunctionTransformer`. Keep them stateless or learn state only in `fit`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Chaining | `Pipeline`, `make_pipeline` | Build a pipeline from preprocessing to model from memory |
| Per-column logic | `ColumnTransformer`, `make_column_selector`, `remainder` | Route numeric, categorical and text columns correctly |
| Custom steps | `FunctionTransformer`, `BaseEstimator`, `TransformerMixin` | Add a feature step that is safe inside CV |
| Introspection | `named_steps`, `get_feature_names_out`, `set_output`, `pipeline.get_params()` | Map model coefficients back to feature names |
| Visualisation | `sklearn.set_config(display="diagram")` in Jupyter | Inspect pipeline structure |

### Common mistakes

- Doing feature engineering in Pandas before the split using statistics of the whole dataset (group means, target encodings), outside the pipeline.
- Serving a different column order or missing columns than training used.
- Putting the target into the features by accident.
- Using the model and the preprocessing as two separately saved files that drift out of sync. Save one pipeline.

### Self-check

- What three problems does a Pipeline solve?
- How do you set a hyperparameter of the second step of a nested pipeline?
- A column present at training is absent in a production request. Where should this fail, and how?

---

## 9. Common models and when to use them

**In plain words:** Many kinds of models exist. Start with a simple baseline (a very simple model used for comparison), then try a stronger one. Choose by data size, need to explain results, and speed.

### Understand

Choose by data size, data type, need for interpretability, and latency. Always begin with a **baseline** (a majority-class predictor or the mean, via `DummyClassifier` or `DummyRegressor`) so you know whether a model adds anything.

| Model | Good for | Strengths | Watch out for |
|---|---|---|---|
| Linear regression, Ridge, Lasso | numeric targets, interpretable relationships | fast, transparent, regularisation reduces overfitting | assumes linear effects; scale features; outliers |
| Logistic regression | binary and multiclass classification baselines | fast, calibrated-ish probabilities, coefficients readable | linear boundary; scale features; tune regularisation `C` |
| k-nearest neighbours | small data, simple baselines | no training | slow at prediction; needs scaling; poor in high dimensions |
| Decision tree | explainable rules | handles mixed scales, no scaling | overfits when deep |
| Random forest | strong general default on tabular data | robust, little tuning, handles non-linearity | larger and slower models; extrapolation is poor |
| Gradient boosting (`HistGradientBoosting*`; also XGBoost, LightGBM) | often the best accuracy on tabular data | native missing-value support in the histogram variants, fast on large data | more hyperparameters; overfits without early stopping |
| SVM | small to medium data, high dimensions | strong with a good kernel | poor scaling to large data; needs scaling |
| Naive Bayes | text classification baselines | very fast | independence assumption |
| k-means, DBSCAN | clustering | simple | requires scaling; k must be chosen; clusters may not mean anything |
| PCA | compression, visualisation | removes correlated dimensions | components are hard to interpret |

Practical guidance:
- **Tabular data**: start with logistic or linear regression as a baseline, then a gradient-boosted tree or random forest. This covers most business problems.
- **Text**: TF-IDF plus a linear model is a strong, cheap baseline before considering anything heavier.
- **Deep learning** is out of scope; scikit-learn does not target it.
- **Interpretability**: linear models and shallow trees; for others use feature importances, permutation importance (`sklearn.inspection.permutation_importance`), and partial dependence, understanding that importance is not causation.
- **Latency and size**: a linear model predicts in microseconds and is a few kilobytes; a large forest can be hundreds of megabytes and slower. Measure before committing.
- scikit-learn estimators train on one machine. Many accept `n_jobs` for parallelism via joblib.

Quality of the data usually matters more than the choice of model. Label noise, leakage and a wrong problem definition dominate algorithm choice.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Baselines | `DummyClassifier`, `DummyRegressor` | Always report a baseline next to a model |
| Linear models | `LinearRegression`, `Ridge`, `Lasso`, `LogisticRegression` | Tune regularisation and read coefficients |
| Tree ensembles | `RandomForestClassifier`, `HistGradientBoostingClassifier`, `HistGradientBoostingRegressor` | Train, tune and explain a boosted model |
| Clustering and reduction | `KMeans`, `DBSCAN`, `PCA` | Use for exploration and know their limits |
| Inspection | `feature_importances_`, `permutation_importance`, `PartialDependenceDisplay` | Explain a model's behaviour to a stakeholder |

### Common mistakes

- Reaching for a complex model before a baseline.
- Reading tree `feature_importances_` as causal.
- Using accuracy to select a model on an imbalanced target (chapter 10).
- Comparing models using different splits.
- Ignoring prediction latency and model size.

### Self-check

- Which model would you try first on a tabular classification task, and why?
- Why might a random forest fail to extrapolate beyond the training range?
- What does a baseline tell you?

---

## 10. Metrics

**In plain words:** A metric is a number that tells you how good a model is. Choose one that matches the real cost of mistakes. Accuracy can look good on rare-event data while the model is useless.

### Understand

A metric must match the business cost of errors. Compute metrics on held-out data, never on the training data.

**Classification.** Start from the **confusion matrix**: true positives (TP), false positives (FP), true negatives (TN), false negatives (FN).

| Metric | Definition | Use when |
|---|---|---|
| Accuracy | (TP + TN) / total | classes are balanced and errors cost the same; misleading otherwise |
| Precision | TP / (TP + FP) | false alarms are costly (flagging legitimate users) |
| Recall (sensitivity) | TP / (TP + FN) | misses are costly (missed fraud, missed disease) |
| F1 | harmonic mean of precision and recall | one number balancing the two |
| ROC AUC | probability a random positive scores above a random negative | ranking quality, roughly balanced classes |
| PR AUC (average precision) | area under precision-recall curve | heavy imbalance, positive class matters |
| Log loss | penalises confident wrong probabilities | probability quality matters |

Key ideas:
- **Accuracy trap.** With 1% positives, always predicting "negative" scores 99% accuracy and is useless.
- Most classifiers output a probability. The **decision threshold** (default 0.5 for `predict`) is a business choice. Move it using the precision-recall curve (`precision_recall_curve`) according to the cost of each error type.
- **Calibration**: whether a predicted 0.8 corresponds to about 80% observed. It matters when probabilities feed downstream decisions (`CalibratedClassifierCV`, `calibration_curve`).
- Multiclass averaging: `macro` (each class equal), `micro` (each sample equal), `weighted` (by support).

**Regression.**

| Metric | Meaning | Notes |
|---|---|---|
| MAE | mean absolute error | same units as target, robust to outliers |
| MSE / RMSE | squared error and its root | penalises large errors; RMSE in target units |
| R-squared | fraction of variance explained | can be negative on test data; says nothing about error size in business units |
| MAPE | percentage error | breaks near zero targets |

Report error in the target's units and compare with the baseline (predicting the mean or last value).

```python
from sklearn.metrics import classification_report, roc_auc_score, average_precision_score

proba = clf.predict_proba(X_test)[:, 1]
print(classification_report(y_test, proba >= 0.5))
print(roc_auc_score(y_test, proba), average_precision_score(y_test, proba))
```

Scoring strings used by cross-validation and search utilities (for example `"roc_auc"`, `"average_precision"`, `"f1"`, `"neg_mean_absolute_error"`) follow a "greater is better" convention, which is why error metrics appear negated. Confirm the exact names in the scoring documentation.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Classification | `confusion_matrix`, `classification_report`, `precision_recall_curve`, `roc_auc_score`, `average_precision_score`, `log_loss` | Choose a metric from the business cost |
| Thresholding | `precision_recall_curve`, `roc_curve`, manual threshold on `predict_proba` | Pick a threshold to meet a precision or recall target |
| Calibration | `calibration_curve`, `CalibratedClassifierCV` | Check whether probabilities are trustworthy |
| Regression | `mean_absolute_error`, `mean_squared_error`, `r2_score` | Report error in business units against a baseline |
| Display helpers | `ConfusionMatrixDisplay`, `RocCurveDisplay`, `PrecisionRecallDisplay` | Plot results for a review |

### Common mistakes

- Reporting accuracy on an imbalanced problem.
- Choosing a threshold on the test set.
- Evaluating on training data.
- Comparing ROC AUC across datasets with different positive rates as if it were a business metric.
- Reporting R-squared without error in real units.

### Self-check

- A fraud model has 99.5% accuracy at 0.5% fraud rate. Is it good? What would you check?
- Explain precision and recall in a spam filter.
- How do you choose a threshold for a model that must catch at least 90% of fraud?

---

## 11. Validation: cross-validation, leakage, fit and imbalance

**In plain words:** Validation checks that a model will work on data it has not seen. The biggest danger is leakage, where the model gets information it will not have in real use. Split data in a way that matches how the model will be used.

### Understand

**Cross-validation (CV).** One split gives a noisy estimate. **k-fold CV** splits the training data into k parts, trains on k-1 and validates on the remaining one, rotating k times, then averages. It uses data efficiently and reveals variance (the spread across folds).

```python
from sklearn.model_selection import StratifiedKFold, cross_val_score

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)
scores = cross_val_score(clf, X_train, y_train, cv=cv, scoring="average_precision")
print(scores.mean(), scores.std())
```

Choose the splitter to match the data's structure:
- **StratifiedKFold** for classification, preserving class ratios.
- **GroupKFold** when rows from the same entity (user, device, patient) must not appear in both train and validation. Otherwise the model memorises entities and scores are inflated.
- **TimeSeriesSplit** for temporal data, always training on the past and validating on the future.

**Data leakage.** Leakage is information in training that will not be available at prediction time, producing optimistic scores that collapse in production. It is the most common serious error. Forms:
1. **Preprocessing leakage**: fitting scalers, imputers or encoders on all data before splitting. Fix: use a Pipeline.
2. **Target leakage**: a feature derived from the target or recorded after the outcome (for example "refund issued" as a feature to predict fraud, or a `days_until_churn` column).
3. **Temporal leakage**: random splits on time-ordered data, so the model sees the future. Fix: split by time; build features using only data before the prediction timestamp ("point-in-time correctness").
4. **Group leakage**: the same entity in both train and test.
5. **Duplicate leakage**: duplicate rows across splits. Deduplicate before splitting.
6. **Tuning leakage**: choosing hyperparameters by test score. Fix: tune with CV on training data; score the test set once.

Warning signs: a near-perfect score on a hard problem, one feature dominating importance, and a large gap between offline and production metrics.

**Overfitting and underfitting.**
- **Underfitting (high bias)**: the model is too simple; training and validation errors are both high. Fix: richer features, a more flexible model, less regularisation.
- **Overfitting (high variance)**: the model memorises noise; training error is low and validation error is high. Fix: more data, simpler model, regularisation, feature selection, early stopping, ensembling.
- Diagnose with the gap between training and validation scores, and with `learning_curve` (score versus training size) and `validation_curve` (score versus a hyperparameter).

**Class imbalance.** When one class is rare (fraud, churn, defects):
- Use stratified splits and metrics that ignore the majority class (PR AUC, recall at a precision target, F1), not accuracy.
- Options: `class_weight="balanced"` (supported by many estimators), resampling (oversample the minority or undersample the majority, applied **inside training folds only**; the separate `imbalanced-learn` library provides samplers and a compatible pipeline), and **threshold tuning** on predicted probabilities, which is often the simplest effective step.
- Resampling changes the apparent class prior, so predicted probabilities become miscalibrated; recalibrate if probabilities are used.
- Always keep the validation and test sets at the **true** class distribution.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Cross-validation | `cross_val_score`, `cross_validate`, `cross_val_predict`, `StratifiedKFold`, `GroupKFold`, `TimeSeriesSplit` | Match splitter to data structure |
| Diagnostics | `learning_curve`, `validation_curve` | Distinguish over- and under-fitting from curves |
| Leakage control | `Pipeline`, point-in-time feature queries, `GroupKFold` | Audit a pipeline for the six leakage forms |
| Imbalance | `class_weight`, `sample_weight`, threshold selection, `imbalanced-learn` | Handle a 1% positive class end to end |
| Regularisation | `C`, `alpha`, `max_depth`, `min_samples_leaf`, early stopping | Tune model complexity |

### Common mistakes

- Preprocessing before the split.
- Random splits on time-ordered or entity-grouped data.
- Oversampling before cross-validation, so duplicated minority rows appear in both train and validation folds.
- Reporting the best CV score from a search as the expected production score.
- Trusting a very high score without hunting for leakage.

### Self-check

- List three kinds of leakage and how a pipeline prevents the first.
- Why is a random split wrong for predicting next week's churn from past behaviour?
- Training accuracy 99%, validation 70%. What is happening and what are three fixes?
- Why must resampling happen inside the cross-validation loop?

---

## 12. Hyperparameter search and model persistence

**In plain words:** Hyperparameters are settings you choose before training, and a search tries many values. After training, you save the model to a file. Load files only from sources you trust, and keep the library versions the same.

### Understand

**Hyperparameters versus parameters.** Parameters (coefficients, tree splits) are learned by `fit`. Hyperparameters (regularisation strength, tree depth, number of estimators, learning rate) are set before training and chosen by search against validation performance.

**Search strategies.**
- `GridSearchCV`: tries every combination. Exhaustive and expensive; fine for two or three small dimensions.
- `RandomizedSearchCV`: samples combinations from distributions for a fixed budget. Usually more efficient when only a few hyperparameters matter.
- Halving searches (`HalvingGridSearchCV`, `HalvingRandomSearchCV`) spend less on poor candidates; they are still experimental, and the API may change without a deprecation cycle. Enable them with `from sklearn.experimental import enable_halving_search_cv` first.
- Specialised libraries (for example Optuna) offer smarter search; the principle is the same.

```python
from sklearn.model_selection import RandomizedSearchCV

search = RandomizedSearchCV(
    clf,
    param_distributions={
        "model__learning_rate": [0.03, 0.05, 0.1, 0.2],
        "model__max_depth": [None, 3, 5, 8],
        "model__l2_regularization": [0.0, 0.1, 1.0],
    },
    n_iter=20,
    scoring="average_precision",
    cv=5,
    n_jobs=-1,
    random_state=0,
)
search.fit(X_train, y_train)
best = search.best_estimator_      # refit on all training data by default
```

Search runs inside the training data using CV, so the test set stays untouched. The cost is `n_candidates x n_folds` fits; budget it. `n_jobs` parallelises across fits using joblib; mind total memory when each worker holds a copy of the data.

**Model persistence.**
- scikit-learn models are Python objects. The standard approach is `joblib.dump(model, "model.joblib")` and `joblib.load(...)`, efficient for objects holding large NumPy arrays. `pickle` is the underlying mechanism.
- **Security:** loading a pickle or joblib file executes arbitrary code. Load only artefacts you produced or from a trusted, access-controlled store. Never load a model from user input or an untrusted bucket. Alternatives that avoid pickle are `skops.io` (loads only trusted types, but is slower and supports fewer types) and ONNX (the most secure option in the scikit-learn guide, but not every estimator converts, and you cannot rebuild the Python object). Serve ONNX in a sandbox as well.
- **Version compatibility:** scikit-learn documents no supported way to load a model trained with a different version. Loading under a different version may fail or silently misbehave. Record the library versions with the artefact, and ideally serve with the same pinned environment.
- Persist the **whole pipeline** (preprocessing plus model), so serving cannot drift from training.
- Record metadata alongside the artefact: training data version or query, date, feature names and dtypes, library versions, metrics, and a content hash. Store artefacts in object storage or a model registry (for example MLflow's registry) rather than in the application repository.

```python
import joblib

joblib.dump(search.best_estimator_, "churn_model_2024-06-01.joblib")
model = joblib.load("churn_model_2024-06-01.joblib")
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Search | `GridSearchCV`, `RandomizedSearchCV`, `param_grid`, `best_params_`, `cv_results_` | Run a search without touching the test set |
| Parallelism | `n_jobs`, `joblib.Parallel`, `joblib.delayed` | Parallelise safely and know the memory cost |
| Persistence | `joblib.dump`, `joblib.load`, `pickle` risks, `skops`, ONNX | Save a pipeline and state the security and version caveats |
| Tracking and registry | MLflow, DVC, object storage with content hashes | Tie a model to its data, code and metrics |

### Common mistakes

- Searching on the test set, or reporting the search's best CV score as the final estimate.
- Large grids with no budget, then a job that never finishes.
- Loading a model file from an untrusted source.
- Saving the model but not the preprocessing, or not recording library versions.
- Overwriting `model.joblib` in place, losing the ability to roll back.

### Self-check

- Why does a hyperparameter search use cross-validation on training data and not the test set?
- Why is `joblib.load` on an untrusted file a security risk?
- What metadata must travel with a persisted model?

---

## 13. matplotlib and the Jupyter workflow

**In plain words:** matplotlib draws charts, and Jupyter notebooks mix code and results. Notebooks are good for exploring but not for production. Move stable code into normal tested modules.

### Understand

**matplotlib.** The standard plotting library. Learn the **object-oriented interface**: a `Figure` contains one or more `Axes`, and you draw on axes. It is more predictable than the implicit `pyplot` state machine.

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(daily["date"], daily["orders"], label="orders")
ax.set_xlabel("Date")
ax.set_ylabel("Orders")
ax.set_title("Daily orders")
ax.legend()
fig.tight_layout()
fig.savefig("orders.png", dpi=150)
plt.close(fig)
```

Habits:
- Label axes with units, give a title, and choose the chart for the question: line for trends over time, bar for category comparison, histogram or box plot for distributions, scatter for relationships, heat map for matrices.
- On servers and in jobs, with no display, select a non-interactive backend (`matplotlib.use("Agg")` before importing pyplot, or the `MPLBACKEND` environment variable) and write files.
- `matplotlib` is not thread-safe in the global pyplot state; do not generate plots inside request handlers. Pre-render in a job.
- Always close figures in loops, or memory grows.
- `df.plot()` wraps matplotlib for quick looks. Seaborn and Plotly build on or alongside it for statistical and interactive charts.
- Useful diagnostic plots for models: confusion matrix, ROC and precision-recall curves, learning curves, residual plots, and feature importances.

**Jupyter workflow.** A notebook interleaves code, output and prose, which suits exploration, communication and teaching. Its weaknesses are the reasons it must not be production code:
- **Hidden state**: cells run out of order leave variables that no longer match the visible code. Before sharing, **restart the kernel and run all cells**.
- Poor version control: notebooks are JSON with outputs. Strip outputs before commit (`nbstripout`) or pair notebooks with plain scripts (`jupytext`), and review diffs accordingly.
- Hard to test and to parameterise.
- Secrets pasted into cells and committed. Load them from the environment.

A sound progression: explore in a notebook, then move stable logic into importable modules (functions with tests), keep the notebook as a thin report that imports them, and run reproducible notebooks in jobs with `papermill` (parameterised execution) or convert with `jupyter nbconvert`. Pin the environment with a lock file and use a dedicated kernel per project.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Plotting | `matplotlib` (`subplots`, `Axes` methods, `savefig`), `df.plot`, seaborn | Produce a labelled, saved figure with the object API |
| Headless rendering | `Agg` backend, `MPLBACKEND`, `plt.close` | Render charts in a scheduled job |
| Notebook hygiene | Restart and run all, `nbstripout`, `jupytext`, `%load_ext autoreload` | Keep notebooks reproducible and reviewable |
| Productionising | `nbconvert`, `papermill`, extracting modules, `pytest` | Move logic out of notebooks into tested code |
| Magics | `%timeit`, `%prun`, `%matplotlib inline`, `%who` | Profile and inspect interactively |

### Common mistakes

- Treating the notebook's final state as proof the code works.
- Committing large outputs, data or credentials.
- Creating figures in a loop without closing them.
- Using plots with no axis labels or units, or a chart type that hides the point (truncated bars, 3-D pies).

### Self-check

- Why must you "restart and run all" before trusting a notebook?
- How do you generate a chart in a cron job on a machine with no display?
- What goes in a notebook and what goes in a module?

---

## 14. Serving and scheduling: batch versus online

**In plain words:** A trained model helps only when it runs reliably. Batch means scoring many rows on a schedule; online means scoring one request at a time. Plan for versions, monitoring and rollback.

### Understand

A trained model creates value only when something runs it reliably. As a backend engineer this is the part you own.

**Batch inference.** A scheduled job reads many rows, scores them, and writes results to a table, file or cache.
- Fits when predictions are needed per entity at known times and need not be fresh to the second (nightly churn scores, weekly recommendations, a risk score refreshed hourly).
- Advantages: simple, cheap, high throughput, easy to retry, easy to validate before publishing, no latency constraint. Failures do not hit users directly.
- Costs: staleness, and compute spent on entities nobody requests.
- Implementation: a script or module run by cron, a systemd timer, or an orchestrator (Airflow, Prefect, Dagster, or a cloud scheduler). Needs idempotency (re-running a day gives the same output, writing with upsert or overwrite by partition), retries with backoff, overlap protection (a lock), alerting on failure or on abnormal output volume, and a record of which model version produced each row.

**Online inference.** A service scores a request on demand, typically behind an HTTP or gRPC API.
- Fits when features depend on the live request or must be fresh (fraud at checkout, search ranking).
- Constraints: latency budget, concurrency, availability, and the same features at serving as at training.
- Pattern: load the model **once at startup**, not per request; keep the process warm; validate the request against a schema; return the prediction with the model version.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pydantic import BaseModel
import joblib
import pandas as pd

state = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    state["model"] = joblib.load("models/churn/2024-06-01/model.joblib")
    yield

app = FastAPI(lifespan=lifespan)

class Features(BaseModel):
    age: float
    amount: float
    days_since_signup: float
    country: str
    plan: str

@app.post("/predict")
def predict(f: Features):
    row = pd.DataFrame([f.model_dump()])         # same column names as training
    p = float(state["model"].predict_proba(row)[0, 1])
    return {"score": p, "model_version": "2024-06-01"}
```

Operational concerns specific to model serving:
- **CPU-bound work blocks the event loop.** Declare the handler with plain `def` (FastAPI runs it in a thread pool) or use a worker pool; for heavy models use separate worker processes. Gunicorn or Uvicorn workers each hold a copy of the model, so memory scales with worker count.
- **Per-request overhead.** Building a one-row DataFrame costs milliseconds. For strict latency, convert to a NumPy array in a fixed feature order, or batch requests (micro-batching).
- **Thread safety.** `predict` on a fitted scikit-learn model is generally safe for concurrent reads, but confirm for the estimators you use, and never call `fit` concurrently on a shared object.
- **Input validation** at the boundary: types, ranges, allowed categories, missing values. Reject or default explicitly.
- **Timeouts, fallbacks and circuit breakers**: if the model or a feature source fails, return a safe default or a rule-based answer, not a 500.
- **Feature consistency**: the largest source of production bugs is **training/serving skew**, where features are computed differently offline and online. Share one feature code path, or use a feature store, and log served features to compare with training distributions.

**Hybrid.** Batch-compute expensive features or scores, store them in a key-value store or database, and have the online service look them up. This is common and often the best trade-off.

**Model versioning and rollout.**
- Version every artefact immutably (a path or registry version, never overwritten). Record training data snapshot, code commit, environment, metrics and approver.
- Deploy with the same discipline as code: staging, shadow mode (score in parallel, do not act on it), canary or A/B rollout, and fast rollback by pointing back to the previous version.
- Keep the model file separate from the service image where practical, so model and code release independently, but pin the compatible library versions.

**Monitoring.** Models degrade silently. Monitor:
- Service metrics: latency percentiles, error rate, throughput, memory.
- **Data drift**: the distribution of input features versus training (missing rates, ranges, category frequencies).
- **Prediction drift**: the distribution of scores.
- **Quality**: when ground truth arrives (often delayed), compute accuracy-type metrics over time and compare with the baseline.
- **Retraining**: scheduled or triggered by drift or quality decay, producing a new versioned artefact that passes the same evaluation gate.

**Testing.** Unit-test transformations with small fixtures. Add a schema check on inputs, a golden-file test (fixed inputs produce expected outputs for a given model version), and a smoke test that loads the artefact and scores a sample. Pin and test against the exact library versions used in production.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Batch scheduling | cron, systemd timers, `flock`, Airflow, Prefect, Dagster | Run an idempotent, non-overlapping, alerting job |
| Online serving | FastAPI or Flask with `joblib.load` at startup, Gunicorn/Uvicorn workers, `pydantic` schemas | Serve a pipeline with validation and a version field |
| Packaging | Docker, pinned `requirements`/lock files, `uv` or `pip-tools` | Build an image that reproduces the training environment |
| Versioning | MLflow tracking and registry, DVC, object storage with immutable paths | Trace a prediction to a model, data and code version |
| Interchange | ONNX and ONNX Runtime, where supported | Know when a non-Python runtime is warranted |
| Monitoring | Prometheus metrics, structured logs, drift checks, dashboards | Alert on latency, errors, input drift and output drift |
| Testing | `pytest`, `pandera` or `pydantic` for schemas, golden-file tests | Catch schema and behaviour regressions in CI |

### Common mistakes

- Loading the model on every request.
- Different feature logic in the training notebook and the service (skew).
- Unpinned dependencies, so an image rebuild changes predictions or breaks `joblib.load`.
- No model version in logs or responses, making incidents impossible to trace.
- Running CPU-heavy scoring on the async event loop.
- Batch jobs that are not idempotent, producing duplicates on retry.
- No monitoring beyond HTTP status codes.
- Overwriting the production artefact with no rollback path.

### Self-check

- When would you choose batch over online inference, and what is a hybrid?
- What is training/serving skew and how do you reduce it?
- How do you roll out and roll back a new model version safely?
- Which metrics would you alert on for a model in production?

---

## 15. Practice labs

**In plain words:** These labs are practice tasks. Do each one yourself before you look anything up. Each lab says what finished work looks like.

Do these on a laptop with a virtual environment. Use any public dataset (for example a public tabular dataset from the scikit-learn or UCI collections) or synthetic data. Each lab states the goal and what "done" looks like. Attempt before looking anything up.

1. **Memory model.** Create a 2-D array, take a slice, a transpose, a boolean mask and a fancy index. For each, show with `np.shares_memory` whether it is a view or a copy, and demonstrate a write that propagates and one that does not.
2. **Broadcasting and axes.** Standardise the columns of a matrix without loops. Then deliberately cause an accidental `(n, n)` broadcast and show how to detect it from the shape.
3. **Vectorise a loop.** Write a Python loop computing a distance or running statistic over a million rows, then the vectorised version. Time both with `%timeit` and explain the gap.
4. **Pandas profile.** Load a messy CSV with explicit `usecols` and `dtype`. Report dtypes, null counts, duplicates and memory use. Cut memory by at least 40% using categories and downcasting, and show the before and after.
5. **SettingWithCopy.** Reproduce a chained assignment that fails to update a frame, then fix it with `loc`. State what your Pandas version does and cite its release notes.
6. **Join audit.** Join an orders table to a customers table that contains a duplicated key. Show the row inflation and the wrong revenue total, then add `validate=` and `indicator=` to catch it.
7. **Group features.** For an events table, compute per-user count, last event, share of user total using `transform`, and the previous event time using `shift`. Reproduce the same result in SQL and compare.
8. **Time handling.** Parse timestamps with mixed time zones, normalise to UTC, resample to daily and weekly counts, and explain the effect of a daylight-saving transition on one day.
9. **Parquet versus CSV.** Write the same data to both, compare file size, load time and dtype fidelity (timestamps, categoricals, nullable integers), then read only two columns from Parquet.
10. **Chunked aggregation.** Aggregate a CSV larger than a chosen memory cap using `chunksize`, then repeat it with SQL or DuckDB and compare.
11. **Leak on purpose.** Train a model with the scaler fitted before the split and another with a Pipeline. Then add a target-leaking feature and observe the inflated score. Write down the signs that would have alerted you.
12. **End-to-end pipeline.** Build a `ColumnTransformer` plus gradient-boosting `Pipeline` for a classification task with numeric, categorical and missing values. Compare against a `DummyClassifier` using cross-validation with a metric suited to the class balance.
13. **Imbalance.** Create a 1%-positive dataset. Compare accuracy, PR AUC and recall at fixed precision for the baseline, `class_weight="balanced"` and a tuned threshold. Explain which you would deploy.
14. **Time-aware split.** Build a dataset with a time trend and compare a random split to `TimeSeriesSplit`. Explain why the random result is optimistic.
15. **Persist and break.** Save a pipeline with `joblib`, store its metadata (versions, columns, metrics, hash), reload it in a fresh environment and then in one with a different scikit-learn version. Record what happens.
16. **Serve it.** Wrap the pipeline in a small HTTP service with schema validation, a startup-loaded model and a version field. Load-test it, observe latency and memory per worker, and add a request that sends a missing field.
17. **Schedule it.** Turn a scoring script into an idempotent batch job run by cron or a systemd timer with a lock, logging, a non-zero exit on failure and a row-count sanity check. Run it twice and prove no duplicates.
18. **Drift check.** Compare the feature distributions of a training sample and a deliberately shifted "production" sample using summary statistics and a simple statistical test or population-stability measure. Write the alert rule you would deploy.

---

## 16. Interview questions

In plain words: this chapter lists practice questions. Each has a short model answer at the end, with the same number. Each question carries a level tag: [Fresher], [Mid] or [Senior]. Questions marked (B) are common for backend roles; unmarked ones suit data-heavy or ML-platform roles. Practice answers do not guarantee any hiring result.

### Top 10 must-know

| # | Level | Question (full answer in the model answers below) | Key point |
|---|---|---|---|
| 2 | [Fresher] | View versus copy in NumPy | Slices share memory; masks and fancy indexing copy |
| 6 | [Fresher] | `loc` versus `iloc` | Label (end inclusive) versus position (end exclusive) |
| 12 | [Fresher] | `fit`, `predict`, `transform` | Learn state, apply the model, apply learned preprocessing |
| 15 | [Mid] | Left join returns extra rows | Duplicate right keys; use `validate=` and assert row counts |
| 16 | [Mid] | SettingWithCopyWarning | Use one `loc` call; Copy-on-Write is the 3.0 default |
| 21 | [Mid] | Time zones | Store and compute in UTC; convert for display |
| 23 | [Mid] | Data leakage | Information from outside training; fix with a Pipeline and correct splits |
| 24 | [Mid] | Accuracy on imbalanced data | Use PR AUC, recall at a precision target, and a tuned threshold |
| 29 | [Mid] | Saving a model safely | `joblib` the whole pipeline; trusted files only; pin versions |
| 31 | [Senior] | Serving a model online | Load once, validate input, avoid skew, monitor, version, roll back |

### Fresher

1. [Fresher] (B) Why is NumPy faster than a Python list for numeric work?
2. [Fresher] (B) What is the difference between a view and a copy in NumPy?
3. [Fresher] What is broadcasting? Give an example.
4. [Fresher] What does `axis=0` mean in `a.sum(axis=0)`?
5. [Fresher] (B) What is the difference between a Series and a DataFrame?
6. [Fresher] (B) What is the difference between `loc` and `iloc`?
7. [Fresher] How do you filter a DataFrame on two conditions?
8. [Fresher] (B) How do you find and handle missing values in Pandas?
9. [Fresher] (B) What is the difference between `inner`, `left` and `outer` joins in `merge`?
10. [Fresher] What does `groupby(...).agg(...)` return?
11. [Fresher] What is a train/test split and why is it needed?
12. [Fresher] (B) What do `fit`, `predict` and `transform` do in scikit-learn?
13. [Fresher] What is overfitting?
14. [Fresher] What is the difference between classification and regression?

### Mid-level

15. [Mid] (B) A left join returns more rows than the left table. Why, and how do you prevent it?
16. [Mid] (B) What is SettingWithCopyWarning and how do you avoid the underlying problem?
17. [Mid] What is the difference between `agg` and `transform` in groupby?
18. [Mid] (B) Why prefer Parquet to CSV?
19. [Mid] (B) How would you process a CSV that does not fit in memory?
20. [Mid] (B) Why is `df.apply(axis=1)` slow and what do you do instead?
21. [Mid] (B) How do you handle time zones when working with timestamps?
22. [Mid] What is a Pipeline and why use one?
23. [Mid] (B) What is data leakage? Give three examples.
24. [Mid] Why is accuracy a poor metric for an imbalanced problem, and what do you use?
25. [Mid] Explain precision and recall and how to choose a threshold.
26. [Mid] What is cross-validation and when do you use `GroupKFold` or `TimeSeriesSplit`?
27. [Mid] How do you handle categorical features in scikit-learn?
28. [Mid] How do you tune hyperparameters without contaminating the test set?
29. [Mid] (B) How do you save and load a trained scikit-learn model, and what are the risks?
30. [Mid] (B) When is Pandas the wrong tool?

### Senior

31. [Senior] (B) Design an online service that serves a trained model. What are the main concerns?
32. [Senior] (B) Batch versus online inference: how do you choose?
33. [Senior] (B) What is training/serving skew and how do you prevent it?
34. [Senior] (B) How would you version, roll out and roll back models?
35. [Senior] How would you monitor a deployed model?
36. [Senior] (B) A model scored well offline but performs poorly in production. How do you investigate?
37. [Senior] How would you build a point-in-time-correct training set from event data?
38. [Senior] (B) How would you make a nightly scoring job reliable?
39. [Senior] You have a 1% positive class and a business that wants to catch most positives. Walk through your approach.
40. [Senior] (B) How do you decide whether to build with Pandas, SQL or a distributed engine?
41. (B) What would you review in a data scientist's notebook before it goes to production?
42. How do you reproduce a result from six months ago?

### Model answers (condensed)

**1.** NumPy stores same-typed values in one contiguous block and runs compiled loops over it. A list holds pointers to boxed Python objects with per-element dispatch overhead and poor cache locality.

**2.** A view shares memory with the original, so writes propagate. A copy has its own memory. Basic slicing gives views; boolean masks and fancy indexing give copies. Check with `np.shares_memory` or `.base`.

**3.** Combining arrays of different shapes by aligning shapes from the right and stretching dimensions of size 1. Example: a `(3, 1)` array plus a `(4,)` array gives `(3, 4)`. Subtracting column means of shape `(n_features,)` from `(n_samples, n_features)` is the common use.

**4.** Collapse axis 0, so rows are combined and you get one value per column.

**5.** A Series is a labelled one-dimensional array. A DataFrame is a table of Series (columns) sharing an index, each with its own dtype.

**6.** `loc` selects by label with inclusive slice ends. `iloc` selects by integer position with exclusive ends.

**7.** Combine boolean masks with `&` and parentheses: `df.loc[(df.a > 1) & (df.b == "x")]`. Do not use `and`.

**8.** `df.isna().sum()` to find them. Then drop (`dropna`), fill (`fillna`, forward fill), interpolate or impute inside a model pipeline, after deciding why the data is missing. Do not fill with a value that has real meaning.

**9.** `inner` keeps matching keys only. `left` keeps all left rows with nulls where there is no match. `outer` keeps all rows from both sides.

**10.** One row per group, with the aggregated columns, indexed by the group keys unless `as_index=False` or `reset_index()` is used.

**11.** Hold out data the model never trains on, to estimate performance on unseen data. Without it you measure memorisation.

**12.** `fit` learns from data and stores learned state. `predict` produces outputs for new inputs from that state. `transform` applies learned preprocessing to data (learned in `fit`).

**13.** The model fits noise in the training data, scoring well on it and poorly on unseen data. Fix with more data, simpler models, regularisation, or early stopping.

**14.** Classification predicts a category. Regression predicts a continuous number.

**15.** The right table has duplicate keys, so each left row matches several. Prevent with `validate="many_to_one"`, deduplicate the right side, and assert row counts after the join. `indicator=True` helps audit it.

**16.** It flags assignment on something that may be a copy of a slice (chained indexing like `df[mask]["c"] = v`), so the write may not reach the original. Use a single `df.loc[mask, "c"] = v`, or `.copy()` for an independent subset. Copy-on-Write removes the ambiguity: it is the default and only mode in pandas 3.0, and in 2.x it must be enabled with `pd.options.mode.copy_on_write = True`.

**17.** `agg` reduces each group to one row. `transform` returns a result with the same length and index as the input, such as the group mean on every row, so it can be assigned back as a column.

**18.** Typed, compressed, columnar. It preserves dtypes and time zones, reads only needed columns, supports filtering and partitioning, and is smaller and faster. CSV is untyped text that must be parsed and can be ambiguous.

**19.** Read only needed columns with explicit dtypes; use `chunksize` and aggregate incrementally; convert to Parquet; push the work to a database or use an out-of-core engine such as DuckDB or Polars; or use a distributed engine if truly large.

**20.** It calls a Python function per row, losing vectorisation. Use column arithmetic, `np.where`, `np.select`, `map`, or built-in groupby and string operations.

**21.** Parse to timezone-aware timestamps, store and compute in UTC, convert to local time only for display. Never mix naive and aware values, and remember DST makes some local times ambiguous or missing.

**22.** A Pipeline chains preprocessing and a model into one object. It prevents leakage (steps fit only on training data, including within CV folds), ships preprocessing with the model to avoid skew, and lets you tune all steps together.

**23.** Information in training that will not exist at prediction time. Examples: fitting a scaler or encoder on all data before splitting; a feature recorded after the outcome (a refund flag when predicting fraud); a random split of time-ordered data; the same user in both train and test; duplicates across splits.

**24.** A model predicting the majority class gets high accuracy and is useless. Use precision, recall, F1, PR AUC or recall at a fixed precision, with stratified splits and a tuned threshold.

**25.** Precision is the share of predicted positives that are correct. Recall is the share of actual positives found. Pick the threshold from the precision-recall curve according to the relative cost of false positives and false negatives, on validation data.

**26.** It averages performance over several train/validation splits for a more stable estimate. `GroupKFold` keeps all rows of an entity in one fold. `TimeSeriesSplit` always trains on earlier data and validates on later data.

**27.** Encode inside a `ColumnTransformer`: `OneHotEncoder(handle_unknown="ignore")` for low cardinality, ordinal encoding for ordered categories, and hashing, frequency or carefully cross-validated target encoding for high cardinality. Tree boosters in some libraries handle categories natively.

**28.** Search with cross-validation on the training set (`GridSearchCV` or `RandomizedSearchCV`), choose the best, and evaluate once on the untouched test set.

**29.** `joblib.dump` and `joblib.load` of the whole pipeline. Risks: loading executes arbitrary code, so only load trusted artefacts; and artefacts depend on library versions, so record and pin them. Safer formats or ONNX can be alternatives.

**30.** When SQL in the database suffices, when data exceeds memory, for per-request low-latency logic, for concurrent transactional state, and for distributed processing needs.

**31.** Load the model once at startup; validate inputs against a schema; ensure identical feature logic to training; keep the CPU-bound call off the event loop; size workers for memory (each holds a copy); set timeouts and a fallback; return and log the model version; expose latency and error metrics; support rollback.

**32.** Batch when predictions can be precomputed and slightly stale: simpler, cheaper, easy to validate and retry. Online when features depend on the live request or must be fresh and a latency budget can be met. Hybrid: precompute heavy features or scores and look them up online.

**33.** Features computed differently offline and online, so the model sees different data in production. Prevent with one shared feature code path or a feature store, a single persisted pipeline, schema validation, and logging served features to compare with training distributions.

**34.** Immutable versioned artefacts with data, code, environment and metrics recorded; an evaluation gate; staging, then shadow or canary rollout; rollback by repointing to the previous version; the version logged with every prediction.

**35.** Service health (latency, errors, memory), input drift (missing rates, ranges, category frequencies), output drift (score distribution), and delayed quality metrics when labels arrive, with alert thresholds and a retraining trigger.

**36.** Check for leakage in the offline evaluation, training/serving skew (compare served and training feature distributions), data drift, label delay and definition mismatches, a different population, threshold or calibration changes, and a stale or wrong model version.

**37.** For each training example, compute features using only data with timestamps before that example's prediction time, with a time-based split. Use as-of joins (`merge_asof` or SQL window logic), and never use aggregates that include future events.

**38.** Idempotent writes (overwrite or upsert by partition), retries with backoff, a lock against overlap, a row-count and null-rate sanity check before publishing, alerts on failure and on abnormal volume, versioned model reference, and a run log. Write to a staging location and swap in only after checks pass.

**39.** Stratified splits; a baseline; PR AUC and recall at a precision target as metrics; class weights or resampling inside folds; threshold tuning on validation data to hit the recall goal; calibration if probabilities are used; check leakage; monitor the positive rate and quality in production.

**40.** SQL when data is in a database and the task is a filter, join or aggregate. Pandas for in-memory work on data that fits comfortably and needs flexible transformation. DuckDB or Polars for larger single-machine analytics. A distributed engine when data and compute exceed one machine. Choose by data size, team skill, latency and operational cost.

**41.** Restart and run all; leakage; split strategy; hard-coded paths and secrets; unpinned versions; seeds; data assumptions (nulls, key uniqueness, time zones); metric choice against a baseline; logic that should become tested modules; the whole pipeline being persistable and servable with the same columns.

**42.** Pin the environment (lock file or image), version the code (commit), version or snapshot the data (immutable dataset path or a query with a fixed as-of time), fix random seeds and save split indices, and log parameters, metrics and the artefact in a tracking system.

---

## 17. References

Primary sources, preferred over blog posts:

- NumPy documentation, including the user guide on broadcasting and indexing: https://numpy.org/doc/stable/
- pandas documentation, user guide and release notes (read the notes for Copy-on-Write and deprecations): https://pandas.pydata.org/docs/
- pandas user guide on merging, groupby and time series: https://pandas.pydata.org/docs/user_guide/index.html
- scikit-learn user guide (estimator API, pipelines, model selection, metrics, model persistence): https://scikit-learn.org/stable/user_guide.html
- scikit-learn common pitfalls and recommended practices (leakage, randomness): https://scikit-learn.org/stable/common_pitfalls.html
- scikit-learn model persistence and security notes: https://scikit-learn.org/stable/model_persistence.html
- matplotlib documentation and usage guide: https://matplotlib.org/stable/
- Project Jupyter documentation: https://docs.jupyter.org/
- Apache Arrow and pyarrow documentation: https://arrow.apache.org/docs/python/
- Apache Parquet documentation: https://parquet.apache.org/docs/
- joblib documentation: https://joblib.readthedocs.io/
- SQLAlchemy documentation (engines and connections used by `read_sql`): https://www.sqlalchemy.org/docs/stable/
- DuckDB documentation: https://duckdb.org/docs/2.0/
- Polars documentation: https://docs.pola.rs/
- MLflow documentation (tracking and model registry): https://mlflow.org/docs/latest/
- FastAPI documentation (lifespan events, dependencies): https://fastapi.tiangolo.com/
- Wes McKinney, *Python for Data Analysis* (book)
- Aurelien Geron, *Hands-On Machine Learning with Scikit-Learn, Keras and TensorFlow* (book)
- Chip Huyen, *Designing Machine Learning Systems* (book)
- D. Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015)

URLs were checked against the live sites on 2026-11-06. Pages move, so verify a URL before depending on it.

---

## 18. Glossary

| Term | Plain meaning |
|---|---|
| ndarray | NumPy's main object: a block of same-type numbers with a shape |
| dtype | The data type of every element (for example `int64`, `float32`) |
| strides | How many bytes to step in memory to reach the next item along each axis |
| view | A new name for the same memory; a change through it changes the original |
| copy | A new, separate block of memory; changes do not reach the original |
| broadcasting | A rule that stretches a smaller array so two shapes can be combined |
| vectorised | Done on a whole array by compiled code, not by a Python loop |
| index (Pandas) | The labels of the rows; Pandas matches data by these labels |
| alignment | Matching values by label before doing arithmetic |
| Copy-on-Write | A Pandas rule: a derived object acts as a separate copy, and the real copy is made only when you change it |
| groupby | Split rows into groups by a key, compute per group, join the results |
| transform | A groupby result with the same length as the input |
| many-to-many join | A join where the key repeats on both sides, so rows multiply |
| nullable dtype | A type that can hold a missing value without changing to float |
| categorical | A column stored as small integer codes plus a list of allowed values |
| Parquet | A compressed, typed file format that stores data by column |
| out-of-core | Processing data that is larger than memory by reading it in parts |
| estimator | A scikit-learn object with `fit` |
| hyperparameter | A setting you choose before training; it is not learned |
| Pipeline | One object that chains preprocessing steps and a model |
| data leakage | Training on information that will not exist at prediction time |
| cross-validation | Training and testing several times on different splits, then averaging |
| class imbalance | One class is much rarer than the other |
| precision / recall | Share of predicted positives that are right / share of real positives found |
| calibration | Whether a predicted probability matches how often the event really happens |
| overfitting | A model that fits noise in training data and does badly on new data |
| training/serving skew | Features computed differently in training and in production |
| idempotent | Running the job twice gives the same result as running it once |
| data drift | Production inputs change compared with the training data |
| pickle | Python's object-saving format; loading an untrusted file can run harmful code |
