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