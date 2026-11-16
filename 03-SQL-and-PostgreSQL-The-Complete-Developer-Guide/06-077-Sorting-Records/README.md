## 🔃 59. The Basics of Sorting

**What it does:** `ORDER BY` sorts the rows of a result. By default it sorts **smallest to largest** (A to Z, low to high, oldest to newest).
**Why it is used:** Without it, Postgres returns rows in **no guaranteed order**. Anything shown to a person (leaderboards, price lists, newest-first feeds) needs a sort.
**When to use it:** Any time the order of the rows matters to whoever reads the result.

**Analogy:** A librarian who sorts a pile of returned books by title before shelving them. The books are the same, only their order changes.

**Sort by a number (lowest first):**
```sql
SELECT name, price
FROM phones
ORDER BY price;
```
**Output:**
```
    name     | price
-------------+-------
 N1280       |   199
 S5620 Monte |   250
 Galaxy S5   |   299
 iPhone 5S   |   399
(4 rows)
```

**Sort from highest to lowest with `DESC`:**
```sql
SELECT name, price
FROM phones
ORDER BY price DESC;
```
**Output:**
```
    name     | price
-------------+-------
 iPhone 5S   |   399
 Galaxy S5   |   299
 S5620 Monte |   250
 N1280       |   199
(4 rows)
```

**Sort text alphabetically:**
```sql
SELECT username
FROM users
ORDER BY username;
```
**Output:**
```
  username
------------
 99stroman
 lonelyuser
 monahan93
 pferrer
 si93onis
(5 rows)
```

**Sort by a calculated value, using its alias:**
```sql
SELECT name, price * units_sold AS revenue
FROM phones
ORDER BY revenue DESC;
```
**Output:**
```
    name     | revenue
-------------+---------
 iPhone 5S   | 3764964
 Galaxy S5   |  705341
 S5620 Monte |  596250
 N1280       |  383075
(4 rows)
```

**Sort grouped results:**
```sql
SELECT manufacturer, SUM(units_sold) AS total_units
FROM phones
GROUP BY manufacturer
ORDER BY total_units DESC;
```
**Output:**
```
 manufacturer | total_units
--------------+-------------
 Apple        |        9436
 Samsung      |        4744
 Nokia        |        1925
(3 rows)
```

**Where `ORDER BY` goes:** it is the **last** of the filtering and grouping clauses.

| Order you write | Order Postgres runs |
|---|---|
| `SELECT` | 3rd to last... see below |
| `FROM` | 1 |
| `WHERE` | 2 |
| `GROUP BY` | 3 |
| `HAVING` | 4 |
| `SELECT` | 5 |
| `ORDER BY` | 6 |

Because `ORDER BY` runs **after** `SELECT`, it can use your **aliases** (`ORDER BY revenue`). That's the reverse of `WHERE` and `HAVING`, which can't (topics 17 and 57).

**What you can sort by:**

| You sort by | Example |
|---|---|
| A column | `ORDER BY price` |
| A calculation | `ORDER BY price * units_sold` |
| An alias | `ORDER BY revenue` |
| An aggregate | `ORDER BY SUM(units_sold)` |
| A column position | `ORDER BY 2` (works, but breaks if you reorder the columns) |

**Summary of the keywords:**

| Keyword | Meaning |
|---|---|
| `ASC` | Ascending, smallest first (the **default**, so you can leave it out) |
| `DESC` | Descending, largest first |

> ⚠️ **Warning:** **Without `ORDER BY`, row order is not guaranteed**, even if a table seems to come back in id order today. Never depend on it.

> ⚠️ **Warning:** Text sorting follows the database's **collation** (language rules). Upper and lower case, accents, and symbols can sort differently on different servers, so the order of mixed-case text may not match mine.

> ⚠️ **Warning:** Numbers stored as **text** sort alphabetically, so `'10'` comes before `'9'`. Store numbers in number columns.

> 🏆 **Golden Rule:** **If order matters, say so with `ORDER BY`.** Never assume a table returns rows in a particular order.



## 🔀 60. Two Variations on Sorting

**What it does:** Two ways to make `ORDER BY` more precise. **Variation 1:** choose the **direction** (`ASC` or `DESC`). **Variation 2:** sort by **several columns**, so a second column breaks ties in the first.
**Why it is used:** One column often has repeated values. Without a tie-breaker, rows with equal values come back in an unpredictable order.
**When to use it:** Leaderboards with tied scores, products sorted by brand and then price, any list that must be consistent from one run to the next.

**Analogy:** A phone book sorts by **last name first**. Among people with the same last name, it then sorts by **first name**. The second rule only matters when the first one ties.

### Variation 1: ascending or descending
```sql
SELECT name, price FROM phones ORDER BY price ASC;    -- same as no keyword
SELECT name, price FROM phones ORDER BY price DESC;   -- reversed
```
**Output of the `DESC` version:**
```
    name     | price
-------------+-------
 iPhone 5S   |   399
 Galaxy S5   |   299
 S5620 Monte |   250
 N1280       |   199
(4 rows)
```

### Variation 2: sort by multiple columns
**Sort by manufacturer first, then by price within each manufacturer:**
```sql
SELECT manufacturer, name, price
FROM phones
ORDER BY manufacturer, price;
```
**Output:**
```
 manufacturer |    name     | price
--------------+-------------+-------
 Apple        | iPhone 5S   |   399
 Nokia        | N1280       |   199
 Samsung      | S5620 Monte |   250
 Samsung      | Galaxy S5   |   299
(4 rows)
```
Both Samsung phones tie on `manufacturer`, so `price` decides: 250 before 299.

**Each column gets its own direction:**
```sql
SELECT manufacturer, name, price
FROM phones
ORDER BY manufacturer ASC, price DESC;
```
**Output:**
```
 manufacturer |    name     | price
--------------+-------------+-------
 Apple        | iPhone 5S   |   399
 Nokia        | N1280       |   199
 Samsung      | Galaxy S5   |   299
 Samsung      | S5620 Monte |   250
(4 rows)
```
`DESC` applies **only to the column right before it**. Now the Samsungs are 299 then 250.

**A tie-breaker makes results reliable.** Many photos share a `user_id`:
```sql
SELECT id, url, user_id
FROM photos
ORDER BY user_id, id;
```
**Output:**
```
 id |       url       | user_id
----+-----------------+---------
  2 | http://two.jpg  |       1
  3 | http://25.jpg   |       1
  4 | http://36.jpg   |       1
  5 | http://754.jpg  |       2
  6 | http://35.jpg   |       3
  1 | http://one.jpg  |       4
  7 | http://256.jpg  |       4
  8 | http://lost.jpg |
(8 rows)
```
Adding `id` (which is unique) guarantees the same order every time.

### Bonus: where do `NULL`s go?
In an **ascending** sort, `NULL`s come **last**. In a **descending** sort, they come **first**:
```sql
SELECT id, user_id FROM photos ORDER BY user_id DESC, id;
```
**Output:**
```
 id | user_id
----+---------
  8 |
  1 |       4
  7 |       4
  6 |       3
  5 |       2
  2 |       1
  3 |       1
  4 |       1
(8 rows)
```
Override this with `NULLS FIRST` or `NULLS LAST`:
```sql
SELECT id, user_id FROM photos ORDER BY user_id DESC NULLS LAST, id;
```
**Output:** the same rows, but photo 8 now comes at the **bottom**.

**Quick reference:**

| You write | Result |
|---|---|
| `ORDER BY price` | Lowest first |
| `ORDER BY price DESC` | Highest first |
| `ORDER BY a, b` | Sort by `a`, break ties with `b` |
| `ORDER BY a DESC, b` | `a` highest first, ties by `b` lowest first |
| `ORDER BY a DESC, b DESC` | Both descending |
| `ORDER BY a NULLS LAST` | Put missing values at the end |

> ⚠️ **Warning:** Writing `ORDER BY a, b DESC` does **not** sort both descending. `DESC` belongs only to `b`. Add `DESC` to each column you want reversed.

> ⚠️ **Warning:** If the first column has **no ties**, the second column never has any effect.

> ⚠️ **Warning:** Rows with tied values and no tie-breaker can reorder between runs. That's harmless for a one-off look, but a problem for **pagination** (topic 61).

> 🏆 **Golden Rule:** **End every important sort with a unique column** (usually the `id`). It removes all ambiguity.


## ✂️ 61. Offset and Limit

**What it does:** `LIMIT` returns only the **first N rows** of a result. `OFFSET` **skips N rows** before it starts returning.
**Why it is used:** Showing "the top 3," "the latest 10," or "page 2 of results" means returning a slice of the rows, not all of them.
**When to use it:** Top-N lists, leaderboards, and pagination (splitting results across pages).

**Analogy:** A shop window display. `LIMIT` is how many items fit in the window. `OFFSET` is how many items you move past before you start filling it.

**Keep only the first 2 rows (the two most expensive phones):**
```sql
SELECT name, price
FROM phones
ORDER BY price DESC
LIMIT 2;
```
**Output:**
```
   name    | price
-----------+-------
 iPhone 5S |   399
 Galaxy S5 |   299
(2 rows)
```

**The single cheapest phone:**
```sql
SELECT name, price
FROM phones
ORDER BY price
LIMIT 1;
```
**Output:**
```
 name  | price
-------+-------
 N1280 |   199
(1 row)
```

**Skip the first row, then take 2 (`OFFSET`):**
```sql
SELECT name, price
FROM phones
ORDER BY price DESC
LIMIT 2 OFFSET 1;
```
**Output:**
```
    name     | price
-------------+-------
 Galaxy S5   |   299
 S5620 Monte |   250
(2 rows)
```
The most expensive phone (399) was skipped.

**Pagination with photos, 3 per page:**

| Page | Query ends with | Rows returned |
|---|---|---|
| 1 | `LIMIT 3 OFFSET 0` | ids 1, 2, 3 |
| 2 | `LIMIT 3 OFFSET 3` | ids 4, 5, 6 |
| 3 | `LIMIT 3 OFFSET 6` | ids 7, 8 |

```sql
SELECT id, url
FROM photos
ORDER BY id
LIMIT 3 OFFSET 3;
```
**Output (page 2):**
```
 id |      url
----+----------------
  4 | http://36.jpg
  5 | http://754.jpg
  6 | http://35.jpg
(3 rows)
```
**The formula:**
```
OFFSET = (page_number - 1) * page_size
```

**Asking for a page beyond the end is not an error:**
```sql
SELECT id FROM photos ORDER BY id LIMIT 3 OFFSET 100;
```
**Output:**
```
 id
----
(0 rows)
```

**The standard SQL form** also works in PostgreSQL:
```sql
SELECT name, price FROM phones ORDER BY price DESC
OFFSET 1 ROWS FETCH FIRST 2 ROWS ONLY;
```
It returns the same two rows as `LIMIT 2 OFFSET 1`.

**Where it goes in a query:**

| Order you write | |
|---|---|
| `SELECT` ... `FROM` ... | |
| `WHERE` ... `GROUP BY` ... `HAVING` ... | |
| `ORDER BY` ... | sort first |
| `LIMIT` ... `OFFSET` ... | **last**, cut the slice |

`LIMIT` and `OFFSET` run **after** sorting. That is why "top 2 by price" needs `ORDER BY price DESC` first.

> ⚠️ **Warning:** **`LIMIT` without `ORDER BY` returns an arbitrary set of rows.** "The first 3 rows" means nothing until you say what "first" is.

> ⚠️ **Warning:** For pagination, always sort by something **unique** (topic 60). Otherwise tied rows can shuffle between pages, causing duplicates or skipped rows.

> ⚠️ **Warning:** A large `OFFSET` gets slow, because Postgres still has to read and discard all the skipped rows. For huge tables, apps often page with a condition like `WHERE id > last_seen_id ORDER BY id LIMIT 20` instead.

> 🏆 **Golden Rule:** **Sort first, then slice.** `ORDER BY` decides which rows come first, and `LIMIT`/`OFFSET` choose the slice.