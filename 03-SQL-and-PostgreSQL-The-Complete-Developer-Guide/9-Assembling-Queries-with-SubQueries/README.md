## 🪆 66. What's a Subquery

**What it does:** A subquery is a **query written inside another query**, wrapped in parentheses. The inner query runs first, and its result is handed to the outer query.
**Why it is used:** Many questions have two steps: "find the average price, **then** find the phones above it." A subquery does both steps in one statement, and always with fresh numbers.
**When to use it:** Whenever one answer depends on another answer.

**Analogy:** Asking "who here is taller than the **average** person?" You can't answer it directly. First you work out the average height (inner question), then you compare everyone to it (outer question).

**Doing it by hand takes two queries:**
```sql
SELECT AVG(price) FROM phones;
```
**Output:**
```
         avg
----------------------
 286.7500000000000000
(1 row)
```
```sql
SELECT name, price FROM phones WHERE price > 286.75;
```
**Output:**
```
   name    | price
-----------+-------
 iPhone 5S |   399
 Galaxy S5 |   299
(2 rows)
```
This works, but `286.75` is typed in by hand. If a price changes, the number is stale.

**Doing it with a subquery, in one statement:**
```sql
SELECT name, price
FROM phones
WHERE price > (SELECT AVG(price) FROM phones);
```
**Output (row order may differ):**
```
   name    | price
-----------+-------
 iPhone 5S |   399
 Galaxy S5 |   299
(2 rows)
```

**The anatomy:**
```
SELECT name, price                  ← the OUTER query
FROM phones
WHERE price > (                     ← parentheses are required
  SELECT AVG(price) FROM phones     ← the INNER query (the subquery)
);
```

**Another example: the most expensive phone.**
```sql
SELECT name, price
FROM phones
WHERE price = (SELECT MAX(price) FROM phones);
```
**Output:**
```
   name    | price
-----------+-------
 iPhone 5S |   399
(1 row)
```

**Where a subquery can go:**

| Place | Covered in |
|---|---|
| The `SELECT` list | Topic 68 |
| The `FROM` clause | Topics 69 and 70 |
| A `JOIN` | Topic 71 |
| The `WHERE` clause | Topics 72 to 74 |

**A debugging habit:** highlight just the inner query and run it **on its own**. If it works alone, you know any bug is in how the outer query uses it.

**Subqueries can nest.** The inner query can contain its own subquery (an example comes in topic 72).

**Forgetting the parentheses:**
```sql
SELECT name FROM phones WHERE price > SELECT AVG(price) FROM phones;
```
**Output:**
```
ERROR:  syntax error at or near "SELECT"
LINE 1: SELECT name FROM phones WHERE price > SELECT AVG(price) FROM...
```

> ⚠️ **Warning:** A subquery always needs **parentheses** around it.

> ⚠️ **Warning:** The result of the subquery must have the **right shape** for where you put it (topic 67). A mismatch gives an error, or a silent wrong answer.

> 🏆 **Golden Rule:** **If you'd run one query, read the answer, and type it into a second query, use a subquery instead.** It stays correct when the data changes.



## 🧱 67. Thinking About the Structure of Data

**What it does:** Teaches you to look at any query's result and describe its **shape**: a single **value**, a **list**, or a full **table**. The shape decides where a subquery is allowed to go.
**Why it is used:** Almost every subquery error is a shape mismatch, such as handing a list to something that wants one value. If you check the shape first, you can predict the error before you run anything.
**When to use it:** Before you place a subquery anywhere.

**Analogy:** A mail slot, a shelf, and a table. A letter (one value) fits the mail slot. A row of letters (a list) fits a shelf. A full spread of letters and parcels (a table) needs the table. Pushing a table through the mail slot jams.

**The two questions to ask of any result:**

| Question | Answers |
|---|---|
| How many **columns**? | 1 or many |
| How many **rows**? | 1 or many |

**The three shapes:**

| Columns | Rows | Shape | Looks like |
|---|---|---|---|
| 1 | 1 | **Scalar** (a single value) | `399` |
| 1 | many | **List** | `299, 250` |
| many | many | **Table** | name + price for each phone |

**Scalar: one column, one row**
```sql
SELECT MAX(price) FROM phones;
```
**Output:**
```
 max
-----
 399
(1 row)
```

**List: one column, many rows**
```sql
SELECT price FROM phones WHERE manufacturer = 'Samsung';
```
**Output:**
```
 price
-------
   299
   250
(2 rows)
```

**Table: many columns, many rows**
```sql
SELECT name, price FROM phones;
```
**Output:**
```
    name     | price
-------------+-------
 N1280       |   199
 iPhone 5S   |   399
 Galaxy S5   |   299
 S5620 Monte |   250
(4 rows)
```

**Which shape each position accepts:**

| Where the subquery goes | Shape it must return |
|---|---|
| In the `SELECT` list | **Scalar** |
| After `=`, `>`, `<`, and similar in `WHERE` | **Scalar** |
| After `IN`, `NOT IN`, `ANY`, `ALL` | **List** (one column) |
| In `FROM` | **Table** (any shape, even a scalar or a list) |
| In a `JOIN` | **Table** |

**Shape mismatch 1: many rows where one value is expected**
```sql
SELECT name
FROM phones
WHERE price > (SELECT price FROM phones WHERE manufacturer = 'Samsung');
```
**Output:**
```
ERROR:  more than one row returned by a subquery used as an expression
```
The subquery returns two prices (299 and 250), but `>` wants one.

**Shape mismatch 2: two columns where one is expected**
```sql
SELECT name FROM phones WHERE price IN (SELECT name, price FROM phones);
```
**Output:**
```
ERROR:  subquery has too many columns
```

**An empty result is a shape too:**

| Used as... | An empty result acts like... |
|---|---|
| A scalar | `NULL` (so `price > NULL` is unknown and matches **no** rows) |
| A list for `IN` | An empty list (nothing matches) |
| A table in `FROM` | A table with zero rows |

**Quick shape check habit:** run the inner query alone and count its columns and rows.

> ⚠️ **Warning:** A scalar subquery can work today and **fail tomorrow**. `WHERE name = (SELECT ...)` succeeds while the inner query returns one row, then errors when a second matching row appears.

> ⚠️ **Warning:** A scalar subquery that matches **no** rows gives `NULL`, not an error. A comparison against `NULL` quietly returns no rows (topic 11).

> 🏆 **Golden Rule:** **Count the columns and rows first.** One value goes anywhere, a one-column list goes with `IN`, and a full table goes in `FROM` or a `JOIN`.



## 🧾 68. Subqueries in a Select

**What it does:** Puts a subquery **in the `SELECT` list**, so it becomes an extra **column** in the output. It must return a **single value** (a scalar, topic 67).
**Why it is used:** You can show a summary number (like the highest price) **next to every row**, or do math between each row and that number.
**When to use it:** Comparing each row to a total, an average, a maximum, or any one-off lookup.

**Analogy:** A school report card that prints your score and also prints the **class average** on every line, so you can compare at a glance.

**Show the maximum price beside every phone:**
```sql
SELECT name, price, (SELECT MAX(price) FROM phones) AS max_price
FROM phones;
```
**Output:**
```
    name     | price | max_price
-------------+-------+-----------
 N1280       |   199 |       399
 iPhone 5S   |   399 |       399
 Galaxy S5   |   299 |       399
 S5620 Monte |   250 |       399
(4 rows)
```
The inner query runs **once**, and its single value is repeated on every row.

**Do math with it:** how far is each phone below the top price?
```sql
SELECT name, price, (SELECT MAX(price) FROM phones) - price AS below_max
FROM phones;
```
**Output:**
```
    name     | price | below_max
-------------+-------+-----------
 N1280       |   199 |       200
 iPhone 5S   |   399 |         0
 Galaxy S5   |   299 |       100
 S5620 Monte |   250 |       149
(4 rows)
```

### A subquery that looks at the outer row (correlated)
The subquery can refer to a column of the **outer** query. Then it re-runs **for each outer row**.
```sql
SELECT
  username,
  (SELECT COUNT(*) FROM photos WHERE photos.user_id = users.id) AS photo_count
FROM users;
```
**Output:**
```
  username  | photo_count
------------+-------------
 monahan93  |           3
 pferrer    |           1
 si93onis   |           1
 99stroman  |           2
 lonelyuser |           0
(5 rows)
```
For each user, the subquery counts the photos whose `user_id` matches **that** user's `id`. `lonelyuser` gets `0` because `COUNT(*)` returns `0` when there are no rows.

**Not every aggregate returns 0 for no rows:**
```sql
SELECT
  username,
  (SELECT MAX(id) FROM photos WHERE photos.user_id = users.id) AS newest_photo_id
FROM users;
```
**Output:**
```
  username  | newest_photo_id
------------+-----------------
 monahan93  |               4
 pferrer    |               5
 si93onis   |               6
 99stroman  |               7
 lonelyuser |
(5 rows)
```
`MAX` of nothing is `NULL`.

**The two shape errors you can hit here:**
```sql
SELECT name, (SELECT price FROM phones) FROM phones;
```
**Output:**
```
ERROR:  more than one row returned by a subquery used as an expression
```
```sql
SELECT name, (SELECT name, price FROM phones LIMIT 1) FROM phones;
```
**Output:**
```
ERROR:  subquery must return only one column
```

**Plain vs correlated:**

| | Plain subquery | Correlated subquery |
|---|---|---|
| **Refers to the outer row?** | No | Yes (`users.id`) |
| **Runs** | Once | Once **per outer row** (conceptually) |
| **Example** | `(SELECT MAX(price) FROM phones)` | `(SELECT COUNT(*) ... WHERE photos.user_id = users.id)` |

**The correlated count could also be written with a join:**
```sql
SELECT u.username, COUNT(p.id) AS photo_count
FROM users AS u
LEFT JOIN photos AS p ON p.user_id = u.id
GROUP BY u.id;
```
Same result. Joins are usually faster on big tables, but the subquery version is often easier to read.

> ⚠️ **Warning:** A subquery in `SELECT` must return **exactly one column** and **at most one row**. More rows is an error, and zero rows gives `NULL`.

> ⚠️ **Warning:** A correlated subquery on a very large table can be slow, because it may run once for every row.

> 🏆 **Golden Rule:** **A `SELECT`-list subquery adds one computed column.** Make sure it can only ever return one value.



## 📋 69. Subqueries in a From

**What it does:** Puts a subquery **in the `FROM` clause**, so its result acts like a **temporary table** that the outer query reads from.
**Why it is used:** Some answers need **two rounds of summarizing**, like "the average of the per-group maximums." You can't nest aggregate functions directly, but you can aggregate the result of an earlier aggregate.
**When to use it:** Multi-step calculations, or filtering on a calculated column.

**Analogy:** A spreadsheet where you **copy a finished summary table** to a new sheet, then calculate on that sheet. The subquery builds the summary, and the outer query works on it.

### Example 1: the average of each manufacturer's top price
**Step 1: the inner query on its own.** It gives one maximum per manufacturer:
```sql
SELECT MAX(price) AS max_price
FROM phones
GROUP BY manufacturer;
```
**Output (row order may differ):**
```
 max_price
-----------
       199
       399
       299
(3 rows)
```
**Step 2: average that little table:**
```sql
SELECT AVG(max_price) AS avg_of_maxes
FROM (
  SELECT MAX(price) AS max_price
  FROM phones
  GROUP BY manufacturer
) AS p;
```
**Output:**
```
    avg_of_maxes
----------------------
 299.0000000000000000
(1 row)
```
`(199 + 399 + 299) / 3 = 299`.

**Why not do it in one step?**
```sql
SELECT AVG(MAX(price)) FROM phones GROUP BY manufacturer;
```
**Output:**
```
ERROR:  aggregate function calls cannot be nested
```

### Example 2: the most photos any one user has
```sql
SELECT MAX(photo_count) AS most_photos
FROM (
  SELECT COUNT(*) AS photo_count
  FROM photos
  WHERE user_id IS NOT NULL
  GROUP BY user_id
) AS counts;
```
**Output:**
```
 most_photos
-------------
           3
(1 row)
```
The inner query counts photos per user (3, 1, 1, 2), and the outer query takes the maximum.

### Example 3: the alias workaround
You can't use a `SELECT` alias in `WHERE` (topic 17). A subquery in `FROM` fixes that, because the alias becomes a **real column** of the temporary table:
```sql
SELECT name, price
FROM (
  SELECT name, price, price * units_sold AS revenue
  FROM phones
) AS p
WHERE revenue > 600000;
```
**Output:**
```
   name    | price
-----------+-------
 iPhone 5S |   399
 Galaxy S5 |   299
(2 rows)
```

**Rules for a subquery in `FROM`:**

| Rule | Meaning |
|---|---|
| It needs an **alias** | `... ) AS p` |
| The outer query sees the **inner `SELECT` list's column names** | Give calculated columns aliases (`AS max_price`) |
| The outer query can only use columns the inner query **selected** | Unselected columns are gone |
| It can return any shape | One value, a list, or a full table |

**Alias in older versions:**
```sql
SELECT AVG(max_price) FROM (SELECT MAX(price) AS max_price FROM phones GROUP BY manufacturer);
```
**Output on PostgreSQL 15 and earlier:**
```
ERROR:  subquery in FROM must have an alias
HINT:  For example, FROM (SELECT ...) [AS] foo.
```
PostgreSQL 16 and newer allow it to be omitted, but **always write one** so the query works everywhere.

> ⚠️ **Warning:** Column names come from the inner query. Without an alias, `MAX(price)` is named `max`, and `price * units_sold` gets the unhelpful name `?column?`.

> ⚠️ **Warning:** Only the columns you **select inside** the subquery exist outside it. Forgetting one means `ERROR: column "..." does not exist`.

> 🏆 **Golden Rule:** **`FROM (subquery) AS alias` turns a query result into a table you can query again.** Use it for a second round of aggregation or to filter on a calculated column.