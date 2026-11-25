## 📝 110. Requirements of a Like System

**What it does:** Writes down **exactly what a like system must do** before designing any tables, so the design can be judged against a checklist.
**Why it is used:** "Likes" sounds trivial, but a good design has to handle duplicates, deletion, counting, and more than one kind of target. Listing requirements first stops you from building something that works in a demo and breaks in production.
**When to use it:** At the start of any feature, before opening the diagram tool (topics 105 to 107).

**Analogy:** A **job description** written before interviewing anyone. Without it, you can't tell which candidate is actually good.

### What the like system must do
| # | Requirement |
|---|---|
| R1 | A user can **like a post** |
| R2 | A user can **like a comment** |
| R3 | A user can like a given thing **only once** |
| R4 | A user can **take a like back** (unlike) |
| R5 | We can tell **who** liked something |
| R6 | We can **count** the likes on something quickly |
| R7 | A like can only point at a **real** user and a **real** post or comment |
| R8 | Deleting a post, comment, or user **removes** the related likes |
| R9 | *(Maybe later)* more than one kind of reaction (love, haha, sad) |

### The questions the database must answer
| Question | Needs |
|---|---|
| How many likes does post 1 have? | Count likes per target |
| Who liked post 1? | A join from likes to users |
| Did user 2 already like post 1? | A fast yes/no lookup |
| What has user 2 liked? | Likes filtered by user |
| Which posts are most liked? | Group, count, sort |

### From requirements to tools
| Requirement | Database tool |
|---|---|
| R3 once only | A unique constraint (topics 100 and 101) |
| R5 who liked | A row per like, with a `user_id` |
| R6 counting | `COUNT(*)` and `GROUP BY` (topics 51 and 52) |
| R7 real targets | Foreign keys (topic 27) |
| R8 cleanup | `ON DELETE CASCADE` (topic 32) |
| R1 and R2 two targets | **The hard part.** Topics 114 to 116 |

### Set up the practice data
This assumes the `social_app` database and the four tables from topic 109. If you haven't built them, go back and run steps 1 to 5 of that topic first.
```sql
\c social_app
```
**Output:**
```
You are now connected to database "social_app" as user "postgres".
```
**Reset the data** (this wipes the four tables and restarts their ids at 1):
```sql
TRUNCATE likes, comments, posts, users RESTART IDENTITY CASCADE;

INSERT INTO users (username, email)
VALUES
  ('monahan93', 'monahan@example.com'),
  ('pferrer', 'pferrer@example.com'),
  ('si93onis', 'si93@example.com'),
  ('99stroman', 'stroman@example.com');

INSERT INTO posts (url, caption, user_id)
VALUES
  ('http://one.jpg', 'Sunrise', 1),
  ('http://two.jpg', 'Lunch', 2),
  ('http://three.jpg', 'Sunset', 1);

INSERT INTO comments (contents, user_id, post_id)
VALUES
  ('Nice shot!', 2, 1),
  ('Love this', 3, 1);
```
**Output:**
```
TRUNCATE TABLE
INSERT 0 4
INSERT 0 3
INSERT 0 2
```

**What you now have:**

| users | posts | comments |
|---|---|---|
| 1 monahan93, 2 pferrer, 3 si93onis, 4 99stroman | 1 Sunrise (user 1), 2 Lunch (user 2), 3 Sunset (user 1) | 1 "Nice shot!" (user 2, post 1), 2 "Love this" (user 3, post 1) |

### The plan for the next topics
| Topic | Design | Verdict |
|---|---|---|
| 111 | A counter column, an array, a loose table | ❌ How **not** to do it |
| 112 | A proper `post_likes` table | ✅ Solves R1, R3 to R8 |
| 113 | Adding a reaction type | ✅ Solves R9 |
| 114 | One table pointing at anything | ⚠️ Polymorphic, loses R7 |
| 115 | One table, two nullable keys | ✅ Solves R2 in one table |
| 116 | Two tables | ✅ Solves R2 most simply |
| 117 | Choosing | Decision guide |

> ⚠️ **Warning:** `TRUNCATE ... CASCADE` empties tables **without asking**. Run the reset script only in this practice database.

> 🏆 **Golden Rule:** **Write the requirements as a numbered list, then grade every design against it.** A design that fails a requirement isn't "simple," it's incomplete.




## 🚫 111. How Not to Design a Like System

**What it does:** Shows the three designs that look obvious and **fail the requirements** from topic 110: a counter column, an array of user ids, and a loose likes table with no rules.
**Why it is used:** Seeing *why* they fail teaches you what the good design must fix. All three have been shipped in real apps.
**When to use it:** As a warning whenever a quick shortcut tempts you.

**Analogy:** Counting votes by **adding tally marks to a blackboard.** You know the total, but not who voted, whether anyone voted twice, or whether the voters even exist.

> These demos use `TEMP` tables, which disappear when your session ends, so nothing needs cleaning up.

### Bad design 1: a counter column
```sql
CREATE TEMP TABLE counter_posts (
  id SERIAL PRIMARY KEY,
  url VARCHAR(200),
  likes INTEGER NOT NULL DEFAULT 0
);

INSERT INTO counter_posts (url) VALUES ('http://one.jpg');

UPDATE counter_posts SET likes = likes + 1 WHERE id = 1;
UPDATE counter_posts SET likes = likes + 1 WHERE id = 1;

SELECT * FROM counter_posts;
```
**Output:**
```
CREATE TABLE
INSERT 0 1
UPDATE 1
UPDATE 1
 id |      url       | likes
----+----------------+-------
  1 | http://one.jpg |     2
(1 row)
```
**What's wrong:**

| Requirement | Result |
|---|---|
| R3 once only | ❌ The same person can add 1 forever |
| R4 unlike | ❌ You can subtract, but you can't tell **whose** like to remove |
| R5 who liked | ❌ Impossible, no one is recorded |
| R7 real users | ❌ Nothing to check |

### Bad design 2: an array of user ids on the post
```sql
CREATE TEMP TABLE array_posts (
  id SERIAL PRIMARY KEY,
  liked_by INTEGER[] NOT NULL DEFAULT '{}'
);

INSERT INTO array_posts DEFAULT VALUES;

UPDATE array_posts SET liked_by = array_append(liked_by, 2);
UPDATE array_posts SET liked_by = array_append(liked_by, 2);
UPDATE array_posts SET liked_by = array_append(liked_by, 999);

SELECT * FROM array_posts;
```
**Output:**
```
CREATE TABLE
INSERT 0 1
UPDATE 1
UPDATE 1
UPDATE 1
 id | liked_by
----+----------
  1 | {2,2,999}
(1 row)
```
**What's wrong:**

| Problem | Why |
|---|---|
| User 2 appears **twice** | An array has no uniqueness rule |
| User **999** doesn't exist | An array element can't be a foreign key |
| Deleting a user leaves the id behind | No `ON DELETE CASCADE` |
| "What has user 2 liked?" | Must scan **every post's array** |
| Each like rewrites the whole array | Slow and contention-prone for a popular post |

### Bad design 3: a loose likes table
This is on the right track (one row per like), but with **no rules**:
```sql
CREATE TEMP TABLE loose_likes (user_id INTEGER, post_id INTEGER);

INSERT INTO loose_likes
VALUES (2, 1), (2, 1), (999, 1), (NULL, NULL);

SELECT COUNT(*) AS likes_for_post_1
FROM loose_likes
WHERE post_id = 1;
```
**Output:**
```
CREATE TABLE
INSERT 0 4
 likes_for_post_1
------------------
                3
(1 row)
```
Post 1 shows **3 likes**, but one is a duplicate, one is from a user who doesn't exist, and there's a row that likes nothing at all.

### Scorecard
| Design | R3 once | R5 who | R7 real | R8 cleanup |
|---|---|---|---|---|
| Counter column | ❌ | ❌ | ❌ | n/a |
| Array of ids | ❌ | ✅ | ❌ | ❌ |
| Loose table | ❌ | ✅ | ❌ | ❌ |

### A counter isn't always wrong
A like **counter** can be a useful **cache** on top of a proper likes table (so a feed doesn't recount millions of rows). The mistake is making it the **only** record. If you add one later, keep it in sync with a trigger or in the same transaction.

> ⚠️ **Warning:** **Every one of these "works" in a demo.** The bugs show up with real users, concurrent clicks, and deleted accounts.

> ⚠️ **Warning:** A column that holds a **list** (comma-separated text, an array) usually means you need a **separate table** (topics 2 and 23).

> 🏆 **Golden Rule:** **One like = one row, with real foreign keys and a uniqueness rule.** Anything else loses information the database could have protected.




## ✅ 112. Designing a Like System

**What it does:** Builds the **correct** like table for posts: one row per like, a foreign key to the user and the post, and a **composite primary key** so nobody can like the same post twice.
**Why it is used:** This one table satisfies almost every requirement from topic 110. The rest of the section is variations on it.
**When to use it:** Whenever something can be "liked" and the target is a **single kind** of thing (here, posts).

**Analogy:** A **guest book** where each line says "who" and "which event," and the book refuses a second line from the same person for the same event.

### The table
```sql
DROP TABLE IF EXISTS post_likes;

CREATE TABLE post_likes (
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, post_id)
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```
(You may see a `NOTICE: table "post_likes" does not exist, skipping` on a first run.)

| Piece | Requirement it satisfies |
|---|---|
| One row per like | R5 who liked, R6 counting |
| `REFERENCES users(id)` and `REFERENCES posts(id)` | R7 real user, real post |
| `ON DELETE CASCADE` | R8 cleanup |
| `PRIMARY KEY (user_id, post_id)` | R3 once only (no separate id needed) |
| `NOT NULL` | No half-empty likes |

### Add some likes
```sql
INSERT INTO post_likes (user_id, post_id)
VALUES (2, 1), (3, 1), (4, 1), (2, 2);
```
**Output:**
```
INSERT 0 4
```

### Break the rules (these should fail)
**The same like twice (R3):**
```sql
INSERT INTO post_likes (user_id, post_id) VALUES (2, 1);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "post_likes_pkey"
DETAIL:  Key (user_id, post_id)=(2, 1) already exists.
```
**A user who doesn't exist (R7):**
```sql
INSERT INTO post_likes (user_id, post_id) VALUES (999, 1);
```
**Output:**
```
ERROR:  insert or update on table "post_likes" violates foreign key constraint "post_likes_user_id_fkey"
DETAIL:  Key (user_id)=(999) is not present in table "users".
```

### Answer the questions from topic 110
**Likes per post, including posts with none:**
```sql
SELECT p.id, p.caption, COUNT(l.post_id) AS likes
FROM posts AS p
LEFT JOIN post_likes AS l ON l.post_id = p.id
GROUP BY p.id
ORDER BY p.id;
```
**Output:**
```
 id | caption | likes
----+---------+-------
  1 | Sunrise |     3
  2 | Lunch   |     1
  3 | Sunset  |     0
(3 rows)
```
`COUNT(l.post_id)` (not `COUNT(*)`) makes post 3 show `0` (topic 53).

**Who liked post 1?**
```sql
SELECT u.username
FROM post_likes AS l
JOIN users AS u ON u.id = l.user_id
WHERE l.post_id = 1
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

**Did user 2 like post 1?**
```sql
SELECT EXISTS (
  SELECT 1 FROM post_likes WHERE user_id = 2 AND post_id = 1
) AS liked;
```
**Output:**
```
 liked
-------
 t
(1 row)
```

### Unlike, and a "safe" like
**Unlike (R4):**
```sql
DELETE FROM post_likes WHERE user_id = 2 AND post_id = 2;
```
**Output:**
```
DELETE 1
```
**Like without an error if it already exists:**
```sql
INSERT INTO post_likes (user_id, post_id) VALUES (2, 2) ON CONFLICT DO NOTHING;
INSERT INTO post_likes (user_id, post_id) VALUES (2, 2) ON CONFLICT DO NOTHING;
```
**Output:**
```
INSERT 0 1
INSERT 0 0
```
The first inserted the like. The second did nothing and raised no error, which makes a "like" button safe to click twice.

**What has user 2 liked?**
```sql
SELECT p.caption
FROM post_likes AS l
JOIN posts AS p ON p.id = l.post_id
WHERE l.user_id = 2
ORDER BY p.id;
```
**Output:**
```
 caption
---------
 Sunrise
 Lunch
(2 rows)
```

### Speed: add an index for the other direction
The primary key `(user_id, post_id)` makes lookups **by user** fast. "Count the likes of a post" searches by `post_id`, which needs its own index (topic 101):
```sql
CREATE INDEX post_likes_post_id_idx ON post_likes (post_id);
```
**Output:**
```
CREATE INDEX
```

### Primary key choices
| Choice | When |
|---|---|
| `PRIMARY KEY (user_id, post_id)` (used here) | Nothing else needs to reference a single like |
| `id SERIAL PRIMARY KEY` plus `UNIQUE (user_id, post_id)` | Other tables must point at an individual like |

> ⚠️ **Warning:** Without the second index, counting likes on one post in a huge table scans too much. Add `post_id` indexes before it's slow.

> ⚠️ **Warning:** This table likes **posts only**. Comments need another design (topics 114 to 116).

> 🏆 **Golden Rule:** **One row per like, two foreign keys, and a composite primary key.** That single pattern gives you uniqueness, real references, and cleanup for free.




## 😍 113. Making a Reaction System Instead

**What it does:** Extends the like table with a **`reaction_type`** column, so users can respond with `like`, `love`, `haha`, `sad`, or `angry` instead of only a plain like.
**Why it is used:** A like is just one reaction. Adding a column handles the whole family, and "like" becomes one allowed value.
**When to use it:** When your app wants more than a thumbs-up (requirement R9 from topic 110).

**Analogy:** Upgrading a **single light switch** to a **dial with five settings**. You still have one control per person, but each can be set to a different value.

### The table
```sql
DROP TABLE IF EXISTS post_reactions;

CREATE TABLE post_reactions (
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  reaction_type VARCHAR(10) NOT NULL
    CHECK (reaction_type IN ('like', 'love', 'haha', 'sad', 'angry')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, post_id)
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```
It's the same table as topic 112, plus one `CHECK`ed column (topic 102). The primary key stays `(user_id, post_id)`, so each person has **one** reaction per post.

### Add reactions
```sql
INSERT INTO post_reactions (user_id, post_id, reaction_type)
VALUES (2, 1, 'like'), (3, 1, 'love'), (4, 1, 'like'), (2, 2, 'haha');
```
**Output:**
```
INSERT 0 4
```
**An invalid reaction is rejected:**
```sql
INSERT INTO post_reactions (user_id, post_id, reaction_type) VALUES (1, 2, 'meh');
```
**Output:**
```
ERROR:  new row for relation "post_reactions" violates check constraint "post_reactions_reaction_type_check"
```
(A `DETAIL:` line with the failing row follows.)

### Count by reaction type
```sql
SELECT post_id, reaction_type, COUNT(*) AS total
FROM post_reactions
GROUP BY post_id, reaction_type
ORDER BY post_id, reaction_type;
```
**Output:**
```
 post_id | reaction_type | total
---------+---------------+-------
       1 | like          |     2
       1 | love          |     1
       2 | haha          |     1
(3 rows)
```

### One row per post, with the counts side by side
```sql
SELECT
  post_id,
  COUNT(*) FILTER (WHERE reaction_type = 'like') AS likes,
  COUNT(*) FILTER (WHERE reaction_type = 'love') AS loves,
  COUNT(*) AS total
FROM post_reactions
GROUP BY post_id
ORDER BY post_id;
```
**Output:**
```
 post_id | likes | loves | total
---------+-------+-------+-------
       1 |     2 |     1 |     3
       2 |     0 |     0 |     1
(2 rows)
```
`FILTER` is the PostgreSQL shortcut for a conditional count (topic 84).

### Change a reaction (an "upsert")
User 2 changes from `like` to `love` on post 1:
```sql
INSERT INTO post_reactions (user_id, post_id, reaction_type)
VALUES (2, 1, 'love')
ON CONFLICT (user_id, post_id)
DO UPDATE SET reaction_type = EXCLUDED.reaction_type;
```
**Output:**
```
INSERT 0 1
```
```sql
SELECT user_id, reaction_type FROM post_reactions WHERE post_id = 1 ORDER BY user_id;
```
**Output:**
```
 user_id | reaction_type
---------+---------------
       2 | love
       3 | love
       4 | like
(3 rows)
```
`EXCLUDED` means "the row I tried to insert." If the pair already exists, the reaction is **updated** instead of failing.

**Remove a reaction:**
```sql
DELETE FROM post_reactions WHERE user_id = 2 AND post_id = 1;
```
**Output:**
```
DELETE 1
```

### Two ways to model "which reactions can a user give?"
| Primary key | Behavior | Like |
|---|---|---|
| `(user_id, post_id)` (used here) | **One** reaction per user per post. Changing it is an `UPDATE` | Facebook-style |
| `(user_id, post_id, reaction_type)` | A user can give **several different** reactions to one post | Slack-style emoji |

### Three ways to restrict the allowed values
| Approach | Pros | Cons |
|---|---|---|
| `CHECK (... IN (...))` (used here) | Simple, no extra object | Changing the list means altering the constraint |
| A **lookup table** plus a foreign key | Add a reaction with a plain `INSERT` | One more table to join |
| An `ENUM` type | Compact, self-documenting | Adding is easy, **removing** a value is awkward |

> ⚠️ **Warning:** With `PRIMARY KEY (user_id, post_id)`, a second reaction from the same user **fails** unless you use `ON CONFLICT ... DO UPDATE`.

> ⚠️ **Warning:** If you also want comments to take reactions, you hit the same two-target problem as before (topics 114 to 116).

> 🏆 **Golden Rule:** **A reaction system is a like system with one extra, validated column.** Choose the primary key based on whether a user may react more than once.




## 🎭 114. Polymorphic Associations

**What it does:** A **polymorphic association** lets one table point at **several different kinds of rows** by storing two columns: the **id** of the target and its **type** (`'posts'` or `'comments'`).
**Why it is used:** It's the quickest answer to "a like can go on a post **or** a comment." One table, no extra tables, and adding a third target means just a new type string.
**When to use it:** Understand it so you recognize it. Many frameworks (such as Rails and Django) have built-in support, but in plain SQL it **costs you data safety**, as shown below.

**Analogy:** A **luggage tag** that says "item #1, type: bag." Nobody checks that item #1 is really a bag, or that it exists at all.

### The table
```sql
DROP TABLE IF EXISTS poly_likes;

CREATE TABLE poly_likes (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  liked_id INTEGER NOT NULL,
  liked_type VARCHAR(20) NOT NULL
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```
Notice that `liked_id` has **no `REFERENCES`**. It can't have one, because one foreign key can only point at one table (topic 25).

### What it accepts
```sql
INSERT INTO poly_likes (user_id, liked_id, liked_type)
VALUES
  (2, 1, 'posts'),       -- a real post
  (3, 1, 'comments'),    -- a real comment
  (4, 999, 'posts'),     -- a post that doesn't exist
  (2, 1, 'banana');      -- a type that makes no sense
```
**Output:**
```
INSERT 0 4
```
**All four were accepted.** The database has no way to know that post 999 doesn't exist or that `'banana'` isn't a table.

### Reading it back takes a union
There's no single join, because the target lives in different tables:
```sql
SELECT pl.user_id, 'post' AS target, p.url AS detail
FROM poly_likes AS pl
JOIN posts AS p ON pl.liked_type = 'posts' AND p.id = pl.liked_id
UNION ALL
SELECT pl.user_id, 'comment', c.contents
FROM poly_likes AS pl
JOIN comments AS c ON pl.liked_type = 'comments' AND c.id = pl.liked_id
ORDER BY user_id;
```
**Output:**
```
 user_id | target  |   detail
---------+---------+----------------
       2 | post    | http://one.jpg
       3 | comment | Nice shot!
(2 rows)
```
The two junk likes silently vanish from the result. The data is still there, just unreachable.

### Deletes leave orphans behind
```sql
BEGIN;

DELETE FROM posts WHERE id = 1;

SELECT COUNT(*) AS orphaned_post_likes
FROM poly_likes AS pl
WHERE pl.liked_type = 'posts'
  AND NOT EXISTS (SELECT 1 FROM posts AS p WHERE p.id = pl.liked_id);

ROLLBACK;
```
**Output:**
```
BEGIN
DELETE 1
 orphaned_post_likes
---------------------
                   2
(1 row)
ROLLBACK
```
Likes that point at a deleted post stay in the table forever. `ROLLBACK` put the post back.

### Scorecard
| Requirement | Result |
|---|---|
| R1 and R2 both targets, one table | ✅ |
| R5 who liked | ✅ |
| R6 counting | ✅ (filter by type and id) |
| R7 real targets | ❌ **Not enforced** |
| R8 cleanup on delete | ❌ **Not enforced** |
| Adding a third target | ✅ Easy |

### What you can add to patch it up
| Patch | Helps with | Still missing |
|---|---|---|
| `CHECK (liked_type IN ('posts', 'comments'))` | No `'banana'` | The id is still unchecked |
| `UNIQUE (user_id, liked_type, liked_id)` | Duplicate likes | No existence check |
| Triggers or app code to check and clean up | Existence and cleanup | You maintain it yourself |

> ⚠️ **Warning:** **The database can't protect you here.** Every guarantee depends on every program that writes to the table being correct, forever.

> ⚠️ **Warning:** Some ORMs don't model polymorphic relations directly. **Prisma**, for example, has no built-in support at the time of writing, so check its docs before choosing this design for a Next.js app.

> 🏆 **Golden Rule:** **A foreign key is the database's promise that a link is real.** A polymorphic association gives that promise up in exchange for flexibility. Do that only on purpose.




## 🔀 115. Polymorphic Association Alternative Implementation

**What it does:** Keeps **one likes table**, but replaces `liked_id` and `liked_type` with **one nullable foreign key per target** (`post_id`, `comment_id`) and a `CHECK` that **exactly one** of them is filled in.
**Why it is used:** You get the single table of topic 114 **and** real foreign keys, so the database enforces that targets exist and cleans up on delete.
**When to use it:** A small, fixed number of targets (two or three) where you want one table.
**Note:** this is the design already built as `likes` in topic 109. This topic rebuilds it as `two_key_likes` so you can compare it with the others.

**Analogy:** A form with two boxes, "post number" and "comment number," where the rule is **fill in exactly one**. Both boxes empty is invalid, and so are both full.

### The table
```sql
DROP TABLE IF EXISTS two_key_likes;

CREATE TABLE two_key_likes (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
  comment_id INTEGER REFERENCES comments(id) ON DELETE CASCADE,
  CONSTRAINT exactly_one_target CHECK (num_nonnulls(post_id, comment_id) = 1)
);

CREATE UNIQUE INDEX two_key_likes_user_post_idx
  ON two_key_likes (user_id, post_id) WHERE post_id IS NOT NULL;

CREATE UNIQUE INDEX two_key_likes_user_comment_idx
  ON two_key_likes (user_id, comment_id) WHERE comment_id IS NOT NULL;
```
**Output:**
```
DROP TABLE
CREATE TABLE
CREATE INDEX
CREATE INDEX
```
The two **partial unique indexes** are needed because a plain `UNIQUE (user_id, post_id, comment_id)` doesn't work: `NULL`s never compare equal, so duplicates slip through (topics 101 and 109).

### Three ways to write "exactly one"
| Check | Notes |
|---|---|
| `num_nonnulls(post_id, comment_id) = 1` | Shortest. PostgreSQL-specific |
| `(post_id IS NOT NULL)::int + (comment_id IS NOT NULL)::int = 1` | Counts the filled columns. Scales to more columns |
| `(post_id IS NULL) <> (comment_id IS NULL)` | A tidy "exactly one" for **two** columns only |

### Add likes
```sql
INSERT INTO two_key_likes (user_id, post_id) VALUES (2, 1), (3, 1);
INSERT INTO two_key_likes (user_id, comment_id) VALUES (4, 1), (2, 2);
```
**Output:**
```
INSERT 0 2
INSERT 0 2
```

### Break the rules (these should fail)
**Both targets:**
```sql
INSERT INTO two_key_likes (user_id, post_id, comment_id) VALUES (1, 1, 1);
```
**Output:**
```
ERROR:  new row for relation "two_key_likes" violates check constraint "exactly_one_target"
```
**No target:**
```sql
INSERT INTO two_key_likes (user_id) VALUES (1);
```
**Output:**
```
ERROR:  new row for relation "two_key_likes" violates check constraint "exactly_one_target"
```
**A post that doesn't exist:**
```sql
INSERT INTO two_key_likes (user_id, post_id) VALUES (1, 999);
```
**Output:**
```
ERROR:  insert or update on table "two_key_likes" violates foreign key constraint "two_key_likes_post_id_fkey"
DETAIL:  Key (post_id)=(999) is not present in table "posts".
```
**A duplicate like:**
```sql
INSERT INTO two_key_likes (user_id, post_id) VALUES (2, 1);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "two_key_likes_user_post_idx"
DETAIL:  Key (user_id, post_id)=(2, 1) already exists.
```

### Query across both targets
```sql
SELECT 'post' AS kind, post_id AS id, COUNT(*) AS likes
FROM two_key_likes
WHERE post_id IS NOT NULL
GROUP BY post_id
UNION ALL
SELECT 'comment', comment_id, COUNT(*)
FROM two_key_likes
WHERE comment_id IS NOT NULL
GROUP BY comment_id
ORDER BY kind, id;
```
**Output:**
```
  kind   | id | likes
---------+----+-------
 comment |  1 |     1
 comment |  2 |     1
 post    |  1 |     2
(3 rows)
```

### The cleanup works
```sql
BEGIN;

DELETE FROM posts WHERE id = 1;
SELECT COUNT(*) AS remaining_likes FROM two_key_likes;

ROLLBACK;
```
**Output:**
```
BEGIN
DELETE 1
 remaining_likes
-----------------
               0
(1 row)
```
Deleting post 1 also deleted its two comments, and every like on the post **and** on those comments, all through foreign keys. Contrast that with the orphans in topic 114. `ROLLBACK` restores everything.

### Scorecard
| Requirement | Result |
|---|---|
| R1 and R2 both targets | ✅ |
| R3 once only | ✅ (partial unique indexes) |
| R7 real targets | ✅ **Enforced** |
| R8 cleanup | ✅ **Enforced** |
| Adding a third target | ⚠️ New column, new index, and the check changes |

> ⚠️ **Warning:** Every new likeable thing means **altering the table**: another nullable column, another index, and an updated check. Past three or four targets, this gets clumsy.

> ⚠️ **Warning:** Most of the cells in this table are `NULL`, which is why the check and the partial indexes are essential. Don't skip them.

> ⚠️ **Warning:** `num_nonnulls` is PostgreSQL-specific. Use the `::int` version if you need portability.

> 🏆 **Golden Rule:** **One table, one nullable foreign key per target, and a check that exactly one is set.** You keep the real references that polymorphic associations throw away.




## 🧱 116. The Simplest Alternative

**What it does:** Uses **a separate likes table for each kind of target**: `post_likes` for posts and `comment_likes` for comments. No nullable columns and no checks.
**Why it is used:** It's the **least clever** design, and that's its strength. Each table is the simple pattern from topic 112, with full foreign keys and a plain composite primary key.
**When to use it:** A good default whenever you have a handful of targets, and you value simple rules over a single table.

**Analogy:** **Two guest books**, one per event. There's no tricky rule about "which event is this line for," because each book only ever covers one event.

### The tables
```sql
DROP TABLE IF EXISTS comment_likes;
DROP TABLE IF EXISTS post_likes;

CREATE TABLE post_likes (
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, post_id)
);

CREATE TABLE comment_likes (
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  comment_id INTEGER NOT NULL REFERENCES comments(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, comment_id)
);
```
**Output:**
```
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
```

### Add likes
```sql
INSERT INTO post_likes (user_id, post_id) VALUES (2, 1), (3, 1), (2, 2);
INSERT INTO comment_likes (user_id, comment_id) VALUES (2, 2), (4, 1);
```
**Output:**
```
INSERT 0 3
INSERT 0 2
```

### Every rule is a plain, ordinary constraint
| Rule | How it's enforced |
|---|---|
| Real user and real target | `REFERENCES` (no nulls to worry about) |
| Like once | The composite primary key |
| Cleanup | `ON DELETE CASCADE` |
| "Exactly one target" | **Not needed.** The table itself decides the target |

**Likes per comment, with a plain join:**
```sql
SELECT c.id, c.contents, COUNT(cl.comment_id) AS likes
FROM comments AS c
LEFT JOIN comment_likes AS cl ON cl.comment_id = c.id
GROUP BY c.id
ORDER BY c.id;
```
**Output:**
```
 id |  contents  | likes
----+------------+-------
  1 | Nice shot! |     1
  2 | Love this  |     1
(2 rows)
```

### Querying across both is a union
**Everything user 2 liked:**
```sql
SELECT 'post' AS kind, post_id AS id FROM post_likes WHERE user_id = 2
UNION ALL
SELECT 'comment', comment_id FROM comment_likes WHERE user_id = 2
ORDER BY kind, id;
```
**Output:**
```
  kind   | id
---------+----
 comment |  2
 post    |  1
 post    |  2
(3 rows)
```
**Total likes in the whole app:**
```sql
SELECT
  (SELECT COUNT(*) FROM post_likes) + (SELECT COUNT(*) FROM comment_likes) AS total_likes;
```
**Output:**
```
 total_likes
-------------
           5
(1 row)
```

### Scorecard
| Requirement | Result |
|---|---|
| R1 and R2 both targets | ✅ (two tables) |
| R3 once only | ✅ Primary key |
| R7 real targets | ✅ Enforced |
| R8 cleanup | ✅ Enforced |
| Adding a third target | ✅ Add **one new table**, and no existing table changes |
| Cross-target queries | ⚠️ Need `UNION ALL` |

### Mapping to your Next.js app
This design maps neatly to **Prisma**, because each table is a normal model with a composite id:
```prisma
model PostLike {
  userId Int
  postId Int
  user   User @relation(fields: [userId], references: [id], onDelete: Cascade)
  post   Post @relation(fields: [postId], references: [id], onDelete: Cascade)

  @@id([userId, postId])
}
```
Check the Prisma docs for your version, since the schema syntax and the generated client's field names can vary.

> ⚠️ **Warning:** You repeat the same table pattern for each target, so changes (like adding `created_at`) must be made in each. Code generation or a migration helper can reduce that.

> ⚠️ **Warning:** "Show every like in the system" is a `UNION ALL` over several tables, which grows as targets are added.

> 🏆 **Golden Rule:** **When in doubt, use one table per relationship.** Plain foreign keys and plain primary keys beat clever constraints.