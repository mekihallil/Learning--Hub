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




## 🔀 84. The Case Keyword

**What it does:** `CASE` is SQL's **if / else if / else**. It checks conditions in order and returns the value of the **first one that is true**.
**Why it is used:** It lets a query produce different values for different rows: labels, categories, or conditional counts.
**When to use it:** Turning numbers into labels, replacing codes with readable text, and counting or summing only some rows.

**Analogy:** A traffic light controller reading a list of rules from the top: "if red, stop; else if yellow, slow; else go." It acts on the **first rule that matches** and ignores the rest.

### Searched `CASE` (the flexible form)
```sql
SELECT
  name,
  price,
  CASE
    WHEN price > 300 THEN 'high'
    WHEN price > 200 THEN 'medium'
    ELSE 'low'
  END AS tier
FROM phones;
```
**Output:**
```
    name     | price | tier
-------------+-------+--------
 N1280       |   199 | low
 iPhone 5S   |   399 | high
 Galaxy S5   |   299 | medium
 S5620 Monte |   250 | medium
(4 rows)
```

| Part | Meaning |
|---|---|
| `CASE` | Start |
| `WHEN condition THEN value` | If true, return this value |
| `ELSE value` | If nothing matched, return this |
| `END` | Finish (forgetting it is a syntax error) |

### Simple `CASE` (compare one column to values)
```sql
SELECT
  name,
  CASE manufacturer
    WHEN 'Apple' THEN 'United States'
    WHEN 'Nokia' THEN 'Finland'
    WHEN 'Samsung' THEN 'South Korea'
    ELSE 'Unknown'
  END AS country
FROM phones;
```
**Output:**
```
    name     |    country
-------------+---------------
 N1280       | Finland
 iPhone 5S   | United States
 Galaxy S5   | South Korea
 S5620 Monte | South Korea
(4 rows)
```
This is shorthand for repeated `WHEN manufacturer = '...'` tests.

### Without `ELSE`, unmatched rows get `NULL`
```sql
SELECT name, CASE WHEN price > 300 THEN 'high' END AS tier FROM phones;
```
**Output:**
```
    name     | tier
-------------+------
 N1280       |
 iPhone 5S   | high
 Galaxy S5   |
 S5620 Monte |
(4 rows)
```

### Order matters: the first true `WHEN` wins
```sql
SELECT name, price,
  CASE
    WHEN price > 200 THEN 'medium'
    WHEN price > 300 THEN 'high'
    ELSE 'low'
  END AS tier
FROM phones;
```
**Output (wrong!):**
```
    name     | price |  tier
-------------+-------+--------
 N1280       |   199 | low
 iPhone 5S   |   399 | medium
 Galaxy S5   |   299 | medium
 S5620 Monte |   250 | medium
(4 rows)
```
The iPhone (399) matches `> 200` first, so it never reaches the `> 300` test. **Put the most specific test first.**

### `CASE` in a grouped query
```sql
SELECT
  CASE WHEN user_id IS NULL THEN 'orphan' ELSE 'owned' END AS status,
  COUNT(*) AS photos
FROM photos
GROUP BY status;
```
**Output (row order may differ):**
```
 status | photos
--------+--------
 owned  |      7
 orphan |      1
(2 rows)
```
PostgreSQL lets `GROUP BY` use the output alias here, which `WHERE` and `HAVING` can't.

### Conditional counting: several totals in one pass
```sql
SELECT
  COUNT(CASE WHEN price > 300 THEN 1 END) AS pricey,
  COUNT(CASE WHEN price <= 300 THEN 1 END) AS affordable
FROM phones;
```
**Output:**
```
 pricey | affordable
--------+------------
      1 |          3
(1 row)
```
Rows that don't match return `NULL`, and `COUNT` skips `NULL`s (topic 53).

**The PostgreSQL shortcut for the same thing:**
```sql
SELECT
  COUNT(*) FILTER (WHERE price > 300) AS pricey,
  COUNT(*) FILTER (WHERE price <= 300) AS affordable
FROM phones;
```
The same output. `FILTER` is PostgreSQL-friendly, but `CASE` works in every database.

### Every branch must return a compatible type
```sql
SELECT CASE WHEN price > 300 THEN 'high' ELSE 0 END FROM phones;
```
**Output:**
```
ERROR:  invalid input syntax for type integer: "high"
```
Text and numbers can't share one result column.

**Where `CASE` can appear:**

| Place | Use |
|---|---|
| `SELECT` list | Labels and categories |
| Inside an aggregate | Conditional `COUNT` and `SUM` |
| `ORDER BY` | Custom sort orders |
| `WHERE` | Rare, but allowed |

> ⚠️ **Warning:** `CASE` stops at the **first true** `WHEN`. List specific conditions before general ones.

> ⚠️ **Warning:** Always add an `ELSE`, unless you actually want `NULL` for unmatched rows.

> ⚠️ **Warning:** A simple `CASE x WHEN NULL THEN ...` **never matches**, because `NULL = NULL` isn't true (topic 11). Use `CASE WHEN x IS NULL THEN ...`.

> 🏆 **Golden Rule:** **`CASE` is a column-sized `if / else if / else`.** Write conditions from most specific to least, include an `ELSE`, and end with `END`.


## 💾 85. PostgreSQL Installation on macOS, Linux and Windows

**What it does:** Walks through installing **PostgreSQL** on your own computer, so you can run every query in these notes yourself.
**Why it is used:** Reading SQL is only half the skill. Running it against a real database shows you the errors and the outputs.
**When to use it:** Once, before you start practicing. If you already have Postgres, jump to **Verify the install** below.

**Analogy:** Building the workshop before the lessons. PostgreSQL is the workshop, `psql` is the front door, and SQL is the tool set.

> **Tip:** This is a setup topic. In your README you may want to move it to a **"Getting Started"** section at the very top, before topic 1.

**Which version?** Pick the **latest stable major version** (**18** at the time of writing). Check [postgresql.org/download](https://www.postgresql.org/download/) for the current one. The commands below use `18`, so replace the number if a newer major version is out.

**Three things you'll set or choose, on every system:**

| Item | Typical value |
|---|---|
| Superuser name | `postgres` |
| Superuser password | One **you** choose, so write it down |
| Port | `5432` (the default) |

---

### 🪟 Windows

**Step 1.** Go to [postgresql.org/download/windows](https://www.postgresql.org/download/windows/) and download the **installer from EDB** for your version.
**Step 2.** Run the installer. Keep the default components ticked:

| Component | Why |
|---|---|
| PostgreSQL Server | The database itself |
| pgAdmin 4 | The graphical tool (topic 86) |
| Command Line Tools | Includes `psql` |
| Stack Builder | Optional extras, safe to untick |

**Step 3.** Keep the default **data directory**.
**Step 4.** Enter a **password** for the `postgres` superuser, and don't forget it.
**Step 5.** Keep the **port** as `5432` and the default **locale**, then click through to finish.

**Open `psql`:** press Start, search for **SQL Shell (psql)**, and press **Enter** at each prompt to accept the defaults (Server `localhost`, Database `postgres`, Port `5432`, Username `postgres`). Then type your password.

**Optional: use `psql` from any terminal** by adding the `bin` folder to your `PATH`, for example:
```
C:\Program Files\PostgreSQL\18\bin
```
Then open a **new** PowerShell window and run:
```powershell
psql -U postgres
```

---

### 🍎 macOS

Choose **one** of three options.

**Option A: the EDB graphical installer (easiest, includes pgAdmin).**
Download it from [postgresql.org/download/macosx](https://www.postgresql.org/download/macosx/), run it, and follow the same steps as Windows (set a password, keep port `5432`). PostgreSQL 18 needs macOS 13 or newer.

**Option B: Postgres.app.**
A simple menubar app. Open it and the server runs, close it and the server stops. Download it from [postgresapp.com](https://postgresapp.com).

**Option C: Homebrew (good for developers).**
```bash
brew install postgresql@18
brew services start postgresql@18
```
Version formulas are named `postgresql@NN`. Check what's available with:
```bash
brew search postgresql
```
Versioned formulas aren't linked into your `PATH` automatically. Add them:
```bash
# Apple Silicon
echo 'export PATH="/opt/homebrew/opt/postgresql@18/bin:$PATH"' >> ~/.zshrc
# Intel Mac: use /usr/local/opt/postgresql@18/bin instead
source ~/.zshrc
```
With Homebrew, the superuser is **your macOS username** and there's **no password** by default. Connect with:
```bash
psql postgres
```

---

### 🐧 Linux

**Debian or Ubuntu:**
```bash
sudo apt update
sudo apt install postgresql
sudo systemctl status postgresql
```
This installs the version your distribution ships. For the **newest** version, add the official PostgreSQL APT repository first (instructions at [apt.postgresql.org](https://wiki.postgresql.org/wiki/Apt)).

**Fedora, RHEL, Rocky, AlmaLinux:**
```bash
sudo dnf install postgresql-server
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
```
For a newer version on RHEL-family systems, use the PostgreSQL Yum repository from [postgresql.org/download/linux](https://www.postgresql.org/download/linux/).

**Connect as the `postgres` user:**
```bash
sudo -u postgres psql
```
**Give the `postgres` user a password** (you'll need it for pgAdmin in topic 86):
```sql
ALTER USER postgres PASSWORD 'choose-a-strong-password';
```
**Output:**
```
ALTER ROLE
```

---

### ✅ Verify the install (every system)
```bash
psql --version
```
**Output:**
```
psql (PostgreSQL) 18.x
```
Inside `psql`:
```sql
SELECT version();
```
**Output:**
```
                          version
------------------------------------------------------------
 PostgreSQL 18.x on ... (details depend on your system)
(1 row)
```
**Create your practice database (topic 20):**
```sql
CREATE DATABASE photo_app;
```
**Output:**
```
CREATE DATABASE
```
```sql
\c photo_app
```
**Output:**
```
You are now connected to database "photo_app" as user "postgres".
```
**Leave `psql`:**
```
\q
```

**Handy `psql` commands (they start with `\`):**

| Command | What it does |
|---|---|
| `\l` | List databases |
| `\c name` | Connect to a database |
| `\dt` | List tables |
| `\d table` | Describe a table |
| `\q` | Quit |

### Troubleshooting
| Message | Likely cause and fix |
|---|---|
| `psql: command not found` | `psql` isn't on your `PATH`. Add the `bin` folder (see above) and open a **new** terminal |
| `password authentication failed for user "postgres"` | Wrong password. Reset it with `ALTER USER postgres PASSWORD '...';` |
| `connection refused` / `could not connect to server` | The server isn't running. Start it (`brew services start ...`, `sudo systemctl start postgresql`, or the Windows service) |
| `role "yourname" does not exist` | You're connecting as a user that doesn't exist. On Homebrew use `psql postgres`, elsewhere use `-U postgres` |
| `Peer authentication failed for user "postgres"` (Linux) | Use `sudo -u postgres psql` instead of plain `psql` |
| Port `5432` already in use | Another Postgres is running. Stop it, or install on a different port |

> ⚠️ **Warning:** The `postgres` account is a **superuser** that can do anything. Choose a strong password, and never use it as the login for a real application. Create a separate user with limited rights.

> ⚠️ **Warning:** Don't expose port `5432` to the internet. For learning, keep the server reachable from your own computer only (the default).

> ⚠️ **Warning:** Installer screens and package names change between versions. If a step doesn't match, follow the [official download page](https://www.postgresql.org/download/) for your system.

> 🏆 **Golden Rule:** **Install, verify with `SELECT version();`, then create a practice database.** If you can connect and run a query, you're ready.