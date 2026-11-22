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


## 📅 93. Times, Dates, and Timestamps

**What it does:** Covers the types that store **when** something happens: `DATE`, `TIME`, `TIMESTAMP`, and `TIMESTAMPTZ`, plus the functions for working with them.
**Why it is used:** Almost every app records when things happen: when a user signed up, when a photo was posted, when an order shipped.
**When to use it:** Any column that records or schedules a moment in time.

**Analogy:** A `DATE` is a **page of a wall calendar**. A `TIME` is a **clock face**. A `TIMESTAMP` is **both together**. A `TIMESTAMPTZ` is both together **plus the knowledge of which time zone you were in**.

### The four types
| Type | Stores | Example |
|---|---|---|
| `DATE` | A calendar day | `2026-11-20` |
| `TIME` | A time of day | `14:30:00` |
| `TIMESTAMP` | Date and time, **no** time zone | `2026-11-20 14:30:00` |
| `TIMESTAMPTZ` | Date and time, **with** time zone | `2026-11-20 14:30:00+00` |

`TIMESTAMPTZ` is the long form of `TIMESTAMP WITH TIME ZONE`.

### Writing values: use the ISO format
```sql
SELECT
  '2026-11-20'::date AS a_date,
  '14:30:00'::time AS a_time,
  '2026-11-20 14:30:00'::timestamp AS a_timestamp;
```
**Output:**
```
   a_date   | a_time   |     a_timestamp
------------+----------+---------------------
 2026-11-20 | 14:30:00 | 2026-11-20 14:30:00
(1 row)
```
The safe format is **year-month-day** (`YYYY-MM-DD`). It can't be misread.

**Other formats are ambiguous:**
```sql
SHOW DateStyle;
```
**Output:**
```
 DateStyle
-----------
 ISO, MDY
(1 row)
```
With `MDY`, `'03/04/2026'` means **March 4**. Under a `DMY` setting it would mean April 3.

**Impossible dates are rejected:**
```sql
SELECT '2026-02-30'::date;
```
**Output:**
```
ERROR:  date/time field value out of range: "2026-02-30"
```

### `TIMESTAMP` vs `TIMESTAMPTZ`
| | `TIMESTAMP` | `TIMESTAMPTZ` |
|---|---|---|
| **Stores** | The exact clock reading you gave | An **exact moment** (internally in UTC) |
| **Display** | Always the same | Converted to **your session's time zone** |
| **Best for** | Local schedules ("store opens at 09:00") | **Anything that happened**, like `created_at` |

**The same moment, shown in different time zones:**
```sql
SET TIME ZONE 'UTC';
SELECT '2026-11-20 12:00:00+00'::timestamptz AS moment;
```
**Output:**
```
SET
         moment
------------------------
 2026-11-20 12:00:00+00
(1 row)
```
```sql
SET TIME ZONE 'America/New_York';
SELECT '2026-11-20 12:00:00+00'::timestamptz AS moment;
```
**Output:**
```
SET
         moment
------------------------
 2026-11-20 07:00:00-05
(1 row)
```
```sql
SET TIME ZONE 'Asia/Tokyo';
SELECT '2026-11-20 12:00:00+00'::timestamptz AS moment;
```
**Output:**
```
SET
         moment
------------------------
 2026-11-20 21:00:00+09
(1 row)
```
All three are the **same instant**, just displayed differently. Put things back afterwards:
```sql
RESET TIME ZONE;
```
**Output:**
```
RESET
```
`SET TIME ZONE` only affects your **current session**.

### Getting the current date and time
```sql
SELECT CURRENT_DATE AS today, NOW() AS right_now;
```
**Output (yours shows the real current moment):**
```
   today    |           right_now
------------+-------------------------------
 2026-11-20 | 2026-11-20 10:30:00.123456+00
(1 row)
```

### Pulling parts out with `EXTRACT`
```sql
SELECT
  EXTRACT(YEAR FROM DATE '2026-11-20') AS year,
  EXTRACT(MONTH FROM DATE '2026-11-20') AS month,
  EXTRACT(DOW FROM DATE '2026-11-20') AS day_of_week;
```
**Output:**
```
 year | month | day_of_week
------+-------+-------------
 2026 |    11 |           5
(1 row)
```
`DOW` counts **Sunday as 0**, so `5` is Friday.

### Rounding down with `DATE_TRUNC`
```sql
SELECT DATE_TRUNC('month', TIMESTAMP '2026-11-20 14:30:00') AS month_start;
```
**Output:**
```
     month_start
---------------------
 2026-11-01 00:00:00
(1 row)
```
Useful for grouping by day, month, or year.

### Formatting with `TO_CHAR`
```sql
SELECT TO_CHAR(DATE '2026-11-20', 'FMDay, DD Mon YYYY') AS pretty;
```
**Output:**
```
        pretty
----------------------
 Friday, 20 Nov 2026
(1 row)
```
`FM` removes the extra padding spaces.

### Date arithmetic
```sql
SELECT
  DATE '2026-12-25' - DATE '2026-11-20' AS days_until,
  DATE '2026-11-20' + 30 AS thirty_days_later;
```
**Output:**
```
 days_until | thirty_days_later
------------+-------------------
         35 | 2026-12-20
(1 row)
```
Subtracting two `DATE`s gives a **whole number of days**. Adding an integer to a `DATE` adds days. For richer durations, see `INTERVAL` (topic 94).

### Filtering by date
```sql
CREATE TEMP TABLE events (name VARCHAR(50), event_date DATE);

INSERT INTO events (name, event_date)
VALUES ('Kickoff', '2026-11-05'), ('Review', '2026-11-20'), ('Launch', '2026-12-10');

SELECT name, event_date
FROM events
WHERE event_date BETWEEN '2026-11-01' AND '2026-11-30'
ORDER BY event_date;
```
**Output:**
```
CREATE TABLE
INSERT 0 3
  name   | event_date
---------+------------
 Kickoff | 2026-11-05
 Review  | 2026-11-20
(2 rows)
```

> ⚠️ **Warning:** **For "when did this happen" columns, use `TIMESTAMPTZ`.** A plain `TIMESTAMP` has no time zone, so the same value can mean different moments to different people.

> ⚠️ **Warning:** Avoid `TIME WITH TIME ZONE`. A time of day without a date can't handle daylight saving properly.

> ⚠️ **Warning:** `BETWEEN` includes **both ends**. Comparing a `TIMESTAMP` against `'2026-11-30'` means midnight at the **start** of that day, so events later on the 30th are missed.

> 🏆 **Golden Rule:** **Write dates as `YYYY-MM-DD`, store moments as `TIMESTAMPTZ`, and let the database do the date maths.**



## ⏱️ 94. Really Awesome Intervals

**What it does:** `INTERVAL` stores a **length of time**: "3 days," "2 hours 30 minutes," "1 year 2 months." You use it to add to or subtract from dates and timestamps.
**Why it is used:** Questions like "what's the date 30 days from now?" or "how long did that take?" are all about durations.
**When to use it:** Deadlines, expiry times, "in the last 7 days" filters, and session lengths.

**Analogy:** A **measuring tape for time**. A date is a spot on the calendar, and an interval is the distance between two spots.

### Writing an interval
```sql
SELECT
  INTERVAL '1 year 2 months 3 days' AS long_one,
  INTERVAL '2 hours 30 minutes' AS short_one,
  INTERVAL '2 weeks' AS weeks;
```
**Output:**
```
        long_one        | short_one |  weeks
------------------------+-----------+---------
 1 year 2 mons 3 days   | 02:30:00  | 14 days
(1 row)
```
The units you can use: years, months, weeks, days, hours, minutes, and seconds. Postgres abbreviates months as `mons` in the output.

### Adding intervals to dates and timestamps
```sql
SELECT
  TIMESTAMP '2026-11-20 09:00' + INTERVAL '2 hours 30 minutes' AS meeting_end,
  TIMESTAMP '2026-11-20 12:00' - INTERVAL '7 days' AS week_ago;
```
**Output:**
```
     meeting_end     |       week_ago
---------------------+---------------------
 2026-11-20 11:30:00 | 2026-11-13 12:00:00
(1 row)
```
A `DATE` plus an interval gives a `TIMESTAMP`:
```sql
SELECT DATE '2026-11-20' + INTERVAL '1 month' AS next_month;
```
**Output:**
```
     next_month
---------------------
 2026-12-20 00:00:00
(1 row)
```

### Subtracting timestamps gives an interval
```sql
SELECT
  TIMESTAMP '2026-11-20 18:00' - TIMESTAMP '2026-11-20 09:30' AS worked,
  TIMESTAMP '2026-11-22 08:00' - TIMESTAMP '2026-11-20 09:30' AS elapsed;
```
**Output:**
```
  worked  |     elapsed
----------+------------------
 08:30:00 | 1 day 22:30:00
(1 row)
```

**Two different "differences":**

| Subtracting... | Gives |
|---|---|
| `DATE - DATE` | A whole **number of days** (topic 93) |
| `TIMESTAMP - TIMESTAMP` | An **interval** |

### Interval maths
```sql
SELECT
  INTERVAL '1 day' * 3 AS three_days,
  INTERVAL '1 day' + INTERVAL '12 hours' AS day_and_a_half,
  INTERVAL '90 minutes' > INTERVAL '1 hour' AS longer;
```
**Output:**
```
 three_days | day_and_a_half | longer
------------+----------------+--------
 3 days     | 1 day 12:00:00 | t
(1 row)
```

### Age between two dates
```sql
SELECT AGE(DATE '2026-11-20', DATE '2000-05-15') AS age;
```
**Output:**
```
             age
------------------------------
 26 years 6 mons 5 days
(1 row)
```
`AGE` counts **calendar** years, months, and days. With one argument, `AGE(date)` measures from today.

### Turning an interval into a number
```sql
SELECT EXTRACT(EPOCH FROM INTERVAL '1 hour 30 minutes') AS seconds;
```
**Output:**
```
   seconds
-------------
 5400.000000
(1 row)
```
`EPOCH` is the total number of seconds. Divide by 60 for minutes, or by 3600 for hours.

### Building an interval from numbers
```sql
SELECT make_interval(hours => 2, mins => 30) AS built;
```
**Output:**
```
  built
----------
 02:30:00
(1 row)
```
This is handy when the amount comes from a column or a variable rather than being typed out.

### Practical example: "in the last 7 days"
```sql
CREATE TEMP TABLE logins (username VARCHAR(50), logged_in_at TIMESTAMP);

INSERT INTO logins (username, logged_in_at)
VALUES
  ('monahan93', '2026-11-19 08:00'),
  ('pferrer',   '2026-11-14 18:30'),
  ('si93onis',  '2026-10-01 12:00');

SELECT username, logged_in_at
FROM logins
WHERE logged_in_at > TIMESTAMP '2026-11-20 12:00' - INTERVAL '7 days'
ORDER BY logged_in_at;
```
**Output:**
```
CREATE TABLE
INSERT 0 3
 username  |    logged_in_at
-----------+---------------------
 pferrer   | 2026-11-14 18:30:00
 monahan93 | 2026-11-19 08:00:00
(2 rows)
```
In a real app, replace the fixed timestamp with `NOW()`:
```sql
WHERE logged_in_at > NOW() - INTERVAL '7 days'
```

### The month-end surprise
Months aren't a fixed number of days, so adding one to a month-end date can land on an earlier day:
```sql
SELECT DATE '2026-01-31' + INTERVAL '1 month' AS result;
```
**Output:**
```
       result
---------------------
 2026-02-28 00:00:00
(1 row)
```
Postgres clamps to the last day of February.

**Why intervals are stored in three parts:** an interval keeps its **months**, **days**, and **time** separately, because "1 month" and "1 day" don't always equal a fixed number of hours (months differ in length, and a day can be 23 or 25 hours around daylight-saving changes).

### Quick reference
| You want... | Write |
|---|---|
| 30 days from a date | `date_col + INTERVAL '30 days'` (or `date_col + 30`) |
| 1 hour ago | `NOW() - INTERVAL '1 hour'` |
| How long between two moments | `end_ts - start_ts` |
| That length in seconds | `EXTRACT(EPOCH FROM (end_ts - start_ts))` |
| Years, months, days between dates | `AGE(later, earlier)` |

> ⚠️ **Warning:** Don't assume `INTERVAL '1 month'` is 30 days. Use `'30 days'` if you mean exactly 30 days.

> ⚠️ **Warning:** `DATE + INTERVAL` returns a `TIMESTAMP`, not a `DATE`. Cast back with `::date` if you need a date.

> ⚠️ **Warning:** Subtracting two `TIMESTAMPTZ` values across a daylight-saving change gives the **real** elapsed time, which may differ from the clock difference by an hour.

> 🏆 **Golden Rule:** **Let `INTERVAL` do the counting.** Add it to dates, subtract timestamps to get one, and use `EXTRACT(EPOCH ...)` when you need a plain number.




## 🛡️ 95. Thinking About Validation

**What it does:** Validation means **refusing bad data before it is stored**. In a database, the rules that do this are called **constraints**: required values, defaults, uniqueness, and custom checks.
**Why it is used:** Data types alone (topic 88) only stop the wrong **kind** of value. They won't stop a negative price, a missing name, or the same product entered twice.
**When to use it:** At design time, **before** the table has real data in it (topic 21).

**Analogy:** A **bouncer at the door** of the table. The types say "no animals," and constraints add "no one under 18, no one without a ticket, no one who's already inside."

### What a table without rules lets in
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40),
  department VARCHAR(40),
  price INTEGER,
  weight INTEGER
);

INSERT INTO products (name, department, price, weight)
VALUES
  ('Shirt', 'Clothes', 20, 1),
  ('Shirt', 'Clothes', 20, 1),
  (NULL, NULL, NULL, NULL),
  ('Hammer', 'Tools', -15, 2),
  ('Teddy bear', 'Toys', 12, -3);
```
**Output:**
```
NOTICE:  table "products" does not exist, skipping
DROP TABLE
CREATE TABLE
INSERT 0 5
```
(The `NOTICE` appears only if the table didn't exist. If it did, you'll see just `DROP TABLE`.)
```sql
SELECT * FROM products;
```
**Output:**
```
 id |    name    | department | price | weight
----+------------+------------+-------+--------
  1 | Shirt      | Clothes    |    20 |      1
  2 | Shirt      | Clothes    |    20 |      1
  3 |            |            |       |
  4 | Hammer     | Tools      |   -15 |      2
  5 | Teddy bear | Toys       |    12 |     -3
(5 rows)
```
Every row was accepted, and four of the five are wrong:

| Row | Problem |
|---|---|
| 1 and 2 | The **same product twice** |
| 3 | A completely **empty** product |
| 4 | A **negative price** |
| 5 | A **negative weight** |

### Turn the problems into rules
| Problem | Rule | Tool | Topic |
|---|---|---|---|
| Missing name, department, or price | Must have a value | `NOT NULL` | 97 and 98 |
| Missing price should fall back to something | Use a standard value | `DEFAULT` | 99 |
| Same product twice | No duplicates | `UNIQUE` | 100 and 101 |
| Negative price or weight | Value must make sense | `CHECK` | 102 and 103 |

### Three places a rule can live
| Layer | Example | Strength |
|---|---|---|
| **The form** | HTML `required`, `min="1"` | Friendly, but a user can bypass it |
| **Your app** | Zod and Server Actions (Next.js topics 62 and 73) | Good error messages, but only protects data entered **through the app** |
| **The database** | Constraints (this section) | **Cannot be bypassed**, because every write goes through it |

The database layer matters most because data arrives from many places: your app, a script, a teammate with `psql`, a future second app. Topic 104 ties the three layers together.

> ⚠️ **Warning:** **Add constraints early.** Adding a rule to a table that already contains bad rows fails until you clean the data first (topics 98, 100, and 102).

> ⚠️ **Warning:** Validation in the app is **not** a substitute for constraints. It only protects the paths that go through the app.

> 🏆 **Golden Rule:** **Decide the rules before the data arrives.** For each column ask: can it be empty, does it need a default, must it be unique, and what values make sense?



## 🖱️ 96. Creating and Viewing Tables in pgAdmin

**What it does:** Shows how to **create a table with clicks** in pgAdmin instead of typing `CREATE TABLE`, and how to **view** its structure and rows afterwards.
**Why it is used:** The dialog lists every option (types, `NOT NULL`, defaults, constraints) in one place, and it writes the SQL for you. That makes it a good way to **learn** the syntax and a quick way to inspect tables.
**When to use it:** While learning constraints, or whenever you'd rather click than type. You set up pgAdmin in topic 86.

**Analogy:** A form-based way to fill out the same paperwork. The SQL is still the official document, and pgAdmin just helps you fill it in.

> **Note:** menu labels differ slightly between pgAdmin versions. If a label doesn't match exactly, look for the closest one.

### Create a table
**Step 1.** Start fresh. Drop the practice table from topic 95 in the Query Tool:
```sql
DROP TABLE IF EXISTS products;
```
**Output:**
```
DROP TABLE
```
**Step 2.** In the left panel expand **Servers**, then your server, then **Databases**, then **photo_app**, then **Schemas**, then **public**.
**Step 3.** Right-click **Tables**, then choose **Create**, then **Table...**.
**Step 4.** On the **General** tab, set **Name** to `products`.
**Step 5.** On the **Columns** tab, click **+** to add each column:

| Name | Data type | Length | Not NULL? | Primary key? |
|---|---|---|---|---|
| `id` | `serial` | | ✅ | ✅ |
| `name` | `character varying` | 40 | | |
| `department` | `character varying` | 40 | | |
| `price` | `integer` | | | |
| `weight` | `integer` | | | |

**Step 6.** Click the **SQL** tab to **preview** the statement pgAdmin will run.
**Step 7.** Click **Save**.

### The SQL pgAdmin generates
The **SQL** tab shows something like this (your formatting may differ slightly):
```sql
CREATE TABLE IF NOT EXISTS public.products
(
    id serial NOT NULL,
    name character varying(40),
    department character varying(40),
    price integer,
    weight integer,
    CONSTRAINT products_pkey PRIMARY KEY (id)
);
```
That's the same table you've been writing by hand, just more verbose. **Reading this tab is one of the best ways to learn constraint syntax**, because you tick a box and watch the SQL change.

### Where each option lives in the dialog
| Option | Where in pgAdmin |
|---|---|
| `NOT NULL` | **Columns** tab, **Not NULL?** switch |
| Primary key | **Columns** tab, **Primary key?** switch |
| `DEFAULT` | **Columns** tab, **Default** field |
| `UNIQUE` | **Constraints** tab, then **Unique** |
| `CHECK` | **Constraints** tab, then **Check** |
| Foreign key | **Constraints** tab, then **Foreign Key** |

### View the table
**The table appears in the tree.** If it doesn't, right-click **Tables** and choose **Refresh**.

| What you want | How |
|---|---|
| See the **columns** | Expand **products**, then **Columns** |
| See the **constraints** | Expand **products**, then **Constraints** |
| See the **rows** | Right-click **products**, then **View/Edit Data**, then **All Rows** |
| See the **CREATE statement** | Select the table and open the **SQL** tab in the right-hand panel |
| Change the structure | Right-click **products**, then **Properties...** |
| Run your own queries | Right-click **photo_app**, then **Query Tool** |

### The same view in `psql`
```sql
\d products
```
**Output:**
```
                                   Table "public.products"
   Column   |         Type          | Collation | Nullable |               Default
------------+-----------------------+-----------+----------+--------------------------------------
 id         | integer               |           | not null | nextval('products_id_seq'::regclass)
 name       | character varying(40) |           |          |
 department | character varying(40) |           |          |
 price      | integer               |           |          |
 weight     | integer               |           |          |
Indexes:
    "products_pkey" PRIMARY KEY, btree (id)
```
Both tools show the same facts. From here on, every topic changes the rules and shows the matching SQL.

> ⚠️ **Warning:** Clicking **Save** runs the change **immediately**. Check the **SQL** tab first, especially when editing a table that already holds data.

> ⚠️ **Warning:** **Delete/Drop** in the right-click menu removes the table and its data with no undo, exactly like `DROP TABLE`.

> ⚠️ **Warning:** A table you made in the GUI is the same as one made in SQL. Don't create the same table both ways, or you'll get `ERROR: relation "products" already exists`.

> 🏆 **Golden Rule:** **Use the SQL tab as a free tutor.** Configure the table with clicks, then read the SQL, and soon you won't need the clicks.



## 🚫 97. Applying a Null Constraint

**What it does:** `NOT NULL` on a column means **every row must have a value there**. An insert or update that leaves it empty is rejected.
**Why it is used:** Some facts are essential. A product with no name or no price is meaningless, and every query that touches it has to guard against the gap (topic 11).
**When to use it:** Any column that must always be filled in. A good default is: **make columns `NOT NULL` unless "unknown" is genuinely meaningful.**

**Analogy:** The **required fields** on a form, marked with a red asterisk. You can't submit without them.

### Add `NOT NULL` when creating the table
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL,
  department VARCHAR(40) NOT NULL,
  price INTEGER NOT NULL,
  weight INTEGER
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```
Here `weight` stays optional on purpose, because some products might not have one recorded.

### A valid row
```sql
INSERT INTO products (name, department, price, weight)
VALUES ('Shirt', 'Clothes', 20, 1);
```
**Output:**
```
INSERT 0 1
```

### An explicit `NULL` is rejected
```sql
INSERT INTO products (name, department, price, weight)
VALUES ('Hammer', 'Tools', NULL, 2);
```
**Output:**
```
ERROR:  null value in column "price" of relation "products" violates not-null constraint
DETAIL:  Failing row contains (2, Hammer, Tools, null, 2).
```

### Leaving the column out is also rejected
```sql
INSERT INTO products (name, department)
VALUES ('Hammer', 'Tools');
```
**Output:**
```
ERROR:  null value in column "price" of relation "products" violates not-null constraint
DETAIL:  Failing row contains (3, Hammer, Tools, null, null).
```
Omitting a column is the same as giving it `NULL` (unless it has a default, topic 99).

### Optional columns still accept `NULL`
```sql
INSERT INTO products (name, department, price)
VALUES ('Pen', 'Stationery', 3);

SELECT * FROM products;
```
**Output:**
```
INSERT 0 1
 id | name  | department | price | weight
----+-------+------------+-------+--------
  1 | Shirt | Clothes    |     20 |      1
  4 | Pen   | Stationery |      3 |
(2 rows)
```
Ids 2 and 3 are missing because the failed inserts still used them (topic 26).

### See the constraint
```sql
\d products
```
**Output (the part that matters):**
```
   Column   |         Type          | Collation | Nullable |               Default
------------+-----------------------+-----------+----------+--------------------------------------
 id         | integer               |           | not null | nextval('products_id_seq'::regclass)
 name       | character varying(40) |           | not null |
 department | character varying(40) |           | not null |
 price      | integer               |           | not null |
 weight     | integer               |           |          |
```

### Add or remove it later
```sql
ALTER TABLE products ALTER COLUMN weight SET NOT NULL;   -- add (may fail, see topic 98)
ALTER TABLE products ALTER COLUMN weight DROP NOT NULL;  -- remove
```
**Output:**
```
ALTER TABLE
```

### Other places you'll meet it
| Place | Behavior |
|---|---|
| Primary key | Implicitly `NOT NULL` (topic 24) |
| Foreign key | Allows `NULL` unless you add `NOT NULL` (topic 31) |
| `UPDATE ... SET col = NULL` | Also rejected |

> ⚠️ **Warning:** **`NOT NULL` does not stop empty strings.** `''` is a value, not `NULL`, so a blank name slips through. Topic 102 adds a `CHECK` for that.

> ⚠️ **Warning:** Don't make a column `NOT NULL` just to be safe if "unknown" is a real possibility. Forcing a fake value (like `0` for an unknown weight) is worse than `NULL`.

> 🏆 **Golden Rule:** **Make a column `NOT NULL` by default.** Allow `NULL` only when "we don't know" is a real, meaningful state.




## 🪤 98. Solving a Gotcha with Null Constraints

**What it does:** Explains the most common `NOT NULL` problem: **adding the constraint to a table that already contains `NULL`s fails**, and shows how to fix the data first.
**Why it is used:** In real life tables **already have data**. You usually add constraints later, and the data is rarely clean.
**When to use it:** Any time `ALTER TABLE ... SET NOT NULL` throws an error.

**Analogy:** A new "no one under 18" rule at a club that already has teenagers inside. You must first deal with the people already in (let them out or change their status) before the rule can take effect.

### Set it up: a table with existing `NULL`s
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40),
  department VARCHAR(40),
  price INTEGER,
  weight INTEGER
);

INSERT INTO products (name, department, price, weight)
VALUES
  ('Shirt', 'Clothes', 20, 1),
  ('Hammer', 'Tools', NULL, 2),
  ('Teddy bear', 'Toys', 12, NULL);
```
**Output:**
```
DROP TABLE
CREATE TABLE
INSERT 0 3
```

### The gotcha
```sql
ALTER TABLE products ALTER COLUMN price SET NOT NULL;
```
**Output:**
```
ERROR:  column "price" of relation "products" contains null values
```
Postgres checks **every existing row**. The Hammer has no price, so the rule can't be applied.

### The fix, in three steps
**Step 1: find the offending rows.**
```sql
SELECT * FROM products WHERE price IS NULL;
```
**Output:**
```
 id |  name  | department | price | weight
----+--------+------------+-------+--------
  2 | Hammer | Tools      |       |      2
(1 row)
```
**Step 2: decide what to do with them.**

| Choice | When it fits |
|---|---|
| Give them a real value (`UPDATE`) | You know the right value, or a sensible placeholder |
| Delete the rows (`DELETE`) | The rows are junk |
| Leave the column nullable | "Unknown" really is meaningful |

**Step 3: fix the data, then add the constraint.** Do both inside a transaction, so nothing is half-done:
```sql
BEGIN;

UPDATE products SET price = 10 WHERE price IS NULL;
ALTER TABLE products ALTER COLUMN price SET NOT NULL;

COMMIT;
```
**Output:**
```
BEGIN
UPDATE 1
ALTER TABLE
COMMIT
```
**Check:**
```sql
SELECT id, name, price FROM products ORDER BY id;
```
**Output:**
```
 id |    name    | price
----+------------+-------
  1 | Shirt      |    20
  2 | Hammer     |    10
  3 | Teddy bear |    12
(3 rows)
```
(`10` is just a placeholder for the exercise. In real data, use the true value.)

### A second gotcha: adding a new `NOT NULL` column
```sql
ALTER TABLE products ADD COLUMN sku VARCHAR(20) NOT NULL;
```
**Output:**
```
ERROR:  column "sku" of relation "products" contains null values
```
The new column starts empty for **every existing row**. Give it a default so those rows get a value (topic 99):
```sql
ALTER TABLE products ADD COLUMN sku VARCHAR(20) NOT NULL DEFAULT 'unknown';
```
**Output:**
```
ALTER TABLE
```

### A third gotcha: the "empty" value that isn't `NULL`
```sql
INSERT INTO products (name, department, price)
VALUES ('', 'Tools', 5);
```
**Output:**
```
INSERT 0 1
```
`NOT NULL` happily accepted an empty string. To stop that you need a `CHECK` (topic 102):
```sql
-- Clean up first, then the rule can be added
DELETE FROM products WHERE name = '';
ALTER TABLE products ADD CONSTRAINT name_not_blank CHECK (LENGTH(TRIM(name)) > 0);
```
**Output:**
```
DELETE 1
ALTER TABLE
```

### Troubleshooting summary
| Error | Cause | Fix |
|---|---|---|
| `column "x" of relation "t" contains null values` | Existing rows have `NULL` | Update or delete them, then retry |
| Same error on `ADD COLUMN ... NOT NULL` | New column is empty for old rows | Add a `DEFAULT` |
| Blank strings still get in | `''` isn't `NULL` | Add a `CHECK` |

> ⚠️ **Warning:** Replacing `NULL` with a fake value (`0`, `'unknown'`) can **hide** a real data problem. Make sure the placeholder is honest.

> ⚠️ **Warning:** On a big production table, `SET NOT NULL` scans the whole table and takes a lock while it does. Plan it for a quiet moment.

> 🏆 **Golden Rule:** **Clean the data first, then add the constraint.** Find the `NULL`s with `WHERE col IS NULL`, fix them, and wrap both steps in a transaction.




## 🎁 99. Default Column Values

**What it does:** `DEFAULT` gives a column a **fallback value** that Postgres uses whenever an insert doesn't provide one.
**Why it is used:** It saves you typing the same value over and over, and it works nicely with `NOT NULL`: the column is required, but you don't have to supply it every time.
**When to use it:** A standard starting value (a status of `'pending'`, a price of `999`, a flag of `true`) or an automatic timestamp.

**Analogy:** The **pre-selected option** in a dropdown. If you don't change it, that's what gets submitted.

### Set it up
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL,
  department VARCHAR(40) NOT NULL,
  price INTEGER NOT NULL DEFAULT 999,
  weight INTEGER,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```

### Leave the column out, and the default is used
```sql
INSERT INTO products (name, department)
VALUES ('Gift card', 'Misc');

SELECT id, name, price, weight, created_at FROM products;
```
**Output (your timestamp shows the current moment):**
```
INSERT 0 1
 id |   name    | price | weight |          created_at
----+-----------+-------+--------+-------------------------------
  1 | Gift card |   999 |        | 2026-11-20 10:30:00.123456+00
(1 row)
```
`price` became `999`, and `created_at` was filled in automatically. `weight` has no default, so it's `NULL`.

### Gotcha: an explicit `NULL` is **not** replaced by the default
```sql
INSERT INTO products (name, department, price)
VALUES ('Mug', 'Kitchen', NULL);
```
**Output:**
```
ERROR:  null value in column "price" of relation "products" violates not-null constraint
```
(A `DETAIL:` line with the failing row follows.) `NULL` is a value you **chose**, so the default doesn't step in.

**Ask for the default explicitly with the `DEFAULT` keyword:**
```sql
INSERT INTO products (name, department, price)
VALUES ('Mug', 'Kitchen', DEFAULT);
```
**Output:**
```
INSERT 0 1
```

### Change a default later: existing rows are not touched
```sql
ALTER TABLE products ALTER COLUMN weight SET DEFAULT 1;

INSERT INTO products (name, department)
VALUES ('Pen', 'Stationery');

SELECT name, price, weight FROM products ORDER BY id;
```
**Output:**
```
ALTER TABLE
INSERT 0 1
   name    | price | weight
-----------+-------+--------
 Gift card |   999 |
 Mug       |   999 |
 Pen       |   999 |      1
(3 rows)
```
Only **new** rows get `weight = 1`. The older rows stay `NULL`. If you want them changed too, run an `UPDATE`.

### Remove a default
```sql
ALTER TABLE products ALTER COLUMN weight DROP DEFAULT;
```
**Output:**
```
ALTER TABLE
```

### Useful kinds of default
| Default | Example | Result |
|---|---|---|
| A constant number | `price INTEGER DEFAULT 999` | `999` |
| A constant text | `status VARCHAR(20) DEFAULT 'pending'` | `'pending'` |
| A boolean | `is_active BOOLEAN DEFAULT true` | `t` |
| The current time | `created_at TIMESTAMPTZ DEFAULT NOW()` | The moment of the insert |
| The current date | `signed_up DATE DEFAULT CURRENT_DATE` | Today |
| A calculation | `expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '30 days'` | 30 days from the insert |

### Insert a row made entirely of defaults
```sql
CREATE TEMP TABLE counters (id SERIAL PRIMARY KEY, hits INTEGER DEFAULT 0);
INSERT INTO counters DEFAULT VALUES;
SELECT * FROM counters;
```
**Output:**
```
CREATE TABLE
INSERT 0 1
 id | hits
----+------
  1 |    0
(1 row)
```

### How `NOT NULL` and `DEFAULT` work together
| Setup | Behavior |
|---|---|
| `NOT NULL`, no default | The caller **must** provide a value |
| `NOT NULL DEFAULT x` | Required, but filled in automatically if left out |
| Nullable, `DEFAULT x` | Filled with `x` if left out. An explicit `NULL` still stores `NULL` |
| Nullable, no default | Left out means `NULL` |

> ⚠️ **Warning:** An **explicit `NULL` overrides the default.** Only leaving the column out (or writing `DEFAULT`) triggers it.

> ⚠️ **Warning:** `SET DEFAULT` doesn't change existing rows, which is easy to forget.

> ⚠️ **Warning:** A default isn't a rule. Nothing stops a later insert from supplying a different value. Use a `CHECK` if only certain values are allowed (topic 102).

> 🏆 **Golden Rule:** **`NOT NULL DEFAULT ...` is the standard pairing**: the column is always filled, and callers only specify it when it differs from the usual.




## 🔑 100. Applying a Unique Constraint to One Column

**What it does:** `UNIQUE` on a column means **no two rows may share the same value** there. A duplicate insert or update is rejected.
**Why it is used:** Some things must be one of a kind: usernames, email addresses, product names, order numbers. Without a rule, duplicates sneak in and later cause confusing bugs.
**When to use it:** Any column that identifies something but isn't the primary key.

**Analogy:** **Seat numbers in a theater.** Two tickets can't have the same seat, and the box office checks every new ticket against the ones already sold.

### Create the table with a unique column
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL UNIQUE,
  department VARCHAR(40) NOT NULL,
  price INTEGER NOT NULL
);

INSERT INTO products (name, department, price)
VALUES ('Shirt', 'Clothes', 20);
```
**Output:**
```
DROP TABLE
CREATE TABLE
INSERT 0 1
```

### A duplicate is rejected
```sql
INSERT INTO products (name, department, price)
VALUES ('Shirt', 'Toys', 15);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "products_name_key"
DETAIL:  Key (name)=(Shirt) already exists.
```
Postgres named the constraint for you: `<table>_<column>_key`. That name is what you'll see in errors, and what you'd use to drop it.

### Case matters
```sql
INSERT INTO products (name, department, price)
VALUES ('shirt', 'Clothes', 18);
```
**Output:**
```
INSERT 0 1
```
`'shirt'` and `'Shirt'` count as different. To block that, make a unique index on the lowercase value:
```sql
CREATE UNIQUE INDEX products_name_lower_idx ON products (LOWER(name));
```
That would fail here, because `'Shirt'` and `'shirt'` already exist, so clean the data first. Once it's in place, a case-different duplicate is rejected.

### `NULL`s don't collide
A unique column with no `NOT NULL` can hold **many `NULL`s**, because `NULL = NULL` is never true (topic 43):
```sql
CREATE TEMP TABLE people (email VARCHAR(50) UNIQUE);
INSERT INTO people (email) VALUES (NULL), (NULL);
```
**Output:**
```
CREATE TABLE
INSERT 0 2
```
In PostgreSQL 15 and newer you can change that:
```sql
CREATE TEMP TABLE people2 (email VARCHAR(50) UNIQUE NULLS NOT DISTINCT);
```
Now a second `NULL` is rejected.

### Inspect it
```sql
\d products
```
**Output (the part that matters):**
```
Indexes:
    "products_pkey" PRIMARY KEY, btree (id)
    "products_name_key" UNIQUE CONSTRAINT, btree (name)
```
A unique constraint is backed by an **index**, which also makes lookups by that column fast.

### Add or remove it later
```sql
ALTER TABLE products ADD CONSTRAINT products_name_key UNIQUE (name);
ALTER TABLE products DROP CONSTRAINT products_name_key;
```
**Gotcha:** adding it fails if duplicates already exist:
```
ERROR:  could not create unique index "products_name_key"
DETAIL:  Key (name)=(Shirt) is duplicated.
```
Fix the duplicates first (rename or delete them), then add the constraint, as in topic 98.

### Choose how to handle a duplicate in an insert
```sql
INSERT INTO products (name, department, price)
VALUES ('Shirt', 'Toys', 15)
ON CONFLICT (name) DO NOTHING;
```
**Output:**
```
INSERT 0 0
```
`INSERT 0 0` means nothing was inserted, and there was no error. (`ON CONFLICT ... DO UPDATE` lets you update the existing row instead.)

### `UNIQUE` vs `PRIMARY KEY`
| | `PRIMARY KEY` | `UNIQUE` |
|---|---|---|
| **Per table** | Exactly one | As many as you like |
| **Allows `NULL`?** | Never | Yes |
| **Purpose** | The row's identity | Another column that must not repeat |

> ⚠️ **Warning:** A failed insert still uses up a `SERIAL` id (topic 26), so a duplicate error can leave a gap. That's normal.

> ⚠️ **Warning:** Make unique columns `NOT NULL` too, unless you really want several `NULL`s.

> ⚠️ **Warning:** If two users submit the same name at the exact same moment, only the **database** can reliably stop it. An app-level "does this name exist?" check alone can lose that race.

> 🏆 **Golden Rule:** **If a value must be one of a kind, make it `UNIQUE`.** Let the database enforce it, then catch the error in your app and show a friendly message (topic 104).




## 🧬 101. Multi-Column Uniqueness

**What it does:** A unique constraint over **several columns together** means the **combination** must be unique, even though each column can repeat on its own.
**Why it is used:** Sometimes uniqueness only makes sense in context. A shirt can exist in Clothes and in Toys, but you shouldn't have two "Shirt" products in Clothes.
**When to use it:** Names unique per department, one review per user per product, a seat number unique per show.

**Analogy:** A **phone book entry** is unique by first name **and** last name together. Many people are named Sam, and many are named Lee, but only one is Sam Lee.

### Create the table
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL,
  department VARCHAR(40) NOT NULL,
  price INTEGER NOT NULL,
  UNIQUE (name, department)
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```
`UNIQUE (name, department)` is a **table-level** constraint, written on its own line.

### Try it
```sql
INSERT INTO products (name, department, price) VALUES ('Shirt', 'Clothes', 20);
INSERT INTO products (name, department, price) VALUES ('Shirt', 'Toys', 8);
```
**Output:**
```
INSERT 0 1
INSERT 0 1
```
Same name, different department, so it's allowed.
```sql
INSERT INTO products (name, department, price) VALUES ('Shirt', 'Clothes', 25);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "products_name_department_key"
DETAIL:  Key (name, department)=(Shirt, Clothes) already exists.
```
Same name **and** same department, so it's rejected.
```sql
SELECT id, name, department, price FROM products ORDER BY id;
```
**Output:**
```
 id | name  | department | price
----+-------+------------+-------
  1 | Shirt | Clothes    |    20
  2 | Shirt | Toys       |     8
(2 rows)
```

### Which combinations are allowed?
| name | department | Allowed? |
|---|---|---|
| Shirt | Clothes | ✅ first one |
| Shirt | Toys | ✅ different department |
| Hammer | Clothes | ✅ different name |
| Shirt | Clothes | ❌ **exact repeat** |

### Column-level vs table-level syntax
| Style | Syntax | Covers |
|---|---|---|
| Column-level | `name VARCHAR(40) UNIQUE` | **One** column |
| Table-level | `UNIQUE (name, department)` | **One or more** columns |

Name it yourself for clearer errors:
```sql
ALTER TABLE products DROP CONSTRAINT products_name_department_key;
ALTER TABLE products
  ADD CONSTRAINT unique_name_per_department UNIQUE (name, department);
```
**Output:**
```
ALTER TABLE
ALTER TABLE
```

### Two columns each unique on their own is different
| Rule | Meaning |
|---|---|
| `UNIQUE (name), UNIQUE (department)` | No name repeats **and** no department repeats (very restrictive) |
| `UNIQUE (name, department)` | Only the **pair** can't repeat |

### Composite primary keys follow the same idea
The join table from topic 23 uses one:
```sql
CREATE TEMP TABLE photos_tags (
  photo_id INTEGER,
  tag_id INTEGER,
  PRIMARY KEY (photo_id, tag_id)
);

INSERT INTO photos_tags VALUES (1, 1), (1, 2), (2, 1);
INSERT INTO photos_tags VALUES (1, 1);
```
**Output:**
```
CREATE TABLE
INSERT 0 3
ERROR:  duplicate key value violates unique constraint "photos_tags_pkey"
DETAIL:  Key (photo_id, tag_id)=(1, 1) already exists.
```
A photo can have many tags and a tag many photos, but the **same pairing** can't be recorded twice.

> ⚠️ **Warning:** **`NULL` in any of the columns lets duplicates through**, because `NULL` never equals `NULL`. `('Shirt', NULL)` can be inserted many times. Make the columns `NOT NULL` if that matters.

> ⚠️ **Warning:** Column **order** in the constraint doesn't change what's allowed, but it affects which lookups the index speeds up. `UNIQUE (name, department)` helps searches by `name` more than by `department` alone.

> 🏆 **Golden Rule:** **Use multi-column `UNIQUE` when the rule is "this combination only once."** The error message lists the whole combination, so name the constraint to make it readable.




## ✔️ 102. Adding a Validation Check

**What it does:** `CHECK (condition)` is a **custom rule for one row**. Any insert or update where the condition isn't true is rejected.
**Why it is used:** It's the most flexible constraint. It can say "price must be positive," "weight can't be negative," or "department must be one of these."
**When to use it:** Whenever a value must fall in a range, match a pattern, or come from a fixed set.

**Analogy:** A **height requirement sign** at a ride: "you must be at least 120 cm." Everyone is measured, and anyone who fails is turned away.

### Create the table with checks
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL,
  department VARCHAR(40) NOT NULL,
  price INTEGER NOT NULL CHECK (price > 0),
  weight INTEGER CHECK (weight >= 0)
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```

### A valid row, and an invalid one
```sql
INSERT INTO products (name, department, price, weight)
VALUES ('Shirt', 'Clothes', 20, 1);
```
**Output:**
```
INSERT 0 1
```
```sql
INSERT INTO products (name, department, price, weight)
VALUES ('Hammer', 'Tools', 0, 2);
```
**Output:**
```
ERROR:  new row for relation "products" violates check constraint "products_price_check"
DETAIL:  Failing row contains (2, Hammer, Tools, 0, 2).
```
Postgres named it `<table>_<column>_check`.

### A `NULL` passes a check
```sql
INSERT INTO products (name, department, price, weight)
VALUES ('Pen', 'Stationery', 3, NULL);
```
**Output:**
```
INSERT 0 1
```
`NULL >= 0` is **unknown**, not false, and a `CHECK` only rejects a row when the condition is **false**. That's why `price` also has `NOT NULL`.

### Name your constraints
```sql
CREATE TABLE products_named (
  id SERIAL PRIMARY KEY,
  price INTEGER NOT NULL CONSTRAINT price_positive CHECK (price > 0)
);
INSERT INTO products_named (price) VALUES (-5);
```
**Output:**
```
CREATE TABLE
ERROR:  new row for relation "products_named" violates check constraint "price_positive"
DETAIL:  Failing row contains (1, -5).
```
`price_positive` tells you the broken rule at a glance. (Clean up with `DROP TABLE products_named;`.)

### Add a check to an existing table
```sql
ALTER TABLE products
  ADD CONSTRAINT dept_allowed
  CHECK (department IN ('Clothes', 'Tools', 'Stationery', 'Toys'));
```
**Output:**
```
ALTER TABLE
```
```sql
INSERT INTO products (name, department, price) VALUES ('Sofa', 'Furniture', 300);
```
**Output:**
```
ERROR:  new row for relation "products" violates check constraint "dept_allowed"
DETAIL:  Failing row contains (4, Sofa, Furniture, 300, null).
```
**Gotcha:** if existing rows already break the rule, adding it fails:
```
ERROR:  check constraint "dept_allowed" of relation "products" is violated by some row
```
Find the bad rows with a `SELECT ... WHERE NOT (condition)`, fix them, and try again (topic 98).

### Fix the "blank name" gap from topics 97 and 98
```sql
ALTER TABLE products
  ADD CONSTRAINT name_not_blank CHECK (LENGTH(TRIM(name)) > 0);

INSERT INTO products (name, department, price) VALUES ('   ', 'Tools', 5);
```
**Output:**
```
ALTER TABLE
ERROR:  new row for relation "products" violates check constraint "name_not_blank"
DETAIL:  Failing row contains (5, , Tools, 5, null).
```

### What a check can test
| Rule | `CHECK` |
|---|---|
| Positive number | `CHECK (price > 0)` |
| Range | `CHECK (rating BETWEEN 1 AND 5)` |
| One of a list | `CHECK (status IN ('draft', 'live'))` |
| Minimum length | `CHECK (LENGTH(password) >= 8)` |
| Not blank | `CHECK (LENGTH(TRIM(name)) > 0)` |
| Pattern | `CHECK (code ~ '^[A-Z]{3}[0-9]{3}$')` |

### Remove one
```sql
ALTER TABLE products DROP CONSTRAINT dept_allowed;
```
**Output:**
```
ALTER TABLE
```

### What a check cannot do
| Not allowed | Instead use |
|---|---|
| Look at **other rows** or other tables (a subquery) | A foreign key, a trigger, or a unique constraint |
| Depend on values that **change over time** (`NOW()`) | A default, or application logic |

> ⚠️ **Warning:** **`NULL` passes a `CHECK`.** Pair it with `NOT NULL` whenever a missing value would be a problem.

> ⚠️ **Warning:** A fixed list in a check (`IN ('Clothes', ...)`) is hard to change later. If the list grows often, a separate table with a foreign key (topic 27) is easier to maintain.

> ⚠️ **Warning:** `CHECK` rejects only when the condition is **false**. If your logic has holes, bad rows with an unknown result get through.

> 🏆 **Golden Rule:** **Write the rule as a true statement about every valid row.** `CHECK (price > 0)` reads as "every price is positive." Name it, and add `NOT NULL` if empty values aren't allowed.