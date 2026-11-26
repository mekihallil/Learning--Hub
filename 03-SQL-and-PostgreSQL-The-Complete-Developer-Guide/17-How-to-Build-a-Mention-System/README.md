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