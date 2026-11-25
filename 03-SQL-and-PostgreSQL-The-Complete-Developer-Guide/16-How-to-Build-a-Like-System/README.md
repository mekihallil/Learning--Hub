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