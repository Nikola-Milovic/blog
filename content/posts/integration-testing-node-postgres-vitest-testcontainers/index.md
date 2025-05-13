---
date: 2025-04-15T17:43:38+02:00
draft: false
tags: ["nodejs", "postgres", "testing","vitest","integration testing"]
title: Integration Testing Node.js Postgres interaction Like You Mean It (with Vitest & Testcontainers)
cover: 
  image: cover.png
  alt: integration testing with postgres and nodejs
  hidden: true
---

# Integration Testing Node.js Postgres interaction Like You Mean It (with Vitest & Testcontainers)

Look, unit tests are great for checking isolated logic. But eventually, you need to know if your code _actually_ works with a real database, message queue, or whatever external service you depend on. Mocking everything is fragile and often hides bugs that only show up when things get real. Shared dev databases? A recipe for flaky tests and stepping on colleagues' toes.

Coming from Go, I got used to writing integration tests that gave me real confidence. I wanted that same feeling in my TypeScript projects. The answer? **Testcontainers**.

## The Core Idea: Real Dependencies, Isolated Tests

The plan is simple:

1. **Spin up real dependencies** (like Postgres, Redis, etc.) inside Docker containers _for your tests_.
2. Use a library like [Testcontainers for Node.js](https://github.com/testcontainers/testcontainers-node) to manage these containers programmatically.
3. Run your migrations or setup scripts against the fresh container to establish a baseline state.
4. **Snapshot** the clean, migrated database state (if your DB module supports it, like `@testcontainers/postgresql`). Alternatively, investigate the `docker commit` method if snapshotting isn't built-in.
5. For each test (or test suite), **restore** from that snapshot to get a perfectly clean slate, ensuring test isolation.
6. Run your test logic against the application, which talks to the containerized dependency.
7. Tear down the container(s) when done.

This gives you the best of both worlds: the realism of testing against the actual database software you use in production, and the isolation needed for reliable, repeatable tests.

**Why not just use SQLite or an in-memory DB?**

Because it's _not the same_. You might use database-specific features, syntax quirks, or transaction behaviors that SQLite doesn't replicate. Testing against a different database than production is asking for trouble.

> Never use a different DB for development/testing than what you use in production. If you're using Postgres in prod, use Postgres containers for testing. Don't cheat with SQLite just because it seems easier. Your ORM won't save you from every difference.
{.warning}

## Let's Build It: A Simple CRUD Example

We'll set up a dead-simple Express app with basic CRUD operations for "items" stored in a Postgres database. Then, we'll write integration tests for it using Vitest and Testcontainers.

**(You can find the [complete project code over at github](https://github.com/Nikola-Milovic/blog-projects/tree/master/integration-testing-node-vitest-testcontainers))**

**1. Project Setup**

Make sure you have Node.js and Docker installed.

```bash
mkdir node-testcontainers-example
cd node-testcontainers-example
npm init -y
npm install express pg # Or your preferred framework/DB driver
npm install -D typescript @types/node @types/express @types/pg ts-node nodemon vitest testcontainers @testcontainers/postgresql # Dev dependencies
# Initialize tsconfig.json
npx tsc --init --rootDir src --outDir dist --esModuleInterop --resolveJsonModule --lib esnext --module nodenext --allowJs true --noImplicitAny true
```

Create a simple `src/server.ts`:

```typescript
import express, { type Response, type Request } from "express";
import { getDB } from "./db";

const app = express();
app.use(express.json());

app.post("/items", async (req: Request, res: Response) => {
 const { name } = req.body;
 if (!name) {
  res.status(400).send({ error: "Name is required" });
  return;
 }
 try {
  const db = getDB();
  const result = await db.query(
   "INSERT INTO items(name) VALUES($1) RETURNING *",
   [name],
  );
  res.status(201).send(result.rows[0]); 
 } catch (err) {
  console.error(err);
  res.status(500).send({ error: "Failed to create item" }); 
 }
});

app.get("/items/:id", async (req: Request, res: Response) => {
 const { id } = req.params;
 try {
  const db = getDB();
  const result = await db.query("SELECT * FROM items WHERE id = $1", [id]);
  if (result.rows.length === 0) {
   res.status(404).send({ error: "Item not found" });
   return;
  }
  res.send(result.rows[0]); 
 } catch (err) {
  console.error(err);
  res.status(500).send({ error: "Failed to retrieve item" }); 
 }
});

// Only start listening if the file is run directly
if (require.main === module) {
 const port = process.env.PORT || 3000;
 app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
  // You'd typically run migrations here on real app startup
  // For this example, we assume the table exists or is created by tests/manually
 });
}

export default app;
```

And `src/db.ts`:

```typescript
import { Pool } from 'pg';

// Default connection for manual running or real deployment
const pool = new Pool({
 connectionString: process.env.DATABASE_URL || 'postgresql://postgres:postgres@localhost:5432/postgres',
});

// We export a function to get the pool. This allows us to easily mock
// it in tests to point to our Testcontainers-managed database instead.
export const getDB = () => pool;
```

**Manual Verification (Optional)**

To verify our server manually, we'd first need a running Postgres instance and the `items` table created. You could achieve this using Docker:

```bash
# Start a Postgres container
docker run -d \
  --name my-pg-container \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  postgres:17

# Execute the CREATE TABLE command inside the container
docker exec -it my-pg-container psql -U postgres -d postgres -c "CREATE TABLE IF NOT EXISTS items (id SERIAL PRIMARY KEY, name VARCHAR(100) NOT NULL);"
```

Then, add a start script to `package.json`:

```json
  "scripts": {
    "start": "tsc && node ./dist/server.js"
  }
```

Start the server (`npm start`) and use `curl` to interact with it:

```shell
➜ npm start
> tsc && node ./dist/server.js

Server listening on port 3000

➜ curl -X POST http://localhost:3000/items -H "Content-Type: application/json" -d '{"name": "do the dishes"}'
> {"id":1,"name":"do the dishes"}

➜ curl http://localhost:3000/items/1
> {"id":1,"name":"do the dishes"}
```

This works, but manually managing the database for testing is cumbersome and error-prone. Let's automate it. (Remember to stop and remove the manual container: `docker stop my-pg-container && docker rm my-pg-container`)

## Automated Testing with Testcontainers

Add a `vitest.config.ts`:

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true, // Use Vitest globals (describe, it, etc.)
    setupFiles: [], // Optional: setup files for tests
    environment: 'node', // Specify Node environment
    // Increase timeout for container startup, snapshotting etc.
    testTimeout: 60000, // 60 seconds
    hookTimeout: 60000, // 60 seconds for hooks too
  },
});
```

**2. The Test Helper (`PostgresContainerManager`)**

Managing the container lifecycle (start, migrate, snapshot, restore, stop) and ensuring a clean database state for each test can be repetitive. Let's encapsulate this logic in a `PostgresContainerManager` helper class. This class will handle starting a Postgres container, running initial migrations within it, creating a baseline snapshot of that state, and providing functions to get a fresh database connection reset to that baseline for each test.

_Heads up:_ The snapshot feature requires `testcontainers` version `10.23.0` or later for the Postgres module (check your `package.json`!).

```typescript
// src/testing/db-helper.ts
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Client, Pool, type PoolClient } from 'pg';

// Interface for what setupTestDatabase will return
export interface TestDatabase {
 db: PoolClient; // Test should use this client
 container: StartedPostgreSqlContainer; // Reference to the container
 cleanup: () => Promise<void>; // Function to release the client/pool
}

// Manages the PostgreSQL container lifecycle for tests
export class PostgresContainerManager {
 private static instance: PostgresContainerManager | null = null;
 private container: StartedPostgreSqlContainer | null = null;
 private snapshotName = 'clean-db-snapshot'; // Name for our baseline snapshot

 // Singleton pattern to potentially share container across test suites (though we scope it per-file here)
 private constructor() { }

 public static getInstance(): PostgresContainerManager {
  if (!PostgresContainerManager.instance) {
   PostgresContainerManager.instance = new PostgresContainerManager();
  }
  return PostgresContainerManager.instance;
 }

 // Starts container, runs migrations *in* the container DB, takes snapshot
 async initialize(): Promise<void> {
  if (this.container) {
   console.log('Container already initialized.');
   return;
  }

  console.log('Starting PostgreSQL container...');
  this.container = await new PostgreSqlContainer('postgres:17')
   .withDatabase('test') // Use a specific DB name for testing
   .withUsername('test-user')
   .withPassword('test-password')
   .withExposedPorts(5432)
   .start();
  console.log(`Container started on port ${this.container.getMappedPort(5432)}`);

  // Connect directly to the containerized database to run migrations
  const migrationClient = new Client({ connectionString: this.container.getConnectionUri() });
  await migrationClient.connect();
  try {
   console.log("Running migrations...");
   // In a real app, use your migration tool (node-pg-migrate, TypeORM migrations, etc.)
   // For this example, we create the table directly:
   await migrationClient.query(`
                CREATE TABLE items (
                    id SERIAL PRIMARY KEY,
                    name VARCHAR(100) NOT NULL
                );
            `);
   console.log('Migrations complete.');
  } catch (error) {
   console.error('Migration failed:', error);
   throw error; // Fail fast if migrations don't work
  } finally {
   await migrationClient.end();
  }

  // Take a snapshot of the database state *after* migrations
  console.log(`Taking snapshot '${this.snapshotName}'...`);
  await this.container.snapshot(this.snapshotName);
  console.log('Snapshot taken.');
 }

 // Restores the 'clean' snapshot and provides a connection pool/client
 async setupTestDatabase(): Promise<TestDatabase> {
  if (!this.container) {
   throw new Error('Container not initialized. Call initialize() first.');
  }

  try {
   // Restore the database to the state captured in the snapshot
   console.log(`Restoring snapshot '${this.snapshotName}'...`);
   await this.container.restoreSnapshot(this.snapshotName);
   console.log('Snapshot restored.');

   // Create a *new pool* connecting to the restored database for this test
   // This ensures connection isolation if tests run concurrently within the same file (though less common)
   const testPool = new Pool({ connectionString: this.container.getConnectionUri() });
   const testClient = await testPool.connect(); // Get a client for the test

   // Cleanup function specific to this test's pool/client
   const cleanup = async () => {
    try {
     await testClient.release(); // Release client back to pool
     await testPool.end();      // Close the pool
    } catch (error) {
     console.error('Error during test DB cleanup:', error);
    }
   };

   return {
    db: testClient, // Provide the client to the test
    container: this.container,
    cleanup,
   };
  } catch (error) {
   console.error('Error setting up test database:', error);
   throw error;
  }
 }

 // Stops and removes the container
 async teardown(): Promise<void> {
  if (this.container) {
   console.log('Stopping PostgreSQL container...');
   await this.container.stop();
   this.container = null;
   console.log('Container stopped.');
  }
  PostgresContainerManager.instance = null; // Reset singleton state
 }
}
```

**3. Writing the Integration Test**

Now, let's write the integration test using Vitest and our helper. We'll use `supertest` to make HTTP requests to our Express app. The key parts are:

- Mocking our `db.ts` module so the application uses the test database connection provided by the helper (`testDb.db`).
- Using Vitest hooks (`beforeAll`, `afterAll`, `beforeEach`, `afterEach`) to manage the container lifecycle and ensure test isolation via snapshot restoration.
- Note: Using `beforeAll`/`afterAll` within the test file means Vitest's default parallel execution (by file) will likely spin up one independent container _per test file_, providing strong isolation but potentially using more resources.

```typescript
// src/server.test.ts
import { afterAll, afterEach, beforeAll, beforeEach, describe, expect, it, vi } from 'vitest';
import request from 'supertest'; // For making HTTP requests
import type TestAgent from 'supertest/lib/agent'; // Type for supertest agent
import app from './server'; // Our Express app
import { getDB } from './db'; // The function we need to mock
import { PostgresContainerManager, type TestDatabase } from './testing/db-helper'; // Our helper

// This is crucial: tell Vitest to replace the real './db' module
// with our mock, so we can control what getDB() returns in tests.
vi.mock('./db');

describe('Items API', () => {
 const containerManager = PostgresContainerManager.getInstance();
 let testDb: TestDatabase; // Will hold the connection/cleanup func for each test
 let agent: TestAgent;     // Supertest agent for making requests

 // Start the single container ONCE before all tests in this file
 beforeAll(async () => {
  await containerManager.initialize();
  agent = request(app); // Create supertest agent targeting our app
 }, 60000); // Increase timeout for container init

 // Restore snapshot and get a fresh DB connection BEFORE EACH test
 beforeEach(async () => {
  testDb = await containerManager.setupTestDatabase();
  // Point the mocked getDB function to return our test database client
  vi.mocked(getDB).mockReturnValue(testDb.db); 
 });

 // Clean up the test database connection AFTER EACH test
 afterEach(async () => {
  await testDb.cleanup(); // Release pool client and end pool
  vi.clearAllMocks();     // Reset mocks between tests
 });

 // Stop the single container ONCE after all tests in this file are done
 afterAll(async () => {
  await containerManager.teardown();
 }, 60000); // Increase timeout for container teardown

 it('POST /items should create an item', async () => {
  const newItemName = 'Test Item 1';
  const response = await agent
   .post('/items')
   .send({ name: newItemName })
   .expect(201); // Assert HTTP status code

  // Assert response body
  expect(response.body.id).toBeDefined();
  expect(response.body.name).toBe(newItemName);

  // Optional but recommended: Verify directly in the DB
  const dbResult = await testDb.db.query('SELECT * FROM items WHERE id = $1', [response.body.id]);
  expect(dbResult.rows.length).toBe(1);
  expect(dbResult.rows[0].name).toBe(newItemName);
 });

 it('GET /items/:id should retrieve an existing item', async () => {
  // Arrange: Insert an item directly using the test DB client
  const insertResult = await testDb.db.query("INSERT INTO items(name) VALUES('Get Me') RETURNING id");
  const itemId = insertResult.rows[0].id;

  // Act: Make request to the API endpoint
  const response = await agent
   .get(`/items/${itemId}`)
   .expect(200);

  // Assert
  expect(response.body.id).toBe(itemId);
  expect(response.body.name).toBe('Get Me');
 });

 it('GET /items/:id should return 404 for non-existent item', async () => {
  await agent
   .get('/items/99999') // Use an ID that almost certainly won't exist
   .expect(404);
 });

 it('POST /items should return 400 if name is missing', async () => {
  await agent
   .post('/items')
   .send({}) // Send empty body
   .expect(400);
 });
});
```

**4. Running the Tests**

Add test scripts to your `package.json`:

```json
{
  "scripts": {
    "start": "tsc && node ./dist/server.js",
    "test": "vitest run",
    "test:watch": "vitest"
    // ... other scripts
  }
}
```

Now run:

```bash
npm test
```

You should see Vitest start, the helper log messages about starting the container, running migrations, taking/restoring snapshots, and finally the test results passing.

## Leveling Up

**1. Speeding Up Slow Migrations**

If your migrations take a _long_ time to run, the initial `initialize` step can become a bottleneck. The snapshot helps for subsequent test _runs_, but the first one is still slow.

**Solution:** Pre-build a Docker image with migrations already applied.

1. Start a standard Postgres container manually: `docker run -d --name postgres-migrated -e POSTGRES_PASSWORD=password postgres:17`
2. Connect to it and run your migrations.
3. Commit the container state to a new image: `docker commit postgres-migrated my-app/postgres-migrated:latest`
4. Stop and remove the temporary container: `docker stop postgres-migrated && docker rm postgres-migrated`
5. In your `PostgresContainerManager`, change the image name:

    ```typescript
    // In initialize()
    this.container = await new PostgreSqlContainer('my-app/postgres-migrated:latest') // Use your committed image
        // .withDatabase(...) // Ensure user/pass match how you set it up
        // ... rest of setup ...
        .start();
    
    // You can likely SKIP the migration running step and snapshotting
    // if the image already represents the clean, migrated state.
    // Just start the container and proceed to setupTestDatabase.
    // You might need a different helper logic for pre-built images.
    // Remember to adjust initialize() to skip migrations/snapshotting if using this.
    ```

> docker commit creates an image from a container's filesystem state. It's generally better for capturing installed software or data setup (like migrations) than for runtime state. Check the Docker commit docs for details.
{.note}

**Trade-offs:** Faster test startup vs. managing another custom Docker image in your build process.

**2. Running Tests in Parallel**

Vitest runs test _files_ in parallel by default. Because our `PostgresContainerManager` setup (`beforeAll`/`afterAll`) is scoped _within_ a test suite, each test suite that follows this pattern will get its _own_ independent Docker container.

- **Pro:** True parallelism, maximum isolation between test suites.
- **Con:** Resource intensive. If you have many test files, you'll be running many database containers simultaneously. Make sure your machine (and CI environment) can handle it.

If resource usage is a major concern, you _could_ explore:

- **Shared Container, Schemas/DBs per Test File:** Modify the `PostgresContainerManager` to be a true singleton across the entire test run (e.g., using Vitest's global setup). In `setupTestDatabase`, instead of restoring a snapshot (which resets the whole DB), you might create a unique _schema_ or entirely new database _within the single container_ for each test file (or even each `describe` block).
- **Vitest `--no-parallel` or `--max-workers`:** Force sequential execution or limit concurrency if parallelism causes issues or consumes too many resources.
- Using technologies like [PGLite](https://pglite.dev/), (stay tuned for the next post ;) )

## Pros and Cons Recap

**Pros:**

- **High Confidence:** You're testing against the real database system.
- **Catches Integration Bugs:** Finds issues related to database interactions, transactions, constraints, etc., that mocks miss.
- **Realistic Environment:** Mimics production dependencies more closely.
- **Isolated & Repeatable:** Each test gets a clean slate via snapshot restore, avoiding flakiness.

**Cons:**

- **Slower than Unit Tests:** Starting containers and interacting with a real DB takes more time.
- **Resource Usage:** Docker containers consume CPU, RAM, and disk space, especially when ran in parallel.
- **Dependency on Docker:** Requires Docker to be installed and running where tests execute (local dev, CI).

## Conclusion

Setting up integration tests with Testcontainers is something I've been incorporating into every project I am starting nowadays with Node.js, it gives me great velocity while developing and ease of mind when refactoring. The payoff in confidence and catching real-world bugs is huge. By managing container lifecycles programmatically and ensuring clean state via snapshots or unique databases per test, you can build a robust, reliable integration test suite for your Node.js applications. I'd even argue that this is actually simpler than mocking since there isn't a good way (that I am aware of) to properly mock these database interactions without using an ORM that supports it or mocking the data layer representation you're using (`store`/`repository`/`dao`'s)

Stop guessing if your database interaction code works – test it properly.

---

**Give these amazing projects a star!** This workflow wouldn't be possible without them:

- [vitest](https://github.com/vitest-dev/vitest) ⭐
- [testcontainers-node](https://github.com/testcontainers/testcontainers-node) ⭐

**Bonus Cool Project:** _(Not sponsored, just want to make it a habit to share a cool project with every post)_

- [WebTUI](https://github.com/webtui/webtui): An open-source, modular CSS Library that brings the beauty of Terminal UIs to the browser
