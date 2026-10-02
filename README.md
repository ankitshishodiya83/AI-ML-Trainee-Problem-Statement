# AI-ML-Trainee-Problem-Statement
# Problem Statement 1: Most Complex Code

This document answers two questions from the assignment:
1. Link to the most complex **Python code** written
2. Link to the most complex **database code** written

Both answers point to the same notebook: **Books API to SQLite**
(`01_books_api_to_sqlite.ipynb`), because it has the fullest pipeline
(API call, cleaning, database, display) and the most advanced
database logic of the three tasks done for this assignment.

---

## 1. Most complex Python code

**File:** `01_books_api_to_sqlite.ipynb`

**Why this one is the most complex:**
- Calls an external REST API (Open Library) with query parameters and a timeout
- Handles network errors and bad JSON with `try` / `except`
- Inspects raw API data before using it, since the author field comes as a list
  and some fields can be missing
- Cleans every record: skips unusable rows, joins multiple authors into one
  string, and falls back to "Unknown" or empty values
- Stores the cleaned data in SQLite and reads it back with pandas to display it

Compared to the other two notebooks:
- The **student scores notebook** fetches data, calculates averages, and plots
  a chart. It has no database.
- The **CSV to SQLite notebook** reads a local file, so there is no network or
  JSON handling.

```python
# Fetch
response = requests.get(URL, params=params, timeout=15)
response.raise_for_status()
data = response.json()
books = data["docs"]

# Clean
clean_books = []
for b in books:
    title = (b.get("title") or "").strip()
    key = (b.get("key") or "").strip()
    if not title or not key:
        continue
    authors = b.get("author_name") or []
    author = ", ".join(authors[:3]) if authors else "Unknown"
    year = b.get("first_publish_year")
    clean_books.append((key, title, author, year))
```

---

## 2. Most complex database code

**File:** `01_books_api_to_sqlite.ipynb` (table creation and insert cells)

**Why this is the most complex database code:**
- Uses a `UNIQUE` constraint on the API's own ID (`source_key`) to prevent
  duplicate rows
- Uses an **upsert** (`ON CONFLICT ... DO UPDATE`) instead of a plain insert,
  so re-running the notebook updates existing rows instead of duplicating them
- Inserts all rows in one `executemany` call, committed as a single transaction
- Uses `?` placeholders in every query, which protects against SQL injection
- Reads the data back from the database (not from the API) to prove it was
  stored correctly, using `pandas.read_sql_query`

Compared to the other two notebooks:
- The **CSV to SQLite notebook** uses a simpler `INSERT OR IGNORE`, which only
  skips duplicates and cannot update an existing row.
- The **student scores notebook** has no database code.

```python
# Table with a UNIQUE key
cursor.execute("""
    CREATE TABLE IF NOT EXISTS books (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        source_key TEXT NOT NULL UNIQUE,
        title      TEXT NOT NULL,
        author     TEXT NOT NULL,
        pub_year   INTEGER
    )
""")
conn.commit()

# Upsert: insert new rows, update existing ones, no duplicates
cursor.executemany("""
    INSERT INTO books (source_key, title, author, pub_year)
    VALUES (?, ?, ?, ?)
    ON CONFLICT(source_key) DO UPDATE SET
        title    = excluded.title,
        author   = excluded.author,
        pub_year = excluded.pub_year
""", clean_books)
conn.commit()

# Read back from the database to display it
df = pd.read_sql_query(
    "SELECT title, author, pub_year FROM books ORDER BY pub_year, title",
    conn,
)
```

---

## Summary

| Question | Answer |
|---|---|
| Most complex Python code | `01_books_api_to_sqlite.ipynb` |
| Most complex database code | `01_books_api_to_sqlite.ipynb` (table, upsert and read-back cells) |
