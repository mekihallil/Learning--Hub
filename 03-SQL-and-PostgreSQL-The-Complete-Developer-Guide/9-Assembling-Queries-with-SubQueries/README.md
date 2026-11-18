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