---
title: "Module 7: Testing, Real-Time & Background Jobs"
description: "Test Node.js and Bun APIs with node:test, bun test and a real Postgres database; add live updates with Server-Sent Events and WebSockets; scale real-time with Redis pub/sub; and move slow work into reliable background jobs with BullMQ."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/nodejs-module-7.png"
tags: [Node.js, Bun, Testing, WebSockets, SSE, BullMQ, Background-Jobs, Interview-Prep]
keywords: ["node:test tutorial", "bun test", "Node.js integration testing Postgres", "Server-Sent Events Express", "WebSocket ws Node.js", "Bun WebSocket pub/sub", "BullMQ tutorial", "Node.js background jobs", "scaling WebSockets Redis"]
---

# Module 7: Testing, Real-Time & Background Jobs

![Node.js Module 7](/images/nodejs-module-7.png)

## Words you will meet in this module

| Word                      | Simple meaning                                                                                      |
| ------------------------- | --------------------------------------------------------------------------------------------------- |
| **Unit test**             | Tests one small piece (a function, a service) in isolation.                                         |
| **Integration test**      | Tests several real parts together: HTTP route + service + real database.                            |
| **Fixture / factory**     | Code that creates known test data (a user, a project) quickly.                                      |
| **Mock / stub / fake**    | A stand-in for a real dependency (an email sender, the clock) that you control in a test.           |
| **Flaky test**            | A test that sometimes passes and sometimes fails with no code change. Poison for trust.             |
| **SSE**                   | Server-Sent Events: one-way, server-to-browser stream of events over plain HTTP.                     |
| **WebSocket**             | A two-way, long-lived connection between browser and server, started with an HTTP upgrade.          |
| **Pub/Sub**               | Publish/Subscribe: senders publish to a channel, every subscriber receives it. Redis supports it.   |
| **Job queue**             | A durable list of tasks that separate **worker** processes pick up and run.                         |
| **Worker**                | A process (not a request handler) that runs queued jobs.                                            |
| **At-least-once**         | A delivery guarantee: a job may run **more than once**, so handlers must be idempotent.             |
| **Dead-letter queue**     | Where jobs go after exhausting retries, so you can inspect and replay them.                         |
| **Webhook**               | An HTTP call another service makes to **your** API when something happens (a payment succeeded).    |

## 7.1: Testing with `node:test`, `bun test` and Vitest

### Why tests?

Tests are the safety net that lets you change code (upgrade Node, refactor, add features) without fear. A good suite runs in seconds, fails only when behavior is really broken, and is simple to read as documentation.

### The three choices

| Runner          | Install                    | Style                                  | Notes                                                                  |
| --------------- | -------------------------- | -------------------------------------- | ---------------------------------------------------------------------- |
| **`node:test`** | Built into Node.js         | `describe` / `it`, `node:assert`       | Zero dependencies. Stable. Good default for Node services              |
| **`bun test`**  | Built into Bun             | Jest-compatible (`expect`, `mock`)     | Very fast. Great if you run on Bun                                     |
| **Vitest**      | `npm i -D vitest`          | Jest-compatible, rich tooling          | Same runner as your React app (Part 1). Runs on Node                   |

They are all similar. Pick one per project.

### `node:test`

```ts
// src/lib/slug.test.ts
import { describe, it } from "node:test";
import assert from "node:assert/strict";
import { slugify } from "./slug.ts";

describe("slugify", () => {
  it("lowercases and replaces spaces", () => {
    assert.equal(slugify("Hello World"), "hello-world");
  });

  it("removes characters that are not url-safe", () => {
    assert.equal(slugify("Café & Co!"), "cafe-co");
  });

  it("rejects empty input", () => {
    assert.throws(() => slugify(""), { message: /empty/i });
  });
});
```

```bash
node --test                          # finds *.test.* files and test/ directories
node --test --watch                  # re-run on change
node --test --experimental-test-coverage     # coverage (the flag name has changed over versions: check `node --help`)
node --test --test-name-pattern="slugify"
```

Async tests, setup and teardown, and mocking:

```ts
import { describe, it, before, after, beforeEach, mock } from "node:test";

describe("emailService", () => {
  it("retries once when the provider fails", async () => {
    const send = mock.fn(async () => { throw new Error("provider down"); });
    send.mock.mockImplementationOnce(async () => { throw new Error("provider down"); }, 0);   // first call fails
    // ... call the service with `send` injected, then:
    assert.equal(send.mock.callCount(), 2);
  });

  it("does not wait for real time", async (t) => {
    t.mock.timers.enable({ apis: ["setTimeout"] });      // fake timers
    const done = delayedGreeting();                      // uses setTimeout(…, 5000)
    t.mock.timers.tick(5000);                            // jump 5 seconds instantly
    assert.equal(await done, "hello");
  });
});
```

### `bun test`

```ts
import { describe, test, expect, mock, beforeAll } from "bun:test";

describe("slugify", () => {
  test("lowercases and replaces spaces", () => {
    expect(slugify("Hello World")).toBe("hello-world");
  });
});
```

```bash
bun test              # also: bun test --watch, bun test --coverage
```

### What makes a test good

| Do                                                                    | Avoid                                                                 |
| --------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Test **behavior** (inputs → outputs, status codes, stored data)       | Testing private details so that every refactor breaks tests           |
| One reason to fail per test, a clear name (`rejects expired sessions`) | Giant tests with 15 assertions                                      |
| **Arrange, Act, Assert** structure                                    | Hidden shared state between tests                                     |
| Independent tests that run in any order, even in parallel             | Tests that depend on the previous test's data                         |
| Control **time and randomness** (inject a clock, use fake timers)     | `setTimeout(…, 2000)` "wait for it" sleeps                            |
| Cover **failure paths**: invalid input, missing auth, DB errors        | Only the happy path                                                   |

## 7.2: Designing Code So It Can Be Tested

You already did the key things in Module 4: **`createApp()` separate from `listen()`**, services with no HTTP knowledge, and repositories behind a small interface. Two more habits make testing painless.

### Pass dependencies in (no framework needed)

```ts
// A service that depends on an email sender and a clock, provided by the caller
export function createInviteService(deps: {
  sendEmail: (to: string, body: string) => Promise<void>;
  now: () => Date;
}) {
  return {
    async invite(email: string) {
      const expires = new Date(deps.now().getTime() + 24 * 3600_000);
      await deps.sendEmail(email, `Join us before ${expires.toISOString()}`);
      return { email, expires };
    },
  };
}

// production: createInviteService({ sendEmail: realProviderSend, now: () => new Date() })
// test:       createInviteService({ sendEmail: fakeSend, now: () => new Date("2026-11-04T10:00:00Z") })
```

This is **dependency injection** in its simplest form: function parameters. Heavy DI containers are rarely needed in Node.

### Fakes for the outside world, the real thing for your database

| Dependency                                        | In tests                                                                    |
| ------------------------------------------------- | --------------------------------------------------------------------------- |
| Email, SMS, payment providers, third-party APIs   | **Fake** (a recording stub) or intercept the network (MSW / undici `MockAgent`) |
| The clock and randomness                          | **Inject** or use fake timers                                               |
| **PostgreSQL**                                    | **The real database** (a test instance), not a mock                         |
| Redis                                             | A real local Redis, or a thin fake for pure unit tests                      |

> **Mocking the database is a trap.** SQL behavior (constraints, `RETURNING`, locking, JSON, collation) is exactly what breaks in production, and a mock cannot catch it. Use a real Postgres, started with `docker compose` locally and as a **service container** in CI. (**Testcontainers** can start throwaway containers from the test code itself.)

## 7.3: Integration Tests Against HTTP and a Real Database

### A test database and a reset helper

```ts
// test/helpers.ts
import { once } from "node:events";
import type { AddressInfo } from "node:net";
import type { Server } from "node:http";
import { createApp } from "../src/app.ts";
import { pool } from "../src/db/pool.ts";

export async function resetDb() {
  await pool.query("TRUNCATE sessions, tasks, projects, users RESTART IDENTITY CASCADE");
}

export async function startServer() {
  const server: Server = createApp().listen(0);            // port 0 = the OS picks a free port
  await once(server, "listening");
  const { port } = server.address() as AddressInfo;
  return { server, baseUrl: `http://127.0.0.1:${port}` };
}

// A tiny HTTP client that remembers the session cookie, like a browser
export function createClient(baseUrl: string) {
  let cookie = "";
  return {
    async request(method: string, path: string, body?: unknown) {
      const res = await fetch(baseUrl + path, {
        method,
        headers: { "content-type": "application/json", ...(cookie ? { cookie } : {}) },
        body: body === undefined ? undefined : JSON.stringify(body),
      });
      const setCookie = res.headers.getSetCookie()[0];
      if (setCookie) cookie = setCookie.split(";")[0];     // keep "tf_sid=…"
      const text = await res.text();
      return { status: res.status, body: text ? JSON.parse(text) : null, headers: res.headers };
    },
  };
}
```

Use `.env.test` (a **separate database name**, e.g. `taskflow_test`) and **never point tests at a development or production database**, because `TRUNCATE` is destructive. Fail loudly if `NODE_ENV !== "test"`.

### The tests

```ts
// test/tasks.test.ts
import { describe, it, before, after, beforeEach } from "node:test";
import assert from "node:assert/strict";
import { startServer, createClient, resetDb } from "./helpers.ts";
import { pool } from "../src/db/pool.ts";

let ctx: Awaited<ReturnType<typeof startServer>>;
before(async () => { ctx = await startServer(); });
after(async () => { ctx.server.close(); await pool.end(); });
beforeEach(resetDb);

async function signUp(email: string) {
  const c = createClient(ctx.baseUrl);
  const res = await c.request("POST", "/api/auth/register", { email, name: "Test", password: "correct horse battery" });
  assert.equal(res.status, 201);
  return c;
}

describe("tasks API", () => {
  it("requires authentication", async () => {
    const res = await createClient(ctx.baseUrl).request("GET", "/api/projects");
    assert.equal(res.status, 401);
  });

  it("creates and lists tasks for the owner", async () => {
    const alice = await signUp("alice@example.com");
    const project = (await alice.request("POST", "/api/projects", { name: "Site" })).body;

    const created = await alice.request("POST", `/api/projects/${project.id}/tasks`, { title: "Buy milk" });
    assert.equal(created.status, 201);

    const list = await alice.request("GET", `/api/projects/${project.id}/tasks`);
    assert.deepEqual(list.body.map((t: any) => t.title), ["Buy milk"]);
  });

  it("rejects invalid input with 422 and a machine-readable code", async () => {
    const alice = await signUp("alice@example.com");
    const project = (await alice.request("POST", "/api/projects", { name: "Site" })).body;
    const res = await alice.request("POST", `/api/projects/${project.id}/tasks`, { title: "   " });
    assert.equal(res.status, 422);
    assert.equal(res.body.error.code, "VALIDATION_ERROR");
  });

  it("does not let another user read or change someone else's task (BOLA)", async () => {
    const alice = await signUp("alice@example.com");
    const bob = await signUp("bob@example.com");
    const project = (await alice.request("POST", "/api/projects", { name: "Private" })).body;
    const task = (await alice.request("POST", `/api/projects/${project.id}/tasks`, { title: "Secret" })).body;

    assert.equal((await bob.request("GET", `/api/projects/${project.id}/tasks`)).status, 404);
    assert.equal((await bob.request("PATCH", `/api/tasks/${task.id}`, { status: "done" })).status, 404);
    assert.equal((await bob.request("DELETE", `/api/tasks/${task.id}`)).status, 404);
  });

  it("does not let a client choose its own role", async () => {
    const c = createClient(ctx.baseUrl);
    const res = await c.request("POST", "/api/auth/register",
      { email: "eve@example.com", name: "Eve", password: "correct horse battery", role: "admin" });
    assert.equal(res.body.role, "member");
  });
});
```

Run with `NODE_ENV=test node --env-file=.env.test --test`. By default `node --test` runs **test files in parallel**, so tests that share one database can interfere. Either run files serially (`--test-concurrency=1`), give each file its own schema or database, or keep one file per feature and reset between tests.

### Library alternatives

- **`supertest`** (`npm i -D supertest`) is popular with Express: `await request(app).post("/api/tasks").send({...}).expect(201)`, with `request.agent(app)` to keep cookies. It needs no port.
- **Hono** apps can be called directly: `const res = await app.request("/api/tasks")`.
- For mocking outbound HTTP, use **MSW** or undici's `MockAgent`.

### Test data builders

```ts
export const makeTask = (overrides: Partial<Task> = {}): Task => ({
  id: 1, projectId: 1, title: "A task", status: "todo", version: 1, ...overrides,
});
```

### CI: run it with a real Postgres

```yaml
# .github/workflows/ci.yml (excerpt)
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17-alpine
        env: { POSTGRES_USER: app, POSTGRES_PASSWORD: secret, POSTGRES_DB: taskflow_test }
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U app" --health-interval 5s --health-timeout 3s --health-retries 10
    env:
      NODE_ENV: test
      DATABASE_URL: postgres://app:secret@localhost:5432/taskflow_test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 26, cache: npm }
      - run: npm ci
      - run: npm run typecheck
      - run: node scripts/migrate.ts
      - run: node --test
```

See [CI/CD Pipelines](devops/ci-cd-pipelines) for the bigger picture.

#### Interview traps

- "Unit vs. integration tests?" Unit: one piece in isolation, fast. Integration: several real parts (route + service + DB) together. A healthy suite has many unit tests for logic and a solid set of integration tests for the API contract.
- "Should you mock the database?" For testing SQL and constraints, no. Use a real instance. Mock only things you do not own or that are slow or non-deterministic (third-party APIs, time).
- "Why are my tests flaky?" Shared state between tests, real time or randomness, un-awaited async work, order dependence, parallel tests hitting one database.
- "Why start the server on port 0?" The OS assigns a free port, so tests never collide.

#### Tricky question and answer

###### Q1 [SDE-2]: Tests pass locally but fail intermittently in CI with "duplicate key" errors. Why, and how do you fix it?

**Answer:** Test files run in parallel and share one database, so one file's inserts collide with another's (or a `TRUNCATE` in one file deletes another's data mid-test). Fixes: run DB-backed files serially (`--test-concurrency=1`), give each worker its own schema or database (for example by suffixing the database name with a worker id), or isolate each test in a transaction that is rolled back. Also generate unique values (random emails) instead of hard-coded ones, and never depend on test order.

## 7.4: Real-Time Options: Which One Do You Need?

HTTP is **request/response**: the client asks, the server answers. For **live** features (a teammate completes a task and your screen updates, notifications, chat, live prices), the server must **push**. The options, from simplest to most capable:

| Technique                 | How it works                                                  | Direction     | Complexity | Use for                                              |
| ------------------------- | ------------------------------------------------------------- | ------------- | ---------- | ---------------------------------------------------- |
| **Polling**               | Client asks every N seconds                                   | Client → server | Lowest   | Low-frequency updates, "good enough"                 |
| **Long polling**          | Server holds the request open until there is news             | Server → client | Low-medium | Legacy fallback                                    |
| **Server-Sent Events**    | One long HTTP response streaming `text/event-stream`          | **Server → client only** | Low | Live feeds, notifications, progress, dashboards  |
| **WebSocket**             | Upgraded connection, both sides send freely                   | **Two-way**   | Medium-high | Chat, collaborative editing, games, bidirectional streams |
| **Socket.IO**             | A library on WebSocket with rooms, auto-reconnect and fallbacks | Two-way     | Medium     | Apps that want those conveniences ([Socket.IO intro](socketio/introduction-to-socketio)) |

**Choose the simplest that works.** If the browser only needs to **receive** updates (most dashboards and notification feeds), **SSE** is often the best answer: it is plain HTTP, works through proxies and cookies, and reconnects automatically. Reach for WebSockets when the **client also sends frequent messages** on the same channel.

## 7.5: Server-Sent Events (SSE)

### How it works

The server sends a response that **never ends**, with `Content-Type: text/event-stream`, and writes events in a simple text format:

```
event: task.updated
data: {"id":7,"status":"done"}

: this line is a comment, used as a heartbeat

event: task.created
id: 42
data: {"id":8,"title":"Buy milk"}

```

Each event ends with a **blank line**. The browser's built-in **`EventSource`** parses them and **reconnects automatically**, sending the last event `id` in a `Last-Event-ID` header so you can resume.

### Implementation in Express

```ts
// src/realtime/hub.ts: an in-process registry of connected clients, grouped by project
import type { Response } from "express";

const subscribers = new Map<number, Set<Response>>();

export function subscribe(projectId: number, res: Response) {
  if (!subscribers.has(projectId)) subscribers.set(projectId, new Set());
  subscribers.get(projectId)!.add(res);
  return () => {
    const set = subscribers.get(projectId);
    set?.delete(res);
    if (set?.size === 0) subscribers.delete(projectId);
  };
}

export function localPublish(projectId: number, event: string, data: unknown) {
  const payload = `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`;
  for (const res of subscribers.get(projectId) ?? []) res.write(payload);
}
```

```ts
// src/routes/events.ts
eventsRouter.get("/projects/:projectId/events", requireAuth, async (req, res) => {
  const projectId = toId(req.params.projectId);
  await assertProjectOwner(projectId, req.user!.id);            // authorize BEFORE opening the stream

  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache, no-transform",
    "X-Accel-Buffering": "no",                                    // tell nginx not to buffer this response
  });
  res.write("retry: 3000\n\n");                                   // client reconnect delay

  const heartbeat = setInterval(() => res.write(": ping\n\n"), 25_000);   // keeps proxies from closing an idle stream
  const unsubscribe = subscribe(projectId, res);

  req.on("close", () => {                                         // the client left: clean up or you LEAK
    clearInterval(heartbeat);
    unsubscribe();
  });
});
```

Publish from your service layer after a change is **committed**:

```ts
const task = await taskRepo.update(pool, id, patch, userId);
publish(task.projectId, "task.updated", task);       // every connected tab of this project updates
```

### The React side

```tsx
useEffect(() => {
  const es = new EventSource(`/api/projects/${projectId}/events`, { withCredentials: true });

  es.addEventListener("task.updated", (e) => {
    const task = JSON.parse((e as MessageEvent).data);
    queryClient.setQueriesData({ queryKey: ["tasks"] }, (old: Task[] | undefined) =>
      old?.map((t) => (t.id === task.id ? task : t)));
  });

  return () => es.close();                         // cleanup (Part 1, Module 3)
}, [projectId]);
```

### Limits and gotchas

| Topic                          | Detail                                                                                                          |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| One-way, text only             | Server to client. For client messages use a normal `POST`. No binary frames                                      |
| **Compression / proxy buffering** | `compression` middleware and some proxies buffer the response, so events arrive late or in a burst. Disable for this route and set `X-Accel-Buffering: no` |
| **Connection limit on HTTP/1.1** | Browsers allow about 6 connections per host, shared across tabs. HTTP/2 multiplexes and removes the problem    |
| **No custom headers**          | `EventSource` cannot set `Authorization`. Use cookies (as here) or a short-lived token in the URL                |
| **Resource use**               | Each client holds an open connection and memory. Heartbeats and cleanup on `close` are mandatory                  |
| **Multiple instances**         | A client is connected to **one** instance. See the Redis bridge below                                           |

### Scaling to several instances: Redis Pub/Sub

If the task is updated through instance A but the viewer is connected to instance B, B never learns about it. Use **Redis Pub/Sub** as the backbone: every instance publishes changes to a channel, and every instance subscribes and relays to its **own** connected clients.

```ts
// src/realtime/bus.ts
import { redis } from "../cache.ts";
import { localPublish } from "./hub.ts";

const subscriber = redis.duplicate();                         // a subscribing connection can do nothing else
await subscriber.connect();
await subscriber.subscribe("task-events", (message) => {
  const { projectId, event, data } = JSON.parse(message);
  localPublish(projectId, event, data);                       // deliver to clients on THIS instance
});

export const publish = (projectId: number, event: string, data: unknown) =>
  redis.publish("task-events", JSON.stringify({ projectId, event, data }));
```

Redis Pub/Sub is **fire-and-forget**: an instance that is down misses messages. That is fine for "refresh your UI" signals (clients can refetch on reconnect), but use a **stream or a queue** when delivery must be guaranteed.

#### Interview traps

- "SSE vs. WebSocket?" SSE: one-way, plain HTTP, auto-reconnect, simple, works with cookies and most proxies. WebSocket: two-way, binary, lower overhead for chatty bidirectional traffic, but needs more care (heartbeats, upgrade handling, scaling).
- "How does `EventSource` resume after a dropped connection?" It reconnects automatically and sends the `Last-Event-ID` header, so the server can replay missed events if it keeps them.
- "What breaks SSE in production?" Proxy buffering or compression, idle timeouts without heartbeats, and forgetting to clean up on `close`.

## 7.6: WebSockets

### How it works

A WebSocket starts as an HTTP request with `Upgrade: websocket`. The server answers `101 Switching Protocols`, and the **same TCP connection** now carries **two-way frames** (text or binary) with very little per-message overhead. It stays open until one side closes it.

```
 Client                                       Server
   │── GET /ws  Upgrade: websocket ───────────►│
   │◄─ 101 Switching Protocols ────────────────│
   │◄══════════ frames both ways ═════════════►│   (until close)
```

### With the `ws` library (Node)

```bash
npm install ws cookie
npm install -D @types/ws
```

```ts
// src/realtime/ws.ts
import type { Server } from "node:http";
import { WebSocketServer, type WebSocket } from "ws";
import { parse as parseCookie } from "cookie";
import { z } from "zod";
import { COOKIE_NAME, getSessionUser } from "../auth/sessions.ts";
import { env } from "../env.ts";

type Live = WebSocket & { isAlive: boolean; userId: number };

export function attachWebSocket(server: Server) {
  const wss = new WebSocketServer({ noServer: true, maxPayload: 16 * 1024 });   // cap message size

  server.on("upgrade", async (req, socket, head) => {
    try {
      if (req.url !== "/ws") throw new Error("not found");
      if (req.headers.origin !== env.CORS_ORIGIN) throw new Error("bad origin");      // WebSockets are NOT covered by CORS!
      const token = parseCookie(req.headers.cookie ?? "")[COOKIE_NAME];
      const user = token ? await getSessionUser(token) : undefined;
      if (!user) throw new Error("unauthenticated");

      wss.handleUpgrade(req, socket, head, (ws) => {
        (ws as Live).userId = user.id;
        wss.emit("connection", ws, req);
      });
    } catch {
      socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");                              // reject BEFORE upgrading
      socket.destroy();
    }
  });

  const Message = z.discriminatedUnion("type", [
    z.object({ type: z.literal("ping") }),
    z.object({ type: z.literal("typing"), taskId: z.number().int() }),
  ]);

  wss.on("connection", (raw) => {
    const ws = raw as Live;
    ws.isAlive = true;
    ws.on("pong", () => { ws.isAlive = true; });

    ws.on("message", (data) => {
      const parsed = Message.safeParse(safeJson(data.toString()));      // validate EVERY incoming message
      if (!parsed.success) return ws.close(1008, "invalid message");
      // handle parsed.data ...
    });

    ws.on("error", (err) => console.error("ws error", err));
  });

  // heartbeat: find and drop dead connections (phones go offline without saying goodbye)
  const interval = setInterval(() => {
    for (const client of wss.clients) {
      const ws = client as Live;
      if (!ws.isAlive) { ws.terminate(); continue; }
      ws.isAlive = false;
      ws.ping();
    }
  }, 30_000);
  wss.on("close", () => clearInterval(interval));

  return wss;
}

const safeJson = (s: string) => { try { return JSON.parse(s); } catch { return undefined; } };
```

In `server.ts`: `const server = app.listen(env.PORT); attachWebSocket(server);`

### Browser client with reconnection

```ts
function connect(attempt = 0) {
  const ws = new WebSocket(`wss://${location.host}/ws`);
  ws.onopen = () => { attempt = 0; };
  ws.onmessage = (e) => handle(JSON.parse(e.data));
  ws.onclose = () => setTimeout(() => connect(attempt + 1), Math.min(30_000, 500 * 2 ** attempt) + Math.random() * 500);   // backoff + jitter
}
```

WebSockets have **no automatic reconnect**. You write it (or use Socket.IO, which does). The **jitter** matters: if a server restarts and 10,000 clients retry at the same instant, the thundering herd takes it down again.

### Bun: native WebSockets with pub/sub

```ts
const server = Bun.serve({
  port: 3000,
  async fetch(req, server) {
    const user = await authenticate(req);                         // cookie check
    if (!user) return new Response("Unauthorized", { status: 401 });
    if (server.upgrade(req, { data: { userId: user.id } })) return;   // upgraded: return nothing
    return new Response("Upgrade failed", { status: 400 });
  },
  websocket: {
    maxPayloadLength: 16 * 1024,
    open(ws) { ws.subscribe("project:1"); },                      // topic-based pub/sub is built in
    message(ws, message) { server.publish("project:1", message); },   // fan out to every subscriber
    close(ws) { /* cleanup */ },
  },
});
```

Hono provides `upgradeWebSocket` helpers (via adapters such as `@hono/node-ws`), and **Socket.IO** adds rooms, acknowledgements and reconnection on either runtime. See the [Socket.IO introduction](socketio/introduction-to-socketio).

### Scaling WebSockets

| Challenge                         | Approach                                                                                           |
| --------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Stateful connections**          | Each client is tied to one instance. Fan out across instances with Redis Pub/Sub (as above) or the Socket.IO Redis adapter |
| **Load balancer behavior**        | Must support WebSocket upgrades and long idle timeouts. See [Load Balancing](system-design/load-balancing) |
| **File descriptors and memory**   | Each connection uses a socket and memory. Raise `ulimit -n`, monitor connection counts             |
| **Slow clients**                  | Check `ws.bufferedAmount` before sending. Drop or throttle slow consumers (backpressure)           |
| **Deploys**                       | Restarting an instance drops its connections. Clients reconnect with jittered backoff. Drain gradually |
| **Security**                      | Authenticate during the **upgrade**, check `Origin`, validate and size-limit every message, rate-limit message rates |
| **Presence ("who is online")**    | Store in Redis with TTLs refreshed by heartbeats                                                   |

For a full-scale design see [WhatsApp Design](system-design/whatsapp-design) and [Messaging and Communication](system-design/messaging-and-communication).

#### Interview traps

- "Is CORS applied to WebSockets?" **No.** The browser sends an `Origin` header, but the server must check it itself. Without a check, any website can open a WebSocket to your server using the visitor's cookies (**cross-site WebSocket hijacking**).
- "How do you authenticate a WebSocket?" During the HTTP upgrade (cookie or short-lived token), before accepting the connection. Do not accept first and "authenticate later."
- "How do you scale WebSockets horizontally?" Every instance serves its own connections, and a shared pub/sub backbone (Redis) distributes messages between instances. Use sticky sessions only if the protocol requires them (Socket.IO long-polling fallback).
- "Why heartbeats?" Connections can die silently (mobile networks, NAT timeouts). Pings detect dead peers and keep intermediaries from closing idle connections.

## 7.7: Background Jobs and Queues

### Why not do everything in the request?

Some work is **slow**, **unreliable** or **not needed to answer the user**: sending email, resizing images, generating a PDF report, calling a flaky third-party API, syncing data. Doing it inside the request handler makes the user wait, ties up your server, and loses the work if the process crashes or the call fails.

```
 request ──► validate ──► save to DB ──► ENQUEUE job ──► respond 202 / 201 immediately
                                              │
                                              ▼
                                   [ queue in Redis / Postgres ]
                                              │
                                              ▼
                              worker process picks it up, runs it,
                              retries on failure, records the result
```

### Options

| Approach                                  | Durable? | Notes                                                                            |
| ----------------------------------------- | -------- | -------------------------------------------------------------------------------- |
| `setImmediate(() => doWork())` in process | **No**   | Lost if the process dies, no retries. Fine only for trivial, loss-tolerant work  |
| **BullMQ** (Redis)                        | Yes      | The most popular Node queue: retries, backoff, delays, priorities, rate limits, schedulers, a dashboard |
| **pg-boss** (PostgreSQL)                  | Yes      | A queue built on Postgres. **No extra infrastructure** if you already run Postgres |
| Cloud queues (SQS, Pub/Sub, Cloud Tasks)  | Yes      | Managed, scales massively. Pairs with serverless workers                         |
| Message brokers (RabbitMQ, Kafka)         | Yes      | Event-driven architectures and streaming. See [Event-Driven Architecture](system-design/event-driven) |

### BullMQ example

```bash
npm install bullmq
```

```ts
// src/jobs/queues.ts
import { Queue } from "bullmq";
import { env } from "../env.ts";

const url = new URL(env.REDIS_URL);
export const connection = { host: url.hostname, port: Number(url.port || 6379) };

export const emailQueue = new Queue("email", { connection });

// Producer: called from a service or route
export function enqueueWelcomeEmail(userId: number) {
  return emailQueue.add(
    "welcome",
    { userId },                                        // small payload: IDs, not whole objects
    {
      jobId: `welcome:${userId}`,                      // a deterministic id de-duplicates repeats
      attempts: 5,                                     // total tries
      backoff: { type: "exponential", delay: 2_000 },  // 2s, 4s, 8s, ...
      removeOnComplete: 1_000,                         // keep the last 1000 for inspection
      removeOnFail: 5_000,                             // failed jobs stay inspectable (dead-letter)
    },
  );
}
```

```ts
// src/worker.ts: a SEPARATE entry point, run as its own process: `node src/worker.ts`
import { Worker } from "bullmq";
import { connection } from "./jobs/queues.ts";
import { pool } from "./db/pool.ts";
import { sendEmail } from "./lib/mailer.ts";

const worker = new Worker(
  "email",
  async (job) => {
    const { rows } = await pool.query("SELECT email, name, welcome_sent_at FROM users WHERE id = $1", [job.data.userId]);
    const user = rows[0];
    if (!user || user.welcome_sent_at) return;                         // IDEMPOTENT: already sent or user gone
    await sendEmail(user.email, "Welcome to TaskFlow", `Hi ${user.name}!`);
    await pool.query("UPDATE users SET welcome_sent_at = now() WHERE id = $1", [job.data.userId]);
  },
  { connection, concurrency: 5 },
);

worker.on("failed", (job, err) => console.error(`job ${job?.id} failed`, err));

process.on("SIGTERM", async () => {                                    // graceful: finish the current job, then exit
  await worker.close();
  await pool.end();
  process.exit(0);
});
```

### The rules of reliable jobs

| Rule                                         | Why                                                                                              |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Make every job idempotent**                | Delivery is **at-least-once**. A worker can crash after doing the work but before reporting it, so the job runs again. Running twice must be harmless (check state first, use unique keys, idempotency keys with providers) |
| **Retry with exponential backoff and a cap** | Transient failures (a provider blip) fix themselves. Hammering a failing service makes it worse   |
| **Keep failed jobs** (dead-letter)           | You can inspect, fix the bug and **replay** them                                                  |
| **Small payloads (IDs)**                     | Data in the queue gets stale, and queues are not databases                                        |
| **Separate worker processes**                | Scale and deploy workers independently from the API. A slow job must never block requests         |
| **Timeouts per job**                         | A stuck job should not hold a worker forever                                                      |
| **Do not enqueue before the commit**         | If you enqueue inside a DB transaction that later rolls back (or the worker runs before commit), the job sees missing data. Enqueue **after** commit, or use the **outbox pattern** |
| **Monitor queue depth and age**              | A growing backlog is your early-warning alarm. Use a dashboard (Bull Board)                       |

### The outbox pattern in one picture

Enqueuing after the commit has a gap: the process can die between `COMMIT` and `queue.add`, losing the job. The **transactional outbox** closes it: in the **same transaction** as the business change, insert a row into an `outbox` table. A small relay process reads unsent rows, enqueues them, and marks them sent. The job can be delayed but never lost.

### Scheduled (cron) jobs

```ts
// BullMQ job scheduler: one repeatable schedule, shared by every instance (no duplicates)
await emailQueue.upsertJobScheduler("purge-sessions", { pattern: "0 3 * * *" }, { name: "purge-sessions" });
```

(Scheduler APIs have changed across BullMQ versions, so check its docs.) A common mistake is running `node-cron` inside the API: with 5 instances, the job runs **5 times**. Use a queue-based scheduler, a single dedicated scheduler process, a Postgres **advisory lock** so only one instance runs it, or the platform's own cron (a Kubernetes `CronJob`, a cloud scheduler).

### Webhooks: the reverse direction

A **webhook** is another service calling **your** endpoint (Stripe says "payment succeeded"). The reliable pattern:

1. Read the **raw body** (do not parse JSON first, signatures cover the exact bytes) and **verify the signature** with HMAC and `timingSafeEqual` (Module 3).
2. Check the **event ID** against a table of processed events, because providers **retry** and may send duplicates.
3. **Respond `200` quickly**, and put the real work in a **job**. Providers time out and retry slow endpoints.

```ts
app.post("/webhooks/payments", express.raw({ type: "application/json" }), async (req, res) => {
  if (!verifySignature(req.body, String(req.get("x-signature")), env.WEBHOOK_SECRET)) {
    throw new AppError("Invalid signature", 401, "BAD_SIGNATURE");
  }
  const event = JSON.parse(req.body.toString());
  const { rowCount } = await pool.query(
    "INSERT INTO processed_events (id) VALUES ($1) ON CONFLICT DO NOTHING", [event.id]);
  if (rowCount) await paymentsQueue.add("handle", { eventId: event.id });     // only the first delivery enqueues
  res.sendStatus(200);
});
```

(Register this route **before** `express.json()` so the body stays raw.)

### File uploads, the right way

Do not stream large uploads through your API. Have the client ask the API for a short-lived **presigned URL** and upload **directly to object storage** (S3 or compatible). The API stays light and only records the result. See [File Storage Design](system-design/file-storage-design).

#### Interview traps

- "Why do jobs need to be idempotent?" Queues deliver **at least once**. Crashes, timeouts and retries cause the same job to run again, and running it twice must not double-charge, double-email or double-insert.
- "Why not run cron inside every API instance?" Each instance fires the job, so it runs N times. Use a single scheduler, a lock, or the platform's cron.
- "Redis queue vs. Postgres queue?" Redis (BullMQ) is faster and feature-rich. Postgres (pg-boss) removes a moving part and lets you enqueue **inside** the same transaction as your data, which avoids the outbox problem. Choose by scale and operational preference.
- "What should a webhook handler do first?" Verify the signature, dedupe by event ID, enqueue, and return `2xx` fast.

#### Tricky questions and answers

###### Q2 [SDE-3]: A user signs up. You insert the user and then call `queue.add("welcome")`. Occasionally the welcome email is never sent. Why, and how do you fix it?

**Answer:** The two actions are not atomic. The process can crash (or Redis can be unreachable) after the database commit but before the job is added, so the user exists without a job. Fix with the **transactional outbox**: write an `outbox` row in the same transaction as the user, and let a relay enqueue it (retrying until it succeeds). Alternatively use a Postgres-based queue such as pg-boss so the job insert shares the transaction. In either design, the worker must be idempotent because the relay itself delivers at least once.

###### Q3 [SDE-3]: You run 4 API instances behind a load balancer, and users on different instances do not see each other's live updates. Explain and fix.

**Answer:** SSE and WebSocket connections live in the memory of **one** process, and the in-process subscriber map only knows about local clients. A change handled by instance A is never broadcast to clients connected to B. Add a shared **pub/sub backbone**: every instance publishes events to Redis, and every instance subscribes and relays to its own connected clients. For guaranteed delivery or replay, use a stream or queue instead of fire-and-forget pub/sub, and make clients refetch current state on reconnect so a missed message heals itself.

###### Q4 [SDE-2]: How would you test a background job?

**Answer:** Test the **handler function** directly as a plain async function with a real test database and a fake mailer: call it with a job payload, assert the effects (email recorded, `welcome_sent_at` set), then call it **again** and assert nothing happens twice (idempotency). Test the enqueue side by asserting that the API adds the right job (with a fake queue or a real Redis). Keep the worker wiring thin, so almost all logic is in testable functions.

---

## Hands-On Lab: Make TaskFlow Live and Reliable

```
1. TESTS. Create test/helpers.ts and a taskflow_test database. Write the five
   tests above plus: login with a wrong password (401), rate limit on login (429),
   logout invalidates the cookie, validation 422, malformed JSON 400.
   Run with node --test, then bun test (adjust imports). Make them pass in both.
2. Add a CI workflow with a Postgres service container that runs typecheck,
   migrations and tests on every push.
3. Mutation check: temporarily remove the owner_id condition from one query.
   Does a test fail? If not, add the missing negative test.
4. SSE. Implement GET /api/projects/:id/events with heartbeats and cleanup.
   Open two browser tabs on the same project (the React app with EventSource).
   Update a task in one tab and watch the other update. Check in DevTools →
   Network that the request stays open.
5. Kill the tab and confirm the server removed the subscriber (log subscriber count).
6. Run TWO instances of the API on ports 3000 and 3001. Connect tab A to 3000 and
   update through 3001: it does not appear. Add the Redis Pub/Sub bridge. Retest.
7. Add the ws WebSocket endpoint with Origin and cookie checks. Write a Node script
   using the global WebSocket client that connects without a cookie (must be
   rejected) and with an invalid message (must be closed with 1008).
8. Add BullMQ and a worker process. On registration enqueue a welcome email
   (log to the console instead of sending). Kill the worker mid-job and restart it:
   was the job retried? Make the handler idempotent and prove a double run is safe.
9. Add a repeatable job that purges expired sessions every night, with only one
   instance running it when you start three API processes.
10. Add POST /webhooks/test that verifies an HMAC signature, dedupes by event id,
    enqueues a job and returns 200 quickly. Replay the same event twice.
```

Questions to answer for yourself:

- Which of your tests would catch the removal of `owner_id` scoping? Which would not?
- What happens to connected browsers when you restart an API instance? What did you do to make recovery smooth?
- Which jobs could cause damage if they ran twice, and how does each protect itself?

## Module 7 Cheat Sheet

| Concept                     | One-line interview answer                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------- |
| Test runners                | `node:test` (built in), `bun test`, Vitest. Same ideas, pick one.                                  |
| Good test                   | Behavior-focused, independent, deterministic, covers failure paths, readable name.                  |
| Dependency injection        | Pass collaborators as function arguments. No framework needed.                                      |
| Database in tests           | Use a real Postgres (Docker or CI service). Mocking SQL hides the bugs that matter.                 |
| `createApp()` + port `0`    | Test the app without fixed ports. The OS gives a free one.                                          |
| Flaky tests                 | Shared state, real time or randomness, un-awaited promises, parallel runs on one DB.                |
| Negative auth tests         | Always test that user B cannot touch user A's data.                                                 |
| Polling / SSE / WebSocket   | Simple pull / one-way push over HTTP / two-way persistent. Choose the simplest that works.          |
| SSE essentials              | `text/event-stream`, heartbeat comments, cleanup on `close`, disable buffering and compression.      |
| `EventSource`               | Auto-reconnects and sends `Last-Event-ID`. GET only, cookies not custom headers.                    |
| WebSocket handshake         | HTTP `Upgrade` → `101`. Authenticate **during the upgrade**. Check `Origin` (CORS does not apply).   |
| WebSocket hygiene           | Heartbeats, max payload size, validate messages, backpressure, client backoff with jitter.          |
| Scaling real-time           | Connections are per-instance. Redis Pub/Sub (or an adapter) fans out across instances.              |
| Redis Pub/Sub               | Fire-and-forget. Missed messages are gone. Refetch on reconnect or use a stream/queue.              |
| Job queue                   | Durable work handed to separate workers: retries, backoff, delays, dead-letter.                     |
| At-least-once delivery      | Jobs can run twice. Handlers must be **idempotent**.                                                |
| Outbox pattern              | Insert the job row in the same DB transaction, relay it later. Never lose or phantom-enqueue a job. |
| Cron in a cluster           | Do not run `node-cron` in every instance. Use a queue scheduler, a lock, or platform cron.          |
| Webhook handling            | Verify HMAC on the raw body, dedupe by event id, enqueue, return `2xx` fast.                        |
| Uploads                     | Presigned URLs straight to object storage. Do not proxy big files through the API.                  |
