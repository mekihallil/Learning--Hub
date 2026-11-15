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


## ⚠️ 53. A Gotcha with Count

**What it does:** Explains that `COUNT(*)` and `COUNT(column)` can give **different answers**, and shows the classic bug it causes with `LEFT JOIN` and `GROUP BY`.
**Why it is used:** The bug is silent. The query runs, the numbers look reasonable, and a user who has **zero** photos is reported as having **one**.
**When to use it:** Any time you count rows from the "many" side of an outer join.

**Analogy:** A teacher counting "how many students handed in homework." `COUNT(*)` counts **every name on the roster**. `COUNT(homework)` counts only the names with **an actual paper**. Different questions, different answers.

**The three counts:**

| Expression | Counts | Skips `NULL`? |
|---|---|---|
| `COUNT(*)` | Every **row** | No |
| `COUNT(column)` | Rows where the column is **not `NULL`** | ✅ Yes |
| `COUNT(DISTINCT column)` | **Different** non-`NULL` values | ✅ Yes |

**Seeing the difference on `photos`:**
```sql
SELECT
  COUNT(*) AS all_rows,
  COUNT(user_id) AS with_owner,
  COUNT(DISTINCT user_id) AS different_owners
FROM photos;
```
**Output:**
```
 all_rows | with_owner | different_owners
----------+------------+------------------
        8 |          7 |                4
(1 row)
```
8 rows, but only 7 have an owner (`lost.jpg` doesn't), and those 7 belong to 4 different users.

### The bug: counting after a LEFT JOIN
**Goal:** how many photos does **each user** have, including users with none?
```sql
SELECT u.username, COUNT(*) AS photo_count
FROM users AS u
LEFT JOIN photos AS p ON p.user_id = u.id
GROUP BY u.username;
```
**Output (wrong!):**
```
  username  | photo_count
------------+-------------
 monahan93  |           3
 pferrer    |           1
 si93onis   |           1
 99stroman  |           2
 lonelyuser |           1
(5 rows)
```
`lonelyuser` has **no** photos, yet shows `1`.

**Why:** a `LEFT JOIN` keeps `lonelyuser` as a row with **`NULL`** in every photo column (topic 42). `COUNT(*)` counts that row.

**The fix: count a column from the other table.**
```sql
SELECT u.username, COUNT(p.id) AS photo_count
FROM users AS u
LEFT JOIN photos AS p ON p.user_id = u.id
GROUP BY u.username;
```
**Output (correct):**
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
`p.id` is `NULL` for the unmatched row, so `COUNT(p.id)` skips it and returns `0`.

**What each version counts for `lonelyuser`:**

| Expression | Their joined row | Result |
|---|---|---|
| `COUNT(*)` | `(5, 'lonelyuser', NULL, NULL, NULL)` is still a row | **1** ❌ |
| `COUNT(p.id)` | `p.id` is `NULL`, skipped | **0** ✅ |

**Also remember: an `INNER JOIN` would drop `lonelyuser` completely**, so they'd be missing from the results rather than showing `0`.

**Which count to choose:**

| You want... | Use |
|---|---|
| The number of rows in a table or group | `COUNT(*)` |
| The number of rows that **have a value** in a column | `COUNT(column)` |
| After a `LEFT JOIN`, the number of **matches** | `COUNT(right_table.primary_key)` |
| How many **different** values | `COUNT(DISTINCT column)` |

> ⚠️ **Warning:** **After a `LEFT JOIN`, `COUNT(*)` counts the unmatched row too.** Count the other table's primary key instead.

> ⚠️ **Warning:** Count a column that is **never `NULL` in a real row**, such as the primary key. Counting a column that is sometimes `NULL` gives a misleading answer.

> 🏆 **Golden Rule:** **`COUNT(*)` counts rows. `COUNT(column)` counts values.** When a `LEFT JOIN` is involved, `COUNT(other_table.id)` is the safe choice.


## 🧺 54. Visualizing More Grouping

**What it does:** Looks at grouping more closely. You'll see what happens with **different grouping columns** and with **more than one grouping column**, using the `comments` table.
**Why it is used:** Changing what you group by changes the question you're answering. Picturing the buckets tells you which grouping to pick.
**When to use it:** Whenever you need a different "angle" on the same table, such as per photo, per user, or per combination.

**Analogy:** The same pile of receipts can be sorted **by store**, **by month**, or **by store and month together**. Each way of sorting answers a different question.

**First, add more comments so the groups have something to show:**
```sql
INSERT INTO comments (contents, user_id, photo_id)
VALUES
  ('Great colors', 2, 2),
  ('Amazing', 3, 2),
  ('Wow', 4, 5),
  ('So cool', 2, 5),
  ('Lovely', 2, 3),
  ('Seen it twice', 2, 2);
```
**Output:**
```
INSERT 0 6
```
**The data for topics 54 to 58:**
```sql
SELECT * FROM comments;
```
**Output:**
```
 id |       contents        | user_id | photo_id
----+-----------------------+---------+----------
  1 | Nice shot!            |       2 |        1
  2 | Love this             |       3 |        1
  3 | Where was this taken? |       4 |        2
  4 | Beautiful             |       1 |        5
  5 | Great colors          |       2 |        2
  6 | Amazing               |       3 |        2
  7 | Wow                   |       4 |        5
  8 | So cool               |       2 |        5
  9 | Lovely                |       2 |        3
 10 | Seen it twice         |       2 |        2
(10 rows)
```

### Grouping by one column: `photo_id`
```
Bucket photo 1: comments 1, 2             → 2
Bucket photo 2: comments 3, 5, 6, 10      → 4
Bucket photo 3: comment 9                 → 1
Bucket photo 5: comments 4, 7, 8          → 3
```
```sql
SELECT photo_id, COUNT(*) AS comment_count
FROM comments
GROUP BY photo_id;
```
**Output (row order may differ):**
```
 photo_id | comment_count
----------+---------------
        1 |             2
        2 |             4
        3 |             1
        5 |             3
(4 rows)
```
Photos 4, 6, 7, and 8 have no comments, so they have **no bucket** at all.

### Grouping by a different column: `user_id`
```sql
SELECT user_id, COUNT(*) AS comments_written
FROM comments
GROUP BY user_id;
```
**Output:**
```
 user_id | comments_written
---------+------------------
       1 |                1
       2 |                5
       3 |                2
       4 |                2
(4 rows)
```
Same table, but a different question: **who** comments, not **what** gets commented on.

### Grouping by two columns
A bucket is now a **unique combination** of both values.
```sql
SELECT photo_id, user_id, COUNT(*) AS comment_count
FROM comments
GROUP BY photo_id, user_id;
```
**Output (row order may differ):**
```
 photo_id | user_id | comment_count
----------+---------+---------------
        1 |       2 |             1
        1 |       3 |             1
        2 |       2 |             2
        2 |       3 |             1
        2 |       4 |             1
        3 |       2 |             1
        5 |       1 |             1
        5 |       2 |             1
        5 |       4 |             1
(9 rows)
```
The row `(photo 2, user 2)` has a count of **2**. That user commented twice on photo 2 (comments 5 and 10).

**More columns means smaller buckets:**

| Grouped by | Number of groups | What each group means |
|---|---|---|
| `photo_id` | 4 | All comments on one photo |
| `user_id` | 4 | All comments by one user |
| `photo_id, user_id` | 9 | All comments by one user **on one photo** |

**You can group by a calculation too:**
```sql
SELECT manufacturer, price > 250 AS above_250, COUNT(*) AS phones
FROM phones
GROUP BY manufacturer, price > 250;
```
**Output:**
```
 manufacturer | above_250 | phones
--------------+-----------+--------
 Nokia        | f         |      1
 Apple        | t         |      1
 Samsung      | t         |      1
 Samsung      | f         |      1
(4 rows)
```

**Reminder of the rule (topic 50):** everything in `SELECT` must be in `GROUP BY` or inside an aggregate. With two grouping columns, **both** are allowed in `SELECT`.

> ⚠️ **Warning:** Adding a grouping column can **quietly change the meaning** of your query. If a count looks too small, check whether you grouped by too many columns.

> ⚠️ **Warning:** The inserted comments change your data. Rerun the reset script from topic 36 if you want the original 4 comments back.

> 🏆 **Golden Rule:** **Each grouping column makes buckets more specific.** Group by exactly the columns that define the question you're asking.



## 🧗 55. Adding a Layer of Difficulty

**What it does:** Combines **joins** with **grouping and aggregates**, so you can summarize data that is spread across several tables, such as comments per username or comments received per photo owner.
**Why it is used:** Ids like `user_id` are meaningless in a report. Joins bring in the **names** and **URLs** people actually want to read.
**When to use it:** Almost every real report needs this combination.

**Analogy:** First you **look up each name tag** in the guest list (the join), then you **count the guests at each table** (the grouping).

**The method (in this order):**

| Step | Action | Check |
|---|---|---|
| 1 | Write the join **without** grouping | Do the rows look right? |
| 2 | Decide what to **group by** | Which column defines a bucket? |
| 3 | Add the **aggregate** | One row per bucket? |

**Step 1: look at the plain join first:**
```sql
SELECT u.username, c.contents
FROM comments AS c
JOIN users AS u ON u.id = c.user_id;
```
**Output:** 10 rows, one per comment, each with its author's name.

**Step 2 and 3: group and count:**
```sql
SELECT u.username, COUNT(*) AS comments_written
FROM comments AS c
JOIN users AS u ON u.id = c.user_id
GROUP BY u.username;
```
**Output (row order may differ):**
```
 username  | comments_written
-----------+------------------
 monahan93 |                1
 pferrer   |                5
 si93onis  |                2
 99stroman |                2
(4 rows)
```

**Comments per photo URL:**
```sql
SELECT p.url, COUNT(*) AS comment_count
FROM comments AS c
JOIN photos AS p ON p.id = c.photo_id
GROUP BY p.url;
```
**Output:**
```
      url       | comment_count
----------------+---------------
 http://one.jpg |             2
 http://two.jpg |             4
 http://25.jpg  |             1
 http://754.jpg |             3
(4 rows)
```

**Include users with zero comments** (use a `LEFT JOIN` and `COUNT(c.id)`, from topic 53):
```sql
SELECT u.username, COUNT(c.id) AS comments_written
FROM users AS u
LEFT JOIN comments AS c ON c.user_id = u.id
GROUP BY u.username;
```
**Output:**
```
  username  | comments_written
------------+------------------
 monahan93  |                1
 pferrer    |                5
 si93onis   |                2
 99stroman  |                2
 lonelyuser |                0
(5 rows)
```

**A harder one: comments received by each photo owner (three tables):**
```sql
SELECT owner.username, COUNT(c.id) AS comments_received
FROM users AS owner
LEFT JOIN photos AS p ON p.user_id = owner.id
LEFT JOIN comments AS c ON c.photo_id = p.id
GROUP BY owner.id;
```
**Output:**
```
  username  | comments_received
------------+-------------------
 monahan93  |                 5
 pferrer    |                 3
 si93onis   |                 0
 99stroman  |                 2
 lonelyuser |                 0
(5 rows)
```
**Sanity check:** 5 + 3 + 0 + 2 + 0 = **10**, the total number of comments. ✅

**Two things in that query worth noting:**

| Detail | Explanation |
|---|---|
| `GROUP BY owner.id` instead of `owner.username` | Usernames **aren't guaranteed unique**. The id is, so it's the safer grouping column |
| `SELECT owner.username` is still allowed | `users.id` is a **primary key**, so PostgreSQL knows each id has exactly one username |

**Both `LEFT JOIN`s are deliberate.** Making the second one a plain `JOIN` would drop users whose photos have no comments, because the unmatched rows would be removed (topic 48).

**Sanity-check habit:** the totals should add up to something you can verify.

| Check | Expected |
|---|---|
| Sum of the per-user comment counts | 10 (total comments) |
| Number of rows after a `LEFT JOIN` from `users` | At least 5 (one per user) |

> ⚠️ **Warning:** A join before grouping can **multiply rows**. If a number looks too big, inspect the un-grouped join result first.

> ⚠️ **Warning:** After a `LEFT JOIN`, count a column from the **right** table (`COUNT(c.id)`), not `COUNT(*)` (topic 53).

> ⚠️ **Warning:** Grouping by a non-unique column (like `username`) can silently merge different people.

> 🏆 **Golden Rule:** **Join first, group second.** Get the joined rows right, then collapse them with `GROUP BY` and an aggregate.



## 🚪 56. Filtering Groups with Having

**What it does:** `HAVING` filters **whole groups** after they've been formed and summarized. `WHERE` filters **individual rows** before grouping.
**Why it is used:** You can't use an aggregate inside `WHERE` (topic 51). To ask "which users have **more than 1 photo**?" you need a filter that can see the finished counts.
**When to use it:** Any time the condition depends on an aggregate: "more than 3," "total over 1000," "average under 5."

**Analogy:** A restaurant. The **host at the door** (`WHERE`) decides which **individual people** may come in. After everyone is seated at tables (`GROUP BY`), the **manager** (`HAVING`) closes any **table** with fewer than 2 guests.

**Users with more than 1 photo:**
```sql
SELECT user_id, COUNT(*) AS photo_count
FROM photos
GROUP BY user_id
HAVING COUNT(*) > 1;
```
**Output:**
```
 user_id | photo_count
---------+-------------
       1 |           3
       4 |           2
(2 rows)
```
The groups with a count of 1 (users 2 and 3, and the `NULL` group) were removed.

**Why `WHERE` can't do this:**
```sql
SELECT user_id, COUNT(*)
FROM photos
WHERE COUNT(*) > 1
GROUP BY user_id;
```
**Output:**
```
ERROR:  aggregate functions are not allowed in WHERE
LINE 3: WHERE COUNT(*) > 1
```
When `WHERE` runs, **no groups exist yet**, so there's nothing to count.

**The order Postgres works in:**

| Step | Clause | What happens |
|---|---|---|
| 1 | `FROM` / `JOIN` | Build the rows |
| 2 | `WHERE` | Remove **individual rows** |
| 3 | `GROUP BY` | Sort the remaining rows into buckets |
| 4 | Aggregates | Calculate each bucket's `COUNT`, `SUM`, and so on |
| 5 | `HAVING` | Remove **whole buckets** |
| 6 | `SELECT` | Choose the columns to show |

The order you **write** them is `SELECT`, `FROM`, `WHERE`, `GROUP BY`, `HAVING`.

**`WHERE` vs `HAVING`:**

| | `WHERE` | `HAVING` |
|---|---|---|
| **Filters** | Individual **rows** | Whole **groups** |
| **Runs** | **Before** grouping | **After** grouping |
| **Can use aggregates?** | ❌ No | ✅ Yes |
| **Can use plain columns?** | ✅ Yes | Only grouped columns |
| **Example** | `WHERE user_id IS NOT NULL` | `HAVING COUNT(*) > 1` |

**Using both together:**
```sql
SELECT user_id, COUNT(*) AS photo_count
FROM photos
WHERE user_id IS NOT NULL
GROUP BY user_id
HAVING COUNT(*) > 1;
```
**Output:**
```
 user_id | photo_count
---------+-------------
       1 |           3
       4 |           2
(2 rows)
```
`WHERE` first removed the orphan photo (a row), then `HAVING` removed the small groups.

> ⚠️ **Warning:** `HAVING` without a working `GROUP BY` mindset is confusing. Think: **group first, then filter the groups**.

> ⚠️ **Warning:** A **row** condition belongs in `WHERE`, not `HAVING`. It's faster to throw rows away **before** grouping.

> 🏆 **Golden Rule:** **`WHERE` filters rows, `HAVING` filters groups.** If your condition contains an aggregate, it must go in `HAVING`.