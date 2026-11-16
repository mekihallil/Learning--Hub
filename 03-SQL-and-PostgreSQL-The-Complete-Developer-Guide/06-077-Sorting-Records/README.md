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