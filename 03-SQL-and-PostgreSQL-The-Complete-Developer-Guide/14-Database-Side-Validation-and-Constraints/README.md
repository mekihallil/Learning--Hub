



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





## 🔗 103. Checks Over Multiple Columns

**What it does:** A **table-level `CHECK`** compares **two or more columns of the same row**, such as "the delivery date must come after the order date."
**Why it is used:** Some rules only make sense **between** columns. A single-column check can't express them.
**When to use it:** Start before end, sale price below regular price, "at least one contact method," or "if status is shipped then a ship date must exist."

**Analogy:** A **hotel booking** where check-out must be after check-in. Neither date is wrong on its own, but the **pair** can be.

### Create the table
```sql
DROP TABLE IF EXISTS orders;

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL,
  created_at TIMESTAMP NOT NULL,
  est_delivery TIMESTAMP NOT NULL,
  CONSTRAINT delivery_after_creation CHECK (created_at < est_delivery)
);
```
**Output:**
```
NOTICE:  table "orders" does not exist, skipping
DROP TABLE
CREATE TABLE
```
The `CHECK` sits on its own line, **after** the columns, so it can mention several of them.

### Valid and invalid rows
```sql
INSERT INTO orders (name, created_at, est_delivery)
VALUES ('Laptop', '2026-11-20 10:00', '2026-11-25 10:00');
```
**Output:**
```
INSERT 0 1
```
```sql
INSERT INTO orders (name, created_at, est_delivery)
VALUES ('Phone', '2026-11-20 10:00', '2026-11-18 10:00');
```
**Output:**
```
ERROR:  new row for relation "orders" violates check constraint "delivery_after_creation"
DETAIL:  Failing row contains (2, Phone, 2026-11-20 10:00:00, 2026-11-18 10:00:00).
```
The same moment is also rejected, because the rule uses `<`, not `<=`:
```sql
INSERT INTO orders (name, created_at, est_delivery)
VALUES ('Tablet', '2026-11-20 10:00', '2026-11-20 10:00');
```
**Output:**
```
ERROR:  new row for relation "orders" violates check constraint "delivery_after_creation"
```

### Checks also run on `UPDATE`
```sql
UPDATE orders SET est_delivery = '2026-11-01 10:00' WHERE name = 'Laptop';
```
**Output:**
```
ERROR:  new row for relation "orders" violates check constraint "delivery_after_creation"
```

### A "sale price" rule
```sql
CREATE TEMP TABLE items (
  id SERIAL PRIMARY KEY,
  price INTEGER NOT NULL,
  sale_price INTEGER,
  CONSTRAINT sale_below_price CHECK (sale_price IS NULL OR sale_price < price)
);

INSERT INTO items (price, sale_price) VALUES (100, 80), (100, NULL);
INSERT INTO items (price, sale_price) VALUES (100, 120);
```
**Output:**
```
CREATE TABLE
INSERT 0 2
ERROR:  new row for relation "items" violates check constraint "sale_below_price"
DETAIL:  Failing row contains (3, 100, 120).
```
`sale_price IS NULL OR ...` allows items with no sale. Without it, a `NULL` comparison would be unknown (and pass anyway), but spelling it out makes the intent clear.

### "At least one of these must be filled in"
```sql
CREATE TEMP TABLE contacts (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL,
  email VARCHAR(80),
  phone VARCHAR(20),
  CONSTRAINT needs_contact CHECK (email IS NOT NULL OR phone IS NOT NULL)
);

INSERT INTO contacts (name, email) VALUES ('Sam', 'sam@example.com');
INSERT INTO contacts (name) VALUES ('Alex');
```
**Output:**
```
CREATE TABLE
INSERT 0 1
ERROR:  new row for relation "contacts" violates check constraint "needs_contact"
DETAIL:  Failing row contains (2, Alex, null, null).
```

### Column-level vs table-level checks
| | Column-level | Table-level |
|---|---|---|
| **Written** | Next to the column | After all the columns |
| **Can mention** | Normally just that column | **Any columns** of the row |
| **Example** | `price INTEGER CHECK (price > 0)` | `CHECK (created_at < est_delivery)` |
| **Auto name** | `table_column_check` | `table_check` (or one you choose) |

### One check or several?
```sql
-- One combined rule
CHECK (price > 0 AND weight >= 0)

-- Two separate rules: you learn exactly which one broke
CONSTRAINT price_positive CHECK (price > 0),
CONSTRAINT weight_not_negative CHECK (weight >= 0)
```
Separate, named checks give **more precise errors**.

### Clean up the temporary demos
The `TEMP` tables disappear on their own. Drop the real one when you're done:
```sql
DROP TABLE IF EXISTS orders;
```
**Output:**
```
DROP TABLE
```

> ⚠️ **Warning:** **If either column is `NULL`, the comparison is unknown and the row passes.** Make both columns `NOT NULL` (as above), or write the `NULL` case into the rule.

> ⚠️ **Warning:** Use `<` or `<=` deliberately. Decide whether "same moment" should be allowed.

> ⚠️ **Warning:** A check only sees **one row**. It can't compare a row with others, so "no two orders overlap" needs a different tool (an exclusion constraint, which comes later).

> 🏆 **Golden Rule:** **Use a table-level `CHECK` when the rule is about the relationship between columns.** Name it, and keep the columns `NOT NULL` so the rule can't be dodged.




## 🧭 104. So Where Are We Applying Validation

**What it does:** Pulls the whole section together and answers the big question: **which layer should validate what?** The answer is "all of them, for different reasons," with the database as the last line of defense.
**Why it is used:** You now have validation in three places (the browser, your Next.js app, and the database). Knowing each layer's job avoids both gaps and duplicated effort.
**When to use it:** When designing any feature that accepts user input.

**Analogy:** Airport security. There's a ticket check at the **entrance** (browser), a **security scan** (your app), and a **boarding gate** (the database). Each stops problems the others might miss, and only the last one is truly impossible to walk around.

### The three layers
| Layer | Tools | Strength | Weakness |
|---|---|---|---|
| **1. Browser / form** | `required`, `min`, `type="email"` | Instant feedback | A user can bypass it |
| **2. Your app** | Zod and Server Actions (Next.js topics 62, 73, 74) | Friendly, specific messages, and it can check permissions | Only guards data that comes **through the app** |
| **3. Database** | `NOT NULL`, `DEFAULT`, `UNIQUE`, `CHECK`, keys | **Cannot be bypassed**, and it protects data from every source | Error messages are technical |

### What each layer is best at
| Rule | Browser | App | Database |
|---|---|---|---|
| "Name is required" | ✅ hint | ✅ message | ✅ `NOT NULL` |
| "Price is positive" | ✅ `min="1"` | ✅ Zod | ✅ `CHECK (price > 0)` |
| "Name is unique per department" | ❌ can't know | ⚠️ can check, but can race | ✅ **only the database is reliable** |
| "Order belongs to a real user" | ❌ | ⚠️ | ✅ foreign key |
| "Is this user allowed to do this?" | ❌ | ✅ | ⚠️ limited |
| "Friendly wording" | ✅ | ✅ | ❌ |

**Uniqueness is the clearest example.** Two requests can both pass an "does it exist yet?" check in the app and then both insert. Only a `UNIQUE` constraint guarantees one wins and the other fails.

### The full table with every rule from this section
```sql
DROP TABLE IF EXISTS products;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(40) NOT NULL,
  department VARCHAR(40) NOT NULL,
  price INTEGER NOT NULL DEFAULT 999
    CONSTRAINT price_positive CHECK (price > 0),
  weight INTEGER
    CONSTRAINT weight_not_negative CHECK (weight >= 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT unique_name_per_department UNIQUE (name, department),
  CONSTRAINT name_not_blank CHECK (LENGTH(TRIM(name)) > 0)
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```
```sql
\d products
```
**Output:**
```
                                      Table "public.products"
   Column   |           Type           | Collation | Nullable |               Default
------------+--------------------------+-----------+----------+--------------------------------------
 id         | integer                  |           | not null | nextval('products_id_seq'::regclass)
 name       | character varying(40)    |           | not null |
 department | character varying(40)    |           | not null |
 price      | integer                  |           | not null | 999
 weight     | integer                  |           |          |
 created_at | timestamp with time zone |           | not null | now()
Indexes:
    "products_pkey" PRIMARY KEY, btree (id)
    "unique_name_per_department" UNIQUE CONSTRAINT, btree (name, department)
Check constraints:
    "name_not_blank" CHECK (length(btrim(name::text)) > 0)
    "price_positive" CHECK (price > 0)
    "weight_not_negative" CHECK (weight >= 0)
```
(Postgres rewrites `TRIM` as `btrim` in this listing. It's the same thing.)

### The constraint cheat sheet
| Constraint | Guarantees | Topic |
|---|---|---|
| `NOT NULL` | A value is always present | 97 and 98 |
| `DEFAULT` | A fallback when no value is given | 99 |
| `UNIQUE` | No duplicates (one column or a combination) | 100 and 101 |
| `CHECK` | A custom rule holds for each row | 102 and 103 |
| `PRIMARY KEY` | Unique, never `NULL`, identifies the row | 24 |
| `FOREIGN KEY` | The link points at a real row | 25 to 27 |

### Reading the database's errors in your app
Every constraint failure carries a standard **error code** (the SQLSTATE):

| Code | Meaning | Example message |
|---|---|---|
| `23502` | Not-null violation | `null value in column ... violates not-null constraint` |
| `23503` | Foreign key violation | `violates foreign key constraint` |
| `23505` | Unique violation | `duplicate key value violates unique constraint` |
| `23514` | Check violation | `violates check constraint` |

**With Prisma in a Server Action** (the pattern from topic 68), catch the error and turn it into a friendly message:

| Problem | Prisma error code |
|---|---|
| Unique constraint failed | `P2002` |
| Foreign key constraint failed | `P2003` |
| Null constraint violation | `P2011` |

```tsx
import { Prisma } from "@prisma/client";

try {
  await db.product.create({ data: { name, department, price } });
} catch (err) {
  if (err instanceof Prisma.PrismaClientKnownRequestError && err.code === "P2002") {
    return { errors: { name: ["That product already exists in this department"] } };
  }
  return { errors: { _form: ["Something went wrong. Please try again."] } };
}
```

### A recommended workflow
| Step | What to do |
|---|---|
| 1 | Decide the rules at design time (topic 95) |
| 2 | Put the **hard guarantees** in the database (constraints) |
| 3 | Mirror them in the app with Zod for friendly messages |
| 4 | Add light hints in the form (`required`, `min`) |
| 5 | In the app, **catch constraint errors** and translate them |

### Clean up the practice tables
```sql
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS orders;
```
**Output:**
```
DROP TABLE
DROP TABLE
```

> ⚠️ **Warning:** **Prisma's schema file doesn't express `CHECK` constraints directly.** Teams usually add them with a hand-edited SQL migration (for example `npx prisma migrate dev --create-only`, then editing the generated file). Check the Prisma docs for your version, since this changes.

> ⚠️ **Warning:** The `Prisma` import path depends on your generator setup, and newer Prisma versions (including 7) can generate the client into a custom folder. Import from **that generated path** if `@prisma/client` doesn't export it (topic 76).

> ⚠️ **Warning:** Don't show raw database error text to users. It exposes table and column names. Catch the error, log it on the server, and return a clean message.

> 🏆 **Golden Rule:** **Validate in the browser for convenience, in the app for clarity, and in the database for certainty.** If a rule must **never** be broken, it belongs in a constraint.