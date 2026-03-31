# Fastify Streams — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to
> master Fastify request/response streaming from internals and tests.

---

## Table of Contents

1. [Why Streams Matter in Fastify](#1-why-streams-matter-in-fastify)
2. [Two Stream Pipelines You Must Know](#2-two-stream-pipelines-you-must-know)
3. [Outgoing Streams (`reply.send`) — Deep Dive](#3-outgoing-streams-replysend--deep-dive)
   - 3.1 [Node Readable streams](#31-node-readable-streams)
   - 3.2 [WHATWG `ReadableStream`](#32-whatwg-readablestream)
   - 3.3 [WHATWG `Response`](#33-whatwg-response)
   - 3.4 [Headers, status, serialization, and hooks](#34-headers-status-serialization-and-hooks)
4. [Incoming Request Streams — Deep Dive](#4-incoming-request-streams--deep-dive)
   - 4.1 [`preParsing` stream replacement contract](#41-preparsing-stream-replacement-contract)
   - 4.2 [`receivedEncodedLength`, `content-length`, and `bodyLimit`](#42-receivedencodedlength-content-length-and-bodylimit)
   - 4.3 [Custom content-type parsers with streams](#43-custom-content-type-parsers-with-streams)
5. [Error Paths and Stream Lifecycle](#5-error-paths-and-stream-lifecycle)
6. [HEAD, Abort, and Premature Close Behavior](#6-head-abort-and-premature-close-behavior)
7. [TS-First End-to-End Example](#7-ts-first-end-to-end-example)
8. [JavaScript Equivalent Snippets](#8-javascript-equivalent-snippets)
9. [Common Pitfalls and Fast Fixes](#9-common-pitfalls-and-fast-fixes)
10. [Source Grounding Map](#10-source-grounding-map)

---

## 1. Why Streams Matter in Fastify

Streams are where Fastify performance and correctness meet:

- **Low memory footprint:** large payloads do not need full buffering.
- **Backpressure-aware output:** Fastify pipes stream data directly to Node's
  response stream.
- **Composable input transformations:** you can replace incoming payload streams
  in `preParsing`.
- **Interop across APIs:** Fastify supports Node streams, WHATWG
  `ReadableStream`, and WHATWG `Response`.

Fastify stream handling is implemented mostly in:

- [lib/reply.js](../../lib/reply.js)
- [lib/route.js](../../lib/route.js)
- [lib/hooks.js](../../lib/hooks.js)
- [lib/content-type-parser.js](../../lib/content-type-parser.js)

---

## 2. Two Stream Pipelines You Must Know

There are two different stream paths.

### A) Outgoing stream path (response)

```text
handler -> reply.send(payload)
        -> onSend hooks
        -> stream/web-stream/response path
        -> write to socket
        -> onResponse
```

### B) Incoming stream path (request body)

```text
raw request stream
  -> preParsing hooks (optional stream replacement)
  -> content-type parser consumes stream
  -> request.body
  -> preValidation / preHandler / handler
```

Keep these separate mentally: output stream semantics are owned by `reply`,
while input stream semantics are owned by routing + content-type parsing.

---

## 3. Outgoing Streams (`reply.send`) — Deep Dive

At send-time Fastify classifies the payload. In [lib/reply.js](../../lib/reply.js),
`reply.send(payload)` treats these as pre-serialized stream-like values:

- Node stream (`payload.pipe`)
- WHATWG stream (`payload.getReader`)
- WHATWG `Response` (`[object Response]`)

Those payloads skip JSON serialization and go through stream-specific send logic.

### 3.1 Node Readable streams

TS-first route example:

```typescript
import Fastify, { FastifyInstance } from 'fastify'
import { createReadStream } from 'node:fs'

const app: FastifyInstance = Fastify()

app.get('/file', async (_request, reply) => {
  const source = createReadStream('./large-file.txt')
  reply.type('text/plain; charset=utf-8')
  return reply.send(source)
})
```

Important behavior from tests:

- Stream payloads are sent as-is (`test/stream.1.test.js`).
- If no `content-type` is set, runtime behavior for Node streams can be `null`
  in response headers (`test/stream.1.test.js`).
- `onSend` can replace payload with another stream (`test/stream.2.test.js`).

### 3.2 WHATWG `ReadableStream`

TS-first example:

```typescript
import Fastify from 'fastify'
import { createReadStream } from 'node:fs'
import { Readable } from 'node:stream'

const app = Fastify()

app.get('/web-stream', async (_request, reply) => {
  const nodeStream = createReadStream('./large-file.txt')
  const webStream = Readable.toWeb(nodeStream)
  return reply.send(webStream)
})
```

Locking rules matter:

- Sending a locked `ReadableStream` returns 500 with
  `FST_ERR_REP_READABLE_STREAM_LOCKED`
  (`test/reply-web-stream-locked.test.js`, `test/web-api.test.js`).

### 3.3 WHATWG `Response`

`Response` lets you carry status, headers, and body as one payload:

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.get('/proxy', async () => {
  const upstream = await fetch('https://example.com/data')
  return new Response(upstream.body, {
    status: upstream.status,
    headers: upstream.headers
  })
})
```

Edge cases covered by tests (`test/web-api.test.js`):

- `Response.bodyUsed === true` before `reply.send(response)` results in
  `FST_ERR_REP_RESPONSE_BODY_CONSUMED`.
- A locked response body stream fails with
  `FST_ERR_REP_READABLE_STREAM_LOCKED`.

### 3.4 Headers, status, serialization, and hooks

- Stream-like payloads are treated as pre-serialized.
- `onSend` receives stream/Response payloads and may replace them.
- For async handlers, `return reply.send(stream)` is the safe pattern.
- Stream errors before headers are sent can flow through Fastify error handling.

---

## 4. Incoming Request Streams — Deep Dive

Incoming payload handling starts in route orchestration and lands in
content-type parsing.

- Route path setup: [lib/route.js](../../lib/route.js)
- `preParsing` runner: [lib/hooks.js](../../lib/hooks.js)
- Raw-body collection and limits: [lib/content-type-parser.js](../../lib/content-type-parser.js)

### 4.1 `preParsing` stream replacement contract

`preParsing` receives the current payload stream and can replace it by:

- calling `done(null, newStream)`, or
- `return newStream` from async hook

Fastify stores replacement stream on internal request symbol and passes it to
the parser path.

TS-first example:

```typescript
import Fastify from 'fastify'
import { Transform } from 'node:stream'

const app = Fastify()

app.addHook('preParsing', async (request, _reply, payload) => {
  const passthrough = new Transform({
    transform (chunk, _enc, callback) {
      const src = payload as NodeJS.ReadableStream & { receivedEncodedLength?: number }
      passthrough.receivedEncodedLength = src.receivedEncodedLength ?? 0
      callback(null, chunk)
    }
  }) as Transform & { receivedEncodedLength?: number }

  payload.pipe(passthrough)
  return passthrough
})
```

### 4.2 `receivedEncodedLength`, `content-length`, and `bodyLimit`

`rawBody` enforces both:

- decoded size limits, and
- encoded length checks (using `receivedEncodedLength` when present)

This is why docs and tests require maintaining `receivedEncodedLength` when you
replace payload stream in `preParsing`.

If encoded length tracking or content length mismatch is wrong, parsing fails
with content-length/body-limit errors.

### 4.3 Custom content-type parsers with streams

Content-type parsers consume stream data and produce `request.body`.

TS-first parser sketch:

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.addContentTypeParser('application/x-lines', (request, payload, done) => {
  let body = ''
  payload.setEncoding('utf8')
  payload.on('data', (chunk) => { body += chunk })
  payload.on('end', () => done(null, body.split('\n').filter(Boolean)))
  payload.on('error', done)
})

app.post('/lines', async (request) => {
  return { lines: request.body }
})
```

---

## 5. Error Paths and Stream Lifecycle

Stream-specific error behavior to rely on:

- If stream source errors before headers are committed, Fastify can still send
  an error response (`test/stream.1.test.js`, `test/stream.4.test.js`).
- If response has started, stream errors are logged and socket is destroyed
  (`lib/reply.js`).
- When destination closes first, Fastify attempts cleanup via
  `destroy()`, then `close()`, then `abort()`; otherwise it warns that payload
  did not end properly (`test/stream.2.test.js`, `test/stream.3.test.js`,
  `test/stream.4.test.js`).

Operational takeaway: always return streams with deterministic end/error
behavior and attach upstream error handling where appropriate.

---

## 6. HEAD, Abort, and Premature Close Behavior

### HEAD behavior

For HEAD routes, Fastify's head onSend handler consumes/cancels stream payloads
and sends no body (`lib/head-route.js`).

- Node stream payloads are resumed.
- WHATWG streams are cancelled.

This avoids leaking stream resources while preserving HEAD semantics.

### Abort and premature close

When clients disconnect early:

- Fastify avoids crashing and may log premature close depending on state.
- Stream cleanup method fallback order is applied (`destroy`/`close`/`abort`).
- Aborted request scenarios are covered in stream tests and should be treated as
  normal operational events.

---

## 7. TS-First End-to-End Example

```typescript
import Fastify, { FastifyInstance } from 'fastify'
import { createReadStream } from 'node:fs'
import { createGunzip } from 'node:zlib'

async function build(): Promise<FastifyInstance> {
  const app = Fastify()

  app.addHook('preParsing', async (_request, _reply, payload) => {
    const gunzip = createGunzip() as ReturnType<typeof createGunzip> & {
      receivedEncodedLength?: number
    }

    payload.on('data', (chunk: Buffer) => {
      gunzip.receivedEncodedLength = (gunzip.receivedEncodedLength ?? 0) + chunk.length
    })

    payload.pipe(gunzip)
    return gunzip
  })

  app.post('/echo-json', async (request) => {
    return request.body
  })

  app.get('/download', async (_request, reply) => {
    reply.type('application/octet-stream')
    return reply.send(createReadStream('./archive.tar'))
  })

  app.get('/proxy', async () => {
    const upstream = await fetch('https://example.com/stream')
    return new Response(upstream.body, {
      status: upstream.status,
      headers: upstream.headers
    })
  })

  return app
}
```

What this demonstrates:

- input stream transformation in `preParsing`
- output stream via Node readable
- output stream via WHATWG `Response`

---

## 8. JavaScript Equivalent Snippets

### Node stream response

```js
const Fastify = require('fastify')
const { createReadStream } = require('node:fs')

const app = Fastify()

app.get('/file', async (_request, reply) => {
  return reply.type('text/plain; charset=utf-8').send(createReadStream('./large-file.txt'))
})
```

### Web stream response

```js
const { Readable } = require('node:stream')
const { createReadStream } = require('node:fs')

app.get('/web-stream', async (_request, reply) => {
  return reply.send(Readable.toWeb(createReadStream('./large-file.txt')))
})
```

### `preParsing` replacement

```js
const { Transform } = require('node:stream')

app.addHook('preParsing', (request, reply, payload, done) => {
  const transformed = new Transform({
    transform (chunk, enc, cb) {
      transformed.receivedEncodedLength = (transformed.receivedEncodedLength || 0) + chunk.length
      cb(null, chunk)
    }
  })
  payload.pipe(transformed)
  done(null, transformed)
})
```

---

## 9. Common Pitfalls and Fast Fixes

1. **Locked web streams**
   - Symptom: 500 with `FST_ERR_REP_READABLE_STREAM_LOCKED`.
   - Fix: release reader lock before sending.

2. **Consumed `Response` body**
   - Symptom: 500 with `FST_ERR_REP_RESPONSE_BODY_CONSUMED`.
   - Fix: do not call `.text()`, `.json()`, etc. before `reply.send(response)`.

3. **Broken `preParsing` replacement stream**
   - Symptom: invalid content-length/body-limit failures.
   - Fix: update `receivedEncodedLength` correctly on transformed stream.

4. **Mixing stream flow with manual raw writes**
   - Symptom: warnings or truncated responses.
   - Fix: choose one response strategy (`reply.send(stream)` preferred).

5. **Assuming all docs text always matches runtime edge behavior**
   - Fix: verify critical stream assumptions against current tests listed below.

---

## 10. Source Grounding Map

Core internals:

- [lib/reply.js](../../lib/reply.js)
- [lib/route.js](../../lib/route.js)
- [lib/hooks.js](../../lib/hooks.js)
- [lib/content-type-parser.js](../../lib/content-type-parser.js)
- [lib/head-route.js](../../lib/head-route.js)
- [lib/symbols.js](../../lib/symbols.js)

Reference docs:

- [docs/Reference/Reply.md](../docs/Reference/Reply.md)
- [docs/Reference/Hooks.md](../docs/Reference/Hooks.md)
- [docs/Reference/ContentTypeParser.md](../docs/Reference/ContentTypeParser.md)

Examples:

- [examples/simple-stream.js](../examples/simple-stream.js)
- [examples/parser.js](../examples/parser.js)
- [examples/benchmark/webstream.js](../examples/benchmark/webstream.js)

Tests that validate stream behavior:

- [test/stream.1.test.js](../test/stream.1.test.js)
- [test/stream.2.test.js](../test/stream.2.test.js)
- [test/stream.3.test.js](../test/stream.3.test.js)
- [test/stream.4.test.js](../test/stream.4.test.js)
- [test/stream.5.test.js](../test/stream.5.test.js)
- [test/reply-web-stream-locked.test.js](../test/reply-web-stream-locked.test.js)
- [test/web-api.test.js](../test/web-api.test.js)

---

**Related deep dives:**

- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Lifecycle - Complete Tutorial](./fastify-lifecycle.tutorial.md)

End of tutorial.
