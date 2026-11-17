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


## 📌 63. A Few Notes on Union

**What it does:** The rules and details you need to use `UNION` correctly: matching columns, matching types, naming, and sorting the final result.
**Why it is used:** Most `UNION` errors come from breaking one of a few simple rules. Knowing them turns the error messages into obvious fixes.
**When to use it:** As a checklist whenever a set operation fails or returns something unexpected. These rules apply to `INTERSECT` and `EXCEPT` too (topics 64 and 65).

**Analogy:** Stacking two sheets of paper only lines up if both sheets have the **same number of columns**, and each column holds the **same kind of thing**.

### Rule 1: both queries need the same number of columns
```sql
SELECT name, price FROM phones
UNION
SELECT name FROM phones;
```
**Output:**
```
ERROR:  each UNION query must have the same number of columns
LINE 3: SELECT name FROM phones;
               ^
```

### Rule 2: matching columns need compatible types
```sql
SELECT name FROM phones
UNION
SELECT price FROM phones;
```
**Output:**
```
ERROR:  UNION types character varying and integer cannot be matched
```
You can't stack text on top of numbers in the same column.

### Rule 3: column names come from the **first** query
```sql
SELECT name AS item FROM phones WHERE manufacturer = 'Apple'
UNION
SELECT url FROM photos WHERE id = 1;
```
**Output (row order may differ):**
```
      item
----------------
 iPhone 5S
 http://one.jpg
(2 rows)
```
The column is called `item`, from the first query. Names in the second query are ignored.

### Rule 4: `ORDER BY` at the end sorts the **whole** result
```sql
SELECT user_id FROM photos WHERE user_id IS NOT NULL
UNION
SELECT user_id FROM comments
ORDER BY user_id DESC;
```
**Output:**
```
 user_id
---------
       4
       3
       2
       1
(4 rows)
```
Because the final `ORDER BY` has no parentheses, it applies to the combined list. It can only refer to the **output column names** (or positions).

### Rule 5: parentheses give each side its own `ORDER BY` / `LIMIT`
This is the pattern from topic 62:
```sql
(SELECT ... ORDER BY ... LIMIT 2)
UNION
(SELECT ... ORDER BY ... LIMIT 2);
```

### Rule 6: `NULL`s count as equal when removing duplicates
Two `NULL`s in the same column are treated as one value, so `UNION` keeps just one. (This differs from `WHERE`, where `NULL = NULL` is never true.)

### Often there is a simpler way
If both sides read the **same table**, one `WHERE` with `OR` may do the job:
```sql
SELECT name FROM phones WHERE price > 300 OR units_sold > 2380;
```
Use `UNION` when the sides come from **different tables**, or when each side needs its **own sort or limit**.

**Checklist for a failing `UNION`:**

| Symptom | Likely cause |
|---|---|
| `each UNION query must have the same number of columns` | Different column counts |
| `UNION types ... cannot be matched` | Incompatible types in the same position |
| `syntax error at or near "UNION"` | Missing parentheses around a side with `ORDER BY` / `LIMIT` |
| Fewer rows than expected | `UNION` removed duplicates. Try `UNION ALL` |
| Odd column names | They come from the **first** query |

> ⚠️ **Warning:** Columns are matched by **position**, not by name. Putting `name, price` in one query and `price, name` in the other mixes the two silently, if the types happen to be compatible.

> ⚠️ **Warning:** `UNION` has to find duplicates (by sorting or hashing), which costs time on large results. Use `UNION ALL` when you can.

> ⚠️ **Warning:** `ORDER BY` after the last query sorts everything. If you wanted to sort just the last query, wrap it in parentheses.

> 🏆 **Golden Rule:** **Same number of columns, compatible types, matched by position.** Parenthesize any side that has its own `ORDER BY` or `LIMIT`, and put a final `ORDER BY` at the very end for the whole result.