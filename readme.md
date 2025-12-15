# Bikes DB (SQLite) — contents & usage

This repository contains the extracted motorcycle data packaged into an SQLite database file `bikes.db` .

If you only need to publish a single artifact, publish `bikes.db` and this README — the DB contains the brands and models and everything needed to query the dataset.

Database structure

The SQLite database `bikes.db` contains two tables:

- `brands` (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT UNIQUE)
- `models` (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, years TEXT, brand_id INTEGER REFERENCES brands(id))

Notes about `years`:

- The `years` column contains free-form text extracted from the HTML such as `(2004 - now)` or `(1990 - 1991)`. For simple filtering you can use text matches; for robust numeric range queries parse the text in application code (example below).

Example SQL queries

- List all models for a given brand (case-insensitive):

```sql
SELECT m.name, m.years
FROM models m
JOIN brands b ON m.brand_id = b.id
WHERE lower(b.name) = lower('HONDA')
ORDER BY m.name;
```

- List models that mention a specific year (simple text match):

```sql
SELECT b.name AS brand, m.name, m.years
FROM models m
JOIN brands b ON m.brand_id = b.id
WHERE m.years LIKE '%2017%'
ORDER BY b.name, m.name;
```

Python example: find models active during a specific year

```python
import re
import sqlite3

YEAR_RE = re.compile(r"\((\d{4})\s*-\s*(\d{4}|now)\)")

def models_active_in(year, db_path='bikes.db'):
	year = int(year)
	conn = sqlite3.connect(db_path)
	cur = conn.cursor()
	rows = cur.execute('SELECT m.name, m.years, b.name FROM models m JOIN brands b ON m.brand_id=b.id').fetchall()
	out = []
	for name, years, brand in rows:
		if not years:
			continue
		m = YEAR_RE.search(years)
		if not m:
			# fallback: match the year textually
			if str(year) in years:
				out.append((brand, name, years))
			continue
		start = int(m.group(1))
		end = m.group(2)
		end = 9999 if end == 'now' else int(end)
		if start <= year <= end:
			out.append((brand, name, years))
	conn.close()
	return out

for brand, name, years in models_active_in(2017):
	print(f"{brand} — {name} {years}")
```

Using the sqlite3 CLI

If you have the `sqlite3` command-line client installed you can run queries directly:

```powershell
# interactive
sqlite3 bikes.db
# one-off query with pretty headers
sqlite3 -header -column bikes.db "SELECT b.name AS brand, m.name, m.years FROM models m JOIN brands b ON m.brand_id=b.id WHERE lower(b.name)='honda';"
```

Tips & next steps

- The `years` column is free-form: if you expect many queries filtered by numeric year I can add normalized `year_from` / `year_to` integer columns and populate them (I can implement and run the migration for you).
- You can open the file with GUI tools (DB Browser for SQLite) for exploration and export to CSV/JSON.

If you'd like I can add a small `scripts/query_examples.py` with some common queries and a `scripts/normalize_years.py` migration that adds `year_from`/`year_to` and backfills them.

License

The SQLite database file `bikes.db` included in this repository is provided under the MIT License. See the `LICENSE` file for the full license text. Replace the placeholder copyright holder in `LICENSE` with the appropriate name if needed.

