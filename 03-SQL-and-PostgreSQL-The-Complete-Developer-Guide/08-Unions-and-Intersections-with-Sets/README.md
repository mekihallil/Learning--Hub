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



## 🤝 64. Commonalities with Intersect

**What it does:** `INTERSECT` returns only the rows that appear in **both** queries. Anything found in just one of them is thrown away.
**Why it is used:** Some questions are "which things satisfy **both** conditions?" and each condition comes from a different query or table.
**When to use it:** Finding overlap: customers who bought both products, photos that match two different searches, ids that appear in two tables.

**Analogy:** Two guest lists again, but now you want only the people **on both lists**. That's the overlap of two circles, the middle of a Venn diagram.

```
UNION       INTERSECT     EXCEPT
 (###(###)###)  ( (###) )    (###( ) )
 everything     only the     first list
 from both      overlap      minus overlap
```

**Example 1: phones that are both pricey and popular.**
```sql
SELECT name FROM phones WHERE price > 250
INTERSECT
SELECT name FROM phones WHERE units_sold > 2000;
```
**Output (row order may differ):**
```
   name
-----------
 iPhone 5S
 Galaxy S5
(2 rows)
```

| Query | Rows it returned |
|---|---|
| Price over 250 | iPhone 5S (399), Galaxy S5 (299) |
| Units sold over 2000 | iPhone 5S (9436), Galaxy S5 (2359), S5620 Monte (2385) |
| **Intersect** | iPhone 5S, Galaxy S5 |

S5620 Monte sold well, but costs exactly 250, so it fails the first test.

**Example 2: photos by user 1 that also have comments.** The two lists come from **different tables**:
```sql
SELECT id FROM photos WHERE user_id = 1
INTERSECT
SELECT photo_id FROM comments;
```
**Output (row order may differ):**
```
 id
----
  2
  3
(2 rows)
```

| Query | Rows it returned |
|---|---|
| Photos owned by user 1 | 2, 3, 4 |
| Photos that have any comment | 1, 2, 3, 5 |
| **Intersect** | 2, 3 |

Photo 4 has no comments, and photos 1 and 5 belong to other users.

**`INTERSECT ALL` keeps duplicates.** For each value it keeps the **smaller** of its two counts:
```sql
SELECT user_id FROM comments
INTERSECT ALL
SELECT user_id FROM photos WHERE user_id IS NOT NULL;
```
**Output (row order may differ):**
```
 user_id
---------
       1
       2
       3
       4
       4
(5 rows)
```

| user_id | Times in `comments` | Times in `photos` | Kept by `INTERSECT ALL` |
|---|---|---|---|
| 1 | 1 | 3 | **1** |
| 2 | 5 | 1 | **1** |
| 3 | 2 | 1 | **1** |
| 4 | 2 | 2 | **2** |

Plain `INTERSECT` would return just `1, 2, 3, 4`.

**Order doesn't matter for `INTERSECT`.** Swapping the two queries gives the same rows:
```sql
SELECT photo_id FROM comments
INTERSECT
SELECT id FROM photos WHERE user_id = 1;
```
**Output:** `2` and `3` again. (The column is now named `photo_id`, because names come from the first query, topic 63.)

**The same result with other tools:**

| Approach | Query ends with |
|---|---|
| `INTERSECT` | `... INTERSECT SELECT photo_id FROM comments` |
| `IN` (topic 15) | `WHERE user_id = 1 AND id IN (SELECT photo_id FROM comments)` |
| `JOIN` | `JOIN comments ON comments.photo_id = photos.id`, which needs `DISTINCT` to avoid repeats |

```sql
SELECT id FROM photos
WHERE user_id = 1 AND id IN (SELECT photo_id FROM comments);
```
**Output:** `2` and `3`. (That inner `SELECT` is a **subquery**, a bigger topic that comes later.)

`INTERSECT` reads nicely when both sides are **complete queries** that you can test separately.

**The rules are the same as `UNION` (topic 63):**

| Rule | Meaning |
|---|---|
| Same number of columns | Both queries return the same count of columns |
| Compatible types | Matching positions must hold comparable types |
| Matched by **position** | Not by column name |
| Names come from the **first** query | Later names are ignored |
| `NULL`s count as equal | Two `NULL`s match each other here |

**A note on mixing set operators:** `INTERSECT` binds **more tightly** than `UNION` and `EXCEPT`. So `A UNION B INTERSECT C` means `A UNION (B INTERSECT C)`. Add parentheses whenever you mix them.

> ⚠️ **Warning:** **`INTERSECT` removes duplicates** just like `UNION`. Use `INTERSECT ALL` if the counts matter.

> ⚠️ **Warning:** An empty overlap is **not an error**. If nothing is shared you simply get `(0 rows)`, so check the two queries separately if you expected results.

> ⚠️ **Warning:** `NULL = NULL` is never true in a `WHERE` (topic 11), but `INTERSECT` **does** treat two `NULL`s as matching. That surprises people who switch between the two approaches.

> 🏆 **Golden Rule:** **`INTERSECT` = rows present in both queries.** Check each side on its own first, then intersect them.