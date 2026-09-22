# 02 — Data Gathering

**Playlist videos 15–18.** Step 2 of the MLDLC (Section 01.8) is getting data into your hands in
the first place. This section is deliberately more practical than theoretical — the "theory" here
is really a set of honest facts about each data source: what shape the data comes in, what the
tool actually does under the hood, and where it breaks. Four sources, in increasing order of how
much stands between you and clean rows: a file already on disk, a database or nested document, a
service that hands you data on request, and a web page that was never meant to hand you data at
all.

---

## 1. Working with CSV Files

CSV (comma-separated values) is the default interchange format for tabular data — plain text, one
row per line, columns separated by a delimiter. Its appeal is simplicity: no schema, no special
software, human-readable. Its weaknesses come from that same simplicity: no enforced types (every
cell is text until something parses it), no standard for encoding, and the delimiter itself can
appear inside a field (handled by quoting, which is a common source of parsing bugs).

In pandas, `read_csv` is the workhorse, and its parameters exist mostly to compensate for exactly
those weaknesses:

```python
import pandas as pd

df = pd.read_csv(
    "data.csv",
    sep=",",              # delimiter — override for tab/pipe-separated files (sep="\t")
    header=0,              # row index to use as column names; None if there's no header row
    index_col=0,           # column to use as the DataFrame index instead of a fresh RangeIndex
    usecols=["id", "amt"], # load only these columns — cheaper than loading all and dropping
    dtype={"id": "int32"}, # force a column's type instead of letting pandas infer it
    na_values=["NA", "?"], # extra strings that should be treated as missing/NaN
    nrows=1000,            # read only the first N rows — useful to preview a huge file fast
    encoding="utf-8",      # character encoding; non-UTF-8 files raise UnicodeDecodeError
)
```

A few of these matter more than they look:

- **`dtype` and inference cost.** By default pandas scans the data to infer each column's type.
  For large files this is slow and sometimes wrong (a column of IDs that happens to be all digits
  gets read as `int64`, silently dropping a leading zero). Specifying `dtype` up front is both
  faster and safer.
- **`na_values`.** Real-world CSVs encode "missing" inconsistently — empty string, `"NA"`, `"-"`,
  `"999"` as a placeholder. `read_csv` only recognizes a fixed set of default missing markers; you
  have to tell it about the rest, or they'll silently become garbage strings.
- **`chunksize` for files that don't fit in memory.** Passing `chunksize=N` makes `read_csv` return
  an iterator of DataFrames of N rows each instead of one giant DataFrame, so you can process a
  multi-gigabyte file in pieces:

```python
total = 0
for chunk in pd.read_csv("huge.csv", chunksize=100_000):
    total += chunk["amt"].sum()   # aggregate chunk by chunk, never holding the whole file
```

This is the same idea as **out-of-core learning** from Section 01.4 — stream through data too big
for RAM rather than loading it all at once.

---

## 2. Working with JSON / SQL

### JSON

JSON is **semi-structured**: unlike a CSV, records don't need identical fields, and values can be
nested objects or lists (an order record embedding a list of line items, say). This is closer to
how APIs and document databases naturally represent data (see Section 3), which is why JSON shows
up constantly once you leave flat files behind.

For flat JSON, `pd.read_json` behaves much like `read_csv`. For nested JSON, the useful tool is
`json_normalize`, which flattens nested keys into dotted/underscored column names and can explode a
list-of-records field into rows:

```python
import pandas as pd
import json

with open("orders.json") as f:
    data = json.load(f)

# data looks like: [{"id": 1, "customer": {"name": "A", "city": "NY"}, "items": [...]}]
df = pd.json_normalize(data, sep="_")
# nested "customer.name" becomes a column "customer_name"

# to flatten a nested list field into its own rows, normalize with record_path:
items_df = pd.json_normalize(
    data, record_path="items", meta=["id"]
)
```

The honest caveat: flattening is a modeling decision, not a mechanical one. Deeply nested or
irregular JSON (fields that are sometimes present, sometimes absent, sometimes a different type)
often needs custom logic before `json_normalize` produces something you'd want as a training table.

### SQL

Data frequently lives in a relational database rather than a file. pandas doesn't talk to databases
directly — it delegates to a **connection object** made by a driver library (e.g. `sqlite3`,
`psycopg2`, or a SQLAlchemy engine), and just runs your query through it:

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("app.db")
df = pd.read_sql("SELECT id, name, amount FROM orders WHERE amount > 100", conn)
conn.close()
```

`read_sql` (and its narrower siblings `read_sql_query`, `read_sql_table`) is really just: run this
query, and hand back the result set as a DataFrame. All the real filtering/joining power is SQL
itself — pandas is the receiving end, not a query engine. This matters for large tables: pushing
filters into the SQL query (`WHERE`, column selection) is much cheaper than pulling the entire
table into memory and filtering in pandas afterward.

---

## 3. Fetching Data From an API

An API (specifically, a **REST API** here) is a service that hands back data over HTTP in response
to a request, instead of you reading a file someone already produced. The `requests` library is
the standard way to do this in Python:

```python
import requests

resp = requests.get(
    "https://api.example.com/v1/movies",
    params={"year": 2020},                       # becomes ?year=2020 in the URL
    headers={"Authorization": "Bearer <API_KEY>"} # auth: most APIs require a key/token
)
resp.raise_for_status()   # raises if the server returned an error status (4xx/5xx)
data = resp.json()        # most APIs return JSON; requests parses it into a dict/list
```

Once you have `data`, it's the same JSON-to-DataFrame problem as Section 2 above — often
`pd.json_normalize(data["results"])` or similar, since APIs commonly wrap the actual records in an
envelope object alongside metadata.

A few realities that separate "hello world" API code from something that actually gathers a full
dataset:

- **Authentication.** Most non-trivial APIs require an API key or token, usually sent as a header
  or query param. Treat keys as secrets — don't hard-code them into scripts you'll share or commit.
- **Pagination.** APIs rarely return everything in one response; results are split across pages
  (via a `page`/`offset` param, or a `next` link/cursor in the response). Gathering a full dataset
  means looping until there's no next page, not just calling the endpoint once.
- **Rate limits.** Providers cap how many requests you can make per minute/hour, and respond with
  a `429` status (or similar) once you exceed it. Respecting the limit (spacing out requests,
  backing off on `429`) is necessary, not optional — get too aggressive and you get blocked
  entirely.
- **Response shape isn't guaranteed to be stable.** Unlike a CSV you control, an API's schema can
  change between calls or versions; defensive parsing (checking keys exist before assuming they do)
  saves you from a pipeline that breaks silently.

---

## 4. Fetching Data Using Web Scraping

Web scraping is the fallback when there's a website with the data you want but no API or download
exposing it. The idea: fetch the raw HTML with `requests`, then parse the HTML structure to pull
out the specific text/attributes you need — typically with **BeautifulSoup**:

```python
import requests
from bs4 import BeautifulSoup

resp = requests.get("https://example.com/listings")
soup = BeautifulSoup(resp.text, "html.parser")

titles = [tag.get_text(strip=True) for tag in soup.select(".listing-title")]
prices = [tag.get_text(strip=True) for tag in soup.select(".listing-price")]

df = pd.DataFrame({"title": titles, "price": prices})
```

`BeautifulSoup` parses the HTML into a navigable tree and lets you select elements by tag, class,
id, or CSS selector — essentially the same targeting a browser's dev tools would show you, done in
code instead of by eye.

**When scraping is the right call, and when it isn't:**

- Scraping is appropriate when the data is genuinely only available as rendered web pages, and
  reasonable when the site's terms don't prohibit it. It's the *last* resort among the four methods
  in this section, precisely because of the issues below — reach for a CSV/DB/API first if one
  exists.
- **Check `robots.txt` and terms of service first.** `robots.txt` (e.g. `example.com/robots.txt`)
  states which paths a site permits automated crawlers to access. Scraping disallowed paths, or
  scraping in a way a site's terms forbid, is an ethical and potentially legal problem, not just a
  technical one — this is a judgment call to make deliberately, not an afterthought.
- **Scrapers are fragile by nature.** You're depending on a specific HTML structure the site owner
  didn't design for you and can change at any time — a class name gets renamed and your selectors
  return nothing. A scraper that works today has no guarantee of working next month, so scraped
  pipelines need more maintenance and validation than a file or API integration.
- Respect the server: space out requests, set a reasonable `User-Agent`, and don't hammer a site
  with concurrent requests it wasn't built to handle.

---

## Key takeaways

- CSV is simple and universal but type-blind and encoding-sensitive; `read_csv`'s parameters
  (`dtype`, `na_values`, `usecols`, `chunksize`) exist to handle the specific ways real CSVs are
  messy or too big for memory.
- JSON is semi-structured (nested, variable fields); `json_normalize` flattens it into a DataFrame,
  but deep/irregular nesting still needs custom judgment. SQL access via `read_sql` is just pandas
  running your query through a driver connection — push filtering into the query, not into pandas.
- Fetching from an API means handling auth, pagination, and rate limits *in addition to* parsing the
  JSON response — a single successful request is rarely the whole dataset.
- Web scraping (requests + BeautifulSoup) is the last-resort source: check `robots.txt` and terms
  before scraping, and expect scrapers to break whenever the target site's HTML changes.
- Across all four: gathering data is about getting *raw* rows into a DataFrame reliably. Making
  sense of what's in them is the next step.

**Next:** [03 — EDA](03-eda.md) — understanding the data you just gathered.
