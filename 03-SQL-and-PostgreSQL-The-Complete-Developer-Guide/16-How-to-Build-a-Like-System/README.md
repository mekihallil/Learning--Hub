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