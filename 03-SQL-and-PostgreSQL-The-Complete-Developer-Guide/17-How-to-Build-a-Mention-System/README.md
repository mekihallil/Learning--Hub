## ✨ 118. Additional Features Around Posts

**What it does:** Lists the extra features a post needs (caption, location, tagged people, mentions, hashtags) and decides which become **columns** and which become **tables**.
**Why it is used:** The right shape depends on one question: is there **one** of these per post, or **many**?
**When to use it:** Before adding any new feature to an existing table.

**Analogy:** A photo album page. The caption and location are **printed on the page** (columns). The people you tag are **sticky notes stuck on it** (their own rows, because there can be many).

### One per post vs many per post
| Feature | How many per post? | Becomes |
|---|---|---|
| Caption | One | A column on `posts` |
| Location (lat/lng) | One | Two columns on `posts` |
| People tagged in the photo | Many | Rows in another table |
| People mentioned in the caption (`@name`) | Many | Rows in another table |
| Hashtags (`#sunrise`) | Many | A join table (topic 23) |

**The rule:** one value per post goes in a column. Many values per post need their own table, with a foreign key back to `posts` (topic 22).

### See what `posts` already has
```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'posts'
ORDER BY ordinal_position;
```
**Output:**
```
 column_name |        data_type
-------------+--------------------------
 id          | integer
 created_at  | timestamp with time zone
 updated_at  | timestamp with time zone
 url         | character varying
 caption     | character varying
 lat         | double precision
 lng         | double precision
 user_id     | integer
(8 rows)
```
Caption and location are already there from topic 109.

### The plan
| Topic | Feature |
|---|---|
| 119 | Caption and location columns |
| 120 | Photo tags vs caption mentions |
| 121 | Comparing the tag designs |
| 122 | The final `tags` table |

> ⚠️ **Warning:** Don't store a list in one column (like `'pferrer,si93onis'`). You can't search it, count it, or protect it with a foreign key.

> 🏆 **Golden Rule:** **One per post is a column. Many per post is a table.**





## 📍 119. Adding Captions and Locations

**What it does:** Adds a **caption** and a **location** (latitude and longitude) to `posts`, with a `CHECK` that keeps the coordinates on the globe.
**Why it is used:** Almost every photo app shows a caption and a place. Both are optional, so they allow `NULL`.
**When to use it:** Whenever a post can carry a description or a map pin.

**Analogy:** The **label and map pin** on a photo print. Some prints have both, some have neither.

### Add the columns
These already exist from topic 109, so this is safe to run again:
```sql
ALTER TABLE posts ADD COLUMN IF NOT EXISTS caption VARCHAR(240);
ALTER TABLE posts ADD COLUMN IF NOT EXISTS lat DOUBLE PRECISION;
ALTER TABLE posts ADD COLUMN IF NOT EXISTS lng DOUBLE PRECISION;
```
**Output:**
```
NOTICE:  column "caption" of relation "posts" already exists, skipping
ALTER TABLE
NOTICE:  column "lat" of relation "posts" already exists, skipping
ALTER TABLE
NOTICE:  column "lng" of relation "posts" already exists, skipping
ALTER TABLE
```
On a table without them, the columns would be added, and existing rows would get `NULL`.

### Give two posts a location
```sql
UPDATE posts SET lat = 35.68, lng = 139.69 WHERE id = 1;
UPDATE posts SET lat = 48.86, lng = 2.35 WHERE id = 2;

SELECT id, caption, lat, lng FROM posts ORDER BY id;
```
**Output:**
```
UPDATE 1
UPDATE 1
 id | caption |  lat  |  lng
----+---------+-------+--------
  1 | Sunrise | 35.68 | 139.69
  2 | Lunch   | 48.86 |   2.35
  3 | Sunset  |       |
(3 rows)
```

### The check rejects impossible coordinates
```sql
UPDATE posts SET lat = 95 WHERE id = 1;
```
**Output:**
```
ERROR:  new row for relation "posts" violates check constraint "posts_lat_range"
```
(A `DETAIL:` line follows.) A `NULL` coordinate passes the check, which is what we want for posts with no location (topic 102).

### Query by location
**Posts inside a rough box:**
```sql
SELECT id, caption
FROM posts
WHERE lat BETWEEN 30 AND 40
  AND lng BETWEEN 130 AND 150;
```
**Output:**
```
 id | caption
----+---------
  1 | Sunrise
(1 row)
```
**How many posts have a location?**
```sql
SELECT
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE lat IS NOT NULL) AS with_location
FROM posts;
```
**Output:**
```
 total | with_location
-------+---------------
     3 |             2
(1 row)
```

> ⚠️ **Warning:** Store **both** coordinates or **neither**. If a post could have `lat` without `lng`, add `CHECK ((lat IS NULL) = (lng IS NULL))`.

> ⚠️ **Warning:** A box search is only an approximation. For real "within 5 km" searches, use PostgreSQL's **PostGIS** extension.

> 🏆 **Golden Rule:** **Optional facts are nullable columns, and a `CHECK` keeps them sensible.**




## 🏷️ 120. Photo Mentions vs Caption Mentions

**What it does:** Separates two ways a user can be linked to a post. A **photo tag** puts a user **on the picture** at an (x, y) position. A **caption mention** names a user in the **text** with `@username`.
**Why it is used:** They look alike, but they carry different data. Only a photo tag has a position.
**When to use it:** Whenever your app lets people tag friends in a photo or mention them in a caption.

**Analogy:** A photo tag is **pointing at someone in the picture**. A caption mention is **saying their name out loud**.

### Two tables
```sql
CREATE TABLE photo_tags (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  x INTEGER NOT NULL CHECK (x >= 0),
  y INTEGER NOT NULL CHECK (y >= 0),
  UNIQUE (post_id, user_id)
);

CREATE TABLE caption_tags (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  UNIQUE (post_id, user_id)
);
```
**Output:**
```
CREATE TABLE
CREATE TABLE
```

### Add some tags to post 1
```sql
INSERT INTO photo_tags (post_id, user_id, x, y)
VALUES (1, 2, 120, 80), (1, 3, 300, 150);

INSERT INTO caption_tags (post_id, user_id)
VALUES (1, 2), (1, 4);
```
**Output:**
```
INSERT 0 2
INSERT 0 2
```
```sql
SELECT u.username, t.x, t.y
FROM photo_tags AS t
JOIN users AS u ON u.id = t.user_id
WHERE t.post_id = 1
ORDER BY u.username;
```
**Output:**
```
 username | x   | y
----------+-----+-----
 pferrer  | 120 |  80
 si93onis | 300 | 150
(2 rows)
```

### Caption mentions come from the text
Your app (or SQL) finds the `@names` in the caption:
```sql
SELECT regexp_matches('Great day with @pferrer and @si93onis', '@(\w+)', 'g');
```
**Output:**
```
 regexp_matches
----------------
 {pferrer}
 {si93onis}
(2 rows)
```
Each match is then looked up in `users` and saved as a `caption_tags` row.

### The difference
| | Photo tag | Caption mention |
|---|---|---|
| **Where** | On the image | In the caption text |
| **Position** | `x`, `y` required | None |
| **Created by** | The user taps the photo | Parsing `@name` in the text |
| **Table** | `photo_tags` | `caption_tags` |

> ⚠️ **Warning:** Caption mention rows can **drift out of sync** with the text if the caption is edited. Re-parse and update them whenever the caption changes.





## ⚖️ 121. Considerations on Photo Tags vs Caption Tags

**What it does:** Weighs the **two-table** design from topic 120 against a **single table**, so you can see which one causes fewer problems.
**Why it is used:** Two near-identical tables mean doubled code and awkward questions like "which posts is this user in, in any way?"
**When to use it:** Whenever two tables look almost the same.

**Analogy:** Two **guest lists** for the same party, one for people on the stage and one for people in the crowd. To count everyone, you have to merge both, and watch for anyone on both.

### The problem, with real data
```sql
SELECT post_id, user_id FROM photo_tags
UNION ALL
SELECT post_id, user_id FROM caption_tags
ORDER BY post_id, user_id;
```
**Output:**
```
 post_id | user_id
---------+---------
       1 |       2
       1 |       2
       1 |       3
       1 |       4
(4 rows)
```
`pferrer` (user 2) appears **twice**, because they're both in the photo and in the caption. Plain `UNION` would collapse that (topic 62), but you must remember to use it every time.

### Three possible designs
| Design | How | Trade-off |
|---|---|---|
| **A. Two tables** (topic 120) | `photo_tags` and `caption_tags` | Clear, but every query needs a `UNION` |
| **B. One table plus `tag_type`** | A `'photo'` or `'caption'` column | One table, but `x`, `y` must be nullable and need a check |
| **C. One table, position optional** | `x`, `y` are `NULL` for caption mentions | The simplest, and no type column needed |

### Questions to ask
| Question | Answer |
|---|---|
| Is a user ever **both** photo-tagged and caption-mentioned on one post? | Yes, so decide whether that's one tag or two |
| Do you often ask "all posts this user appears in"? | Favors **one table** |
| Do the two kinds need different rules? | Only the position does |
| Will you add more kinds later (such as video tags)? | A single table adapts more easily |

### Verdict
The two tables share the same **link** (post and user), and differ only by the **optional position**. That points to design **C**, which is built in topic 122.

> ⚠️ **Warning:** Design C means a user can appear **once per post**, as a photo tag **or** a mention. If you need both at once, add a type column and include it in the unique rule.

> 🏆 **Golden Rule:** **If two tables differ only by an optional column, merge them.** Make that column nullable, and protect it with a `CHECK`.




## 🔄 122. Update For Tags

**What it does:** **Replaces** the two tag tables from topic 120 with **one `tags` table**. A tag has a position (`x`, `y`) if it's on the photo, and no position if it's a caption mention.
**Why it is used:** One table means one set of queries, no `UNION`, and no double counting.
**When to use it:** This is the final tag design for the `social_app` schema. It updates the plan from topic 108.

**Analogy:** One **guest list** where each guest either has a **seat number** (photo tag) or no seat (caption mention).

### Replace the old tables
```sql
DROP TABLE IF EXISTS photo_tags, caption_tags;

CREATE TABLE tags (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  x INTEGER,
  y INTEGER,
  CONSTRAINT tags_position_both_or_neither CHECK ((x IS NULL) = (y IS NULL)),
  CONSTRAINT tags_position_not_negative CHECK (x >= 0 AND y >= 0),
  UNIQUE (post_id, user_id)
);
```
**Output:**
```
DROP TABLE
CREATE TABLE
```
| Rule | Tool |
|---|---|
| Both coordinates or neither | `CHECK ((x IS NULL) = (y IS NULL))` |
| No negative position | `CHECK (x >= 0 AND y >= 0)` (`NULL` passes) |
| One tag per user per post | `UNIQUE (post_id, user_id)` |
| Cleanup | `ON DELETE CASCADE` |

### Add tags
```sql
INSERT INTO tags (post_id, user_id, x, y)
VALUES
  (1, 2, 120, 80),
  (1, 3, 300, 150),
  (1, 4, NULL, NULL);
```
**Output:**
```
INSERT 0 3
```
User 4 is a caption mention, so it has no position.

### Break the rules
**A half position:**
```sql
INSERT INTO tags (post_id, user_id, x, y) VALUES (1, 1, 50, NULL);
```
**Output:**
```
ERROR:  new row for relation "tags" violates check constraint "tags_position_both_or_neither"
```
**The same user twice on one post:**
```sql
INSERT INTO tags (post_id, user_id) VALUES (1, 2);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "tags_post_id_user_id_key"
DETAIL:  Key (post_id, user_id)=(1, 2) already exists.
```

### Query it
**Everyone tagged in post 1, and how:**
```sql
SELECT
  u.username,
  CASE WHEN t.x IS NULL THEN 'caption' ELSE 'photo' END AS tag_type,
  t.x, t.y
FROM tags AS t
JOIN users AS u ON u.id = t.user_id
WHERE t.post_id = 1
ORDER BY u.username;
```
**Output:**
```
 username  | tag_type |  x  |  y
-----------+----------+-----+-----
 99stroman | caption  |     |
 pferrer   | photo    | 120 |  80
 si93onis  | photo    | 300 | 150
(3 rows)
```
**Count each kind:**
```sql
SELECT
  COUNT(*) FILTER (WHERE x IS NOT NULL) AS photo_tags,
  COUNT(*) FILTER (WHERE x IS NULL) AS caption_tags
FROM tags
WHERE post_id = 1;
```
**Output:**
```
 photo_tags | caption_tags
------------+--------------
          2 |            1
(1 row)
```

> ⚠️ **Warning:** `DROP TABLE` deletes the old tag data. In a real database, copy it into `tags` first (`INSERT INTO tags SELECT ...`), then drop the old tables.

> ⚠️ **Warning:** If a user can be **both** photo-tagged and mentioned on the same post, change the unique rule to `UNIQUE (post_id, user_id, x, y)` with `NULLS NOT DISTINCT` (PostgreSQL 15+), or add a type column.

> 🏆 **Golden Rule:** **Merge tables that differ by one optional column.** Make it nullable, add a `CHECK`, and the design gets simpler without losing any safety.