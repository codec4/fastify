# Fastify Hooks — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's hook system from first principles.

---

## Table of Contents

1. [What are Hooks?](#1-what-are-hooks)
2. [Hook Categories](#2-hook-categories)
3. [Request / Response Lifecycle](#3-request--response-lifecycle)
4. [Lifecycle Hooks — Deep Dive](#4-lifecycle-hooks--deep-dive)
   - 4.1 [onRequest](#41-onrequest)
   - 4.2 [preParsing](#42-preparsing)
   - 4.3 [preValidation](#43-prevalidation)
   - 4.4 [preHandler](#44-prehandler)
   - 4.5 [preSerialization](#45-preserialization)
   - 4.6 [onSend](#46-onsend)
   - 4.7 [onResponse](#47-onresponse)
   - 4.8 [onError](#48-onerror)
   - 4.9 [onTimeout](#49-ontimeout)
   - 4.10 [onRequestAbort](#410-onrequestabort)
5. [Application Hooks — Deep Dive](#5-application-hooks--deep-dive)
   - 5.1 [onRoute](#51-onroute)
   - 5.2 [onRegister](#52-onregister)
   - 5.3 [onReady](#53-onready)
   - 5.4 [onListen](#54-onlisten)
   - 5.5 [preClose](#55-preclose)
   - 5.6 [onClose](#56-onclose)
6. [Callback vs Async Signatures](#6-callback-vs-async-signatures)
7. [Encapsulation & Scope](#7-encapsulation--scope)
8. [Route-Level Hooks](#8-route-level-hooks)
9. [Hook Execution Order](#9-hook-execution-order)
10. [Error Handling in Hooks](#10-error-handling-in-hooks)
11. [Real-World Patterns](#11-real-world-patterns)
12. [TypeScript Quick-Reference](#12-typescript-quick-reference)

Related deep dives:

- [Fastify Lifecycle - Complete Tutorial](./fastify-lifecycle.tutorial.md)
- [Fastify Streams — Complete Tutorial](./fastify-streams.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)

---

## 1. What are Hooks?

Hooks are ordered lists of functions that Fastify runs at well-defined points in either
the **request lifecycle** or the **application lifecycle**.  They let you inject
cross-cutting logic — authentication, logging, payload transformation, metrics — without
coupling that logic to individual route handlers.

```
addHook(name, fn)   --- registers fn in the hook list for the current scope
                         fn runs every time that lifecycle point is reached
```

Internally Fastify stores hook functions in a plain `Hooks` object (see
[lib/hooks.js](../../lib/hooks.js)):

```
Hooks {
  onRequest:        []
  preParsing:       []
  preValidation:    []
  preSerialization: []
  preHandler:       []
  onSend:           []
  onResponse:       []
  onError:          []
  onTimeout:        []
  onRequestAbort:   []
  onRoute:          []
  onRegister:       []
  onReady:          []
  onListen:         []
  preClose:         []
  onClose:          (avvio-managed)
}
```

When a child plugin is registered, Fastify **copies** the parent hook array
(`h.onRequest.slice()`, etc.) so every child starts with the parent's hooks already
in place — this is the heart of encapsulation.

---

## 2. Hook Categories

```
+---------------------------------------------------------+
|                     HOOK CATEGORIES                     |
+---------------------------+-----------------------------+
|   LIFECYCLE HOOKS         |   APPLICATION HOOKS         |
|  (per-request)            |  (once, at startup/shutdown)|
+---------------------------+-----------------------------+
|  onRequest                |  onRoute                    |
|  preParsing               |  onRegister                 |
|  preValidation            |  onReady                    |
|  preHandler               |  onListen                   |
|  preSerialization         |  preClose                   |
|  onSend                   |  onClose                    |
|  onResponse               |                             |
|  onError                  |                             |
|  onTimeout                |                             |
|  onRequestAbort           |                             |
+---------------------------+-----------------------------+
```

---

## 3. Request / Response Lifecycle

The diagram below covers the *happy path* (no errors).  Error branches are shown with
`x`.

```
  Incoming HTTP Request
         |
         v
  +-------------+
  |  onRequest  |  <- set auth context, rate-limit, CORS headers
  +------+------+
         |
         v
  +-------------+
  | preParsing  |  <- inspect / replace raw payload stream
  +------+------+
         |  body parsed by content-type parser
         v
  +------------------+
  |  preValidation   |  <- custom auth/sanitise before schema check
  +--------+---------+
           |  JSON-Schema validation runs here
           v
  +-------------+
  | preHandler  |  <- last chance to mutate request before handler
  +------+------+
         |
         v
  +----------------------------+
  |   Route Handler            |
  |   (your business logic)    |
  +--------------+-------------+
                 |  reply.send(payload)
                 v
  +----------------------+
  |  preSerialization    |  <- mutate payload object before serialization
  +----------+-----------+
             |  JSON serializer runs
             v
  +----------------------+
  |      onSend          |  <- mutate serialized payload string/buffer
  +----------+-----------+
             |  headers + body written to socket
             v
  +------------------------+
  |      onResponse        |  <- logging, metrics, cleanup
  +------------------------+


  -- Error branch -------------------------------------
  Any hook or handler throws / calls done(err)
         |
         v
  +--------------+
  |   onError    |  <- customise error reply, DO NOT call reply.send here
  +------+-------+
         |
         v  (error serialised -> onSend -> onResponse)
  +------------------------+
  |      onResponse        |
  +------------------------+

  -- Timeout branch -----------------------------------
  connectionTimeout or requestTimeout fires
         |
         v
  +--------------+
  |  onTimeout   |
  +--------------+

  -- Abort branch -------------------------------------
  Client disconnects mid-flight
         |
         v
  +------------------+
  |  onRequestAbort  |
  +------------------+
```

---

## 4. Lifecycle Hooks — Deep Dive

### 4.1 `onRequest`

**When:** immediately after the HTTP socket produces a request object, before any body
reading.

**Signature:**
```typescript
// callback style
fastify.addHook('onRequest', (
  request: FastifyRequest,
  reply:   FastifyReply,
  done:    HookHandlerDoneFunction
) => void)

// async style
fastify.addHook('onRequest', async (
  request: FastifyRequest,
  reply:   FastifyReply
) => void)
```

**Use-cases:** authentication, setting request-scoped state, CORS pre-flight, IP
allow-listing.

```typescript
import Fastify, {
  FastifyInstance,
  FastifyRequest,
  FastifyReply
} from 'fastify'

const app: FastifyInstance = Fastify({ logger: true })

// -- Example: Bearer-token authentication ----------------------------------
app.addHook('onRequest', async (request: FastifyRequest, reply: FastifyReply) => {
  const auth = request.headers['authorization']
  if (!auth || !auth.startsWith('Bearer ')) {
    // Throwing inside an async hook routes to onError then the error serialiser
    reply.code(401)
    throw new Error('Missing or invalid Authorization header')
  }
  // Attach decoded identity to request for downstream use
  ;(request as any).identity = decodeToken(auth.slice(7))
})

function decodeToken (token: string): { userId: string } {
  // simplified — use a real JWT library in production
  return { userId: Buffer.from(token, 'base64').toString() }
}
```

**Key behaviour:**
- Runs **before** the body is read — ideal for short-circuiting unauthenticated requests
  cheaply.
- Calling `reply.send()` inside the hook terminates the lifecycle; `onResponse` still
  fires.

---

### 4.2 `preParsing`

**When:** after `onRequest`, before the body is parsed by the content-type parser.

**Signature:**
```typescript
fastify.addHook('preParsing', (
  request: FastifyRequest,
  reply:   FastifyReply,
  payload: RequestPayload,           // the raw readable stream
  done:    (err: Error | null, value?: RequestPayload) => void
) => void)

// async — return new stream to replace payload
fastify.addHook('preParsing', async (
  request: FastifyRequest,
  reply:   FastifyReply,
  payload: RequestPayload
): Promise<RequestPayload | void> => { /* … */ })
```

**Use-cases:** decompression, decryption, body size limiting at stream level.

```typescript
import { Readable } from 'node:stream'
import { createGunzip } from 'node:zlib'
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

const app = Fastify()

// -- Example: transparent gzip decompression -------------------------------
app.addHook('preParsing', (
  request: FastifyRequest,
  reply:   FastifyReply,
  payload: Readable,
  done: (err: Error | null, stream?: Readable) => void
) => {
  if (request.headers['content-encoding'] === 'gzip') {
    const gunzip = createGunzip()
    done(null, payload.pipe(gunzip))
  } else {
    done(null, payload)   // pass through unchanged
  }
})
```

**Key behaviour:**
- The `done` callback (or async return value) accepts a **new stream** that replaces the
  original payload stream.
- If `reply.sent` is `true` when `done` fires, the runner stops — preventing double
  responses.

---

### 4.3 `preValidation`

**When:** after body parsing, **before** JSON-Schema validation.

**Signature:**
```typescript
fastify.addHook('preValidation', (
  request: FastifyRequest,
  reply:   FastifyReply,
  done:    HookHandlerDoneFunction
) => void)

fastify.addHook('preValidation', async (
  request: FastifyRequest,
  reply:   FastifyReply
) => void)
```

**Use-cases:** coerce/normalise body fields, add computed properties to `request.body`
before schema validation runs, role-based access checks that need the parsed body.

```typescript
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

interface CreateUserBody {
  email: string
  role?: string
}

const app = Fastify()

// -- Example: normalise email + inject default role ------------------------
app.addHook('preValidation', async (request: FastifyRequest) => {
  const body = request.body as CreateUserBody | undefined
  if (body?.email) {
    body.email = body.email.toLowerCase().trim()
  }
  if (body && !body.role) {
    body.role = 'viewer'
  }
})

app.post<{ Body: CreateUserBody }>(
  '/users',
  {
    schema: {
      body: {
        type: 'object',
        required: ['email'],
        properties: {
          email: { type: 'string', format: 'email' },
          role:  { type: 'string', enum: ['admin', 'editor', 'viewer'] }
        }
      }
    }
  },
  async (request, reply) => {
    // body.email is already lowercased when we reach here
    reply.send({ created: request.body.email })
  }
)
```

---

### 4.4 `preHandler`

**When:** after JSON-Schema validation passes, immediately before the route handler.

**Signature:**
```typescript
fastify.addHook('preHandler', (
  request: FastifyRequest,
  reply:   FastifyReply,
  done:    HookHandlerDoneFunction
) => void)

fastify.addHook('preHandler', async (
  request: FastifyRequest,
  reply:   FastifyReply
) => void)
```

**Use-cases:** permission checks that need validated body/params, enriching request
objects with database-fetched data.

```typescript
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

const app = Fastify()

// -- Example: load current user from DB and attach to request --------------
app.addHook('preHandler', async (request: FastifyRequest) => {
  const userId = (request as any).identity?.userId
  if (userId) {
    ;(request as any).currentUser = await fetchUserFromDb(userId)
  }
})

async function fetchUserFromDb (id: string) {
  // replace with real DB call
  return { id, name: 'Alice', roles: ['admin'] }
}

app.get('/me', async (request, reply) => {
  reply.send((request as any).currentUser)
})
```

---

### 4.5 `preSerialization`

**When:** after the handler calls `reply.send(payload)`, before the payload is
JSON-serialised.  Not called when the payload is a `string`, `Buffer`, `stream`, or
`null`.

**Signature:**
```typescript
fastify.addHook('preSerialization', (
  request: FastifyRequest,
  reply:   FastifyReply,
  payload: unknown,     // the object passed to reply.send()
  done:    (err: Error | null, value?: unknown) => void
) => void)

fastify.addHook('preSerialization', async (
  request: FastifyRequest,
  reply:   FastifyReply,
  payload: unknown
): Promise<unknown> => { /* return (possibly modified) payload */ })
```

**Use-cases:** wrap every response in an envelope, add pagination metadata, redact
sensitive fields.

```typescript
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

const app = Fastify()

interface Envelope<T> {
  data: T
  meta: { timestamp: number; version: string }
}

// -- Example: universal response envelope ---------------------------------
app.addHook('preSerialization', async <T>(
  request: FastifyRequest,
  reply:   FastifyReply,
  payload: T
): Promise<Envelope<T>> => {
  return {
    data: payload,
    meta: {
      timestamp: Date.now(),
      version:   '1.0.0'
    }
  }
})

app.get('/ping', async () => ({ pong: true }))
// Response body: { "data": { "pong": true }, "meta": { "timestamp": …, "version": "1.0.0" } }
```

---

### 4.6 `onSend`

**When:** after serialisation — payload is now a `string`, `Buffer`, or `stream`.
This is your last chance to alter the bytes going to the client.

**Signature:**
```typescript
fastify.addHook('onSend', (
  request: FastifyRequest,
  reply:   FastifyReply,
  serializedPayload: string | Buffer | null,
  done: (err: Error | null, value?: string | Buffer | null) => void
) => void)

fastify.addHook('onSend', async (
  request: FastifyRequest,
  reply:   FastifyReply,
  serializedPayload: string | Buffer | null
): Promise<string | Buffer | null | undefined> => { /* … */ })
```

**Use-cases:** response compression, adding/modifying headers based on final payload
size, masking/scrubbing serialised output.

```typescript
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

const app = Fastify()

// -- Example: add ETag header based on payload hash ------------------------
import { createHash } from 'node:crypto'

app.addHook('onSend', async (
  request:  FastifyRequest,
  reply:    FastifyReply,
  payload:  string | Buffer | null
): Promise<string | Buffer | null> => {
  if (payload) {
    const hash = createHash('sha1')
      .update(typeof payload === 'string' ? payload : payload)
      .digest('hex')
      .slice(0, 16)
    reply.header('etag', `"${hash}"`)
  }
  return payload  // returning undefined would keep existing payload
})
```

**Key behaviour:**
- Returning (or calling `done` with) a **new value** replaces the payload.
- Returning `undefined` (or calling `done(null)` with no second argument) preserves
  the existing payload.

---

### 4.7 `onResponse`

**When:** after the response has been sent to the client (socket write finished).

**Signature:**
```typescript
fastify.addHook('onResponse', (
  request: FastifyRequest,
  reply:   FastifyReply,
  done:    HookHandlerDoneFunction
) => void)

fastify.addHook('onResponse', async (
  request: FastifyRequest,
  reply:   FastifyReply
) => void)
```

**Use-cases:** metrics/tracing, audit logging, releasing resources.

```typescript
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

const app = Fastify()

// -- Example: record request duration metric -------------------------------
app.addHook('onResponse', async (request: FastifyRequest, reply: FastifyReply) => {
  const durationMs = reply.elapsedTime  // built-in elapsed time in ms
  recordMetric({
    route:      request.routeOptions.url,
    method:     request.method,
    statusCode: reply.statusCode,
    durationMs
  })
})

function recordMetric (m: object) {
  // push to Prometheus, StatsD, OpenTelemetry, etc.
  console.log('[metric]', m)
}
```

---

### 4.8 `onError`

**When:** when an error reaches Fastify's error handler — either thrown from a hook or
a route handler, or passed to `done(err)`.

**Signature:**
```typescript
fastify.addHook('onError', (
  request: FastifyRequest,
  reply:   FastifyReply,
  error:   FastifyError,
  done:    () => void          // <- NO error argument; can't change the error here
) => void)

fastify.addHook('onError', async (
  request: FastifyRequest,
  reply:   FastifyReply,
  error:   FastifyError
) => void)
```

> **Important:** `onError` is for **side-effects** (logging, metrics) only.  You
> cannot change the error or the response from here.  To customise the error response
> use `setErrorHandler`.

```typescript
import Fastify, { FastifyRequest, FastifyReply, FastifyError } from 'fastify'

const app = Fastify()

// -- Example: structured error audit log -----------------------------------
app.addHook('onError', async (
  request: FastifyRequest,
  reply:   FastifyReply,
  error:   FastifyError
) => {
  request.log.error({
    err:    error,
    url:    request.url,
    method: request.method
  }, 'Request error')
})
```

---

### 4.9 `onTimeout`

**When:** when `connectionTimeout` or `requestTimeout` elapses before the response is
sent.

**Signature:**
```typescript
fastify.addHook('onTimeout', (
  request: FastifyRequest,
  reply:   FastifyReply,
  done:    HookHandlerDoneFunction
) => void)

fastify.addHook('onTimeout', async (
  request: FastifyRequest,
  reply:   FastifyReply
) => void)
```

```typescript
import Fastify, { FastifyRequest } from 'fastify'

const app = Fastify({ requestTimeout: 5000 })

// -- Example: log slow requests --------------------------------------------
app.addHook('onTimeout', async (request: FastifyRequest) => {
  request.log.warn({
    url:    request.url,
    method: request.method
  }, 'Request timed out')
})
```

---

### 4.10 `onRequestAbort`

**When:** when the client closes the connection before the server finishes responding.

> Available since Fastify v4.

**Signature:**
```typescript
fastify.addHook('onRequestAbort', (
  request: FastifyRequest,
  done:    HookHandlerDoneFunction
) => void)

fastify.addHook('onRequestAbort', async (
  request: FastifyRequest
) => void)
```

```typescript
import Fastify, { FastifyRequest } from 'fastify'

const app = Fastify()

// -- Example: cancel in-flight DB queries when client disconnects ----------
app.addHook('onRequestAbort', async (request: FastifyRequest) => {
  const abortController = (request as any).abortController as AbortController | undefined
  abortController?.abort()
  request.log.info({ url: request.url }, 'Client aborted request')
})
```

---

## 5. Application Hooks — Deep Dive

Application hooks fire once during server startup or shutdown.  They are **not**
per-request; they do not receive `request`/`reply` objects.

### 5.1 `onRoute`

**When:** synchronously whenever a new route is registered in the current scope.

**Signature:**
```typescript
fastify.addHook('onRoute', (routeOptions: RouteOptions) => void)
```

> `onRoute` is **synchronous only** — no `done` callback, no `async`.

```typescript
import Fastify, { FastifyInstance, RouteOptions } from 'fastify'

const app: FastifyInstance = Fastify()

// -- Example: enforce Content-Type schema on every POST route --------------
app.addHook('onRoute', (routeOptions: RouteOptions) => {
  if (routeOptions.method === 'POST' || routeOptions.method === 'PUT') {
    routeOptions.schema = routeOptions.schema ?? {}
    routeOptions.schema.headers = routeOptions.schema.headers ?? {
      type: 'object',
      required: ['content-type'],
      properties: {
        'content-type': { type: 'string' }
      }
    }
  }
})
```

---

### 5.2 `onRegister`

**When:** synchronously when a new plugin scope (child `FastifyInstance`) is created.

**Signature:**
```typescript
fastify.addHook('onRegister', (
  instance: FastifyInstance,
  opts:     RegisterOptions
) => void)
```

```typescript
import Fastify, { FastifyInstance } from 'fastify'

const app: FastifyInstance = Fastify()

// -- Example: track registered plugin names --------------------------------
const registeredPlugins: string[] = []

app.addHook('onRegister', (instance: FastifyInstance) => {
  registeredPlugins.push(instance.pluginName)
})
```

---

### 5.3 `onReady`

**When:** after all plugins have loaded and all `register` calls have resolved, just
before the server starts listening.  Triggered by `fastify.ready()` or
`fastify.listen()`.

**Signature:**
```typescript
// callback style — `this` is the Fastify instance
fastify.addHook('onReady', function (done: HookHandlerDoneFunction): void { … })

// async style
fastify.addHook('onReady', async function (): Promise<void> { … })
```

**Key behaviours (from tests):**
- Called once per `ready()` call regardless of how many times `ready()` is awaited.
- Fired depth-first through the plugin tree (root -> child1 -> child2 …).
- `this` is bound to the Fastify instance that owns the hook.

```typescript
import Fastify, { FastifyInstance } from 'fastify'

const app: FastifyInstance = Fastify()

// -- Example: warm up a connection pool -----------------------------------
app.addHook('onReady', async function (this: FastifyInstance) {
  this.log.info('Initialising DB connection pool…')
  await initDbPool()
  this.log.info('DB pool ready')
})

async function initDbPool () {
  // await pool.connect()
}
```

---

### 5.4 `onListen`

**When:** after the server is bound to a port and actively accepting connections.  Fires
**only** when `fastify.listen()` is called (not on `fastify.ready()`).

**Signature:**
```typescript
fastify.addHook('onListen', function (done: HookHandlerDoneFunction): void { … })
fastify.addHook('onListen', async function (): Promise<void> { … })
```

**Key behaviours (from tests):**
- Multiple `onListen` hooks execute **in registration order**.
- If a hook throws/errors, the error is **logged but execution continues** to the
  next hook — `onListen` errors are non-fatal by design.

```typescript
import Fastify from 'fastify'

const app = Fastify({ logger: true })

// -- Example: announce service readiness to a health-check endpoint --------
app.addHook('onListen', async function () {
  const address = app.server.address()
  console.log(`Service listening on ${JSON.stringify(address)}`)
  // e.g. register with service registry, start metrics scraper…
})
```

---

### 5.5 `preClose`

**When:** triggered when `fastify.close()` is called, **before** active connections are
drained and before `onClose`.

**Signature:**
```typescript
fastify.addHook('preClose', function (done: HookHandlerDoneFunction): void { … })
fastify.addHook('preClose', async function (): Promise<void> { … })
```

```typescript
import Fastify from 'fastify'

const app = Fastify()

// -- Example: stop accepting new jobs before draining ---------------------
app.addHook('preClose', async function () {
  await jobQueue.pause()   // stop consuming new messages
  app.log.info('Job queue paused — draining in-flight work')
})

declare const jobQueue: { pause(): Promise<void> }
```

---

### 5.6 `onClose`

**When:** after all connections are drained, just before the process can exit.

**Signature:**
```typescript
fastify.addHook('onClose', (
  instance: FastifyInstance,
  done:     HookHandlerDoneFunction
) => void)

fastify.addHook('onClose', async (
  instance: FastifyInstance
) => void)
```

```typescript
import Fastify, { FastifyInstance } from 'fastify'

const app = Fastify()

// -- Example: close DB pool on shutdown -----------------------------------
app.addHook('onClose', async (instance: FastifyInstance) => {
  instance.log.info('Closing DB pool…')
  await closeDbPool()
})

async function closeDbPool () {
  // await pool.end()
}
```

---

## 6. Callback vs Async Signatures

Fastify detects which style you use via the **function arity** (`.length`) and whether
the return value has a `.then` method.

```
+----------------------------------------------------------------------------+
|  Hook style detection (lib/hooks.js hookRunnerGenerator)                  |
+----------------------------------+-----------------------------------------+
|  fn.length >= expected with done |  callback-style — Fastify calls done()  |
|  fn returns a Promise (.then)    |  async-style    — Fastify awaits result  |
|  neither                         |  sync — Fastify auto-calls done()        |
+----------------------------------+-----------------------------------------+
```

### Callback style

```typescript
import Fastify, { FastifyRequest, FastifyReply, HookHandlerDoneFunction } from 'fastify'

const app = Fastify()

app.addHook('preHandler', (
  request: FastifyRequest,
  reply:   FastifyReply,
  done:    HookHandlerDoneFunction
): void => {
  // synchronous work
  ;(request as any).startAt = Date.now()
  done()          // must call done — omitting it hangs the request
})
```

### Async style

```typescript
app.addHook('preHandler', async (
  request: FastifyRequest,
  reply:   FastifyReply
): Promise<void> => {
  ;(request as any).startAt = Date.now()
  // no done() needed — Fastify awaits the returned Promise
  // throw to signal an error
})
```

> **Rule of thumb:** prefer `async/await` — it's harder to forget `done()` and
> interacts safely with `try/catch`.

---

## 7. Encapsulation & Scope

Fastify's plugin system creates a **tree of scoped instances**.  Hook registration
respects this tree:

```
  Root instance
  |  addHook('onRequest', authHook)  <--- runs for ALL routes in root + children
  |
  +-- Plugin A (child scope)
  |   |  addHook('preHandler', adminHook)  <--- runs only inside Plugin A
  |   |
  |   +-- Plugin B (grandchild)
  |       +-- GET /admin/report         <--- runs authHook + adminHook
  |
  +-- GET /public                       <--- runs authHook only
```

```typescript
import Fastify, {
  FastifyInstance,
  FastifyRequest,
  FastifyReply,
  FastifyPluginAsync
} from 'fastify'

const app: FastifyInstance = Fastify()

// -- root-level hook: applies everywhere -----------------------------------
app.addHook('onRequest', async (request: FastifyRequest) => {
  ;(request as any).requestId = crypto.randomUUID()
})

// -- scoped plugin: internal admin routes ----------------------------------
const adminPlugin: FastifyPluginAsync = async (instance) => {
  instance.addHook('preHandler', async (request: FastifyRequest, reply: FastifyReply) => {
    const user = (request as any).currentUser
    if (!user?.roles.includes('admin')) {
      reply.code(403)
      throw new Error('Forbidden')
    }
  })

  instance.get('/admin/report', async () => ({ report: 'data' }))
}

app.register(adminPlugin, { prefix: '/internal' })
app.get('/health', async () => ({ ok: true }))
```

To **share** a hook across sibling plugins use
[`fastify-plugin`](https://github.com/fastify/fastify-plugin) which disables
encapsulation:

```typescript
import fp from 'fastify-plugin'
import { FastifyInstance, FastifyPluginAsync } from 'fastify'

const sharedHookPlugin: FastifyPluginAsync = async (instance: FastifyInstance) => {
  instance.addHook('onRequest', async (request) => {
    ;(request as any).traceId = crypto.randomUUID()
  })
}

// fp() breaks encapsulation — hook leaks up to parent scope
app.register(fp(sharedHookPlugin))
```

---

## 8. Route-Level Hooks

Hooks can be attached directly to a single route definition.  They run **after** the
scope-level hooks of the same type.

```typescript
import Fastify, {
  FastifyRequest,
  FastifyReply
} from 'fastify'

const app = Fastify()

app.get<{ Params: { id: string } }>(
  '/orders/:id',
  {
    // runs after any preHandler hooks registered on the instance
    preHandler: async (request: FastifyRequest<{ Params: { id: string } }>, reply: FastifyReply) => {
      const orderId = request.params.id
      if (!isValidOrderId(orderId)) {
        reply.code(400)
        throw new Error(`Invalid order id: ${orderId}`)
      }
    },
    // arrays are also supported
    onSend: [
      async (request, reply, payload) => {
        reply.header('x-served-by', 'orders-service')
        return payload
      }
    ]
  },
  async (request, reply) => {
    reply.send({ id: request.params.id, status: 'shipped' })
  }
)

function isValidOrderId (id: string): boolean {
  return /^\d{6,10}$/.test(id)
}
```

**Execution order for a route with both scope and route hooks:**
```
  scope:onRequest[0]  ->  scope:onRequest[1]  ->  route:onRequest
  scope:preHandler[0] ->  route:preHandler[0] ->  route:preHandler[1]
  handler
  …
```

---

## 9. Hook Execution Order

### Within a single scope

Hooks execute in **registration order**.

```typescript
app.addHook('onRequest', async (req) => { console.log('1') })
app.addHook('onRequest', async (req) => { console.log('2') })
app.addHook('onRequest', async (req) => { console.log('3') })
// Output: 1  2  3
```

### Across plugin scopes

Each child scope **inherits a copy** of the parent's hooks, then appends its own.

```
  root -> addHook('onRequest', A)
    +- plugin -> addHook('onRequest', B)

  Request to plugin route: A -> B
  Request to root route:   A
```

### `onReady` — depth-first

```
  root      -> onReady[0]
    child1  -> onReady[0]
      child2  -> onReady[0]
```

---

## 10. Error Handling in Hooks

```
  Hook throws / done(err)
        |
        v
  +---------------------------------+
  |  Is there a setErrorHandler?    |
  |  (scoped or root)               |
  +--------------+------------------+
                 | yes                          no
                 v                              v
     custom errorHandler runs      default 500 JSON response
                 |
                 v
         onError hook(s)  <- side effects only
                 |
                 v
         onSend  ->  onResponse
```

```typescript
import Fastify, {
  FastifyInstance,
  FastifyRequest,
  FastifyReply,
  FastifyError
} from 'fastify'

const app: FastifyInstance = Fastify()

// --- Custom error handler -------------------------------------------------
app.setErrorHandler(async (
  error:   FastifyError,
  request: FastifyRequest,
  reply:   FastifyReply
) => {
  const statusCode = error.statusCode ?? 500
  reply.code(statusCode).send({
    error:   error.message,
    code:    error.code,
    traceId: (request as any).traceId
  })
})

// --- onError for side-effects ---------------------------------------------
app.addHook('onError', async (
  request: FastifyRequest,
  reply:   FastifyReply,
  error:   FastifyError
) => {
  // emit to external error tracker
  console.error('[error-tracker]', {
    url:    request.url,
    status: error.statusCode,
    msg:    error.message
  })
})
```

### Recovering inside a hook

Call `reply.send()` inside `onRequest` / `preHandler` to bypass the remaining lifecycle:

```typescript
app.addHook('onRequest', async (request: FastifyRequest, reply: FastifyReply) => {
  if (request.headers['x-maintenance'] === '1') {
    await reply.code(503).send({ message: 'Service under maintenance' })
    // remaining hooks + handler are skipped; onResponse still fires
  }
})
```

---

## 11. Real-World Patterns

### Pattern 1 — Request tracing with context propagation

```typescript
import Fastify, { FastifyRequest, FastifyReply, FastifyInstance } from 'fastify'
import { AsyncLocalStorage } from 'node:async_hooks'

interface TraceContext { traceId: string; spanId: string }

const als = new AsyncLocalStorage<TraceContext>()
const app: FastifyInstance = Fastify()

app.addHook('onRequest', (
  request: FastifyRequest,
  _reply:  FastifyReply,
  done:    () => void
) => {
  const ctx: TraceContext = {
    traceId: request.headers['x-trace-id'] as string ?? crypto.randomUUID(),
    spanId:  crypto.randomUUID()
  }
  als.run(ctx, done)   // all downstream code shares this context
})

app.addHook('onResponse', async (_request: FastifyRequest, reply: FastifyReply) => {
  const ctx = als.getStore()
  if (ctx) reply.header('x-trace-id', ctx.traceId)
})
```

### Pattern 2 — Payload compression with onSend

```typescript
import Fastify from 'fastify'
import { gzip } from 'node:zlib'
import { promisify } from 'node:util'

const gzipAsync = promisify(gzip)
const app = Fastify()

app.addHook('onSend', async (request, reply, payload: string | Buffer | null) => {
  if (!payload) return payload
  const acceptEncoding = request.headers['accept-encoding'] ?? ''
  if (!acceptEncoding.includes('gzip')) return payload

  const compressed = await gzipAsync(
    typeof payload === 'string' ? Buffer.from(payload) : payload
  )
  reply.header('content-encoding', 'gzip')
  reply.removeHeader('content-length')   // length changed after compression
  return compressed
})
```

### Pattern 3 — Plugin with lifecycle ownership

```typescript
import fp from 'fastify-plugin'
import Fastify, {
  FastifyInstance,
  FastifyPluginAsync,
  FastifyRequest,
  FastifyReply
} from 'fastify'

// A plugin that owns its own lifecycle hooks
const metricsPlugin: FastifyPluginAsync = async (instance: FastifyInstance) => {
  const histogram = new Map<string, number[]>()

  instance.addHook('onRequest', async (request: FastifyRequest) => {
    ;(request as any)._metricStart = performance.now()
  })

  instance.addHook('onResponse', async (request: FastifyRequest, reply: FastifyReply) => {
    const start = (request as any)._metricStart as number | undefined
    if (start === undefined) return
    const ms = performance.now() - start
    const key = `${request.method} ${request.routeOptions.url ?? 'unknown'}`
    const bucket = histogram.get(key) ?? []
    bucket.push(ms)
    histogram.set(key, bucket)
  })

  instance.addHook('onClose', async () => {
    // flush metrics before shutdown
    for (const [route, timings] of histogram) {
      const avg = timings.reduce((a, b) => a + b, 0) / timings.length
      console.log(`[metrics] ${route} p50≈${avg.toFixed(1)}ms (${timings.length} reqs)`)
    }
  })

  // expose histogram for testing
  instance.decorate('metricsHistogram', histogram)
}

export default fp(metricsPlugin, { name: 'metrics' })
```

---

## 12. TypeScript Quick-Reference

```typescript
import Fastify, {
  FastifyInstance,
  FastifyRequest,
  FastifyReply,
  FastifyError,
  HookHandlerDoneFunction,
  RouteOptions,
  RegisterOptions
} from 'fastify'

const app: FastifyInstance = Fastify()

// -- Lifecycle hooks -------------------------------------------------------

// onRequest / preValidation / preHandler / onResponse / onTimeout
app.addHook('onRequest', async (req: FastifyRequest, reply: FastifyReply) => {})
app.addHook('onRequest', (req: FastifyRequest, reply: FastifyReply, done: HookHandlerDoneFunction) => { done() })

// preParsing — extra `payload` argument
app.addHook('preParsing', async (req, reply, payload) => { return payload })

// preSerialization / onSend — extra payload argument, can return new value
app.addHook('preSerialization', async (req, reply, payload: unknown) => payload)
app.addHook('onSend',           async (req, reply, payload: string | null) => payload)

// onError — no reply.send(), side-effects only
app.addHook('onError', async (req: FastifyRequest, reply: FastifyReply, err: FastifyError) => {})

// onRequestAbort — request only
app.addHook('onRequestAbort', async (req: FastifyRequest) => {})

// -- Application hooks -----------------------------------------------------

// onRoute (sync only)
app.addHook('onRoute', (opts: RouteOptions) => {})

// onRegister (sync only)
app.addHook('onRegister', (instance: FastifyInstance, opts: RegisterOptions) => {})

// onReady / onListen / preClose — `this` = FastifyInstance
app.addHook('onReady',   function (done: HookHandlerDoneFunction) { done() })
app.addHook('onListen',  async function () {})
app.addHook('preClose',  async function () {})

// onClose — instance passed as first argument
app.addHook('onClose', async (instance: FastifyInstance) => {})
```

---

> **Further reading**
> - [Official Hooks docs](https://fastify.dev/docs/latest/Reference/Hooks/)
> - [Lifecycle doc](https://fastify.dev/docs/latest/Reference/Lifecycle/)
> - [lib/hooks.js](../../lib/hooks.js) — internal runner implementations
> - [test/hooks.test.js](../test/hooks.test.js) — comprehensive integration tests
