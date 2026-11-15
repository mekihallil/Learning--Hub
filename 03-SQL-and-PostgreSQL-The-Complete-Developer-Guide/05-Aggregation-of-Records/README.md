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