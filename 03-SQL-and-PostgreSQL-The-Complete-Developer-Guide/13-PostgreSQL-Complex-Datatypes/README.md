## 🔁 87. What'd We Just Do

**What it does:** A recap of everything covered so far, plus a cheat sheet for the order SQL clauses are written and the order Postgres actually runs them.
**Why it is used:** After 86 topics, it helps to zoom out. Seeing the whole toolbox in one place shows how the pieces fit together.
**When to use it:** As a checkpoint before starting the next section, and as a quick reference later.

**Analogy:** The end of a long hike where you stop at a viewpoint and look at the trail you've covered, before the next climb.

### The route so far
| Topics | What you learned | Key tools |
|---|---|---|
| 1 to 9 | Tables, inserting, selecting, calculated columns, strings | `CREATE TABLE`, `INSERT`, `SELECT`, `AS`, `\|\|`, `LOWER` |
| 10 to 19 | Filtering and changing data | `WHERE`, `AND`/`OR`, `IN`, `UPDATE`, `DELETE` |
| 20 to 35 | Database design and relationships | Primary keys, foreign keys, `SERIAL`, `ON DELETE` |
| 36 to 48 | Combining tables | `JOIN`, `LEFT`/`RIGHT`/`FULL`, three-way joins |
| 49 to 58 | Summarizing data | `COUNT`, `SUM`, `GROUP BY`, `HAVING` |
| 59 to 61 | Sorting and slicing | `ORDER BY`, `LIMIT`, `OFFSET` |
| 62 to 65 | Combining result sets | `UNION`, `INTERSECT`, `EXCEPT` |
| 66 to 78 | Queries inside queries | Subqueries, correlated subqueries, `EXISTS`, `ANY`/`ALL` |
| 79 to 84 | Handy extras | `DISTINCT`, `GREATEST`, `LEAST`, `CASE` |
| 85 to 86 | Your own workshop | Installing PostgreSQL and pgAdmin |

### The order you write a query
```sql
SELECT   columns
FROM     table
JOIN     other_table ON ...
WHERE    row conditions
GROUP BY grouping columns
HAVING   group conditions
ORDER BY sort columns
LIMIT    n OFFSET m;
```

### The order Postgres runs it
| Step | Clause | What happens |
|---|---|---|
| 1 | `FROM` / `JOIN` | Build the rows |
| 2 | `WHERE` | Remove individual rows |
| 3 | `GROUP BY` | Make buckets |
| 4 | Aggregates | Calculate `COUNT`, `SUM`, and so on |
| 5 | `HAVING` | Remove whole buckets |
| 6 | `SELECT` | Choose and calculate columns, then `DISTINCT` |
| 7 | `ORDER BY` | Sort |
| 8 | `LIMIT` / `OFFSET` | Cut the slice |

This order explains most "why doesn't this work?" moments:

| Surprise | Reason |
|---|---|
| `WHERE` can't use a `SELECT` alias | `WHERE` runs **before** `SELECT` |
| `WHERE` can't use `COUNT(*)` | Groups don't exist yet, use `HAVING` |
| `ORDER BY` **can** use an alias | `ORDER BY` runs **after** `SELECT` |
| `LIMIT` without `ORDER BY` is random | The slice is cut from an unordered set |

### Which tool for which question?
| Question | Tool |
|---|---|
| Show only some rows | `WHERE` |
| Combine rows from two tables | `JOIN` |
| Keep unmatched rows too | `LEFT` / `RIGHT` / `FULL JOIN` |
| How many, how much, what average | Aggregates |
| ...per category | `GROUP BY` |
| ...only for big categories | `HAVING` |
| Stack two result lists | `UNION` |
| Rows in both lists / only in the first | `INTERSECT` / `EXCEPT` |
| The answer depends on another answer | Subquery |
| Turn values into labels | `CASE` |

### Check that your practice database is healthy
```sql
\dt
```
**Output:**
```
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | comments | table | postgres
 public | phones   | table | postgres
 public | photos   | table | postgres
 public | users    | table | postgres
(4 rows)
```
```sql
SELECT
  (SELECT COUNT(*) FROM users) AS users,
  (SELECT COUNT(*) FROM photos) AS photos,
  (SELECT COUNT(*) FROM comments) AS comments;
```
**Output:**
```
 users | photos | comments
-------+--------+----------
     5 |      8 |       10
(1 row)
```
If your numbers differ, run the reset script from topic 36.

### What's next
So far you've **used** data. The next topics look at what you can **store**: the kinds of values a column can hold (numbers, text, booleans, dates, and durations), and why picking the right type matters.

> ⚠️ **Warning:** Skills fade without practice. Re-run a few earlier queries, change one thing, and predict the result **before** running it.

> 🏆 **Golden Rule:** **Know the running order.** `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `SELECT`, `ORDER BY`, `LIMIT`. Almost every SQL puzzle becomes clear once you ask "which step am I at?"



## 🏷️ 88. Data Types

**What it does:** A data type tells Postgres **what kind of value** a column holds (whole number, decimal, text, true/false, date) and what you're allowed to do with it.
**Why it is used:** Types protect your data. A `price` column can't accept the word `cheap`, and a date column can't accept `February 30th`. Types also decide how much space values use and how they sort and compare.
**When to use it:** Every time you write `CREATE TABLE`, and every time you wonder "why did this calculation give a strange answer?"

**Analogy:** Different kinds of containers. A measuring cup (number), a label maker (text), a light switch (boolean), and a calendar (date). Putting the wrong thing in the wrong container either fails or gives nonsense.

### The families you'll use most
| Family | Types | Topic |
|---|---|---|
| **Numbers** | `SMALLINT`, `INTEGER`, `BIGINT`, `NUMERIC`, `REAL`, `DOUBLE PRECISION`, `SERIAL` | 89 and 90 |
| **Text** | `CHAR(n)`, `VARCHAR(n)`, `TEXT` | 91 |
| **True/false** | `BOOLEAN` | 92 |
| **Dates and times** | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ` | 93 |
| **Durations** | `INTERVAL` | 94 |

PostgreSQL also has many more (`UUID`, `JSONB`, arrays, `ENUM`, and so on), which you can learn when you need them.

### Ask Postgres what type a value is
```sql
SELECT
  pg_typeof(1) AS whole,
  pg_typeof(1.5) AS decimal,
  pg_typeof('hi') AS words,
  pg_typeof(true) AS flag,
  pg_typeof(CURRENT_DATE) AS day,
  pg_typeof(NOW()) AS moment;
```
**Output:**
```
  whole  | decimal | words | flag    | day  |          moment
---------+---------+-------+---------+------+--------------------------
 integer | numeric | text  | boolean | date | timestamp with time zone
(1 row)
```
`pg_typeof(...)` is a handy way to check what Postgres thinks a value is.

### The type is checked every time you write data
```sql
INSERT INTO phones (name, price) VALUES ('Test', 'cheap');
```
**Output:**
```
ERROR:  invalid input syntax for type integer: "cheap"
LINE 1: INSERT INTO phones (name, price) VALUES ('Test', 'cheap');
                                                          ^
```
Postgres refused, and nothing was inserted.

### Converting between types (casting)
Two spellings do the same thing:
```sql
SELECT '42'::integer + 1 AS with_colons, CAST('42' AS integer) + 1 AS with_cast;
```
**Output:**
```
 with_colons | with_cast
-------------+-----------
          43 |        43
(1 row)
```
`::` is the short PostgreSQL form. `CAST(... AS ...)` is standard SQL and works in every database.

**Casting has rules:**
```sql
SELECT 3.7::integer AS rounded;
```
**Output:**
```
 rounded
---------
       4
(1 row)
```
A decimal cast to an integer is **rounded**, not chopped.
```sql
SELECT '3.7'::integer;
```
**Output:**
```
ERROR:  invalid input syntax for type integer: "3.7"
```
But **text** that looks like a decimal can't be cast straight to an integer.

### How to see a table's types
```sql
\d phones
```
**Output:**
```
                    Table "public.phones"
    Column    |         Type          | Collation | Nullable | Default
--------------+-----------------------+-----------+----------+---------
 name         | character varying(50) |           |          |
 manufacturer | character varying(50) |           |          |
 price        | integer               |           |          |
 units_sold   | integer               |           |          |
```

### Why choosing well matters
| Wrong choice | Consequence |
|---|---|
| Money stored as a float | Rounding errors (topic 90) |
| A number stored as text | Sorts wrongly (`'10'` before `'9'`) and can't be summed |
| A date stored as text | No date maths, no validation |
| A flag stored as `'yes'`/`'no'` text | Typos like `'Yes '` and `'yess'` slip in |

> ⚠️ **Warning:** Changing a column's type **after** the table holds data is possible but needs care. Choose well at the start (topic 2).

> ⚠️ **Warning:** `pg_typeof('hi')` reports `text` for an untyped string in a `SELECT`, but a string literal is flexible and takes on the type of the column it's compared with or inserted into.

> 🏆 **Golden Rule:** **Use the most specific type that fits the data.** The database then rejects bad values for you.



## ⚡ 89. Fast Rules on Numeric Data Types

**What it does:** Four quick rules for picking a numeric type, so you don't have to memorize every option.
**Why it is used:** PostgreSQL has many number types, and the right one depends on **what the number means**, not just how it looks.
**When to use it:** Every time you add a number column to a table.

**Analogy:** Choosing a container. A small box for a few items (`SMALLINT`), a normal box (`INTEGER`), a huge crate (`BIGINT`), and a precision scale for money (`NUMERIC`).

### The four rules
| What are you storing? | Use | Example |
|---|---|---|
| An **auto-generated id** (primary key) | `SERIAL` | `id SERIAL PRIMARY KEY` |
| A **whole number** (no decimals) | `INTEGER` | `units_sold INTEGER` |
| A **decimal that must be exact** (money, totals you add up) | `NUMERIC(precision, scale)` | `price NUMERIC(10, 2)` |
| A **decimal where tiny rounding is fine** (sensors, science) | `DOUBLE PRECISION` | `temperature DOUBLE PRECISION` |

If it doesn't fit rule 1 or 3 or 4, it's almost certainly rule 2.

### Ranges of the whole-number types
| Type | Size | Smallest | Largest |
|---|---|---|---|
| `SMALLINT` | 2 bytes | -32,768 | 32,767 |
| `INTEGER` | 4 bytes | -2,147,483,648 | 2,147,483,647 |
| `BIGINT` | 8 bytes | about -9.2 quintillion | about 9.2 quintillion |

**The `SERIAL` family** (auto-counting ids, topic 26):

| Type | Behaves like | Largest id |
|---|---|---|
| `SMALLSERIAL` | `SMALLINT` | 32,767 |
| `SERIAL` | `INTEGER` | 2,147,483,647 |
| `BIGSERIAL` | `BIGINT` | about 9.2 quintillion |

`SERIAL` is plenty for most tables (over 2 billion rows). Use `BIGSERIAL` if you expect more.

### Pick the types for a new table
```sql
CREATE TABLE products (
  id SERIAL PRIMARY KEY,           -- rule 1: auto id
  name VARCHAR(100),
  stock INTEGER,                   -- rule 2: whole number
  price NUMERIC(10, 2),            -- rule 3: money, exact
  weight_kg DOUBLE PRECISION       -- rule 4: measurement, tiny error is fine
);
```
**Output:**
```
CREATE TABLE
```
(You can `DROP TABLE products;` afterwards if you don't want to keep it.)

### What Postgres picks for a number you type
```sql
SELECT
  pg_typeof(5) AS small_whole,
  pg_typeof(3000000000) AS big_whole,
  pg_typeof(5.5) AS with_decimal;
```
**Output:**
```
 small_whole | big_whole | with_decimal
-------------+-----------+--------------
 integer     | bigint    | numeric
(1 row)
```
A number too large for `INTEGER` is automatically a `BIGINT`, and a number with a decimal point is a `NUMERIC`.

### A quick decision guide
| Ask yourself | Answer |
|---|---|
| Does it have a decimal point? | No, so `INTEGER` or `SERIAL` |
| Is it **money** or something you'll add up and compare for exact equality? | `NUMERIC` |
| Is it a measurement where 0.0000001 doesn't matter? | `DOUBLE PRECISION` |
| Could it exceed about 2 billion? | `BIGINT` |

> ⚠️ **Warning:** **Never use a float for money.** `DOUBLE PRECISION` and `REAL` store approximations (topic 90).

> ⚠️ **Warning:** `SERIAL` isn't a real type, it's shorthand for "an `INTEGER` plus an auto-counter." Postgres also offers the SQL-standard `GENERATED ... AS IDENTITY` (topic 26).

> 🏆 **Golden Rule:** **Auto id → `SERIAL`. Whole number → `INTEGER`. Exact decimal → `NUMERIC`. Approximate decimal → `DOUBLE PRECISION`.**




## 🔢 90. More on Number Data Types

**What it does:** The details behind the four rules in topic 89: why floats are inexact, how `NUMERIC(precision, scale)` works, what happens on overflow, and a few surprises.
**Why it is used:** Number bugs are silent. The query runs, the figures look reasonable, and they're slightly wrong.
**When to use it:** Any time money, totals, or very large numbers are involved.

**Analogy:** A **ruler** (float) is good enough for woodwork, but you wouldn't count cash with one. A **cash register** (`NUMERIC`) counts every cent exactly.

### Floats are approximations
```sql
SELECT 0.1::double precision + 0.2::double precision AS float_sum,
       0.1::numeric + 0.2::numeric AS exact_sum;
```
**Output:**
```
       float_sum        | exact_sum
------------------------+-----------
    0.30000000000000004 |       0.3
(1 row)
```
Computers store floats in binary, and `0.1` has no exact binary form, just like 1/3 has no exact decimal form.

**It breaks equality tests:**
```sql
SELECT 0.1::double precision + 0.2::double precision = 0.3::double precision AS is_equal;
```
**Output:**
```
 is_equal
----------
 f
(1 row)
```

**And the error grows when you add many values:**
```sql
SELECT SUM(0.1::double precision) AS float_total,
       SUM(0.1::numeric) AS exact_total
FROM generate_series(1, 10);
```
**Output:**
```
     float_total      | exact_total
----------------------+-------------
   0.9999999999999999 |         1.0
(1 row)
```
Ten tenths should be exactly 1.

### `REAL` vs `DOUBLE PRECISION`
| Type | Size | Reliable digits |
|---|---|---|
| `REAL` | 4 bytes | about 6 |
| `DOUBLE PRECISION` | 8 bytes | about 15 |

Prefer `DOUBLE PRECISION` unless you have a reason to save the space.

### `NUMERIC(precision, scale)`
| Part | Meaning |
|---|---|
| **precision** | Total number of digits |
| **scale** | How many of them come **after** the decimal point |

`NUMERIC(5, 2)` means 5 digits in total, 2 after the point. So the largest value is `999.99`.

```sql
SELECT 123.456::numeric(5, 2) AS rounded;
```
**Output:**
```
 rounded
---------
  123.46
(1 row)
```
Extra decimals are **rounded** to fit the scale.
```sql
SELECT 1000.00::numeric(5, 2);
```
**Output:**
```
ERROR:  numeric field overflow
DETAIL:  A field with precision 5, scale 2 must round to an absolute value less than 10^3.
```
Too many digits **before** the point is an **error**.

**`NUMERIC` with no arguments** stores any precision you give it (up to 131,072 digits before the point and 16,383 after).

### Whole numbers overflow too
```sql
SELECT 2147483647 + 1;
```
**Output:**
```
ERROR:  integer out of range
```
**The fix: use a bigger type:**
```sql
SELECT 2147483647::bigint + 1 AS bigger;
```
**Output:**
```
   bigger
------------
 2147483648
(1 row)
```
```sql
SELECT 40000::smallint;
```
**Output:**
```
ERROR:  smallint out of range
```

### Integer division (a reminder from topic 7)
```sql
SELECT 10 / 4 AS whole_division, 10.0 / 4 AS decimal_division;
```
**Output:**
```
 whole_division | decimal_division
----------------+------------------
              2 | 2.5000000000000000
(1 row)
```
Two integers give an integer. Make one side a decimal (`10.0` or `::numeric`) to keep the fraction.

### Rounding is not the same for every type
```sql
SELECT ROUND(2.5::numeric) AS numeric_round, ROUND(2.5::double precision) AS float_round;
```
**Output:**
```
 numeric_round | float_round
---------------+-------------
             3 |           2
(1 row)
```
`NUMERIC` rounds halves **away from zero**. For `DOUBLE PRECISION` the behavior is platform-dependent, and **round-to-nearest-even** is the most common, so `2.5` becomes `2`.

### Which type for money?
| Approach | Verdict |
|---|---|
| `NUMERIC(12, 2)` | ✅ Best for most apps |
| `INTEGER` / `BIGINT` storing **cents** (`1999` means $19.99) | ✅ Good, and fast |
| `DOUBLE PRECISION` / `REAL` | ❌ Rounding errors |
| The `MONEY` type | ⚠️ Avoid, because it depends on locale settings |

> ⚠️ **Warning:** **Never compare floats with `=`.** Compare within a small range instead, or use `NUMERIC`.

> ⚠️ **Warning:** `NUMERIC(5, 2)` rounds **extra decimals** but **rejects** too many whole-number digits. Choose a precision with headroom.

> ⚠️ **Warning:** Overflow errors appear **in the middle of a calculation**, not only when storing. `SUM` of many `INTEGER`s is promoted to `BIGINT` automatically, but your own `a * b` is not.

> 🏆 **Golden Rule:** **Exact answers need `NUMERIC`.** If an amount will be added, compared, or shown to a customer, don't store it as a float.




## 🔤 91. Reminder on Character Types

**What it does:** A refresher on the three text types, `CHAR(n)`, `VARCHAR(n)`, and `TEXT`, and when to use each.
**Why it is used:** They look similar, but they differ on padding, length limits, and what happens when a value is too long.
**When to use it:** Every time you add a text column.

**Analogy:** `CHAR(5)` is a **fixed-size parking spot**: a short car still takes the whole spot. `VARCHAR(5)` is a spot with a **length limit** but no wasted space. `TEXT` is an open lot with **no limit**.

### The three types
| Type | Behavior |
|---|---|
| `CHAR(n)` | **Fixed length.** Shorter values are padded with spaces up to `n` |
| `VARCHAR(n)` | **Up to `n` characters.** No padding |
| `TEXT` | **Any length** (up to about 1 GB) |
| `VARCHAR` (no number) | Same as `TEXT` |

**In PostgreSQL, all three perform about the same.** There is no speed advantage to `VARCHAR(10)` over `TEXT`.

### `CHAR(n)` pads with spaces
```sql
SELECT 'ab'::char(5) AS padded,
       LENGTH('ab'::char(5)) AS len,
       OCTET_LENGTH('ab'::char(5)) AS bytes;
```
**Output:**
```
 padded | len | bytes
--------+-----+-------
 ab     |   2 |     5
(1 row)
```
`padded` really holds `ab` plus three trailing spaces (invisible here). `LENGTH` ignores the padding, but it still takes up 5 bytes. This is why most people avoid `CHAR(n)`.

### A too-long value is rejected on insert
```sql
CREATE TEMP TABLE codes (code VARCHAR(3));
INSERT INTO codes (code) VALUES ('abcdef');
```
**Output:**
```
CREATE TABLE
ERROR:  value too long for type character varying(3)
```
`TEMP` tables disappear when you close your session, so they're good for experiments.

### The explicit-cast surprise
```sql
SELECT 'abcdef'::varchar(3) AS cut_short;
```
**Output:**
```
 cut_short
-----------
 abc
(1 row)
```
An **insert** is rejected, but an **explicit cast** silently **truncates**. Don't use a cast to "validate" length.

### Characters vs bytes
```sql
SELECT LENGTH('héllo') AS characters, OCTET_LENGTH('héllo') AS bytes;
```
**Output:**
```
 characters | bytes
------------+-------
          5 |     6
(1 row)
```
`VARCHAR(n)` limits **characters**, not bytes. The accent takes two bytes in UTF-8, but it counts as one character. (This assumes a UTF-8 database, which is the common default.)

### Quotes and escaping (a reminder from topic 5)
```sql
SELECT 'It''s fine' AS apostrophe;
```
**Output:**
```
 apostrophe
------------
 It's fine
(1 row)
```
Double the single quote inside a text value. Double quotes mean "a column or table name."

### Which one should I use?
| Situation | Use |
|---|---|
| A **hard limit** is part of the rule (a 240-character comment, a 2-letter country code) | `VARCHAR(n)` |
| No natural limit (a bio, an article, a description) | `TEXT` |
| Fixed-length codes padded on purpose | Rarely `CHAR(n)`, and usually `VARCHAR(n)` is safer |
| You aren't sure | `TEXT` |

### Reminders for working with text
| Tool | Reminder |
|---|---|
| Comparisons | `'Delhi' = 'delhi'` is **false**, so use `LOWER()` to ignore case (topics 9 and 11) |
| Joining | `\|\|` returns `NULL` if any part is `NULL`, while `CONCAT()` ignores it (topic 9) |
| Sorting | Follows the database's collation, so mixed case and accents can order differently |
| Numbers as text | `'10'` sorts before `'9'`, so store numbers as numbers |

> ⚠️ **Warning:** A hard length limit becomes a **bug** when real data doesn't fit. Names, addresses, and URLs are longer than people expect. Don't set limits you don't need.

> ⚠️ **Warning:** Changing a `VARCHAR` limit later can be slow on huge tables, and making it **smaller** fails if existing values are too long.

> 🏆 **Golden Rule:** **Use `TEXT` unless a real rule demands a limit.** Then use `VARCHAR(n)`. Avoid `CHAR(n)`.




## ✅ 92. Boolean Data Types

**What it does:** `BOOLEAN` stores one of three states: **true**, **false**, or **`NULL`** (unknown).
**Why it is used:** Yes/no facts (is this task done? is the account active? is the photo public?) are cleaner and safer as a real boolean than as text like `'yes'`.
**When to use it:** Any column that is conceptually a flag or a switch.

**Analogy:** A **light switch**: on or off. `NULL` is a switch you haven't looked at yet, so you don't know which position it's in.

### Create and fill a small table
```sql
CREATE TEMP TABLE tasks (
  id SERIAL PRIMARY KEY,
  title VARCHAR(50),
  done BOOLEAN
);

INSERT INTO tasks (title, done)
VALUES ('Write notes', true), ('Run tests', false), ('Deploy', NULL);
```
**Output:**
```
CREATE TABLE
INSERT 0 3
```
```sql
SELECT * FROM tasks;
```
**Output:**
```
 id |    title    | done
----+-------------+------
  1 | Write notes | t
  2 | Run tests   | f
  3 | Deploy      |
(3 rows)
```
`psql` shows `t` for true, `f` for false, and an empty cell for `NULL`.

### Accepted ways to write a boolean
| True | False |
|---|---|
| `true`, `'true'` | `false`, `'false'` |
| `'t'`, `'y'`, `'yes'` | `'f'`, `'n'`, `'no'` |
| `'on'`, `'1'` | `'off'`, `'0'` |

```sql
SELECT 'yes'::boolean AS a, 'off'::boolean AS b, '1'::boolean AS c, 'T'::boolean AS d;
```
**Output:**
```
 a | b | c | d
---+---+---+---
 t | f | t | t
(1 row)
```
```sql
SELECT 'maybe'::boolean;
```
**Output:**
```
ERROR:  invalid input syntax for type boolean: "maybe"
```
Prefer plain `true` and `false` in your own SQL.

### Filtering on a boolean
```sql
SELECT title FROM tasks WHERE done;
```
**Output:**
```
    title
-------------
 Write notes
(1 row)
```
`WHERE done` already means `WHERE done = true`, so you don't need `= true`.
```sql
SELECT title FROM tasks WHERE NOT done;
```
**Output:**
```
   title
-----------
 Run tests
(1 row)
```
**`Deploy` is missing from both!** Its value is `NULL`, which is neither true nor false.

**To find the "not done yet" rows including unknowns:**
```sql
SELECT title FROM tasks WHERE done IS NOT TRUE;
```
**Output:**
```
   title
-----------
 Run tests
 Deploy
(2 rows)
```
Or find only the unknowns:
```sql
SELECT title FROM tasks WHERE done IS NULL;
```
**Output:**
```
 title
--------
 Deploy
(1 row)
```

### Comparisons produce booleans
```sql
SELECT name, price, price > 300 AS is_pricey FROM phones;
```
**Output:**
```
    name     | price | is_pricey
-------------+-------+-----------
 N1280       |   199 | f
 iPhone 5S   |   399 | t
 Galaxy S5   |   299 | f
 S5620 Monte |   250 | f
(4 rows)
```
Every `WHERE` condition you've written is really a boolean expression.

### Three-valued logic with `NULL`
| Expression | Result |
|---|---|
| `TRUE AND NULL` | `NULL` |
| `FALSE AND NULL` | `FALSE` |
| `TRUE OR NULL` | `TRUE` |
| `FALSE OR NULL` | `NULL` |
| `NOT NULL` | `NULL` |

`WHERE` keeps a row **only when the result is true**, so `NULL` rows are dropped, as in topic 43.

### Counting booleans
```sql
SELECT
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE done) AS finished,
  COUNT(*) FILTER (WHERE done IS NOT TRUE) AS unfinished
FROM tasks;
```
**Output:**
```
 total | finished | unfinished
-------+----------+------------
     3 |        1 |          2
(1 row)
```

### Good design habits
| Habit | Why |
|---|---|
| Name flags like questions: `is_active`, `has_paid` | Reads naturally in `WHERE is_active` |
| Forbid `NULL` when you can: `BOOLEAN NOT NULL DEFAULT false` | No unknown third state |
| Don't store flags as `'yes'` / `'no'` text | Typos like `'Yes '` slip in |

```sql
CREATE TABLE accounts (
  id SERIAL PRIMARY KEY,
  is_active BOOLEAN NOT NULL DEFAULT true
);
```

> ⚠️ **Warning:** `NOT done` does **not** return `NULL` rows. If a flag can be `NULL`, handle it with `IS NOT TRUE`, `IS NULL`, or `COALESCE`.

> ⚠️ **Warning:** Never write `= NULL` (topic 11). Use `IS NULL`.

> 🏆 **Golden Rule:** **A boolean has three possible states.** Decide what `NULL` means, or forbid it with `NOT NULL DEFAULT ...`.