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