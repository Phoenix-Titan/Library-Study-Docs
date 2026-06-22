# MongoDB & Document Database Design — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** Anyone going from "I've heard MongoDB stores JSON" to "I can model, index, query, aggregate, scale and operate a document database in production" — without an internet connection. Every concept is explained in prose first (what it is, *why* it exists, when to reach for it), then shown with heavily-commented examples. App-layer code is given in **both** Node.js (the native `mongodb` driver **and** Mongoose, from Express/NestJS) **and** Go (the official `mongo-go-driver`, from Gin). Read top-to-bottom the first time; afterwards use the Table of Contents as a reference. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** This guide targets **MongoDB 8.x (current in 2026)**. Things worth knowing about the modern server:
> - **Aggregation framework** is the primary query/analytics engine and keeps gaining operators every release. It is assumed throughout.
> - **Time series collections** (GA since 5.0, much improved in 6/7/8) are a first-class storage type — you rarely hand-roll the bucket pattern anymore, but you must understand it.
> - **Multi-document ACID transactions** work on replica sets (4.0+) and sharded clusters (4.2+). Single-document writes have always been atomic.
> - **Vector Search & Atlas Search** (managed Lucene + HNSW vector indexes on MongoDB Atlas) make Mongo a viable store for AI/RAG embeddings. Covered in §14.
> - **Queryable Encryption**, **Atlas Stream Processing**, **`$vectorSearch`**, and richer `$lookup`/`$documents`/`$densify` stages are all current.
> - The default storage engine is **WiredTiger** (document-level concurrency, compression, snapshots).
>
> Where behaviour is version-sensitive it is flagged with **⚡ Version note**. The author is on **Windows 11**, so cross-platform notes (Docker, paths, shells) are called out. Confirm exact APIs at mongodb.com/docs.

---

## Table of Contents

1. [What MongoDB Is & When to Use It](#1-what-mongodb-is--when-to-use-it) **[B]**
2. [CRUD in the Shell (mongosh)](#2-crud-in-the-shell-mongosh) **[B]**
3. [Data Modeling — Embedding vs Referencing](#3-data-modeling--embedding-vs-referencing) **[B/I]**
4. [Schema Design Patterns](#4-schema-design-patterns) **[I/A]**
5. [Indexing](#5-indexing) **[I]**
6. [The Aggregation Pipeline](#6-the-aggregation-pipeline) **[I/A]**
7. [Schema Validation](#7-schema-validation) **[I]**
8. [Transactions](#8-transactions) **[I/A]**
9. [Relationships & Joins](#9-relationships--joins) **[I]**
10. [Replication & Sharding](#10-replication--sharding) **[A]**
11. [Node.js Usage — Native Driver & Mongoose](#11-nodejs-usage--native-driver--mongoose) **[I/A]**
12. [Go Usage — mongo-go-driver from Gin](#12-go-usage--mongo-go-driver-from-gin) **[I/A]**
13. [Performance & Operations](#13-performance--operations) **[A]**
14. [Vector Search & Atlas Search](#14-vector-search--atlas-search) **[A]**
15. [Anti-Patterns & Gotchas](#15-anti-patterns--gotchas) **[I/A]**
16. [Study Path & Build-to-Learn Projects](#16-study-path--build-to-learn-projects)

---

## 1. What MongoDB Is & When to Use It

### 1.1 The document model **[B]**

MongoDB is a **document database** (often shelved under "NoSQL"). Instead of storing data in rows and columns spread across normalized tables like a relational database (PostgreSQL, MySQL), it stores **documents** — self-contained, nested, JSON-like records — grouped into **collections**, grouped into **databases**.

A document is a set of field/value pairs. Values can be scalars (string, number, boolean, date, null), **arrays**, or **nested sub-documents** (objects within objects). This is the single most important idea: a document can hold a whole *object graph* — an order with its line items, a blog post with its tags and embedded author summary — in one place, retrievable in one read. Relational databases force you to split that graph across tables and re-assemble it with JOINs at query time; MongoDB lets you store it the shape your application actually uses it in.

The mental mapping from the relational world:

| Relational (SQL) | MongoDB | Notes |
|---|---|---|
| Database | Database | Same concept |
| Table | Collection | A collection has *no fixed schema* by default |
| Row | Document | A BSON object, up to 16 MB |
| Column | Field | Fields can differ between documents in a collection |
| Primary key | `_id` field | Auto-created, unique, indexed; usually an `ObjectId` |
| JOIN | Embedding **or** `$lookup` | Often you *don't* join — you embed |
| Foreign key + JOIN | `ObjectId` reference + `$lookup` | No DB-enforced referential integrity |

"Schema-less" is a misleading slogan. MongoDB does not *enforce* a schema unless you ask it to (see §7), but **every successful application has a schema** — it just lives in your application code and your design discipline rather than in `CREATE TABLE` statements. The work shifts from "design tables and normalize" to "design documents around access patterns." That is what most of this guide is about.

### 1.2 BSON vs JSON **[B]**

You write and read documents that *look* like JSON, but MongoDB stores **BSON** ("Binary JSON") on disk and on the wire. Why a binary format instead of plain text JSON?

- **More types.** JSON has only string, number, boolean, null, array, object. BSON adds **proper types** that databases need: distinct 32-bit (`int`) and 64-bit (`long`) integers, `double`, `Decimal128` (exact decimal for money), `Date` (a 64-bit ms-since-epoch timestamp), `ObjectId`, `BinData` (binary blobs), and more. This matters: in JSON, `5` and `5.0` are ambiguous; in BSON they are an `int` and a `double`.
- **Speed.** BSON is length-prefixed and traversable without parsing the whole document, so the server can skip fields and index efficiently.
- **Lossless round-trips.** Your `Date` stays a `Date`, your `Decimal128` stays exact.

When you `JSON.stringify` a document you lose this richness, so MongoDB defines **Extended JSON** (a JSON encoding that tags types, e.g. `{"$oid": "..."}`, `{"$date": "..."}`, `{"$numberDecimal": "9.99"}`) for tools and import/export. You rarely type Extended JSON by hand; drivers handle conversion.

```js
// In mongosh these constructors create the proper BSON types:
ObjectId("507f1f77bcf86cd799439011")  // a 12-byte ObjectId
ISODate("2026-06-21T10:00:00Z")        // a BSON Date (NOT a string!)
NumberInt(42)                          // 32-bit int  (default JS numbers become double)
NumberLong("9007199254740993")         // 64-bit long
NumberDecimal("19.99")                 // Decimal128 — use this for money, never double
```

> **Gotcha:** Plain JavaScript numbers in mongosh become BSON `double`. If you need an exact integer count or currency, wrap them (`NumberInt`, `NumberLong`, `NumberDecimal`). Storing money as a `double` causes the classic `0.1 + 0.2 != 0.3` rounding errors.

### 1.3 ObjectId — the default primary key **[B]**

Every document has a unique `_id`. If you don't provide one on insert, MongoDB generates an **`ObjectId`**: a 12-byte value composed of a 4-byte Unix timestamp (seconds), a 5-byte random value per process, and a 3-byte incrementing counter. Properties worth internalizing:

- **Globally unique without coordination.** Any client can generate one offline; collisions are practically impossible. This is huge for distributed systems — you don't need a central sequence/auto-increment.
- **Roughly time-ordered.** Because the high bytes are a timestamp, sorting by `_id` ≈ sorting by creation time, and you can extract the creation time: `myId.getTimestamp()`.
- It is a **12-byte binary value, not a string.** A common bug is storing some references as `ObjectId` and others as the 24-char hex *string* — they will not match in queries or `$lookup`. Pick one and be consistent (binary `ObjectId` is the convention).

You *may* use your own `_id` (e.g. a SKU, an email, a UUID) when you have a natural key — it saves an index and a lookup. Just make sure it's immutable and unique.

### 1.4 When a document DB fits — the honest trade-offs **[B/I]**

Document databases are not "better than" relational databases; they make a different trade. Use this framing honestly.

**MongoDB tends to fit well when:**
- Your data is naturally **hierarchical / object-shaped** and is **read and written as a unit** (a product with its variants, a user with their settings, an event with its payload). One read returns the whole thing.
- Your **schema evolves quickly** or **varies per record** (catalogs where every product category has different attributes; event/log payloads with different shapes).
- You need to **scale writes/reads horizontally** across many machines; sharding is built in.
- You favor **developer velocity** and want the stored shape to match your application objects.

**A relational database is usually the better choice when:**
- Your data is highly **relational and queried in many different combinations** (lots of ad-hoc JOINs across many entities). SQL JOINs across normalized tables are more flexible than `$lookup`.
- You need **strict, multi-row transactional integrity across many entities** as the norm (accounting ledgers, inventory with complex constraints). Mongo *can* do multi-document transactions (§8), but if *most* operations need them, the document model is fighting you.
- You rely on **rich declarative constraints** (foreign keys, `CHECK`, complex uniqueness across tables) enforced by the database.
- Your team and tooling are deeply SQL-centric and the data is tabular.

The deepest trade-off: **MongoDB asks you to optimize the data layout for your access patterns up front** (denormalize, embed, duplicate), trading some flexibility and some write-time duplication for fast, single-read access. Relational databases keep data normalized (each fact stored once) and pay the cost at read time with JOINs. "Model around your queries" is the MongoDB mantra; if you don't yet know your queries, that's a risk.

> **The #1 mistake:** treating MongoDB like a relational database — one collection per "table," `ObjectId` foreign keys everywhere, and `$lookup` to reassemble. You get the worst of both worlds. If you find yourself doing that, either embrace embedding or use Postgres. (More in §15.)

### 1.4a The same data, relational vs document **[B]**

Concretely, here is a blog modeled both ways so the trade-off is tangible. In SQL you normalize into three tables and JOIN:

```sql
-- Relational: 3 tables, foreign keys, reassemble with JOINs at read time.
CREATE TABLE users  (id SERIAL PRIMARY KEY, name TEXT, email TEXT UNIQUE);
CREATE TABLE posts  (id SERIAL PRIMARY KEY, author_id INT REFERENCES users(id), title TEXT);
CREATE TABLE tags   (post_id INT REFERENCES posts(id), tag TEXT);
-- "Get a post with author name and tags" => a 3-way JOIN every read.
SELECT p.title, u.name, t.tag FROM posts p
  JOIN users u ON u.id = p.author_id
  LEFT JOIN tags t ON t.post_id = p.id WHERE p.id = 42;
```

In MongoDB you embed what is read together (tags belong to the post; the author's display name is duplicated via an extended reference) and reference what is shared/unbounded (the full author record, comments):

```js
// Document: one read returns the whole post view — no joins for the common case.
{
  _id: ObjectId("..."),
  title: "Modeling in MongoDB",
  tags: ["mongodb", "design"],               // embedded: small, bounded, read together
  author: { _id: ObjectId("a1"), name: "Ada" }, // extended reference (§4.5): id + display name
  body: "...",
  createdAt: ISODate("2026-06-21T10:00:00Z")
}
// Comments live in their own collection (unbounded) and reference the post id.
```

Notice what changed: the relational version stores each fact once and pays at read time (JOINs); the document version pre-joins for the hot path and pays a little at write time (duplicating the author's name). Neither is "right" — they optimize different things. The whole craft of document design (§3–4) is deciding, per relationship, which trade to make.

### 1.4b BSON data types reference **[B]**

Knowing the types prevents the classic bugs (money as `double`, dates as strings, id type mismatch).

| Type | mongosh constructor | Use for | Notes |
|---|---|---|---|
| String | `"text"` | text | UTF-8 |
| Double | `3.14` | floats, measurements | default for JS numbers — **not** money |
| Int (32-bit) | `NumberInt(5)` | counts, small ints | |
| Long (64-bit) | `NumberLong("9e9")` | large ints, ids | |
| Decimal128 | `NumberDecimal("9.99")` | **money**, exact decimals | no float rounding |
| Boolean | `true` / `false` | flags | |
| Date | `new Date()` / `ISODate(...)` | timestamps | ms since epoch; supports range/date math |
| ObjectId | `ObjectId()` | default `_id`, references | 12 bytes, time-ordered, binary not string |
| Array | `[1, 2, 3]` | lists | multikey-indexable |
| Object | `{ a: 1 }` | nested sub-documents | dot-notation queryable |
| Null | `null` | absence | distinct from "field missing" (use `$exists`) |
| BinData | `BinData(...)` | binary blobs, UUIDs | |
| Regex | `/pattern/i` | stored patterns | |
| Timestamp | `Timestamp()` | internal (oplog) | rarely user-facing |

### 1.5 Install & run with Docker **[B]**

The cleanest cross-platform way to get a MongoDB locally is Docker. (On Windows, install Docker Desktop.)

```bash
# Pull and run a single MongoDB 8 server, mapping the default port 27017,
# persisting data in a named Docker volume so it survives container restarts.
docker run -d --name mongo8 \
  -p 27017:27017 \
  -v mongodata:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret \
  mongo:8

# Open the shell INSIDE the container (mongosh ships with the image):
docker exec -it mongo8 mongosh -u admin -p secret --authenticationDatabase admin
```

For transactions and change streams you need a **replica set**, not a standalone (transactions require an oplog). A quick single-node replica set for local dev:

```bash
docker run -d --name mongo-rs -p 27017:27017 -v mongors:/data/db \
  mongo:8 --replSet rs0          # start mongod with a replica-set name

# Initialize the replica set once (the host MUST be reachable under this name):
docker exec -it mongo-rs mongosh --eval 'rs.initiate({
  _id: "rs0",
  members: [{ _id: 0, host: "localhost:27017" }]
})'
```

A `docker-compose.yml` for a reproducible local stack:

```yaml
services:
  mongo:
    image: mongo:8
    command: ["--replSet", "rs0", "--bind_ip_all"]
    ports: ["27017:27017"]
    volumes: ["mongodata:/data/db"]
volumes:
  mongodata:
```

> **⚡ Version note (2026):** **MongoDB Atlas** (the managed cloud service) is where Vector Search and Atlas Search live — those features are *not* in the self-hosted Community Server in the same form. For local AI/search work many devs use the **Atlas Local** Docker image (`mongodb/mongodb-atlas-local`) which bundles the search/vector engine, or develop against a free Atlas cluster. Plain `mongod` covers everything else in this guide.

### 1.5a Importing & exporting data **[B]**

The MongoDB Database Tools (`mongoimport`, `mongoexport`, `mongodump`, `mongorestore`) move data in and out. Use them to load sample datasets while learning.

```bash
# Import a JSON array file into a collection (each array element becomes a document):
mongoimport --uri "mongodb://localhost:27017" -d shop -c products --jsonArray --file products.json
# Import a CSV with a header row:
mongoimport --uri "mongodb://localhost:27017" -d shop -c products --type csv --headerline --file products.csv
# Export a collection to JSON:
mongoexport --uri "mongodb://localhost:27017" -d shop -c products --out products.json
# Binary backup / restore of a whole database (preserves BSON types & indexes):
mongodump   --uri "mongodb://localhost:27017" -d shop --out ./backup
mongorestore --uri "mongodb://localhost:27017" ./backup
```

> `mongoimport` reads **Extended JSON**, so `{"$oid": "..."}` / `{"$date": "..."}` round-trip as real `ObjectId`/`Date`. Plain CSV imports everything as strings unless you specify field types (`--columnsHaveTypes`).

### 1.6 The mongosh shell **[B]**

`mongosh` is the modern MongoDB shell — a full JavaScript REPL wired to a server. You issue commands against the *currently selected database* held in the global `db` variable.

```js
// Connect (from your host machine if mongosh is installed locally):
//   mongosh "mongodb://admin:secret@localhost:27017/?authSource=admin"

show dbs                 // list databases (only those with data show up)
use shop                 // switch to (or lazily create) the `shop` database
db                       // prints "shop" — the current database
show collections         // list collections in the current db

// Collections and databases are created LAZILY — the first insert materializes them.
db.products.insertOne({ name: "Keyboard", price: NumberDecimal("49.99") })

db.products.find()       // query everything (returns a cursor, prints first batch)
db.products.countDocuments()        // count matching docs (accurate)
db.products.estimatedDocumentCount() // fast count from metadata (approximate)

db.products.drop()       // delete the whole collection
db.dropDatabase()        // delete the current database
```

Useful shell facts:
- The shell auto-prints the first ~20 results of a cursor; type `it` to get the next batch.
- It's real JavaScript: `for`, `const`, functions, `Array.map`, `printjson()`, etc. all work.
- `db.collection.help()` and tab-completion are your friends.

---

## 2. CRUD in the Shell (mongosh)

CRUD = Create, Read, Update, Delete. Learn it in the shell first; every driver (§11, §12) mirrors these exact methods and operators, so this knowledge transfers directly. Throughout, remember that **a single-document write is always atomic** in MongoDB — no other client sees a half-written document.

### 2.1 Create — insert **[B]**

```js
// insertOne: insert a single document. Returns the inserted _id.
db.users.insertOne({
  name: "Ada Lovelace",
  email: "ada@example.com",
  age: 36,
  roles: ["admin", "author"],        // arrays are first-class
  address: { city: "London", zip: "EC1" },  // nested sub-documents are first-class
  createdAt: new Date()              // store a real BSON Date, not a string
})
// => { acknowledged: true, insertedId: ObjectId("...") }

// insertMany: bulk insert. Faster than many insertOne calls (one round trip).
db.users.insertMany([
  { name: "Alan Turing", email: "alan@example.com", age: 41, roles: ["author"] },
  { name: "Grace Hopper", email: "grace@example.com", age: 52, roles: ["admin"] }
])
// By default insertMany is ORDERED: it stops at the first error. Pass
// { ordered: false } to attempt all and report errors at the end (useful for bulk imports).
db.users.insertMany([ /* ... */ ], { ordered: false })
```

Notice the documents have different fields — that's allowed. The discipline of consistency is yours (and validation's, §7).

### 2.2 Read — find & query operators **[B]**

`find(filter, projection)` returns a **cursor** over all matching documents; `findOne(filter, projection)` returns the **first match as a single document** (or `null`). Use `findOne` when you expect at most one (e.g. by `_id` or email).

The **query filter** is itself a document. An empty filter `{}` matches everything. Field equality is written plainly; everything else uses **`$`-prefixed query operators** inside a sub-object.

```js
// Equality (these two are equivalent):
db.users.find({ age: 36 })
db.users.find({ age: { $eq: 36 } })

// Comparison operators:
db.users.find({ age: { $gt: 40 } })          // greater than
db.users.find({ age: { $gte: 40, $lte: 60 } })// 40..60 inclusive (range)
db.users.find({ age: { $ne: 36 } })          // not equal
db.users.find({ age: { $in: [36, 41, 52] } }) // matches any in the list
db.users.find({ age: { $nin: [36] } })        // matches none in the list

// Logical operators combine conditions:
db.users.find({ $and: [ { age: { $gt: 30 } }, { age: { $lt: 50 } } ] })
db.users.find({ $or:  [ { age: { $lt: 40 } }, { roles: "admin" } ] })
db.users.find({ age: { $not: { $gt: 40 } } }) // negation
// NOTE: multiple fields in one filter object are an implicit AND:
db.users.find({ age: { $gt: 30 }, roles: "author" })  // age>30 AND has role author

// Existence and type:
db.users.find({ middleName: { $exists: false } })   // field absent
db.users.find({ age: { $type: "int" } })            // field has BSON type int

// Pattern matching (regex). Anchor and use ^ to allow an index to help:
db.users.find({ email: { $regex: /@example\.com$/i } })  // case-insensitive suffix
db.users.find({ name: { $regex: /^Ada/ } })              // prefix (index-friendly)

// Querying ARRAYS:
db.users.find({ roles: "admin" })           // array CONTAINS "admin"
db.users.find({ roles: { $all: ["admin", "author"] } })  // contains ALL listed
db.users.find({ roles: { $size: 2 } })       // array length is exactly 2
// $elemMatch: a SINGLE array element must match ALL the given conditions
db.scores.find({ grades: { $elemMatch: { $gte: 80, $lt: 90 } } })

// Querying NESTED fields with dot notation (quote the key):
db.users.find({ "address.city": "London" })
```

**Projection** is the second argument: choose which fields to return. `1` = include, `0` = exclude. You can't mix include and exclude (except for `_id`, which you can always turn off).

```js
db.users.find({ age: { $gt: 30 } }, { name: 1, email: 1 })        // name, email, _id
db.users.find({ age: { $gt: 30 } }, { name: 1, _id: 0 })          // name only, no _id
db.users.find({}, { roles: 0 })                                    // everything except roles
```

**Cursor modifiers** shape the result set. Order matters conceptually: filter → sort → skip → limit.

```js
db.users.find().sort({ age: -1 })       // -1 descending, 1 ascending
db.users.find().sort({ age: -1 }).limit(10)        // top 10 oldest
db.users.find().sort({ age: 1 }).skip(20).limit(10) // page 3 (offset pagination)
db.users.find().count()                 // deprecated on cursor; prefer countDocuments(filter)
```

> **Gotcha — `skip` is slow for deep pages.** `skip(100000)` makes the server walk and discard 100k docs. For large datasets use **range (keyset) pagination** instead: remember the last `_id`/sort value and query `{ _id: { $gt: lastId } }`. It uses the index and stays fast no matter how deep you page.

### 2.3 Update — update operators **[B/I]**

`updateOne(filter, update)` modifies the **first** matching document; `updateMany` modifies **all** matches. The `update` argument is *not* a replacement document — it's a set of **update operators** describing *changes*. (Passing a plain document to `updateOne` would be an error in modern drivers; use `replaceOne` for full replacement.)

```js
// $set: set/overwrite fields (creates them if absent). The workhorse.
db.users.updateOne(
  { email: "ada@example.com" },
  { $set: { age: 37, "address.city": "Oxford" } }  // dot notation edits nested fields
)

// $unset: remove a field entirely.
db.users.updateOne({ email: "ada@example.com" }, { $unset: { middleName: "" } })

// $inc: atomically increment (or decrement with a negative). Great for counters.
db.posts.updateOne({ _id: postId }, { $inc: { views: 1, likes: 5 } })

// $mul, $min, $max, $rename, $currentDate:
db.products.updateOne({ _id: id }, { $mul: { price: 1.1 } })          // +10%
db.products.updateOne({ _id: id }, { $min: { lowestPrice: 9.99 } })   // only if lower
db.users.updateOne({ _id: id }, { $rename: { "phone": "mobile" } })
db.users.updateOne({ _id: id }, { $currentDate: { updatedAt: true } })// server time

// ARRAY update operators — crucial:
db.users.updateOne({ _id: id }, { $push: { roles: "editor" } })   // append (allows dups)
db.users.updateOne({ _id: id }, { $addToSet: { roles: "editor" } })// append only if absent (set semantics)
db.users.updateOne({ _id: id }, { $pull: { roles: "editor" } })   // remove all matching
db.users.updateOne({ _id: id }, { $pop: { roles: 1 } })           // remove last (-1 = first)
db.users.updateOne({ _id: id }, { $push: { roles: { $each: ["a", "b"] } } }) // push many

// $push with modifiers — keep an array sorted and capped (e.g. recent activity):
db.users.updateOne({ _id: id }, {
  $push: { recent: { $each: [newEvent], $sort: { at: -1 }, $slice: 20 } }
}) // append, re-sort by time desc, keep only newest 20 — bounds the array!

// Positional operators update a matched array element:
db.orders.updateOne(
  { _id: id, "items.sku": "ABC" },
  { $set: { "items.$.qty": 3 } }            // $ = the FIRST matched element
)
db.orders.updateMany(
  { },
  { $inc: { "items.$[elem].qty": 1 } },
  { arrayFilters: [ { "elem.qty": { $lt: 5 } } ] }  // $[<id>] = all elements matching the filter
)
```

**Upsert** = "update if it exists, otherwise insert." Pass `{ upsert: true }`. The filter fields plus the update are combined to build the new document when nothing matches. This is the idiomatic way to do "insert-or-merge."

```js
db.counters.updateOne(
  { _id: "pageviews" },
  { $inc: { value: 1 } },
  { upsert: true }      // first call creates { _id: "pageviews", value: 1 }, then increments
)
```

`replaceOne` swaps the whole document (keeping `_id`):

```js
db.users.replaceOne({ _id: id }, { name: "New", email: "new@x.com" }) // old fields gone
```

`findOneAndUpdate` updates and **returns the document** (before or after, your choice) in one atomic step — perfect for "claim a job" / "generate next sequence" patterns.

```js
db.jobs.findOneAndUpdate(
  { status: "pending" },
  { $set: { status: "running", startedAt: new Date() } },
  { sort: { priority: -1 }, returnDocument: "after" }  // "before" | "after"
)
```

### 2.4 Delete **[B]**

```js
db.users.deleteOne({ email: "ada@example.com" })   // delete first match
db.users.deleteMany({ age: { $lt: 18 } })          // delete all matches
db.users.deleteMany({})                            // delete EVERYTHING (careful!)
db.users.findOneAndDelete({ _id: id })             // delete and return the doc
```

### 2.5 Bulk writes **[I]**

For mixed batches (insert + update + delete in one round trip) use `bulkWrite`. Unordered bulk ops run in parallel on the server and are far faster than per-document calls.

```js
db.users.bulkWrite([
  { insertOne: { document: { name: "X" } } },
  { updateOne: { filter: { name: "X" }, update: { $set: { active: true } } } },
  { deleteOne: { filter: { name: "Y" } } }
], { ordered: false })
```

### 2.5a CRUD method reference **[B]**

| Method | Returns | Use when |
|---|---|---|
| `insertOne(doc)` | inserted id | adding one document |
| `insertMany([docs])` | inserted ids | bulk add (one round trip) |
| `findOne(filter, proj)` | one doc or null | expect at most one (by id/email) |
| `find(filter, proj)` | a cursor | many results; chain `.sort/.limit/.skip` |
| `countDocuments(filter)` | exact count | accurate counts |
| `estimatedDocumentCount()` | fast count | approximate total from metadata |
| `distinct(field, filter)` | array of values | distinct values of a field |
| `updateOne(f, update)` | match/modified counts | change first match (operators) |
| `updateMany(f, update)` | match/modified counts | change all matches |
| `replaceOne(f, doc)` | counts | swap a whole document |
| `findOneAndUpdate(f, u, opts)` | the doc (before/after) | atomic claim/return-document |
| `findOneAndDelete(f)` | the deleted doc | delete and read in one step |
| `deleteOne(f)` / `deleteMany(f)` | deleted count | remove documents |
| `bulkWrite([ops])` | aggregate result | mixed batch in one round trip |
| `aggregate([stages])` | a cursor | grouping/joins/reports (§6) |

### 2.5b Subtle query semantics worth memorizing **[I]**

These trip up nearly everyone at least once:

```js
// 1) null matches BOTH "field is null" AND "field is missing". Use $exists to distinguish.
db.users.find({ middleName: null })                          // null OR absent
db.users.find({ middleName: { $exists: true, $eq: null } })  // present AND null

// 2) Equality on an array field matches if the array CONTAINS the value (not equals it):
db.users.find({ roles: "admin" })          // any doc whose roles array includes "admin"
db.users.find({ roles: ["admin"] })        // EXACT array match: roles is exactly ["admin"]

// 3) Multiple conditions on an array WITHOUT $elemMatch can match across DIFFERENT elements:
db.scores.find({ grades: { $gte: 80, $lt: 90 } })
//   ^ matches if SOME element >= 80 AND SOME (possibly other) element < 90.
db.scores.find({ grades: { $elemMatch: { $gte: 80, $lt: 90 } } })
//   ^ matches only if a SINGLE element is in [80, 90). Usually what you actually want.

// 4) Dot notation queries into arrays of sub-documents by ANY element:
db.orders.find({ "items.sku": "ABC" })     // any line item with sku "ABC"

// 5) Comparisons are TYPE-aware: a number won't match a numeric STRING.
db.t.find({ age: 30 })       // does NOT match documents where age is the string "30"

// 6) Sort order across mixed types follows BSON type ordering — keep fields one type.
```

### 2.6 Query operator reference **[B/I]**

| Operator | Meaning | Example |
|---|---|---|
| `$eq` `$ne` | equal / not equal | `{ age: { $ne: 30 } }` |
| `$gt` `$gte` `$lt` `$lte` | comparisons | `{ age: { $gte: 18 } }` |
| `$in` `$nin` | in / not in list | `{ status: { $in: ["a","b"] } }` |
| `$and` `$or` `$nor` `$not` | logical | `{ $or: [ {a:1}, {b:2} ] }` |
| `$exists` | field present? | `{ phone: { $exists: true } }` |
| `$type` | BSON type check | `{ x: { $type: "string" } }` |
| `$regex` | pattern match | `{ name: { $regex: /^A/ } }` |
| `$all` | array contains all | `{ tags: { $all: ["a","b"] } }` |
| `$elemMatch` | one array element matches all conditions | see above |
| `$size` | exact array length | `{ tags: { $size: 3 } }` |
| `$mod` | modulo | `{ n: { $mod: [4, 0] } }` |
| `$text` | full-text (needs text index) | `{ $text: { $search: "coffee" } }` |
| `$expr` | use aggregation expressions in a query | `{ $expr: { $gt: ["$spent", "$budget"] } }` |

---

## 3. Data Modeling — Embedding vs Referencing

This is the heart of document database design, and the decision you will make over and over. There is no `JOIN`-by-default; **you choose, per relationship, whether related data lives *inside* a document (embedding) or in a separate collection linked by id (referencing)**. Get this right and reads are one fast operation; get it wrong and you fight the database forever.

### 3.1 The two tools **[B]**

**Embedding** = nest the related data directly inside the parent document.

```js
// An order with its line items embedded — one document, one read:
{
  _id: ObjectId("..."),
  customer: "ada@example.com",
  status: "shipped",
  items: [                                  // embedded array of sub-documents
    { sku: "KB-1", name: "Keyboard", qty: 1, price: NumberDecimal("49.99") },
    { sku: "MS-2", name: "Mouse",    qty: 2, price: NumberDecimal("19.99") }
  ],
  shippingAddress: { city: "London", zip: "EC1" }  // embedded sub-document
}
```

**Referencing** = store the related data in another collection and keep a pointer (its `_id`).

```js
// authors collection:
{ _id: ObjectId("a1"), name: "Ada", bio: "..." }
// books collection — references the author by id:
{ _id: ObjectId("b1"), title: "Notes", authorId: ObjectId("a1") }
// To get the author you query again, or use $lookup (§6/§9).
```

### 3.2 The core decision rules **[B/I]**

Ask these questions about the relationship. They almost always point to the right answer.

**Lean toward EMBEDDING when:**
- The data is **"contained by"** or **"part of"** the parent (a "has-a" / composition relationship): an order *has* line items, a blog post *has* comments early on, a user *has* an address.
- You **read them together** the vast majority of the time. Embedding turns N reads into 1.
- It's a **one-to-one** or **one-to-few** relationship (a handful, not unbounded).
- The embedded data **doesn't need to be queried or updated independently** very often, and isn't shared by other parents.

The payoff: **a single read returns everything, atomically.** No joins, no second query, and updates to the whole object are atomic because it's one document.

**Lean toward REFERENCING when:**
- The relationship is **one-to-many (large)** or **many-to-many**. Comments on a viral post could be millions — you cannot embed them all (16 MB limit, §3.4).
- The set is **unbounded / grows without limit** (audit logs, time-series events, messages in a channel). Unbounded embedded arrays are an anti-pattern (§15).
- The related entity is **shared** across many parents (a `category` referenced by thousands of products; a `user` who authored many posts). Duplicating shared data everywhere makes updates a nightmare.
- The related entity is **large** and you usually *don't* need it when reading the parent (don't drag a 2 MB profile into every order).
- The related entity has its **own lifecycle** and is queried on its own ("show me all comments by this user across all posts").

A compact rule of thumb that captures most cases:

> **Embed for "one-to-few that you read together." Reference for "one-to-many/unbounded/shared/queried-independently."** When in doubt about array growth, reference. When in doubt about read locality, embed.

### 3.3 Modeling the three cardinalities **[I]**

**One-to-one.** Almost always **embed** — that's literally what sub-documents are for. A user and their profile/settings:

```js
{ _id: u1, name: "Ada", profile: { bio: "...", avatarUrl: "...", theme: "dark" } }
```
Split into a referenced doc only if the one side is huge, rarely needed, or accessed separately.

**One-to-many.** This is where judgement lives.
- **One-to-few (bounded, read together):** embed an array.
  ```js
  { _id: u1, name: "Ada", addresses: [ { label: "home", ... }, { label: "work", ... } ] }
  ```
- **One-to-many (large/unbounded):** reference from the "many" side (the child holds the parent id). This is the relational instinct and it's correct here.
  ```js
  // posts then comments, comment points back to post:
  { _id: c1, postId: p1, body: "Nice!", author: "alan", at: ISODate("...") }
  ```
- **Hybrid (the subset pattern, §4.4):** embed a *bounded recent slice* (last 10 comments) for fast display, reference the full set for the "see all" page. Best of both.

**Many-to-many.** Two common shapes:
- **Array of references on one (or both) side(s)** — good when one side's list is bounded. Students and courses:
  ```js
  { _id: s1, name: "Ada", courseIds: [ c1, c2, c3 ] }   // student references courses
  ```
  Putting the array on the side with the smaller, more bounded list avoids unbounded arrays.
- **A junction/linking collection** when the relationship itself has attributes (enrolment date, grade) or both sides are unbounded — exactly like a relational join table:
  ```js
  { _id: e1, studentId: s1, courseId: c2, enrolledAt: ISODate("..."), grade: "A" }
  ```

### 3.4 The 16 MB document limit — and why it shapes everything **[B/I]**

A single BSON document may not exceed **16 MB**. This isn't an arbitrary annoyance; it's a guardrail that forces good design. It means **no embedded array can grow without bound.** If "the number of embedded things" has no ceiling (comments, log lines, chat messages, sensor readings), you *will* eventually blow the limit — and long before that you'll suffer, because:

- The **whole document is rewritten** on most updates; a 10 MB document is slow to update and moves around on disk.
- Every read pulls the whole document over the wire even if you only wanted one field (though projection helps server-side, the working set still bloats memory).
- WiredTiger caches whole documents; giant documents wreck your cache hit rate.

So the limit operationalizes the rule: **bounded → embed; unbounded → reference (or bucket, §4.1).** For genuinely large binary blobs (>16 MB files) MongoDB offers **GridFS**, which splits a file into chunks across two collections — but for most apps, store files in object storage (S3/R2) and keep just the URL in the document.

### 3.5 A modeling decision walkthrough **[I]**

Let's model an **e-commerce order system** end to end, applying the rules, so you see the reasoning rather than just the answer.

**Entities:** customers, products, orders (with line items), reviews.

1. **Order → line items.** A line item is *part of* an order, the count is small and bounded (you don't order 50,000 distinct SKUs), and you always read items with the order. → **Embed** the items array in the order. One read renders the whole order; updates to the order are atomic.

2. **Line item → product.** A product is **shared** by many orders and has its own lifecycle (price changes, restock). You must *not* embed the live product. But you also don't want a `$lookup` to show the order — and crucially, the order should record the price **as it was at purchase time** (prices change!). → **Extended reference**: embed `productId` + a snapshot of `name`, `price`, `qty` in the line item. The snapshot is intentionally frozen; the full product is referenced.

3. **Order → customer.** A customer is shared and has many orders. → **Reference** by `customerId`, plus an **extended reference** (name, city) so the order list renders without a join.

4. **Product → reviews.** Reviews are **unbounded** (a popular product has thousands) and queried independently ("reviews by this user"). → **Reference** reviews to the product (review holds `productId`). Apply the **subset pattern**: embed the 3 most recent reviews on the product for the product page, keep `reviewCount` + `averageRating` (**computed pattern**) for display.

The resulting order document:

```js
{
  _id: ObjectId("..."),
  status: "shipped",
  customer: { _id: ObjectId("c1"), name: "Ada Lovelace", city: "London" }, // extended ref
  items: [                                                    // embedded, bounded
    { productId: ObjectId("p1"), name: "Keyboard", qty: 1, price: NumberDecimal("49.99") },
    { productId: ObjectId("p2"), name: "Mouse",    qty: 2, price: NumberDecimal("19.99") }
  ],
  total: NumberDecimal("89.97"),                              // computed pattern
  placedAt: ISODate("2026-06-21T10:00:00Z")
}
```

Every decision traces to a rule: contains + bounded + read-together → embed; shared/unbounded/independent → reference; "show a label without a join" → extended reference; "expensive to compute on read" → computed. That is the entire discipline.

---

## 4. Schema Design Patterns

Patterns are reusable, battle-tested document shapes that solve recurring modeling problems. Knowing them is what separates "I store JSON" from "I design schemas." Each below: what it is, when to use it, the trade-off, and an example.

### 4.1 The Bucket Pattern **[A]** — time-series / IoT

**What & why.** When you have a high-volume stream of small, time-stamped measurements (sensor readings, page views, stock ticks), storing **one document per reading** creates billions of tiny documents — huge index overhead and per-document storage overhead dominate. The bucket pattern **groups many readings into one "bucket" document** covering a time window (e.g. one document per device per hour), holding the readings in an array plus rollup metadata. You trade hundreds of documents for one.

**When.** Time-series, IoT, metrics, logs, anything appended over time and queried by time range. The array is **bounded** (cap it at, say, 200 readings or one hour) so you don't violate §3.4 — when full, start a new bucket.

**Trade-off.** Far fewer documents, smaller indexes, much better compression and read locality for time-range scans. Cost: queries for a single reading must dig into an array, and bucket sizing needs thought.

```js
// One bucket = one device, one hour. New readings $push into the open bucket via upsert.
{
  _id: ObjectId("..."),
  deviceId: "sensor-42",
  startTs: ISODate("2026-06-21T10:00:00Z"),  // bucket window start
  count: 3,                                   // number of readings (cap this)
  sumTemp: 64.2, minTemp: 21.0, maxTemp: 21.7,// precomputed rollups (computed pattern!)
  readings: [
    { ts: ISODate("2026-06-21T10:00:05Z"), temp: 21.0 },
    { ts: ISODate("2026-06-21T10:00:35Z"), temp: 21.5 },
    { ts: ISODate("2026-06-21T10:01:05Z"), temp: 21.7 }
  ]
}
```

```js
// Append a reading: find the open bucket for this device/hour (count < cap) or create one.
const hourStart = ISODate("2026-06-21T10:00:00Z");
db.readings.updateOne(
  { deviceId: "sensor-42", startTs: hourStart, count: { $lt: 200 } },
  {
    $push: { readings: { ts: new Date(), temp: 21.9 } },
    $inc:  { count: 1, sumTemp: 21.9 },
    $min:  { minTemp: 21.9 },
    $max:  { maxTemp: 21.9 },
    $setOnInsert: { deviceId: "sensor-42", startTs: hourStart }
  },
  { upsert: true }
)
```

> **⚡ Version note:** Since MongoDB 5.0, **native Time Series collections** implement bucketing *for you* — you `db.createCollection("readings", { timeseries: { timeField: "ts", metaField: "deviceId", granularity: "hours" } })` and insert one document per reading; the server buckets/compresses internally and adds `$densify`/`$fill` for gap handling. Prefer native time-series for new time-series workloads; understand the manual bucket pattern because (a) it's the model time-series collections use internally and (b) it still applies to non-time data you want to group.

### 4.2 The Computed Pattern **[I]**

**What & why.** Don't recompute expensive aggregates on every read — **compute once on write and store the result.** A product's `averageRating` and `reviewCount`, an order's `total`, a user's `followerCount`. Reads become trivial field lookups.

**When.** Read-heavy values derived from many writes, where reads vastly outnumber writes (the usual case). **Trade-off:** the stored value can drift if you forget to update it; you accept eventual consistency and a small write cost in exchange for cheap reads.

```js
{ _id: p1, name: "Mug", reviewCount: 128, ratingSum: 540, averageRating: 4.22 }
// On new review: $inc reviewCount and ratingSum, then recompute average
db.products.updateOne({ _id: p1 }, [
  { $set: { reviewCount: { $add: ["$reviewCount", 1] },
            ratingSum:   { $add: ["$ratingSum", 5] } } },
  { $set: { averageRating: { $divide: ["$ratingSum", "$reviewCount"] } } }
])  // an aggregation-pipeline update: later stages see earlier ones' results
```

### 4.3 The Outlier Pattern **[A]**

**What & why.** Most documents fit your model (a post has a few hundred comments) but **rare outliers** explode it (a celebrity's post has 2 million). Designing for the outlier (always reference) penalizes the 99%; designing for the common case (embed) breaks on the outlier. The outlier pattern **handles the common case inline and flags outliers to overflow elsewhere.**

**When.** Skewed distributions: viral content, mega-customers, super-nodes. **Trade-off:** application code must branch on the flag.

```js
{ _id: p1, title: "Normal post", comments: [ /* embedded, a few */ ] }
// A viral post: stop embedding, set a flag, push overflow into another collection.
{ _id: p2, title: "Viral post", commentCount: 2_000_000,
  comments: [ /* most recent few embedded */ ],
  hasExtras: true }   // app sees hasExtras -> reads `comments_overflow` collection too
```

### 4.4 The Subset Pattern **[I]**

**What & why.** You need *some* of a large related set on the main view (the 5 most recent reviews on a product page) but not all 10,000. **Embed the hot subset; reference the full set.** The product page does one read; "see all reviews" pages the `reviews` collection.

**When.** "Recent N" / "top N" UI alongside a full collection. **Trade-off:** the subset must be kept in sync (use `$push` with `$slice` to keep it capped, §2.3).

```js
{ _id: p1, name: "Mug",
  topReviews: [ /* 5 most recent, embedded for the product page */ ],
  reviewCount: 10342 }   // full reviews live in their own collection, queried on demand
```

### 4.5 The Extended Reference Pattern **[I]**

**What & why.** A pure reference forces a `$lookup`/second query to show even a label. The extended reference **duplicates the few fields you frequently display** alongside the id, so common reads need no join. An order embeds the customer's `_id` **and** their name + city — enough to render the order list without touching the customers collection.

**When.** You reference something but almost always also need a couple of its fields. **Trade-off:** denormalized copies can go stale — only duplicate **rarely-changing** fields, and have a plan (a background job or change stream) to refresh them when the source changes.

```js
{ _id: o1, total: NumberDecimal("69.98"),
  customer: { _id: c1, name: "Ada Lovelace", city: "London" }, // extended reference
  items: [ /* ... */ ] }
```

### 4.6 The Approximation Pattern **[A]**

**What & why.** When an exact value is expensive to keep perfectly accurate and "close enough" is fine, **update it approximately** to save writes. A page-view counter that increments by 1 only 1-in-100 times (then jumps by ~100), or a "trending score" recomputed periodically rather than per event. You trade precision for a massive reduction in write load.

**When.** High-volume counters/metrics where exactness doesn't matter (view counts, "people are looking at this"). **Trade-off:** displayed numbers are estimates.

```js
// Increment by 100 only ~1% of the time (probabilistic counting):
if (Math.random() < 0.01) {
  db.pages.updateOne({ _id: id }, { $inc: { views: 100 } }) // 100x fewer writes
}
```

### 4.7 The Schema Versioning Pattern **[I/A]**

**What & why.** Documents written over years have different shapes. Instead of a risky, all-at-once migration of a billion documents, **stamp each document with a `schemaVersion`** and let the application handle each version (and lazily upgrade on write). New code reads old docs correctly; old docs upgrade as they're touched.

**When.** Long-lived collections with evolving schemas. **Trade-off:** app code carries version-handling logic until you finish a background backfill.

```js
{ _id: u1, schemaVersion: 1, name: "Ada Lovelace" }          // old: single name field
{ _id: u2, schemaVersion: 2, firstName: "Grace", lastName: "Hopper" } // new shape
// App: if (doc.schemaVersion < 2) { split name -> firstName/lastName; optionally re-save }
```

### 4.8 The Polymorphic Pattern **[I]**

**What & why.** Several related-but-different entity types share a collection because you query them together (all "products" regardless of category; a feed of mixed "events"). Each document carries a **discriminator field** (`type`/`kind`) and the fields appropriate to its type. This is single-table-inheritance, done naturally because Mongo doesn't require uniform fields.

**When.** Heterogeneous items listed/queried together. **Trade-off:** application/validation must branch on the type; indexes should cover the shared query fields.

```js
{ _id: a1, type: "book",  title: "Notes", author: "Ada", pages: 320 }
{ _id: a2, type: "movie", title: "Hidden Figures", director: "Melfi", runtime: 127 }
// Query all products: db.products.find({ price: { $lt: 50 } })
// Branch in app on `type` to render the right fields.
```

### 4.9 Tree & Hierarchy Patterns **[A]**

Hierarchies (categories, org charts, comment threads, file systems) are common and have **four** classic modeling strategies, each optimizing a different operation. Choosing depends on whether you read up (ancestors), read down (descendants), or move subtrees often.

**(a) Parent references** — each node stores its parent's id. Simplest. Finding *direct* children is easy; finding *all* descendants requires recursion (`$graphLookup`, §6).
```js
{ _id: "electronics", parent: null }
{ _id: "laptops", parent: "electronics" }
{ _id: "gaming-laptops", parent: "laptops" }
```

**(b) Child references** — each node stores an array of its children's ids. Reading direct children is one read; finding the parent or full ancestry is awkward.
```js
{ _id: "electronics", children: ["laptops", "phones"] }
```

**(c) Array of ancestors** — each node stores its parent **and** an ordered array of all ancestors. Finding the whole subtree or the full path is a single indexed query. Costs more on writes/moves.
```js
{ _id: "gaming-laptops", parent: "laptops",
  ancestors: ["electronics", "laptops"] }   // find subtree: { ancestors: "electronics" }
```

**(d) Materialized paths** — store the full path as a delimited string; query subtrees with an anchored regex (which can use an index).
```js
{ _id: "gaming-laptops", path: ",electronics,laptops," }
// All descendants of electronics: db.cats.find({ path: { $regex: /^,electronics,/ } })
```

| Hierarchy pattern | Find children | Find descendants | Find ancestors | Move subtree |
|---|---|---|---|---|
| Parent refs | easy | recursive (`$graphLookup`) | recursive | easy |
| Child refs | easy | recursive | hard | medium |
| Array of ancestors | query | one query | one query | update many |
| Materialized paths | regex | anchored regex | parse string | update many paths |

**Rule of thumb:** read-heavy hierarchies that rarely move → array of ancestors or materialized paths (great reads). Frequently restructured trees → parent references + `$graphLookup`.

---

## 5. Indexing

### 5.1 Why indexes exist **[I]**

Without an index, a query is a **collection scan (`COLLSCAN`)**: the server reads every document and tests your filter — O(n), catastrophic at scale. An **index** is an ordered B-tree of selected field values plus pointers to the documents, so the server can jump straight to matches and even return results pre-sorted. The trade-off is universal: **indexes speed up reads but slow down writes** (every insert/update must also update each affected index) and **consume RAM and disk** (you want your indexes to fit in the WiredTiger cache). So you index deliberately, driven by your actual query patterns — not "just in case."

Every collection has an automatic unique index on `_id`. You add the rest.

```js
db.users.createIndex({ email: 1 })                 // 1 = ascending, -1 = descending
db.users.getIndexes()                              // list indexes on a collection
db.users.dropIndex("email_1")                      // remove by name
```

### 5.2 Single-field & compound indexes; the ESR rule **[I/A]**

A **compound index** covers multiple fields in a defined order, and **field order is everything.** A compound index `{ a: 1, b: 1, c: 1 }` can serve queries filtering on `a`, on `a+b`, and on `a+b+c` — the **prefix rule** (it can use any left-prefix) — but *not* a query that filters only on `b` or only on `c`.

To order the fields, use the **ESR rule — Equality, Sort, Range**:

1. **Equality** fields first (fields tested with `$eq`/exact match). These narrow the search the most and pin a single point in the index.
2. **Sort** fields next (fields in your `.sort()`). Placing them here lets the index return results already ordered — no expensive in-memory sort.
3. **Range** fields last (`$gt`, `$lt`, `$in`, `$regex`). Ranges scan a span of the index, so anything after them can't be used for further selection or sorting.

```js
// Query: find active users in a city, newest first, registered after a date.
//   filter: { status: "active", city: "London", createdAt: { $gt: someDate } }
//   sort:   { createdAt: -1 }
// status & city are EQUALITY, createdAt(sort) is SORT, createdAt(range) is RANGE.
// Here sort and range share a field, so:
db.users.createIndex({ status: 1, city: 1, createdAt: -1 })  // E, E, S/R
// The index satisfies the equality match, returns rows pre-sorted by createdAt desc,
// and bounds the range — all from the index, no in-memory sort, no COLLSCAN.
```

> **Gotcha:** if you put a **range** field before a **sort** field, the index can't provide the sort and the server does an in-memory `SORT` stage — which **fails outright if it exceeds 100 MB** unless you allow disk use. ESR exists precisely to avoid this.

### 5.3 Multikey, text, geospatial, hashed **[I/A]**

**Multikey** — an index on an array field. MongoDB automatically makes it "multikey," indexing **each element**. This is how `{ tags: "mongodb" }` is fast. You don't ask for it; it happens. Caveat: you can't have a compound index with **two** array fields (the combinatorial explosion is disallowed).

```js
db.posts.createIndex({ tags: 1 })   // tags is an array -> multikey, indexes every tag
```

**Text** — tokenized full-text search across string fields (stemming, stop words). One text index per collection (it can cover multiple fields). Query with `$text`/`$search`.

```js
db.posts.createIndex({ title: "text", body: "text" })
db.posts.find({ $text: { $search: "coffee shop" } },
              { score: { $meta: "textScore" } }).sort({ score: { $meta: "textScore" } })
```
> For serious search (typo tolerance, relevance, facets) use **Atlas Search** (§14), not the built-in text index.

**Geospatial** — `2dsphere` indexes GeoJSON points/shapes for "near me" and "within polygon" queries on a sphere (Earth).

```js
db.places.createIndex({ location: "2dsphere" })
db.places.find({ location: { $near: {
  $geometry: { type: "Point", coordinates: [-0.12, 51.50] }, // [lng, lat] order!
  $maxDistance: 1000 }}})    // within 1 km
```

**Hashed** — hashes the field value; used for hash-based **sharding** to spread writes evenly (§10). Supports equality, not range.

```js
db.events.createIndex({ userId: "hashed" })
```

### 5.4 TTL, partial, unique, sparse, wildcard **[I]**

**TTL (time-to-live)** — auto-deletes documents a set number of seconds after a date field. Perfect for sessions, OTPs, ephemeral caches, logs with retention. A background task sweeps roughly once a minute.

```js
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 }) // delete after 1h
```

**Unique** — enforces no two documents share the value. The DB *does* enforce this constraint (one of the few it does).

```js
db.users.createIndex({ email: 1 }, { unique: true })
```

**Partial** — index only documents matching a filter. Smaller, cheaper index when you only query a subset (e.g. only active users), and the way to do "unique among non-null" correctly.

```js
db.users.createIndex({ email: 1 },
  { unique: true, partialFilterExpression: { email: { $exists: true } } })
db.orders.createIndex({ shippedAt: 1 },
  { partialFilterExpression: { status: "shipped" } })   // index only shipped orders
```

**Sparse** — index only documents where the field exists (older, blunter than partial; prefer partial).

**Wildcard** (`{ "$**": 1 }`) — index *all* fields, including unknown ones. Useful for highly polymorphic/dynamic documents where you can't predict query fields. Costly; don't reach for it casually.

### 5.5 explain() — reading the query plan **[I/A]**

`explain()` tells you *how* the server will run a query — whether it used an index and how much work it did. This is your primary tuning tool.

```js
db.users.find({ email: "ada@example.com" }).explain("executionStats")
```

Key things to read in the output:
- **`winningPlan.stage`**: `IXSCAN` (index scan — good) vs `COLLSCAN` (full scan — bad for selective queries). A `FETCH` after `IXSCAN` is normal (fetch docs the index pointed to). A standalone `SORT` stage means no index served the sort (consider ESR).
- **`executionStats.totalDocsExamined`** vs **`nReturned`**: the ratio should be close to 1. Examining 100,000 docs to return 10 means a missing/poor index.
- **`totalKeysExamined`**: index entries scanned. Lower is better.

A **covered query** is the ideal: the index contains every field the query needs (filter + projection), so the server answers entirely from the index with **zero document fetches** (`totalDocsExamined: 0`). Achieve it by projecting only indexed fields and excluding `_id` if it's not in the index.

### 5.6 Index types — quick reference **[I]**

| Index type | Created with | Solves | Watch out for |
|---|---|---|---|
| Single field | `{ f: 1 }` | equality/range/sort on one field | — |
| Compound | `{ a: 1, b: -1 }` | multi-field filters & sorts | field order = ESR; left-prefix rule |
| Multikey | `{ arr: 1 }` (array field) | querying inside arrays | no two-array compound index |
| Text | `{ f: "text" }` | basic full-text | one per collection; prefer Atlas Search |
| 2dsphere | `{ loc: "2dsphere" }` | geo near/within | GeoJSON `[lng, lat]` order |
| Hashed | `{ f: "hashed" }` | even sharding | equality only, no range |
| TTL | `{ t: 1 }, {expireAfterSeconds}` | auto-expiry | date field only; ~60s sweep |
| Unique | `{ f: 1 }, {unique:true}` | enforce uniqueness | nulls count; combine with partial |
| Partial | `{ f: 1 }, {partialFilterExpression}` | index a subset | query must match the filter to use it |
| Wildcard | `{ "$**": 1 }` | unknown/dynamic fields | large; last resort |

### 5.7 How indexes affect writes — the honest cost **[I/A]**

Every index is a second data structure the server must keep consistent. On **insert**, every index gets a new entry. On **update**, only indexes whose fields changed are touched — but if you update an indexed array, several entries may change. On **delete**, every index entry is removed. So a collection with 8 indexes does roughly 8× the index work per write versus an un-indexed one. The practical guidance:

- Index the fields you **filter and sort on**, then stop. Don't index "just in case."
- Prefer **one good compound index** (ESR) over several overlapping single-field ones — the compound index serves more queries and costs one structure.
- Periodically review `$indexStats` and **drop indexes with zero `accesses`** — they're pure write tax.
- On very write-heavy collections, fewer indexes can matter more than faster reads; measure both sides.

---

## 6. The Aggregation Pipeline

### 6.1 What it is and why **[I]**

The aggregation pipeline is MongoDB's data-processing engine — think of it as a programmable assembly line. Documents flow in at one end and pass through an **ordered array of stages**; each stage transforms the stream (filters, groups, reshapes, joins, computes) and feeds the next. It's how you do everything `find()` can't: grouping, joins, computed reports, analytics, pivots. It replaces (and far exceeds) the old `group`/`mapReduce`.

Two reasons it matters enormously: (1) it runs **on the server, close to the data**, so you don't drag thousands of documents to the client to summarize them; (2) it's **optimized** — the planner reorders and merges stages, and early `$match`/`$sort` can use indexes.

```js
db.collection.aggregate([ { stage1 }, { stage2 }, { stage3 } ])
```

### 6.2 The essential stages **[I/A]**

| Stage | What it does |
|---|---|
| `$match` | Filter documents (same syntax as `find`). Put it FIRST to use indexes & shrink the stream. |
| `$project` | Reshape: include/exclude/rename/compute fields. |
| `$addFields` / `$set` | Add or overwrite fields, keeping the rest. |
| `$group` | Group by a key and compute accumulators (sum, avg, count, push...). |
| `$sort` | Order the stream. Before `$group` it can use an index; after, it's in-memory. |
| `$limit` / `$skip` | Cap / offset. |
| `$unwind` | Explode an array field into one document per element. |
| `$lookup` | Left-outer JOIN to another collection. |
| `$facet` | Run multiple sub-pipelines on the same input (multi-metric dashboards). |
| `$bucket` / `$bucketAuto` | Group into ranges (histograms). |
| `$count` | Count documents reaching this stage. |
| `$unionWith` | Concatenate another collection/pipeline's output. |
| `$out` / `$merge` | Write results to a collection (materialized views, ETL). |
| `$graphLookup` | Recursive lookup — traverse hierarchies/graphs. |

### 6.3 `$group` and accumulators **[I]**

`$group` collapses many documents into one per distinct `_id` (the group key). `_id: null` groups everything into one. Accumulators compute per-group values.

```js
// Total revenue and order count per customer, top spenders first.
db.orders.aggregate([
  { $match: { status: "completed" } },              // 1. filter first (uses index)
  { $group: {
      _id: "$customerId",                           // group key (note the $ = "field value")
      totalSpent: { $sum: "$total" },               // accumulator: sum a field
      orderCount: { $sum: 1 },                       // sum of 1 = count
      avgOrder:   { $avg: "$total" },
      lastOrder:  { $max: "$createdAt" },
      skus:       { $addToSet: "$items.sku" }        // distinct set across the group
  } },
  { $sort: { totalSpent: -1 } },                     // biggest spenders first
  { $limit: 10 }
])
```
Common accumulators: `$sum`, `$avg`, `$min`, `$max`, `$first`, `$last`, `$push` (collect all values into an array), `$addToSet` (distinct), `$count`. (MongoDB 5.0+ also adds window functions via `$setWindowFields` for running totals, moving averages and rankings.)

### 6.4 Expressions & operators **[I]**

Inside stages you use **aggregation expressions**: `"$field"` means "the value of this field"; operators are objects like `{ $add: [...] }`. They compute new values.

```js
db.orders.aggregate([
  { $project: {
      customer: 1,
      total: 1,
      // Arithmetic:
      withTax: { $multiply: ["$total", 1.2] },
      // Conditional (if/then/else):
      tier: { $cond: { if: { $gte: ["$total", 100] }, then: "gold", else: "standard" } },
      // Switch:
      band: { $switch: { branches: [
                { case: { $lt: ["$total", 50] },  then: "low" },
                { case: { $lt: ["$total", 200] }, then: "mid" } ],
              default: "high" } },
      // Dates:
      year:  { $year: "$createdAt" },
      month: { $dateToString: { format: "%Y-%m", date: "$createdAt" } },
      // String:
      upperEmail: { $toUpper: "$customer" },
      // Null handling:
      coupon: { $ifNull: ["$coupon", "NONE"] }
  } }
])
```

### 6.5 `$lookup` — joining collections **[I/A]**

`$lookup` performs a **left outer join**: for each input document it finds matching documents in another collection and attaches them as an array. This is how you "join" referenced data in one query instead of an application-side second query.

```js
// Attach each order's customer document.
db.orders.aggregate([
  { $lookup: {
      from: "customers",          // the collection to join
      localField: "customerId",   // field on the order
      foreignField: "_id",        // field on the customer
      as: "customer"              // result array field (always an ARRAY, even for one match)
  } },
  { $unwind: "$customer" },        // flatten the 1-element array into an object
  { $project: { _id: 1, total: 1, "customer.name": 1, "customer.city": 1 } }
])
```

For richer joins (filter the joined docs, correlated sub-queries) use the **pipeline form** with `let`:

```js
db.posts.aggregate([
  { $lookup: {
      from: "comments",
      let: { pid: "$_id" },                         // expose post _id to the sub-pipeline
      pipeline: [
        { $match: { $expr: { $eq: ["$postId", "$$pid"] } } }, // $$ = a let variable
        { $match: { approved: true } },             // only approved comments
        { $sort: { createdAt: -1 } },
        { $limit: 5 }                               // newest 5 only
      ],
      as: "recentComments"
  } }
])
```

> **Performance:** `$lookup` is fundamentally a per-document nested loop — **the `foreignField` must be indexed**, or each input document triggers a collection scan of the foreign collection. `$lookup` across huge collections is expensive; this is exactly why embedding / extended references (§4.5) often beat joins for hot paths. Put `$match`/`$limit` *before* `$lookup` to reduce how many lookups run.

### 6.6 `$unwind`, `$facet`, `$bucket`, `$graphLookup` **[A]**

**`$unwind`** turns one document with an N-element array into N documents — essential before grouping by array elements (e.g. "revenue per product across orders").

```js
db.orders.aggregate([
  { $unwind: "$items" },                              // one row per line item
  { $group: { _id: "$items.sku",
              unitsSold: { $sum: "$items.qty" },
              revenue:   { $sum: { $multiply: ["$items.qty", "$items.price"] } } } },
  { $sort: { revenue: -1 } }
])
```

**`$facet`** runs several independent pipelines over the *same* input in one pass — ideal for a dashboard or a search results page that needs results + counts + facet breakdowns at once.

```js
db.products.aggregate([
  { $match: { category: "electronics" } },
  { $facet: {
      priceBuckets: [ { $bucket: { groupBy: "$price",
                          boundaries: [0, 50, 100, 500, 1000],
                          default: "1000+",
                          output: { count: { $sum: 1 } } } } ],
      byBrand:      [ { $group: { _id: "$brand", n: { $sum: 1 } } } ],
      topRated:     [ { $sort: { rating: -1 } }, { $limit: 5 },
                      { $project: { name: 1, rating: 1 } } ]
  } }
])
// Result: ONE document with three arrays — priceBuckets, byBrand, topRated.
```

**`$graphLookup`** recursively follows references — traverse a category tree, an org chart, or "friends of friends."

```js
// Walk a category tree downward from "electronics" using parent references.
db.categories.aggregate([
  { $match: { _id: "electronics" } },
  { $graphLookup: {
      from: "categories",
      startWith: "$_id",
      connectFromField: "_id",       // follow this...
      connectToField: "parent",      // ...to documents whose `parent` matches
      as: "descendants",
      maxDepth: 5
  } }
])
```

### 6.7 A real report end-to-end **[A]**

```js
// "Monthly revenue and unique customers for completed orders in 2026, by month."
db.orders.aggregate([
  { $match: { status: "completed",
              createdAt: { $gte: ISODate("2026-01-01"), $lt: ISODate("2027-01-01") } } },
  { $group: {
      _id: { $dateToString: { format: "%Y-%m", date: "$createdAt" } },  // group by month
      revenue:        { $sum: "$total" },
      orders:         { $sum: 1 },
      uniqueBuyers:   { $addToSet: "$customerId" }
  } },
  { $addFields: { uniqueBuyerCount: { $size: "$uniqueBuyers" } } },
  { $project: { uniqueBuyers: 0 } },          // drop the big array, keep its size
  { $sort: { _id: 1 } }                       // chronological
])
```

### 6.7a `$setWindowFields` — running totals, moving averages, rankings **[A]**

**What & why.** `$group` collapses rows; sometimes you want a per-row computation that *looks at neighbouring rows* without collapsing them — a running total, a 7-day moving average, a rank within a partition. That's a **window function** (the SQL `OVER (...)` concept). `$setWindowFields` (5.0+) partitions the stream, orders it, defines a sliding window, and computes accumulators over that window, attaching the result to each document.

```js
// Running revenue total and 3-day moving average, per region, ordered by day.
db.dailySales.aggregate([
  { $setWindowFields: {
      partitionBy: "$region",                       // like GROUP BY for the window
      sortBy: { day: 1 },                           // order within each partition
      output: {
        runningTotal: {
          $sum: "$revenue",
          window: { documents: ["unbounded", "current"] }   // all rows up to this one
        },
        movingAvg3: {
          $avg: "$revenue",
          window: { documents: [-2, 0] }            // this row and the 2 before it
        },
        rank: { $rank: {} }                          // dense ranking within the partition
      }
  } },
  { $sort: { region: 1, day: 1 } }
])
```

Window operators include `$sum`, `$avg`, `$min`/`$max`, `$rank`/`$denseRank`/`$documentNumber`, `$shift` (lag/lead), `$first`/`$last`, and `$derivative`/`$integral` (great for time-series rates). This is the modern replacement for self-joins and client-side post-processing.

### 6.7b `$out` / `$merge` — materialized views & ETL **[A]**

Aggregations are read-only by default, but `$out` and `$merge` write the pipeline's output to a collection. `$out` **replaces** the target collection wholesale; `$merge` **upserts** into it (incremental). Use them to pre-compute expensive reports on a schedule (a poor-man's materialized view) so the live app reads a tiny, ready-made summary collection.

```js
db.events.aggregate([
  { $match: { day: ISODate("2026-06-21") } },
  { $group: { _id: "$page", views: { $sum: 1 } } },
  { $merge: { into: "dailyPageStats", on: "_id",
              whenMatched: "replace", whenNotMatched: "insert" } }  // incremental rollup
])
```

### 6.8 Aggregation performance tips **[A]**

- **`$match` and `$sort` first**, before `$project`/`$group`/`$lookup`, so they can use indexes and shrink the stream early.
- **`$project` away** big unused fields early to reduce memory.
- Each stage has a **100 MB memory limit**; pass `{ allowDiskUse: true }` for big sorts/groups (slower but won't fail).
- Index the `foreignField` for `$lookup`.
- `explain()` works on `aggregate` too — check that `$match` uses an `IXSCAN`.

---

### 6.9 Aggregation recipe cheat-sheet **[I]**

Patterns you will reach for constantly:

```js
// Count by category (GROUP BY + COUNT):
[{ $group: { _id: "$category", n: { $sum: 1 } } }]

// Average / sum / min / max of a field per group:
[{ $group: { _id: "$region", avg: { $avg: "$price" }, max: { $max: "$price" } } }]

// Distinct count (collect a set, then size it):
[{ $group: { _id: null, users: { $addToSet: "$userId" } } },
 { $project: { count: { $size: "$users" } } }]

// Top-N per group (e.g. top 3 products per category) using window functions:
[{ $setWindowFields: { partitionBy: "$category", sortBy: { sales: -1 },
     output: { rank: { $rank: {} } } } },
 { $match: { rank: { $lte: 3 } } }]

// Pivot tags into counts (unwind then group):
[{ $unwind: "$tags" }, { $group: { _id: "$tags", n: { $sum: 1 } } }, { $sort: { n: -1 } }]

// Join + keep only matched (inner join): $lookup then $match on non-empty result:
[{ $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "u" } },
 { $match: { "u.0": { $exists: true } } }]

// Date rollup (per day):
[{ $group: { _id: { $dateTrunc: { date: "$ts", unit: "day" } }, n: { $sum: 1 } } }]

// Conditional sum (count only matching, like SUM(CASE WHEN ...)):
[{ $group: { _id: null,
     paid: { $sum: { $cond: [{ $eq: ["$status", "paid"] }, 1, 0] } } } }]
```

## 7. Schema Validation

### 7.1 What & when **[I]**

By default MongoDB lets you insert anything. **Schema validation** attaches rules (typically **JSON Schema**) to a collection so the server **rejects writes that violate them**. This gives you relational-style guarantees (required fields, types, ranges, enums) while keeping document flexibility.

**When to enforce:** as soon as a collection matters and is written by multiple code paths. Validation is your safety net against typos (`emial`), missing required fields, and type drift (a price stored as a string). It does **not** replace application-level validation (validate at the edge for good error messages) — it's defense in depth at the last line.

### 7.2 Defining a validator **[I]**

```js
db.createCollection("products", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "price", "category"],     // these MUST be present
      additionalProperties: true,                  // allow extra fields? (false = strict)
      properties: {
        name:  { bsonType: "string", minLength: 1, description: "must be a non-empty string" },
        price: { bsonType: "decimal", description: "must be a Decimal128 (money)" },
        category: { enum: ["electronics", "books", "home"], description: "must be a known category" },
        tags:  { bsonType: "array", items: { bsonType: "string" }, maxItems: 20 },
        inStock: { bsonType: "int", minimum: 0 }
      }
    }
  },
  validationLevel: "strict",      // "strict" = all writes | "moderate" = only valid existing docs
  validationAction: "error"       // "error" = reject | "warn" = log only (good for rollout)
})

// Add/replace a validator on an existing collection:
db.runCommand({ collMod: "products", validator: { $jsonSchema: { /* ... */ } } })
```

> **Rollout tip:** start with `validationAction: "warn"` to surface offenders in the log without breaking writes; once the data is clean, switch to `"error"`. Validation applies only to documents being written, not retroactively, so audit existing data separately.

---

## 8. Transactions

### 8.1 What single-document atomicity already gives you **[I]**

The most important fact: **every single-document write in MongoDB is atomic**, including updates to nested arrays and sub-documents. So if your design **embeds** the things that must change together (an order and its line items in one document), you get all-or-nothing semantics **for free**, with no transaction needed. A huge amount of "I need a transaction" instinct from the relational world disappears once you model around the document. This is by design — embedding is partly *how* MongoDB avoids needing transactions.

### 8.2 Multi-document ACID transactions — when you do need them **[I/A]**

When a logically atomic operation must touch **multiple documents in multiple collections** (the classic example: transfer money — debit one account doc, credit another), you need a **multi-document transaction**. MongoDB supports full **ACID** transactions across documents, collections, and even shards.

Requirements & costs to know:
- **Requires a replica set or sharded cluster** (transactions need the oplog) — not a standalone `mongod`.
- A transaction runs inside a **session** and either **commits** (all changes apply) or **aborts** (none do).
- They are **not free**: they hold locks/snapshots and contend; a transaction has a **default 60-second** limit and can abort on write conflicts (you must be prepared to **retry**). Use them for genuine cross-document invariants, not as a habit.

```js
// mongosh: transfer 100 from account A to account B, atomically.
const session = db.getMongo().startSession();
const accounts = session.getDatabase("bank").accounts;
session.startTransaction({ readConcern: { level: "snapshot" },
                           writeConcern: { w: "majority" } });
try {
  accounts.updateOne({ _id: "A" }, { $inc: { balance: -100 } });
  accounts.updateOne({ _id: "B" }, { $inc: { balance:  100 } });
  session.commitTransaction();      // both apply, or neither
} catch (e) {
  session.abortTransaction();       // roll everything back
  throw e;
} finally {
  session.endSession();
}
```

> **Rule:** if *most* of your operations need multi-document transactions, your data is probably relational — reconsider the model (embed more) or reconsider MongoDB. Drivers expose `withTransaction()` which automatically retries on transient errors — prefer it (shown in §11/§12).

---

## 9. Relationships & Joins

This section ties §3, §6.5 and §8 together into a decision framework, because "how do I relate data?" is the question beginners ask most.

You have **three** ways to relate data, in rough order of preference for read-heavy paths:

1. **Embed** (§3) — the related data lives in the document. **No join at all.** Best when bounded and read together. This should be your default for "contains" relationships.
2. **Reference + `$lookup`** (§6.5) — store an `ObjectId` (or natural key) and join server-side at query time. Best for one-to-many/many-to-many/shared data you sometimes need together. Requires an index on the foreign key.
3. **Reference + application-side join** — fetch the parent, then issue a second query for the children (or batch with `$in`). Sometimes simpler, cacheable, and avoids `$lookup` cost; common in app code (Mongoose `populate` does exactly this, §11).

How to choose between `$lookup` and an app-side join when you've decided to reference:
- `$lookup` keeps it **one round trip** and lets you continue the pipeline (group/sort the joined data) — better for reports and when the joined set is needed in further aggregation.
- App-side join (`find` then `find({ _id: { $in: ids } })`) is **easier to cache**, spreads load, and avoids a costly nested-loop join on hot read paths. Mongoose `populate` and DataLoader-style batching live here.

References themselves are just fields holding the target's `_id`. **Type consistency is critical** — store the same type everywhere (binary `ObjectId` is conventional). If you ever store an id as a hex *string* in some docs and an `ObjectId` in others, joins and `$in` queries silently miss. There are **no database-enforced foreign keys**: a referenced document can be deleted, leaving a dangling reference — handle that in application logic or with periodic cleanup.

```js
// Reference + application-side join (batched), the pattern Mongoose populate uses:
const posts = db.posts.find({ authorId: { $in: authorIds } }).toArray();
const ids   = [...new Set(posts.map(p => p.authorId))];
const authors = db.users.find({ _id: { $in: ids } }).toArray();  // ONE query for all authors
// then stitch in app code by _id  -> avoids N+1 queries
```

---

## 10. Replication & Sharding

### 10.1 Replica sets — high availability **[A]**

A **replica set** is a group of `mongod` processes holding the **same data**: one **primary** (takes all writes) and one or more **secondaries** (replicate the primary's **oplog** — an operation log — and stay in sync). If the primary fails, the remaining members **elect** a new primary automatically (usually within seconds). This is how MongoDB provides **high availability and durability** — and it's why transactions and change streams require a replica set.

- Reads default to the primary (strong consistency). You can opt to read from secondaries via **read preference** (`primary`, `primaryPreferred`, `secondary`, `nearest`) to spread read load — accepting that secondaries may lag slightly (eventual consistency).
- An odd number of voting members (3, 5) avoids election ties; an **arbiter** can vote without holding data (use sparingly).

```js
// Inspect and reconfigure a replica set in mongosh:
rs.status()                       // members, who is PRIMARY, replication lag
rs.conf()                         // current configuration
rs.add("mongo2:27017")            // add a member (becomes a secondary, syncs)
rs.addArb("mongo-arb:27017")      // add a voting-only arbiter (no data)
rs.stepDown(60)                   // ask the primary to step down (triggers election)
// A member can be given a priority (higher = more likely to be elected primary)
// and votes; hidden/delayed members serve backups/analytics without taking traffic.
```

**Failover, concretely:** if the primary becomes unreachable, secondaries that can see a majority hold an **election** and one becomes the new primary, usually within a few seconds. During that window writes are briefly unavailable (the driver queues/retries — this is why `retryWrites: true` matters). When the old primary returns, any of its writes that weren't replicated to the majority may be **rolled back** — which is precisely why important writes use `w: "majority"` (a majority-acknowledged write survives any single failover).

### 10.2 Write concern & read concern — the consistency dials **[A]**

These two settings let you trade durability/consistency against latency.

**Write concern (`w`)** — how many members must acknowledge a write before it's "done":
- `w: 1` — primary only (fast, but a failover before replication could lose it).
- `w: "majority"` — a majority of members (durable; survives a primary failure). **The safe default for important data.**
- `j: true` — also require the write be flushed to the on-disk journal.

**Read concern** — what consistency a read guarantees:
- `local` — whatever the node has (may be rolled back).
- `majority` — only data acknowledged by a majority (won't be rolled back).
- `linearizable` / `snapshot` — strongest; for transactions and read-your-write needs.

```js
db.orders.insertOne({ /*...*/ }, { writeConcern: { w: "majority", j: true } })
db.orders.find({ /*...*/ }).readConcern("majority")
```

### 10.3 Sharding — horizontal scale **[A]**

When data or throughput outgrows one machine, **sharding** partitions a collection across multiple replica sets (**shards**). A **`mongos`** router sits in front and directs each query to the right shard(s); **config servers** hold the metadata (which ranges live where). This scales **writes and storage horizontally** — the thing replication alone can't do.

The single most important decision is the **shard key** — the field(s) MongoDB uses to partition documents. A good shard key has:
- **High cardinality** — many distinct values, so data divides finely.
- **Even write distribution** — avoid **monotonically increasing keys** (like raw timestamps or `ObjectId`), which send *all* new writes to one shard (a "hot shard"). Use a **hashed** shard key or a **compound** key to spread them.
- **Query alignment** — most queries should include the shard key so `mongos` can target one shard instead of **scatter-gathering** to all of them.

```js
sh.enableSharding("shop")
// Hashed key spreads writes evenly (good for high insert rates, point lookups by userId):
sh.shardCollection("shop.events", { userId: "hashed" })
// Compound key: target queries by tenant, distribute within it:
sh.shardCollection("shop.orders", { tenantId: 1, orderId: 1 })
```

> **Gotcha:** the shard key is (historically) hard to change and immutable per document. Choosing it badly — e.g. a low-cardinality `status` field, or a monotonically increasing timestamp — is one of the most expensive mistakes in MongoDB. Model your dominant query and write patterns *before* sharding. **Don't shard prematurely**; a well-indexed replica set handles a lot. (⚡ 8.x has made *reshardable* keys and key refinement much more practical, but it's still a heavy operation.)

---

### 10.4 Connection strings (URIs) — what every part means **[I]**

Drivers and `mongosh` connect via a URI. Understanding it saves a lot of "can't connect" pain.

```text
mongodb://user:pass@host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0&authSource=admin&w=majority&retryWrites=true
  scheme  └─ credentials ─┘ └────────── seed list (all RS members) ──────────┘ └db┘ └──────────────── options ────────────────────────────┘
```

| Part / option | Meaning |
|---|---|
| `mongodb://` vs `mongodb+srv://` | plain host list vs DNS SRV record (Atlas uses `+srv`, auto-discovers members) |
| seed list | list all replica-set members so the driver can find the primary |
| `/mydb` | default database for operations |
| `replicaSet=rs0` | the set name — required for the driver to do failover |
| `authSource=admin` | which db holds the user (usually `admin`) |
| `w=majority` | default write concern |
| `retryWrites=true` | auto-retry a write once on transient errors (failover) |
| `maxPoolSize=50` | connection pool size |
| `tls=true` | require TLS |

## 11. Node.js Usage — Native Driver & Mongoose

You have two main ways to talk to MongoDB from Node: the **native `mongodb` driver** (thin, fast, exactly the shell API) and **Mongoose** (an ODM — Object Document Mapper — that adds schemas, validation, middleware, and `populate`). Use the native driver when you want full control and minimal overhead; use Mongoose when you want structure, validation, and convenience. We'll show both, then a worked blog example.

### 11.1 The native driver — connection & CRUD **[I]**

```js
// npm i mongodb
import { MongoClient, ObjectId } from "mongodb";

// Create ONE client per process and reuse it — it manages a connection POOL internally.
// Never open a connection per request (that exhausts the server and kills performance).
const client = new MongoClient("mongodb://localhost:27017", {
  maxPoolSize: 50,            // tune to your concurrency
  retryWrites: true           // auto-retry transient write failures
});
await client.connect();
const db = client.db("shop");
const users = db.collection("users");

// CRUD mirrors the shell exactly:
const { insertedId } = await users.insertOne({ name: "Ada", email: "ada@x.com", roles: ["admin"] });
const ada  = await users.findOne({ _id: insertedId });
const many = await users.find({ "roles": "admin" }).sort({ name: 1 }).limit(10).toArray();
await users.updateOne({ _id: insertedId }, { $set: { age: 37 }, $inc: { logins: 1 } });
await users.deleteOne({ _id: insertedId });

// Aggregation:
const report = await db.collection("orders").aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customerId", total: { $sum: "$total" } } },
  { $sort: { total: -1 } }
]).toArray();
```

Express handler with the native driver:

```js
import express from "express";
const app = express();
app.use(express.json());

// Reuse the connected `db` from above (initialize at startup, not per-request).
app.get("/users/:id", async (req, res) => {
  try {
    const user = await db.collection("users").findOne({ _id: new ObjectId(req.params.id) });
    if (!user) return res.status(404).json({ error: "not found" });
    res.json(user);
  } catch (e) {
    res.status(400).json({ error: "invalid id" });   // ObjectId() throws on bad hex
  }
});

app.post("/users", async (req, res) => {
  const { insertedId } = await db.collection("users").insertOne(req.body);
  res.status(201).json({ _id: insertedId });
});
```

### 11.2 Mongoose — schemas, models, validation **[I]**

Mongoose adds an application-level schema layer on top of the driver. You define a **Schema** (field types, validation, defaults), compile it into a **Model**, and use model methods to query. It gives you structure, casting, validation, virtuals, middleware (hooks), and `populate`.

```js
// npm i mongoose
import mongoose, { Schema, model } from "mongoose";
await mongoose.connect("mongodb://localhost:27017/blog");

const userSchema = new Schema({
  name:  { type: String, required: true, trim: true },
  email: { type: String, required: true, unique: true, lowercase: true,
           match: [/^\S+@\S+\.\S+$/, "invalid email"] },   // built-in validation
  roles: { type: [String], default: ["author"], enum: ["author", "admin", "editor"] },
  bio:   String
}, { timestamps: true });   // auto-adds createdAt / updatedAt

const User = model("User", userSchema);   // collection name -> "users" (pluralized)

// CRUD via the model (returns Mongoose Documents with helpers):
const u = await User.create({ name: "Ada", email: "ADA@X.com" }); // email lowercased by schema
const found = await User.findById(u._id);
await User.updateOne({ _id: u._id }, { $set: { bio: "pioneer" } });
const admins = await User.find({ roles: "admin" }).sort("name").limit(10);
```

**Lean queries** — by default Mongoose hydrates each result into a full Document (with getters, virtuals, change tracking) which is expensive. For **read-only** endpoints add `.lean()` to get plain JS objects — significantly faster and lighter.

```js
const fast = await User.find({ roles: "admin" }).lean();  // plain objects, no Mongoose overhead
```

**Middleware / hooks** run logic around operations — e.g. hash a password before saving, cascade deletes, audit:

```js
userSchema.pre("save", function (next) {
  if (this.isModified("email")) this.email = this.email.toLowerCase();
  next();
});
userSchema.post("save", function (doc) { console.log("saved", doc._id); });
```

**Virtuals** are computed properties not stored in the DB:

```js
userSchema.virtual("displayName").get(function () { return `${this.name} <${this.email}>`; });
```

**Embedded sub-documents & nested schemas** map embedding (§3) directly onto Mongoose. Define a sub-schema and embed it as a field or an array — validation, defaults and casting apply to the embedded shape too:

```js
const addressSchema = new Schema({
  label: { type: String, enum: ["home", "work"] },
  city:  { type: String, required: true },
  zip:   String
}, { _id: false });   // _id:false -> don't add an _id to each embedded address

const customerSchema = new Schema({
  name: { type: String, required: true },
  addresses: { type: [addressSchema], default: [] }  // embedded array, validated per element
});
// Querying embedded fields uses dot notation, same as the shell:
await Customer.find({ "addresses.city": "London" });
// Atomic array updates use the same operators:
await Customer.updateOne({ _id: id }, { $push: { addresses: { label: "work", city: "Leeds" } } });
```

> **Lean vs hydrated, restated:** use full documents when you need to call `.save()`, run validation, or use virtuals/middleware on the result; use `.lean()` for pure read endpoints (lists, APIs) where you just serialize to JSON — it can be several times faster and uses far less memory because it skips building Mongoose Document wrappers.

**Custom validators & async validation** let you enforce business rules at the schema layer (this is application-level validation; pair it with server-side JSON Schema validation, §7, for defense in depth):

```js
const orderSchema = new Schema({
  email: { type: String, required: true,
    validate: { validator: v => /^\S+@\S+\.\S+$/.test(v), message: "bad email" } },
  qty: { type: Number, min: [1, "qty must be >= 1"], max: 999 },
  status: { type: String, enum: ["pending", "paid", "shipped"], default: "pending" }
});
// Validation runs automatically on .create()/.save() and (opt-in) on updates with
// { runValidators: true }. By default updateOne does NOT run schema validators — pass it!
await Order.updateOne({ _id: id }, { $set: { qty: 0 } }, { runValidators: true }); // rejects
```

> **Gotcha:** Mongoose validators run on `save`/`create` but **not on `updateOne`/`findOneAndUpdate` by default** — you must pass `{ runValidators: true }`. Server-side JSON Schema validation (§7) has no such gap; it applies to every write regardless of source.

### 11.3 Relationships & populate **[I]**

`populate` is Mongoose's application-side join: store a `ref` to another model's `_id`, and `populate` issues the secondary query and stitches the documents in.

```js
const postSchema = new Schema({
  title:  { type: String, required: true },
  body:   String,
  author: { type: Schema.Types.ObjectId, ref: "User", required: true }, // reference
  tags:   [String]
}, { timestamps: true });
const Post = model("Post", postSchema);

// Populate the author when reading a post (Mongoose does a second find under the hood):
const post = await Post.findById(id).populate("author", "name email").lean();
// post.author is now the User document (name & email fields), not just an ObjectId.
```

> **N+1 warning:** `populate` inside a loop fires one query per item. Either `populate` on the list query (Mongoose batches with `$in`), or use an aggregation `$lookup`. Prefer `$lookup` when you then need to group/sort on the joined data.

### 11.4 NestJS integration **[I/A]**

NestJS wraps Mongoose with `@nestjs/mongoose` (decorator-based schemas + DI):

```ts
// user.schema.ts
import { Prop, Schema, SchemaFactory } from "@nestjs/mongoose";
import { Document } from "mongoose";

@Schema({ timestamps: true })
export class User {
  @Prop({ required: true }) name: string;
  @Prop({ required: true, unique: true, lowercase: true }) email: string;
  @Prop({ type: [String], default: ["author"] }) roles: string[];
}
export type UserDocument = User & Document;
export const UserSchema = SchemaFactory.createForClass(User);

// user.module.ts
@Module({
  imports: [MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])],
  providers: [UserService], controllers: [UserController],
})
export class UserModule {}

// user.service.ts — inject the model
@Injectable()
export class UserService {
  constructor(@InjectModel(User.name) private model: Model<UserDocument>) {}
  create(dto: CreateUserDto) { return this.model.create(dto); }
  findAll() { return this.model.find().lean().exec(); }
  findOne(id: string) { return this.model.findById(id).exec(); }
}
```

### 11.5 Transactions in Node **[A]**

```js
// Native driver — withTransaction auto-retries transient errors. Pass the session to each op!
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    const accts = client.db("bank").collection("accounts");
    await accts.updateOne({ _id: "A" }, { $inc: { balance: -100 } }, { session });
    await accts.updateOne({ _id: "B" }, { $inc: { balance:  100 } }, { session });
  });
} finally { await session.endSession(); }

// Mongoose:
const s = await mongoose.startSession();
await s.withTransaction(async () => {
  await Account.updateOne({ _id: "A" }, { $inc: { balance: -100 } }, { session: s });
  await Account.updateOne({ _id: "B" }, { $inc: { balance:  100 } }, { session: s });
});
s.endSession();
```

### 11.5a Change streams — reacting to data in real time **[A]**

A **change stream** lets your app subscribe to inserts/updates/deletes on a collection (it tails the oplog under the hood), so you can build real-time features — live dashboards, cache invalidation, notifications, syncing to a search index — **without polling**. Requires a replica set.

```js
// Native driver: watch for new comments and react.
const stream = db.collection("comments").watch([
  { $match: { operationType: "insert" } }     // only inserts
], { fullDocument: "updateLookup" });          // include the full doc on updates too

for await (const change of stream) {
  const comment = change.fullDocument;
  // e.g. push to a websocket, bump a counter, invalidate a cache, index in search...
  console.log("new comment on post", comment.post);
}
// Change streams are resumable: persist change._id (the resume token) and pass
// { resumeAfter: token } on reconnect so you never miss events after a restart.
```

### 11.5b Iterating cursors & handling errors **[I]**

```js
// Don't .toArray() a huge result set into memory — iterate the cursor instead:
const cursor = db.collection("events").find({ year: 2026 });
for await (const doc of cursor) {
  process(doc);            // streamed in batches; constant memory
}

// Duplicate-key (unique index) errors surface as code 11000:
try {
  await users.insertOne({ email: "ada@x.com" });
} catch (e) {
  if (e.code === 11000) { /* email already taken -> 409 Conflict */ }
  else throw e;
}
```

### 11.6 Worked example — a blog (users, posts, comments) **[I/A]**

A realistic small schema: users authored posts; comments reference both. We embed nothing unbounded; comments are referenced (could be many). We keep a computed `commentCount` (computed pattern) and show an aggregation report.

```js
// ---- Schemas (Mongoose) ----
const userSchema = new Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true, lowercase: true }
}, { timestamps: true });

const postSchema = new Schema({
  title:  { type: String, required: true },
  body:   String,
  author: { type: Schema.Types.ObjectId, ref: "User", required: true },
  tags:   { type: [String], index: true },          // multikey index for tag queries
  commentCount: { type: Number, default: 0 }         // computed pattern
}, { timestamps: true });
postSchema.index({ author: 1, createdAt: -1 });      // ESR: equality author, sort createdAt

const commentSchema = new Schema({
  post:   { type: Schema.Types.ObjectId, ref: "Post", required: true, index: true },
  author: { type: Schema.Types.ObjectId, ref: "User", required: true },
  body:   { type: String, required: true }
}, { timestamps: true });

const User = model("User", userSchema);
const Post = model("Post", postSchema);
const Comment = model("Comment", commentSchema);

// ---- Create flow ----
const ada = await User.create({ name: "Ada", email: "ada@x.com" });
const post = await Post.create({ title: "Hello", body: "First post", author: ada._id, tags: ["intro"] });

// Add a comment AND keep commentCount in sync (atomically per doc; in a txn if you need strictness):
await Comment.create({ post: post._id, author: ada._id, body: "Nice!" });
await Post.updateOne({ _id: post._id }, { $inc: { commentCount: 1 } });

// ---- Read: a post page with author + recent comments (populate / app-side join) ----
const page = await Post.findById(post._id).populate("author", "name").lean();
const recent = await Comment.find({ post: post._id })
  .sort({ createdAt: -1 }).limit(10)
  .populate("author", "name").lean();

// ---- Aggregation report: top authors by post count and total comments ----
const topAuthors = await Post.aggregate([
  { $group: { _id: "$author", posts: { $sum: 1 }, comments: { $sum: "$commentCount" } } },
  { $lookup: { from: "users", localField: "_id", foreignField: "_id", as: "user" } },
  { $unwind: "$user" },
  { $project: { _id: 0, name: "$user.name", posts: 1, comments: 1 } },
  { $sort: { posts: -1 } },
  { $limit: 5 }
]);
```

---

## 12. Go Usage — mongo-go-driver from Gin

Go uses the **official driver** (`go.mongodb.org/mongo-driver/v2/mongo` in the 2.x line). There's no Mongoose-style ODM in the standard toolkit — you map documents to **structs with `bson` tags** and use the driver directly. The API mirrors the shell: `InsertOne`, `Find`, `UpdateOne`, `Aggregate`, etc. Below mirrors the Node blog example, from a Gin HTTP server.

### 12.1 Connect & model with BSON struct tags **[I]**

```go
package main

import (
	"context"
	"time"

	"go.mongodb.org/mongo-driver/v2/bson"
	"go.mongodb.org/mongo-driver/v2/mongo"
	"go.mongodb.org/mongo-driver/v2/mongo/options"
)

// Structs map to documents via `bson` tags. omitempty skips zero values on write.
type User struct {
	ID        bson.ObjectID `bson:"_id,omitempty" json:"id"`
	Name      string        `bson:"name"           json:"name"`
	Email     string        `bson:"email"          json:"email"`
	CreatedAt time.Time     `bson:"createdAt"      json:"createdAt"`
}

type Post struct {
	ID           bson.ObjectID `bson:"_id,omitempty"  json:"id"`
	Title        string        `bson:"title"          json:"title"`
	Body         string        `bson:"body"           json:"body"`
	Author       bson.ObjectID `bson:"author"         json:"author"`
	Tags         []string      `bson:"tags"           json:"tags"`
	CommentCount int           `bson:"commentCount"   json:"commentCount"`
	CreatedAt    time.Time     `bson:"createdAt"      json:"createdAt"`
}

// Create ONE client per process and reuse it (it pools connections internally).
func connect(ctx context.Context) (*mongo.Client, error) {
	opts := options.Client().
		ApplyURI("mongodb://localhost:27017").
		SetMaxPoolSize(50)
	return mongo.Connect(opts)   // v2: no ctx arg here; ping separately
}
```

### 12.2 CRUD **[I]**

```go
func crudExamples(ctx context.Context, db *mongo.Database) error {
	users := db.Collection("users")

	// Insert one
	res, err := users.InsertOne(ctx, User{
		Name: "Ada", Email: "ada@x.com", CreatedAt: time.Now(),
	})
	if err != nil {
		return err
	}
	id := res.InsertedID.(bson.ObjectID) // the generated _id

	// Find one -> decode into a struct
	var u User
	if err := users.FindOne(ctx, bson.M{"_id": id}).Decode(&u); err != nil {
		return err // mongo.ErrNoDocuments if not found
	}

	// Find many -> iterate the cursor (always Close it)
	cur, err := users.Find(ctx, bson.M{"name": bson.M{"$regex": "^A"}},
		options.Find().SetSort(bson.D{{Key: "name", Value: 1}}).SetLimit(10))
	if err != nil {
		return err
	}
	defer cur.Close(ctx)
	var list []User
	if err := cur.All(ctx, &list); err != nil { // decode all at once
		return err
	}

	// Update with operators
	_, err = users.UpdateOne(ctx, bson.M{"_id": id},
		bson.M{"$set": bson.M{"name": "Ada Lovelace"}, "$inc": bson.M{"logins": 1}})
	if err != nil {
		return err
	}

	// Delete
	_, err = users.DeleteOne(ctx, bson.M{"_id": id})
	return err
}
```

> **`bson.M` vs `bson.D`:** `bson.M` is an unordered `map` — convenient for filters where order doesn't matter. `bson.D` is an *ordered* slice — use it where **order matters**: compound index keys, `$sort`, and **aggregation pipeline stages** (a pipeline is an ordered list of single-key stage documents). Using `bson.M` for a multi-key stage can reorder keys unpredictably.

### 12.3 Aggregation **[I/A]**

```go
// Top authors by post count + total comments (mirrors the Node example).
func topAuthors(ctx context.Context, db *mongo.Database) ([]bson.M, error) {
	pipeline := mongo.Pipeline{
		// Stages are bson.D because key/stage order matters.
		bson.D{{Key: "$group", Value: bson.D{
			{Key: "_id", Value: "$author"},
			{Key: "posts", Value: bson.D{{Key: "$sum", Value: 1}}},
			{Key: "comments", Value: bson.D{{Key: "$sum", Value: "$commentCount"}}},
		}}},
		bson.D{{Key: "$lookup", Value: bson.D{
			{Key: "from", Value: "users"},
			{Key: "localField", Value: "_id"},
			{Key: "foreignField", Value: "_id"},
			{Key: "as", Value: "user"},
		}}},
		bson.D{{Key: "$unwind", Value: "$user"}},
		bson.D{{Key: "$sort", Value: bson.D{{Key: "posts", Value: -1}}}},
		bson.D{{Key: "$limit", Value: 5}},
	}
	cur, err := db.Collection("posts").Aggregate(ctx, pipeline)
	if err != nil {
		return nil, err
	}
	defer cur.Close(ctx)
	var out []bson.M
	if err := cur.All(ctx, &out); err != nil {
		return nil, err
	}
	return out, nil
}
```

### 12.4 Transactions **[A]**

```go
func transfer(ctx context.Context, client *mongo.Client) error {
	session, err := client.StartSession()
	if err != nil {
		return err
	}
	defer session.EndSession(ctx)

	// WithTransaction auto-retries on transient/commit errors.
	_, err = session.WithTransaction(ctx, func(sc context.Context) (interface{}, error) {
		accts := client.Database("bank").Collection("accounts")
		if _, err := accts.UpdateOne(sc, bson.M{"_id": "A"},
			bson.M{"$inc": bson.M{"balance": -100}}); err != nil {
			return nil, err
		}
		if _, err := accts.UpdateOne(sc, bson.M{"_id": "B"},
			bson.M{"$inc": bson.M{"balance": 100}}); err != nil {
			return nil, err
		}
		return nil, nil
	})
	return err
}
```

### 12.5 Gin handlers — worked blog endpoints **[I/A]**

```go
package main

import (
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"go.mongodb.org/mongo-driver/v2/bson"
	"go.mongodb.org/mongo-driver/v2/mongo"
)

type Server struct{ db *mongo.Database }

// POST /posts  — create a post
func (s *Server) createPost(c *gin.Context) {
	var in struct {
		Title  string   `json:"title" binding:"required"`
		Body   string   `json:"body"`
		Author string   `json:"author" binding:"required"` // hex string from client
		Tags   []string `json:"tags"`
	}
	if err := c.ShouldBindJSON(&in); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	authorID, err := bson.ObjectIDFromHex(in.Author) // convert hex string -> ObjectID
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid author id"})
		return
	}
	post := Post{
		Title: in.Title, Body: in.Body, Author: authorID,
		Tags: in.Tags, CreatedAt: time.Now(),
	}
	res, err := s.db.Collection("posts").InsertOne(c, post)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusCreated, gin.H{"id": res.InsertedID})
}

// GET /posts/:id  — fetch a single post
func (s *Server) getPost(c *gin.Context) {
	id, err := bson.ObjectIDFromHex(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
		return
	}
	var post Post
	err = s.db.Collection("posts").FindOne(c, bson.M{"_id": id}).Decode(&post)
	if err == mongo.ErrNoDocuments {
		c.JSON(http.StatusNotFound, gin.H{"error": "not found"})
		return
	} else if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, post)
}

func main() {
	client, _ := connect(c)             // from §12.1 (handle errors in real code)
	s := &Server{db: client.Database("blog")}
	r := gin.Default()
	r.POST("/posts", s.createPost)
	r.GET("/posts/:id", s.getPost)
	r.GET("/posts", s.listPosts)
	r.POST("/posts/:id/comments", s.addComment)
	r.Run(":8080")
}
```

### 12.6 Listing with pagination & a comment-with-counter transaction **[A]**

This mirrors the Node blog: keyset pagination for the list, and adding a comment while keeping the post's `commentCount` in sync inside a transaction.

```go
// GET /posts?after=<hexId>&limit=20  — keyset pagination (fast at any depth, §2.2)
func (s *Server) listPosts(c *gin.Context) {
	filter := bson.M{}
	if after := c.Query("after"); after != "" {
		if id, err := bson.ObjectIDFromHex(after); err == nil {
			filter["_id"] = bson.M{"$gt": id} // continue after the last id seen
		}
	}
	opts := options.Find().
		SetSort(bson.D{{Key: "_id", Value: 1}}). // sort by indexed _id, not skip()
		SetLimit(20)

	cur, err := s.db.Collection("posts").Find(c, filter, opts)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	defer cur.Close(c)

	var posts []Post
	if err := cur.All(c, &posts); err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"posts": posts})
}

// POST /posts/:id/comments — insert a comment AND bump the post's commentCount atomically.
func (s *Server) addComment(c *gin.Context) {
	postID, err := bson.ObjectIDFromHex(c.Param("id"))
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid post id"})
		return
	}
	var in struct {
		Author string `json:"author" binding:"required"`
		Body   string `json:"body" binding:"required"`
	}
	if err := c.ShouldBindJSON(&in); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	authorID, err := bson.ObjectIDFromHex(in.Author)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid author id"})
		return
	}

	session, err := s.db.Client().StartSession()
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	defer session.EndSession(c)

	// Both writes commit together, or neither does. Pass the session ctx (sc) to each op.
	_, err = session.WithTransaction(c, func(sc context.Context) (interface{}, error) {
		comment := bson.M{"post": postID, "author": authorID,
			"body": in.Body, "createdAt": time.Now()}
		if _, err := s.db.Collection("comments").InsertOne(sc, comment); err != nil {
			return nil, err
		}
		_, err := s.db.Collection("posts").UpdateOne(sc,
			bson.M{"_id": postID}, bson.M{"$inc": bson.M{"commentCount": 1}})
		return nil, err
	})
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusCreated, gin.H{"ok": true})
}
```

> **Symmetry note:** compare this with the Node example (§11.6). The model is identical — referenced comments, a computed `commentCount`, keyset pagination — only the syntax differs. Go gives you explicit types and `context`-based cancellation/timeouts; Node/Mongoose gives you schemas, validation and `populate` out of the box. Pick per team and workload, not dogma.

---

## 13. Performance & Operations

### 13.1 Indexing strategy **[A]**

- **Index for your queries, not your schema.** Look at the queries you actually run (and their sorts), then build compound indexes via the **ESR rule** (§5.2). One well-ordered compound index often replaces several single-field ones.
- **Each index costs writes and RAM.** Audit with `db.collection.aggregate([{ $indexStats: {} }])` (usage counts) and drop indexes nobody uses.
- **Aim to keep indexes (and the working set) in RAM.** When indexes spill to disk, performance falls off a cliff.
- Build indexes on **large collections** in the background / during low traffic; index builds use resources.

### 13.2 Query profiling & finding slow queries **[A]**

```js
// The database profiler logs slow operations to system.profile.
db.setProfilingLevel(1, { slowms: 100 })   // log ops slower than 100ms
db.system.profile.find().sort({ ts: -1 }).limit(5)   // recent slow ops
db.setProfilingLevel(0)                     // turn off

// For a single query, explain() with executionStats (see §5.5) is the precise tool.
// Watch for: COLLSCAN, a SORT stage, or totalDocsExamined >> nReturned.
```
The `mongod` log also records slow queries (default threshold 100 ms). In Atlas, the Performance Advisor suggests indexes automatically.

### 13.3 Connection pooling **[I]**

Both Node and Go drivers maintain an internal **connection pool**. The cardinal rule: **create one client/connection at process startup and reuse it for the whole process.** Opening a client per request (or worse, per query) exhausts the server's connections and adds handshake latency. Tune `maxPoolSize` to roughly match your app's concurrency. In serverless/Lambda, cache the client across invocations (module-scope) so warm starts reuse it.

### 13.4 Common slow-query causes **[A]**

| Symptom | Likely cause | Fix |
|---|---|---|
| `COLLSCAN` in explain | No usable index | Add an index matching the filter (ESR) |
| `SORT` stage, high time | Sort not served by index | Reorder compound index per ESR |
| `totalDocsExamined >> nReturned` | Index not selective / wrong order | Better compound index; covered query |
| Slow `$lookup` | `foreignField` not indexed | Index it; or embed/denormalize |
| Slow deep pagination | `skip(N)` with large N | Range/keyset pagination |
| Slow regex | Unanchored / case-insensitive regex | Anchor `^`; use a real index or Atlas Search |
| Latency spikes on writes | Too many indexes / huge documents | Drop unused indexes; shrink docs |

### 13.5 Data size considerations **[A]**

- **Field names are stored in every document** — short-ish names save space at scale (don't go cryptic, but `qty` over `quantityOrdered` across a billion docs adds up). The Attribute Pattern and compression mitigate this.
- WiredTiger **compresses** data (snappy by default; zstd available) — design still matters, but disk is rarely the binding constraint; RAM (working set) is.
- Keep documents **small and bounded**; giant documents and unbounded arrays (§15) wreck cache efficiency and update speed.

---

## 14. Vector Search & Atlas Search

### 14.1 Why this matters in 2026 **[A]**

Modern AI apps (semantic search, RAG, recommendations) represent text/images as **embedding vectors** — arrays of floats where "similar things are close." MongoDB Atlas can **store these vectors alongside your documents and search by similarity**, so you don't need a separate vector database: your operational data and its embeddings live together, and you can pre-filter by normal fields and rank by vector closeness in one query.

> **⚡ Version note:** Vector Search and **Atlas Search** (Lucene-backed full-text search with relevance, fuzzy matching, facets, autocomplete) are **Atlas features** (also available via the Atlas Local Docker image / Enterprise search nodes), not the plain Community `mongod`. Index definitions and the `$vectorSearch` stage are evolving quickly — confirm against current Atlas docs.

### 14.2 Storing embeddings & the vector index **[A]**

Store the embedding as an array of numbers on the document:

```js
{
  _id: ObjectId("..."),
  title: "Intro to MongoDB",
  text: "MongoDB is a document database...",
  category: "database",
  embedding: [0.0123, -0.0456, 0.0789, /* ...e.g. 1536 dims from your model... */]
}
```

Create a **vector search index** (Atlas; via UI, Admin API, or `createSearchIndex`). `numDimensions` must match your embedding model; `similarity` is usually `cosine`.

```js
db.articles.createSearchIndex("vector_index", "vectorSearch", {
  fields: [
    { type: "vector", path: "embedding", numDimensions: 1536, similarity: "cosine" },
    { type: "filter", path: "category" }   // allow pre-filtering by category
  ]
})
```

### 14.3 Querying with `$vectorSearch` **[A]**

```js
// Embed the user's query with the SAME model that produced the stored embeddings,
// then find the nearest documents (semantic search / RAG retrieval).
const queryVector = await embed("how do I model relationships?");  // your embedding call
db.articles.aggregate([
  { $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryVector,
      numCandidates: 200,          // ANN candidates to consider (recall vs speed)
      limit: 5,                    // top-K to return
      filter: { category: "database" }   // pre-filter (uses the filter field above)
  } },
  { $project: { title: 1, text: 1, score: { $meta: "vectorSearchScore" } } }
])
```

Atlas Search (full-text) uses the `$search` stage similarly — `text`, `autocomplete`, `compound`, fuzzy matching, faceting — when you need keyword relevance rather than (or combined with, "hybrid search") semantic similarity.

---

## 15. Anti-Patterns & Gotchas

These are the mistakes that turn a fast MongoDB into a slow, painful one. Most trace back to ignoring the access patterns or treating Mongo like SQL.

- **Unbounded array growth.** Embedding an array that grows forever (all comments, all events, all log lines) eventually hits the 16 MB limit — and degrades long before that, because the whole document is rewritten on each update. **Fix:** reference the children, or use the bucket / outlier / subset patterns (§4).
- **Massive documents.** Big documents (especially with large arrays or blobs) blow the WiredTiger cache and slow every read/update. **Fix:** keep documents small and bounded; store files in object storage + a URL, or GridFS for true large files.
- **Treating it like SQL.** One collection per "table," foreign keys everywhere, `$lookup` to reassemble on every read. You lose embedding's speed and gain join cost. **Fix:** model around your reads — embed what you read together.
- **Missing indexes.** The most common cause of slowness. A query with no usable index does a `COLLSCAN`. **Fix:** `explain()` your hot queries; build ESR-ordered indexes.
- **Too many indexes.** The opposite failure: every index taxes writes and RAM. **Fix:** drop unused indexes (`$indexStats`).
- **Over-embedding.** Embedding shared/large/independently-queried data leads to massive duplication and update fan-out. **Fix:** reference shared/unbounded data; use extended references for the few fields you display.
- **`ObjectId` vs string id mismatch.** Storing references as hex strings in some docs and `ObjectId` in others — joins and `$in` silently miss. **Fix:** pick one type (binary `ObjectId`) and be consistent everywhere.
- **The 16 MB limit, ignored.** Designs that "work in dev" with small data explode in production. **Fix:** ensure every array has a known upper bound by design.
- **Money as `double`.** Floating-point rounding errors in currency. **Fix:** `Decimal128` (`NumberDecimal`) or integer minor units (cents).
- **Dates as strings.** Storing `"2026-06-21"` as a string breaks range queries and date math. **Fix:** store real BSON `Date`.
- **Unbounded `$skip` pagination.** `skip(100000)` walks and discards 100k docs per page. **Fix:** keyset/range pagination on an indexed sort key.
- **Unanchored / case-insensitive regex.** `{$regex: /name/i}` can't use an index. **Fix:** anchor with `^`, or use Atlas Search for fuzzy/text needs.
- **Reading from secondaries expecting strong consistency.** Secondaries lag; `secondary` reads can be stale. **Fix:** read from primary for read-your-writes, or use causal consistency / appropriate read concern.
- **Premature sharding / bad shard key.** Sharding a small dataset, or picking a monotonically increasing or low-cardinality shard key (hot shard, scatter-gather). **Fix:** don't shard until you must; choose a high-cardinality, write-distributing, query-aligned key (§10.3).
- **Per-request connections.** Opening a client per request exhausts the pool. **Fix:** one reused client per process.
- **Case-insensitive matching with `$regex` everywhere.** Slow and unindexed. **Fix:** store a normalized (lowercased) field and index it, or use a **collation** on the index for proper case-insensitive comparison.
- **Trusting client input as a query.** Passing user JSON straight into a filter enables **NoSQL injection** (e.g. `{ "$gt": "" }` smuggled in). **Fix:** validate/cast types at the edge; never let strings become operator objects.

### 15.1 Security & deployment checklist **[A]**

Defaults are convenient, not safe. Before anything faces a network:

- **Enable authentication & authorization** (`--auth`); create least-privilege roles per service (a read-only reporting user, a read-write app user). Never run an internet-facing `mongod` without auth — open MongoDB instances are routinely scanned and ransomed.
- **Bind to private interfaces / use TLS.** Don't expose 27017 publicly; require TLS for client connections.
- **Use a connection string from a secret**, not hard-coded. Set `w: "majority"` for important writes.
- **Back up** with `mongodump`/snapshots (or Atlas continuous backup) and **test restores**.
- **Monitor** replication lag, cache hit ratio, slow queries, and connection counts.
- Consider **Queryable Encryption** / field-level encryption for sensitive fields (PII, secrets) so they're encrypted even in memory/at rest.

---

## 16. Study Path & Build-to-Learn Projects

**Suggested order:** §1–2 (model + CRUD in mongosh, get fluent) → §3 (embedding vs referencing — the core skill) → §5 (indexing, including ESR) → §6 (aggregation — practice until comfortable) → §4 (design patterns — revisit as you model real things) → §7–9 (validation, transactions, relationships) → §11 or §12 (your app language) → §10, §13 (replication/sharding & ops) → §14–15 (vector search & avoiding anti-patterns).

**Practice as you go in mongosh.** Load a sample dataset (`mongoimport`, or Atlas sample data) and answer questions with aggregations: top N, per-month rollups, joins. `explain()` every query and try to turn a `COLLSCAN` into an `IXSCAN`.

**Build these to cement it:**

1. **A blog (the §11/§12 example, expanded).** Users, posts, comments. Decide embedding vs referencing for each relationship and justify it. Add: tag search (multikey index), a "recent comments" subset on the post (subset pattern), a `commentCount` (computed pattern), and an aggregation report (top authors, posts per month). Build it in Express+Mongoose **and** Gin+Go to feel the difference. Exercises §2–6, §11/§12.

2. **An analytics / time-series store (bucket pattern).** Ingest a high-rate stream of events or sensor readings. First implement the **manual bucket pattern** (§4.1) so you understand it; then switch to a **native time-series collection** and compare storage and query speed. Build dashboards with `$facet` and `$bucket`, hourly/daily rollups with `$group` + `$dateToString`, and TTL to expire old raw data. Exercises §4.1, §5.4, §6.

3. **A product catalog (polymorphic + faceted search).** Products across categories with different attributes (polymorphic pattern). Add: faceted browse with `$facet` (price buckets, brand counts), an extended reference to category, schema validation (§7) on the core fields, and — if you have Atlas — **Atlas Search** for full-text and **vector search** for "find similar products" (§14). Exercises §4, §6, §7, §14.

**Next steps after this guide:** change streams (real-time reactive apps), Atlas Stream Processing, Queryable Encryption, and pairing MongoDB with a cache (Redis) or a relational store where each fits best. For the relational counterpart and a JOIN-heavy contrast, compare with the PostgreSQL guide in this library; for the ORM layer, the Prisma and Mongoose-in-NestJS material.

---

*Part of the offline developer study library. Written for MongoDB 8.x as of 2026. Confirm fast-moving APIs (Atlas Search, Vector Search, driver 2.x) against mongodb.com/docs.*
