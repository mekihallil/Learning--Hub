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




## 🔢 127. Why No Number of Followers or Posts

**What it does:** Explains why `users` has **no** `follower_count` or `post_count` column. Those numbers can be **calculated** from the real rows whenever you need them.
**Why it is used:** A stored count is a **second copy** of the truth. Two copies eventually disagree.
**When to use it:** Any time you're tempted to store a number that is just a count of other rows.

**Analogy:** A **sign on a shop door** saying "42 customers today." The moment someone walks out and nobody changes the sign, it's wrong. Counting heads when asked is always right.

### First, the followers table
It's the last table of the plan in topic 108:
```sql
CREATE TABLE followers (
  leader_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  follower_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (leader_id, follower_id),
  CONSTRAINT followers_not_self CHECK (leader_id <> follower_id)
);

CREATE INDEX followers_follower_id_idx ON followers (follower_id);

INSERT INTO followers (leader_id, follower_id)
VALUES (1, 2), (1, 3), (1, 4), (2, 1);
```
**Output:**
```
CREATE TABLE
CREATE INDEX
INSERT 0 4
```
A row means "`follower_id` follows `leader_id`."

### Calculate the numbers when needed
```sql
SELECT
  u.username,
  (SELECT COUNT(*) FROM posts AS p WHERE p.user_id = u.id) AS posts,
  (SELECT COUNT(*) FROM followers AS f WHERE f.leader_id = u.id) AS followers,
  (SELECT COUNT(*) FROM followers AS f WHERE f.follower_id = u.id) AS following
FROM users AS u
ORDER BY u.id;
```
**Output:**
```
 username  | posts | followers | following
-----------+-------+-----------+-----------
 monahan93 |     2 |         3 |         1
 pferrer   |     1 |         1 |         1
 si93onis  |     0 |         0 |         1
 99stroman |     0 |         0 |         1
(4 rows)
```
These are correct **by construction**, because they come straight from the rows.

### What goes wrong with a stored counter
```sql
BEGIN;

ALTER TABLE users ADD COLUMN num_posts INTEGER NOT NULL DEFAULT 0;
UPDATE users SET num_posts = (SELECT COUNT(*) FROM posts WHERE posts.user_id = users.id);

INSERT INTO posts (url, user_id) VALUES ('http://four.jpg', 1);

SELECT
  username,
  num_posts AS stored,
  (SELECT COUNT(*) FROM posts WHERE posts.user_id = users.id) AS real
FROM users
WHERE id = 1;

ROLLBACK;
```
**Output:**
```
BEGIN
ALTER TABLE
UPDATE 4
INSERT 0 1
 username  | stored | real
-----------+--------+------
 monahan93 |      2 |    3
(1 row)
```
The new post was inserted, but nobody updated `num_posts`, so the stored number is **stale**. `ROLLBACK` undid the column and the test post.

### Why a stored count drifts
| Event | Counter update needed |
|---|---|
| A post is created | +1 |
| A post is deleted | -1 |
| A user is deleted (cascade) | Update every related counter |
| Two requests at once | Both read, both add 1, one update is lost |
| A bug or a manual fix in `psql` | Nobody updates it |

### Counting is cheap with the right index
| Count | Index that makes it fast |
|---|---|
| Posts per user | `posts (user_id)` |
| Followers of a user | The primary key `(leader_id, follower_id)` |
| Accounts a user follows | `followers (follower_id)` (created above) |

```sql
CREATE INDEX posts_user_id_idx ON posts (user_id);
```
**Output:**
```
CREATE INDEX
```

### In your Next.js app
Prisma can count related rows in the same query, with no stored column. Check the docs for your version:
```tsx
const users = await db.user.findMany({
  select: {
    username: true,
    _count: { select: { posts: true } },
  },
});
```

### When a stored count is acceptable
| Situation | Approach |
|---|---|
| Millions of followers, and the count is shown constantly | A **cached** counter, kept correct by a trigger or one transaction with the insert |
| A dashboard total | A materialized view, refreshed on a schedule |

Even then, the **real rows stay the source of truth**, so the counter can always be rebuilt (topic 111).

> ⚠️ **Warning:** Start with `COUNT(*)` and an index. Add a cache only after you've **measured** a real problem.

> ⚠️ **Warning:** The `CHECK` blocks following yourself, but not someone you already follow twice, because the composite primary key does that.

> 🏆 **Golden Rule:** **Don't store what you can calculate.** The rows are the truth, and a count is just a question you can ask them.




## 👥 128. Designing a Follower System

**What it does:** Designs "user follows user" with **one join table** whose two foreign keys both point at `users`. It's a many-to-many relationship (topic 23) from a table **back to itself**.
**Why it is used:** Following is **directional**. You can follow someone who doesn't follow you, so each row must say who follows whom.
**When to use it:** Any social feature where one kind of thing relates to the same kind of thing: followers, friends, blocked users, referrals.

**Analogy:** A **mailing list** where each line reads "Sam subscribes to Alex." Subscribing doesn't mean Alex subscribes to Sam.

### Requirements
| # | Requirement | Tool |
|---|---|---|
| F1 | A user follows **many** users, and has **many** followers | Many-to-many join table |
| F2 | Following is **one-way** | Two differently named columns |
| F3 | You can follow someone **only once** | Composite primary key |
| F4 | You **can't follow yourself** | `CHECK` |
| F5 | Both users must be **real** | Foreign keys |
| F6 | Deleting a user removes their follow links | `ON DELETE CASCADE` |
| F7 | We can list followers, list "following," and count both | Joins and `COUNT` |

### The table
```sql
CREATE TABLE IF NOT EXISTS followers (
  leader_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  follower_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (leader_id, follower_id),
  CONSTRAINT followers_not_self CHECK (leader_id <> follower_id)
);

CREATE INDEX IF NOT EXISTS followers_follower_id_idx ON followers (follower_id);

INSERT INTO followers (leader_id, follower_id)
VALUES (1, 2), (1, 3), (1, 4), (2, 1)
ON CONFLICT DO NOTHING;
```
**Output:**
```
CREATE TABLE
CREATE INDEX
INSERT 0 4
```
(On a repeat run you'll see `NOTICE ... already exists, skipping` and `INSERT 0 0`.)

**Read each row as:** "`follower_id` follows `leader_id`."

| leader_id | follower_id | Meaning |
|---|---|---|
| 1 | 2 | pferrer follows monahan93 |
| 1 | 3 | si93onis follows monahan93 |
| 1 | 4 | 99stroman follows monahan93 |
| 2 | 1 | monahan93 follows pferrer |

### The questions
**Who follows `monahan93`?** Match on `leader_id`, show the `follower_id` user:
```sql
SELECT u.username
FROM followers AS f
JOIN users AS u ON u.id = f.follower_id
WHERE f.leader_id = 1
ORDER BY u.username;
```
**Output:**
```
 username
-----------
 99stroman
 pferrer
 si93onis
(3 rows)
```
**Who does `pferrer` follow?** Match on `follower_id`, show the `leader_id` user:
```sql
SELECT u.username
FROM followers AS f
JOIN users AS u ON u.id = f.leader_id
WHERE f.follower_id = 2;
```
**Output:**
```
 username
-----------
 monahan93
(1 row)
```
**Does `si93onis` follow `monahan93`?**
```sql
SELECT EXISTS (
  SELECT 1 FROM followers WHERE leader_id = 1 AND follower_id = 3
) AS follows;
```
**Output:**
```
 follows
---------
 t
(1 row)
```
**Who follows each other (mutual follows)?** Join the table to itself with the columns swapped:
```sql
SELECT ua.username AS user_a, ub.username AS user_b
FROM followers AS a
JOIN followers AS b ON b.leader_id = a.follower_id AND b.follower_id = a.leader_id
JOIN users AS ua ON ua.id = a.leader_id
JOIN users AS ub ON ub.id = a.follower_id
WHERE a.leader_id < a.follower_id;
```
**Output:**
```
  user_a   | user_b
-----------+---------
 monahan93 | pferrer
(1 row)
```
`a.leader_id < a.follower_id` keeps each pair **once** instead of twice.

**Followers per user, including users with none:**
```sql
SELECT u.username, COUNT(f.follower_id) AS followers
FROM users AS u
LEFT JOIN followers AS f ON f.leader_id = u.id
GROUP BY u.id
ORDER BY followers DESC, u.username;
```
**Output:**
```
 username  | followers
-----------+-----------
 monahan93 |         3
 pferrer   |         1
 99stroman |         0
 si93onis  |         0
(4 rows)
```
`COUNT(f.follower_id)` gives the zeros (topic 53). Counts are calculated, never stored (topic 127).

### Follow and unfollow
**Follow safely (clicking twice does no harm):**
```sql
INSERT INTO followers (leader_id, follower_id) VALUES (3, 4) ON CONFLICT DO NOTHING;
```
**Output:**
```
INSERT 0 1
```
**Unfollow:**
```sql
DELETE FROM followers WHERE leader_id = 3 AND follower_id = 4;
```
**Output:**
```
DELETE 1
```

### Break the rules
**Follow yourself:**
```sql
INSERT INTO followers (leader_id, follower_id) VALUES (1, 1);
```
**Output:**
```
ERROR:  new row for relation "followers" violates check constraint "followers_not_self"
```
**Follow twice:**
```sql
INSERT INTO followers (leader_id, follower_id) VALUES (1, 2);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "followers_pkey"
DETAIL:  Key (leader_id, follower_id)=(1, 2) already exists.
```
**Follow a user who doesn't exist:**
```sql
INSERT INTO followers (leader_id, follower_id) VALUES (999, 1);
```
**Output:**
```
ERROR:  insert or update on table "followers" violates foreign key constraint "followers_leader_id_fkey"
DETAIL:  Key (leader_id)=(999) is not present in table "users".
```

### Cleanup on delete
```sql
BEGIN;

DELETE FROM users WHERE id = 4;
SELECT COUNT(*) AS remaining_follows FROM followers;

ROLLBACK;
```
**Output:**
```
BEGIN
DELETE 1
 remaining_follows
-------------------
                 3
(1 row)
```
Deleting user 4 removed the follow row `(1, 4)`. `ROLLBACK` brought the user back.

### Why the two indexes?
| Search | Index that helps |
|---|---|
| "Followers of user X" (`leader_id = X`) | The **primary key** `(leader_id, follower_id)` |
| "Who does user X follow" (`follower_id = X`) | `followers_follower_id_idx` |

### Optional: private accounts
`users.is_private` (topic 126) suggests **follow requests**. One way is a `status` column on `followers` (`'pending'` or `'accepted'`) with a `CHECK`. This isn't applied here. If you add it, count and list only the `'accepted'` rows.

### In Prisma
A table that points at the same model twice needs **named relations**. Check the docs for your version:
```prisma
model Follow {
  leaderId   Int
  followerId Int
  leader     User @relation("Leaders", fields: [leaderId], references: [id], onDelete: Cascade)
  follower   User @relation("Followers", fields: [followerId], references: [id], onDelete: Cascade)

  @@id([leaderId, followerId])
}
```
Prisma's schema can't express the `CHECK (leader_id <> follower_id)`, so add it in a custom SQL migration (topic 104).

> ⚠️ **Warning:** **The two column names are easy to mix up.** Always read a row as "follower follows leader," and test queries on a small, known set of rows.

> ⚠️ **Warning:** The `<>` check stops `(1, 1)`, but it can't stop **blocked** users from following. That needs a separate rule or table.

> ⚠️ **Warning:** `followers_follower_id_idx` costs a little on every insert, but saves a lot on "who do I follow" lookups. Keep it.

> 🏆 **Golden Rule:** **One join table with two foreign keys to the same table, a composite primary key, and a "not yourself" check.** Name the columns by role (`leader_id`, `follower_id`), never `user1` and `user2`.