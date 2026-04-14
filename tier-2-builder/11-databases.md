# Databases: How Your Application Remembers

## What is it?

A database is a structured way to store and retrieve data. Your application needs to remember things: users, posts, orders, sessions, preferences. The database is where that memory lives.

The two broad categories you'll encounter are **relational** (SQL) and **document** (NoSQL) stores.

**Relational databases** store data in tables with rows and columns. Each row is a record; each column is a field. Tables relate to each other through foreign keys. The big players are PostgreSQL, MySQL, and SQLite. SQL (Structured Query Language) is the language you use to talk to them.

Think of a relational database like a spreadsheet with multiple tabs. One tab has users, one has posts, and the posts tab has a column that references the users tab. Everything is normalized — each piece of data lives in exactly one place.

**Document databases** store data as documents — usually JSON-like objects. MongoDB and Firestore are the common ones. A document for a user might contain their posts, preferences, and recent activity all in one place.

Think of a document database like a filing cabinet where each folder contains everything about one entity. No joins, no relationships to maintain explicitly.

**Key-value stores** (Redis, DynamoDB) are a simpler form — you store a key and retrieve its value. Great for caching and session data. Not for complex queries.

## Why it matters for vibe coding

Without database knowledge, you cannot review AI-generated code responsibly. AI will generate schemas that duplicate data, migrations that destroy production data, queries that are catastrophically slow, and transaction patterns that corrupt state. You will not know why until users report data loss.

Here are the specific failure modes:

**AI generates migrations that drop columns with data.** When you ask AI to "add a new field" or "rename a column," it often generates a migration that recreates the table — dropping the old column in the process. In development this is fine. In production with real data, it is a catastrophe. You will not know this migration is dangerous until you read it.

**AI doesn't understand foreign key constraints.** AI will generate code that inserts a row referencing a parent record that doesn't exist, or deletes a parent record while children still reference it. The generated code looks syntactically correct. It fails at runtime with a cryptic constraint violation that the AI will try to "fix" by removing the constraint entirely — making the problem worse.

**AI generates indexes that don't match the queries actually used.** AI often adds indexes speculatively without looking at the actual query patterns. You'll end up with 40 indexes on a table, but the slow query that users complain about has no index at all. Meanwhile, write performance degrades with every unused index.

**AI doesn't understand transaction boundaries.** AI will generate code that reads data, makes a decision, and writes based on that decision — without wrapping it in a transaction. Under concurrent load, this produces race conditions where two requests read the same state, both make decisions, and one overwrites the other. Users call this "my data disappeared."

**AI puts secrets in code instead of environment variables.** AI will hardcode database URLs, API keys, and passwords directly in configuration files because that's how the code is structured in training data. This is how breaches happen. You must know to use environment variables and `.env` files.

## The 20% you need to know

### SQL vs. NoSQL: When to use each

Use a relational database (PostgreSQL is the standard choice) when:

- Your data has clear relationships (users have posts, posts have comments, comments have authors)
- You need complex queries that join multiple tables
- Data integrity is critical (financial transactions, inventory)
- You need ACID guarantees (see below)
- You're building something that will scale with predictable data patterns

Use a document database when:

- Your data structure varies per entity (a user profile that can have arbitrary custom fields)
- You need to store nested, hierarchical data that doesn't map cleanly to tables
- You're optimizing for read-heavy workloads with simple queries
- You're prototyping and schema flexibility matters more than query power

For everything you're building right now as a vibe coder: **use PostgreSQL**. It is the default choice for a reason. It has the best combination of relational power, JSON support (you can store document-like data when needed), and ecosystem maturity. SQLite is fine for local development and simple apps. MySQL is a legacy choice that offers few advantages over Postgres. MongoDB is only worth considering when you have a specific, documented reason to use it.

### Schema Design and Normalization

A schema defines the structure of your data — what tables exist, what columns each table has, and how tables relate to each other.

**Normalization** is the practice of structuring your schema to reduce data duplication. Each piece of data lives in exactly one place. If you need it, you reference it by ID, not by copying it.

Consider a `posts` table. A bad schema copies the author's name into the `posts` table:

```
users: id, name, email
posts: id, author_name (copied!), content, created_at
```

What happens when Alice changes her name? You have to update every post she ever wrote. A normalized schema references the user:

```
users: id, name, email
posts: id, author_id (reference), content, created_at
```

Now you update the name in one place and every query that joins users to posts gets the correct name automatically.

**Basic normalization levels (forms):**
- **1NF**: Each cell has a single value (no arrays or lists in a single column)
- **2NF**: No partial dependencies (every non-key column depends on the whole primary key)
- **3NF**: No transitive dependencies (a column shouldn't depend on another non-key column)

In practice, you don't need to memorize the forms. Just apply the rule: don't duplicate data across tables. If you find yourself copying a value (like `author_name`) into multiple tables, split it out and use a foreign key reference.

Denormalization (intentionally duplicating data) is sometimes appropriate for performance — for example, caching a post's comment count directly on the post record so you don't have to count every time. This is a deliberate trade-off, not a failure of normalization.

### What Migrations Are and Why They Matter

A migration is a versioned change to your database schema. You start at schema v1, apply migrations to get to v2, v3, and so on. Migrations are code, so they live in your repository next to your application code.

Migrations matter for three reasons:

1. **Reproducibility** — Any developer (or CI pipeline) can start with an empty database and apply every migration in order to reach the current schema.
2. **Auditability** — You can see exactly what changed, when, and why (via commit history).
3. **Safe rollbacks** — If a migration causes problems, you can roll back to a previous state.

A migration has two parts: an `up` (apply the change) and a `down` (undo the change). Example:

```sql
-- Migration: add_users_table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Rollback:
DROP TABLE users;
```

The most common migration mistake that destroys data: using `ALTER TABLE ... DROP COLUMN` or `ALTER TABLE ... TYPE` on a column that has existing data. The database drops the column and all its data. There is no recycle bin.

Safe migrations for production:
- Add nullable columns first. Deploy code that writes to the new column.
- Backfill existing rows with default values.
- Add a NOT NULL constraint once all existing rows are filled.
- Use renaming carefully: add the new column, migrate data, drop the old column — never rename in place.

### ORMs and the SQL They Generate

An ORM (Object-Relational Mapper) is a library that lets you interact with the database using code that looks like your programming language, instead of writing SQL directly. Prisma, Sequelize, SQLAlchemy, and Hibernate are common ORMs.

The appeal is obvious: instead of writing `SELECT * FROM users WHERE email = $1`, you write `users.find({ email: 'alice@example.com' })`. This is more readable and works across database types.

The problem: ORMs generate SQL, and the SQL they generate is often inefficient. You need to know what SQL your ORM is producing so you can catch bad queries.

Example: Prisma query
```typescript
const users = await prisma.user.findMany({
  where: { email: 'alice@example.com' }
});
```

Generates roughly:
```sql
SELECT id, name, email, created_at FROM users WHERE email = 'alice@example.com';
```

But a complex Prisma query with nested includes can generate dozens of queries (the "N+1 problem") or return far more data than you need.

The rule: treat your ORM as a tool that generates SQL, not a replacement for SQL. When you write a complex query, write the SQL first and then figure out how to express it in your ORM. When you get unexpected behavior, log the raw SQL and inspect it.

### Indexing Basics

An index is a data structure that makes reads faster. Without an index, the database has to scan every row in a table to find what you're looking for (a "full table scan"). With an index on a column, the database jumps directly to the matching rows.

Indexes are not free. Every write (INSERT, UPDATE, DELETE) on an indexed column must update the index. Too many indexes slow down writes and consume disk space. The rule: only add an index when you have a demonstrable slow query, and drop it if the query pattern changes.

When a query needs an index:
- It filters by a column (`WHERE email = '...'` or `WHERE status = 'pending'`)
- It sorts by a column (`ORDER BY created_at`)
- It joins on a column (`JOIN ON users.id = posts.author_id`)

A composite index covers multiple columns in order. An index on `(status, created_at)` helps queries that filter by status first, then sort by created_at. It does NOT help queries that only filter by `created_at`.

Check if a query needs an index with `EXPLAIN` (or `EXPLAIN ANALYZE`):

```sql
EXPLAIN SELECT * FROM posts WHERE author_id = 123;
```

The output shows whether the database is doing a sequential scan (bad for large tables) or using an index (good). If you see `Seq Scan` on a large table for a frequent query, add an index.

### ACID Transactions

ACID is a set of guarantees that a properly configured relational database provides for operations wrapped in a transaction:

- **Atomicity** — All operations in a transaction succeed or all fail together. No partial state.
- **Consistency** — The database moves from one valid state to another. Constraints are never violated.
- **Isolation** — Concurrent transactions don't interfere with each other. The result of running two transactions simultaneously is the same as running them one after another.
- **Durability** — Once a transaction commits, the data is persisted even if the database crashes.

When you need ACID: financial transactions, inventory updates, any operation where reading stale data or having a partial write would cause real problems.

Example of a transaction:

```sql
BEGIN;

-- Deduct from sender
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- Add to receiver
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Both succeeded, commit
COMMIT;
```

If the second UPDATE fails, the first is automatically rolled back. The account balance is unchanged.

Without a transaction, if the second UPDATE fails, the sender loses $100 and the receiver never receives it. This is not a theoretical problem — under concurrent load, non-transactional code produces exactly these race conditions.

### Connection Pooling

Every database connection consumes server resources. Opening a new connection for every request is slow and eventually exhausts the database's connection limit.

A **connection pool** maintains a set of open connections that are reused across requests. When your application needs a connection, it borrows one from the pool. When done, it returns it. The pool grows and shrinks within configured bounds.

PostgreSQL's default limit is often 100 connections. If your app has 500 concurrent users and each opens its own connection, the database refuses connections.

Common pooling tools: **PgBouncer** (for Postgres), **Prisma's built-in pooler**, **Knex.js connection pool**.

Environment variable for a database URL:

```
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
```

Never hardcode this. Store it in `.env` and load it at runtime. The `.env` file is not deployed — it stays on your machine. The deployed application uses environment variables set by the hosting platform.

## Hands-on Exercise

**Build a schema, run migrations, and inspect what the ORM generates.**

Time: 15 minutes

1. Set up a Node project with Prisma and SQLite (no database server needed):

```bash
mkdir db-demo && cd db-demo
npm init -y
npm install prisma @prisma/client
npx prisma init --datasource-provider sqlite
```

2. Define a schema. Edit `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())
  posts     Post[]
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id])
  createdAt DateTime @default(now())
}
```

3. Add to `.env`:

```
DATABASE_URL="file:./dev.db"
```

4. Generate the client and run the initial migration:

```bash
npx prisma migrate dev --name init
```

5. Inspect the generated SQL. Run this in a Node script:

```javascript
// script.js
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

async function main() {
  // See the SQL for a simple find
  const user = await prisma.user.findUnique({
    where: { email: 'alice@example.com' }
  });
}

main().finally(() => prisma.$disconnect());
```

Run with Prisma's query logging to see the actual SQL:

```javascript
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient({
  log: ['query'],
});

async function main() {
  await prisma.user.create({
    data: { name: 'Alice', email: 'alice@example.com' }
  });

  const user = await prisma.user.findUnique({
    where: { email: 'alice@example.com' },
    include: { posts: true }
  });

  console.log('User with posts:', JSON.stringify(user, null, 2));
}

main().finally(() => prisma.$disconnect());
```

```bash
node script.js
```

6. Read the generated SQL from the logs. Notice:
   - How the `include: { posts: true }` generates a JOIN
   - How `findUnique` generates `SELECT ... WHERE id = ?` or `WHERE email = ?`
   - Whether the ORM generates additional queries (N+1 detection)

7. Add an index by editing the schema:

```prisma
model Post {
  // ...
  @@index([authorId])  // Add this line
}
```

Run `npx prisma migrate dev --name add_post_author_index` and notice the generated SQL creates an index.

**Deliverable:** You see the SQL your ORM generates. You understand that the ORM is a translation layer, not a replacement for SQL knowledge.

## Common Mistakes

**Mistake 1: Dropping columns in production migrations without checking for data loss.**

What happens: AI or a developer runs a migration that ALTER TABLE ... DROP COLUMN. All data in that column is gone instantly.

Why it happens: In development, the column is empty. The migration succeeds. In production, the column has millions of rows of data.

How to fix: Every destructive migration (dropping columns, dropping tables, changing column types) should be reviewed by a human before running against production data. Always take a backup before running migrations on production. Use `pg_dump` for Postgres or configure auto-backups on your managed database.

**Mistake 2: Adding NOT NULL constraints to columns with existing NULL values.**

What happens: The migration runs and fails, or succeeds and corrupts data by inserting empty strings or default values you didn't intend.

Why it happens: The constraint check runs as part of the migration, not after backfilling.

How to fix: Add the column as nullable first, backfill all existing rows with a default value, then add the NOT NULL constraint in a separate migration. Three steps instead of one.

**Mistake 3: Not using transactions for multi-step writes.**

What happens: A billing operation debits one account but fails before crediting another. The customer is charged but the recipient never receives the funds. Race conditions produce duplicate charges.

Why it happens: Each step is a separate database call without a transaction wrapper.

How to fix: Wrap all related writes in a transaction. In Prisma: `prisma.$transaction([...])`. In raw SQL: `BEGIN ... COMMIT`. If any step fails, all steps roll back.

**Mistake 4: Creating an index on every column "for performance."**

What happens: Write performance degrades because every INSERT and UPDATE must update multiple indexes. Disk usage increases. The database spends more time maintaining indexes than answering queries.

Why it happens: It feels like indexes are "free" optimization.

How to fix: Only add indexes for demonstrable slow queries. Use `EXPLAIN ANALYZE` to confirm an index is used. If a query doesn't appear in your slow query log, don't index its columns preemptively.

**Mistake 5: Hardcoding database credentials in source code or committing .env files.**

What happens: Credentials end up in GitHub, in Docker images, in Slack messages. Attackers scan GitHub for database URLs constantly.

Why it happens: It's the path of least resistance and AI often generates code this way.

How to fix: Use environment variables. Use a `.env` file that is in `.gitignore`. Never commit `.env` to version control. Use a secret manager (AWS Secrets Manager, HashiCorp Vault) in production.

## AI-Specific Pitfalls

**AI generates migrations that drop or rename columns in place.** When you ask AI to "rename the `name` column to `full_name`," it often generates `ALTER TABLE users RENAME COLUMN name TO full_name`. This works — until it doesn't. If your ORM (Prisma, etc.) has cached the old schema, it generates SQL against the old column name until you regenerate the client. Rename by adding a new column, migrating data in a transaction, then dropping the old column.

**AI doesn't understand foreign key constraint implications.** AI will generate code that deletes a user without first deleting their posts, causing a constraint violation. Or it will "solve" the constraint error by removing the FK entirely, creating orphaned records. Always verify AI-generated delete patterns handle child records — either through `ON DELETE CASCADE` in the schema or explicit deletion of children before parents.

**AI generates indexes based on the schema, not the queries.** AI sees a column called `email` and adds an index because "emails should be indexed." This is correct for a `WHERE email = ?` filter, but useless if your actual query is `WHERE lower(email) = ?` (which requires a functional index). Look at the actual query patterns in your application before trusting AI's indexing advice.

**AI doesn't understand transaction boundaries in ORMs.** AI will generate two separate ORM calls — `create()` then `create()` — expecting them to be atomic, when they are actually two separate transactions. If the second fails, the first is committed. Wrap multi-step writes in `$transaction` or equivalent.

**AI generates connection strings with passwords in plain text.** AI will output `postgresql://user:password@host/db` in code examples. Passwords belong in environment variables, not in code. When reviewing AI output, treat any hardcoded credential as a critical security issue.

**AI uses SQLite syntax for Postgres or vice versa.** Minor differences — like `SERIAL` vs. `AUTO_INCREMENT`, or `now()` vs. `CURRENT_TIMESTAMP` — will cause silent bugs or runtime errors. Don't mix syntaxes. Know which database you're targeting.

## Quick Reference

### SQL vs. NoSQL Decision Tree

```
Is your data highly structured with clear relationships?
  → Yes: Use PostgreSQL
Is schema flexibility more important than query power?
  → Yes: Consider MongoDB (with documented justification)
Do you need ACID guarantees?
  → Yes: Use PostgreSQL (ACID is default, not optional)
Is this a simple prototype or local tool?
  → Yes: SQLite is fine
Are you building a cache or session store?
  → Yes: Redis
```

### Index Decision Tree

```
Does the query filter/sort/join on this column?
  → No: No index needed
Is the column highly selective (>5% of rows match)?
  → Consider: Index may not help, test with EXPLAIN
Is the table large (>10k rows)?
  → Yes: Test with EXPLAIN ANALYZE
Is this a write-heavy table?
  → Yes: Fewer indexes, test performance carefully
```

### Migration Safety Checklist

- [ ] Is this migration destructive (drop column, drop table, change type)?
- [ ] If yes: Is there a backup? Has data been backfilled?
- [ ] Is NOT NULL being added to a column that might have NULLs?
- [ ] Is this running against production data?
- [ ] Is there a rollback migration ready?
- [ ] Does the ORM client need to be regenerated?

### ACID vs. No-ACID

| Feature | ACID (PostgreSQL) | No-ACID (MongoDB default) |
|---|---|---|
| Transactions | Full ACID | Single document only |
| Consistency | Always consistent | Eventual consistency possible |
| Foreign keys | Enforced | Not enforced |
| Use when | Data integrity is critical | Schema flexibility, scale |

## Go Deeper

- [PostgreSQL Documentation](https://www.postgresql.org/docs/) (verified 2026-04) — The definitive reference. Start with the tutorial and the SQL chapter.
- [Prisma Documentation: Data Modeling](https://www.prisma.io/docs/orm/prisma-schema) (verified 2026-04) — Good for understanding how ORMs model schemas, applicable even if you use a different ORM.
- [Use the Index, Luke](https://use-the-index-luke.com/) (verified 2026-04) — The best free resource on database indexing. Practical, SQL-agnostic, by a database consultant.
- [Postgres Guide: Transactions](http://www.postgresguide.com/transactions/) (verified 2026-04) — Concise explanation of ACID with Postgres examples.
- [Database of Database Regulated: Migration Tools](https://github.com/dbt-labs/corp/blob/main/blog/2024-01-15-database-migration-tools.md) (verified 2026-04) — Survey of migration approaches across frameworks.
