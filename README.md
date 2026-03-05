
```python
--- Report ---
  Snapshots: 807
  Total rows: 819100
  Date: 2026-01-23 01:00:00 - 2026-03-05 19:00:00
  Columns: ['date', 'instrument_name', 'expiry_name', 'expiry_date', 'strike', 'type', 'volume', 'open_interest', 'rho', 'theta', 'vega', 'gamma', 'delta', 'bid_iv', 'ask_iv', 'mark_iv']
  CSV: full_data.csv (116.1 MB)
  PARQUET: full_data.parquet (17.3 MB)
  Samples: full_data_sample.csv, sample.csv
```

## Reading Parquet files with Pandas and Polars

### Pandas

```python
import pandas as pd

# Basic reading
df = pd.read_parquet("file.parquet")

df = pd.read_parquet("file.parquet", engine="pyarrow")

print(df.head())
print(df.dtypes)
```

---

### Polars

```python
import polars as pl

# Basic reading
df = pl.read_parquet("file.parquet")

# Reading only the main columns
df = pl.read_parquet("file.parquet")

print(df.head())
print(df.schema)
```

---

### Examples

#### Lazy reading with Polars (recommended for large files)

```python
import polars as pl

# Filtering only CALL options with volume > 0, for a specific instrument
df = (
    pl.scan_parquet("file.parquet")
      .filter(
          (pl.col("type") == "C") &           # only CALL options
          (pl.col("volume") > 0) &            # with volume
          (pl.col("date") >= "2024-01-01")    # from a specific date
      )
      .select([
          "date",
          "instrument_name",
          "expiry_date",
          "strike",
          "type",
          "volume",
          "open_interest",
          "delta",
          "mark_iv"
      ])
      .sort("date")
      .collect()
)
```

#### Practical examples using the Greeks 

```python
import polars as pl

df = pl.scan_parquet("file.parquet")

# Options with high gamma and significant volume
high_gamma = (
    df.filter(
        (pl.col("gamma") > 0.05) &
        (pl.col("volume") > 100)
    )
    .select(["date", "instrument_name", "strike", "type", "gamma", "delta", "volume"])
    .collect()
)

# IV spread (ask_iv - bid_iv) per instrument
iv_spread = (
    df.select([
        "date",
        "instrument_name",
        "strike",
        "bid_iv",
        "ask_iv",
        (pl.col("ask_iv") - pl.col("bid_iv")).alias("iv_spread")
    ])
    .sort("iv_spread", descending=True)
    .collect()
)
```

The lazy API is especially useful here since options data can get very large — filtering by `date`, `type`, or `volume` before loading everything into memory makes a big difference in performance.