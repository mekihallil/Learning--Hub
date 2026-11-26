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

> 🏆 **Golden Rule:** **Same link (user to post), different data.** The position is what makes a photo tag special.