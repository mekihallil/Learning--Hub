## 🧩 105. Approaching More Complicated Designs

**What it does:** A step-by-step method for designing a database with **many tables and trickier relationships**, by working from the app's **features** down to its tables.
**Why it is used:** Topics 21 to 35 handled three tables. Real apps have a dozen, and the hard parts (likes on two different things, users following users, hashtags) don't fit the simple patterns. A method keeps you from drowning.
**When to use it:** At the start of any project bigger than a toy.

**Analogy:** Planning a city, not a house. You start with what the **citizens need** (homes, roads, shops), then decide the buildings, and only last draw the streets between them.

### The method
| Step | Question | Output |
|---|---|---|
| 1 | What can users **do**? | A feature list |
| 2 | What **things** does each feature need to remember? | Candidate tables |
| 3 | What does each thing **know about itself**? | Columns |
| 4 | How do the things **relate**? | Foreign keys and join tables |
| 5 | Which relationships are **tricky**? | Design decisions |
| 6 | What rules must **never** be broken? | Constraints (topics 95 to 104) |

### Step 1: list the features
For our photo app:
- People sign up and have a profile
- People post photos with a caption and a location
- People comment on posts
- People like **posts** and **comments**
- People follow other people
- Posts can have hashtags

### Step 2: turn the features into tables
| Feature | Needs to remember | Table |
|---|---|---|
| Sign up | Each person | `users` |
| Post photos | Each post | `posts` |
| Comment | Each comment | `comments` |
| Like | Each like | `likes` |
| Follow | Who follows whom | `followers` |
| Hashtags | Each tag, and which posts use it | `hashtags`, `post_hashtags` |

### Step 3 and 4: sketch the relationships
```
users ──< posts ──< comments >── users
  │         │            │
  │         └── likes ───┘      (a like points at a post OR a comment)
  │
  └──< followers >── users      (users relate to users)

posts >──< hashtags             (many-to-many, via post_hashtags)
```
Read `──<` as "one has many" (topic 22).

### Step 5: the tricky parts
These are where designs go wrong, so give them extra thought.

**Trickiness 1: a like can target a post or a comment.** One foreign key can only point at one table (topic 25). Three ways to handle it:

| Option | Design | Verdict |
|---|---|---|
| **A** | One `likes` table with **two nullable foreign keys** (`post_id`, `comment_id`) plus a `CHECK` that **exactly one** is filled | ✅ Real foreign keys, one table |
| **B** | **Two tables**: `post_likes` and `comment_likes` | ✅ Simplest rules, but two tables to query |
| **C** | One table with `liked_type` and `liked_id` (no foreign key) | ❌ The database can't check the id exists |

We'll use **option A** (topics 108 and 109).

**Trickiness 2: users follow users.** Both sides of the relationship are the **same table**. That's a many-to-many (topic 23) from `users` back to `users`, so it needs a join table with **two** foreign keys, both pointing at `users`.

**Trickiness 3: hashtags.** A post has many hashtags, and a hashtag is on many posts, so it's another many-to-many with a join table.

### Step 6: list the rules before writing SQL
| Rule | Tool |
|---|---|
| Usernames are unique | `UNIQUE` |
| A user has an email or a phone | `CHECK` over two columns (topic 103) |
| Latitude is between -90 and 90 | `CHECK` |
| A like targets exactly one thing | `CHECK` |
| Nobody follows themselves | `CHECK` |
| The same follow can't happen twice | Composite primary key (topic 101) |
| Delete a user, and their content goes too | `ON DELETE CASCADE` (topic 32) |

### Good habits for bigger designs
| Habit | Why |
|---|---|
| Design on paper or in a tool **before** SQL | Changing a diagram is free (topics 106 and 107) |
| Give every table a `SERIAL` primary key, except pure join tables | Simple, stable identity |
| Add `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` to every table | You'll always want to know **when** |
| Name foreign keys `<singular_table>_id` | `user_id`, `post_id` (topic 24) |
| Decide the delete behavior for every foreign key | The database will ask you anyway (topic 32) |
| Build parent tables **before** child tables | A foreign key needs its target to exist (topic 27) |

> ⚠️ **Warning:** **Don't design every possible future feature.** Cover what you need now, and note what you're deliberately leaving out. Tables are cheap to add later and expensive to remove.

> ⚠️ **Warning:** A design that looks fine with five rows can be painful with five million. Think about which questions you'll ask most (such as "all comments on this post") and make sure they're easy.

> 🏆 **Golden Rule:** **Features first, tables second, relationships third, rules last.** Spend the time on the tricky relationships, because the simple ones design themselves.