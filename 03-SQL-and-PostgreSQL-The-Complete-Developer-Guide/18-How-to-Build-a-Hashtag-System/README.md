## #️⃣ 123. Designing a Hashtag System

**What it does:** Lists what a hashtag system must do, and shows how hashtags are **found inside caption text** before they are stored.
**Why it is used:** Searching raw text for `#sunrise` is slow and gives wrong matches. A proper design stores each tag as data.
**When to use it:** Before building the tables in topics 124 and 125.

**Analogy:** Hashtags are **index cards** pulled out of a book. You can search the cards instead of re-reading every page.

### Requirements
| # | Requirement |
|---|---|
| H1 | A post can have **many** hashtags |
| H2 | A hashtag can be on **many** posts |
| H3 | `#Travel` and `#travel` are the **same** tag |
| H4 | Clicking a hashtag lists its posts |
| H5 | We can rank the **most used** hashtags |
| H6 | Deleting a post removes its hashtag links |

### Give three posts some hashtags
```sql
UPDATE posts SET caption = 'Sunrise #sunrise #travel' WHERE id = 1;
UPDATE posts SET caption = 'Lunch #food' WHERE id = 2;
UPDATE posts SET caption = 'Sunset #sunrise2 #travel' WHERE id = 3;
```
**Output:**
```
UPDATE 1
UPDATE 1
UPDATE 1
```

### Why not just search the caption?
```sql
SELECT id, caption FROM posts WHERE caption ILIKE '%#sunrise%' ORDER BY id;
```
**Output:**
```
 id |          caption
----+---------------------------
  1 | Sunrise #sunrise #travel
  3 | Sunset #sunrise2 #travel
(2 rows)
```
Post 3 matched by accident, because `#sunrise2` **contains** `#sunrise`. A text search also can't use a normal index, so it scans every row.

### Pull the tags out of the text
```sql
SELECT id, lower((regexp_matches(caption, '#(\w+)', 'g'))[1]) AS tag
FROM posts
ORDER BY id, tag;
```
**Output:**
```
 id |   tag
----+----------
  1 | sunrise
  1 | travel
  2 | food
  3 | sunrise2
  3 | travel
(5 rows)
```
`lower()` handles H3. In a real app, your **Server Action** does this when a caption is saved, then writes the tags to the database.

### The design options
| Option | Idea | Topic |
|---|---|---|
| A | Search the caption text | ❌ Slow and inexact (shown above) |
| B | One table of (post, title) | 124 |
| C | A `hashtags` table plus a join table | 125 |

> ⚠️ **Warning:** The regular expression `#(\w+)` is a simple example. Real hashtag rules (accents, emoji, length limits) need more care.

> 🏆 **Golden Rule:** **Parse the text once when it's saved, and store the tags as rows.** Never search free text for structured data.


## 🗂️ 124. Tables for Hashtags

**What it does:** Builds the **simplest** hashtag design: one table that stores each (post, hashtag title) pair directly. It works, but it shows exactly where the trouble starts.
**Why it is used:** Seeing the weakness first makes the better design in topic 125 easy to appreciate.
**When to use it:** Only for tiny apps or prototypes.

**Analogy:** Writing the tag **name** on every sticky note, instead of pointing to one master list.

### The table
```sql
DROP TABLE IF EXISTS simple_hashtags;

CREATE TABLE simple_hashtags (
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  title VARCHAR(30) NOT NULL,
  PRIMARY KEY (post_id, title)
);

INSERT INTO simple_hashtags (post_id, title)
VALUES (1, 'sunrise'), (1, 'travel'), (2, 'food'), (3, 'sunrise2'), (3, 'travel');
```
**Output:**
```
DROP TABLE
CREATE TABLE
INSERT 0 5
```
The composite primary key stops the same tag appearing twice on one post.

### It answers the basic questions
**Posts with `#travel`:**
```sql
SELECT p.id, p.caption
FROM simple_hashtags AS h
JOIN posts AS p ON p.id = h.post_id
WHERE h.title = 'travel'
ORDER BY p.id;
```
**Output:**
```
 id |          caption
----+---------------------------
  1 | Sunrise #sunrise #travel
  3 | Sunset #sunrise2 #travel
(2 rows)
```
**Most used hashtags:**
```sql
SELECT title, COUNT(*) AS posts
FROM simple_hashtags
GROUP BY title
ORDER BY posts DESC, title;
```
**Output:**
```
  title   | posts
----------+-------
 travel   |     2
 food     |     1
 sunrise  |     1
 sunrise2 |     1
(4 rows)
```

### Where it breaks
**Case and typos slip in** (H3 from topic 123):
```sql
BEGIN;
INSERT INTO simple_hashtags (post_id, title) VALUES (2, 'Travel');
SELECT title, COUNT(*) FROM simple_hashtags WHERE LOWER(title) = 'travel' GROUP BY title ORDER BY title;
ROLLBACK;
```
**Output:**
```
BEGIN
INSERT 0 1
 title  | count
--------+-------
 Travel |     1
 travel |     2
(2 rows)
```
`Travel` and `travel` are now **two tags**. `ROLLBACK` removed the test row.

| Problem | Why |
|---|---|
| Case and typos create duplicate tags | Nothing defines what a valid tag is |
| Renaming a tag means updating **every** row | The text is copied onto each post |
| No place for tag facts (created date, banned, follower count) | A tag isn't an object, only text |
| Repeated text wastes space | `'travel'` is stored again for every post |

> ⚠️ **Warning:** This table is only for comparison. You'll drop it in topic 125.

> 🏆 **Golden Rule:** **If the same text is repeated across many rows, it probably deserves its own table.**




## ✅ 125. Including the Hashtag Table

**What it does:** Splits the design in two: a **`hashtags`** table that holds each tag **once**, and a **`post_hashtags`** join table that links posts to tags by id. It's the many-to-many pattern from topic 23.
**Why it is used:** Tags become real objects. They're unique, validated, renameable in one place, and cheap to count.
**When to use it:** The default for any hashtag, label, or category system.

**Analogy:** A **master list of tags** at the front desk, and each post only holds a ticket number pointing at an entry.

### The tables
```sql
DROP TABLE IF EXISTS simple_hashtags;

CREATE TABLE hashtags (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  title VARCHAR(30) NOT NULL UNIQUE,
  CONSTRAINT hashtags_title_format CHECK (title ~ '^[a-z0-9_]+$')
);

CREATE TABLE post_hashtags (
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  hashtag_id INTEGER NOT NULL REFERENCES hashtags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, hashtag_id)
);

CREATE INDEX post_hashtags_hashtag_id_idx ON post_hashtags (hashtag_id);
```
**Output:**
```
DROP TABLE
CREATE TABLE
CREATE TABLE
CREATE INDEX
```
| Rule | Tool |
|---|---|
| A tag exists once | `UNIQUE` on `title` |
| Only lowercase letters, digits, underscore | `CHECK` with a pattern |
| Same tag once per post | Composite primary key |
| Fast "posts for this tag" | Index on `hashtag_id` |
| Cleanup | `ON DELETE CASCADE` on both sides |

### Fill them
```sql
INSERT INTO hashtags (title) VALUES ('sunrise'), ('travel'), ('food'), ('sunrise2');

INSERT INTO post_hashtags (post_id, hashtag_id)
VALUES (1, 1), (1, 2), (2, 3), (3, 4), (3, 2);
```
**Output:**
```
INSERT 0 4
INSERT 0 5
```

### Break the rules
**Wrong case:**
```sql
INSERT INTO hashtags (title) VALUES ('Travel');
```
**Output:**
```
ERROR:  new row for relation "hashtags" violates check constraint "hashtags_title_format"
```
**A duplicate tag:**
```sql
INSERT INTO hashtags (title) VALUES ('travel');
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "hashtags_title_key"
DETAIL:  Key (title)=(travel) already exists.
```

### Find or create a tag in one statement
```sql
INSERT INTO hashtags (title) VALUES ('travel')
ON CONFLICT (title) DO UPDATE SET title = EXCLUDED.title
RETURNING id;
```
**Output:**
```
 id
----
  2

INSERT 0 1
```
It returns the existing id, or creates the tag and returns the new one. (`DO NOTHING` would return no row on a conflict.)

### Query it
**Most used hashtags:**
```sql
SELECT h.title, COUNT(*) AS posts
FROM hashtags AS h
JOIN post_hashtags AS ph ON ph.hashtag_id = h.id
GROUP BY h.id
ORDER BY posts DESC, h.title;
```
**Output:**
```
  title   | posts
----------+-------
 travel   |     2
 food     |     1
 sunrise  |     1
 sunrise2 |     1
(4 rows)
```
**Rename a tag in one row:**
```sql
BEGIN;
UPDATE hashtags SET title = 'journey' WHERE title = 'travel';
SELECT p.id, h.title FROM post_hashtags ph
JOIN hashtags h ON h.id = ph.hashtag_id
JOIN posts p ON p.id = ph.post_id
WHERE h.id = 2 ORDER BY p.id;
ROLLBACK;
```
**Output:**
```
BEGIN
UPDATE 1
 id |  title
----+---------
  1 | journey
  3 | journey
(2 rows)
```
One `UPDATE` changed it everywhere. `ROLLBACK` undid the test.

### Compared with topic 124
| | One table (124) | Two tables (125) |
|---|---|---|
| Unique tags | ❌ | ✅ |
| Rename a tag | Update many rows | **One row** |
| Tag facts | Nowhere to put them | Add columns to `hashtags` |
| Extra table to join | No | Yes |

> ⚠️ **Warning:** Always `lower()` and validate a tag in your app **before** inserting. The `CHECK` is the safety net, not the first line of defense.

> 🏆 **Golden Rule:** **Store each repeated thing once, and link to it by id.** A join table with a composite primary key handles the many-to-many.




## 👤 126. A Few More User Columns

**What it does:** Adds four useful columns to `users`: a **display name**, a **privacy flag**, an account **status**, and a **last login** time.
**Why it is used:** These are small, single-value facts about a user, so they belong in **columns** (topic 118), not in new tables.
**When to use it:** When a profile needs more than a username.

**Analogy:** Adding **extra fields to a membership card**: preferred name, "private account," active or suspended, and last visit.

### Add the columns
```sql
ALTER TABLE users
  ADD COLUMN IF NOT EXISTS display_name VARCHAR(50),
  ADD COLUMN IF NOT EXISTS is_private BOOLEAN NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS status VARCHAR(15) NOT NULL DEFAULT 'active',
  ADD COLUMN IF NOT EXISTS last_login_at TIMESTAMPTZ;

ALTER TABLE users
  ADD CONSTRAINT users_status_allowed
  CHECK (status IN ('active', 'suspended', 'deleted'));
```
**Output:**
```
ALTER TABLE
ALTER TABLE
```
(Running the second statement twice fails with `constraint "users_status_allowed" already exists`.)

| Column | Type | Why this shape |
|---|---|---|
| `display_name` | `VARCHAR(50)`, optional | Not everyone sets one |
| `is_private` | `BOOLEAN NOT NULL DEFAULT false` | A real flag, never `NULL` (topic 92) |
| `status` | `VARCHAR(15)` with a `CHECK` | A fixed list of values (topic 102) |
| `last_login_at` | `TIMESTAMPTZ`, optional | A moment in time. `NULL` means never logged in |

Adding a `NOT NULL` column works on existing rows because of the `DEFAULT` (topic 98).

### Check the result
```sql
SELECT id, username, is_private, status FROM users ORDER BY id;
```
**Output:**
```
 id | username  | is_private | status
----+-----------+------------+--------
  1 | monahan93 | f          | active
  2 | pferrer   | f          | active
  3 | si93onis  | f          | active
  4 | 99stroman | f          | active
(4 rows)
```

### Use them
**Record a login:**
```sql
UPDATE users SET last_login_at = NOW() WHERE id = 1;
```
**Output:**
```
UPDATE 1
```
**An invalid status is rejected:**
```sql
UPDATE users SET status = 'banned' WHERE id = 4;
```
**Output:**
```
ERROR:  new row for relation "users" violates check constraint "users_status_allowed"
```
**Public accounts that are active:**
```sql
SELECT username FROM users WHERE NOT is_private AND status = 'active' ORDER BY id;
```
**Output:**
```
 username
-----------
 monahan93
 pferrer
 si93onis
 99stroman
(4 rows)
```

### Deliberately left out
| Column | Why |
|---|---|
| `password` | With Auth.js and OAuth (topics 40 to 47), the provider handles sign-in. If you ever store passwords, store a **hash** and **never** plain text |
| `follower_count`, `post_count` | Derived data. See topic 127 |

> ⚠️ **Warning:** Keep `status` values in one place. If the list grows or changes often, a lookup table with a foreign key is easier to maintain than a `CHECK` (topic 113).

> ⚠️ **Warning:** `last_login_at` is not set automatically. Your sign-in code must update it.

> 🏆 **Golden Rule:** **A single fact about a user is a column.** Make it `NOT NULL DEFAULT ...` when "unknown" has no meaning, and nullable when it does.