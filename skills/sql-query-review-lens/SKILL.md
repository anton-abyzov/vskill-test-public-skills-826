---
name: sql-query-review-lens
version: 1.1.0
description: Review SQL queries for correctness, performance, and safety — index usage, N+1 patterns, injection risk, and EXPLAIN-driven tuning.
authors:
  - name: Fixture Maintainers
    email: maintainers@acme.example
license: MIT
tags:
  - sql
  - database
  - performance
  - code-review
  - security
---
# sql-query-review-lens

A structured checklist for reviewing SQL — whether hand-written, ORM-generated,
or produced by another agent. The goal is queries that are correct, fast, and
safe.

## Review order

Work top-down. Correctness first; a fast wrong answer is still wrong.

### 1. Correctness

- Do `JOIN` conditions match on the right keys? A missing condition silently
  becomes a cross join.
- Are `NULL`s handled? `col = NULL` is never true — use `IS NULL`. `NOT IN`
  with a NULL in the list returns no rows.
- Does `GROUP BY` list every non-aggregated selected column?
- Are window vs. aggregate semantics what you intend? `COUNT(*) OVER ()` differs
  from `COUNT(*)` + `GROUP BY`.

### 2. Performance

Run `EXPLAIN` (or `EXPLAIN ANALYZE` on Postgres) and read the plan:

| Plan node            | Usually means                          | Action                         |
|----------------------|----------------------------------------|--------------------------------|
| `Seq Scan` on a big table | no usable index for the predicate | add or fix an index            |
| `Nested Loop` with high rows | join blowup                     | check join keys / add index    |
| `Sort` spilling to disk | `work_mem` too small or huge set    | reduce rows early, paginate    |
| `Hash Join` build huge | building hash over the big side      | reorder / filter first         |

Common fixes:

- **Sargable predicates**: `WHERE created_at >= $1` can use an index;
  `WHERE date(created_at) = $1` cannot. Avoid wrapping the indexed column in a
  function.
- **Covering indexes**: include the selected columns so the engine skips the
  heap fetch.
- **Leading-column rule**: a composite index `(a, b)` helps `WHERE a = ?` and
  `WHERE a = ? AND b = ?`, but not `WHERE b = ?` alone.
- **Pagination**: prefer keyset pagination (`WHERE id > $last ORDER BY id LIMIT
  50`) over large `OFFSET`.

### 3. N+1 and round trips

Flag application code that runs one query per row of a previous result. Replace
with a single `JOIN` or an `IN (...)` batch. In ORMs, look for lazy-loaded
associations inside a loop.

### 4. Safety

- **Injection**: values must be bound parameters (`$1`, `?`, named binds), never
  string-concatenated. Identifiers that must be dynamic should be validated
  against an allowlist.
- **Mass updates**: every `UPDATE`/`DELETE` should have a `WHERE`. Run the
  matching `SELECT` first to confirm the row count.
- **Transactions**: multi-statement writes belong in one transaction with a
  clear isolation level.

## Example review

```sql
-- before: non-sargable, no parameter
SELECT * FROM orders
WHERE YEAR(created_at) = 2026 AND customer_email = '...';

-- after: sargable range + bound params + only needed columns
SELECT id, total_cents, status
FROM orders
WHERE created_at >= $1 AND created_at < $2
  AND customer_email = $3;
```

Pair the rewrite with an index on `(customer_email, created_at)`.

## Agent guidance

- Always ask for the schema and approximate row counts before judging a plan.
- Quote the specific plan node you are worried about; do not say "looks slow."
- Recommend at most the indexes the query needs — over-indexing slows writes.
- Never approve a query that builds SQL by concatenating untrusted input.
