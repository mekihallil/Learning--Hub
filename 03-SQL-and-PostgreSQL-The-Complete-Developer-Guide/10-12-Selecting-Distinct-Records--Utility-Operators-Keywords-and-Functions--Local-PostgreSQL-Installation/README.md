## 🧼 79. Selecting Distinct Values

**What it does:** `SELECT DISTINCT` removes **duplicate rows** from the result, so each different row appears **once**.
**Why it is used:** Many columns repeat values (a manufacturer shared by several phones, a user who owns several photos). DISTINCT answers "what **different** values exist?"
**When to use it:** Building a dropdown of options, listing categories, or counting how many different things there are.

**Analogy:** A guest list where some people signed in twice. DISTINCT hands you the list with each person written **once**.

**All values, with repeats:**
```sql
SELECT manufacturer FROM phones;
```
**Output:**
```
 manufacturer
--------------
 Nokia
 Apple
 Samsung
 Samsung
(4 rows)
```

**Only the different values:**
```sql
SELECT DISTINCT manufacturer FROM phones;
```
**Output (row order may differ):**
```
 manufacturer
--------------
 Apple
 Nokia
 Samsung
(3 rows)
```

**`NULL` counts as one value:**
```sql
SELECT DISTINCT user_id FROM photos;
```
**Output (row order may differ):**
```
 user_id
---------

       1
       2
       3
       4
(5 rows)
```
The blank is `NULL` (the orphan photo). All `NULL`s collapse into **one** row, the same as in `GROUP BY` (topic 50).

### Count the different values
```sql
SELECT COUNT(DISTINCT manufacturer) AS manufacturers FROM phones;
```
**Output:**
```
 manufacturers
---------------
             3
(1 row)
```
`COUNT(DISTINCT ...)` ignores `NULL`s (topic 53):
```sql
SELECT COUNT(DISTINCT user_id) AS owners FROM photos;
```
**Output:**
```
 owners
--------
      4
(1 row)
```

### DISTINCT looks at the whole row
With several columns, a row is a duplicate only if **every column** matches:
```sql
SELECT DISTINCT photo_id, user_id FROM comments;
```
**Output:** 9 rows, one per different (photo, user) pair, down from 10 comments. User 2 commented on photo 2 twice, and that pair now appears once.

### DISTINCT ON: one row per group (a PostgreSQL feature)
`DISTINCT ON (column)` keeps **only the first row** for each value of that column. "First" is decided by `ORDER BY`.
```sql
SELECT DISTINCT ON (manufacturer) manufacturer, name, price
FROM phones
ORDER BY manufacturer, price DESC;
```
**Output:**
```
 manufacturer |   name    | price
--------------+-----------+-------
 Apple        | iPhone 5S |   399
 Nokia        | N1280     |   199
 Samsung      | Galaxy S5 |   299
(3 rows)
```
That's the priciest phone for each manufacturer, a neat shortcut for the correlated subquery in topic 75.

### DISTINCT vs GROUP BY
These two give the same rows:
```sql
SELECT DISTINCT manufacturer FROM phones;
SELECT manufacturer FROM phones GROUP BY manufacturer;
```

| | `DISTINCT` | `GROUP BY` |
|---|---|---|
| **Purpose** | Remove duplicate rows | Make buckets to summarize |
| **Aggregates (`COUNT`, `SUM`)** | Not needed | Usually used |
| **Reads like** | "Different values only" | "One row per group, with a summary" |

> ⚠️ **Warning:** `DISTINCT` applies to the **whole row**, not to one column. `SELECT DISTINCT manufacturer, name` returns every phone, because every (manufacturer, name) pair is different.

> ⚠️ **Warning:** `DISTINCT` is not a function. `SELECT DISTINCT(manufacturer), name` looks like it applies to one column, but the parentheses change nothing.

> ⚠️ **Warning:** Finding duplicates costs time on big tables. If you can't tell why rows repeat, check for a **join** multiplying them (topic 71) before reaching for `DISTINCT`.

> 🏆 **Golden Rule:** **`DISTINCT` = each different row once.** Use `COUNT(DISTINCT col)` to count different values, and `DISTINCT ON` to keep one row per group.