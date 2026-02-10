---
layout: post
title: "SQLite Is the Database You Already Have"
date: 2026-02-11
tags: [sqlite, nodejs, database, architecture]
description: "Most web apps don't need Postgres. SQLite handles millions of rows, supports concurrent reads, and deploys as a single file. Here's when and how to use it."
---

You're building a web app. You set up Postgres. You configure connection pools, manage migrations, worry about connection limits, and pay for a managed database. For what? To store 50,000 rows that fit in 2 MB.

SQLite ships with every programming language. It handles millions of rows. It supports concurrent reads via WAL mode. And it deploys as a single file you can backup with `cp`.

## When SQLite is enough

SQLite works when:
- Your app runs on **one server** (no horizontal scaling needed)
- **Read-heavy** workload (which most web apps are — 90%+ reads)
- Database size under **1 TB** (SQLite's practical limit)
- You want **zero operational overhead** (no connection pools, no separate service)

SQLite doesn't work when:
- Multiple servers need to write to the same database
- You need replication or failover
- Heavy concurrent writes (100+ writes/second sustained)

Most solo developer projects, internal tools, and even moderate SaaS products fall squarely in the "SQLite is enough" category.

## WAL mode: the essential setting

By default, SQLite blocks readers while writing. WAL (Write-Ahead Logging) mode fixes this:

```javascript
import Database from 'better-sqlite3';

const db = new Database('app.db');
db.pragma('journal_mode = WAL');
```

With WAL:
- **Readers never block writers.** Multiple concurrent reads proceed without waiting.
- **Writers never block readers.** A write in progress doesn't stall read queries.
- **One writer at a time.** Writes are serialized — but each write is fast (microseconds for simple inserts).

This single pragma transforms SQLite from "embedded database" to "web-capable database."

## Prepared statements: performance and safety

Always use prepared statements. They're faster (compiled once, executed many times) and prevent SQL injection:

```javascript
// Prepare once at startup
const getUser = db.prepare('SELECT * FROM users WHERE id = ?');
const createUser = db.prepare(
  'INSERT INTO users (id, name, email) VALUES (?, ?, ?)'
);

// Execute many times
const user = getUser.get(userId);      // .get() returns one row
const users = getUser.all();           // .all() returns array
createUser.run(id, name, email);       // .run() for writes
```

With `better-sqlite3` (Node.js), prepared statements are synchronous — no callbacks, no promises, no connection pool. This is intentional: SQLite operations are so fast that async overhead would actually slow things down.

## Transactions: the secret weapon

SQLite transactions are powerful and often overlooked:

```javascript
const transferFunds = db.transaction((fromId, toId, amount) => {
  const from = getBalance.get(fromId);
  if (from.balance < amount) throw new Error('Insufficient funds');

  debit.run(fromId, amount);
  credit.run(toId, amount);
  logTransfer.run(fromId, toId, amount);
});

// Atomic: all succeed or all fail
transferFunds('alice', 'bob', 100);
```

Transactions in `better-sqlite3` also batch writes efficiently. Inserting 1,000 rows individually: ~1 second. Inside a transaction: ~5 milliseconds. That's a 200x speedup.

## Database as queue

One of the most underappreciated SQLite patterns: using a table as a job queue.

```sql
CREATE TABLE jobs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  payload TEXT NOT NULL,
  created TEXT NOT NULL DEFAULT (datetime('now')),
  processed TEXT  -- NULL = pending
);

CREATE INDEX idx_jobs_pending ON jobs(created) WHERE processed IS NULL;
```

Enqueue:
```javascript
const enqueue = db.prepare(
  'INSERT INTO jobs (payload) VALUES (?)'
);
enqueue.run(JSON.stringify({ task: 'send_email', to: 'user@example.com' }));
```

Dequeue and process:
```javascript
const dequeue = db.prepare(`
  UPDATE jobs SET processed = datetime('now')
  WHERE id = (SELECT id FROM jobs WHERE processed IS NULL ORDER BY created LIMIT 1)
  RETURNING *
`);

// Process one job at a time
const job = dequeue.get();
if (job) {
  const payload = JSON.parse(job.payload);
  // do work...
}
```

No Redis. No RabbitMQ. No separate infrastructure. The partial index on `processed IS NULL` keeps the query fast even with millions of completed jobs in the table.

## Schema as code

Keep your schema in a SQL file and run it on startup:

```javascript
import { readFileSync } from 'node:fs';

const schema = readFileSync('schema.sql', 'utf-8');
db.exec(schema);
```

Using `CREATE TABLE IF NOT EXISTS` and `CREATE INDEX IF NOT EXISTS` makes this idempotent — safe to run every time the app starts. For more complex migrations, number your migration files and track which ones have been applied:

```sql
CREATE TABLE IF NOT EXISTS migrations (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL,
  applied TEXT NOT NULL DEFAULT (datetime('now'))
);
```

## Backup is a file copy

```bash
# Hot backup (safe while app is running with WAL mode)
sqlite3 app.db ".backup backup.db"

# Or just copy the file (safe if no writes are in progress)
cp app.db app-backup.db
```

Compare this to `pg_dump`, managed database snapshots, or S3 backup pipelines. SQLite backup is a file operation.

## Real-world scale

SQLite handles more than you think:
- [Expensify](https://blog.expensify.com/2018/01/08/scaling-sqlite-to-4m-qps-on-a-single-server/) runs 4 million queries per second on SQLite
- [Pieter Levels](https://x.com/levelsio) runs multiple profitable SaaS products on SQLite
- Cloudflare D1 is a distributed SQLite service powering edge applications
- Turso and LiteFS enable SQLite replication for when you do need multiple servers

The SQLite engine itself is one of the most tested pieces of software ever written — 100% branch coverage, millions of test cases.

## When to graduate

Move to Postgres when:
1. You need multiple application servers writing to the same database
2. You need real-time replication or read replicas
3. You need advanced features: full-text search with custom tokenizers, JSONB with GIN indexes, PostGIS
4. Write throughput exceeds what a single SQLite file can handle (~50K writes/second)

For most projects, that day never comes. And if it does, the migration path is straightforward — SQL is SQL.

## Getting started

```bash
npm install better-sqlite3
```

```javascript
import Database from 'better-sqlite3';

const db = new Database('myapp.db');
db.pragma('journal_mode = WAL');

db.exec(`
  CREATE TABLE IF NOT EXISTS items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    created TEXT NOT NULL DEFAULT (datetime('now'))
  );
`);

const insert = db.prepare('INSERT INTO items (name) VALUES (?)');
const getAll = db.prepare('SELECT * FROM items ORDER BY created DESC');

insert.run('First item');
console.log(getAll.all());
```

That's a complete database setup. No server, no connection string, no Docker container. Just a file.
