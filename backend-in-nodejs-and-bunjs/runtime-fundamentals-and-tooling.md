---
title: "Module 1: Runtime Fundamentals & Tooling (Node.js and Bun)"
description: "Learn what Node.js and Bun are, how to install and pick the right version in 2026, run JavaScript and TypeScript outside the browser, use CommonJS and ES modules, manage packages with package.json, and use the built-in globals that replace many npm packages."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/nodejs-module-1.png"
tags: [Node.js, Bun, JavaScript, npm, ES-Modules, Interview-Prep]
keywords: ["What is Node.js", "Node.js vs Bun", "Node.js LTS 2026", "CommonJS vs ES modules", "package.json explained", "Node.js native TypeScript", "Node.js tutorial for beginners"]
---

# Module 1: Runtime Fundamentals & Tooling (Node.js and Bun)

![Node.js Module 1](/images/nodejs-module-1.png)

## Words you will meet in this module

| Word                  | Simple meaning                                                                                           |
| --------------------- | -------------------------------------------------------------------------------------------------------- |
| **Runtime**           | A program that runs your code and gives it tools (files, network, timers). Node.js and Bun are runtimes. |
| **JavaScript engine** | The part that actually executes JS. Node uses **V8** (Chrome's). Bun uses **JavaScriptCore** (Safari's). |
| **libuv**             | The C library inside Node.js that handles the event loop and asynchronous I/O.                           |
| **Module**            | A file that exports things so other files can import them.                                               |
| **CommonJS (CJS)**    | The older module system: `require()` and `module.exports`.                                               |
| **ES Modules (ESM)**  | The standard module system: `import` and `export`. Used by browsers and modern Node.                     |
| **npm**               | The package registry and the default package manager that ships with Node.js.                            |
| **package.json**      | The file that describes your project: name, scripts, dependencies, module type.                          |
| **Lockfile**          | A file recording the exact versions installed, so every machine gets the same tree.                      |
| **Semver**            | Semantic versioning, `MAJOR.MINOR.PATCH`. Major changes can break you.                                   |
| **LTS**               | Long-Term Support: a Node.js release line that gets fixes for years. Use it in production.               |
| **Type stripping**    | Node (and Bun) running `.ts` files by simply erasing the type annotations. No compiler step.             |

## 1.1 What Are Node.js and Bun?

### The idea in plain words

JavaScript was born inside the browser. It could change a web page but could not read a file or open a network port. In 2009 **Node.js** took Chrome's JavaScript engine (V8), put it in a normal program, and added the things a server needs: files, network sockets, processes, timers. Suddenly JavaScript could run **anywhere**, and one language could power both the front end (your React app) and the back end.

```mermaid
flowchart LR
    subgraph Browser["Browser"]
        direction TB
        B1["JS engine (V8 ...)"]
        B2["+ DOM, window,<br/>fetch, localStorage"]
        B3["No file access, sandboxed"]
        B1 --- B2 --- B3
    end

    subgraph Node["Node.js / Bun"]
        direction TB
        N1["JS engine (V8 / JavaScriptCore)"]
        N2["+ file system, network,<br/>processes, streams, crypto"]
        N3["No DOM, no window.<br/>Full OS access"]
        N1 --- N2 --- N3
    end
```

### What is inside Node.js

| Part               | Job                                                                                           |
| ------------------ | --------------------------------------------------------------------------------------------- |
| **V8**             | Compiles and runs your JavaScript very fast (JIT compilation).                                |
| **libuv**          | The **event loop**, a thread pool, and non-blocking file, network and timer I/O (Module 2).   |
| **Core modules**   | Built-in libraries: `node:fs`, `node:http`, `node:crypto`, `node:stream` and more (Module 3). |
| **Bindings (C++)** | Glue between your JavaScript and the operating system.                                        |
| **npm**            | The package manager and the biggest package registry in the world.                            |

### The big idea: one thread of JavaScript, many things in flight

Your JavaScript runs on **one thread**. That sounds slow, but a server mostly **waits**: for a database, a file, a network reply. Node starts the slow operation, **does not wait**, and handles another request. When the operation finishes, a callback runs. One waiter can serve many tables if the waiter never stands still at one table. Module 2 explains this fully.

> **Good fit:** APIs, real-time apps (chat, live dashboards), proxies, tooling (Vite, ESLint, your React build all run on Node), CLIs, server-side rendering.
> **Poor fit (on the main thread):** heavy CPU work such as video encoding or big number crunching. You can still do it with worker threads or other services (Module 2).

### What is Bun?

**Bun** is a newer all-in-one JavaScript runtime. It uses Safari's **JavaScriptCore** engine instead of V8, and it bundles a **package manager**, a **test runner**, a **bundler** and a TypeScript transpiler into one fast binary. Its goal is to be a drop-in replacement for Node.js for most projects, with much faster startup and installs. Bun's current line is **1.4.x**. Its release notes describe the 1.4 release as a rewrite of the runtime in Rust (older articles say Zig), and Bun tracks Node.js 26 compatibility.

| Question         | Node.js                                          | Bun                                                                                    |
| ---------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------- |
| Engine           | V8                                               | JavaScriptCore                                                                         |
| Maturity         | Since 2009. The industry standard.               | 1.0 in September 2023. Newer, fast-moving.                                             |
| What you install | Node + npm (+ your own test, TS tools)           | One binary: runtime, `bun install`, `bun test`, `bun build`                            |
| TypeScript       | Runs `.ts` by type stripping (erasable syntax)   | Runs `.ts` and `.tsx` directly                                                         |
| `.env` files     | `--env-file=.env`                                | Loaded automatically                                                                   |
| Built-in extras  | `node:test`, `node:sqlite`, `fetch`              | `Bun.serve`, `Bun.sql`, `Bun.file`, `bun:sqlite`, `Bun.password`, S3 and Redis clients |
| Compatibility    | The reference                                    | High, but not 100%. Check native addons and edge cases.                                |
| Typical use      | Production APIs, enterprise, anything long-lived | Fast tooling, scripts, tests, greenfield services                                      |

> **Context worth knowing:** in December 2025 Bun's creator announced that Bun is joining Anthropic, the company that makes Claude (the assistant writing this tutorial). Treat that as background on who maintains the project, not as a reason to choose or avoid it. Module 8, Topic 8.8 gives a neutral decision guide.

### How this series handles both

Every example **runs on Node.js**, the baseline that all hosting platforms support. Wherever Bun differs, a short **Bun note** shows the difference or the Bun-native alternative. Most code is identical, because both runtimes implement the same web-standard APIs (`fetch`, `URL`, `Request`, `Response`, streams).

#### Interview traps

- "Is Node.js a language, a framework or a library?" None of them. It is a **runtime** for JavaScript.
- "Is Node.js single-threaded?" Your JS runs on one thread, but Node itself uses a **thread pool** and the OS for I/O, and you can add worker threads. "Single-threaded event loop, multi-threaded under the hood" is the precise answer.
- "Why is Node good at I/O but bad at CPU work?" I/O waits do not block the thread, but CPU work keeps the one JS thread busy, so every other request waits.

#### Tricky question and answer

###### Q1 [SDE-1]: Why can JavaScript read files in Node.js but not in the browser?

**Answer:** The language itself has no file API. Capabilities come from the **host environment**. A browser deliberately gives JavaScript a sandbox (DOM, `fetch`, `localStorage`) for safety. Node gives JavaScript operating-system access through its core modules (`node:fs`, `node:net`). Same language, different host objects.

## 1.2 Installing and Choosing a Version

### The release policy in 2026

Node.js has **LTS** lines you should use for production. As of November 2026:

| Line                   | Status                                                                                                                                       | Supported until |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| Node.js 20             | **End of life** (April 2026). Do not use.                                                                                                    | Ended           |
| Node.js 22 ("Jod")     | Maintenance LTS                                                                                                                              | April 30, 2027  |
| Node.js 24 ("Krypton") | LTS (moving into maintenance from October 2026)                                                                                              | April 30, 2028  |
| Node.js 26 ("Lithium") | **Active LTS**, since October 28, 2026                                                                                                       | April 30, 2029  |
| Node.js 27 and later   | New model: one major release each **April**, promoted to LTS each October. **Every** release becomes LTS (the old odd/even rule is retired). | 2030+           |

**Practical advice:** start new projects on the **current Active LTS (Node.js 26)**. Node 24 is a fine, well-tested choice too. Pin the major version in your project so everyone, including CI and Docker, uses the same one.

Release lines change, so confirm on the official Node.js "releases" page when you read this.

### Installing Node.js with a version manager

Do not install one global Node from a website installer and forget it. Use a **version manager** so each project can use its own version:

```bash
# fnm (fast, cross-platform)             # nvm (the classic)
curl -fsSL https://fnm.vercel.app/install | bash
fnm install --lts                         nvm install --lts
fnm use --lts                             nvm use --lts

node -v        # e.g. v26.x
npm -v
```

Pin the version for the project so teammates and CI match:

```bash
echo "26" > .nvmrc                 # fnm/nvm read this file
```

```json
// package.json
{ "engines": { "node": ">=24" } }
```

### Installing Bun

```bash
curl -fsSL https://bun.sh/install | bash     # macOS / Linux
# or: npm install -g bun      brew install oven-sh/bun/bun
bun --version                                 # e.g. 1.4.x
bun upgrade                                   # self-update
```

### The REPL and one-liners

```bash
node                       # interactive REPL. Type JS, press Enter.
node -e "console.log(2 + 2)"
node -p "process.version"          # -p prints the result
bun -e "console.log(Bun.version)"
```

### Both runtimes can live side by side

Nothing stops you from having Node and Bun installed together. Many teams run **Node in production** and use **Bun for scripts, tests or local tooling**.

## 1.3 Running Code, Watch Mode, Env Files and TypeScript

### Run a file

```js
// hello.js
console.log('Hello from', process.version);
```

```bash
node hello.js
bun hello.js
```

### Watch mode: no `nodemon` needed

```bash
node --watch server.js       # restarts when the file or its imports change
bun --watch server.ts        # restarts the process
bun --hot server.ts          # hot reload without restarting (keeps state)
```

### Environment variables and `.env` files

Configuration (ports, database URLs, secrets) lives in **environment variables**, not in code.

```bash
# .env     (add this file to .gitignore!)
PORT=3000
DATABASE_URL=postgres://app:secret@localhost:5432/taskflow
```

```js
console.log(process.env.PORT); // always a STRING: "3000", or undefined
const port = Number(process.env.PORT ?? 3000);
```

```bash
node --env-file=.env server.js               # error if the file is missing
node --env-file-if-exists=.env server.js     # skip silently if missing
bun server.ts                                # Bun loads .env automatically
```

Node can also load the file from code with `process.loadEnvFile()`. Neither runtime expands `${VAR}` references inside values the way some packages do, so `URL=http://${HOST}` stays literal. Always **validate** environment variables at startup (Module 3, Topic 3.5 and Module 8).

### TypeScript without a build step

Both runtimes can run `.ts` files directly.

```ts
// tasks.ts
type Task = { id: number; title: string; done: boolean };
const t: Task = { id: 1, title: 'Learn Node', done: false };
console.log(t);
```

```bash
node tasks.ts      # Node 22.18+, 23.6+, 24 and 26: type stripping is on by default
bun tasks.ts
```

**Node's type stripping** simply _erases_ type annotations, which means:

| It works                                        | It does not (without an extra flag)                                     |
| ----------------------------------------------- | ----------------------------------------------------------------------- |
| Type annotations, `interface`, `type`, generics | `enum`, `namespace` with runtime code, constructor parameter properties |
| `import type { X } from "./x.ts"`               | Imports of types **without** the `type` keyword can fail at runtime     |
| ESM with explicit extensions: `from "./x.ts"`   | Extensionless imports like `from "./x"`                                 |

Node **never type-checks**. It does not read `tsconfig.json` to run your code. Keep `tsc --noEmit` in your scripts or CI for real type checking. Bun transpiles a wider range of TypeScript, but it also does not type-check.

### The `process` object: your window to the running program

```js
process.argv; // ["/path/to/node", "/path/to/script.js", ...your arguments]
process.env; // environment variables
process.cwd(); // current working directory
process.platform; // "linux" | "darwin" | "win32"
process.pid; // process id
process.memoryUsage(); // heap and memory numbers
process.exitCode = 1; // set exit status (0 = success, anything else = failure)
process.exit(1); // stop NOW. Avoid. It can cut off pending writes.
process.on('SIGTERM', () => {
  /* graceful shutdown (Module 8) */
});
```

Parse command-line arguments with the built-in `util.parseArgs`:

```js
import { parseArgs } from 'node:util';

const { values, positionals } = parseArgs({
  args: process.argv.slice(2),
  options: { title: { type: 'string', short: 't' }, done: { type: 'boolean' } },
  allowPositionals: true,
});
// node cli.js add --title "Buy milk"  →  positionals ["add"], values { title: "Buy milk" }
```

#### Interview traps

- "Why is `process.env.PORT` not a number?" Environment variables are always strings (or `undefined`). Convert and validate.
- "Does Node type-check my TypeScript?" No. It strips types and runs the rest. Use `tsc --noEmit` for checking.
- "Why is `process.exit()` discouraged?" It terminates immediately, possibly before logs flush or connections close. Set `process.exitCode` and let the program end naturally, or shut down gracefully.

## 1.4 Modules: CommonJS and ES Modules

### Why modules?

A module is a file with its own scope. Names inside it do not leak out. It exports what it wants to share, and other files import it. This keeps large programs organized.

### ES Modules (the standard, and what you should write)

```js
// math.js
export const add = (a, b) => a + b; // named export
export default function multiply(a, b) {
  return a * b;
} // default export

// app.js
import multiply, { add } from './math.js'; // note the .js extension in Node ESM
import { readFile } from 'node:fs/promises'; // built-ins use the node: prefix
```

To tell Node that `.js` files are ES modules, set it **once** in `package.json`:

```json
{ "type": "module" }
```

(Or name the file `.mjs`.) Bun treats files as ESM or CJS automatically.

### CommonJS (the older style you will still meet)

```js
// math.cjs
const add = (a, b) => a + b;
module.exports = { add };

// app.cjs
const { add } = require('./math.cjs');
const fs = require('node:fs');
```

### Side-by-side

| Feature                   | CommonJS                      | ES Modules                                             |
| ------------------------- | ----------------------------- | ------------------------------------------------------ |
| Syntax                    | `require()`, `module.exports` | `import`, `export`                                     |
| Loading                   | **Synchronous**, at runtime   | **Asynchronous**, statically analyzed                  |
| Top-level `await`         | Not allowed                   | **Allowed**                                            |
| `__dirname`, `__filename` | Available                     | Use `import.meta.dirname` and `import.meta.filename`   |
| Tree-shaking by bundlers  | Hard                          | Works well                                             |
| Import JSON               | `require("./data.json")`      | `import data from "./data.json" with { type: "json" }` |
| Dynamic loading           | `require(variable)`           | `await import(path)`                                   |
| Standard                  | Node-specific                 | Part of JavaScript, same as in the browser             |

### Details that trip people up

```js
// Top-level await works in ESM
const config = JSON.parse(await readFile('./config.json', 'utf8'));

// __dirname in ESM
import path from 'node:path';
const dataFile = path.join(import.meta.dirname, 'data', 'tasks.json');

// Dynamic import works in both systems and returns a Promise
const { default: heavy } = await import('./heavy.js');
```

**Interop:** ESM can import CommonJS packages. The other direction used to be impossible (`require` of an ES module failed), but recent Node versions let `require()` load synchronous ES modules (no top-level `await`), so the old "ESM-only package breaks CJS projects" problem has mostly eased. New code should still be written as ESM.

### The `node:` prefix

`import fs from "node:fs"` makes it explicit that this is a **built-in**, not an npm package of the same name. It also protects you from someone publishing a malicious package called `fs`. Always use it.

### How Node finds packages

```
import "express"  → look in ./node_modules/express
                  → then ../node_modules/express
                  → then ../../node_modules/express ... up to the root
```

A package decides what it exposes through the **`exports`** field in its own `package.json`, which also lets it ship separate code for ESM and CJS.

#### Interview traps

- "`require` vs. `import`?" `require` is synchronous CommonJS and can be called anywhere. `import` is static ESM, hoisted to the top, supports top-level `await`, and is the standard.
- "Why does `import "./utils"` fail in Node ESM?" Node ESM needs the **full file extension**: `./utils.js`. (Bundlers like Vite relax this, Node does not.)
- "What is `module.exports` vs. `exports`?" `exports` is just a reference to `module.exports`. Assigning a new value to `exports` breaks the link. Assign to `module.exports` when exporting a single thing.

## 1.5 package.json, npm and the Lockfile

### Create a project

```bash
mkdir taskflow-api && cd taskflow-api
npm init -y                  # creates package.json      |  bun init  (interactive)
npm pkg set type=module      # use ES modules
```

### The fields that matter

```json
{
  "name": "taskflow-api",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=24" },
  "scripts": {
    "dev": "node --watch --env-file-if-exists=.env src/server.ts",
    "start": "node --env-file-if-exists=.env src/server.ts",
    "test": "node --test",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": { "express": "^5.1.0" },
  "devDependencies": { "typescript": "^5.9.0", "@types/node": "^24.0.0" }
}
```

| Field                    | Meaning                                                                     |
| ------------------------ | --------------------------------------------------------------------------- |
| `type`                   | `"module"` for ESM, `"commonjs"` (default) for CJS.                         |
| `scripts`                | Named commands: `npm run dev`. `start` and `test` can be run without `run`. |
| `dependencies`           | Packages your app needs **at runtime**.                                     |
| `devDependencies`        | Tools needed only for **development** (test runner, TypeScript, linters).   |
| `engines`                | Which Node versions the project supports.                                   |
| `private: true`          | Prevents accidentally publishing the app to npm.                            |
| `bin`, `main`, `exports` | For **libraries and CLIs** you publish.                                     |

### Installing

```bash
npm install express            # adds to dependencies   | bun add express
npm install -D typescript      # adds to devDependencies | bun add -d typescript
npm uninstall express          # bun remove express
npm ci                         # clean, exact install from the lockfile (use in CI/Docker)
npx tsx script.ts              # run a package binary without installing it  | bunx
```

### Semantic versioning and the lockfile

`MAJOR.MINOR.PATCH`: patch = bug fixes, minor = new features (compatible), **major = breaking**.

| Range in package.json | Allows                                                  |
| --------------------- | ------------------------------------------------------- |
| `"^5.1.0"`            | `>=5.1.0 <6.0.0` (the default. Minor and patch updates) |
| `"~5.1.0"`            | `>=5.1.0 <5.2.0` (patch updates only)                   |
| `"5.1.0"`             | exactly this version                                    |

The **lockfile** (`package-lock.json` for npm, `bun.lock` for Bun, `pnpm-lock.yaml`) records the **exact** version of every package, including nested ones. **Commit it.** Then every developer, CI run and Docker build installs the identical tree. Use `npm ci` (not `npm install`) in automation: it fails if `package.json` and the lockfile disagree.

### Package managers

| Tool            | Notes                                                                                                            |
| --------------- | ---------------------------------------------------------------------------------------------------------------- |
| **npm**         | Ships with Node. Good default.                                                                                   |
| **pnpm**        | Fast, saves disk space with a shared content-addressable store, strict about dependencies. Popular in monorepos. |
| **yarn**        | The older alternative.                                                                                           |
| **bun install** | Very fast installs, reads `package.json`, writes `bun.lock`. Works for Node projects too.                        |

Use **one** package manager per project and commit only its lockfile.

### Supply-chain safety (a real security topic)

Every dependency is code you run with your app's full permissions. Attacks through malicious or hijacked packages happen regularly.

| Habit                                                                   | Why                                                |
| ----------------------------------------------------------------------- | -------------------------------------------------- |
| Commit the lockfile and use `npm ci` in CI                              | Reproducible, no surprise upgrades                 |
| Prefer fewer, well-maintained dependencies. Prefer built-ins            | Less attack surface (see the table in Topic 1.6)   |
| Review new dependencies and their install scripts                       | Install scripts (`postinstall`) run arbitrary code |
| Run `npm audit` and enable automated update PRs (Dependabot / Renovate) | Catch known vulnerabilities                        |
| Be careful with brand-new versions of a package                         | Malicious releases are often caught within days    |
| Consider `--ignore-scripts` for untrusted installs                      | Blocks install-time code execution                 |

#### Interview traps

- "`dependencies` vs. `devDependencies`?" Runtime vs. development-only. Production images are often built with `npm ci --omit=dev`, so a runtime import in a dev dependency breaks production.
- "`npm install` vs. `npm ci`?" `install` may update the lockfile. `ci` installs exactly what the lockfile says and fails on mismatch, so it is correct for CI and Docker.
- "Why does `^` sometimes cause 'works on my machine'?" Without a committed lockfile, different machines resolve `^` ranges to different versions. The lockfile pins them.

## 1.6 Globals and Built-In Web APIs

Modern Node.js and Bun ship many of the **same web-standard APIs as browsers**. You can use them without installing anything.

```js
// fetch: HTTP client (remember: it does NOT reject on 404/500, check res.ok)
const res = await fetch('https://jsonplaceholder.typicode.com/todos/1');
if (!res.ok) throw new Error(`HTTP ${res.status}`);
const todo = await res.json();

// URL and URLSearchParams
const url = new URL('https://api.example.com/tasks?status=done&page=2');
url.searchParams.get('page'); // "2"
url.searchParams.set('page', '3');

// AbortController / AbortSignal: cancel and time out
const signal = AbortSignal.timeout(5_000); // aborts after 5 seconds
await fetch('https://example.com', { signal });

// structuredClone: deep copy
const copy = structuredClone({ a: [1, 2, { b: 3 }] });

// crypto
crypto.randomUUID(); // "3b241101-e2bb-4255-8caf-4136c566a962"

// timers, microtasks, performance
setTimeout(() => {}, 100);
queueMicrotask(() => {});
performance.now();

// text <-> bytes
new TextEncoder().encode('héllo'); // Uint8Array
```

Other globals worth knowing: `Request`, `Response`, `Headers`, `FormData`, `Blob`, `ReadableStream`, `WebSocket` (client), `BroadcastChannel`, `Buffer`, `process`, `console`. Node 26 also enables the **`Temporal`** date-and-time API by default, the long-awaited replacement for the confusing `Date` object.

### "You might not need a package"

Many popular packages now have built-in replacements. Fewer dependencies means a smaller attack surface and faster installs.

| Instead of                         | Use the built-in                      | Notes                                 |
| ---------------------------------- | ------------------------------------- | ------------------------------------- |
| `axios`, `node-fetch`              | `fetch()`                             | Check `res.ok` yourself               |
| `nodemon`                          | `node --watch`                        | Bun: `bun --watch`                    |
| `dotenv`                           | `--env-file`, `process.loadEnvFile()` | Bun loads `.env` automatically        |
| `ts-node`, `tsx` (for running)     | Native `.ts` execution                | Keep `tsc --noEmit` for type checking |
| `jest`, `mocha` (for simple needs) | `node:test` + `node:assert`           | Bun: `bun test`                       |
| `uuid`                             | `crypto.randomUUID()`                 |                                       |
| `minimist`, `yargs` (simple CLIs)  | `util.parseArgs`                      |                                       |
| `better-sqlite3`                   | `node:sqlite`                         | Bun: `bun:sqlite`                     |
| `ws` (client only)                 | Global `WebSocket`                    |                                       |

Built-ins are not always enough (Express, a database driver or an ORM are still worth installing), but "can I do this with a built-in?" is the right first question.

#### Interview traps

- "How do you make an HTTP request in Node without a library?" The global `fetch` (powered by undici). Always check `res.ok`, and use `AbortSignal.timeout()` so a slow server cannot hang you forever.
- "What happens to `fetch` on a 404?" It resolves normally. Only network failures reject.
- "Why prefer `crypto.randomUUID()` over `Math.random()` for IDs?" `Math.random()` is not cryptographically secure and collides far more easily.

#### Tricky questions and answers

###### Q2 [SDE-2]: Your Node ESM script fails with `ERR_MODULE_NOT_FOUND` on `import { helper } from "./utils"`. The same code works in your Vite app. Why?

**Answer:** Vite (a bundler) resolves extensionless imports for you. Node's ESM loader follows the standard and requires an **explicit file extension**, so the import must be `./utils.js` (or `./utils.ts` when running TypeScript natively). Directory imports (`./utils` meaning `./utils/index.js`) also do not work in Node ESM. This is a frequent surprise when moving shared code from a front end to a back end.

###### Q3 [SDE-2]: Why would a team standardize on `npm ci` and commit the lockfile for a service, and what breaks if they do not?

**Answer:** The lockfile records the exact versions of every direct and transitive dependency. Without it, `npm install` resolves semver ranges (`^`) at install time, so two builds a week apart can ship different code. That causes unreproducible bugs, and it lets a freshly published (possibly compromised) version slip in unnoticed. `npm ci` installs exactly the locked tree and fails when `package.json` and the lockfile disagree, which makes builds deterministic and auditable.

###### Q4 [SDE-3]: "We run `node server.ts` in production with native type stripping, so we removed the build step." What are the risks?

**Answer:** Type stripping only supports **erasable** syntax (no `enum`, runtime `namespace`, or parameter properties without an extra flag) and needs `.ts` extensions in imports, so some code must be rewritten. More importantly, nothing is **type-checked at runtime**: you must still run `tsc --noEmit` in CI or the types are decoration. You also lose build-time steps such as bundling, tree-shaking and source-map generation, and some tooling and APIs expect compiled output. For small services it is a great simplification, but treat type checking in CI as mandatory.

---

## Hands-On Lab: Build a `tasks` CLI That Runs on Node and Bun

```
1. mkdir tasks-cli && cd tasks-cli && npm init -y && npm pkg set type=module
2. Create cli.ts that supports:
     node cli.ts add --title "Buy milk"
     node cli.ts list
     node cli.ts done --id 2
   - Parse arguments with util.parseArgs
   - Store tasks in tasks.json next to the script using import.meta.dirname
     (read and write with node:fs/promises, which you will study in Module 3)
   - Use crypto.randomUUID() or an incrementing number for ids
   - Print a friendly error and set process.exitCode = 1 for unknown commands
3. Run it with Node (node cli.ts ...) and with Bun (bun cli.ts ...). Does the
   output match?
4. Add "dev": "node --watch cli.ts list" and watch it re-run when you edit the file.
5. Add .env with APP_NAME=TaskCLI and print it using --env-file (Node) and
   automatic loading (Bun).
6. Add typescript and @types/node as devDependencies and a "typecheck" script
   (tsc --noEmit). Introduce a type error on purpose. Does `node cli.ts` still run?
   Does `npm run typecheck` fail?
7. Run npm ls --all (or bun pm ls) and count your dependency tree.
   Which dependency could be replaced by a built-in?
8. Commit the lockfile. Delete node_modules and run npm ci.
```

Questions to answer for yourself:

- Which Node features in your CLI replaced an npm package you would normally install?
- What happens when `tasks.json` does not exist yet? Which error code does the file system give (Module 3 will name it)?
- Which of your imports needed a file extension, and why?

---

## Module 1 Cheat Sheet

| Concept              | One-line interview answer                                                                                              |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Node.js              | A JavaScript **runtime**: V8 + libuv + core modules. Not a language or framework.                                      |
| Bun                  | An all-in-one runtime (JavaScriptCore) with a built-in package manager, test runner and bundler.                       |
| Single-threaded?     | Your JS runs on one thread. Node uses a thread pool and the OS underneath for I/O.                                     |
| LTS in 2026          | Node 26 is Active LTS. 24 is supported. 22 is maintenance. 20 is end-of-life. From Node 27, every release becomes LTS. |
| Version manager      | fnm / nvm. Pin with `.nvmrc` and `engines`.                                                                            |
| Watch / env / TS     | `node --watch`, `--env-file`, native `.ts` (type stripping). Bun does all three automatically.                         |
| Type stripping       | Erases types, never type-checks. No `enum` / runtime `namespace`. Run `tsc --noEmit` in CI.                            |
| `process`            | `argv`, `env`, `cwd()`, `exitCode`, signals. Env values are always strings.                                            |
| ESM vs. CJS          | `import` / `export`, async, top-level `await`, standard. vs. `require`, sync, Node-only.                               |
| `node:` prefix       | Marks a built-in. Always use it.                                                                                       |
| ESM extension rule   | Node ESM needs explicit extensions (`./x.js`). Bundlers do not.                                                        |
| `__dirname` in ESM   | `import.meta.dirname` and `import.meta.filename`.                                                                      |
| `package.json`       | Name, scripts, `type`, `dependencies` (runtime), `devDependencies` (tooling), `engines`.                               |
| Semver               | `^` allows minor and patch. `~` allows patch. Major is breaking.                                                       |
| Lockfile             | Exact versions of the whole tree. Commit it. Use `npm ci` in CI and Docker.                                            |
| Supply-chain hygiene | Fewer deps, lockfile, audit, review install scripts, prefer built-ins.                                                 |
| Web APIs in Node     | `fetch`, `URL`, `AbortSignal.timeout`, `structuredClone`, `crypto.randomUUID`, `Temporal` (26).                        |
