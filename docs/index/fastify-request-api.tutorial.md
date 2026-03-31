# Fastify Request API — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's `Request` object from first principles — from reading URL parts and headers
> to runtime validation, abort signals, and TypeScript generics.

---

## Table of Contents

1. [What is the Request Object?](#1-what-is-the-request-object)
2. [Request Object Anatomy](#2-request-object-anatomy)
3. [Where Request Lives in the Lifecycle](#3-where-request-lives-in-the-lifecycle)
4. [URL & Routing Properties](#4-url--routing-properties)
   - 4.1 [`url` and `originalUrl`](#41-url-and-originalurl)
   - 4.2 [`method`](#42-method)
   - 4.3 [`params`](#43-params)
   - 4.4 [`query`](#44-query)
   - 4.5 [`is404`](#45-is404)
5. [Request Payload — `body`](#5-request-payload--body)
6. [Headers](#6-headers)
   - 6.1 [Reading Headers](#61-reading-headers)
   - 6.2 [Adding Extra Headers](#62-adding-extra-headers)
   - 6.3 [Raw Headers vs Fastify Headers](#63-raw-headers-vs-fastify-headers)
7. [Network Properties](#7-network-properties)
   - 7.1 [`ip` and `ips`](#71-ip-and-ips)
   - 7.2 [`host`, `hostname`, and `port`](#72-host-hostname-and-port)
   - 7.3 [`protocol`](#73-protocol)
   - 7.4 [`socket`](#74-socket)
8. [Identity & Logging — `id` and `log`](#8-identity--logging--id-and-log)
9. [Server Reference — `server`](#9-server-reference--server)
10. [Route Metadata — `routeOptions`](#10-route-metadata--routeoptions)
11. [Abort Signal — `signal`](#11-abort-signal--signal)
12. [Runtime Validation API](#12-runtime-validation-api)
    - 12.1 [`getValidationFunction()`](#121-getvalidationfunction)
    - 12.2 [`compileValidationSchema()`](#122-compilevalidationschema)
    - 12.3 [`validateInput()`](#123-validateinput)
13. [Decorator Access — `getDecorator()` / `setDecorator()`](#13-decorator-access--getdecorator--setdecorator)
14. [Trust Proxy — Effect on Request Properties](#14-trust-proxy--effect-on-request-properties)
15. [Internal Request Flow Diagram](#15-internal-request-flow-diagram)
16. [TypeScript Quick-Reference](#16-typescript-quick-reference)
17. [Real-World Patterns](#17-real-world-patterns)
    - 17.1 [Authentication via Headers](#171-authentication-via-headers)
    - 17.2 [Pagination with Query Params](#172-pagination-with-query-params)
    - 17.3 [Cooperative Cancellation with `signal`](#173-cooperative-cancellation-with-signal)
    - 17.4 [Runtime Validation of a Dynamic Payload](#174-runtime-validation-of-a-dynamic-payload)
    - 17.5 [Forwarding the Originating IP](#175-forwarding-the-originating-ip)
18. [Common Pitfalls](#18-common-pitfalls)

Related deep dives:

- [Fastify Reply API — Complete Tutorial](./fastify-reply-api.tutorial.md)
- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Lifecycle — Complete Tutorial](./fastify-lifecycle.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)
- [Fastify Error Handling — Complete Tutorial](./fastify-error-handling.tutorial.md)

---

## 1. What is the Request Object?

Every route handler, and every lifecycle hook that participates in the request/response
cycle, receives a `request` argument as its first parameter.  `request` is Fastify's
thin, high-performance wrapper around Node's raw `http.IncomingMessage` (or
`http2.Http2ServerRequest`).

```
  Route handler(request, reply)
  |
  +-- request.raw            -> Node http.IncomingMessage
  +-- request.id             -> unique request id (string)
  +-- request.log            -> pino child logger
  +-- request.server         -> FastifyInstance (scoped)
  +-- request.method         -> 'GET' | 'POST' | ...
  +-- request.url            -> '/items?page=2'
  +-- request.params         -> { id: '42' }
  +-- request.query          -> { page: '2' }
  +-- request.body           -> parsed payload
  +-- request.headers        -> merged headers object
  +-- request.ip             -> '127.0.0.1'
  +-- request.signal         -> AbortSignal (lazy)
  +-- request.routeOptions   -> route config snapshot
```

`request` is defined in [lib/request.js](../../lib/request.js) and instantiated once per
incoming HTTP request before the first `onRequest` hook runs.  It is intentionally
**read-oriented** — most properties are getters that read from the underlying
`raw` object, keeping memory and mutation overhead minimal.

---

## 2. Request Object Anatomy

```
Request {
  // Always-set fields (set in constructor)
  id           : string | number     (unique per request)
  params       : object              (URL path parameters)
  raw          : http.IncomingMessage
  query        : object              (parsed querystring)
  log          : Logger              (pino child logger)
  body         : any                 (undefined until preParsing completes)

  // Getters delegating to raw / context
  server       : FastifyInstance
  url          : string
  originalUrl  : string
  method       : string
  is404        : boolean
  socket       : net.Socket | tls.TLSSocket
  signal       : AbortSignal         (lazy — created on first access)
  ip           : string | undefined
  ips          : string[] | undefined  (only with trustProxy)
  host         : string
  hostname     : string
  port         : number | null
  protocol     : 'http' | 'https' | undefined
  headers      : IncomingHttpHeaders  (includes additionalHeaders if set)
  routeOptions : RouteOptions         (read-only snapshot)

  // Runtime validation methods
  getValidationFunction(schemaOrHttpPart)        -> Function | undefined
  compileValidationSchema(schema, httpPart?)     -> Function
  validateInput(input, schema | httpPart, httpPart?) -> boolean

  // Decorator access (safe, throws on undeclared)
  getDecorator(name)   -> any
  setDecorator(name, value) -> void
}
```

---

## 3. Where Request Lives in the Lifecycle

```
  Incoming HTTP request
        |
        v
  +-------------------------+
  |  Request instantiated   |  <-- id, params, raw, query, log, body=undefined
  +-------------------------+
        |
        v
  [ onRequest hooks ]       <-- request.body is still undefined
        |
        v
  [ preParsing hooks ]      <-- body stream accessible via raw
        |
        v
  [ Content-Type parser ]   <-- body parsed and set on request.body
        |
        v
  [ preValidation hooks ]   <-- request.body available
        |
        v
  [ Schema validation ]     <-- params / query / headers / body validated
        |
        v
  [ preHandler hooks ]      <-- validation passed (or attachValidation=true)
        |
        v
  [ Route handler ]         <-- full request object available
        |
        v
  [ preSerialization ]      <-- request still readable for logging etc.
        |
        v
  [ onSend hooks ]
        |
        v
  [ onResponse hooks ]      <-- response complete, request.log still usable
```

> **Key point:** `request.body` is `undefined` until after the preParsing / content-type
> parsing phase.  Reading `request.body` inside an `onRequest` hook will always yield
> `undefined`.

---

## 4. URL & Routing Properties

### 4.1 `url` and `originalUrl`

```js
fastify.get('/user/:id', (request, reply) => {
  console.log(request.url)         // '/user/42?expand=profile'
  console.log(request.originalUrl) // same unless re-routed internally
})
```

`url` is a getter that reads `request.raw.url`.

`originalUrl` caches the first-seen URL at request creation time.  It becomes
different from `url` only when a plugin or middleware performs internal re-routing
that mutates `raw.url` (e.g. `fastify-express` style rewriting).

```
  Browser: GET /user/42?expand=profile
                |
                v
  raw.url = '/user/42?expand=profile'
                |
  Plugin rewrites raw.url = '/user/42'
                |
  request.url         -> '/user/42'          (current)
  request.originalUrl -> '/user/42?expand=profile'  (original, cached)
```

### 4.2 `method`

```js
console.log(request.method) // 'GET', 'POST', 'PUT', 'DELETE', 'PATCH', ...
```

Delegates to `request.raw.method`.  Always an upper-case string as normalised by
Node's HTTP parser.

### 4.3 `params`

Named URL segments captured by the router pattern.

```js
fastify.get('/files/:category/:name', (request, reply) => {
  const { category, name } = request.params
  // GET /files/images/avatar.png
  // category -> 'images'
  // name     -> 'avatar.png'
})
```

Wildcard routes produce a special `'*'` key:

```js
fastify.get('/assets/*', (request, reply) => {
  console.log(request.params['*']) // 'css/main.css'
})
```

### 4.4 `query`

The parsed querystring object.  The format depends on the
[`querystringParser`](../docs/Reference/Server.md#querystringparser) option
(defaults to Node's built-in `querystring.parse`):

```js
// GET /search?q=fastify&page=2&tag=node&tag=js
fastify.get('/search', (request, reply) => {
  console.log(request.query.q)    // 'fastify'
  console.log(request.query.page) // '2'  (string — always strings by default)
  console.log(request.query.tag)  // ['node', 'js']
})
```

To get numeric or boolean types enforce them through a JSON Schema:

```js
fastify.get('/search', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        page: { type: 'integer', default: 1 }
      }
    }
  }
}, (request, reply) => {
  console.log(request.query.page) // 2  (number after coercion)
})
```

### 4.5 `is404`

```js
fastify.setNotFoundHandler((request, reply) => {
  console.log(request.is404) // true
})

fastify.get('/home', (request, reply) => {
  console.log(request.is404) // false
})
```

`true` when the request is being handled by the 404 / not-found handler.  Useful
inside shared hooks to skip logic that should not run for unmatched routes.

---

## 5. Request Payload — `body`

`body` is set on the `Request` instance after the content-type parser runs.  It
starts as `undefined` and becomes the fully-parsed object (or string / Buffer,
depending on the parser) that was registered for the request's `Content-Type`.

```
  POST /items  Content-Type: application/json

  raw stream  ->  JSON content-type parser  ->  request.body = { name: 'pencil' }
```

```js
fastify.post('/items', async (request, reply) => {
  const { name, price } = request.body  // populated by the JSON parser
  const item = await db.items.create({ name, price })
  return reply.code(201).send(item)
})
```

**Body limits:**  Fastify enforces `bodyLimit` (default 1 MiB) globally and per-route.
Payloads that exceed the limit are rejected with `FST_ERR_CTP_BODY_TOO_LARGE` before
the body is fully buffered.

```js
fastify.post('/upload', {
  bodyLimit: 10 * 1024 * 1024  // 10 MiB for this route only
}, handler)
```

**No body routes:**  GET, HEAD, DELETE, and OPTIONS requests typically have no body.
The body remains `undefined` unless a content-type parser explicitly runs.

---

## 6. Headers

### 6.1 Reading Headers

```js
fastify.get('/protected', (request, reply) => {
  const auth  = request.headers['authorization']
  const ct    = request.headers['content-type']
  const agent = request.headers['user-agent']
})
```

`request.headers` is a getter that merges `request.raw.headers` with any
`additionalHeaders` set programmatically (see §6.2).

### 6.2 Adding Extra Headers

Assigning to `request.headers` does **not** replace the original headers.  The new
object is stored as `additionalHeaders` and merged on the next read:

```js
fastify.addHook('onRequest', async (request) => {
  request.headers = {
    'x-tenant-id': resolveTenant(request)
  }
  // request.headers now includes both raw headers AND 'x-tenant-id'
})
```

```
  request.raw.headers          additionalHeaders
  {                            {
    host: 'example.com',   +     x-tenant-id: 'acme'
    authorization: '...'       }
  }
        |___________________________|
                    |
           request.headers  (merged view)
  {
    host: 'example.com',
    authorization: '...',
    x-tenant-id: 'acme'
  }
```

### 6.3 Raw Headers vs Fastify Headers

| Property              | Contents                                         |
|-----------------------|--------------------------------------------------|
| `request.headers`     | Merged: raw headers + additionalHeaders          |
| `request.raw.headers` | Unmodified Node IncomingMessage headers only     |

> **Note:** JSON Schema header validation can mutate both `request.headers` and
> `request.raw.headers`, stripping undeclared headers if the schema uses
> `additionalProperties: false`.

---

## 7. Network Properties

### 7.1 `ip` and `ips`

Without `trustProxy`:

```js
request.ip   // socket.remoteAddress  e.g. '::ffff:127.0.0.1'
request.ips  // undefined
```

With `trustProxy: true` (or hop count / CIDR list):

```js
// X-Forwarded-For: 203.0.113.5, 10.0.0.1
request.ip   // '203.0.113.5'  (rightmost trusted entry)
request.ips  // ['203.0.113.5', '10.0.0.1']  (closest first)
```

```
  Client --(203.0.113.5)--> LB --(10.0.0.1)--> Node server

  X-Forwarded-For: 203.0.113.5, 10.0.0.1
                                      ^
                                      socket.remoteAddress (10.0.0.1 = trusted proxy)

  request.ip  -> '203.0.113.5'  (real client)
  request.ips -> ['203.0.113.5', '10.0.0.1']
```

> **Security warning:** `ip` can be spoofed if `trustProxy` is enabled but the
> actual network topology does not guarantee that every upstream host listed is a
> legitimate proxy.  Only enable `trustProxy` when you control all proxies in the
> chain or have allow-listed their CIDRs.

### 7.2 `host`, `hostname`, and `port`

```js
// Request: GET /page HTTP/1.1  Host: api.example.com:8080

request.host     // 'api.example.com:8080'
request.hostname // 'api.example.com'
request.port     // 8080

// IPv6: Host: [::1]:3000
request.host     // '[::1]:3000'
request.hostname // '[::1]'
request.port     // 3000
```

With `trustProxy`, `host` is derived from `X-Forwarded-Host` instead of the `Host`
header.

> **Security warning:** `host` / `hostname` originate from client-controlled HTTP
> headers.  Never use them for host-based auth or redirects without explicit
> allow-listing.

### 7.3 `protocol`

```js
request.protocol  // 'https' or 'http'
```

Without `trustProxy`, derived from `socket.encrypted`.
With `trustProxy`, reads the last value in `X-Forwarded-Proto`.

### 7.4 `socket`

Direct access to the underlying `net.Socket` (or `tls.TLSSocket` for HTTPS):

```js
request.socket.remoteAddress  // '127.0.0.1'
request.socket.remotePort     // 54321
request.socket.encrypted      // true for HTTPS
```

---

## 8. Identity & Logging — `id` and `log`

### `id`

Each request receives a unique ID generated by the configured
[`genReqId`](../docs/Reference/Server.md#factory-gen-req-id) function (default:
auto-incrementing integer as a string, or `X-Request-Id` header when present).

```js
console.log(request.id) // 'req-1', 'req-2', ...
```

The ID is automatically included in every log line emitted by `request.log`.

### `log`

A **pino child logger** pre-bound with `{ reqId: request.id }`.  Use it for
structured per-request logging instead of a global logger:

```js
fastify.get('/items', async (request) => {
  request.log.info({ query: request.query }, 'listing items')
  const items = await db.list()
  request.log.debug({ count: items.length }, 'items fetched')
  return items
})
```

The child logger inherits the log level and serialisers from the server-level pino
instance, plus the child bindings `{ reqId, req }` added automatically by Fastify's
`genReqId` / `requestIdLogLabel` options.

---

## 9. Server Reference — `server`

```js
request.server  // the FastifyInstance scoped to the current plugin context
```

This gives route handlers and hooks access to everything registered on the server:

```js
fastify.decorate('db', createDbClient())

fastify.get('/users', async (request) => {
  const users = await request.server.db.list()  // access decoration
  return users
})
```

`request.server` reflects the **encapsulated scope** the route belongs to — it does
not necessarily equal the root Fastify instance when used inside child plugins.

---

## 10. Route Metadata — `routeOptions`

`routeOptions` is a read-only snapshot of the route's configuration, taken at route
registration time:

```js
fastify.get('/items', {
  config: { auth: 'jwt' },
  schema: { querystring: { ... } },
  version: '2.0.0',
  bodyLimit: 512 * 1024
}, handler)

// inside handler or hook:
request.routeOptions.method           // 'GET'
request.routeOptions.url              // '/items'
request.routeOptions.bodyLimit        // 524288
request.routeOptions.handlerTimeout   // ms or undefined
request.routeOptions.config           // { auth: 'jwt' }
request.routeOptions.schema           // the full schema object
request.routeOptions.version          // '2.0.0'
request.routeOptions.attachValidation // boolean
request.routeOptions.logLevel         // 'info' | 'debug' | ...
request.routeOptions.exposeHeadRoute  // boolean
request.routeOptions.prefixTrailingSlash // 'slash' | 'no-slash' | 'both'
request.routeOptions.handler          // the actual handler function
```

A common pattern is reading `config` to drive shared hook logic:

```js
fastify.addHook('preHandler', async (request, reply) => {
  const { auth } = request.routeOptions.config
  if (auth === 'jwt') await verifyJwt(request)
  if (auth === 'api-key') await verifyApiKey(request)
})
```

---

## 11. Abort Signal — `signal`

`request.signal` returns an `AbortSignal` that is automatically aborted when:

- The client disconnects (closes the TCP connection mid-request), or
- The route's `handlerTimeout` fires (if configured).

```
  request.signal  (AbortSignal)
       |
       +-- aborted on client disconnect  -> signal.reason = AbortError
       +-- aborted on handler timeout   -> signal.reason = FST_ERR_HANDLER_TIMEOUT
```

It is **created lazily** on first access, so there is zero overhead when not used.
When `handlerTimeout` is configured the signal is pre-created eagerly.

```js
fastify.get('/slow', async (request) => {
  // pass signal to fetch — automatically cancelled if client disconnects
  const response = await fetch('https://upstream.example.com/data', {
    signal: request.signal
  })
  return response.json()
})
```

Distinguishing the abort reason:

```js
try {
  await someLongOperation({ signal: request.signal })
} catch (err) {
  if (err.name === 'AbortError') {
    if (request.signal.reason?.code === 'FST_ERR_HANDLER_TIMEOUT') {
      request.log.warn('handler timed out')
    } else {
      request.log.info('client disconnected')
    }
  }
  throw err
}
```

---

## 12. Runtime Validation API

These methods let you run schema validation **at runtime inside a handler or hook**,
beyond the automatic route-level validation that happens before `preHandler`.

### 12.1 `getValidationFunction()`

Returns a cached validation function that was already compiled for the route.

```
  getValidationFunction(httpPart: string)   -> Function | undefined
  getValidationFunction(schema: object)     -> Function | undefined

  httpPart values: 'body' | 'headers' | 'params' | 'querystring' | 'query'
```

```js
fastify.post('/items', {
  schema: { body: { type: 'object', properties: { name: { type: 'string' } } } }
}, (request) => {
  const validate = request.getValidationFunction('body')
  // validate is the compiled ajv function for the route's body schema
  validate({ name: 'pencil' })  // true
  validate({ name: 42 })        // false
  console.log(validate.errors)  // ajv errors array
})
```

Returns `undefined` when no validation function is found for the given input (e.g. the
route has no body schema defined).

### 12.2 `compileValidationSchema()`

Compiles an **arbitrary** schema and returns the validation function.  Results are
cached in a `WeakMap` keyed on the schema object reference.

```
  compileValidationSchema(schema: object, httpPart?: string) -> Function
```

```js
const addressSchema = {
  type: 'object',
  required: ['street', 'city'],
  properties: {
    street: { type: 'string' },
    city:   { type: 'string' },
    zip:    { type: 'string', pattern: '^[0-9]{5}$' }
  }
}

fastify.post('/checkout', async (request) => {
  const validateAddress = request.compileValidationSchema(addressSchema)

  if (!validateAddress(request.body.shippingAddress)) {
    return reply.code(400).send({ errors: validateAddress.errors })
  }
  // ...
})
```

### 12.3 `validateInput()`

Combines lookup + compilation + execution in one call.

```
  validateInput(input, httpPart: string)           -> boolean
  validateInput(input, schema: object)             -> boolean
  validateInput(input, schema: object, httpPart)   -> boolean
```

```js
fastify.post('/items', async (request, reply) => {
  // validate using the route's pre-compiled body schema
  const ok = request.validateInput(request.body, 'body')
  if (!ok) return reply.code(400).send({ error: 'invalid body' })
})
```

```js
const patchSchema = {
  type: 'object',
  minProperties: 1,
  properties: {
    name:  { type: 'string' },
    price: { type: 'number', minimum: 0 }
  }
}

fastify.patch('/items/:id', async (request, reply) => {
  const ok = request.validateInput(request.body, patchSchema, 'body')
  if (!ok) return reply.code(400).send({ error: 'invalid patch' })
})
```

---

## 13. Decorator Access — `getDecorator()` / `setDecorator()`

Fastify lets plugins decorate the `Request` prototype.  After decoration, the value
is accessible as a direct property (`request.myProp`).  The `getDecorator` /
`setDecorator` helpers add a declaration-safety check: they throw
`FST_ERR_DEC_UNDECLARED` if the name was never registered with
`fastify.decorateRequest()`.

```js
// Plugin: registers the decoration
fastify.decorateRequest('user', null)

fastify.addHook('preHandler', async (request) => {
  request.user = await loadUser(request.headers.authorization)
  // equivalent safe write:
  request.setDecorator('user', await loadUser(request.headers.authorization))
})

fastify.get('/profile', (request, reply) => {
  const user = request.getDecorator('user')  // throws if 'user' was not declared
  return user
})
```

```
  fastify.decorateRequest('foo', defaultValue)
        |
        v
  Request.prototype.foo = defaultValue   (shared default)
        |
        v
  request.foo = newValue  (per-instance override, on first write)
  request.getDecorator('foo')  -> newValue   (safe, throws if undeclared)
```

> **Note:** For function decorations, `getDecorator` returns the function
> **already bound** to the current `request` instance, so you can call it without
> losing context.

---

## 14. Trust Proxy — Effect on Request Properties

When the `trustProxy` server option is set, Fastify installs overriding property
descriptors on the `Request` prototype that read from proxy-injected headers instead
of the raw socket:

```
  trustProxy setting          ip / ips source
  -------------------         -----------------------
  false (default)             socket.remoteAddress
  true                        proxyAddr.all() (all hops, trusts everything)
  number N                    trusts N proxy hops
  'comma,list,of,cidrs'       trusts listed CIDRs via @fastify/proxy-addr
  function(addr, i)           custom trust function

  property      without trustProxy    with trustProxy
  ----------    ------------------    ---------------------------------
  ip            socket.remoteAddress  rightmost trusted hop in X-Forwarded-For
  ips           undefined             all hops array (closest first)
  host          Host header           X-Forwarded-Host (last value)
  protocol      socket.encrypted      X-Forwarded-Proto (last value)
```

---

## 15. Internal Request Flow Diagram

```
  fastify.listen()
        |
        v
  HTTP server 'request' event
        |
        v
  +-----------------------------------------+
  |  new Request(id, params, raw, query,    |
  |              log, context)              |
  +-----------------------------------------+
        |
        v
  runHooks('onRequest', request, reply)
        |
        v
  runHooks('preParsing', request, reply)
        |
        v
  content-type parser (sets request.body)
        |
        v
  runHooks('preValidation', request, reply)
        |
        v
  schema validation (params/query/headers/body)
        |        |
        |        +-- validation error -> errorHandler
        v
  runHooks('preHandler', request, reply)
        |
        v
  route handler(request, reply)
        |
        v
  (reply.send() called)
        |
        v
  runHooks('preSerialization', request, reply, payload)
        |
        v
  runHooks('onSend', request, reply, serializedPayload)
        |
        v
  raw response written to socket
        |
        v
  runHooks('onResponse', request, reply)
```

---

## 16. TypeScript Quick-Reference

```ts
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

// Typed querystring + params + body
type QueryParams = { page: number; limit: number }
type RouteParams = { id: string }
type CreateBody  = { name: string; price: number }

fastify.get<{
  Querystring: QueryParams
  Params: RouteParams
}>('/items/:id', async (
  request: FastifyRequest<{ Querystring: QueryParams; Params: RouteParams }>,
  reply: FastifyReply
) => {
  const id    = request.params.id         // string
  const page  = request.query.page        // number
  const limit = request.query.limit       // number
  return { id, page, limit }
})

// Typed decoration
declare module 'fastify' {
  interface FastifyRequest {
    user: { id: string; role: string } | null
  }
}

fastify.decorateRequest('user', null)

fastify.addHook('preHandler', async (request: FastifyRequest) => {
  request.user = await authenticate(request.headers.authorization ?? '')
})
```

Generic slots available on `FastifyRequest<RouteGeneric>`:

| Slot          | Maps to           | Default |
|---------------|-------------------|---------|
| `Body`        | `request.body`    | unknown |
| `Querystring` | `request.query`   | unknown |
| `Params`      | `request.params`  | unknown |
| `Headers`     | `request.headers` | unknown |

---

## 17. Real-World Patterns

### 17.1 Authentication via Headers

```js
fastify.decorateRequest('user', null)

fastify.addHook('preHandler', async (request, reply) => {
  const { auth } = request.routeOptions.config
  if (!auth) return  // public route, skip

  const token = request.headers['authorization']?.replace('Bearer ', '')
  if (!token) return reply.code(401).send({ error: 'missing token' })

  try {
    request.user = await verifyJwt(token)
  } catch {
    return reply.code(401).send({ error: 'invalid token' })
  }
})

fastify.get('/profile', { config: { auth: 'jwt' } }, async (request) => {
  return request.user
})
```

### 17.2 Pagination with Query Params

```js
fastify.get('/items', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        page:  { type: 'integer', minimum: 1, default: 1 },
        limit: { type: 'integer', minimum: 1, maximum: 100, default: 20 }
      }
    }
  }
}, async (request) => {
  const { page, limit } = request.query
  const offset = (page - 1) * limit

  const items = await db.items.findMany({ skip: offset, take: limit })
  return { page, limit, items }
})
```

### 17.3 Cooperative Cancellation with `signal`

```js
fastify.get('/report', { config: { handlerTimeout: 10_000 } }, async (request) => {
  const [users, orders] = await Promise.all([
    db.users.findAll({ signal: request.signal }),
    db.orders.findAll({ signal: request.signal })
  ])

  if (request.signal.aborted) {
    request.log.warn('report generation cancelled')
    return
  }

  return buildReport(users, orders)
})
```

### 17.4 Runtime Validation of a Dynamic Payload

```js
const schemas = {
  circle:    { type: 'object', required: ['radius'],        properties: { radius: { type: 'number' } } },
  rectangle: { type: 'object', required: ['width', 'height'], properties: { width: { type: 'number' }, height: { type: 'number' } } }
}

fastify.post('/shapes', async (request, reply) => {
  const { type } = request.body
  const schema = schemas[type]

  if (!schema) return reply.code(400).send({ error: 'unknown shape type' })

  const ok = request.validateInput(request.body, schema)
  if (!ok) {
    const validate = request.compileValidationSchema(schema)
    validate(request.body)                        // re-run to capture errors
    return reply.code(400).send({ errors: validate.errors })
  }

  const area = type === 'circle'
    ? Math.PI * request.body.radius ** 2
    : request.body.width * request.body.height

  return { type, area }
})
```

### 17.5 Forwarding the Originating IP

```js
const fastify = Fastify({ trustProxy: true })

fastify.addHook('onRequest', async (request) => {
  // Log the real client IP without logging the proxy IPs
  request.log.info({ clientIp: request.ip }, 'incoming request')
})

fastify.post('/webhook', async (request, reply) => {
  const allowed = ['203.0.113.0/24']
  if (!isInRange(request.ip, allowed)) {
    return reply.code(403).send({ error: 'forbidden' })
  }
  // process webhook...
})
```

---

## 18. Common Pitfalls

| Pitfall | Explanation | Fix |
|---------|-------------|-----|
| Reading `request.body` in `onRequest` | Body is `undefined` until after preParsing / content-type parser | Move the logic to `preHandler` or the route handler |
| Mutating `request.headers` with assignment overwrites | Assignment stores `additionalHeaders`, not a full replace | Use `Object.assign` pattern or read merged result via the `headers` getter |
| Trusting `host` / `hostname` without allow-listing | These come from client-controlled headers | Allow-list known hosts or use your own domain config |
| Enabling `trustProxy: true` on a public server | All `X-Forwarded-For` values are trusted, allowing IP spoofing | Use CIDR allow-list or hop count matching the exact topology |
| Calling `compileValidationSchema` with a new literal each time | A new object literal on each request means the WeakMap cache is never hit | Hoist schema objects to module scope so reference equality is preserved |
| Accessing `request.signal` unnecessarily | Creates an `AbortController` on first access (minor overhead per request) | Only read `request.signal` when you actually need cooperative cancellation |
| Using `request.context.config` | `context` is deprecated | Use `request.routeOptions.config` instead |
