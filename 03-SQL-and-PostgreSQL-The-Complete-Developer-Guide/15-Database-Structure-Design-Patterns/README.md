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