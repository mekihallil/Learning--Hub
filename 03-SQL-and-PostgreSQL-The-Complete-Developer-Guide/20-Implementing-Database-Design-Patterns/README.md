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




## 🖼️ 130. Posts Creation

**What it does:** Builds `posts`: a required URL, an optional caption and location, a foreign key to the author, and checks that keep coordinates valid.
**Why it is used:** Posts are the heart of the app, and everything else (comments, likes, tags, hashtags) points at them.
**When to use it:** Right after `users`, since a post needs an author to exist.

**Analogy:** A **photo print** with a label (caption) and an optional map pin, always with the **photographer's name** on the back.

### The table
```sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  url VARCHAR(200) NOT NULL,
  caption VARCHAR(240),
  lat DOUBLE PRECISION,
  lng DOUBLE PRECISION,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT posts_lat_range CHECK (lat BETWEEN -90 AND 90),
  CONSTRAINT posts_lng_range CHECK (lng BETWEEN -180 AND 180),
  CONSTRAINT posts_location_both_or_neither CHECK ((lat IS NULL) = (lng IS NULL))
);

CREATE INDEX posts_user_id_idx ON posts (user_id);
```
**Output:**
```
CREATE TABLE
CREATE INDEX
```
Postgres doesn't index foreign key columns automatically, so `posts_user_id_idx` speeds up "all posts by this user" (topic 127).

### Valid rows
```sql
INSERT INTO posts (url, caption, lat, lng, user_id)
VALUES
  ('http://one.jpg', 'Sunrise #sunrise #travel', 35.68, 139.69, 1),
  ('http://two.jpg', 'Lunch #food', NULL, NULL, 2),
  ('http://three.jpg', 'Sunset #sunrise2 #travel', NULL, NULL, 1);

SELECT id, caption, user_id FROM posts ORDER BY id;
```
**Output:**
```
INSERT 0 3
 id |          caption          | user_id
----+---------------------------+---------
  1 | Sunrise #sunrise #travel  |       1
  2 | Lunch #food               |       2
  3 | Sunset #sunrise2 #travel  |       1
(3 rows)
```

### Break the rules
**A latitude off the globe:**
```sql
INSERT INTO posts (url, lat, lng, user_id) VALUES ('http://bad.jpg', 95, 0, 1);
```
**Output:**
```
ERROR:  new row for relation "posts" violates check constraint "posts_lat_range"
```
**Half a location:**
```sql
INSERT INTO posts (url, lat, lng, user_id) VALUES ('http://half.jpg', 10, NULL, 1);
```
**Output:**
```
ERROR:  new row for relation "posts" violates check constraint "posts_location_both_or_neither"
```
**An author who doesn't exist:**
```sql
INSERT INTO posts (url, user_id) VALUES ('http://ghost.jpg', 999);
```
**Output:**
```
ERROR:  insert or update on table "posts" violates foreign key constraint "posts_user_id_fkey"
DETAIL:  Key (user_id)=(999) is not present in table "users".
```

> ⚠️ **Warning:** `NULL` passes a `CHECK`, so the range checks alone don't stop a missing coordinate. The "both or neither" check handles that.

> ⚠️ **Warning:** `ON DELETE CASCADE` means deleting a user deletes their posts. That's intended here, but confirm it matches how your app should behave.

> 🏆 **Golden Rule:** **Required columns are `NOT NULL`, optional ones are nullable with checks, and every foreign key gets an index.**



## 💬 131. Comments Creation

**What it does:** Builds `comments`: text that can't be blank, plus two foreign keys, one to the **author** and one to the **post**.
**Why it is used:** A comment belongs to exactly one user and one post (two one-to-many relationships, topic 22). Both links are required and cascade.
**When to use it:** After `users` and `posts` exist.

**Analogy:** A **sticky note on a photo**: it must have words, it must say who wrote it, and it must be stuck on a real photo.

### The table
```sql
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  contents VARCHAR(240) NOT NULL,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  CONSTRAINT comments_contents_not_blank CHECK (LENGTH(TRIM(contents)) > 0)
);

CREATE INDEX comments_post_id_idx ON comments (post_id);
CREATE INDEX comments_user_id_idx ON comments (user_id);
```
**Output:**
```
CREATE TABLE
CREATE INDEX
CREATE INDEX
```
The `NOT NULL` alone would accept `'   '`. The `TRIM` check closes that gap (topics 98 and 102).

### Valid rows
```sql
INSERT INTO comments (contents, user_id, post_id)
VALUES ('Nice shot!', 2, 1), ('Love this', 3, 1);
```
**Output:**
```
INSERT 0 2
```

### Break the rules
**A blank comment:**
```sql
INSERT INTO comments (contents, user_id, post_id) VALUES ('   ', 2, 1);
```
**Output:**
```
ERROR:  new row for relation "comments" violates check constraint "comments_contents_not_blank"
```
**A post that doesn't exist:**
```sql
INSERT INTO comments (contents, user_id, post_id) VALUES ('Hello', 2, 999);
```
**Output:**
```
ERROR:  insert or update on table "comments" violates foreign key constraint "comments_post_id_fkey"
DETAIL:  Key (post_id)=(999) is not present in table "posts".
```

### Use it: comments with their authors
```sql
SELECT c.id, u.username, c.contents
FROM comments AS c
JOIN users AS u ON u.id = c.user_id
WHERE c.post_id = 1
ORDER BY c.id;
```
**Output:**
```
 id | username |  contents
----+----------+------------
  1 | pferrer  | Nice shot!
  2 | si93onis | Love this
(2 rows)
```

> ⚠️ **Warning:** The 240-character limit is enforced by `VARCHAR(240)`. Make sure your app's validation (Zod) uses the **same** number, so users see a friendly message first.

> ⚠️ **Warning:** This table has no `parent_id`, so comments **can't be replies to other comments**. Nested threads need the self-reference from the Next.js comment topics.

> 🏆 **Golden Rule:** **A child table needs a foreign key to every parent it belongs to, and an index on each.** Block blank text with a `CHECK`, because `NOT NULL` won't.




## ❤️ 132. Likes Creation

**What it does:** Builds `likes` with **two nullable foreign keys** (`post_id`, `comment_id`), a `CHECK` that **exactly one** is filled, and **partial unique indexes** so nobody likes the same thing twice.
**Why it is used:** One table covers likes on posts **and** comments, with real foreign keys and automatic cleanup (design from topics 109 and 115).
**When to use it:** After `posts` and `comments` exist.

**Analogy:** A **ballot with two boxes** (post or comment) where the rule is "tick **exactly one**," and each voter may tick a given item only once.

### The table
```sql
CREATE TABLE likes (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
  comment_id INTEGER REFERENCES comments(id) ON DELETE CASCADE,
  CONSTRAINT likes_exactly_one_target CHECK (num_nonnulls(post_id, comment_id) = 1)
);

CREATE UNIQUE INDEX likes_user_post_idx
  ON likes (user_id, post_id) WHERE post_id IS NOT NULL;
CREATE UNIQUE INDEX likes_user_comment_idx
  ON likes (user_id, comment_id) WHERE comment_id IS NOT NULL;

CREATE INDEX likes_post_id_idx ON likes (post_id);
CREATE INDEX likes_comment_id_idx ON likes (comment_id);
```
**Output:**
```
CREATE TABLE
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
```
| Piece | Why |
|---|---|
| `num_nonnulls(...) = 1` | Exactly one target |
| Two **partial unique** indexes | A plain `UNIQUE (user_id, post_id, comment_id)` fails because `NULL`s never match (topic 101) |
| The last two indexes | Fast "how many likes does this post or comment have" |

### Valid rows
```sql
INSERT INTO likes (user_id, post_id) VALUES (2, 1), (4, 1), (1, 2);
INSERT INTO likes (user_id, comment_id) VALUES (3, 1);
```
**Output:**
```
INSERT 0 3
INSERT 0 1
```

### Break the rules
**Two targets:**
```sql
INSERT INTO likes (user_id, post_id, comment_id) VALUES (1, 1, 1);
```
**Output:**
```
ERROR:  new row for relation "likes" violates check constraint "likes_exactly_one_target"
```
**No target:**
```sql
INSERT INTO likes (user_id) VALUES (1);
```
**Output:**
```
ERROR:  new row for relation "likes" violates check constraint "likes_exactly_one_target"
```
**The same post liked twice:**
```sql
INSERT INTO likes (user_id, post_id) VALUES (2, 1);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "likes_user_post_idx"
DETAIL:  Key (user_id, post_id)=(2, 1) already exists.
```

### Use it: likes per post
```sql
SELECT p.id, COUNT(l.post_id) AS likes
FROM posts AS p
LEFT JOIN likes AS l ON l.post_id = p.id
GROUP BY p.id
ORDER BY p.id;
```
**Output:**
```
 id | likes
----+-------
  1 |     2
  2 |     1
  3 |     0
(3 rows)
```
`COUNT(l.post_id)` gives post 3 a `0` (topic 53).

> ⚠️ **Warning:** `num_nonnulls` is PostgreSQL-specific. In another database, write the check with `CASE` or `IS NOT NULL` arithmetic (topic 115).

> ⚠️ **Warning:** If you add a third likeable thing, you must alter the table, add an index, and update the check. Past three or four targets, separate tables are simpler (topic 116).

> 🏆 **Golden Rule:** **Nullable foreign keys need a `CHECK` for "exactly one" and partial unique indexes for "only once."** Skip either and the table silently allows bad data.