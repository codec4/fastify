# Fastify Reply API — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's `Reply` object from first principles — from sending a simple string to
> streaming, trailers, custom serialization, and hijacking.

---

## Table of Contents

1. [What is the Reply Object?](#1-what-is-the-reply-object)
2. [Reply Object Anatomy](#2-reply-object-anatomy)
3. [Where Reply Lives in the Lifecycle](#3-where-reply-lives-in-the-lifecycle)
4. [Status Codes — `code()` / `status()`](#4-status-codes--code--status)
5. [Headers](#5-headers)
   - 5.1 [Setting Headers — `header()` / `headers()`](#51-setting-headers--header--headers)
   - 5.2 [Reading Headers — `getHeader()` / `getHeaders()` / `hasHeader()`](#52-reading-headers--getheader--getheaders--hasheader)
   - 5.3 [Removing Headers — `removeHeader()`](#53-removing-headers--removeheader)
   - 5.4 [Content-Type Shorthand — `type()`](#54-content-type-shorthand--type)
6. [Sending Responses — `send()`](#6-sending-responses--send)
   - 6.1 [Payload Type Matrix](#61-payload-type-matrix)
   - 6.2 [Sending JSON](#62-sending-json)
   - 6.3 [Sending Strings](#63-sending-strings)
   - 6.4 [Sending Buffers / TypedArrays](#64-sending-buffers--typedarrays)
   - 6.5 [Sending Streams (Node.js)](#65-sending-streams-nodejs)
   - 6.6 [Sending Web Streams (WHATWG)](#66-sending-web-streams-whatwg)
   - 6.7 [Sending a Web `Response` Object](#67-sending-a-web-response-object)
   - 6.8 [Sending Errors](#68-sending-errors)
   - 6.9 [Sending Nothing (Empty Body)](#69-sending-nothing-empty-body)
7. [Redirects — `redirect()`](#7-redirects--redirect)
8. [Early Hints — `writeEarlyHints()`](#8-early-hints--writeearlyhints)
9. [Trailers](#9-trailers)
   - 9.1 [What are HTTP Trailers?](#91-what-are-http-trailers)
   - 9.2 [`trailer()` / `hasTrailer()` / `removeTrailer()`](#92-trailer--hastrailer--removetrailer)
10. [Serialization](#10-serialization)
    - 10.1 [Default Schema-Based Serialization](#101-default-schema-based-serialization)
    - 10.2 [Per-Reply Override — `serializer()`](#102-per-reply-override--serializer)
    - 10.3 [`serialize()` — Explicit Serialization](#103-serialize--explicit-serialization)
    - 10.4 [`serializeInput()` — Schema-Specific Serialization](#104-serializeinput--schema-specific-serialization)
    - 10.5 [`compileSerializationSchema()`](#105-compileserialization-schema)
    - 10.6 [`getSerializationFunction()`](#106-getserializationfunction)
11. [Hijacking the Response — `hijack()`](#11-hijacking-the-response--hijack)
12. [The `sent` Property](#12-the-sent-property)
13. [Elapsed Time — `elapsedTime`](#13-elapsed-time--elapsedtime)
14. [Calling the Not-Found Handler — `callNotFound()`](#14-calling-the-not-found-handler--callnotfound)
15. [Reply as a Promise — `then()`](#15-reply-as-a-promise--then)
16. [Accessing Decorators — `getDecorator()`](#16-accessing-decorators--getdecorator)
17. [Internal Reply Flow Diagram](#17-internal-reply-flow-diagram)
18. [TypeScript Quick-Reference](#18-typescript-quick-reference)
19. [Real-World Patterns](#19-real-world-patterns)
    - 19.1 [Fluent Response Builder](#191-fluent-response-builder)
    - 19.2 [Conditional JSON / Buffer Response](#192-conditional-json--buffer-response)
    - 19.3 [File Download Stream](#193-file-download-stream)
    - 19.4 [Server-Sent Events (SSE) via Hijack](#194-server-sent-events-sse-via-hijack)
    - 19.5 [HTTP `Trailer` with Checksums](#195-http-trailer-with-checksums)
20. [Common Pitfalls](#20-common-pitfalls)

Related deep dives:

- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Lifecycle — Complete Tutorial](./fastify-lifecycle.tutorial.md)
- [Fastify Error Handling — Complete Tutorial](./fastify-error-handling.tutorial.md)
- [Fastify Streams — Complete Tutorial](./fastify-streams.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)

---

## 1. What is the Reply Object?

Every route handler receives two arguments: `request` and `reply`.  The `reply` object
is Fastify's thin, high-performance wrapper around Node's raw `http.ServerResponse`
(or `http2.Http2ServerResponse`).

```
  Route handler(request, reply)
                               |
                               +-- reply.raw         -> Node http.ServerResponse
                               +-- reply.request     -> Fastify Request
                               +-- reply.log         -> pino logger
                               +-- reply.server      -> Fastify instance
                               +-- reply.code()      -> set status
                               +-- reply.header()    -> set header
                               +-- reply.send()      -> end the response
                               +-- reply.redirect()  -> 3xx shortcut
                               +-- reply.hijack()    -> take full control
```

`reply` is defined in [lib/reply.js](../../lib/reply.js) and instantiated once per request
in route dispatch.  It is intentionally **chainable** — most setter methods return
`this`.

---

## 2. Reply Object Anatomy

```
Reply {
  // Public properties
  raw          : http.ServerResponse   (the underlying Node response)
  request      : FastifyRequest        (sibling Fastify request)
  log          : Logger                (pino child logger for this request)
  server       : FastifyInstance       (the top-level Fastify server)
  sent         : boolean               (true once response headers+body flushed)
  statusCode   : number                (getter/setter — delegates to raw)
  routeOptions : RouteOptions          (read-only route config)
  elapsedTime  : number                (ms since onRequest, 0 before first hook)

  // Core methods
  code(statusCode)           -> Reply
  status(statusCode)         -> Reply   (alias for code)
  header(key, value)         -> Reply
  headers(obj)               -> Reply
  getHeader(key)             -> string | undefined
  getHeaders()               -> object
  hasHeader(key)             -> boolean
  removeHeader(key)          -> Reply
  type(contentType)          -> Reply
  send(payload?)             -> Reply
  redirect(url, code?)       -> Reply
  writeEarlyHints(hints, cb?)-> Reply
  hijack()                   -> Reply
  callNotFound()             -> Reply

  // Trailer methods
  trailer(key, fn)           -> Reply
  hasTrailer(key)            -> boolean
  removeTrailer(key)         -> Reply

  // Serialization methods
  serializer(fn)             -> Reply
  serialize(payload)         -> string
  serializeInput(input, schema, status?, contentType?)  -> string
  compileSerializationSchema(schema, httpStatus?, contentType?) -> Function
  getSerializationFunction(schemaOrStatus, contentType?) -> Function | undefined

  // Decorator access
  getDecorator(name)         -> any
}
```

---

## 3. Where Reply Lives in the Lifecycle

Reply is created at the start of route dispatch and travels through every lifecycle
stage as an argument to each hook and the route handler.

```
  Incoming HTTP request
        |
        v
  [onRequest hooks]        <-- reply exists, headers not set yet
        |
        v
  [preParsing hooks]       <-- reply exists, body not parsed
        |
        v
  [preValidation hooks]    <-- reply exists, body parsed
        |
        v
  [preHandler hooks]       <-- reply exists, validation passed
        |
        v
  [Route Handler]          <-- you call reply.send() here (usually)
        |
        v
  [preSerialization hooks] <-- reply.payload object, before JSON stringify
        |
        v
  [onSend hooks]           <-- reply.payload is now a string/Buffer/stream
        |
        v
  [response flushed]       <-- reply.sent === true
        |
        v
  [onResponse hooks]       <-- response completed, reply.elapsedTime available
```

At any stage calling `reply.send()` (or throwing) short-circuits to the error / send
path.  Calling `reply.hijack()` removes reply from Fastify's control entirely.

---

## 4. Status Codes — `code()` / `status()`

```js
reply.code(200)   // same as reply.status(200)
reply.statusCode  // getter: current code (default 200)
reply.statusCode = 201  // setter: delegates to reply.code()
```

Valid range: **100 – 599**.  Values outside this range throw
`FST_ERR_BAD_STATUS_CODE`.

```js
// Common pattern: set status then send
fastify.get('/created', async (request, reply) => {
  const item = await db.insert(request.body)
  return reply.code(201).send(item)
})
```

`code()` and `status()` are identical — `status` is an alias provided for Express
compatibility.

> **Tip:** Fastify only sets `kReplyHasStatusCode = true` after you explicitly call
> `code()` (or assign `statusCode`), even if `raw.statusCode` defaults to 200.  The
> redirect helper uses this flag to decide whether to default to 302.

---

## 5. Headers

### 5.1 Setting Headers — `header()` / `headers()`

```js
reply.header('content-type', 'application/json; charset=utf-8')

// Set multiple at once
reply.headers({
  'x-request-id': request.id,
  'cache-control': 'no-store',
  'x-powered-by':  'fastify'
})

// Chaining
reply
  .code(200)
  .header('x-custom', 'value')
  .send({ ok: true })
```

**`Set-Cookie` is special:** multiple calls to `header('set-cookie', ...)` accumulate
into an array rather than overwriting:

```js
reply.header('set-cookie', 'a=1; Path=/')
reply.header('set-cookie', 'b=2; Path=/')
// raw response will have two Set-Cookie lines
```

Headers are stored in an internal lowercase map (`reply[kReplyHeaders]`) and flushed
to `raw.writeHead()` immediately before the body is written.  This means you can
mutate headers in `onSend` hooks without fear of them already being written.

### 5.2 Reading Headers — `getHeader()` / `getHeaders()` / `hasHeader()`

```js
const ct = reply.getHeader('content-type')  // from Fastify map, falls back to raw

const all = reply.getHeaders()
// { ...raw.getHeaders(), ...fastifyHeaders }  (fastify headers win on conflict)

if (reply.hasHeader('x-rate-limit')) { ... }
```

### 5.3 Removing Headers — `removeHeader()`

```js
reply.removeHeader('x-powered-by')
```

Deletes from the internal Fastify header map.  Has no effect on headers already
written to the socket (i.e., after `sent === true`).

### 5.4 Content-Type Shorthand — `type()`

```js
reply.type('text/html; charset=utf-8').send('<h1>Hello</h1>')
reply.type('application/xml').send(xmlString)
```

`type()` is a convenience wrapper for `header('content-type', value)`.  Fastify will
**not** add its automatic content-type header when you have already set one.

---

## 6. Sending Responses — `send()`

`send()` is the primary method to end the response.  It accepts any of the payload
types described below, runs `preSerialization` and `onSend` hooks, then flushes
headers + body to the socket.

```js
// Signature
reply.send(payload?)
```

Calling `send()` after the reply is already sent logs a warning and returns early
(`FST_ERR_REP_ALREADY_SENT`).  Calling `send()` inside an `onError` hook throws
`FST_ERR_SEND_INSIDE_ONERR`.

### 6.1 Payload Type Matrix

```
Payload type               Auto content-type           Serialization path
-------------------        --------------------------  ----------------------
undefined / void           (none, content-length: 0)   none
null                       (none, content-length: 0)   none
string                     text/plain; charset=utf-8   none (verbatim)
Buffer / TypedArray        application/octet-stream    none (verbatim bytes)
Node.js Readable stream    (none set by Fastify)       pipe to socket
WHATWG ReadableStream      (none set by Fastify)       pull-read to socket
Web Response object        (from Response.headers)     body piped to socket
Error object               application/json            error serializer
plain object / array       application/json            JSON serializer (fast-json-stringify or custom)
```

### 6.2 Sending JSON

```js
fastify.get('/user/:id', async (request, reply) => {
  const user = await db.findUser(request.params.id)
  return reply.send(user)  // object -> JSON serializer -> application/json
})

// Or simply return from async handler (Fastify calls reply.send for you)
fastify.get('/user/:id', async (request, reply) => {
  return db.findUser(request.params.id)
})
```

When a `response` JSON Schema is declared on the route Fastify uses
`fast-json-stringify` to serialize — this is **significantly faster** than
`JSON.stringify` because the serializer is compiled once from the schema.

```js
fastify.get('/products', {
  schema: {
    response: {
      200: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            id:    { type: 'integer' },
            name:  { type: 'string' },
            price: { type: 'number' }
          }
        }
      }
    }
  }
}, async () => db.getProducts())
```

### 6.3 Sending Strings

```js
reply.type('text/plain').send('pong')

// Without type(), Fastify auto-sets text/plain; charset=utf-8 for strings
reply.send('Hello, world!')
```

> **Caution:** if you have a `response` schema defined and `content-type` is JSON (or
> not set), Fastify will still try to run the serializer.  If the payload is already a
> string and the serializer is set, the string is passed through as-is.

### 6.4 Sending Buffers / TypedArrays

```js
const buf = Buffer.from('binary data')
reply.send(buf)
// Auto-sets: content-type: application/octet-stream
// content-length is set to buf.byteLength

// TypedArray (e.g., Uint8Array)
const arr = new Uint8Array([0x89, 0x50, 0x4e, 0x47])
reply.type('image/png').send(arr)
```

Internally, TypedArrays are converted to `Buffer` via
`Buffer.from(payload.buffer, payload.byteOffset, payload.byteLength)`.

### 6.5 Sending Streams (Node.js)

```js
const fs = require('node:fs')

fastify.get('/download/:file', (request, reply) => {
  const stream = fs.createReadStream(`/files/${request.params.file}`)
  reply
    .header('content-disposition', 'attachment; filename="file"')
    .send(stream)
})
```

Fastify pipes the stream directly to `raw` using `stream.pipe(res)`.  It also
attaches `stream.finished` listeners to handle premature closes and errors:

- If `res` closes before the stream ends (client disconnect) → `stream.destroy()`
- If the stream errors before headers are sent → `onError` hook + error handler
- If the stream errors after headers are sent → logs a warning, `res.destroy()`

**Content-type is NOT set automatically for streams.** Always set it explicitly.

### 6.6 Sending Web Streams (WHATWG)

Node.js 18+ exposes WHATWG `ReadableStream` (e.g., from `fetch()`).  Fastify
supports it natively:

```js
fastify.get('/proxy', async (request, reply) => {
  const upstream = await fetch('https://example.com/api/data')
  // upstream.body is a ReadableStream
  reply
    .code(upstream.status)
    .type('application/json')
    .send(upstream.body)
})
```

Fastify detects the WHATWG stream by checking `typeof payload.getReader === 'function'`
and then pulls chunks via `reader.read()`, writing each chunk to `res` with proper
back-pressure handling.

### 6.7 Sending a Web `Response` Object

You can pass a fully-formed WHATWG `Response` object — Fastify extracts status,
headers and body automatically:

```js
fastify.get('/fetch-proxy', async (request, reply) => {
  const webResponse = await fetch('https://api.example.com/resource')
  return reply.send(webResponse)
})
```

Fastify:
1. Copies `webResponse.status` → status code
2. Iterates `webResponse.headers` → sets reply headers
3. Pipes `webResponse.body` (ReadableStream) → socket

> **Warning:** `webResponse.body` must not be consumed (i.e., `bodyUsed === false`)
> before passing to `send()`, otherwise Fastify throws `FST_ERR_REP_RESPONSE_BODY_CONSUMED`.

### 6.8 Sending Errors

```js
fastify.get('/protected', async (request, reply) => {
  if (!request.user) {
    return reply.send(new Error('Unauthorized'))
    // or simply: throw new Error('Unauthorized')
  }
})
```

When `send()` receives an `Error` instance, Fastify routes through the `onError`
hooks then the registered error handler (`setErrorHandler`).  The default error
handler shapes the error as:

```json
{
  "statusCode": 500,
  "error": "Internal Server Error",
  "message": "Unauthorized"
}
```

Use `@fastify/error` or `http-errors`-compatible objects to set a specific status
code.  See the [Error Handling tutorial](./fastify-error-handling.tutorial.md) for
the full treatment.

### 6.9 Sending Nothing (Empty Body)

```js
reply.code(204).send()     // 204 No Content
reply.code(204).send(null) // same — null is treated as no body

reply.code(304).send()     // 304 Not Modified (no content-length either)
```

For status codes 204, 304, and 1xx, Fastify strips `content-type` and
`content-length` headers automatically (per RFC 9110 §8.6).

---

## 7. Redirects — `redirect()`

```js
reply.redirect('/new-path')           // 302 (or existing status if .code() was called)
reply.redirect('/permanent', 301)     // 301 Moved Permanently
reply.redirect('https://example.com') // absolute URL, 302
reply.code(307).redirect('/api/v2')   // 307 Temporary Redirect
```

Internally `redirect` does:

```
reply.header('location', url).code(code).send()
```

The default code is **302**, unless you previously called `reply.code()` in which
case that code is used.  Explicitly passing a second argument always wins.

---

## 8. Early Hints — `writeEarlyHints()`

HTTP 103 Early Hints let the browser start loading sub-resources before the main
response arrives.

```js
fastify.get('/page', async (request, reply) => {
  // Signal the browser to preload critical assets
  reply.writeEarlyHints({
    Link: [
      '</styles/main.css>; rel=preload; as=style',
      '</scripts/app.js>;  rel=preload; as=script'
    ]
  })

  const html = await renderPage()
  return reply.type('text/html').send(html)
})
```

`writeEarlyHints()` delegates directly to `raw.writeEarlyHints()` (Node.js v18.11+)
and returns `this` for chaining.  The  103 response is sent immediately; the route
handler continues building the real response.

---

## 9. Trailers

### 9.1 What are HTTP Trailers?

HTTP trailers are header-like fields sent **after** the response body, supported in
HTTP/1.1 chunked transfers and HTTP/2.  They are useful for checksums or hashes that
can only be computed once the full body is known.

```
  HTTP/1.1 200 OK
  Transfer-Encoding: chunked
  Trailer: X-Checksum

  <chunked body data...>

  X-Checksum: abc123
  (end of chunked body)
```

### 9.2 `trailer()` / `hasTrailer()` / `removeTrailer()`

```js
const crypto = require('node:crypto')
const { Readable } = require('node:stream')

fastify.get('/checksummed-stream', (request, reply) => {
  const hash = crypto.createHash('sha256')
  const stream = Readable.from(generateData())

  stream.on('data', chunk => hash.update(chunk))

  reply
    .trailer('x-checksum', (reply, payload, done) => {
      done(null, hash.digest('hex'))
    })
    .send(stream)
})
```

The trailer callback receives `(reply, payload, done)` and can also return a Promise:

```js
reply.trailer('x-etag', async (reply, payload) => {
  return computeEtag(payload)
})
```

**Forbidden trailer names** (Fastify throws `FST_ERR_BAD_TRAILER_NAME` for these):

```
transfer-encoding  content-length  host          cache-control
max-forwards       te              authorization  set-cookie
content-encoding   content-type    content-range  trailer
```

Trailer utilities:

```js
reply.hasTrailer('x-checksum')   // -> boolean
reply.removeTrailer('x-checksum')
```

Fastify automatically adds the `Trailer` and `Transfer-Encoding: chunked` headers
when trailers are present.

---

## 10. Serialization

### 10.1 Default Schema-Based Serialization

When a route defines a `response` schema, Fastify compiles a serializer at
registration time using `fast-json-stringify`.  This serializer is keyed by HTTP
status code and optional content-type.

```
  route.schema.response = {
    200:  { type: 'object', properties: { id: { type: 'integer' } } },
    404:  { type: 'object', properties: { message: { type: 'string' } } }
  }

  reply.send({ id: 42 })
  -> looks up serializer for statusCode=200
  -> calls fast-json-stringify compiled function
  -> sets content-type: application/json; charset=utf-8
  -> sends '{"id":42}'
```

### 10.2 Per-Reply Override — `serializer()`

Override the serializer for a single reply:

```js
fastify.get('/custom', (request, reply) => {
  reply
    .serializer(payload => JSON.stringify(payload, null, 2))  // pretty-print
    .send({ name: 'Alice' })
})
```

The function receives the raw payload object and must return a `string`.  Once set,
it bypasses schema-based serialization entirely.

### 10.3 `serialize()` — Explicit Serialization

Serialize a payload without sending, using the same logic as `send()`:

```js
const json = reply.serialize({ id: 1, name: 'Bob' })
// Returns a string — useful for caching or logging
```

Resolution order:
1. Per-reply `[kReplySerializer]` if set via `reply.serializer(fn)`
2. Route-level default serializer (`kReplySerializerDefault`)
3. Schema-based serializer for the current status code

### 10.4 `serializeInput()` — Schema-Specific Serialization

Serialize data against an explicit status code or schema without sending:

```js
// Serialize against the 200 response schema
const str = reply.serializeInput({ id: 42 }, 200)

// Serialize against a specific content-type variant
const str2 = reply.serializeInput(data, 200, 'application/json')

// Serialize against an inline schema object
const str3 = reply.serializeInput(data, {
  type: 'object',
  properties: { id: { type: 'integer' } }
})
```

Throws `FST_ERR_MISSING_SERIALIZATION_FN` if no serializer is registered for the
given status / content-type.

### 10.5 `compileSerializationSchema()`

Compile a serializer for an inline schema on demand and cache it on the route context:

```js
const serialize = reply.compileSerializationSchema(
  { type: 'object', properties: { count: { type: 'number' } } },
  200,
  'application/json'
)

const result = serialize({ count: 7 })
```

The compiled function is stored in a `WeakMap` on the route context, so subsequent
calls with the same schema object return the cached function.

### 10.6 `getSerializationFunction()`

Retrieve a pre-compiled serializer by status code, content-type, or schema object:

```js
// By status code (returns the schema-compiled fn)
const fn = reply.getSerializationFunction(200)
const fn2 = reply.getSerializationFunction(200, 'application/json')

// By inline schema (returns from the WeakMap cache)
const schema = { type: 'object', properties: { id: { type: 'integer' } } }
const fn3 = reply.getSerializationFunction(schema)
```

Returns `undefined` if no serializer exists for the given key.

---

## 11. Hijacking the Response — `hijack()`

`hijack()` tells Fastify to step aside and give you full, manual control over the
underlying `reply.raw` (`http.ServerResponse`):

```js
fastify.get('/sse', (request, reply) => {
  reply.hijack()                       // Fastify stops managing this response

  const res = reply.raw
  res.writeHead(200, {
    'content-type':  'text/event-stream',
    'cache-control': 'no-cache',
    'connection':    'keep-alive'
  })

  const interval = setInterval(() => {
    res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`)
  }, 1000)

  res.on('close', () => clearInterval(interval))
})
```

When hijacked:
- `reply.sent` returns `true` immediately.
- Fastify's `onResponse` hooks **do not fire**.
- The timeout timer and abort signal listeners are cleaned up.
- You are responsible for calling `res.end()`.

> **Use hijack sparingly** — prefer streaming payloads or the `onSend` hook for most
> use cases.  Hijack is primarily for long-lived connections (SSE, WebSocket upgrades
> done manually, raw HTTP/2 push).

---

## 12. The `sent` Property

```js
reply.sent   // boolean: true once response is flushed or hijacked

// Internally checks:
return (reply[kReplyHijacked] || reply.raw.writableEnded) === true
```

Useful for guard conditions in async code:

```js
fastify.get('/guarded', async (request, reply) => {
  doAsyncWork()
    .then(result => {
      if (!reply.sent) {            // guard against double-send
        reply.send(result)
      }
    })
    .catch(err => {
      if (!reply.sent) {
        reply.send(err)
      }
    })
  // note: prefer returning a Promise instead of this pattern
})
```

---

## 13. Elapsed Time — `elapsedTime`

```js
reply.elapsedTime   // number (milliseconds)
```

Measures time from when `onRequest` began until now (or until `onResponse` completed
if accessed there).  Returns `0` before `onRequest` runs.

```js
fastify.addHook('onResponse', (request, reply, done) => {
  request.log.info({ ms: reply.elapsedTime }, 'request completed')
  done()
})
```

Internally uses the same high-resolution clock (`now()`) as pino's request logging.

---

## 14. Calling the Not-Found Handler — `callNotFound()`

Delegate to the registered 404 handler from within a route handler:

```js
fastify.get('/resource/:id', async (request, reply) => {
  const item = await db.find(request.params.id)
  if (!item) {
    return reply.callNotFound()   // triggers setNotFoundHandler
  }
  return item
})
```

This is equivalent to route dispatch reaching a dead end, and respects the scoped
`setNotFoundHandler` of the current plugin context.

---

## 15. Reply as a Promise — `then()`

`Reply` implements the *thenable* interface, allowing it to be `await`-ed in async
handlers:

```js
async function handler (request, reply) {
  reply.send('ok')
  await reply          // resolves when raw response stream ends
  console.log('response fully flushed')
}
```

Internally, `then()` attaches a `stream.finished` listener to `reply.raw`.  If the
stream ends with an error (other than `ERR_STREAM_PREMATURE_CLOSE`), the promise
rejects.

This is mostly used internally by Fastify's async handler wrapper — direct usage in
application code is rare.

---

## 16. Accessing Decorators — `getDecorator()`

When you add decorators to the reply prototype, retrieve them type-safely with
`getDecorator()`:

```js
fastify.decorateReply('userContext', null)

fastify.addHook('preHandler', (request, reply, done) => {
  reply.userContext = { id: 42, role: 'admin' }
  done()
})

fastify.get('/me', (request, reply) => {
  const ctx = reply.getDecorator('userContext')
  reply.send(ctx)
})
```

Throws `FST_ERR_DEC_UNDECLARED` if the decorator was never registered.
Function decorators are returned **bound** to `reply`.

---

## 17. Internal Reply Flow Diagram

The diagram below traces a successful `reply.send(object)` call from handler to
socket.

```
  reply.send(payload)
        |
        | payload instanceof Error?
        +-- YES -> onErrorHook -> handleError -> [error handler] -> onSendHook
        |
        | payload is undefined?
        +-- YES -> onSendHook (empty body)
        |
        | payload is stream / web stream / Response?
        +-- YES -> onSendHook (stream path)
        |
        | payload is Buffer / TypedArray?
        +-- YES -> set content-type: octet-stream (if not set)
        |          onSendHook
        |
        | payload is string?
        +-- YES -> set content-type: text/plain (if not set)
        |          onSendHook
        |
        | custom serializer set (reply[kReplySerializer])?
        +-- YES -> serialize payload -> onSendHook
        |
        | content-type contains 'json' or not set?
        +-- YES -> set content-type: application/json
        |          preSerializationHook
        |            -> fast-json-stringify (schema) or JSON.stringify
        |            -> onSendHook
        |
        v
  onSendHook (preSerialization already ran above for objects)
        |
        v
  onSendEnd
        |
        +-- payload is null / undefined?
        |     -> write content-length: 0 header
        |        writeHead -> res.end()
        |
        +-- status 1xx or 204?
        |     -> strip content-type + content-length
        |        writeHead -> res.end() (drain stream if needed)
        |
        +-- payload is Node stream?
        |     -> setHeader loop -> payload.pipe(res)
        |
        +-- payload is WHATWG ReadableStream?
        |     -> setHeader loop -> pull-read loop -> res.write() -> res.end()
        |
        +-- payload is string or Buffer?
              -> compute content-length
                 writeHead -> res.write(payload) -> sendTrailer -> res.end()
```

---

## 18. TypeScript Quick-Reference

```ts
import Fastify, {
  FastifyRequest,
  FastifyReply,
  RouteHandlerMethod
} from 'fastify'

const fastify = Fastify()

// Typed route handler
const handler: RouteHandlerMethod = async (
  request: FastifyRequest,
  reply: FastifyReply
) => {
  return reply
    .code(200)
    .header('x-custom', 'value')
    .send({ ok: true })
}

// Typed decorator
declare module 'fastify' {
  interface FastifyReply {
    userContext: { id: number; role: string } | null
  }
}

fastify.decorateReply<{ id: number; role: string } | null>('userContext', null)

// Typed serialization
interface User { id: number; name: string }

fastify.get<{ Params: { id: string } }>('/user/:id', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          id:   { type: 'integer' },
          name: { type: 'string'  }
        }
      }
    }
  }
}, async (request, reply): Promise<User> => {
  return { id: Number(request.params.id), name: 'Alice' }
})
```

---

## 19. Real-World Patterns

### 19.1 Fluent Response Builder

Chain multiple setters before sending — especially clean for conditional headers:

```js
fastify.get('/document/:id', async (request, reply) => {
  const doc = await db.getDocument(request.params.id)

  if (!doc) return reply.callNotFound()

  reply.code(200)

  if (doc.etag) reply.header('etag', doc.etag)
  if (doc.lastModified) reply.header('last-modified', doc.lastModified.toUTCString())

  return reply
    .type(doc.mimeType)
    .header('cache-control', 'public, max-age=3600')
    .send(doc.content)
})
```

### 19.2 Conditional JSON / Buffer Response

```js
fastify.get('/export', async (request, reply) => {
  const format = request.query.format ?? 'json'
  const data = await db.exportAll()

  if (format === 'csv') {
    const csv = toCsv(data)
    return reply
      .type('text/csv')
      .header('content-disposition', 'attachment; filename="export.csv"')
      .send(Buffer.from(csv, 'utf-8'))
  }

  return reply.send(data)  // default: JSON
})
```

### 19.3 File Download Stream

```js
const fs = require('node:fs')
const path = require('node:path')

fastify.get('/files/:filename', (request, reply) => {
  const filename = path.basename(request.params.filename) // sanitize
  const filepath = path.join('/safe/upload/dir', filename)

  const stream = fs.createReadStream(filepath)

  stream.on('error', err => {
    if (err.code === 'ENOENT') {
      reply.callNotFound()
    } else {
      reply.send(err)
    }
  })

  reply
    .header('content-disposition', `attachment; filename="${filename}"`)
    .type('application/octet-stream')
    .send(stream)
})
```

### 19.4 Server-Sent Events (SSE) via Hijack

```js
fastify.get('/events', (request, reply) => {
  reply.hijack()

  const res = reply.raw
  res.writeHead(200, {
    'content-type':  'text/event-stream; charset=utf-8',
    'cache-control': 'no-cache, no-transform',
    'connection':    'keep-alive',
    'x-accel-buffering': 'no'   // disable nginx buffering
  })
  res.write('retry: 5000\n\n')  // tell client to reconnect after 5s

  // Subscribe to application events
  const unsubscribe = eventBus.on('update', data => {
    res.write(`id: ${data.id}\n`)
    res.write(`event: update\n`)
    res.write(`data: ${JSON.stringify(data)}\n\n`)
  })

  // Cleanup when client disconnects
  res.on('close', unsubscribe)
})
```

### 19.5 HTTP `Trailer` with Checksums

```js
const crypto = require('node:crypto')

fastify.get('/integrity-stream', (request, reply) => {
  const hash = crypto.createHash('sha256')
  const chunks = ['Hello, ', 'World!']
  const stream = new require('node:stream').Readable({
    read () {
      const chunk = chunks.shift()
      if (chunk) {
        hash.update(chunk)
        this.push(chunk)
      } else {
        this.push(null)
      }
    }
  })

  reply
    .type('text/plain')
    .trailer('x-content-sha256', async () => hash.digest('hex'))
    .send(stream)
})
```

---

## 20. Common Pitfalls

| Pitfall | Why it happens | Fix |
|---|---|---|
| `FST_ERR_REP_ALREADY_SENT` | Calling `reply.send()` twice, e.g. in a hook and a handler | Use `if (!reply.sent)` guard or always `return reply.send()` |
| `FST_ERR_SEND_INSIDE_ONERR` | Calling `reply.send()` inside an `onError` hook | Use `setErrorHandler` to shape the error response instead |
| Missing `content-type` on stream | Fastify does not auto-set it for streams | Always call `reply.type(...)` before streaming |
| Headers visible after `sent` | Setting headers after `reply.send()` completes | Set all headers before `send()` or in `onSend` hook |
| `bodyUsed` error with `Response` | `fetch()` response body already consumed | Never read `.json()` / `.text()` before passing to `reply.send()` |
| Trailers on HTTP/1.1 plain responses | Trailers only work with chunked transfer encoding | Fastify auto-adds `Transfer-Encoding: chunked` — ensure no `content-length` override |
| `getDecorator` throws | Decorator not registered before use | Register with `fastify.decorateReply()` at plugin load time |
| Losing `this` in decorator function | Function decorator not bound | `getDecorator()` auto-binds function decorators to `reply` |
| Double serialization | Returning a string from a handler when a JSON schema is set | Return objects (not pre-stringified strings) when using response schemas |
| `redirect` sends wrong code | Calling `reply.code(301)` then `reply.redirect(url)` without code arg | The second argument to `redirect(url, code)` always wins; omit `reply.code()` or pass code to `redirect()` |
