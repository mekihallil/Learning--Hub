## 🏗️ 129. Creating Tables with Checks

**What it does:** Starts the final build with a **clean reset** and the `users` table, including its `NOT NULL`, `DEFAULT`, `UNIQUE`, and `CHECK` rules from topics 95 to 104.
**Why it is used:** After many experiments, one clean script that builds everything from nothing is the only way to know your schema really works.
**When to use it:** Any time you want a fresh, known-good database.

**Analogy:** Clearing the workbench before the real build. You lay the parts out in order, and nothing left over from a trial run can interfere.

### Reset
```sql
\c social_app
```
**Output:**
```
You are now connected to database "social_app" as user "postgres".
```
```sql
DROP TABLE IF EXISTS
  followers, post_hashtags, hashtags, tags, photo_tags, caption_tags,
  likes, comments, posts, users
CASCADE;
```
**Output:**
```
DROP TABLE
```
(Any `NOTICE: table "..." does not exist, skipping` lines are fine.) The names are listed children-first, and `CASCADE` removes the foreign keys that point at them.

### The `users` table
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  username VARCHAR(30) NOT NULL UNIQUE,
  display_name VARCHAR(50),
  bio VARCHAR(400),
  avatar VARCHAR(200),
  email VARCHAR(255),
  phone VARCHAR(25),
  is_private BOOLEAN NOT NULL DEFAULT false,
  status VARCHAR(15) NOT NULL DEFAULT 'active',
  last_login_at TIMESTAMPTZ,
  CONSTRAINT users_username_length CHECK (LENGTH(username) >= 3),
  CONSTRAINT users_need_contact CHECK (email IS NOT NULL OR phone IS NOT NULL),
  CONSTRAINT users_status_allowed CHECK (status IN ('active', 'suspended', 'deleted'))
);
```
**Output:**
```
CREATE TABLE
```

| Rule | Tool | Topic |
|---|---|---|
| Username required and unique | `NOT NULL UNIQUE` | 97, 100 |
| At least 3 characters | `CHECK` on one column | 102 |
| Has an email **or** a phone | `CHECK` over two columns | 103 |
| Status from a fixed list | `CHECK ... IN (...)` | 102 |
| Timestamps fill themselves | `DEFAULT NOW()` | 99 |
| Flag never `NULL` | `BOOLEAN NOT NULL DEFAULT false` | 92 |

### Valid rows
```sql
INSERT INTO users (username, email, phone)
VALUES
  ('monahan93', 'monahan@example.com', NULL),
  ('pferrer', NULL, '555-0102'),
  ('si93onis', 'si93@example.com', NULL),
  ('99stroman', 'stroman@example.com', NULL);
```
**Output:**
```
INSERT 0 4
```

### Break the rules
**No way to be contacted:**
```sql
INSERT INTO users (username) VALUES ('nobody');
```
**Output:**
```
ERROR:  new row for relation "users" violates check constraint "users_need_contact"
```
**Too short:**
```sql
INSERT INTO users (username, email) VALUES ('ab', 'ab@example.com');
```
**Output:**
```
ERROR:  new row for relation "users" violates check constraint "users_username_length"
```
**A duplicate username:**
```sql
INSERT INTO users (username, email) VALUES ('pferrer', 'other@example.com');
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "users_username_key"
DETAIL:  Key (username)=(pferrer) already exists.
```

> ⚠️ **Warning:** `DROP TABLE ... CASCADE` **deletes the data** with no confirmation. Run this script only on a practice database.

> ⚠️ **Warning:** `updated_at` is set on insert but doesn't change by itself on an `UPDATE`. Set it in your query or with a trigger (a later topic).

> 🏆 **Golden Rule:** **Build parents first and drop children first.** Name every `CHECK` so failures tell you exactly which rule broke.