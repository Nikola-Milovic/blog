---
date: 2025-05-04T10:43:38+02:00
draft: false
tags: ["nodejs", "postgres", "testing","vitest","integration testing", "tdd"]
title: "Fun & Sane Node.js TDD: Supercharge Postgres Tests with PGLite, Drizzle & Vitest"
cover: 
  image: cover.png
  alt: integration testing with postgres and nodejs using pglite
  hidden: true
---

# Fun & Sane Node.js TDD: Supercharge Postgres Tests with PGLite, Drizzle & Vitest

Hey everyone! Let's talk about testing in Node.js, specifically when databases get involved.

Unit tests? Absolutely essential for checking your functions in isolation. But let's be real, at some point, you *need* to know if your code actually plays nice with a real database, message queue, or whatever external service it relies on. Mocking everything can feel like building a house of cards, fragile, prone to hiding nasty bugs that only surface when things interact for real. And shared dev databases? Don't even get me started. They're a one-way ticket to flaky tests and accidentally messing up your colleagues' work. Nightmare fuel.

Coming from Go, I got spoiled by integration tests that gave me rock-solid confidence. I craved that same feeling in my TypeScript projects. In a [previous post](https://nikolamilovic.com/posts/integration-testing-node-postgres-vitest-testcontainers/), we explored using `testcontainers` to spin up pristine, isolated Postgres instances for our tests. It's a fantastic approach, giving you high fidelity by testing against the real deal (well, a Dockerized version). We even discussed ways to optimize the startup time.

But here's the thing: if you're like me and embrace a Test Driven Development (TDD) workflow, *every millisecond counts*.

(As always, the final project code can be found over at the [github repo](https://github.com/Nikola-Milovic/blog-projects/tree/master/fun-and-sane-tdd-with-node-pglite))

> I am still evaluating and playing around with this approach, so please do only consider it as an educational post rather than a guide.
{.warning}

### TDD: Not Just a Buzzword, It's About Flow

What *is* TDD, really? You'll find plenty of formal definitions, often involving strict red-green-refactor cycles and meticulous unit testing. Kent Beck, a key figure in TDD, described it as a way to "think through your design before you write your functional code."

Honestly, many resources make TDD sound overly rigid, almost tedious. That's fine if your project demands extreme meticulousness, but for many of us, especially working on CRUD heavy apps or exploring new features, a more pragmatic approach works wonders.

My take on TDD? It's about **developing your code *alongside* your tests**. Tests aren't just safety nets for the future; they're your *active guide* during development. They help you shape the functionality, catch edge cases early, and save you the endless cycle of starting your app, hitting it with `curl`, cleaning the database, and repeating. I tend to lean heavily on *integration tests* – testing slices of functionality, often involving the database, because if the data ends up correct in my source of truth (the DB), I can be fairly confident that it's working as intended.

This kind of workflow boosts my velocity and confidence immensely. But... it hinges on a **fast feedback loop**. Waiting even a few seconds for tests to spin up can break your concentration and kill momentum. That "Time To Result" (TTR, yeah, I made that up) needs to be *fast*.

Last post I showcased `testcontainers` approach, which is great and gets you close to a tight feedback loop (~2-4s initial setup, then subsequent test cases are much faster thanks to snapshots), that initial container spin-up time can still feel sluggish, especially if you want to run many test suites in parallel. What if we could make it *even faster*? (The answer is, *maybe*, this is more of an experiment rather than a recommendation on my part)

### Enter PGLite: Postgres in Your Pocket (Almost!)

Recently, I stumbled upon something that felt like a game changer for this workflow: [**PGLite**](https://github.com/electric-sql/pglite).

Imagine Postgres, but compiled to [WASM](https://webassembly.org/) and packaged as a simple TypeScript library. That's PGLite. You can run a genuine Postgres engine right inside your Node.js (or Bun, or even browser!) process *without installing Postgres or any other dependencies*.

It's tiny (under 3MB gzipped!), incredibly fast to start, and supports many common Postgres extensions (like `pgvector`).

Getting started is ridiculously simple:

```typescript
import { PGlite } from '@electric-sql/pglite';

// Spin up an in-memory Postgres instance
const db = new PGlite(); // Or use a file path for persistence: new PGlite('data-directory')

// Run SQL queries!
const result = await db.query("select 'Hello from PGLite!' as message;");
console.log(result.rows);
// -> [ { message: "Hello from PGLite!" } ]

await db.close(); // Clean up
```

It can run entirely in memory (perfect for ephemeral test databases) or persist to the filesystem (if you plan to run parallel tests you will have to persist to disk, since there is an [issue](https://github.com/electric-sql/pglite/issues/324) with running parallel in memory databases. Developed by the folks at [ElectricSQL](https://electric-sql.com/), it's designed for embedding Postgres, but its use cases are broader:

* **Unit and CI testing:** This is our sweet spot! Lightning-fast setup and teardown. Create a unique, fresh Postgres instance for *every single test* if you want, in milliseconds.
* **Local development:** A lightweight alternative to running a full Postgres server.
* **Remote/Web Containers:** So small it's easily embeddable.
* **Edge AI/RAG:** Full `pgvector` support opens up interesting possibilities.

### Drizzle: The ORM You'll Actually Enjoy

To make this setup even smoother, we'll pair PGLite with [**Drizzle ORM**](https://github.com/drizzle-team/drizzle-orm). If you've been burned by clunky, heavyweight ORMs before, give Drizzle a look.

Why Drizzle?

* **TypeScript Native:** Feels right at home in a TS project. Great type safety.
* **Lightweight:** Tiny bundle size (~7.4kb minified+gzipped) and zero dependencies.
* **Stellar DX:** Seriously, it's a joy to use. As a big raw SQL fan, this has been the *only* ORM I've actually enjoyed.
* **Fast Migrations (with a twist):** We'll use a neat trick to keep migrations snappy without manually generating files every time.

## Let's Get Practical: PGLite + Drizzle + Vitest

Alright, theory's great, but let's see it in action. We'll modify the project from the [previous Testcontainers post](https://github.com/Nikola-Milovic/blog-projects/tree/master/integration-testing-node-vitest-testcontainers) to use PGLite and Drizzle.

**1. Installation:**

```bash
npm install @electric-sql/pglite drizzle-orm
npm install -D drizzle-kit vitest supertest @types/supertest
```

**2. Define Your Schema:**

Let's keep it simple with a Drizzle schema.

`src/schema.ts`

```typescript
import { pgTable, serial, varchar } from "drizzle-orm/pg-core";

export const items = pgTable("items", {
 id: serial("id").primaryKey(),
 name: varchar("name", { length: 100 }).notNull(),
});

// Add other tables here as needed
export type Item = typeof items.$inferSelect;
export type NewItem = typeof items.$inferInsert;
```

**3. The Magic Migration Helper:**

Normally, Drizzle uses migration files or `drizzle-kit push` to update your DB schema. But for rapid TDD, constantly generating migrations is a drag. Here's a helper function that uses `drizzle-kit`'s *programmatic API* to figure out the SQL needed to create your schema from scratch. No migration files needed!

`src/testing/db-helper.ts`

```typescript
import * as schema from "../schema";
import { drizzle } from "drizzle-orm/pglite";
import type * as DrizzleKit from "drizzle-kit/api";
import type { PGlite } from "@electric-sql/pglite";

// Hack needed to dynamically import drizzle-kit's API
// See: https://github.com/drizzle-team/drizzle-orm/issues/2853
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const { generateDrizzleJson, generateMigration } = require("drizzle-kit/api") as typeof DrizzleKit;

/**
 * Generates SQL statements to create the schema defined in `src/schema.ts`
 * and applies them to the given PGLite client.
 * This avoids needing migration files for tests, speeding up schema changes.
 */
export async function pushSchema(client: PGlite) {
 const db = drizzle(client, { schema });

 // Generate an empty schema snapshot
 // See: https://github.com/drizzle-team/drizzle-orm/issues/3913
 const prevJson = generateDrizzleJson({});
 // Generate a snapshot based on your current schema definitions
 const curJson = generateDrizzleJson(
  schema,
  prevJson.id, // Use the empty snapshot's ID
  undefined,
  "snake_case", // Or your preferred naming convention
 );

 // Compare the empty snapshot with the current schema to get all CREATE statements
 const statements = await generateMigration(prevJson, curJson);

 // Apply the generated SQL statements
 console.log("Applying schema...");
 for (const statement of statements) {
  await db.execute(statement);
 }
 console.log("Schema applied.");
}
```

*Benefits of this approach:*

* **Rapid Schema Evolution:** Change your Drizzle schema files, rerun tests, and the DB is updated instantly.
* **No Migration File Clutter:** Keeps your project cleaner.
* **Performance:** Applying one set of `CREATE` statements is faster than running potentially hundreds of historical migration files.
* **Fantastic DX:** Forget `drizzle-kit generate` or `push` during development, just code!

**4. Update Your Tests:**

Now, let's update our Vitest setup (`src/server.test.ts`) to use PGLite. We'll use `beforeEach` and `afterEach` to ensure every test gets a fresh, migrated database instance.

`src/server.test.ts`

```typescript
import {
 afterEach,
 beforeEach,
 describe,
 expect,
 it,
 vi,
} from "vitest";
import request from "supertest";
import { getDB } from "./db.js";
import * as schema from "./schema.js";
import { pushSchema } from "./testing/db-helper.js";
import { PGlite } from "@electric-sql/pglite";
import { drizzle } from "drizzle-orm/pglite";
import { eq } from "drizzle-orm";
import app from "./server.js";
import { PgDatabase } from "drizzle-orm/pg-core";
import TestAgent from "supertest/lib/agent.js";

// Crucial: Mock the db module to intercept calls to getDB()
vi.mock("./db");

describe("Items API with PGLite", () => {
 let db: PgDatabase<any, typeof schema>; // Use the correct type
 let client: PGlite; // Hold the PGLite client instance
 let agent: TestAgent; // Supertest agent

 // Setup a fresh PGLite instance and apply schema before each test
 beforeEach(async () => {
  client = new PGlite(
   // You might want to specify a consistent temp directory:
   // dataDir: `./.pglite-test-data/${dbName}` or `/tmp`
  );

  agent = request(app); // Create supertest agent

  // Apply the schema using our helper
  await pushSchema(client);

  // Create the Drizzle instance connected to PGLite
  db = drizzle(client, { schema });

  // Mock getDB() to return *this specific test's* Drizzle instance
  // Use `vi.mocked` for type safety
  vi.mocked(getDB).mockReturnValue(db as any);
 });

 // Close the PGLite client and clean up mocks after each test
 afterEach(async () => {
  await client.close(); // Release resources
  vi.clearAllMocks(); // Reset mocks
  // Consider adding cleanup for the data directory if you specified one
 });

 it("POST /items should create an item", async () => {
  const newItemName = "Test Item PGLite";
  const response = await agent
   .post("/items")
   .send({ name: newItemName })
   .expect(201); // Assert HTTP status code

  expect(response.body.id).toBeDefined();
  expect(response.body.name).toBe(newItemName);

  // Verify directly in the DB for extra confidence
  const dbResult = await db
   .select()
   .from(schema.items)
   .where(eq(schema.items.id, response.body.id));
  expect(dbResult.length).toBe(1);
  expect(dbResult[0].name).toBe(newItemName);
 });

 it("GET /items/:id should retrieve an existing item", async () => {
  // Arrange: Insert an item directly
  const insertResult = await db
   .insert(schema.items)
   .values({ name: "Get Me PGLite" })
   .returning();
  const itemId = insertResult[0].id;

  // Act: Request the item
  const response = await agent.get(`/items/${itemId}`).expect(200);

  // Assert
  expect(response.body.id).toBe(itemId);
  expect(response.body.name).toBe("Get Me PGLite");
 });

 it("GET /items/:id should return 404 for non-existent item", async () => {
  await agent.get("/items/99999").expect(404);
 });

 it("POST /items should return 400 if name is missing", async () => {
  await agent.post("/items").send({}).expect(400);
 });
});
```

**5. Run Your Tests:**

```bash
npm test
```

### The Need for Speed: Benchmarks

Okay, talk is cheap. Let's look at some rough numbers. These were run on my machine, so your mileage may wary, but they illustrate the difference.

**Testcontainers (with snapshotting after initial setup):**

```
> vitest run
... (Container startup logs) ...
 ✓ src/server.test.ts (4 tests) 4141ms
   ✓ Items API > POST /items should create an item  459ms
   ✓ Items API > GET /items/:id should retrieve an existing item 159ms
   ✓ Items API > GET /items/:id should return 404 for non-existent item 132ms
   ✓ Items API > POST /items should return 400 if name is missing 154ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Duration  4.45s (transform 32ms, setup 0ms, collect 159ms, tests 4.14s, environment 0ms, prepare 40ms)
```

Using `hyperfine` for 10 runs:

```

Time (mean ± σ):      4.773 s ±  0.412 s    [User: 2.730 s, System: 0.991 s]
Range (min … max):    4.333 s …  5.772 s    10 runs
```

**PGLite (fresh instance per test, no optimizations yet):**

```
> vitest run
 ✓ src/server.test.ts (4 tests) 2146ms
   ✓ Items API > POST /items should create an item  576ms
   ✓ Items API > GET /items/:id should retrieve an existing item  503ms
   ✓ Items API > GET /items/:id should return 404 for non-existent item  516ms
   ✓ Items API > POST /items should return 400 if name is missing  550ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
   Duration  2.58s (transform 39ms, setup 0ms, collect 273ms, tests 2.15s, environment 0ms, prepare 42ms)
```

Using `hyperfine` for 10 runs:

```
Time (mean ± σ):      2.822 s ±  0.070 s    [User: 5.768 s, System: 1.766 s]
Range (min … max):    2.748 s …  2.946 s    10 runs
```

**The Verdict (Round 1):** PGLite is already significantly faster overall (~2.8s vs ~4.8s) mainly because we skip the Docker container boot time entirely! Each test takes a bit longer individually because it's initializing PGLite *and* applying the schema every time. Can we do better?

### Even Faster? Enter reusable database

Just like with Testcontainers, we can optimize further by setting up the database *once*, applying the schema, taking a "snapshot," and then quickly restoring that clean state before each test. PGLite doesn't have a direct snapshot command, but we can achieve a similar effect with some filesystem manipulation.

> I am still experimenting with this approach, it does net us some speed benefits but I cannot guarantee its complete correctness and safety, it's easy to setup so you can try it out but be wary. If you have any ideas how to improve this implementation please leave a comment or hit me up somewhere
{.warning}

**1. Update the DB Helper:**

Add these functions to `src/testing/db-helper.ts`:

```typescript
import type { PGlite } from "@electric-sql/pglite";

export async function snapshot(client: PGlite) {
    return client.dumpDataDir("none");
}

export async function restoreSnapshot(snapshot: File | Blob): Promise<PGlite> {
    const clone = new File([Buffer.from(await snapshot.arrayBuffer())], "snapshot", { type: snapshot.type });
    return new PGlite({ loadDataDir: clone });
}
```

**2. Modify the Test Setup:**

We'll now use `beforeAll` to set up *one* PGLite instance for the entire test file, apply the schema, and take the snapshot. `beforeEach` will simply restore the snapshot.

`src/server.test.ts` (Changes highlighted)

```typescript
import {
 // ... other imports
 beforeAll, 
 afterAll, 
 // ...
} from "vitest";
// ... other imports
import { pushSchema, snapshot, restoreSnapshot } from "./testing/db-helper"; 

vi.mock("./db");

describe("Items API with PGLite", () => {
 let db: PgDatabase<any, typeof schema>;
 let client: PGlite;
 let agent: TestAgent;
 let snapshottedDB: File | Blob;

 beforeAll(async () => {
  client = new PGlite(
   // You might want to specify a consistent temp directory:
   // dataDir: `./.pglite-test-data/${dbName}` or `/tmp`
  );

  // Apply the schema once
  await pushSchema(client);

  // Take a snapshot
  snapshottedDB = await snapshot(client);
 });

 // Setup a fresh PGLite instance and apply schema before each test
 beforeEach(async () => {
  agent = request(app); // Create supertest agent

  client = await restoreSnapshot(snapshottedDB)

  // Create the Drizzle instance connected to PGLite
  db = drizzle(client, { schema });

  // Mock getDB() to return *this specific test's* Drizzle instance
  // Use `vi.mocked` for type safety
  vi.mocked(getDB).mockReturnValue(db as any);
 });

 // Close the PGLite client and clean up mocks after each test
 afterEach(async () => {
  await client.close(); // Release resources
  vi.clearAllMocks(); // Reset mocks
 });

 // Consider adding cleanup for the data directory if you specified one
 // afterAll

 it("POST /items should create an item", async () => {
  const newItemName = "Test Item PGLite";
  const response = await agent
   .post("/items")
   .send({ name: newItemName })
   .expect(201); // Assert HTTP status code

  expect(response.body.id).toBeDefined();
  expect(response.body.name).toBe(newItemName);

  // Verify directly in the DB for extra confidence
  // And to make sure the snapshot is working
  const dbResult = await db
   .select()
   .from(schema.items);

  expect(dbResult.length).toBe(1);
  expect(dbResult[0].name).toBe(newItemName);
 });

 it("GET /items/:id should retrieve an existing item", async () => {
  // Arrange: Insert an item directly
  const insertResult = await db
   .insert(schema.items)
   .values({ name: "Get Me PGLite" })
   .returning();
  const itemId = insertResult[0].id;

  // Act: Request the item
  const response = await agent.get(`/items/${itemId}`).expect(200);

  // Verify directly in the DB for extra confidence
  // And to make sure the snapshot is working
  const dbResult = await db
   .select()
   .from(schema.items);

  expect(dbResult.length).toBe(1);
  expect(dbResult[0].name).toBe("Get Me PGLite");

  // Assert
  expect(response.body.id).toBe(itemId);
  expect(response.body.name).toBe("Get Me PGLite");
 });
});
```

**Let's run the benchmarks again!**

**PGLite (with reusable database):**

```
➜ npm run test

> vitest run

 ✓ src/server.test.ts (4 tests) 1331ms
   ✓ Items API with PGLite > POST /items should create an item 199ms
   ✓ Items API with PGLite > GET /items/:id should retrieve an existing item 192ms
   ✓ Items API with PGLite > GET /items/:id should return 404 for non-existent item 182ms
   ✓ Items API with PGLite > POST /items should return 400 if name is missing 189ms
```

Using `hyperfine` for 10 runs:

```
Time (mean ± σ):      1.940 s ±  0.022 s    [User: 5.570 s, System: 1.289 s]
Range (min … max):    1.911 s …  1.987 s    10 runs
```

[**5 BIG BOOMS FOR THIS SPEED!**](https://www.youtube.com/watch?v=KEc8BQK-rbA) Look at that difference! From ~4.8s with Testcontainers down to ~1.3s with PGLite snapshotting. This is the kind of speed that keeps you in the TDD flow state.

One big caveat is that these are still running sequentially, so with a lot of tests, it could add up and become even slower than the `testcontainers` version. I'll make a follow up post on how we could approach architecting our code to make sure it's testable and parallelizable. Since the current approach relies on mocking global accessors like `getDB` which make it impossible to parallelize.

### Final Thoughts & Caveats

1. **Testcontainers is Still Awesome:** Let me be clear, `testcontainers` is a phenomenal tool. For testing against other services (Redis, Kafka, other databases) or when you need the *absolute highest fidelity* with a specific Postgres version/extension setup provided by Docker, it's still my go-to. This PGLite approach is specifically optimized for speeding up the Node.js <-> Postgres interaction during TDD experimentation sessions.
2. **PGLite Fidelity:** While PGLite *is* running actual Postgres compiled to WASM, it's newer technology. Be mindful that there *could* be subtle differences or unsupported features compared to a native Postgres installation or a Docker image. I haven't hit any issues yet for typical web app CRUD operations, but always test thoroughly and be aware of the possibility. Don't let flaky tests give you false confidence.
3. **Potentionally slower** for larger test suites. I am still not certain that this approach scales better than the `testcontainers` + snapshotting, since the bottleneck in that case is the startup time of the container, while subsequent tests run quite quickly. Not sure if the `fs` operations and the spinning up of `PGlite` instances will actually prove to be a bigger bottleneck than the initial startup time of `testcontainers`. But the big benefit of this approach is that it allows parallel execution of a test suite.

This PGLite + Drizzle + Vitest combination, especially with the snapshotting(-ish) technique, has significantly improved my Node.js TDD experience. It brings back that snappy, responsive feeling I loved from Go (with `PGlite` it's even better than Go), making test-driven development truly enjoyable and productive.

---

**Give these amazing projects a star!** This workflow wouldn't be possible without them:

* [pglite](https://github.com/electric-sql/pglite) ⭐
* [drizzle-orm](https://github.com/drizzle-team/drizzle-orm) ⭐
* [vitest](https://github.com/vitest-dev/vitest) ⭐

**Bonus Cool Project:** *(Not sponsored, just want to make it a habit to share a cool project with every post)*

* [TinaCMS](https://github.com/tinacms/tinacms): An open-source, Git-backed headless CMS with visual editing. Really interesting approach if you need content management.
