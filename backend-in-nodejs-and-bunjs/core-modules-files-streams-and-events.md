---
title: "Module 3: Core Modules - Files, Streams, Buffers & Events"
description: "Master the Node.js built-in modules that power every backend: the file system and paths, buffers and encodings, EventEmitter, streams and backpressure, crypto, child processes, and validated configuration - with Bun equivalents."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/nodejs-module-3.png"
tags: [Node.js, Bun, Streams, File-System, EventEmitter, Crypto, Interview-Prep]
keywords: ["Node.js streams explained", "Node.js backpressure", "Node.js fs promises", "Node.js Buffer tutorial", "EventEmitter Node.js", "Node.js crypto HMAC", "Node.js path traversal", "Bun.file Bun.write"]
---

# Module 3: Core Modules - Files, Streams, Buffers & Events

![Node.js Module 3](/images/nodejs-module-3.png)

## Words you will meet in this module

| Word                    | Simple meaning                                                                                     |
| ----------------------- | -------------------------------------------------------------------------------------------------- |
| **Core module**         | A library built into Node.js, imported with the `node:` prefix (`node:fs`, `node:path`...).        |
| **Buffer**              | A fixed-size chunk of raw bytes. A `Uint8Array` with extra helper methods.                         |
| **Encoding**            | How bytes map to text: `utf8`, `hex`, `base64`, `base64url`.                                       |
| **EventEmitter**        | An object that lets code subscribe to named events (`on`) and announce them (`emit`).              |
| **Stream**              | Data that arrives or leaves in **small chunks over time**, instead of all at once.                 |
| **Readable / Writable** | A stream you read from (file, request) / a stream you write to (file, response).                   |
| **Transform**           | A stream in the middle that changes the data (gzip, uppercase, parse).                             |
| **Backpressure**        | A slow receiver telling a fast sender to pause so memory does not blow up.                         |
| **`highWaterMark`**     | A stream's internal buffer size limit (about 64 KB for file streams).                              |
| **HMAC**                | A hash that also uses a secret key. Proves a message was not changed and came from the key holder. |
| **TOCTOU**              | "Time of check, time of use" race: checking a condition and acting on it later, after it changed.  |
| **Path traversal**      | An attack using `../` in a path to escape the folder you meant to serve.                           |

## 3.1 Files and Paths

### Use the promise API

```js
import {
  readFile,
  writeFile,
  appendFile,
  mkdir,
  readdir,
  stat,
  rm,
  rename,
  copyFile,
} from 'node:fs/promises';
import path from 'node:path';

const dir = path.join(import.meta.dirname, 'data');

await mkdir(dir, { recursive: true }); // no error if it already exists
await writeFile(path.join(dir, 'a.txt'), 'hello\n'); // creates or overwrites
await appendFile(path.join(dir, 'a.txt'), 'world\n');
const text = await readFile(path.join(dir, 'a.txt'), 'utf8'); // without "utf8" you get a Buffer

const entries = await readdir(dir, { withFileTypes: true });
for (const e of entries) console.log(e.name, e.isDirectory() ? 'dir' : 'file');

const info = await stat(path.join(dir, 'a.txt'));
console.log(info.size, info.mtime, info.isFile());

await rename(path.join(dir, 'a.txt'), path.join(dir, 'b.txt'));
await rm(dir, { recursive: true, force: true }); // like rm -rf. `force` ignores "not found"
```

`readdir(dir, { recursive: true })` lists nested folders too. The older callback and `*Sync` versions exist (`fs.readFileSync`). Reserve `*Sync` for startup code and scripts, never for request handlers.

> **Bun note:** Bun adds fast, ergonomic file helpers. `Bun.file(path)` is a lazy handle, and `Bun.write` writes strings, buffers, other files or even a `Response`:
>
> ```js
> const f = Bun.file('data/a.txt');
> const text = await f.text(); // also .json(), .bytes(), .stream()
> console.log(f.size, await f.exists());
> await Bun.write('data/b.txt', 'hello');
> ```
>
> `node:fs` works in Bun too.

### Errors you will meet (check `err.code`)

| Code                 | Meaning                                              | Typical reaction                                       |
| -------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| `ENOENT`             | No such file or directory                            | Create it, use a default, or return 404                |
| `EEXIST`             | File already exists                                  | Pick another name or accept it                         |
| `EACCES`, `EPERM`    | Permission denied                                    | Fix permissions, report a clear error                  |
| `EISDIR` / `ENOTDIR` | Expected a file but got a directory (or the reverse) | Validate input                                         |
| `EMFILE`             | Too many open files                                  | Close handles, limit concurrency (Module 2 `mapLimit`) |
| `ENOSPC`             | Disk full                                            | Alert. Free space                                      |

```js
try {
  const raw = await readFile(configPath, 'utf8');
  return JSON.parse(raw);
} catch (err) {
  if (err.code === 'ENOENT') return {}; // missing file is fine
  throw err; // anything else is a real problem
}
```

**Do not** check with `access()` or `exists()` and then open the file. The file may change between the check and the open (**TOCTOU**). Just try the operation and handle the error.

### Writing files safely: the atomic write

`writeFile` truncates the file first. If the process crashes mid-write, you are left with a half-written (corrupt) file. Write to a temporary file, then **rename**, which is atomic on the same filesystem:

```js
async function writeJsonAtomic(file, data) {
  const tmp = `${file}.${process.pid}.tmp`;
  await writeFile(tmp, JSON.stringify(data, null, 2));
  await rename(tmp, file); // readers see the old file or the new file, never half of one
}
```

### `path`: never build paths with string concatenation

| Function                              | Use                                                             |
| ------------------------------------- | --------------------------------------------------------------- |
| `path.join("a", "b", "../c")`         | Join segments and normalize → `a/c`. Uses `/` or `\` per OS.    |
| `path.resolve("x", "y")`              | Build an **absolute** path (relative to the current directory). |
| `path.basename`, `extname`, `dirname` | Parts of a path.                                                |
| `path.parse(p)` / `path.format`       | Break into `{ dir, name, ext }` and back.                       |
| `path.relative(from, to)`             | The relative path between two locations.                        |

### Path traversal (a classic vulnerability)

If a user can influence a file name, `../../etc/passwd` can escape your folder:

```js
// VULNERABLE
app.get('/files/:name', (req, res) =>
  res.sendFile(path.join(uploadsDir, req.params.name)),
);
// GET /files/..%2f..%2fetc%2fpasswd  →  reads a system file

// SAFE: resolve, then verify the result is still inside the base directory
function safeJoin(baseDir, userPath) {
  const base = path.resolve(baseDir);
  const full = path.resolve(base, userPath);
  if (full !== base && !full.startsWith(base + path.sep)) {
    throw new AppError('Forbidden', { status: 403 });
  }
  return full;
}
```

For stronger safety, also resolve symlinks with `fs.realpath`, and prefer storing files under **generated IDs** (UUIDs) instead of user-supplied names.

#### Interview traps

- "`readFile` vs. `createReadStream`?" `readFile` loads the whole file into memory. A stream processes it in chunks. For big files, use a stream (Topic 3.4).
- "Why not `fs.existsSync` then `readFile`?" Race condition (TOCTOU). Try and catch `ENOENT`.
- "`path.join` vs. `path.resolve`?" `join` concatenates and normalizes. `resolve` always returns an absolute path, processing right to left like a series of `cd` commands.
- "How do you prevent path traversal?" Resolve against a base directory, then verify the result starts with that base plus a separator. Better, never use user-supplied names as paths.

## 3.2 Buffers and Encodings

Computers store everything as **bytes**. Text is bytes plus an **encoding** that says how to turn them into characters. A **Buffer** is Node's container for raw bytes, and it is a subclass of `Uint8Array`. You meet buffers when dealing with files, sockets, crypto, and images.

```js
const buf = Buffer.from('héllo', 'utf8');
console.log(buf); // <Buffer 68 c3 a9 6c 6c 6f>
console.log('héllo'.length); // 5 characters
console.log(buf.length); // 6 BYTES (é takes 2 bytes in UTF-8)

buf.toString('utf8'); // "héllo"
buf.toString('hex'); // "68c3a96c6c6f"
buf.toString('base64'); // "aMOpbGxv"
buf.toString('base64url'); // URL-safe base64, used in JWTs

Buffer.from('aMOpbGxv', 'base64'); // decode
Buffer.alloc(16); // 16 zero-filled bytes (safe)
Buffer.concat([bufA, bufB]);
bufA.equals(bufB);
```

### Reading binary data

```js
const header = Buffer.from([0x00, 0x00, 0x01, 0x2c, 0x01]);
header.readUInt32BE(0); // 300   (big-endian 32-bit integer at offset 0)
header[4]; // 1     (a single byte)
```

### Traps with buffers

| Trap                                              | Detail                                                                                     |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **`.length` is bytes, not characters**            | Use it carefully for text limits. Emoji take 4 bytes.                                      |
| **`subarray` / `slice` share memory**             | Changing the slice changes the original. Copy with `Buffer.from(buf)` when needed.         |
| **`Buffer.allocUnsafe(n)`**                       | Faster, but may contain **old memory contents**. Never send it without overwriting.        |
| **Splitting multi-byte characters across chunks** | A stream chunk can end mid-character. Use `stream.setEncoding("utf8")` or `StringDecoder`. |
| **Large buffers**                                 | A single buffer has a size limit (about 2 GB or less). Stream big data instead.            |

> **Bun note:** `Buffer` works in Bun, and Bun encourages the standard `Uint8Array` and `TextEncoder`/`TextDecoder`. Bun's release notes report large speedups for `Buffer` reads and writes.

## 3.3 `EventEmitter`

The **observer pattern**: an object announces "something happened" by emitting a named **event**, and any code that cares has registered a listener. Streams, HTTP servers, sockets and process signals are all event emitters, so understanding this is essential.

```js
import { EventEmitter } from 'node:events';

class OrderService extends EventEmitter {
  place(order) {
    // ...save the order...
    this.emit('order:placed', order); // listeners run SYNCHRONOUSLY, in the order registered
  }
}

const orders = new OrderService();

orders.on('order:placed', (o) => console.log('send email for', o.id));
orders.on('order:placed', (o) => console.log('update stock for', o.id));
orders.once('order:placed', (o) => console.log('first order ever:', o.id)); // runs only once

orders.place({ id: 42 });
```

### The methods

| Method                             | Meaning                                              |
| ---------------------------------- | ---------------------------------------------------- |
| `on(name, fn)` / `addListener`     | Subscribe                                            |
| `once(name, fn)`                   | Subscribe for a single call                          |
| `off(name, fn)` / `removeListener` | Unsubscribe (you need the same function reference)   |
| `emit(name, ...args)`              | Call all listeners. Returns `true` if there were any |
| `removeAllListeners(name)`         | Remove everything for an event                       |

### Rules you must know

1. **Listeners run synchronously** inside `emit`, in registration order. A slow listener delays the code that called `emit`.
2. **The special `"error"` event:** if you `emit("error", err)` and there is **no** listener, Node **throws** the error, which can crash the process. Always register an `error` listener on emitters that can fail (streams, sockets, your own).
3. **Memory leaks:** adding listeners repeatedly without removing them (in a loop, per request) leaks memory. Node warns after 10 listeners on one event (`MaxListenersExceededWarning`). Treat that warning as a real bug, not a setting to raise.
4. `emit` does not await anything. Async listeners that reject are not caught by the emitter (use `events.captureRejections` or handle errors inside the listener).

### Waiting for an event with promises

```js
import { once, on } from 'node:events';

const [order] = await once(orders, 'order:placed'); // resolves with the event's args

for await (const [o] of on(orders, 'order:placed')) {
  // async-iterate events
  console.log(o.id);
}
```

#### Interview traps

- "Are `EventEmitter` listeners synchronous or asynchronous?" **Synchronous.** `emit` calls them immediately, one by one. Use `setImmediate`/`queueMicrotask` inside a listener if you need to defer work.
- "What happens if you emit `error` with no listener?" The error is thrown. In an async context it becomes an uncaught exception and can crash the process.
- "How does an `EventEmitter` leak memory?" Listeners hold references to closures and objects. Registering new listeners per request on a long-lived emitter without removing them grows memory forever.

## 3.4 Streams and Backpressure

You have a 5 GB log file. `readFile` would try to put **all 5 GB into memory** at once. It will fail or take the server down. A **stream** reads the file in small chunks (64 KB at a time), hands each chunk to the next step, and discards it. Memory stays tiny no matter how big the file is. Think of a **conveyor belt** instead of carrying the whole warehouse in your arms.

```
 file ──► [Readable] ──chunk──► [Transform: gzip] ──chunk──► [Writable] ──► file.gz
          64 KB at a time                                     (or an HTTP response)
```

### The four kinds

| Type          | Direction      | Examples                                                            |
| ------------- | -------------- | ------------------------------------------------------------------- |
| **Readable**  | produces data  | `fs.createReadStream`, HTTP **request** (`req`), `process.stdin`    |
| **Writable**  | consumes data  | `fs.createWriteStream`, HTTP **response** (`res`), `process.stdout` |
| **Duplex**    | both           | TCP socket, WebSocket                                               |
| **Transform** | both, modifies | `zlib.createGzip()`, `crypto.createCipheriv()`, your own parser     |

### Use `pipeline`, not `.pipe()`

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';
import { pipeline } from 'node:stream/promises';

await pipeline(
  createReadStream('big.log'),
  createGzip(),
  createWriteStream('big.log.gz'),
);
console.log('compressed');
```

`pipeline` connects the streams, handles **backpressure**, **forwards errors**, and **destroys every stream** if one fails. The old `a.pipe(b).pipe(c)` does not forward errors or clean up, which leaks file descriptors. Always prefer `pipeline`.

### Custom transforms with async generators

The simplest way to write one:

```js
await pipeline(
  createReadStream('in.txt', { encoding: 'utf8' }),
  async function* (source) {
    for await (const chunk of source) yield chunk.toUpperCase();
  },
  createWriteStream('out.txt'),
);
```

Or the class form:

```js
import { Transform } from 'node:stream';
const upper = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  },
});
```

### Reading a huge file line by line

```js
import { createReadStream } from 'node:fs';
import readline from 'node:readline';

const rl = readline.createInterface({
  input: createReadStream('access.log'),
  crlfDelay: Infinity, // treat \r\n as one newline
});

const counts = new Map();
for await (const line of rl) {
  const status = line.split(' ')[8];
  counts.set(status, (counts.get(status) ?? 0) + 1);
}
console.log(counts); // constant memory for any file size
```

### Backpressure, explained

A fast producer and a slow consumer: if the producer keeps pushing, chunks pile up in memory.

```
 fast Readable ─►─►─►─►─►─► [ buffer filling up ... ] ─► slow Writable (a client on a bad connection)
                              ▲
                              └─ without backpressure this grows until the process runs out of memory
```

Streams have an internal buffer limit (`highWaterMark`). When it is full, `writable.write(chunk)` returns **`false`**: "stop, I'm full." You must wait for the **`"drain"`** event before writing more. `pipeline` does this for you. If you write manually:

```js
import { once } from 'node:events';

async function writeMany(out, lines) {
  for (const line of lines) {
    if (!out.write(line + '\n')) {
      await once(out, 'drain'); // respect backpressure
    }
  }
  out.end();
}
```

### Streams in HTTP: the real payoff

```js
import http from 'node:http';
import { createReadStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';

http
  .createServer(async (req, res) => {
    if (req.url === '/video') {
      res.setHeader('Content-Type', 'video/mp4');
      try {
        await pipeline(createReadStream('movie.mp4'), res); // streams to the client with backpressure
      } catch {
        res.destroy(); // client left mid-download
      }
    }
  })
  .listen(3000);
```

Serving a 2 GB file this way uses a few hundred KB of RAM. With `readFile` it would use 2 GB **per concurrent download**.

### Web Streams and interop

`fetch` bodies, `Request`, and `Response` use the **standard Web Streams API** (`ReadableStream`). Node streams and Web streams convert both ways:

```js
import { Readable } from 'node:stream';

const res = await fetch('https://example.com/big.csv');
await pipeline(Readable.fromWeb(res.body), createWriteStream('big.csv')); // download to disk

const webStream = Readable.toWeb(createReadStream('a.txt'));
```

> **Bun note:** Bun is built on web streams. `new Response(Bun.file("movie.mp4"))` streams a file efficiently with zero ceremony, and `Bun.write("big.csv", await fetch(url))` streams a response to disk.

### When to use a stream

| Use a stream when...                                      | Use whole-file / whole-body methods when...      |
| --------------------------------------------------------- | ------------------------------------------------ |
| Data is large or of unknown size (uploads, logs, exports) | The data is small and fixed (config, small JSON) |
| You want to start sending before everything is ready      | You need all of it at once (parse JSON)          |
| You are transforming (compress, encrypt, parse CSV)       | A single read is simpler and safe                |

#### Interview traps

- "What is backpressure?" The mechanism by which a slow consumer signals a fast producer to pause, so buffers do not grow without bound. `write()` returning `false` plus the `drain` event, handled automatically by `pipeline`.
- "`pipe` vs. `pipeline`?" `pipeline` propagates errors, cleans up all streams on failure, and returns a promise. `pipe` does not, which leaks resources.
- "Why would `readFile` fail on a 3 GB file?" It tries to load everything into memory (and buffers and strings have size limits). Stream it.
- "What is the default chunk size?" File streams: **64 KB** (`highWaterMark`). Generic streams: 16 KB.

#### Tricky question and answer

###### Q1 [SDE-2]: An endpoint that exports 2 million rows as CSV builds the whole string in memory and the process runs out of memory at about 10 concurrent exports. How do you redesign it?

**Answer:** Stream it. Query the database with a **cursor or paged reads**, convert each batch of rows to CSV text inside an async generator or Transform, and `pipeline` that to the HTTP response (`Content-Type: text/csv`, `Content-Disposition: attachment`). Memory per export stays at one batch plus the stream buffers, and backpressure from a slow client automatically slows the database reads. Add a row or time limit and authentication, and consider generating very large exports in a background job to object storage (Module 7).

## 3.5 Other Built-Ins: `crypto`, `util`, `child_process`, `os` and Configuration

### `node:crypto`

```js
import {
  randomUUID,
  randomBytes,
  randomInt,
  createHash,
  createHmac,
  timingSafeEqual,
} from 'node:crypto';

randomUUID(); // unique ID
randomBytes(32).toString('base64url'); // an unguessable token (sessions, reset links)
randomInt(0, 1_000_000); // cryptographically secure random integer

createHash('sha256').update('hello').digest('hex'); // fast hash: checksums, cache keys.
// NOT for passwords (Module 6)
```

**HMAC: verify a webhook came from who it claims** (Stripe, GitHub and others sign their requests):

```js
function verifySignature(rawBody, signatureHex, secret) {
  const expected = createHmac('sha256', secret).update(rawBody).digest();
  const given = Buffer.from(signatureHex, 'hex');
  return given.length === expected.length && timingSafeEqual(given, expected); // constant-time compare
}
```

Always compare secrets with `timingSafeEqual`, never `===`, because `===` returns faster on an early mismatch and leaks information through timing.

**Encrypt data at rest with AES-256-GCM** (authenticated encryption):

```js
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

export function encrypt(plaintext, key /* 32-byte Buffer */) {
  const iv = randomBytes(12); // new random IV for EVERY message
  const cipher = createCipheriv('aes-256-gcm', key, iv);
  const data = Buffer.concat([
    cipher.update(plaintext, 'utf8'),
    cipher.final(),
  ]);
  return Buffer.concat([iv, cipher.getAuthTag(), data]).toString('base64');
}

export function decrypt(b64, key) {
  const buf = Buffer.from(b64, 'base64');
  const decipher = createDecipheriv('aes-256-gcm', key, buf.subarray(0, 12));
  decipher.setAuthTag(buf.subarray(12, 28));
  return Buffer.concat([
    decipher.update(buf.subarray(28)),
    decipher.final(),
  ]).toString('utf8');
}
```

Never reuse an IV with the same key. Keep keys in a secret manager or environment, never in code. And: **do not invent your own cryptography.** Use these standard primitives as shown.

### `node:child_process`: running other programs

```js
import { execFile, spawn } from 'node:child_process';
import { promisify } from 'node:util';
const execFileAsync = promisify(execFile);

// SAFE: arguments are passed as an array. No shell is involved
const { stdout } = await execFileAsync('git', ['log', '--oneline', '-5']);

// DANGEROUS: command injection if `name` comes from a user
// exec(`convert ${name}.png out.jpg`)      // name = "x; rm -rf /"
```

| Function   | Shell?  | Use                                                                  |
| ---------- | ------- | -------------------------------------------------------------------- |
| `execFile` | No      | Run one program with an args array. Buffered output. **Preferred**.  |
| `spawn`    | No      | Same, but **streams** stdout/stderr. For long-running or big output. |
| `exec`     | **Yes** | Runs a shell command string. Avoid with any user-influenced input.   |
| `fork`     | No      | Start another Node script with a message channel.                    |

### Small but useful modules

```js
import os from 'node:os';
os.availableParallelism(); // number of CPU cores to use
os.tmpdir();
os.homedir();
os.totalmem();
os.freemem();

import { fileURLToPath, pathToFileURL } from 'node:url';
fileURLToPath(import.meta.url); // same as import.meta.filename

import { promisify, inspect, parseArgs, styleText } from 'node:util';
console.log(inspect(deepObject, { depth: null, colors: true }));

import { setTimeout as sleep } from 'node:timers/promises';
```

### Validate your configuration at startup

A server that boots with a missing `DATABASE_URL` and fails an hour later in production is a bad design. **Fail fast**: validate every environment variable when the process starts.

```js
// src/env.js
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z
    .enum(['development', 'test', 'production'])
    .default('development'),
  PORT: z.coerce.number().int().positive().default(3000), // env values are strings: coerce
  DATABASE_URL: z.string().url(),
  SESSION_SECRET: z.string().min(32),
  CORS_ORIGIN: z.string().default('http://localhost:5173'),
});

const parsed = EnvSchema.safeParse(process.env);
if (!parsed.success) {
  console.error('Invalid environment configuration:', parsed.error.issues);
  process.exit(1);
}

export const env = parsed.data; // typed, validated, defaulted, used everywhere instead of process.env
```

Import `env` everywhere instead of touching `process.env` directly. Zod is introduced properly in Module 4, Topic 4.6.

#### Interview traps

- "`exec` vs. `execFile`?" `exec` runs through a shell, so metacharacters in user input become command injection. `execFile` and `spawn` pass arguments as an array with no shell.
- "Why `timingSafeEqual`?" To compare secrets in constant time. `===` short-circuits on the first different byte, which can leak the secret one byte at a time through timing differences.
- "Is SHA-256 OK for passwords?" No. It is fast, which is exactly the problem (Module 6). Use a slow password hash (argon2id, scrypt, bcrypt).
- "Why validate `process.env` at startup?" Environment values are untyped strings. Failing fast with a clear message beats a mysterious failure later.

#### Tricky questions and answers

###### Q2 [SDE-2]: You attach a `data` listener to a readable stream and write to a slow client with `res.write(chunk)` without checking the return value. What happens under load?

**Answer:** `write` returns `false` when the writable's buffer is full, but the data is still queued. If you ignore it and keep reading at full speed, the queue grows in memory, and with many slow clients the process eventually runs out of memory. The fix is backpressure: pause the source when `write` returns `false` and resume on `"drain"`, or simply use `pipeline(source, res)` which does it automatically.

###### Q3 [SDE-3]: Walk through exactly how `emit("data")` and async listeners interact, and what could go wrong if a listener is `async` and throws.

**Answer:** `emit` calls listeners synchronously in order and ignores their return values. An `async` listener returns a promise that the emitter never sees, so a rejection becomes an **unhandled rejection**, which crashes modern Node by default. The caller of `emit` also cannot wait for async listeners to finish. Wrap the body in `try/catch`, or construct the emitter with `{ captureRejections: true }` so rejections are routed to the `error` event, and if ordering or completion matters, use a different pattern such as an explicit async pipeline or a queue.

---

## Hands-On Lab: Stream a Million-Line Log

```
1. Generate data. Write gen-log.js that writes 1,000,000 fake access-log lines to
   access.log using createWriteStream. Respect backpressure (check write() and await "drain").
   Line format:  203.0.113.7 - - [04/Nov/2026:10:00:00 +0000] "GET /api/tasks HTTP/1.1" 200 512
   Randomize ip, path (5 options), status (200, 201, 404, 500), and bytes.
2. Analyze with streaming. Write analyze.js using readline over createReadStream:
   - count requests per status code
   - find the top 5 paths
   - sum bytes
   Print results and process.memoryUsage().rss. It should stay small (tens of MB).
3. Compare. Write analyze-bad.js using readFile and split("\n"). Check the memory
   (rss). Now try with a 10x bigger file. What happens?
4. Compress with pipeline. Gzip access.log into access.log.gz with pipeline().
   Then gunzip it back and verify the sizes match.
5. Write a Transform (async generator) that keeps only lines with status 500
   and writes errors.log. Chain it in one pipeline: read → filter → gzip → write.
6. Serve it. Create a tiny node:http server with GET /logs that streams
   access.log to the client with pipeline. Download with curl and watch memory
   on the server while 5 curls run at once.
7. Add env.js (Zod validation) and make the log path and PORT configurable.
   What happens if you start the server with a missing variable?
8. Write signature.js: sign a JSON body with HMAC-SHA256 and verify it with
   timingSafeEqual. Tamper with one byte and confirm verification fails.
9. Run steps 2 and 6 with Bun, using Bun.file and new Response(Bun.file(...)).
```

Questions to answer for yourself:

- What would break in step 1 if you ignored `write()`'s return value?
- Why does the streaming version's memory stay flat while the file grows?
- In step 6, what does the server do when a client disconnects halfway? Where must you clean up?

---

## Module 3 Cheat Sheet

| Concept                 | One-line interview answer                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| `node:fs/promises`      | Promise-based file API. Avoid `*Sync` in request handlers.                                       |
| File error codes        | `ENOENT` (missing), `EEXIST`, `EACCES`/`EPERM`, `EMFILE` (too many open files), `ENOSPC`.        |
| TOCTOU                  | Check-then-use race. Do not `exists()` first. Try the operation and handle the error.            |
| Atomic write            | Write a temp file, then `rename` over the target.                                                |
| `path.join` / `resolve` | Join and normalize / make absolute. Never concatenate paths with strings.                        |
| Path traversal          | `../` escaping the base dir. Resolve and verify the result is inside the base.                   |
| Buffer                  | Raw bytes (`Uint8Array`). `.length` is bytes. `subarray` shares memory. Avoid `allocUnsafe`.     |
| Encodings               | `utf8`, `hex`, `base64`, `base64url`.                                                            |
| `EventEmitter`          | Observer pattern. Listeners run **synchronously**. Unhandled `"error"` throws.                   |
| Listener leaks          | Adding listeners repeatedly without removing them. Heed `MaxListenersExceededWarning`.           |
| Streams                 | Chunked data flow in constant memory. Readable, Writable, Duplex, Transform.                     |
| `pipeline`              | Connects streams with backpressure, error forwarding and cleanup. Prefer it over `.pipe()`.      |
| Backpressure            | `write()` returns `false` → wait for `drain`. Stops fast producers from flooding slow consumers. |
| `highWaterMark`         | Internal buffer limit. 64 KB for file streams.                                                   |
| Line-by-line            | `readline` over a read stream with `for await`.                                                  |
| Web Streams interop     | `Readable.fromWeb` / `toWeb`. `fetch` bodies are web streams.                                    |
| `crypto` basics         | `randomUUID`, `randomBytes` for tokens, `createHmac` + `timingSafeEqual` for signatures.         |
| Password hashing        | Not SHA-256. Use argon2id / scrypt / bcrypt (Module 6).                                          |
| `child_process`         | Prefer `execFile` / `spawn` with an args array. `exec` + user input = command injection.         |
| Env validation          | Parse `process.env` with a schema at startup and fail fast.                                      |
| Bun equivalents         | `Bun.file`, `Bun.write`, `new Response(Bun.file(...))`. `node:*` modules also work.              |
