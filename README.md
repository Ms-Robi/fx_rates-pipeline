# FX Rates Pipeline

A Python pipeline that cleans a messy monthly exchange-rate file and loads it into a SQLite database. You can re-run it safely and it won't create duplicates.

> All data in this repo is **fictional**. It was made for practice and is not real exchange-rate data.

## What it does

```
fx_rates_long.csv  ->  read  ->  parse currency  ->  clean rates  ->  drop duplicates  ->  fx_rates_new.csv
                                                                                      \->  fx.db (table: fx_rates)
```

The input has 341 rows with the columns `month`, `currency` and `rate`. The pipeline is five generator functions chained together, followed by a database load:

| Function | Job |
|---|---|
| `read_rates` | Reads the CSV one row at a time |
| `parse_currencies` | Splits a label like `100 Japanese Yen` into a unit (100) and a name (Japanese Yen) |
| `clean_rates` | Drops rows whose rate is missing, negative or zero |
| `drop_duplicates` | Keeps one row per month and currency |
| `write_csv` | Writes the clean rows to `fx_rates_new.csv` |

After that, the rows are loaded into `fx.db`. The table has a primary key on `(month, currency_name)`, and rows are inserted with `INSERT OR REPLACE`.

## How the cleaning works

Each rule runs in this order, and each rule is there for a reason:

| Order | Rule | Why |
|---|---|---|
| 1 | Split the currency label into a unit and a name | Some currencies are quoted per 100 units (e.g. `100 Japanese Yen`), so rates are not comparable until you divide by the unit (`rate_per_unit`) |
| 2 | Skip rows with a missing or non-numeric rate | A missing rate is not zero, so I don't guess it |
| 3 | Skip rows with a negative or zero rate | An exchange rate can't be negative or zero, and I don't "fix" it with `abs()` because that would invent data |
| 4 | Drop duplicate month and currency rows | Each currency should appear once per month |

**Row counts reconcile:** 341 - 25 skipped - 4 negative or zero - 5 duplicates = **307 rows**.

## How to run it

**Requirements:** Python 3.10 or newer. It uses only the standard library (`csv`, `re`, `sqlite3`), so there is nothing to install.

1. Put `fx_pipeline.py` and `fx_rates_long.csv` in the same folder.
2. Run:
   ```
   python fx_pipeline.py
   ```
3. You should see these numbers:

   | Check | Expected |
   |---|---|
   | Rows in `fx_rates` table | 307 |
   | Skipped (missing or invalid) | 25 |
   | Negative or zero rates | 4 |
   | Duplicates removed | 5 |

4. Run it a second time. The numbers are the same and there are no duplicate rows in the table, because the load is idempotent.

It creates `fx_rates_new.csv` and `fx.db`. These are generated files, so they are not in the repo.

## Example query

Latest rate per currency:

```sql
SELECT currency_name, month, rate_per_unit
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY currency_name ORDER BY month DESC) AS rn
  FROM fx_rates
)
WHERE rn = 1;
```

