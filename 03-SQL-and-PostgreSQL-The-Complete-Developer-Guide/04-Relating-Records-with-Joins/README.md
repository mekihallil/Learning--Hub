## 🧭 36. Queries with Joins and Aggregations

**What it does:** Introduces the next section: two new ways of getting answers out of your tables. **Joins** combine rows from **different tables**. **Aggregations** boil **many rows down to one value** (a count, a total, an average).
**Why it is used:** Topics 20 to 35 split data into linked tables. That keeps it tidy, but real questions need it back together: "who wrote this comment?" or "how many photos does each user have?"
**When to use it:** Any time the answer spans more than one table, or you want a summary number instead of a list of rows.

**Analogy:** A **join** is laying two printed lists side by side and matching up the rows that belong together. An **aggregation** is the "total" line at the bottom of a receipt.

**The two tools:**

| Tool | What it does | Example question |
|---|---|---|
| **Join** | Matches rows from two tables using a shared value | "Show each comment with its author's username" |
| **Aggregation** | Collapses many rows into one result | "How many photos exist in total?" |
| **Both together** | Match first, then summarize | "How many comments does each photo have?" |

**A first taste of an aggregation:**
```sql
SELECT COUNT(*) FROM photos;
```
**Output:**
```
 count
-------
     7
(1 row)
```
`COUNT(*)` counts rows. Joins come first in this section, and the other aggregate tools (`SUM`, `AVG`, `MAX`, grouping) follow later.

**Reset your practice database.** Run this to get the exact data used in the next topics:
```sql
DROP TABLE IF EXISTS comments;
DROP TABLE IF EXISTS photos;
DROP TABLE IF EXISTS users;

CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50)
);

CREATE TABLE photos (
  id SERIAL PRIMARY KEY,
  url VARCHAR(200),
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  contents VARCHAR(240),
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  photo_id INTEGER REFERENCES photos(id) ON DELETE CASCADE
);

INSERT INTO users (username)
VALUES ('monahan93'), ('pferrer'), ('si93onis'), ('99stroman');

INSERT INTO photos (url, user_id)
VALUES
  ('http://one.jpg', 4), ('http://two.jpg', 1), ('http://25.jpg', 1),
  ('http://36.jpg', 1), ('http://754.jpg', 2), ('http://35.jpg', 3),
  ('http://256.jpg', 4);

INSERT INTO comments (contents, user_id, photo_id)
VALUES
  ('Nice shot!', 2, 1),
  ('Love this', 3, 1),
  ('Where was this taken?', 4, 2),
  ('Beautiful', 1, 5);
```
**Output:**
```
DROP TABLE
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
INSERT 0 4
INSERT 0 7
INSERT 0 4
```
(You may also see `NOTICE: table "..." does not exist, skipping` on a first run.)

> ⚠️ **Warning:** This script **deletes** the three tables and their data before rebuilding them. Run it only on your practice database.

> ⚠️ **Warning:** The foreign keys here allow `NULL` (no `NOT NULL`). That is on purpose, because topics 42 and 43 need a photo with no owner.

> 🏆 **Golden Rule:** **Joins connect tables, aggregations summarize rows.** Learn them separately first, then combine them.


## 🔗 37. Joining Data from Different Tables

**What it does:** `JOIN` lines up rows from **two tables** into one result, matching them with a condition you write after `ON`.
**Why it is used:** A comment row only stores a `user_id` (a number). To see the actual **username**, you must reach into the `users` table.
**When to use it:** Whenever the data you want to show lives in more than one table.

**Analogy:** A comment has a name tag that says only "user #2." The join walks over to the guest list, finds user #2, and reads out "pferrer."

**Question:** show each comment's text next to the username of the person who wrote it.
```sql
SELECT contents, username
FROM comments
JOIN users ON users.id = comments.user_id;
```
**Output:**
```
       contents        | username
-----------------------+-----------
 Nice shot!            | pferrer
 Love this             | si93onis
 Where was this taken? | 99stroman
 Beautiful             | monahan93
(4 rows)
```
Row order isn't guaranteed in a join. Yours may differ.

**Reading it:**

| Part | Meaning |
|---|---|
| `FROM comments` | Start with the comments table |
| `JOIN users` | Bring in the users table |
| `ON users.id = comments.user_id` | Match a comment to the user whose `id` equals the comment's `user_id` |
| `SELECT contents, username` | Show columns from **either** table |

**How the matching works, for one comment:**
```
Comment 1:  contents = 'Nice shot!',  user_id = 2

   Compare 2 with every user id:
     users.id = 1  →  no
     users.id = 2  →  YES  ← keep this pairing
     users.id = 3  →  no
     users.id = 4  →  no

Result row:  Nice shot!  |  pferrer
```
Postgres does this for every row, then returns only the pairings where `ON` was true.

**Same column name in both tables:** `id` exists in `comments` **and** `users`, so Postgres needs to know which one you mean:
```sql
SELECT id, contents, username
FROM comments
JOIN users ON users.id = comments.user_id;
```
**Output:**
```
ERROR:  column reference "id" is ambiguous
LINE 1: SELECT id, contents, username
```
Fix it by naming the table:
```sql
SELECT comments.id, contents, username
FROM comments
JOIN users ON users.id = comments.user_id;
```
**Output:**
```
 id |       contents        | username
----+-----------------------+-----------
  1 | Nice shot!            | pferrer
  2 | Love this             | si93onis
  3 | Where was this taken? | 99stroman
  4 | Beautiful             | monahan93
(4 rows)
```

**Table order and side order don't change the matches:**
```sql
-- All three return the same pairings
FROM comments JOIN users ON users.id = comments.user_id
FROM users JOIN comments ON users.id = comments.user_id
FROM comments JOIN users ON comments.user_id = users.id
```

**A wrong `ON` still runs, it just means something else:**
```sql
SELECT contents, username
FROM comments
JOIN users ON users.id = comments.photo_id;
```
**Output:**
```
       contents        | username
-----------------------+-----------
 Nice shot!            | monahan93
 Love this             | monahan93
 Where was this taken? | pferrer
(3 rows)
```
This compares a **photo** id against a **user** id. Postgres sees two integers and is happy, but the answer is nonsense.

> ⚠️ **Warning:** **Postgres checks that your `ON` is valid, not that it makes sense.** Always join a **foreign key** to the **primary key** it points at (topic 24).

> ⚠️ **Warning:** A plain `JOIN` **drops rows that find no partner**. Comments with no matching user would silently vanish (topics 42 and 43).

> ⚠️ **Warning:** Column names that exist in both tables must be prefixed with the table name, like `comments.id`.

> 🏆 **Golden Rule:** **`JOIN other_table ON primary_key = foreign_key`.** Say which tables, say which two columns must be equal, then pick the columns to show.


## ⚡ 38. Another Quick Join

**What it does:** A second worked example of `JOIN`, this time between `photos` and `users`, so the pattern from topic 37 becomes automatic.
**Why it is used:** One example shows how a join works. A second shows what **stays the same** every time.
**When to use it:** Whenever you want each photo shown with its owner's details.

**Analogy:** The same card-catalog lookup as before, just on a different shelf.

**Show every photo with its owner:**
```sql
SELECT url, username
FROM photos
JOIN users ON users.id = photos.user_id;
```
**Output:**
```
      url       | username
----------------+-----------
 http://one.jpg | 99stroman
 http://two.jpg | monahan93
 http://25.jpg  | monahan93
 http://36.jpg  | monahan93
 http://754.jpg | pferrer
 http://35.jpg  | si93onis
 http://256.jpg | 99stroman
(7 rows)
```

**Notice the repeats.** `monahan93` appears **three times** and `99stroman` **twice**. Each photo gets its own row, and its owner's name is copied onto it. That is how a one-to-many relationship (topic 22) looks after a join.

**Add a filter with `WHERE` (it runs on the joined rows):**
```sql
SELECT url, username
FROM photos
JOIN users ON users.id = photos.user_id
WHERE username = 'monahan93';
```
**Output:**
```
      url       | username
----------------+-----------
 http://two.jpg | monahan93
 http://25.jpg  | monahan93
 http://36.jpg  | monahan93
(3 rows)
```

**Start from the other table:** same pairs, different starting point:
```sql
SELECT username, url
FROM users
JOIN photos ON photos.user_id = users.id;
```
**Output:**
```
 username  |      url
-----------+----------------
 99stroman | http://one.jpg
 monahan93 | http://two.jpg
 monahan93 | http://25.jpg
 monahan93 | http://36.jpg
 pferrer   | http://754.jpg
 si93onis  | http://35.jpg
 99stroman | http://256.jpg
(7 rows)
```

**The three-question recipe for any join:**

| Question | In this example |
|---|---|
| 1. Which **two tables** hold the data? | `photos` and `users` |
| 2. Which **two columns** connect them? | `photos.user_id` and `users.id` |
| 3. Which **columns** do I want to show? | `url` and `username` |

**The pattern:**
```sql
SELECT <columns from either table>
FROM <table A>
JOIN <table B> ON <table B>.<primary key> = <table A>.<foreign key>;
```

> ⚠️ **Warning:** Don't expect one output row per **user**. A join produces one row per **matching pair**, so users with several photos appear several times.

> ⚠️ **Warning:** Row order isn't guaranteed. Don't rely on it unless you add sorting later.

> 🏆 **Golden Rule:** **Every join follows the same recipe: two tables, one matching condition, then your columns.** Only the table and column names change.


## 📝 39. Exercise Overview

**What it does:** A practice exercise on `JOIN`, using the photo app, including one query that joins **three tables**.
**Why it is used:** Joins click when you write them yourself. Trying each task before reading topic 40 is how it sticks.
**When to use it:** After topics 37 and 38. Attempt all three before reading the solutions.

**Your data (from the reset script in topic 36):**
```sql
SELECT * FROM comments;
```
**Output:**
```
 id |       contents        | user_id | photo_id
----+-----------------------+---------+----------
  1 | Nice shot!            |       2 |        1
  2 | Love this             |       3 |        1
  3 | Where was this taken? |       4 |        2
  4 | Beautiful             |       1 |        5
(4 rows)
```

**Task 1.** Print the `contents` of each comment and the `url` of the photo it was posted on.

**Expected output:**
```
       contents        |      url
-----------------------+----------------
 Nice shot!            | http://one.jpg
 Love this             | http://one.jpg
 Where was this taken? | http://two.jpg
 Beautiful             | http://754.jpg
(4 rows)
```

**Task 2.** Print the `contents` of every comment written by the user `si93onis`.

**Expected output:**
```
 contents
-----------
 Love this
(1 row)
```

**Task 3 (stretch).** Print the `username` of each commenter and the `url` of the photo they commented on. This needs **three tables** (`comments`, `users`, `photos`).

**Expected output:**
```
 username  |      url
-----------+----------------
 pferrer   | http://one.jpg
 si93onis  | http://one.jpg
 99stroman | http://two.jpg
 monahan93 | http://754.jpg
(4 rows)
```
(Row order may differ.)

**Hints:**

| Task | Hint |
|---|---|
| 1 | Which column in `comments` points at `photos`? |
| 2 | Join to find the author, then filter with `WHERE` |
| 3 | `comments` has **two** foreign keys. Each one needs its own `JOIN ... ON` |
| All | Use `table.column` for any name that exists in more than one table |

> ⚠️ **Warning:** Don't start by guessing. First answer the three recipe questions from topic 38: which tables, which connecting columns, which output columns.

> 🏆 **Golden Rule:** **One `JOIN` per table you bring in.** Three tables means two joins.


## ✅ 40. A Joinful Solution

**What it does:** The worked answers to the exercise in topic 39, with an explanation of why each one works.
**Why it is used:** Comparing your attempt with a clear solution shows where your thinking differed.
**When to use it:** After you've tried the three tasks.

**Task 1: comment text with the photo URL**
```sql
SELECT contents, url
FROM comments
JOIN photos ON photos.id = comments.photo_id;
```
**Output:**
```
       contents        |      url
-----------------------+----------------
 Nice shot!            | http://one.jpg
 Love this             | http://one.jpg
 Where was this taken? | http://two.jpg
 Beautiful             | http://754.jpg
(4 rows)
```
The link is `comments.photo_id` → `photos.id`.

**Task 2: comments written by `si93onis`**
```sql
SELECT contents
FROM comments
JOIN users ON users.id = comments.user_id
WHERE username = 'si93onis';
```
**Output:**
```
 contents
-----------
 Love this
(1 row)
```
The join brings `username` within reach, so `WHERE` can filter on it.

**Task 3: commenter and photo together (three tables)**
```sql
SELECT username, url
FROM comments
JOIN users ON users.id = comments.user_id
JOIN photos ON photos.id = comments.photo_id;
```
**Output:**
```
 username  |      url
-----------+----------------
 pferrer   | http://one.jpg
 si93onis  | http://one.jpg
 99stroman | http://two.jpg
 monahan93 | http://754.jpg
(4 rows)
```

**Why Task 3 works:** `comments` sits in the middle and holds **two** foreign keys.
```
        ┌── user_id ──►  users
comments
        └── photo_id ─►  photos
```
Each `JOIN` adds one table, and each has its own `ON`.

**Sanity check:** there are 4 comments, so each query should return **at most 4 rows**. If you get more or fewer, re-check your `ON`.

**Common mistakes:**

| Mistake | What happens |
|---|---|
| Forgetting `ON` in Task 3 | `ERROR: syntax error at or near ";"` |
| `ON photos.id = comments.user_id` | Valid SQL, **wrong** pairings (topic 37) |
| Selecting a bare `id` | `ERROR: column reference "id" is ambiguous` |
| `WHERE username = "si93onis"` | Double quotes read as a column name (topic 5) |
| `WHERE` before the joins | Syntax error, because `WHERE` must come **after** all joins |

> ⚠️ **Warning:** A wrong join often returns plausible-looking rows with no error. Compare the **row count** and a few values against what you expect.

> 🏆 **Golden Rule:** **Build joins one at a time.** Get the first `JOIN` working and check it, then add the next.


## 🔤 41. Alternate Forms of Syntax

**What it does:** Shows the **different ways to write the same join**: `INNER JOIN`, table aliases, column aliases, and the old comma style.
**Why it is used:** You'll meet all of these in tutorials and real codebases. Recognizing them means you can read anyone's SQL.
**When to use it:** Aliases are useful all the time. The old comma style you should only **read**, not write.

**Analogy:** Different dialects of the same language. "Soda," "pop," and "soft drink" all get you the same drink.

**Form 1: the version you've been using**
```sql
SELECT url, username
FROM photos
JOIN users ON users.id = photos.user_id;
```

**Form 2: `INNER JOIN` spelled out.** `JOIN` by itself **means** `INNER JOIN`:
```sql
SELECT url, username
FROM photos
INNER JOIN users ON users.id = photos.user_id;
```

**Form 3: table aliases.** Give each table a short nickname:
```sql
SELECT p.url, u.username
FROM photos AS p
JOIN users AS u ON u.id = p.user_id;
```
`AS` is optional (`FROM photos p JOIN users u ON ...` works too).

**Form 4: column aliases.** Rename the output columns:
```sql
SELECT p.url AS photo, u.username AS owner
FROM photos AS p
JOIN users AS u ON u.id = p.user_id;
```
**Output:**
```
     photo      |   owner
----------------+-----------
 http://one.jpg | 99stroman
 http://two.jpg | monahan93
 http://25.jpg  | monahan93
 http://36.jpg  | monahan93
 http://754.jpg | pferrer
 http://35.jpg  | si93onis
 http://256.jpg | 99stroman
(7 rows)
```
All four forms return the same pairs.

**Form 5: the old comma style (read it, don't write it):**
```sql
SELECT url, username
FROM photos, users
WHERE users.id = photos.user_id;
```
Same result, but the matching rule hides in `WHERE`.

**Why that is risky:** forget the `WHERE` and Postgres pairs **every** photo with **every** user:
```sql
SELECT url, username FROM photos, users;
```
**Output:**
```
(28 rows)
```
That's 7 photos x 4 users, a **cross join**. On big tables this produces a huge, useless result.

**Once you alias a table, use the alias everywhere:**
```sql
SELECT photos.url FROM photos AS p JOIN users AS u ON u.id = p.user_id;
```
**Output:**
```
ERROR:  invalid reference to FROM-clause entry for table "photos"
HINT:  Perhaps you meant to reference the table alias "p".
```

**Comparing the forms:**

| Form | Example | Recommended? |
|---|---|---|
| `JOIN ... ON` | `JOIN users ON ...` | ✅ Yes |
| `INNER JOIN ... ON` | `INNER JOIN users ON ...` | ✅ Yes, more explicit |
| Table aliases | `FROM photos AS p` | ✅ Yes, shorter queries |
| Column aliases | `u.username AS owner` | ✅ Yes, clearer output |
| Comma + `WHERE` | `FROM photos, users WHERE ...` | ❌ Avoid |

**A shortcut you can't use here:** `JOIN ... USING (column)` works only when **both** tables name the connecting column identically. Ours differ (`users.id` and `photos.user_id`), so we use `ON`.

> ⚠️ **Warning:** `AS` for **tables** and `AS` for **columns** look alike but do different jobs. One renames a table inside the query, the other renames a column in the output.

> ⚠️ **Warning:** Avoid `NATURAL JOIN`. It joins on **any columns that happen to share a name**, so adding a column later can silently change your results.

> 🏆 **Golden Rule:** **Write `JOIN ... ON` with short aliases.** The matching condition lives next to the join, where it's easy to see and hard to forget.


## 🕳️ 42. Missing Data in Joins

**What it does:** Shows that a plain `JOIN` **silently drops rows that have no match**, and introduces the joins that keep them: `LEFT JOIN`, `RIGHT JOIN`, and `FULL JOIN`.
**Why it is used:** Sometimes you need **all** photos, even those with no owner, or **all** users, even those with no photos. A plain join would hide them.
**When to use it:** Whenever "include everything from this table" matters, like reports, audits, or finding orphans.

**Analogy:** A wedding seating chart that **only lists guests who RSVP'd**. Anyone who didn't reply simply isn't on the chart, and nothing tells you they were skipped.

**Step 1: add two rows that have no match.**
```sql
INSERT INTO users (username) VALUES ('lonelyuser');
INSERT INTO photos (url, user_id) VALUES ('http://lost.jpg', NULL);
```
**Output:**
```
INSERT 0 1
INSERT 0 1
```
`lonelyuser` has **no photos**. `lost.jpg` has **no owner** (`user_id` is `NULL`).

**Step 2: notice the missing row.**
```sql
SELECT COUNT(*) FROM photos;
```
**Output:**
```
 count
-------
     8
(1 row)
```
```sql
SELECT COUNT(*)
FROM photos
JOIN users ON users.id = photos.user_id;
```
**Output:**
```
 count
-------
     7
(1 row)
```
8 photos exist, but the join returned only 7. **`lost.jpg` disappeared** and there was no error.

**Fix 1: `LEFT JOIN` keeps every row from the table written first (`photos`):**
```sql
SELECT url, username
FROM photos
LEFT JOIN users ON users.id = photos.user_id;
```
**Output:**
```
       url       | username
-----------------+-----------
 http://one.jpg  | 99stroman
 http://two.jpg  | monahan93
 http://25.jpg   | monahan93
 http://36.jpg   | monahan93
 http://754.jpg  | pferrer
 http://35.jpg   | si93onis
 http://256.jpg  | 99stroman
 http://lost.jpg |
(8 rows)
```
`lost.jpg` is back, with an empty (`NULL`) username.

**Fix 2: `RIGHT JOIN` keeps every row from the table written second (`users`):**
```sql
SELECT url, username
FROM photos
RIGHT JOIN users ON users.id = photos.user_id;
```
**Output:**
```
      url       |  username
----------------+------------
 http://one.jpg | 99stroman
 http://256.jpg | 99stroman
 http://two.jpg | monahan93
 http://25.jpg  | monahan93
 http://36.jpg  | monahan93
 http://754.jpg | pferrer
 http://35.jpg  | si93onis
                | lonelyuser
(8 rows)
```
`lonelyuser` appears with no URL, and `lost.jpg` (which has no user) is not shown.

**Fix 3: `FULL JOIN` keeps unmatched rows from both sides:**
```sql
SELECT url, username
FROM photos
FULL JOIN users ON users.id = photos.user_id;
```
**Output:**
```
       url       |  username
-----------------+------------
 http://one.jpg  | 99stroman
 http://two.jpg  | monahan93
 http://25.jpg   | monahan93
 http://36.jpg   | monahan93
 http://754.jpg  | pferrer
 http://35.jpg   | si93onis
 http://256.jpg  | 99stroman
 http://lost.jpg |
                 | lonelyuser
(9 rows)
```
Row order may differ on your machine.

**Which rows each join keeps:**

| Join | Matched pairs | Unmatched left rows (`photos`) | Unmatched right rows (`users`) |
|---|---|---|---|
| `JOIN` / `INNER JOIN` | ✅ | ❌ | ❌ |
| `LEFT JOIN` | ✅ | ✅ | ❌ |
| `RIGHT JOIN` | ✅ | ❌ | ✅ |
| `FULL JOIN` | ✅ | ✅ | ✅ |

> ⚠️ **Warning:** **A plain `JOIN` never warns you about lost rows.** If a row count looks too small, compare it with `SELECT COUNT(*)` on each table.

> ⚠️ **Warning:** The "empty" cells are really `NULL`. Test for them with `IS NULL`, never `= NULL` (topic 11).

> ⚠️ **Warning:** If your `photos.user_id` was created with `NOT NULL` (topic 35), the second `INSERT` above fails. Rebuild with the script in topic 36.

> 🏆 **Golden Rule:** **If you need every row from a table, a plain `JOIN` is the wrong tool.** Use an outer join (`LEFT`, `RIGHT`, or `FULL`).


## ❓ 43. Why Wasn't It Included

**What it does:** Explains **why** the plain join dropped `lost.jpg` and `lonelyuser`, and how to use that knowledge to **find** unmatched rows on purpose.
**Why it is used:** Understanding the rule lets you predict which rows a join will keep, instead of guessing.
**When to use it:** Whenever a join returns fewer rows than you expected.

**Analogy:** A dance where partners are paired by matching ticket numbers. Someone holding **no ticket**, and someone holding a ticket **nobody else has**, both stay on the sidelines. They aren't wrong, they just have no partner.

**The two missing rows had two different reasons:**

| Row | Why it found no partner |
|---|---|
| Photo `lost.jpg` | Its `user_id` is `NULL`. **`NULL` is never equal to anything**, so `ON users.id = photos.user_id` is never true for it |
| User `lonelyuser` (id 5) | No photo has `user_id = 5`, so nobody points at them |

**`NULL` is never equal, not even to itself:**
```sql
SELECT NULL = NULL AS same;
```
**Output:**
```
 same
------

(1 row)
```
The result is `NULL` (unknown), **not** `true`. The `ON` test needs `true`, so these rows never match.

**How an inner join decides, step by step:**
```
1. Take a row from the left table  (photos)
2. Look for rows in the right table (users) where ON is TRUE
3. Found one or more?  → output each pairing
4. Found none?         → INNER JOIN: output nothing
                         LEFT JOIN:  output the left row with NULLs for the right columns
```

**Left and right are decided by position in the query:**
```sql
FROM photos LEFT JOIN users ON ...   -- photos is LEFT, users is RIGHT
```
This is equivalent to swapping the tables **and** the keyword:
```sql
FROM users RIGHT JOIN photos ON ...  -- same rows kept
```

**Use it on purpose: find photos with no owner.**
```sql
SELECT url
FROM photos
LEFT JOIN users ON users.id = photos.user_id
WHERE users.id IS NULL;
```
**Output:**
```
       url
-----------------
 http://lost.jpg
(1 row)
```

**Find users who have posted nothing:**
```sql
SELECT username
FROM users
LEFT JOIN photos ON photos.user_id = users.id
WHERE photos.id IS NULL;
```
**Output:**
```
  username
------------
 lonelyuser
(1 row)
```
The `WHERE ... IS NULL` keeps **only** the rows that found no partner.

**The gotcha: a `WHERE` can quietly turn a `LEFT JOIN` back into an inner join.**
```sql
SELECT url, username
FROM photos
LEFT JOIN users ON users.id = photos.user_id
WHERE username <> 'pferrer';
```
**Output:**
```
      url       | username
----------------+-----------
 http://one.jpg | 99stroman
 http://two.jpg | monahan93
 http://25.jpg  | monahan93
 http://36.jpg  | monahan93
 http://35.jpg  | si93onis
 http://256.jpg | 99stroman
(6 rows)
```
`lost.jpg` vanished again. For that row `username` is `NULL`, and `NULL <> 'pferrer'` is unknown, not true. To keep it:
```sql
WHERE username <> 'pferrer' OR username IS NULL
```
**Output:** 7 rows, including `lost.jpg`.

**Troubleshooting checklist:**

| Symptom | Likely cause |
|---|---|
| Fewer rows than the table has | Plain `JOIN` dropped unmatched rows |
| A row you expected is missing | Its foreign key is `NULL`, or no partner exists |
| `LEFT JOIN` still drops rows | A `WHERE` on a right-table column is filtering out the `NULL`s |
| Way **more** rows than expected | Missing or wrong `ON`, or a one-to-many repeat (topic 38) |

> ⚠️ **Warning:** **Never compare with `NULL` using `=` or `<>`.** Use `IS NULL` and `IS NOT NULL` (topic 11).

> ⚠️ **Warning:** After a `LEFT JOIN`, be careful filtering on **right-table columns** in `WHERE`. Add `OR column IS NULL`, or move the condition into the `ON` clause.

> 🏆 **Golden Rule:** **A row is "included" only if it finds a partner, or if its table is the one the join promises to keep.** `NULL` never finds a partner, and `LEFT JOIN ... WHERE right.id IS NULL` is the standard way to find the unmatched rows.


## 🧩 44. Four Kinds of Joins

**What it does:** Names the **four joins** that decide which rows survive when two tables are combined: `INNER`, `LEFT`, `RIGHT`, and `FULL`. They differ only in what happens to rows that **find no partner**.
**Why it is used:** Topics 37 to 43 showed each one separately. This puts them in one picture so you can pick the right one quickly.
**When to use it:** Every time you write a join, ask: "Which unmatched rows do I want to keep?"

**Analogy:** Two guest lists, one for a wedding (photos) and one for a party (users). You want the people who appear on both lists. Do you also want the people who are only on the wedding list? Only on the party list? Everyone? Each join answers a different version of that question.

**The picture (A = `photos`, B = `users`):**
```
INNER JOIN         LEFT JOIN          RIGHT JOIN         FULL JOIN
 A   both   B       A   both   B       A   both   B       A   both   B
(   (####)   )     (###(####)   )     (   (####)###)     (###(####)###)
only A  only B     keeps A + matches  keeps B + matches  keeps everything
dropped dropped    drops only B       drops only A
```

**The rule for each:**

| Join | Keeps | Unmatched `photos` rows | Unmatched `users` rows |
|---|---|---|---|
| `INNER JOIN` (or just `JOIN`) | Matched pairs only | ❌ Dropped | ❌ Dropped |
| `LEFT JOIN` | Matches + all of the **left** table | ✅ Kept (`NULL` on the right) | ❌ Dropped |
| `RIGHT JOIN` | Matches + all of the **right** table | ❌ Dropped | ✅ Kept (`NULL` on the left) |
| `FULL JOIN` | Matches + everything from both | ✅ Kept | ✅ Kept |

**Left and right** come from the **position in your query**:
```sql
FROM photos LEFT JOIN users ON ...
--   ^left            ^right
```

**How many rows each returns with our data:**
```sql
SELECT COUNT(*)
FROM photos
JOIN users ON users.id = photos.user_id;
```
**Output:**
```
 count
-------
     7
(1 row)
```
Change `JOIN` to the other types and you get:

| Join type | Row count | Why |
|---|---|---|
| `INNER JOIN` | **7** | 7 photos have a real owner |
| `LEFT JOIN` | **8** | 7 matches + `lost.jpg` |
| `RIGHT JOIN` | **8** | 7 matches + `lonelyuser` |
| `FULL JOIN` | **9** | 7 matches + `lost.jpg` + `lonelyuser` |

**Full names (these are the same thing):**

| Short | Long |
|---|---|
| `JOIN` | `INNER JOIN` |
| `LEFT JOIN` | `LEFT OUTER JOIN` |
| `RIGHT JOIN` | `RIGHT OUTER JOIN` |
| `FULL JOIN` | `FULL OUTER JOIN` |

`LEFT`, `RIGHT`, and `FULL` are called **outer joins**.

**Choosing a join:**

| You want... | Use |
|---|---|
| Only rows that have a partner | `INNER JOIN` |
| Every row from table A, plus A's partners if any | `LEFT JOIN` |
| Every row from table B, plus B's partners if any | `RIGHT JOIN` |
| Every row from both, matched where possible | `FULL JOIN` |

**There is a fifth join, `CROSS JOIN`**, which pairs **every row with every row** and has no `ON`. It's rarely what you want here:
```sql
SELECT COUNT(*) FROM photos CROSS JOIN users;
```
**Output:**
```
 count
-------
    40
(1 row)
```
That's 8 photos x 5 users.

> ⚠️ **Warning:** `INNER JOIN` is the default, and it **silently drops** unmatched rows. If a count looks too small, check whether you needed an outer join.

> ⚠️ **Warning:** In practice `RIGHT JOIN` is rare. Most people swap the table order and use `LEFT JOIN` instead, because it reads more naturally (topic 46).

> 🏆 **Golden Rule:** **The four joins differ only in which unmatched rows survive.** `INNER` keeps none, `LEFT` keeps the left's, `RIGHT` keeps the right's, and `FULL` keeps both.



## 🛠️ 45. Each Join in Practice

**What it does:** Runs **the same query four times**, changing only the join keyword, so you can see exactly which rows appear and where the `NULL`s show up.
**Why it is used:** The `NULL`s are a diagnostic tool. A `NULL` on one side tells you exactly which side had no partner.
**When to use it:** Whenever you're deciding between join types, or checking why a join returned what it did.

**Analogy:** Four security cameras pointed at the same door. Each one records the same people entering, but they differ in whether they also record people who stand outside without a ticket.

**The query pattern:** we show both ids so the `NULL`s are easy to spot.
```sql
SELECT p.id AS photo_id, p.url, u.id AS user_id, u.username
FROM photos AS p
<JOIN TYPE> users AS u ON u.id = p.user_id;
```

### INNER JOIN
```sql
SELECT p.id AS photo_id, p.url, u.id AS user_id, u.username
FROM photos AS p
INNER JOIN users AS u ON u.id = p.user_id;
```
**Output:**
```
 photo_id |      url       | user_id | username
----------+----------------+---------+-----------
        1 | http://one.jpg |       4 | 99stroman
        2 | http://two.jpg |       1 | monahan93
        3 | http://25.jpg  |       1 | monahan93
        4 | http://36.jpg  |       1 | monahan93
        5 | http://754.jpg |       2 | pferrer
        6 | http://35.jpg  |       3 | si93onis
        7 | http://256.jpg |       4 | 99stroman
(7 rows)
```
No blanks anywhere: every row has both a photo and a user.

### LEFT JOIN
```sql
SELECT p.id AS photo_id, p.url, u.id AS user_id, u.username
FROM photos AS p
LEFT JOIN users AS u ON u.id = p.user_id;
```
**Output:**
```
 photo_id |       url       | user_id | username
----------+-----------------+---------+-----------
        1 | http://one.jpg  |       4 | 99stroman
        2 | http://two.jpg  |       1 | monahan93
        3 | http://25.jpg   |       1 | monahan93
        4 | http://36.jpg   |       1 | monahan93
        5 | http://754.jpg  |       2 | pferrer
        6 | http://35.jpg   |       3 | si93onis
        7 | http://256.jpg  |       4 | 99stroman
        8 | http://lost.jpg |         |
(8 rows)
```
Blanks appear in the **right** table's columns (`user_id`, `username`). Photo 8 has no owner.

### RIGHT JOIN
```sql
SELECT p.id AS photo_id, p.url, u.id AS user_id, u.username
FROM photos AS p
RIGHT JOIN users AS u ON u.id = p.user_id;
```
**Output:**
```
 photo_id |      url       | user_id |  username
----------+----------------+---------+------------
        1 | http://one.jpg |       4 | 99stroman
        2 | http://two.jpg |       1 | monahan93
        3 | http://25.jpg  |       1 | monahan93
        4 | http://36.jpg  |       1 | monahan93
        5 | http://754.jpg |       2 | pferrer
        6 | http://35.jpg  |       3 | si93onis
        7 | http://256.jpg |       4 | 99stroman
          |                |       5 | lonelyuser
(8 rows)
```
Blanks appear in the **left** table's columns (`photo_id`, `url`). User 5 has no photos.

### FULL JOIN
```sql
SELECT p.id AS photo_id, p.url, u.id AS user_id, u.username
FROM photos AS p
FULL JOIN users AS u ON u.id = p.user_id;
```
**Output:**
```
 photo_id |       url       | user_id |  username
----------+-----------------+---------+------------
        1 | http://one.jpg  |       4 | 99stroman
        2 | http://two.jpg  |       1 | monahan93
        3 | http://25.jpg   |       1 | monahan93
        4 | http://36.jpg   |       1 | monahan93
        5 | http://754.jpg  |       2 | pferrer
        6 | http://35.jpg   |       3 | si93onis
        7 | http://256.jpg  |       4 | 99stroman
        8 | http://lost.jpg |         |
          |                 |       5 | lonelyuser
(9 rows)
```
Blanks can appear on **either** side.

**Reading the blanks:**

| What you see | What it means |
|---|---|
| Blank `user_id` / `username` | A photo with **no owner** |
| Blank `photo_id` / `url` | A user with **no photos** |
| No blanks | Every row matched |

**Practice patterns: find only the unmatched rows** by filtering on the `NULL`:

| Goal | Query ends with | Result |
|---|---|---|
| Photos with no owner | `p LEFT JOIN users u ON ... WHERE u.id IS NULL` | `lost.jpg` |
| Users with no photos | `p RIGHT JOIN users u ON ... WHERE p.id IS NULL` | `lonelyuser` |
| Both kinds at once | `p FULL JOIN users u ON ... WHERE p.id IS NULL OR u.id IS NULL` | `lost.jpg` and `lonelyuser` |

**Example of the third one:**
```sql
SELECT p.url, u.username
FROM photos AS p
FULL JOIN users AS u ON u.id = p.user_id
WHERE p.id IS NULL OR u.id IS NULL;
```
**Output:**
```
       url       |  username
-----------------+------------
 http://lost.jpg |
                 | lonelyuser
(2 rows)
```
Always test the `NULL` on the **primary key** (`p.id`, `u.id`), because a primary key is never `NULL` in a real row.

> ⚠️ **Warning:** Test for missing partners with `IS NULL`, never `= NULL` (topic 11).

> ⚠️ **Warning:** Test the `NULL` on a column that **can't be `NULL` normally**, such as the primary key. Testing a column that is sometimes `NULL` in real data gives misleading results.

> 🏆 **Golden Rule:** **Read the blanks.** Blanks on the right mean the left row had no partner, and blanks on the left mean the right row had none.



## 🔀 46. Does Order Matter

**What it does:** Answers a common question: if you swap the order of the tables in a join, does the result change? **It depends on the join type.**
**Why it is used:** Knowing when order matters prevents silent bugs, especially with `LEFT` and `RIGHT` joins.
**When to use it:** Whenever you write or rewrite a join, or copy a query and reorder its tables.

**Analogy:** Shaking hands is the same from either side (order doesn't matter). Saying "I'll keep everyone from **my** guest list" depends entirely on **whose** list it is (order matters).

**Summary:**

| Join | Does swapping the tables change the rows? |
|---|---|
| `INNER JOIN` | ❌ **No**, the same pairs come back |
| `FULL JOIN` | ❌ **No**, the same rows come back |
| `LEFT JOIN` | ✅ **Yes**, a different table is now being kept |
| `RIGHT JOIN` | ✅ **Yes**, for the same reason |

**Inner join: order doesn't matter.**
```sql
SELECT COUNT(*) FROM photos JOIN users ON users.id = photos.user_id;
SELECT COUNT(*) FROM users JOIN photos ON photos.user_id = users.id;
```
**Output (both):**
```
 count
-------
     7
(1 row)
```

**Left join: order **does** matter.**

**Version A: keep every photo**
```sql
SELECT username, url
FROM photos
LEFT JOIN users ON users.id = photos.user_id;
```
**Output:**
```
 username  |       url
-----------+-----------------
 99stroman | http://one.jpg
 monahan93 | http://two.jpg
 monahan93 | http://25.jpg
 monahan93 | http://36.jpg
 pferrer   | http://754.jpg
 si93onis  | http://35.jpg
 99stroman | http://256.jpg
           | http://lost.jpg
(8 rows)
```

**Version B: keep every user (the tables swapped)**
```sql
SELECT username, url
FROM users
LEFT JOIN photos ON photos.user_id = users.id;
```
**Output:**
```
  username  |      url
------------+----------------
 monahan93  | http://two.jpg
 monahan93  | http://25.jpg
 monahan93  | http://36.jpg
 pferrer    | http://754.jpg
 si93onis   | http://35.jpg
 99stroman  | http://one.jpg
 99stroman  | http://256.jpg
 lonelyuser |
(8 rows)
```
Both have 8 rows, but **different** ones. Version A has `lost.jpg` and no `lonelyuser`. Version B has `lonelyuser` and no `lost.jpg`.

**The equivalence:** a `LEFT JOIN` with the tables swapped is a `RIGHT JOIN`.
```sql
-- These two return the same rows
FROM photos LEFT JOIN users ON users.id = photos.user_id
FROM users RIGHT JOIN photos ON users.id = photos.user_id
```
Both keep **all photos**.

**What order still affects, even when the rows are the same:**

| Thing | Effect of swapping the tables |
|---|---|
| Column order in `SELECT *` | The first table's columns appear first |
| Which table `LEFT`/`RIGHT` refers to | The meaning flips |
| Row order in the output | May change (it was never guaranteed) |

**What order never affects:** the two sides of the `ON` test.
```sql
ON users.id = photos.user_id
ON photos.user_id = users.id   -- identical
```

**A reading habit that avoids confusion:** put the table you want to **keep everything from** first, and use `LEFT JOIN`:
```sql
FROM main_table
LEFT JOIN extra_table ON ...
```

> ⚠️ **Warning:** Copying a `LEFT JOIN` and swapping the tables **without** changing the keyword quietly changes which rows are kept.

> ⚠️ **Warning:** With several tables, a `LEFT JOIN` followed by an `INNER JOIN` can drop the rows the `LEFT JOIN` kept (topic 48).

> 🏆 **Golden Rule:** **For `INNER` and `FULL` joins, order is cosmetic. For `LEFT` and `RIGHT` joins, order decides which table is kept.** When in doubt, write `LEFT JOIN` and put the "keep everything" table first.



## 🎯 47. Where with Join

**What it does:** Shows how to filter joined rows with `WHERE`, and the difference between filtering in `WHERE` and filtering in the join's `ON`.
**Why it is used:** After a join, each row holds columns from **both** tables, so `WHERE` can test either side. The location of the filter also changes the result of outer joins.
**When to use it:** Any time you need only some of the joined rows.

**Analogy:** First you **pair up** the two lists (the join). Then you **cross off** the pairs you don't want (the `WHERE`). Crossing off happens **after** pairing.

**The order Postgres works in:**

| Step | Clause | What happens |
|---|---|---|
| 1 | `FROM` | Start with the first table |
| 2 | `JOIN ... ON` | Pair up rows from the other table |
| 3 | `WHERE` | Keep only the pairs that pass |
| 4 | `SELECT` | Pick the columns to show |

**Filter on a column from one table:**
```sql
SELECT url, username
FROM photos
JOIN users ON users.id = photos.user_id
WHERE username = 'monahan93';
```
**Output:**
```
      url       | username
----------------+-----------
 http://two.jpg | monahan93
 http://25.jpg  | monahan93
 http://36.jpg  | monahan93
(3 rows)
```

**Filter on columns from both tables:**
```sql
SELECT p.url, u.username
FROM photos AS p
JOIN users AS u ON u.id = p.user_id
WHERE u.username = 'monahan93' AND p.id > 2;
```
**Output:**
```
      url      | username
---------------+-----------
 http://25.jpg | monahan93
 http://36.jpg | monahan93
(2 rows)
```

**Use the tools from earlier topics (`IN`, `OR`):**
```sql
SELECT p.url, u.username
FROM photos AS p
JOIN users AS u ON u.id = p.user_id
WHERE u.username IN ('pferrer', 'si93onis');
```
**Output:**
```
      url       | username
----------------+----------
 http://754.jpg | pferrer
 http://35.jpg  | si93onis
(2 rows)
```

**The important difference for outer joins: `ON` vs `WHERE`.**

**Condition in `ON`:** it limits which rows **may match**, and the left table's rows are still kept.
```sql
SELECT u.username, p.url
FROM users AS u
LEFT JOIN photos AS p ON p.user_id = u.id AND p.id > 3;
```
**Output:**
```
  username  |      url
------------+----------------
 monahan93  | http://36.jpg
 pferrer    | http://754.jpg
 si93onis   | http://35.jpg
 99stroman  | http://256.jpg
 lonelyuser |
(5 rows)
```
Every user is still listed, including `lonelyuser`.

**Condition in `WHERE`:** it removes rows **after** the join, including the `NULL` ones.
```sql
SELECT u.username, p.url
FROM users AS u
LEFT JOIN photos AS p ON p.user_id = u.id
WHERE p.id > 3;
```
**Output:**
```
 username  |      url
-----------+----------------
 monahan93 | http://36.jpg
 pferrer   | http://754.jpg
 si93onis  | http://35.jpg
 99stroman | http://256.jpg
(4 rows)
```
`lonelyuser` vanished, because `NULL > 3` is unknown, not true.

| | Filter in `ON` | Filter in `WHERE` |
|---|---|---|
| **Runs** | While matching | After matching |
| **Unmatched left rows** | ✅ Kept | ❌ Removed if the test fails on `NULL` |
| **Use for an inner join** | Same result as `WHERE` | Same result as `ON` |
| **Use for a left join** | "Which partners are allowed" | "Which final rows survive" |

> ⚠️ **Warning:** **For `INNER JOIN`, `ON` and `WHERE` give identical results.** The difference only shows up with outer joins.

> ⚠️ **Warning:** A `WHERE` on a **right-table column** after a `LEFT JOIN` can quietly turn it into an inner join (topic 43).

> ⚠️ **Warning:** Aliases from `SELECT` (like `AS owner`) can't be used in `WHERE`, because `WHERE` runs first (topic 17).

> 🏆 **Golden Rule:** **`WHERE` filters the final pairs. `ON` controls which pairs form.** For outer joins, decide which one you mean.