## 🔁 87. What'd We Just Do

**What it does:** A recap of everything covered so far, plus a cheat sheet for the order SQL clauses are written and the order Postgres actually runs them.
**Why it is used:** After 86 topics, it helps to zoom out. Seeing the whole toolbox in one place shows how the pieces fit together.
**When to use it:** As a checkpoint before starting the next section, and as a quick reference later.

**Analogy:** The end of a long hike where you stop at a viewpoint and look at the trail you've covered, before the next climb.

### The route so far
| Topics | What you learned | Key tools |
|---|---|---|
| 1 to 9 | Tables, inserting, selecting, calculated columns, strings | `CREATE TABLE`, `INSERT`, `SELECT`, `AS`, `\|\|`, `LOWER` |
| 10 to 19 | Filtering and changing data | `WHERE`, `AND`/`OR`, `IN`, `UPDATE`, `DELETE` |
| 20 to 35 | Database design and relationships | Primary keys, foreign keys, `SERIAL`, `ON DELETE` |
| 36 to 48 | Combining tables | `JOIN`, `LEFT`/`RIGHT`/`FULL`, three-way joins |
| 49 to 58 | Summarizing data | `COUNT`, `SUM`, `GROUP BY`, `HAVING` |
| 59 to 61 | Sorting and slicing | `ORDER BY`, `LIMIT`, `OFFSET` |
| 62 to 65 | Combining result sets | `UNION`, `INTERSECT`, `EXCEPT` |
| 66 to 78 | Queries inside queries | Subqueries, correlated subqueries, `EXISTS`, `ANY`/`ALL` |
| 79 to 84 | Handy extras | `DISTINCT`, `GREATEST`, `LEAST`, `CASE` |
| 85 to 86 | Your own workshop | Installing PostgreSQL and pgAdmin |

### The order you write a query
```sql
SELECT   columns
FROM     table
JOIN     other_table ON ...
WHERE    row conditions
GROUP BY grouping columns
HAVING   group conditions
ORDER BY sort columns
LIMIT    n OFFSET m;
```

### The order Postgres runs it
| Step | Clause | What happens |
|---|---|---|
| 1 | `FROM` / `JOIN` | Build the rows |
| 2 | `WHERE` | Remove individual rows |
| 3 | `GROUP BY` | Make buckets |
| 4 | Aggregates | Calculate `COUNT`, `SUM`, and so on |
| 5 | `HAVING` | Remove whole buckets |
| 6 | `SELECT` | Choose and calculate columns, then `DISTINCT` |
| 7 | `ORDER BY` | Sort |
| 8 | `LIMIT` / `OFFSET` | Cut the slice |

This order explains most "why doesn't this work?" moments:

| Surprise | Reason |
|---|---|
| `WHERE` can't use a `SELECT` alias | `WHERE` runs **before** `SELECT` |
| `WHERE` can't use `COUNT(*)` | Groups don't exist yet, use `HAVING` |
| `ORDER BY` **can** use an alias | `ORDER BY` runs **after** `SELECT` |
| `LIMIT` without `ORDER BY` is random | The slice is cut from an unordered set |

### Which tool for which question?
| Question | Tool |
|---|---|
| Show only some rows | `WHERE` |
| Combine rows from two tables | `JOIN` |
| Keep unmatched rows too | `LEFT` / `RIGHT` / `FULL JOIN` |
| How many, how much, what average | Aggregates |
| ...per category | `GROUP BY` |
| ...only for big categories | `HAVING` |
| Stack two result lists | `UNION` |
| Rows in both lists / only in the first | `INTERSECT` / `EXCEPT` |
| The answer depends on another answer | Subquery |
| Turn values into labels | `CASE` |

### Check that your practice database is healthy
```sql
\dt
```
**Output:**
```
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | comments | table | postgres
 public | phones   | table | postgres
 public | photos   | table | postgres
 public | users    | table | postgres
(4 rows)
```
```sql
SELECT
  (SELECT COUNT(*) FROM users) AS users,
  (SELECT COUNT(*) FROM photos) AS photos,
  (SELECT COUNT(*) FROM comments) AS comments;
```
**Output:**
```
 users | photos | comments
-------+--------+----------
     5 |      8 |       10
(1 row)
```
If your numbers differ, run the reset script from topic 36.

### What's next
So far you've **used** data. The next topics look at what you can **store**: the kinds of values a column can hold (numbers, text, booleans, dates, and durations), and why picking the right type matters.

> ⚠️ **Warning:** Skills fade without practice. Re-run a few earlier queries, change one thing, and predict the result **before** running it.

> 🏆 **Golden Rule:** **Know the running order.** `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `SELECT`, `ORDER BY`, `LIMIT`. Almost every SQL puzzle becomes clear once you ask "which step am I at?"