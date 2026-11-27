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