## 🧲 62. Handling Sets with Union

**What it does:** `UNION` **stacks the results of two queries on top of each other** into one list, and removes duplicate rows. `UNION ALL` stacks them but **keeps** duplicates.
**Why it is used:** Some questions are really "this group **plus** that group," where each group needs its own query, its own sort, or its own limit.
**When to use it:** Combining two top-N lists, merging similar data from different tables, building one list from several sources.

**Analogy:** Two guest lists, one for the wedding and one for the party. A **union** is the combined invite list, with anyone who appears on both written **once**. `UNION ALL` just staples the lists together, so people on both lists appear twice.

**A join places tables side by side. A union stacks them vertically:**
```
JOIN:   [ A columns ][ B columns ]     (wider)
UNION:  [ A rows ]
        [ B rows ]                     (taller, same columns)
```

**Example: the 2 most expensive phones plus the 2 best-selling phones.**
```sql
(SELECT name, price, units_sold FROM phones ORDER BY price DESC LIMIT 2)
UNION
(SELECT name, price, units_sold FROM phones ORDER BY units_sold DESC LIMIT 2);
```
**Output (row order may differ):**
```
    name     | price | units_sold
-------------+-------+------------
 iPhone 5S   |   399 |       9436
 Galaxy S5   |   299 |       2359
 S5620 Monte |   250 |       2385
(3 rows)
```

**Where the rows came from:**

| Query | Rows it returned |
|---|---|
| Top 2 by price | iPhone 5S, Galaxy S5 |
| Top 2 by units sold | iPhone 5S, S5620 Monte |
| **Union** (duplicates removed) | iPhone 5S, Galaxy S5, S5620 Monte |

iPhone 5S qualified for **both**, but appears **once**.

**The same query with `UNION ALL`:**
```sql
(SELECT name, price, units_sold FROM phones ORDER BY price DESC LIMIT 2)
UNION ALL
(SELECT name, price, units_sold FROM phones ORDER BY units_sold DESC LIMIT 2);
```
**Output (row order may differ):**
```
    name     | price | units_sold
-------------+-------+------------
 iPhone 5S   |   399 |       9436
 Galaxy S5   |   299 |       2359
 iPhone 5S   |   399 |       9436
 S5620 Monte |   250 |       2385
(4 rows)
```
`iPhone 5S` now appears **twice**.

**Combining two different tables.** Which user ids appear in photos or comments?
```sql
SELECT user_id FROM photos WHERE user_id IS NOT NULL
UNION
SELECT user_id FROM comments;
```
**Output (row order may differ):**
```
 user_id
---------
       1
       2
       3
       4
(4 rows)
```
`UNION ALL` on the same pair gives **17 rows** (7 non-null photo owners + 10 comments), duplicates included.

**`UNION` vs `UNION ALL`:**

| | `UNION` | `UNION ALL` |
|---|---|---|
| **Duplicates** | Removed | Kept |
| **Speed** | Slower (must find duplicates) | Faster |
| **Use when** | You want each row once | You know there are no overlaps, or you want every row |

**Why the parentheses in the phones example?** Each side has its own `ORDER BY` and `LIMIT`. Without parentheses Postgres can't tell which query they belong to:
```sql
SELECT name FROM phones ORDER BY price LIMIT 1
UNION
SELECT name FROM phones ORDER BY units_sold LIMIT 1;
```
**Output:**
```
ERROR:  syntax error at or near "UNION"
LINE 3: UNION
        ^
```

> ⚠️ **Warning:** **`UNION` silently removes duplicate rows.** If you were counting on every row appearing, use `UNION ALL`.

> ⚠️ **Warning:** The order of the combined result is **not guaranteed**. Add an `ORDER BY` at the very end if you need one (topic 63).

> ⚠️ **Warning:** Wrap each side in parentheses whenever it has its own `ORDER BY` or `LIMIT`.

> 🏆 **Golden Rule:** **`UNION` = stack and de-duplicate. `UNION ALL` = just stack.** Prefer `UNION ALL` when duplicates are impossible or acceptable, because it's cheaper.

