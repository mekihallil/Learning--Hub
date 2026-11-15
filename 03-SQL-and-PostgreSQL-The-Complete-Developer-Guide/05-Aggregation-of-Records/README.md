## 📊 49. Aggregating and Grouping

**What it does:** Introduces two linked ideas. **Aggregation** collapses many rows into **one summary value** (a count, a total, an average). **Grouping** splits rows into **buckets** first, so you get **one summary per bucket**.
**Why it is used:** Often you want a **number**, not a list of rows. "How many photos does each user have?" is far more useful than scrolling through every photo.
**When to use it:** Reports, dashboards, statistics, and any question that starts with "how many," "what's the total," "what's the average," or "which is the biggest."

**Analogy:** A teacher has a stack of test papers. **Aggregation** is asking "what is the class average?" and getting one number. **Grouping** is sorting the papers into piles by class first, then finding the average **of each pile**.

**Which tool answers which question:**

| Question | Tools needed |
|---|---|
| How many photos exist? | An aggregate only |
| How many photos does **each user** have? | Grouping + an aggregate |
| Which users have **more than 2** photos? | Grouping + an aggregate + `HAVING` (topic 56) |

**Your first aggregate:**
```sql
SELECT COUNT(*) FROM photos;
```
**Output:**
```
 count
-------
     8
(1 row)
```
Eight photo rows went in, **one** row came out.

**Your first grouping:**
```sql
SELECT user_id, COUNT(*) AS photo_count
FROM photos
GROUP BY user_id;
```
**Output (row order may differ):**
```
 user_id | photo_count
---------+-------------
         |           1
       1 |           3
       2 |           1
       3 |           1
       4 |           2
(5 rows)
```
One row per **user**, with that user's photo count. The blank `user_id` is the orphan photo `lost.jpg`.

**The roadmap for this section:**

| Topic | Idea |
|---|---|
| 50 | A picture of what `GROUP BY` does |
| 51 | The aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) |
| 52 | Using `GROUP BY` and aggregates together |
| 53 | A trap with `COUNT` |
| 54 to 55 | More grouping, including grouping joined tables |
| 56 to 58 | `HAVING`, which filters whole groups |

**Check your data.** Run these three and compare:
```sql
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM photos;
SELECT COUNT(*) FROM comments;
```
**Output:** `5`, `8`, and `4`. If yours differ, run the reset script from topic 36, then the two inserts from topic 42.

> ⚠️ **Warning:** Aggregation **throws away the individual rows**. After `COUNT(*)` you can no longer see which photos were counted, only how many.

> 🏆 **Golden Rule:** **Aggregate = many rows in, one value out. Group = do that once per bucket.**



## 🖼️ 50. Picturing Group By

**What it does:** Shows, step by step, what `GROUP BY` does to a table. It **sorts rows into buckets** that share the same value, then turns **each bucket into a single row**.
**Why it is used:** Most `GROUP BY` mistakes come from not knowing what happens to the original rows. Once you can picture the buckets, the rules explain themselves.
**When to use it:** Any time you're about to write a `GROUP BY`, or an error message mentions it.

**Analogy:** A mail room. Letters arrive in one big pile, and a worker drops each into the **pigeonhole for its address**. At the end, each pigeonhole is a group.

**Start with the photos table:**
```
 id |      url        | user_id
----+-----------------+---------
  1 | http://one.jpg  |       4
  2 | http://two.jpg  |       1
  3 | http://25.jpg   |       1
  4 | http://36.jpg   |       1
  5 | http://754.jpg  |       2
  6 | http://35.jpg   |       3
  7 | http://256.jpg  |       4
  8 | http://lost.jpg |
```

**Step 1: `GROUP BY user_id` sorts rows into buckets by `user_id`:**
```
 Bucket user_id = 1     Bucket user_id = 2     Bucket user_id = 3
 ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
 │ photo 2       │      │ photo 5       │      │ photo 6       │
 │ photo 3       │      └───────────────┘      └───────────────┘
 │ photo 4       │
 └───────────────┘
 Bucket user_id = 4     Bucket user_id = NULL
 ┌───────────────┐      ┌───────────────┐
 │ photo 1       │      │ photo 8       │
 │ photo 7       │      └───────────────┘
 └───────────────┘
```

**Step 2: each bucket becomes ONE output row:**
```sql
SELECT user_id
FROM photos
GROUP BY user_id;
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
8 photo rows became **5** rows, one per bucket.

**Key facts the picture reveals:**

| Fact | Why |
|---|---|
| The number of output rows = the number of **buckets** | One row per group |
| `NULL`s all land in **one** bucket | `GROUP BY` treats `NULL`s as equal to each other |
| `GROUP BY` alone behaves like `SELECT DISTINCT` | It just lists each bucket once |

**The rule that follows:** each bucket becomes **one** row, so you can only show a column if it has **one value per bucket**.
```sql
SELECT user_id, url
FROM photos
GROUP BY user_id;
```
**Output:**
```
ERROR:  column "photos.url" must appear in the GROUP BY clause or be used in an aggregate function
LINE 1: SELECT user_id, url
                        ^
```
Bucket 1 holds **three** different URLs. Postgres can't squeeze three values into one cell, so it refuses.

**Which columns you may select after `GROUP BY`:**

| Column | Allowed? | Why |
|---|---|---|
| The column you grouped by (`user_id`) | ✅ | Same value for the whole bucket |
| An aggregate like `COUNT(*)` | ✅ | Produces one value per bucket (topic 51) |
| Any other plain column (`url`) | ❌ | Different values inside one bucket |

> ⚠️ **Warning:** `GROUP BY` doesn't sort the output. Row order is unpredictable unless you add `ORDER BY` (covered later).

> ⚠️ **Warning:** The error message mentions "the GROUP BY clause or an aggregate function" because those are the **only two** ways to show a column.

> 🏆 **Golden Rule:** **`GROUP BY` turns each bucket into one row.** If a column can have several values inside a bucket, you must summarize it with an aggregate or leave it out.



## 🧮 51. Aggregate Functions

**What it does:** Aggregate functions take **many values** and return **one**: `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`.
**Why it is used:** They are the "summary" half of grouping (topic 52). On their own they summarize the **whole table**.
**When to use it:** Totals, averages, counts, and "biggest or smallest" questions.

**Analogy:** The bottom line of a receipt. Many item lines go in, and a **total** comes out.

**The five functions:**

| Function | Returns | Ignores `NULL`? |
|---|---|---|
| `COUNT(*)` | Number of **rows** | No, counts every row |
| `COUNT(column)` | Number of **non-NULL** values | ✅ Yes |
| `SUM(column)` | Total | ✅ Yes |
| `AVG(column)` | Average | ✅ Yes |
| `MIN(column)` | Smallest value | ✅ Yes |
| `MAX(column)` | Largest value | ✅ Yes |

**Recreate the `phones` table so your numbers match** (it's fine to run this inside `photo_app`):
```sql
DROP TABLE IF EXISTS phones;

CREATE TABLE phones (
  name VARCHAR(50),
  manufacturer VARCHAR(50),
  price INTEGER,
  units_sold INTEGER
);

INSERT INTO phones (name, manufacturer, price, units_sold)
VALUES
  ('N1280', 'Nokia', 199, 1925),
  ('iPhone 5S', 'Apple', 399, 9436),
  ('Galaxy S5', 'Samsung', 299, 2359),
  ('S5620 Monte', 'Samsung', 250, 2385);
```
**Output:**
```
DROP TABLE
CREATE TABLE
INSERT 0 4
```

**Count:**
```sql
SELECT COUNT(*) FROM photos;
```
**Output:**
```
 count
-------
     8
(1 row)
```

**Sum, average, smallest, largest:**
```sql
SELECT SUM(units_sold) FROM phones;
```
**Output:**
```
  sum
-------
 16105
(1 row)
```
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
`AVG` returns a long decimal. Tidy it with `ROUND` (topic 7):
```sql
SELECT ROUND(AVG(price), 2) AS average_price FROM phones;
```
**Output:**
```
 average_price
---------------
        286.75
(1 row)
```
```sql
SELECT MIN(price) AS cheapest, MAX(price) AS priciest FROM phones;
```
**Output:**
```
 cheapest | priciest
----------+----------
      199 |      399
(1 row)
```

**Aggregate a calculation (topic 8):**
```sql
SELECT SUM(price * units_sold) AS total_revenue FROM phones;
```
**Output:**
```
 total_revenue
---------------
       5449630
(1 row)
```

**Several aggregates in one query:**
```sql
SELECT
  COUNT(*) AS phones,
  MIN(price) AS cheapest,
  MAX(price) AS priciest,
  SUM(units_sold) AS total_units
FROM phones;
```
**Output:**
```
 phones | cheapest | priciest | total_units
--------+----------+----------+-------------
      4 |      199 |      399 |       16105
(1 row)
```

**Two rules you'll hit quickly:**

**1. You can't mix an aggregate with a plain column** (without `GROUP BY`):
```sql
SELECT name, MAX(price) FROM phones;
```
**Output:**
```
ERROR:  column "phones.name" must appear in the GROUP BY clause or be used in an aggregate function
```
`MAX(price)` is one value, but `name` has four. Which name would Postgres pick?

**2. You can't use an aggregate in `WHERE`:**
```sql
SELECT * FROM phones WHERE price > AVG(price);
```
**Output:**
```
ERROR:  aggregate functions are not allowed in WHERE
```
`WHERE` runs **before** anything is summarized. Filtering **groups** is the job of `HAVING` (topic 56).

**`MIN` and `MAX` also work on text and dates:** `MAX(created_at)` gives the newest row's date.

> ⚠️ **Warning:** `COUNT(*)` counts rows. `COUNT(column)` **skips `NULL`s**. They can give different answers (topic 53).

> ⚠️ **Warning:** `SUM`, `AVG`, `MIN`, and `MAX` return `NULL` if there are **no non-NULL values** to work on.

> ⚠️ **Warning:** `AVG` of integers returns a long `numeric`. Use `ROUND(..., 2)` for a readable result.

> 🏆 **Golden Rule:** **An aggregate turns a column into one value.** Use it alone for a whole-table summary, or with `GROUP BY` for one summary per bucket.



## 🤝 52. Combining Group By and Aggregates

**What it does:** Uses `GROUP BY` to make the buckets and an **aggregate function** to summarize **each bucket**, giving one summary row per group.
**Why it is used:** This is the real power of both tools. "How many photos **per user**?" and "total units **per manufacturer**?" need both.
**When to use it:** Any per-category summary: per user, per product, per month, per country.

**Analogy:** Sort the test papers into piles by class (`GROUP BY`), then calculate each pile's average (the aggregate).

**Photos per user:**
```sql
SELECT user_id, COUNT(*) AS photo_count
FROM photos
GROUP BY user_id;
```
**Output (row order may differ):**
```
 user_id | photo_count
---------+-------------
         |           1
       1 |           3
       2 |           1
       3 |           1
       4 |           2
(5 rows)
```

**Phones per manufacturer, with two aggregates:**
```sql
SELECT
  manufacturer,
  SUM(units_sold) AS total_units,
  ROUND(AVG(price), 2) AS avg_price
FROM phones
GROUP BY manufacturer;
```
**Output:**
```
 manufacturer | total_units | avg_price
--------------+-------------+-----------
 Nokia        |        1925 |    199.00
 Apple        |        9436 |    399.00
 Samsung      |        4744 |    274.50
(3 rows)
```

**Revenue per manufacturer (aggregate a calculation, topic 8):**
```sql
SELECT manufacturer, SUM(price * units_sold) AS revenue
FROM phones
GROUP BY manufacturer;
```
**Output:**
```
 manufacturer | revenue
--------------+---------
 Nokia        |  383075
 Apple        | 3764964
 Samsung      | 1301591
(3 rows)
```

**How Postgres computed Samsung's row:**
```
Bucket manufacturer = 'Samsung'
  Galaxy S5     price 299   units 2359
  S5620 Monte   price 250   units 2385
                            ───────────
SUM(units_sold)  = 2359 + 2385      = 4744
AVG(price)       = (299 + 250) / 2  = 274.50
```

**The pattern:**
```sql
SELECT <grouping column>, <aggregate(s)>
FROM <table>
GROUP BY <grouping column>;
```

**The matching rule:** every column in `SELECT` must be **either** in `GROUP BY` **or** inside an aggregate.

| Column in `SELECT` | In `GROUP BY` or an aggregate? | Result |
|---|---|---|
| `manufacturer` | In `GROUP BY` | ✅ |
| `SUM(units_sold)` | An aggregate | ✅ |
| `name` | Neither | ❌ Error |

**Common mistakes:**

| Mistake | What happens |
|---|---|
| Forgetting `GROUP BY` | `ERROR: column ... must appear in the GROUP BY clause or be used in an aggregate function` |
| Grouping by the wrong column | No error, but the **wrong** summary |
| Using `WHERE` to filter on an aggregate | `ERROR: aggregate functions are not allowed in WHERE` |

> ⚠️ **Warning:** Postgres can't tell you whether you grouped by the **right** column. Check a result by hand once, like the Samsung example above.

> ⚠️ **Warning:** A name or label may not be unique. Grouping by `username` would merge two users who share the same name. Group by an id when uniqueness matters (topic 55).

> 🏆 **Golden Rule:** **`SELECT` the grouping column plus aggregates, and `GROUP BY` the grouping column.** Those two lists should line up.