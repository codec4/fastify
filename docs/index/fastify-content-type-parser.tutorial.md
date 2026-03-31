# Fastify Content-Type Parser — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's content-type parsing system from first principles.

---

## Table of Contents

1. [What is a Content-Type Parser?](#1-what-is-a-content-type-parser)
2. [Built-in Parsers](#2-built-in-parsers)
3. [Where Parsing Fits in the Lifecycle](#3-where-parsing-fits-in-the-lifecycle)
4. [addContentTypeParser — Core API](#4-addcontenttypeparser--core-api)
   - 4.1 [Callback style](#41-callback-style)
   - 4.2 [Async / Promise style](#42-async--promise-style)
   - 4.3 [Registering multiple types at once](#43-registering-multiple-types-at-once)
   - 4.4 [RegExp matching](#44-regexp-matching)
   - 4.5 [Wildcard catch-all (`*`)](#45-wildcard-catch-all-)
5. [parseAs Option — String, Buffer, or Stream](#5-parseas-option--string-buffer-or-stream)
   - 5.1 [Stream mode (default)](#51-stream-mode-default)
   - 5.2 [String mode](#52-string-mode)
   - 5.3 [Buffer mode](#53-buffer-mode)
6. [bodyLimit — Three Levels of Control](#6-bodylimit--three-levels-of-control)
7. [Overriding Built-in Parsers](#7-overriding-built-in-parsers)
   - 7.1 [Replacing the JSON parser](#71-replacing-the-json-parser)
   - 7.2 [Reusing the default JSON parser for extra types](#72-reusing-the-default-json-parser-for-extra-types)
   - 7.3 [Replacing the plain-text parser](#73-replacing-the-plain-text-parser)
8. [Proto-Poisoning and Constructor-Poisoning Protection](#8-proto-poisoning-and-constructor-poisoning-protection)
9. [hasContentTypeParser, removeContentTypeParser, removeAllContentTypeParsers](#9-hascontenttypeparser-removecontenttypeparser-removeallcontenttypeparsers)
10. [Scoped Parsers and Encapsulation](#10-scoped-parsers-and-encapsulation)
11. [Error Handling](#11-error-handling)
12. [Under the Hood — Internals](#12-under-the-hood--internals)
    - 12.1 [ContentTypeParser and Parser classes](#121-contenttypeparser-and-parser-classes)
    - 12.2 [Lookup algorithm and LRU cache](#122-lookup-algorithm-and-lru-cache)
    - 12.3 [rawBody and the bodyLimit guard](#123-rawbody-and-the-bodylimit-guard)
    - 12.4 [AsyncResource integration](#124-asyncresource-integration)
13. [TypeScript Quick-Reference](#13-typescript-quick-reference)
14. [Error Code Reference](#14-error-code-reference)

Related deep dives:

- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Lifecycle — Complete Tutorial](./fastify-lifecycle.tutorial.md)
- [Fastify Streams — Complete Tutorial](./fastify-streams.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)

---

## 1. What is a Content-Type Parser?

When an HTTP request carries a body, the raw data arrives as a **Node.js `Readable`
stream**.  Fastify needs to turn that stream into a JavaScript value before your route
handler can use it.  The component responsible for that transformation is the
**content-type parser**.

Fastify selects the right parser by matching the `Content-Type` request header value
against a registry of registered parsers.  Once selected, the parser reads the stream,
produces a value, and assigns it to `request.body`.

```
  POST /items  Content-Type: application/json
  Body: {"price":42}
         |
         v
  ContentTypeParser.getParser('application/json')
         |
         v
  defaultJsonParser(req, body, done)
         |
         v
  request.body = { price: 42 }
```

Two fundamental properties make the system powerful:

- **Any MIME type** can be handled — built-in, vendor, custom, or regex-matched.
- **Scope-awareness** — parsers obey Fastify's plugin encapsulation boundaries, so
  different plugins can register the same content-type with different behaviours.

---

## 2. Built-in Parsers

Fastify registers two parsers automatically on every new instance:

| Content-Type         | `parseAs` | Behaviour |
|----------------------|-----------|-----------|
| `application/json`   | `string`  | Parses with `secure-json-parse` (proto-poisoning protection enabled) |
| `text/plain`         | `string`  | Returns the body string as-is |

Every other MIME type is **unhandled by default** and returns a `415 Unsupported Media
Type` error (`FST_ERR_CTP_INVALID_MEDIA_TYPE`).

The built-in parsers can be **overridden** (see §7) or **removed** (see §9).

---

## 3. Where Parsing Fits in the Lifecycle

Parsing happens between the `preParsing` and `preValidation` hooks.

```
  Incoming HTTP Request
        |
        v
  +-------------+
  |  onRequest  |
  +------+------+
         |
         v
  +---------------+
  |  preParsing   |  <- optional: replace the raw payload stream
  +------+--------+
         |
         v
  +------------------------------------+
  |  Content-Type Parser runs here     |  <- THIS tutorial
  |  request.body is populated         |
  +------------------+-----------------+
                     |
                     v
  +----------------------+
  |   preValidation      |  <- request.body is now available
  +----------+-----------+
             |  JSON-Schema validation
             v
  +-------------+
  |  preHandler |
  +------+------+
         |
         v
  +----------------------+
  |    Route Handler     |  req.body is the parsed value
  +----------------------+
```

`preParsing` hooks run **before** the content-type parser and can replace the raw
payload stream entirely.  If you use `preParsing` to decompress or decrypt—say, with
`zlib.createGunzip()`—the content-type parser operates on the transformed stream.  See
[Fastify Streams — Complete Tutorial](./fastify-streams.tutorial.md) for details.

---

## 4. addContentTypeParser — Core API

### 4.1 Callback style

The most flexible parser signature.  The third argument, `done`, must be called exactly
once with either an error or the parsed value.

```js
// JavaScript — callback
fastify.addContentTypeParser(
  'application/x-msgpack',
  function (request, payload, done) {
    const chunks = []
    payload.on('data', chunk => chunks.push(chunk))
    payload.on('end', () => {
      try {
        const body = msgpack.decode(Buffer.concat(chunks))
        done(null, body)
      } catch (err) {
        err.statusCode = 400
        done(err)
      }
    })
    payload.on('error', done)
  }
)
```

When `parseAs` is **not** set (stream mode), `payload` is the raw readable stream.
When `parseAs: 'string'` or `parseAs: 'buffer'` is set, `payload` is a pre-buffered
string or `Buffer` (see §5).

### 4.2 Async / Promise style

Return a promise instead of calling `done`.  Rejected promises are automatically
converted to `400 Bad Request` replies.

```js
// JavaScript — async
fastify.addContentTypeParser(
  'application/x-ndjson',
  { parseAs: 'string' },
  async function (request, body) {
    return body.split('\n').filter(Boolean).map(JSON.parse)
  }
)
```

```js
// JavaScript — promise with stream
fastify.addContentTypeParser('application/x-msgpack', async function (request, payload) {
  const chunks = []
  for await (const chunk of payload) {
    chunks.push(chunk)
  }
  return msgpack.decode(Buffer.concat(chunks))
})
```

### 4.3 Registering multiple types at once

Pass an array of strings to register the same parser for several MIME types in a single
call.

```js
fastify.addContentTypeParser(
  ['application/json', 'application/json; charset=utf-8', 'text/json'],
  { parseAs: 'string' },
  async function (request, body) {
    return JSON.parse(body)
  }
)
```

Internally Fastify iterates the array and calls `.add()` once per entry.

### 4.4 RegExp matching

When an exact-string match is too narrow, use a `RegExp` to match a family of MIME
types.

```js
// Match any vendor JSON type: application/vnd.company+json, application/hal+json, …
fastify.addContentTypeParser(/^application\/.*\+json$/, function (request, payload, done) {
  let data = ''
  payload.setEncoding('utf8')
  payload.on('data', chunk => { data += chunk })
  payload.on('end', () => {
    try { done(null, JSON.parse(data)) } catch (e) { done(e) }
  })
})
```

**RegExp safety warning:** Fastify emits a `FSTSEC001` warning (via
`process.emitWarning`) when a registered RegExp does not start with `^` or does not
include `;?`.  This is because a too-broad pattern like `/json/` can accidentally
match MIME parameters such as `application/json; charset=utf-8`, leading to unexpected
parser selection.  Anchor your RegExps with `^` or include `;?` to silence the warning
and make matching intent explicit.

```js
// Good — anchored
fastify.addContentTypeParser(/^application\/.*\+json$/, parser)

// Also acceptable — handles optional params
fastify.addContentTypeParser(/application\/json;?.*/, parser)

// Bad — triggers FSTSEC001 warning
fastify.addContentTypeParser(/json/, parser)
```

### 4.5 Wildcard catch-all (`*`)

Registering `'*'` installs a **fallback** parser that runs whenever no other parser
matches. This is the escape hatch for accepting any content type.

```js
fastify.addContentTypeParser('*', function (request, payload, done) {
  // Drain and discard; expose raw stream length only
  let bytes = 0
  payload.on('data', chunk => { bytes += chunk.length })
  payload.on('end', () => done(null, { bytes }))
})
```

The wildcard is stored in the parser map under the empty string key `''`.  Named and
regexp parsers are always attempted first; the wildcard is a last resort.

---

## 5. parseAs Option — String, Buffer, or Stream

### 5.1 Stream mode (default)

When you omit `parseAs`, Fastify calls your parser with the **raw
`IncomingMessage`-compatible stream** as the second argument.  You are responsible for
consuming it completely — if you do not, the parser will hang waiting for the stream to
close.

```js
fastify.addContentTypeParser('application/octet-stream', function (req, payload, done) {
  const chunks = []
  payload.on('data', c => chunks.push(c))
  payload.on('end', () => done(null, Buffer.concat(chunks)))
  payload.on('error', done)
})
```

Use stream mode when you want to process data in chunks, or pipe the stream through a
transform (decrypt, decompress, etc.) before buffering.

### 5.2 String mode

`{ parseAs: 'string' }` tells Fastify to buffer the entire body and convert it to a UTF-8
string **before** calling your parser function.  The body limit check and
`Content-Length` validation are both applied during buffering.

```js
fastify.addContentTypeParser(
  'application/x-www-form-urlencoded',
  { parseAs: 'string' },
  function (req, body, done) {
    // body is already a decoded UTF-8 string
    done(null, Object.fromEntries(new URLSearchParams(body)))
  }
)
```

### 5.3 Buffer mode

`{ parseAs: 'buffer' }` behaves like string mode but delivers a raw `Buffer` instead
of a string.  Useful for binary formats such as Protobuf or MessagePack.

```js
fastify.addContentTypeParser(
  'application/protobuf',
  { parseAs: 'buffer' },
  function (req, body, done) {
    // body is a Buffer
    try {
      done(null, MyProto.decode(body))
    } catch (err) {
      err.statusCode = 400
      done(err)
    }
  }
)
```

**Comparison table:**

| Mode        | `parseAs` value | Second argument type | Who buffers? | Body limit enforced by |
|-------------|-----------------|----------------------|--------------|------------------------|
| Stream      | *(omitted)*     | `Readable`           | Your code    | Your code              |
| String      | `'string'`      | `string`             | Fastify      | Fastify (`rawBody`)    |
| Buffer      | `'buffer'`      | `Buffer`             | Fastify      | Fastify (`rawBody`)    |

---

## 6. bodyLimit — Three Levels of Control

Fastify enforces a hard maximum on request body size to protect against runaway memory
usage.  The limit can be set at three different granularities, with the **most specific
wins** rule:

```
  +--------------------------------------------------+
  |  Level 1 -- Server default (lowest precedence)  |
  |  Fastify({ bodyLimit: 1_048_576 })  // 1 MiB    |
  +-------------------------+------------------------+
                            |  overridden by
  +-------------------------v------------------------+
  |  Level 2 -- Per-parser limit                    |
  |  addContentTypeParser('x/foo', { bodyLimit: 512 })|
  +-------------------------+------------------------+
                            |  overridden by
  +-------------------------v------------------------+
  |  Level 3 -- Per-route limit (highest precedence)|
  |  fastify.post('/', { bodyLimit: 100 }, handler) |
  +--------------------------------------------------+
```

```js
const fastify = Fastify({ bodyLimit: 1_048_576 }) // 1 MiB server default

// This parser accepts up to 10 MiB regardless of the server default
fastify.addContentTypeParser(
  'video/mp4',
  { parseAs: 'buffer', bodyLimit: 10_485_760 },
  async (req, body) => body
)

// But this route squeezes it back to 512 bytes
fastify.post('/thumbnail', { bodyLimit: 512 }, async (req) => req.body)
```

When using **stream mode** (no `parseAs`), Fastify does **not** buffer the body and
therefore cannot enforce the per-parser or server bodyLimit automatically.  You must
enforce the limit yourself.

```js
fastify.addContentTypeParser('application/octet-stream', function (req, payload, done) {
  const limit = req.routeOptions.config?.bodyLimit ?? 1_048_576
  let received = 0
  const chunks = []
  payload.on('data', chunk => {
    received += chunk.length
    if (received > limit) {
      payload.destroy(new Error('Payload Too Large'))
      return
    }
    chunks.push(chunk)
  })
  payload.on('end', () => done(null, Buffer.concat(chunks)))
  payload.on('error', done)
})
```

---

## 7. Overriding Built-in Parsers

### 7.1 Replacing the JSON parser

The built-in `application/json` parser can be replaced **before** the server has
started.  This is the standard approach for adding custom deserialization logic (e.g. a
faster JSON parser, EJSON support, or stricter validation).

```js
const fastify = Fastify()

fastify.addContentTypeParser(
  'application/json',
  { parseAs: 'string' },
  function (req, body, done) {
    try {
      // Using native JSON.parse — no proto-poisoning protection!
      // Add your own guards here if needed.
      done(null, JSON.parse(body))
    } catch (err) {
      err.statusCode = 400
      done(err)
    }
  }
)
```

Fastify detects the override by comparing the registered function against the internal
`defaultJsonParser` reference.  As long as you supply a different function, the override
is accepted.

### 7.2 Reusing the default JSON parser for extra types

`fastify.getDefaultJsonParser(onProtoPoisoning, onConstructorPoisoning)` returns the
same `defaultJsonParser` function that backs `application/json`.  Use it to alias
additional MIME types to the existing secure parser without duplicating logic.

```js
fastify.addContentTypeParser(
  ['text/json', 'application/json-patch+json'],
  { parseAs: 'string' },
  fastify.getDefaultJsonParser('ignore', 'ignore')
)
```

The two parameters control what happens when the parsed object has poisoned prototype or
constructor keys:

| Value     | Behaviour |
|-----------|-----------|
| `'error'` | Throws — request fails with 400 |
| `'remove'` | Silently deletes the offending key |
| `'ignore'` | Keeps the key as-is (use with caution) |

### 7.3 Replacing the plain-text parser

The built-in `text/plain` parser is almost a no-op (it returns the string unchanged).
Override it to add trimming, length validation, or encoding normalisation.

```js
fastify.addContentTypeParser(
  'text/plain',
  { parseAs: 'string' },
  async function (req, body) {
    const trimmed = body.trim()
    if (trimmed.length === 0) throw Object.assign(new Error('Empty body'), { statusCode: 400 })
    return trimmed
  }
)
```

---

## 8. Proto-Poisoning and Constructor-Poisoning Protection

JavaScript prototype pollution is a security vulnerability where an attacker sends a
JSON payload containing `__proto__` or `constructor` keys to corrupt the prototype chain
of the parsed object, potentially allowing privilege escalation.

Fastify's default JSON parser uses
[`secure-json-parse`](https://github.com/nicolo-ribaudo/secure-json-parse) which
intercepts these keys before they can cause harm.  The behaviour is configured at the
server level:

```js
const fastify = Fastify({
  // What to do when a key like __proto__ is found
  onProtoPoisoning: 'error',      // 'error' | 'remove' | 'ignore'
  // What to do when a key like constructor is found
  onConstructorPoisoning: 'error' // 'error' | 'remove' | 'ignore'
})
```

The default for both is `'error'`, meaning a request containing those keys receives a
`400 Bad Request`.

**If you replace the JSON parser**, you take ownership of this protection.  Calling
`fastify.getDefaultJsonParser(proto, ctor)` with the appropriate actions lets you reuse
the built-in protection while customising the behaviour.

```js
// Custom parser that still uses secure-json-parse under the hood
fastify.addContentTypeParser(
  'application/json',
  { parseAs: 'string' },
  fastify.getDefaultJsonParser('remove', 'remove')
)
```

---

## 9. hasContentTypeParser, removeContentTypeParser, removeAllContentTypeParsers

These management APIs let you inspect and clean up the parser registry.  Like
`addContentTypeParser`, they all throw `FST_ERR_CTP_INSTANCE_ALREADY_STARTED` if called
after `fastify.listen()` has resolved.

### hasContentTypeParser

Returns `true` if a parser is registered for the given string or RegExp.

```js
console.log(fastify.hasContentTypeParser('application/json')) // true
console.log(fastify.hasContentTypeParser('application/xml'))  // false
console.log(fastify.hasContentTypeParser(/\+json$/))          // depends on registered parsers
```

### removeContentTypeParser

Removes one or more parsers from the registry.

```js
// Remove one
fastify.removeContentTypeParser('text/plain')

// Remove multiple at once
fastify.removeContentTypeParser(['application/json', 'text/plain'])
```

**Common pattern — replace a built-in without triggering the "already present"
guard:**

```js
// Remove then re-add to completely replace a built-in
fastify.removeContentTypeParser('application/json')
fastify.addContentTypeParser('application/json', { parseAs: 'buffer' }, myParser)
```

### removeAllContentTypeParsers

Wipes the entire registry — including both built-in parsers — and resets it to empty.
Typically combined with `addContentTypeParser('*', ...)` or selective re-registration to
create a fully custom parsing tier.

```js
fastify.removeAllContentTypeParsers()

// Now nothing is registered; add exactly what you need
fastify.addContentTypeParser('application/json', { parseAs: 'string' }, myJsonParser)
fastify.addContentTypeParser('*', { parseAs: 'buffer' }, myFallbackParser)
```

---

## 10. Scoped Parsers and Encapsulation

Content-type parsers follow the same encapsulation rules as hooks and decorators: they
**inherit downward** but never leak upward or sideways.

```
root
 +-- [application/json, text/plain]  (built-in)
 |
 +-- plugin A (registers application/xml)
 |    +-- route /a/items  -- has [json, plain, xml]  [ok]
 |    +-- route /a/data   -- has [json, plain, xml]  [ok]
 |
 +-- plugin B (does NOT register application/xml)
      +-- route /b/items  -- has [json, plain]  [no] xml not visible
```

Practical example using `fastify.register`:

```js
import Fastify from 'fastify'
import fp from 'fastify-plugin'

const app = Fastify()

// This plugin is encapsulated — xmlParser only visible inside
app.register(async function xmlPlugin (instance) {
  instance.addContentTypeParser(
    'application/xml',
    { parseAs: 'string' },
    async (req, body) => parseXml(body)
  )

  instance.post('/xml-endpoint', async (req) => {
    // req.body is the parsed XML object
    return req.body
  })
})

// This sibling plugin cannot see application/xml
app.register(async function jsonOnlyPlugin (instance) {
  instance.post('/json-endpoint', async (req) => {
    // Sending application/xml here → 415 Unsupported Media Type
    return req.body
  })
})

await app.listen({ port: 3000 })
```

To make a parser available **globally** (across all plugins and the root scope), either:
- Register it on the root Fastify instance before calling `register`, or
- Wrap the plugin with `fastify-plugin` (which breaks encapsulation intentionally).

```js
// Option A — root-level registration (simplest)
app.addContentTypeParser('application/xml', { parseAs: 'string' }, xmlParser)

// Option B — fastify-plugin
app.register(fp(async function (instance) {
  instance.addContentTypeParser('application/xml', { parseAs: 'string' }, xmlParser)
}))
```

---

## 11. Error Handling

### Parser errors

Call `done(err)` or throw / return a rejected promise.  Fastify will:

1. Set the `connection: close` header (the body stream may still have unread data).
2. Delegate to the error handler with `reply.send(err)`.

Always set `err.statusCode` to communicate the correct HTTP status code.

```js
fastify.addContentTypeParser('application/json', { parseAs: 'string' }, function (req, body, done) {
  try {
    done(null, JSON.parse(body))
  } catch (err) {
    err.statusCode = 400
    done(err)
  }
})
```

### Body-too-large

When Fastify buffers the body (string/buffer mode) and the limit is exceeded, it stops
buffering and calls `done` with `FST_ERR_CTP_BODY_TOO_LARGE` (HTTP 413).

### Unsupported media type

When no parser matches the incoming `Content-Type`, Fastify replies with
`FST_ERR_CTP_INVALID_MEDIA_TYPE` (HTTP 415).  Prevent this globally with a wildcard
parser or per-route by accepting the content type you need.

### Content-Length mismatch

In string/buffer mode, Fastify compares `receivedLength` against the
`Content-Length` request header after fully consuming the stream.  A mismatch (caused by
a corrupted or lying client) produces `FST_ERR_CTP_INVALID_CONTENT_LENGTH` (HTTP 400).

---

## 12. Under the Hood — Internals

### 12.1 ContentTypeParser and Parser classes

The implementation lives in [lib/content-type-parser.js](../../lib/content-type-parser.js).

`ContentTypeParser` is a plain constructor function (not a class) that holds:

```text
ContentTypeParser {
  customParsers:    Map<string, Parser>   // key is content-type string or regexp.toString()
  parserList:       string[]              // string types in registration order (newest first)
  parserRegExpList: RegExp[]              // regexp types in registration order (newest first)
  cache:            Fifo (LRU, cap 100)  // resolved-type -> Parser cache
  [kDefaultJsonParse]: Function          // reference to the current default JSON parser
}
```

`Parser` is a small value object:

```text
Parser {
  asString:  boolean   // true → Fastify buffers as string before calling fn
  asBuffer:  boolean   // true → Fastify buffers as Buffer before calling fn
  bodyLimit: number    // max bytes; null falls back to the server bodyLimit
  fn:        Function  // the actual parser (request, payload/body, done?)
}
```

### 12.2 Lookup algorithm and LRU cache

`ContentTypeParser.prototype.getParser(contentType)` follows this order:

```
1. Normalise incoming content-type with the ContentType helper
   (strips parameters, lowercases)

2. Check the LRU cache (100-entry Fifo map) — O(1) for seen types

3. Exact-string match in customParsers

4. Media-type-only match (ignores ; charset=…, ; boundary=…, etc.)

5. RegExp scan through parserRegExpList

6. Wildcard fallback (customParsers.get(''))
```

The LRU cache means that the RegExp scan — which is `O(n)` over the number of regex
parsers — only runs **once per unique content-type string** seen across the server's
lifetime.  Subsequent requests with the same content-type hit the cache directly.

### 12.3 rawBody and the bodyLimit guard

When `parseAs` is `'string'` or `'buffer'`, Fastify delegates to the internal `rawBody`
function which:

1. Reads `Content-Length` from the request header and immediately rejects if it exceeds
   the limit (fast-fail, no data buffered).
2. Accumulates chunks in a string or `Buffer[]`.
3. After each `data` event, checks both `receivedLength` (decoded bytes accumulated so
   far) and `payload.receivedEncodedLength` (set by the `preParsing` hook chain when a
   compressing transform is in use) against the limit.
4. After `end`, compares the actual accumulated length to the declared `Content-Length`.
5. Calls your parser function with the fully-assembled string or `Buffer`.

The dual-length check (`receivedLength` + `receivedEncodedLength`) guards against **zip
bomb** attacks where a small compressed payload expands into a much larger body.

### 12.4 AsyncResource integration

`ContentTypeParser.prototype.run` wraps each parser call in a `node:async_hooks`
`AsyncResource` named `'content-type-parser:run'`.  This preserves the async context
(e.g. `AsyncLocalStorage` stores) so that code inside parsers — including third-party
libraries — inherits the same async context as the incoming request.

```js
// AsyncResource ensures CLS/ALS context is correctly propagated
const resource = new AsyncResource('content-type-parser:run', request)
const done = resource.bind(onDone)
```

`resource.emitDestroy()` is called inside `onDone` so that `async_hooks` destroy events
fire correctly.

---

## 13. TypeScript Quick-Reference

```typescript
import Fastify, { FastifyInstance, FastifyRequest } from 'fastify'

const app: FastifyInstance = Fastify()

// --- Callback style with parseAs: 'string' ---
app.addContentTypeParser<string>(
  'application/x-yaml',
  { parseAs: 'string' },
  (req: FastifyRequest, body: string, done) => {
    try {
      done(null, parseYaml(body))
    } catch (err: unknown) {
      const error = err as Error
      ;(error as any).statusCode = 400
      done(error)
    }
  }
)

// --- Async style with parseAs: 'buffer' ---
app.addContentTypeParser<Buffer>(
  'application/protobuf',
  { parseAs: 'buffer' },
  async (req: FastifyRequest, body: Buffer) => {
    return MyProto.decode(body)
  }
)

// --- Stream style (no parseAs) ---
app.addContentTypeParser(
  'application/x-msgpack',
  async (req: FastifyRequest, payload: FastifyRequest['raw']) => {
    const chunks: Buffer[] = []
    for await (const chunk of payload) {
      chunks.push(chunk as Buffer)
    }
    return msgpack.decode(Buffer.concat(chunks))
  }
)

// --- RegExp ---
app.addContentTypeParser(
  /^application\/.*\+json$/,
  { parseAs: 'string' },
  async (_req, body: string) => JSON.parse(body)
)

// --- Array of types ---
app.addContentTypeParser(
  ['application/json', 'text/json'],
  { parseAs: 'string' },
  app.getDefaultJsonParser('error', 'error')
)

// --- Management ---
const hasXml: boolean = app.hasContentTypeParser('application/xml')
app.removeContentTypeParser('text/plain')
app.removeAllContentTypeParsers()
```

---

## 14. Error Code Reference

| Error code | HTTP status | When it is thrown |
|---|---|---|
| `FST_ERR_CTP_INVALID_MEDIA_TYPE` | 415 | No parser found for the incoming `Content-Type` |
| `FST_ERR_CTP_BODY_TOO_LARGE` | 413 | Buffered body exceeds the active `bodyLimit` |
| `FST_ERR_CTP_INVALID_CONTENT_LENGTH` | 400 | Buffered length ≠ declared `Content-Length` header |
| `FST_ERR_CTP_EMPTY_JSON_BODY` | 400 | JSON body is a zero-length string |
| `FST_ERR_CTP_INVALID_JSON_BODY` | 400 | JSON body cannot be parsed |
| `FST_ERR_CTP_ALREADY_PRESENT` | — | `addContentTypeParser` called for an already-registered type |
| `FST_ERR_CTP_INVALID_TYPE` | — | `contentType` argument is not a string or RegExp |
| `FST_ERR_CTP_EMPTY_TYPE` | — | `contentType` string is empty after trimming |
| `FST_ERR_CTP_INVALID_HANDLER` | — | Parser argument is not a function |
| `FST_ERR_CTP_INVALID_PARSE_TYPE` | — | `parseAs` option is not `'string'` or `'buffer'` |
| `FST_ERR_CTP_INSTANCE_ALREADY_STARTED` | — | Management API called after server start |
