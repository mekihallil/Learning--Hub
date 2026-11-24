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





## 🎨 106. Using a SQL Design Tool

**What it does:** A visual design tool lets you **draw your tables and relationships** as an **ERD** (entity-relationship diagram) with a mouse, instead of writing SQL straight away.
**Why it is used:** A picture shows the whole design at once. You can spot a missing link, move a table, or add a column in seconds, with no data to migrate and nothing to break.
**When to use it:** After the planning in topic 105, and **before** `CREATE TABLE`.

**Analogy:** An architect's **blueprint on paper**. Moving a wall costs an eraser. Moving it after the building is up costs a demolition crew.

### Tools you can use
| Tool | Where | Notes |
|---|---|---|
| **drawSQL** (drawsql.app) | Browser | A drag-and-drop designer that can export SQL |
| **dbdiagram.io** | Browser | Draws from **text** (topic 107) |
| **pgAdmin's ERD Tool** | Your own computer | Free, built in, and connects to a **real database** (see below) |

Free-plan limits and features change over time, so check each site. The ideas below work in all of them.

### How to read the diagram
Most tools use **crow's foot** notation:
```
┌────────┐ 1      * ┌────────┐
│ users  │──────────<│ posts  │
└────────┘           └────────┘
   one user has many posts
```
| Symbol | Meaning |
|---|---|
| A single bar `\|` | **One** |
| A crow's foot `<` | **Many** |
| A circle `o` | **Optional** (zero allowed) |

A line connects a **foreign key** column on the "many" side to a **primary key** on the "one" side (topic 22).

### A typical workflow
| Step | What to do |
|---|---|
| 1 | Create a new diagram and choose **PostgreSQL** as the database type |
| 2 | **Add a table** and name it (plural, `snake_case`) |
| 3 | **Add columns** with names and types (`serial`, `varchar`, `timestamptz`) |
| 4 | Mark the **primary key**, and tick **not null** and **unique** where the tool offers it |
| 5 | **Add relationships** by dragging from the foreign key column to the primary key it points at |
| 6 | Set the **delete behavior** (`CASCADE`, `SET NULL`, `RESTRICT`) on each relationship if the tool offers it |
| 7 | Rearrange the tables until the diagram reads cleanly |
| 8 | **Export** as SQL for PostgreSQL, or share a link |

### Draw the first tables
```
users                    posts                     comments
-----                    -----                     --------
id          PK           id          PK            id         PK
username                 url                       contents
email                    caption                   user_id    FK → users
phone                    user_id     FK → users    post_id    FK → posts
created_at               created_at                created_at
```

### What an export looks like
Design tools typically generate SQL like this (**yours will differ** by tool and settings):
```sql
CREATE TABLE "users" (
  "id" SERIAL PRIMARY KEY,
  "username" VARCHAR(30),
  "email" VARCHAR(255)
);

CREATE TABLE "posts" (
  "id" SERIAL PRIMARY KEY,
  "url" VARCHAR(200),
  "user_id" INTEGER
);

ALTER TABLE "posts" ADD FOREIGN KEY ("user_id") REFERENCES "users" ("id");
```
Run it in the pgAdmin Query Tool (topic 86) against an empty database.

### Always review the exported SQL
| Check | Why |
|---|---|
| **Double-quoted names** (`"users"`) | Quoted names are **case-sensitive** (topic 4). Keep everything lowercase, and you'll never need quotes |
| **Missing `NOT NULL`** | Add it wherever a value is required (topic 97) |
| **Missing `ON DELETE`** | The default is to block deletes (topic 32) |
| **Missing `CHECK` constraints** | Most visual tools can't express them (topic 102) |
| **Missing `DEFAULT`s** | Add `DEFAULT NOW()` to timestamps (topic 99) |

A good habit is to treat the export as a **first draft**, then hand-edit the final SQL.

### The free option already on your computer: pgAdmin's ERD Tool
**Draw from an existing database:** in pgAdmin, right-click your database (such as `photo_app`) and choose **ERD For Database**. You get a diagram of what's really there, which is a great way to check your own work.

You can also create new tables inside the ERD tool and generate the SQL. (Menu labels vary between pgAdmin versions, so look for the closest match.)

> ⚠️ **Warning:** **A diagram is not the database.** Drawing a table doesn't create it. You still have to run the SQL, and the diagram can drift out of date as the real schema changes.

> ⚠️ **Warning:** Don't paste an export into a database that already has tables with the same names. You'll get `ERROR: relation "users" already exists`. Use a fresh database (topic 109).

> ⚠️ **Warning:** Free design tools may limit the number of tables or diagrams, and cloud tools store your design online. Avoid putting anything confidential in a table or column name.

> 🏆 **Golden Rule:** **Draw first, export second, then edit.** Let the tool give you the skeleton, and add the constraints it can't draw.





## ⌨️ 107. A Config-based Schema Designer

**What it does:** Instead of dragging boxes, you **describe your schema in text**, and the tool draws the diagram for you. The best-known option is **dbdiagram.io**, which uses a small language called **DBML** (Database Markup Language).
**Why it is used:** Text is faster to write and edit than clicking, it can be **copied, reviewed, and stored in Git** (just like your code), and you can see what changed between versions.
**When to use it:** When you want the design to live in your repository next to the code, or when you're comfortable typing and find the mouse slow.

**Analogy:** A **recipe card** instead of a cooking video. It's short, exact, easy to share and edit, and the "picture" gets made from the words.

### DBML in one table
| You want | Write |
|---|---|
| A table | `Table users { ... }` |
| A column | `username varchar(30)` |
| Column settings | `[not null, unique]` |
| A primary key | `[pk]` (or `[primary key]`) |
| An auto-counting id | `[pk, increment]` |
| A default | ``[default: `now()`]`` (backticks for expressions) |
| A relationship | `Ref: posts.user_id > users.id` |
| A delete rule | `Ref: ... [delete: cascade]` |

### The four relationship symbols
| Symbol | Meaning | Example |
|---|---|---|
| `>` | **Many-to-one** | `posts.user_id > users.id` (many posts, one user) |
| `<` | **One-to-many** | `users.id < posts.user_id` (one user, many posts) |
| `-` | **One-to-one** | `users.id - profiles.user_id` |
| `<>` | **Many-to-many** | `authors.id <> books.id` |

The `>` and `<` arrows say the same thing from opposite sides (topic 22). The "many" side is the one the arrow's wide end faces. For a normal foreign key, write it as `child.fk > parent.id`.

### The first four tables in DBML
```dbml
Project social_app {
  database_type: 'PostgreSQL'
}

Table users {
  id integer [pk, increment]
  created_at timestamptz [not null, default: `now()`]
  username varchar(30) [not null, unique]
  email varchar(255)
  phone varchar(25)
}

Table posts {
  id integer [pk, increment]
  created_at timestamptz [not null, default: `now()`]
  url varchar(200) [not null]
  caption varchar(240)
  user_id integer [not null]
}

Table comments {
  id integer [pk, increment]
  created_at timestamptz [not null, default: `now()`]
  contents varchar(240) [not null]
  user_id integer [not null]
  post_id integer [not null]
}

Ref: posts.user_id > users.id [delete: cascade]
Ref: comments.user_id > users.id [delete: cascade]
Ref: comments.post_id > posts.id [delete: cascade]
```
Paste that into the editor on dbdiagram.io and three connected boxes appear.

### Inline relationships (shorter)
You can also put the link on the column itself:
```dbml
Table posts {
  id integer [pk, increment]
  user_id integer [not null, ref: > users.id]
}
```
Use the standalone `Ref:` lines when you want a delete rule or several refs together.

### Many-to-many: with a join table
```dbml
Table post_hashtags {
  post_id integer [not null]
  hashtag_id integer [not null]

  indexes {
    (post_id, hashtag_id) [pk]
  }
}

Ref: post_hashtags.post_id > posts.id
Ref: post_hashtags.hashtag_id > hashtags.id
```
That's the join-table pattern from topic 23, with a composite primary key (topic 101).

### Get SQL out of it
**In the browser:** use the **Export** menu on dbdiagram.io and choose **PostgreSQL**.

**From the command line,** with the official DBML CLI (needs Node.js):
```bash
npm install -g @dbml/cli
dbml2sql schema.dbml --postgres > schema.sql
```
**Output (the start of `schema.sql`, typical shape):**
```sql
CREATE TABLE "users" (
  "id" SERIAL PRIMARY KEY,
  "created_at" timestamptz NOT NULL DEFAULT (now()),
  "username" varchar(30) UNIQUE NOT NULL,
  "email" varchar(255),
  "phone" varchar(25)
);
...
ALTER TABLE "posts" ADD FOREIGN KEY ("user_id") REFERENCES "users" ("id") ON DELETE CASCADE;
```
You can also go the other way and convert existing SQL into DBML, which is handy for documenting a database that already exists.

### Drag-and-drop vs config-based
| | Visual tool (topic 106) | Config-based (this topic) |
|---|---|---|
| **Create tables** | Click and type in dialogs | Type text |
| **Speed for big schemas** | Slower | ✅ Faster |
| **Store in Git** | Awkward (export images) | ✅ A plain text file |
| **Review changes** | Hard to compare two pictures | ✅ A normal `git diff` |
| **Learning curve** | Almost none | A few minutes of syntax |
| **Best for** | Exploring and presenting | Keeping a design as the source of truth |

### Keep it in your repository
```bash
git add schema.dbml
git commit -m "docs(db): add DBML schema for social app"
```
Your Git skills from the start of these notes apply directly.

> ⚠️ **Warning:** **DBML isn't full PostgreSQL.** Some PostgreSQL features (such as partial unique indexes, or checks that use functions like `num_nonnulls`) may not export, so plan to **hand-edit** the generated SQL, as in topic 109.

> ⚠️ **Warning:** A `.dbml` file is a **drawing, not the database**. Changing it doesn't change PostgreSQL. You still run SQL, and you must keep the two in sync yourself.

> ⚠️ **Warning:** Always check the **column types** in the export. DBML accepts any type name you type, so `varchar(30)` is passed through, and a typo like `varchr` won't be caught until you run the SQL.

> ⚠️ **Warning:** The tools evolve (syntax additions, plan limits). Check the DBML docs at dbml.dbdiagram.io/docs if a setting isn't recognized.

> 🏆 **Golden Rule:** **Describe the schema in text, store it in Git, and generate the SQL from it.** Treat the file as the readable blueprint, and the SQL as the final build.




## 📋 108. Here's the Plan

**What it does:** The **finished design** for the social photo app: every table, its columns, its relationships, its rules, and the **order to build them in**. Topics 105 to 107 were the how, and this is the result.
**Why it is used:** Having the whole plan on one page means topic 109 (and the rest) is just typing it in. It's also the checklist you'll verify the finished database against.
**When to use it:** As the reference while you build. This is the version of the plan I wrote for these notes, so adapt it if your course differs.

**Analogy:** The **bill of materials** on a construction site. Every part, every connection, and the order to assemble them, written down before the first nail.

### The diagram
```
                 ┌───────────┐
          ┌─────<│  followers│>─────┐       (users follow users)
          │      └───────────┘      │
          │                         │
        ┌─┴─────┐ 1          * ┌────┴──┐
        │ users │──────────────<│ posts │──────────< post_hashtags >──── hashtags
        └─┬──┬──┘               └───┬───┘
          │  │ 1             *      │ 1
          │  └─────────────<┌───────┴──┐
          │                 │ comments │
          │                 └────┬─────┘
          │ 1            *       │
          └──────────────< likes >┘   (a like targets a post OR a comment)
```

### The tables
| Table | Columns | Notes |
|---|---|---|
| **users** | `id`, `created_at`, `updated_at`, `username`, `bio`, `avatar`, `email`, `phone` | `username` is unique, and at least one of `email` / `phone` is required |
| **posts** | `id`, `created_at`, `updated_at`, `url`, `caption`, `lat`, `lng`, `user_id` | Coordinates are optional but must be in range |
| **comments** | `id`, `created_at`, `updated_at`, `contents`, `user_id`, `post_id` | Belongs to one user and one post |
| **likes** | `id`, `created_at`, `user_id`, `post_id`, `comment_id` | **Exactly one** of `post_id` / `comment_id` is filled |
| **followers** | `leader_id`, `follower_id`, `created_at` | Join table, and the primary key is the **pair** |
| **hashtags** | `id`, `created_at`, `title` | `title` is unique |
| **post_hashtags** | `post_id`, `hashtag_id` | Join table, and the primary key is the **pair** |

### The relationships
| Relationship | Type | Foreign key | On delete |
|---|---|---|---|
| A user has many posts | One-to-many | `posts.user_id` → `users.id` | `CASCADE` |
| A user writes many comments | One-to-many | `comments.user_id` → `users.id` | `CASCADE` |
| A post has many comments | One-to-many | `comments.post_id` → `posts.id` | `CASCADE` |
| A user gives many likes | One-to-many | `likes.user_id` → `users.id` | `CASCADE` |
| A post receives many likes | One-to-many | `likes.post_id` → `posts.id` | `CASCADE` |
| A comment receives many likes | One-to-many | `likes.comment_id` → `comments.id` | `CASCADE` |
| Users follow users | Many-to-many (self) | `followers.leader_id`, `followers.follower_id` → `users.id` | `CASCADE` |
| Posts have hashtags | Many-to-many | `post_hashtags.post_id` → `posts.id`, `post_hashtags.hashtag_id` → `hashtags.id` | `CASCADE` |

`CASCADE` everywhere keeps the data tidy: delete a user and everything they made goes with them (topic 35).

### The rules (constraints)
| Table | Rule | Tool |
|---|---|---|
| users | Username is unique | `UNIQUE` |
| users | Has an email or a phone | `CHECK (email IS NOT NULL OR phone IS NOT NULL)` |
| posts | Latitude is between -90 and 90 | `CHECK (lat BETWEEN -90 AND 90)` |
| posts | Longitude is between -180 and 180 | `CHECK (lng BETWEEN -180 AND 180)` |
| likes | A like has exactly one target | `CHECK (num_nonnulls(post_id, comment_id) = 1)` |
| likes | A user can like a given post once | Partial unique index on `(user_id, post_id)` |
| likes | A user can like a given comment once | Partial unique index on `(user_id, comment_id)` |
| followers | You can't follow yourself | `CHECK (leader_id <> follower_id)` |
| followers | You can follow someone once | `PRIMARY KEY (leader_id, follower_id)` |
| hashtags | A tag exists once | `UNIQUE (title)` |

### The build order
Parents before children, so every foreign key has something to point at (topic 27):
```
1. users
2. posts          (needs users)
3. comments       (needs users, posts)
4. likes          (needs users, posts, comments)
5. followers      (needs users)
6. hashtags
7. post_hashtags  (needs posts, hashtags)
```
Drop them in the **reverse** order (topic 33).

### The plan, in steps
| Step | Action | Topic |
|---|---|---|
| 1 | Create a **new database** so nothing existing is overwritten | 109 |
| 2 | Build `users`, `posts`, `comments`, `likes` with all their rules | 109 |
| 3 | Test each rule with a row that should **fail** | 109 |
| 4 | Build `followers`, `hashtags`, `post_hashtags` | next |
| 5 | Load sample data and run joins and aggregations on it | next |

> ⚠️ **Warning:** **A plan isn't final.** If a rule turns out to be wrong, change it now, while there's no data. Changing a constraint on a table full of rows takes extra work (topics 98 and 102).

> ⚠️ **Warning:** This plan doesn't store passwords. If you use Auth.js, your users sign in through a provider, and the auth library manages its own tables. **Never store plain-text passwords** in any design.

> ⚠️ **Warning:** `updated_at DEFAULT NOW()` only sets the value when a row is **inserted**. It doesn't change by itself on an `UPDATE`. Update it in your app, or use a trigger (a later topic).

> 🏆 **Golden Rule:** **Tables, relationships, rules, build order.** If you can write all four on one page, you can build the database without surprises.




## 🔨 109. Rebuilding Some Schema

**What it does:** Turns the plan from topic 108 into real SQL: it **rebuilds** `users`, `posts`, `comments`, and `likes` with every constraint from sections 95 to 104, in a **new database**, then proves each rule works.
**Why it is used:** The earlier versions of these tables (topics 27 and 35) had only foreign keys. Now you can add what real apps need: required columns, defaults, unique values, ranges, and cross-column checks.
**When to use it:** Whenever you move from "tables that work" to "tables that protect themselves."

**Analogy:** Renovating a house you built quickly. The rooms are the same, but now there are locks on the doors, smoke alarms, and fuse boxes.

### Step 1: a fresh database
This leaves `photo_app` alone:
```sql
CREATE DATABASE social_app;
```
**Output:**
```
CREATE DATABASE
```
```sql
\c social_app
```
**Output:**
```
You are now connected to database "social_app" as user "postgres".
```
(Prefer to rebuild inside `photo_app`? Drop the old tables first, children before parents: `comments`, then `photos`, then `users`.)

### What changed from the old versions
| Table | Before (topics 27 and 35) | After |
|---|---|---|
| `users` | Just `id`, `username` | Timestamps, `UNIQUE` username, contact `CHECK` |
| `photos` becomes `posts` | `url`, `user_id` | Required `url`, caption, validated coordinates |
| `comments` | `contents`, two foreign keys | `NOT NULL` everywhere, `CASCADE` |
| (new) `likes` | Didn't exist | Two nullable keys with an "exactly one" check |

### Step 2: `users`
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  username VARCHAR(30) NOT NULL UNIQUE,
  bio VARCHAR(400),
  avatar VARCHAR(200),
  email VARCHAR(255),
  phone VARCHAR(25),
  CONSTRAINT users_need_contact CHECK (email IS NOT NULL OR phone IS NOT NULL)
);
```
**Output:**
```
CREATE TABLE
```

### Step 3: `posts`
```sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  url VARCHAR(200) NOT NULL,
  caption VARCHAR(240),
  lat DOUBLE PRECISION,
  lng DOUBLE PRECISION,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  CONSTRAINT posts_lat_range CHECK (lat BETWEEN -90 AND 90),
  CONSTRAINT posts_lng_range CHECK (lng BETWEEN -180 AND 180)
);
```
**Output:**
```
CREATE TABLE
```
Coordinates are measurements, so `DOUBLE PRECISION` fits (topic 89). A `NULL` coordinate **passes** the checks (topic 102), which is what we want, since location is optional.

### Step 4: `comments`
```sql
CREATE TABLE comments (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  contents VARCHAR(240) NOT NULL,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE
);
```
**Output:**
```
CREATE TABLE
```

### Step 5: `likes`, the tricky one
```sql
CREATE TABLE likes (
  id SERIAL PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
  comment_id INTEGER REFERENCES comments(id) ON DELETE CASCADE,
  CONSTRAINT likes_exactly_one_target CHECK (num_nonnulls(post_id, comment_id) = 1)
);
```
**Output:**
```
CREATE TABLE
```
`num_nonnulls(a, b)` counts how many arguments are **not** `NULL`. Requiring it to equal `1` means exactly one target is filled in.

**Why not just `UNIQUE (user_id, post_id, comment_id)` to stop double likes?** Because of `NULL` (topic 101):
```sql
CREATE TEMP TABLE naive_likes (
  user_id INTEGER, post_id INTEGER, comment_id INTEGER,
  UNIQUE (user_id, post_id, comment_id)
);
INSERT INTO naive_likes VALUES (2, 1, NULL), (2, 1, NULL);
```
**Output:**
```
CREATE TABLE
INSERT 0 2
```
The same like went in **twice**, because `NULL` never equals `NULL`, so the rows never look like duplicates.

**The fix: two partial unique indexes**, each covering only the rows where its column is filled in:
```sql
CREATE UNIQUE INDEX likes_user_post_idx
  ON likes (user_id, post_id) WHERE post_id IS NOT NULL;

CREATE UNIQUE INDEX likes_user_comment_idx
  ON likes (user_id, comment_id) WHERE comment_id IS NOT NULL;
```
**Output:**
```
CREATE INDEX
CREATE INDEX
```
(On PostgreSQL 15 and newer, `UNIQUE NULLS NOT DISTINCT (user_id, post_id, comment_id)` is another way to do it. The partial indexes work on every version.)

### Verify the structure
```sql
\dt
```
**Output:**
```
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | comments | table | postgres
 public | likes    | table | postgres
 public | posts    | table | postgres
 public | users    | table | postgres
(4 rows)
```
```sql
\d likes
```
Look for the `Check constraints:` and `Foreign-key constraints:` sections, and for the two unique indexes under `Indexes:`.

### Step 6: load valid data
```sql
INSERT INTO users (username, email, phone)
VALUES
  ('monahan93', 'monahan@example.com', NULL),
  ('pferrer', NULL, '555-0102'),
  ('si93onis', 'si93@example.com', NULL);

INSERT INTO posts (url, caption, lat, lng, user_id)
VALUES
  ('http://one.jpg', 'Sunrise', 35.68, 139.69, 1),
  ('http://two.jpg', 'Lunch', NULL, NULL, 2);

INSERT INTO comments (contents, user_id, post_id)
VALUES ('Nice shot!', 2, 1), ('Love this', 3, 1);

INSERT INTO likes (user_id, post_id) VALUES (2, 1);
INSERT INTO likes (user_id, comment_id) VALUES (3, 1);
```
**Output:**
```
INSERT 0 3
INSERT 0 2
INSERT 0 2
INSERT 0 1
INSERT 0 1
```

### Step 7: break every rule on purpose
Each of these **should fail**. If one succeeds, the rule isn't working.

**A user with no way to be contacted:**
```sql
INSERT INTO users (username) VALUES ('nobody');
```
**Output:**
```
ERROR:  new row for relation "users" violates check constraint "users_need_contact"
```
(A `DETAIL:` line with the failing row follows.)

**A duplicate username:**
```sql
INSERT INTO users (username, email) VALUES ('pferrer', 'other@example.com');
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "users_username_key"
DETAIL:  Key (username)=(pferrer) already exists.
```

**A latitude off the globe:**
```sql
INSERT INTO posts (url, lat, lng, user_id) VALUES ('http://bad.jpg', 95, 0, 1);
```
**Output:**
```
ERROR:  new row for relation "posts" violates check constraint "posts_lat_range"
```

**A post with a user that doesn't exist:**
```sql
INSERT INTO posts (url, user_id) VALUES ('http://ghost.jpg', 999);
```
**Output:**
```
ERROR:  insert or update on table "posts" violates foreign key constraint "posts_user_id_fkey"
DETAIL:  Key (user_id)=(999) is not present in table "users".
```

**A like with two targets, and a like with none:**
```sql
INSERT INTO likes (user_id, post_id, comment_id) VALUES (1, 1, 1);
INSERT INTO likes (user_id) VALUES (1);
```
**Output:**
```
ERROR:  new row for relation "likes" violates check constraint "likes_exactly_one_target"
ERROR:  new row for relation "likes" violates check constraint "likes_exactly_one_target"
```

**Liking the same post twice:**
```sql
INSERT INTO likes (user_id, post_id) VALUES (2, 1);
```
**Output:**
```
ERROR:  duplicate key value violates unique constraint "likes_user_post_idx"
DETAIL:  Key (user_id, post_id)=(2, 1) already exists.
```

### Step 8: check the data, then test the cascade
```sql
SELECT id, user_id, post_id, comment_id FROM likes ORDER BY id;
```
**Output:**
```
 id | user_id | post_id | comment_id
----+---------+---------+------------
  1 |       2 |       1 |
  2 |       3 |         |          1
(2 rows)
```
Blank cells are `NULL`. The failed inserts above used up some ids, but they stored no rows.

**Delete one post and see what disappears:**
```sql
DELETE FROM posts WHERE id = 1;

SELECT
  (SELECT COUNT(*) FROM comments) AS comments,
  (SELECT COUNT(*) FROM likes) AS likes;
```
**Output:**
```
DELETE 1
 comments | likes
----------+-------
        0 |     0
(1 row)
```
Deleting post 1 removed its two comments, the like on the post, **and** the like on one of those comments. That's the cascade chain from topic 35.

> ⚠️ **Warning:** **Failed inserts still consume `SERIAL` ids** (topic 26), so your ids may skip numbers. That's normal and harmless.

> ⚠️ **Warning:** **Test the failures, not just the successes.** A constraint that never rejects anything is a constraint you haven't verified.

> ⚠️ **Warning:** `updated_at` doesn't change by itself when you `UPDATE` a row. Set it yourself in the query (`SET updated_at = NOW()`), or use a trigger (a later topic).

> ⚠️ **Warning:** `num_nonnulls` is PostgreSQL-specific. In other databases you'd write the check with `CASE`, such as `CHECK ((post_id IS NOT NULL AND comment_id IS NULL) OR (post_id IS NULL AND comment_id IS NOT NULL))`.

> 🏆 **Golden Rule:** **Rebuild the tables, then try to break them.** If every bad row you throw at the schema is refused, and every valid row goes in, the design is doing its job.