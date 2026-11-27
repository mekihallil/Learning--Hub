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