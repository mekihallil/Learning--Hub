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



## 📝 80. Exercise Overview

**What it does:** A four-task practice exercise on `DISTINCT`, `COUNT(DISTINCT ...)`, and `DISTINCT ON`.
**Why it is used:** The difference between "all rows," "different rows," and "one row per group" only sticks once you've written the queries yourself.
**When to use it:** After topic 79. Try all four before reading the solutions in topic 81.

**Tables to use:** `phones` (manufacturer, name, price) and `comments` (photo_id, user_id, contents).

**Task 1.** Print each **different** manufacturer, sorted alphabetically.

**Expected output:**
```
 manufacturer
--------------
 Apple
 Nokia
 Samsung
(3 rows)
```

**Task 2.** Print **how many different users** have written a comment.

**Expected output:**
```
 commenters
------------
          4
(1 row)
```

**Task 3.** Print **how many different (photo, user) pairs** exist in `comments`. In other words, count each user-on-photo combination once, however many times that user commented on that photo.

**Expected output:**
```
 pairs
-------
     9
(1 row)
```

**Task 4 (stretch).** For each manufacturer, print the name and price of its **cheapest** phone.

**Expected output:**
```
 manufacturer |    name     | price
--------------+-------------+-------
 Apple        | iPhone 5S   |   399
 Nokia        | N1280       |   199
 Samsung      | S5620 Monte |   250
(3 rows)
```

**Hints:**

| Task | Hint |
|---|---|
| 1 | Use `DISTINCT`, then `ORDER BY` (topic 59) |
| 2 | Put `DISTINCT` **inside** the aggregate |
| 3 | You can't put two columns inside `COUNT(DISTINCT ...)` the simple way. Think about a subquery in `FROM` (topic 69) |
| 4 | Look at `DISTINCT ON` and which direction to sort the price |

> ⚠️ **Warning:** Check Task 3 by hand against the list in topic 54: the (photo, user) pairs are (1,2), (1,3), (2,2), (2,3), (2,4), (3,2), (5,1), (5,2), (5,4).

> 🏆 **Golden Rule:** **Ask what each row of your answer should represent.** "One row per manufacturer" points to `DISTINCT`, `DISTINCT ON`, or `GROUP BY`.


## ✅ 81. A Distinct Solution

**What it does:** The worked answers to the exercise in topic 80, with an explanation of each.
**Why it is used:** Comparing your attempt with a clean solution shows where your thinking differed.
**When to use it:** After you've tried the four tasks.

**Task 1: different manufacturers, sorted**
```sql
SELECT DISTINCT manufacturer
FROM phones
ORDER BY manufacturer;
```
**Output:**
```
 manufacturer
--------------
 Apple
 Nokia
 Samsung
(3 rows)
```

**Task 2: how many different commenters**
```sql
SELECT COUNT(DISTINCT user_id) AS commenters
FROM comments;
```
**Output:**
```
 commenters
------------
          4
(1 row)
```
`COUNT(*)` would say `10` (every comment). The `DISTINCT` inside the aggregate counts each user once.

**Task 3: how many different (photo, user) pairs**
```sql
SELECT COUNT(*) AS pairs
FROM (
  SELECT DISTINCT photo_id, user_id
  FROM comments
) AS unique_pairs;
```
**Output:**
```
 pairs
-------
     9
(1 row)
```
The inner query removes the repeat (user 2 on photo 2), leaving 9 rows, and the outer query counts them. This is the "subquery in `FROM`" pattern from topic 69.

PostgreSQL also lets you pass a row to the aggregate directly:
```sql
SELECT COUNT(DISTINCT (photo_id, user_id)) AS pairs FROM comments;
```
**Output:** `9`. The subquery version works in every database, so it's the safer one to learn first.

**Task 4: the cheapest phone per manufacturer**
```sql
SELECT DISTINCT ON (manufacturer) manufacturer, name, price
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
(3 rows)
```
`ORDER BY manufacturer, price` puts each manufacturer's **cheapest** phone first, and `DISTINCT ON` keeps only that first row. (Topic 79 sorted `price DESC` to get the priciest.)

**Common mistakes:**

| Mistake | What happens |
|---|---|
| Task 2 as `SELECT DISTINCT COUNT(user_id) ...` | Returns `10`. `DISTINCT` applies **after** counting, to the single result row |
| Task 3 as `SELECT DISTINCT COUNT(photo_id, user_id)` | `ERROR: function count(integer, integer) does not exist` |
| Task 4 with `ORDER BY price` only | `ERROR: SELECT DISTINCT ON expressions must match initial ORDER BY expressions` |
| Task 4 with `ORDER BY manufacturer` only | No error, but which Samsung phone survives is **arbitrary** |

**Sanity checks:** Task 1 should have **3** rows (3 manufacturers), Task 3's answer should be **at most 10** (the number of comments), and Task 4 should have **one row per manufacturer**.

> ⚠️ **Warning:** `DISTINCT ON` is **PostgreSQL-specific**. Other databases need a different approach, such as a window function or a correlated subquery (topic 75).

> ⚠️ **Warning:** The `ORDER BY` of a `DISTINCT ON` query **must start with the same expression** as the `ON (...)`. After that you choose how ties are broken.

> 🏆 **Golden Rule:** **`DISTINCT` inside an aggregate counts different values. `DISTINCT ON` plus a matching `ORDER BY` picks one row per group.**




## 🔝 82. The Greatest Value in a List

**What it does:** `GREATEST(a, b, c, ...)` returns the **largest** of the values you give it, **within a single row**.
**Why it is used:** It sets a **floor**: "never let this value drop below X." It also compares several columns of the same row.
**When to use it:** Preventing negative numbers, enforcing a minimum price, or picking the larger of two columns.

**Analogy:** A bouncer who says "you must be **at least** this tall." Whatever your height, the result is never below the line.

**The basics:**
```sql
SELECT GREATEST(20, 10, 30) AS biggest;
```
**Output:**
```
 biggest
---------
      30
(1 row)
```
It works on text and dates too:
```sql
SELECT GREATEST('apple', 'cherry', 'banana') AS last_alphabetically;
```
**Output:**
```
 last_alphabetically
---------------------
 cherry
(1 row)
```

### A floor at zero
**How many units did each phone sell above 2,000? Never show a negative number.**
```sql
SELECT name, units_sold, GREATEST(units_sold - 2000, 0) AS above_2000
FROM phones;
```
**Output:**
```
    name     | units_sold | above_2000
-------------+------------+------------
 N1280       |       1925 |          0
 iPhone 5S   |       9436 |       7436
 Galaxy S5   |       2359 |        359
 S5620 Monte |       2385 |        385
(4 rows)
```
N1280 sold 1925, so `1925 - 2000 = -75`, and `GREATEST(-75, 0)` gives `0`.

### `GREATEST` vs `MAX`
They sound alike, but work in different directions:

| | `GREATEST(a, b, c)` | `MAX(column)` |
|---|---|---|
| **Compares** | Values **within one row** | Values **across many rows** |
| **Returns** | One value **per row** | One value **for the whole group or table** |
| **Type** | Ordinary function | Aggregate function (topic 51) |
| **Example** | `GREATEST(price, 250)` | `MAX(price)` |

```sql
SELECT MAX(price) AS top_price, GREATEST(MIN(price), 250) AS floor_price FROM phones;
```
**Output:**
```
 top_price | floor_price
-----------+-------------
       399 |         250
(1 row)
```
`MIN(price)` is 199, and `GREATEST(199, 250)` lifts it to 250.

### How it treats `NULL`
In PostgreSQL, `GREATEST` **ignores `NULL`s**. It returns `NULL` only when **every** value is `NULL`.
```sql
SELECT GREATEST(5, NULL, 8) AS ignores_null, GREATEST(NULL, NULL) AS all_null;
```
**Output:**
```
 ignores_null | all_null
--------------+----------
            8 |
(1 row)
```
This differs from `+` and `||`, where a single `NULL` poisons the result (topics 7 and 9).

### The values must be compatible
```sql
SELECT GREATEST(1, 'a');
```
**Output:**
```
ERROR:  invalid input syntax for type integer: "a"
```

> ⚠️ **Warning:** `GREATEST` raises a value **up to** at least the floor. People often reach for it expecting a **cap**, but a cap needs `LEAST` (topic 83).

> ⚠️ **Warning:** Other databases handle `NULL` differently (MySQL returns `NULL` if **any** argument is `NULL`). These notes are for PostgreSQL.

> ⚠️ **Warning:** All arguments must be the same kind of type, otherwise you get a conversion error.

> 🏆 **Golden Rule:** **`GREATEST(value, floor)` means "at least this much."** Use it to stop calculations from going below a limit.



## 🔻 83. And the Least Value in a List!

**What it does:** `LEAST(a, b, c, ...)` returns the **smallest** of the values you give it, **within a single row**. It's the mirror image of `GREATEST` (topic 82).
**Why it is used:** It sets a **cap**: "never let this value go above X."
**When to use it:** Limiting a discount, capping a score at 100, or picking the smaller of two columns.

**Analogy:** A ride with a **maximum** weight. Whatever you weigh, the number on the sign is never above the limit.

**The basics:**
```sql
SELECT LEAST(20, 10, 30) AS smallest;
```
**Output:**
```
 smallest
----------
       10
(1 row)
```

### A cap at 300
**Show each phone's price, but never show more than 300:**
```sql
SELECT name, price, LEAST(price, 300) AS capped_price
FROM phones;
```
**Output:**
```
    name     | price | capped_price
-------------+-------+--------------
 N1280       |   199 |          199
 iPhone 5S   |   399 |          300
 Galaxy S5   |   299 |          299
 S5620 Monte |   250 |          250
(4 rows)
```
Only the iPhone (399) exceeds the cap, so it's brought **down** to 300.

### Floor and cap together (clamping)
Wrap one inside the other to force a value into a range:
```sql
SELECT name, price, LEAST(GREATEST(price, 200), 300) AS clamped
FROM phones;
```
**Output:**
```
    name     | price | clamped
-------------+-------+---------
 N1280       |   199 |     200
 iPhone 5S   |   399 |     300
 Galaxy S5   |   299 |     299
 S5620 Monte |   250 |     250
(4 rows)
```
`GREATEST(price, 200)` lifts anything under 200 up to 200, and `LEAST(..., 300)` brings anything over 300 down to 300.

### Which one to use?
| You want... | Use | Example |
|---|---|---|
| A **minimum** value (a floor) | `GREATEST` | `GREATEST(price, 200)` |
| A **maximum** value (a cap) | `LEAST` | `LEAST(price, 300)` |
| Both | Nest them | `LEAST(GREATEST(price, 200), 300)` |

This feels backwards at first: the **floor** uses the **bigger** function, and the **cap** uses the **smaller** one. Think of it as the winner: `GREATEST(x, 200)` returns 200 whenever 200 is bigger, so the result can't go below 200.

### `LEAST` vs `MIN`
| | `LEAST(a, b, c)` | `MIN(column)` |
|---|---|---|
| **Compares** | Values within **one row** | Values across **many rows** |
| **Returns** | One value per row | One value per group or table |

### Text and `NULL`
```sql
SELECT LEAST('apple', 'cherry', 'banana') AS first_alphabetically,
       LEAST(5, NULL, 8) AS ignores_null,
       LEAST(NULL, NULL) AS all_null;
```
**Output:**
```
 first_alphabetically | ignores_null | all_null
----------------------+--------------+----------
 apple                |            5 |
(1 row)
```
Like `GREATEST`, `LEAST` **ignores `NULL`s** and returns `NULL` only if all arguments are `NULL`.

> ⚠️ **Warning:** `LEAST` and `GREATEST` are easy to mix up. Remember: `LEAST(x, 300)` can **never exceed 300**, and `GREATEST(x, 200)` can **never go below 200**.

> ⚠️ **Warning:** Ignoring `NULL`s can hide missing data. `LEAST(price, NULL)` quietly returns `price`. Check for `NULL`s if that matters.

> 🏆 **Golden Rule:** **`LEAST(value, limit)` = "at most this much."** Pair it with `GREATEST` to keep a value inside a range.