# SQL

**Production SQL from real BI work, anonymized, published to be read.**

Every script here once fed a Power BI table from a live database. Names of companies, systems, clients, sites and people were replaced before publishing. The logic, the structure and every optimization are exactly what runs in production. None of the scripts will run outside its source database. That is the point: read them, borrow the shape, not the data.

---

## Catalog

| Script | Engine | Lines | What it does | Before → after |
|---|---|---|---|---|
| [`postgres/po-view/PO VIEW optimized.sql`](postgres/po-view/PO%20VIEW%20optimized.sql) | PostgreSQL 16 (Citus) | 2,288 | Purchase-order tracking view: PO lines matched to shipments, lead times, SLA exceptions, customs, invoices, one row per PO line per shipment | **4 min 54 s → 15 s**, same 3,600 rows, no index added |

More scripts land here as they are anonymized. Each one gets a folder with the script, a `NOTES.md` on what was changed and why, and the before and after numbers.

---

## The deploy pattern

Every file is self-contained. It puts the query into a session variable with dollar quoting, then writes that variable into a `sql_source` table keyed by report and page. The semantic model reads one row of that table.

```sql
set dev.po_view =
$sql$
    ...the query...
$sql$;

update sql_source
set  _code    = current_setting ( 'dev.po_view' )
    ,_updated = now()
where 1=1
    and _report = 'TRACKER'
    and _page   = 'PO VIEW';
```

Why it works: dollar quoting means no escaping, the database is the deploy target, git is the mirror, and the file name tells you which model table consumes it. One rule keeps it honest: **editing the file changes nothing on its own.** Edit, run the file, refresh the model, commit.

---

## House style

- Keywords lowercase. `select`, `from`, `where`, `case`, `sum`.
- One column per line, leading commas, no space after the comma.
- Every output column gets an alias starting with `_`, written without `as`, right-aligned into one block.
- Table aliases are one letter.
- `where 1=1`, then one `and` per line. Any filter can be commented out without breaking the query.
- `case` / `when` / `then` stepped one indent per level, alias on the `end` line.

The style is optimized for one thing: a 2,000-line query that scans as vertical blocks, where every line can be toggled on its own.

---

## How the PO view got 19x faster

No index was added. No `work_mem`, no buffer or cache setting was touched. Only the query changed, and the output stayed identical, checked one to one.

| Change | Before | After |
|---|---|---|
| Costs | aggregated for every shipment in every domain (6.6M rows, 2.4 GB) | only the shipments in scope |
| Shipment JSON | decompressed about 250 times per row through 286 separate reads | parsed once into a CTE, then hash joins |
| Port names | the port table scanned once per output row, twice | one join |
| Shipments and tracker view | read twice | read once, for the relevant serials |
| SLA map | built once per row | built once per company |

The biggest single win was not the big tables. Two per-row lookups on a small port table took 73 percent of the old run time and moved hundreds of gigabytes through the buffer cache. The planner expected 14 trillion rows at one window step; 1,600 came out.

Measured back to back on the same server, one run each, read-only and rolled back:

| | before | after |
|---|---|---|
| Execution | 4 min 54 s | 15 s |
| Plan nodes | 213 | 216 |
| Rows | 3,600 | 3,600 |

What is left: most of the remaining 15 seconds is one 1.3 GB scan of an attachments table, kept on purpose. The next fix is already named.

---

## What "anonymized" means here

- Company, system, client and report identifiers in table names, column names, literals and comments are renamed consistently. `focus__shipments` became `tms__shipments`, a client code became `client`, a company prefix became `org`.
- Site names, airport codes, partner codes and the debug serials in commented filters are replaced with placeholders.
- Third-party product names that identify no client (a visibility provider, a TMS vendor host turned into `example.com`) are kept or neutralized.
- Nothing else changed. Diff the file against its description and the shape holds.

---

## Related

- The story behind the PO view: [article on vosk.dev](https://vosk.dev) · [LinkedIn version](#)
- The tool that made it possible, a read-only MCP server for catalog facts, statistics and plan digests: [`sqldev-mcp`](#)
- Author: [vosk.dev](https://vosk.dev)

License: [choose one]. The scripts are published for reading and learning; the data they describe is not here and never was.
