# Fastify Routing & Constraints — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's routing system and constraint mechanism from first principles.

---

## Table of Contents

1. [What is Routing in Fastify?](#1-what-is-routing-in-fastify)
2. [Full Route Declaration](#2-full-route-declaration)
3. [Shorthand Declaration](#3-shorthand-declaration)
4. [URL Patterns](#4-url-patterns)
   - 4.1 [Static Routes](#41-static-routes)
   - 4.2 [Parametric Routes](#42-parametric-routes)
   - 4.3 [Wildcard Routes](#43-wildcard-routes)
   - 4.4 [RegExp Parameters](#44-regexp-parameters)
   - 4.5 [Multi-Parameter Segments](#45-multi-parameter-segments)
   - 4.6 [Optional Parameters](#46-optional-parameters)
   - 4.7 [Route Matching Priority](#47-route-matching-priority)
5. [Route Options Reference](#5-route-options-reference)
6. [Async / Await Handlers](#6-async--await-handlers)
7. [Route Prefixing](#7-route-prefixing)
   - 7.1 [Basic Prefixing](#71-basic-prefixing)
   - 7.2 [prefixTrailingSlash Behaviour](#72-prefixtrailingslash-behaviour)
   - 7.3 [Prefixing and fastify-plugin](#73-prefixing-and-fastify-plugin)
8. [Constraints — Overview](#8-constraints--overview)
9. [Version Constraints](#9-version-constraints)
   - 9.1 [Declaring Versioned Routes](#91-declaring-versioned-routes)
   - 9.2 [Matching Logic](#92-matching-logic)
   - 9.3 [Cache Poisoning Protection](#93-cache-poisoning-protection)
10. [Host Constraints](#10-host-constraints)
    - 10.1 [Exact-Match Host](#101-exact-match-host)
    - 10.2 [RegExp Host](#102-regexp-host)
11. [Custom Constraint Strategies](#11-custom-constraint-strategies)
    - 11.1 [Synchronous Custom Constraint](#111-synchronous-custom-constraint)
    - 11.2 [Asynchronous Custom Constraint](#112-asynchronous-custom-constraint)
    - 11.3 [mustMatchWhenDerived](#113-mustmatchwwhenDerived)
12. [Under the Hood — find-my-way](#12-under-the-hood--find-my-way)
    - 12.1 [Router Internals](#121-router-internals)
    - 12.2 [Constraint Storage Model](#122-constraint-storage-model)
13. [Route Config Object](#13-route-config-object)
14. [Per-Route Overrides](#14-per-route-overrides)
15. [printRoutes & hasRoute & findRoute](#15-printroutes--hasroute--findroute)
16. [Performance Tips](#16-performance-tips)
17. [Common Pitfalls](#17-common-pitfalls)
18. [TypeScript Quick-Reference](#18-typescript-quick-reference)

Related deep dives:

- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Plugins — Complete Tutorial](./fastify-plugins.tutorial.md)
- [Fastify Encapsulation — Complete Tutorial](./fastify-encapsulation.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)

---

## 1. What is Routing in Fastify?

Routing is the process of mapping an incoming HTTP request to the correct handler
function.  Fastify delegates all URL-matching work to
[find-my-way](https://github.com/delvedor/find-my-way), a radix-tree router.

```
HTTP Request
    |
    v
find-my-way radix tree
    |
    |-- method match? ----NO----> 404 / 405
    |
    |-- url match?    ----NO----> 404
    |
    |-- constraints match? --NO-> 404
    |
    v
  handler (Context)
```

Key design decisions:

- Routes are **immutable after `fastify.listen()` / `fastify.ready()`** — you
  cannot add routes at runtime.
- Static segments are always resolved before parametric ones, and parametric
  before wildcard.
- Constraints allow **multiple handlers on the same method+URL** pair, selected
  by request properties (version header, host header, or any custom value).

---

## 2. Full Route Declaration

`fastify.route(options)` is the lowest-level API.  Every shorthand method
(`fastify.get`, `fastify.post`, …) eventually calls it.

```js
fastify.route({
  method: 'GET',        // string or array of strings
  url: '/items/:id',   // alias: path
  schema: {
    params:   { type: 'object', properties: { id: { type: 'integer' } } },
    response: { 200: { type: 'object', properties: { id: { type: 'integer' }, name: { type: 'string' } } } }
  },
  handler: async function (request, reply) {
    return { id: Number(request.params.id), name: 'widget' }
  }
})
```

The `handler` is the only required field (beyond `method` and `url`).  All
other fields are optional overrides or additions.

---

## 3. Shorthand Declaration

Express-style shorthand is available for all standard HTTP methods:

```
fastify.get(url, [options], handler)
fastify.post(url, [options], handler)
fastify.put(url, [options], handler)
fastify.patch(url, [options], handler)
fastify.delete(url, [options], handler)
fastify.head(url, [options], handler)
fastify.options(url, [options], handler)
fastify.all(url, [options], handler)   // registers handler for every method
```

The handler can also live inside `options`:

```js
fastify.get('/ping', {
  schema: { response: { 200: { type: 'string' } } },
  handler: (req, reply) => reply.send('pong')
})
```

> Providing the handler in **both** `options.handler` and as the third argument
> throws `FST_ERR_ROUTE_DUPLICATED_HANDLER`.

---

## 4. URL Patterns

### 4.1 Static Routes

Fixed paths — fastest to match because find-my-way resolves them in O(1) via
the radix-tree without capture groups.

```js
fastify.get('/healthz', handler)
fastify.get('/api/v1/status', handler)
```

### 4.2 Parametric Routes

Named segments prefixed with `:` capture a URL fragment into `request.params`.

```js
fastify.get('/users/:userId', async (req) => {
  // GET /users/42  -->  req.params.userId === '42'
  return { userId: req.params.userId }
})

fastify.get('/users/:userId/orders/:orderId', async (req) => {
  // req.params === { userId: '7', orderId: '99' }
})
```

To include a literal colon in the path use `::`:

```js
fastify.post('/name::verb')  // matches /name:verb
```

### 4.3 Wildcard Routes

A bare `*` captures everything that follows, available as
`request.params['*']`.

```js
fastify.get('/static/*', (req, reply) => {
  // GET /static/css/main.css --> req.params['*'] === 'css/main.css'
})
```

### 4.4 RegExp Parameters

Append `(pattern)` after the parameter name to constrain it with a regular
expression.  **Backslashes must be doubled** in JS strings.

```js
fastify.get('/files/:name(^[a-z]+)\\.log', handler)
// matches /files/access.log  but not /files/123.log

fastify.get('/images/:id(^\\d+).png', handler)
// req.params.id is a string of digits
```

> RegExp matching is significantly slower than plain parametric matching.
> Avoid it on high-traffic routes.

### 4.5 Multi-Parameter Segments

Multiple parameters can share one path segment if they are separated by a
non-`:` delimiter (commonly `-`):

```js
fastify.get('/map/:lat-:lng', (req) => {
  // /map/52N-13E --> { lat: '52N', lng: '13E' }
})

fastify.get('/time/:hh(^\\d{2})h:mm(^\\d{2})m', (req) => {
  // /time/08h30m --> { hh: '08', mm: '30' }
})
```

### 4.6 Optional Parameters

Append `?` to the **last** parameter name to make it optional:

```js
fastify.get('/posts/:slug?', (req) => {
  // /posts        --> req.params.slug === undefined
  // /posts/hello  --> req.params.slug === 'hello'
})
```

### 4.7 Route Matching Priority

find-my-way resolves routes in this fixed order:

```
Priority (highest to lowest)
----------------------------
1.  Static         /users/me
2.  Parametric     /users/:id
3.  RegExp param   /users/:id(^\\d+)
4.  Wildcard       /users/*
```

When the same path could match multiple entries, only the highest-priority one
is selected.  Declare more specific routes before generic ones — Fastify
registers them in insertion order.

---

## 5. Route Options Reference

| Option | Type | Description |
|--------|------|-------------|
| `method` | `string \| string[]` | HTTP method(s). Custom methods require `addHttpMethod`. |
| `url` / `path` | `string` | URL pattern. |
| `handler` | `Function` | Request handler (required). |
| `schema` | `object` | AJV schema for body, querystring, params, headers, response. |
| `constraints` | `object` | Constraint key-value pairs (version, host, or custom). |
| `config` | `object` | Arbitrary data available via `reply.routeOptions.config`. |
| `attachValidation` | `boolean` | Attach schema errors to `request.validationError` instead of throwing. |
| `bodyLimit` | `integer` | Max body bytes. Overrides server-level default (1 MiB). |
| `handlerTimeout` | `integer` | Max ms for the full route lifecycle. Overrides server-level setting. |
| `logLevel` | `string` | Pino log level for this route only. |
| `logSerializers` | `object` | Pino serializer overrides for this route. |
| `exposeHeadRoute` | `boolean` | Auto-create a `HEAD` sibling for `GET` routes. |
| `prefixTrailingSlash` | `'both' \| 'slash' \| 'no-slash'` | How `/` is treated inside prefixed plugins. |
| `errorHandler` | `Function` | Route-scoped error handler. |
| `validatorCompiler` | `Function` | Route-scoped validation compiler. |
| `serializerCompiler` | `Function` | Route-scoped serialization compiler. |
| `schemaErrorFormatter` | `Function` | Route-scoped schema error formatter. |
| `childLoggerFactory` | `Function` | Route-scoped logger factory. |
| `onRequest` … `onResponse` | `Function \| Function[]` | Route-level lifecycle hooks. |

---

## 6. Async / Await Handlers

Return a value and Fastify calls `reply.send` for you:

```js
fastify.get('/users/:id', async (request, reply) => {
  const user = await db.findUser(request.params.id)
  return user  // serialized automatically
})
```

When you call `reply.send()` inside an async handler, **return `reply`** or
**`await reply`** to avoid a race condition between the explicit send and the
automatic resolution:

```js
fastify.get('/edge', async (request, reply) => {
  setImmediate(() => reply.send({ ok: true }))
  return reply   // waits for the setImmediate callback
})
```

Rules:

```
+------------------------------+------------------------------------+
| Style                        | Guideline                          |
+------------------------------+------------------------------------+
| async, return value          | never call reply.send              |
| async, reply.send inside     | return reply  OR  await reply      |
| callback (non-async)         | always call reply.send             |
+------------------------------+------------------------------------+
```

---

## 7. Route Prefixing

### 7.1 Basic Prefixing

Pass `{ prefix: '/v1' }` to `register` to prepend a path segment to all routes
defined inside that plugin:

```js
// server.js
fastify.register(require('./routes/v1/users'), { prefix: '/v1' })
fastify.register(require('./routes/v2/users'), { prefix: '/v2' })
```

```js
// routes/v1/users.js
module.exports = function (fastify, opts, done) {
  fastify.get('/users', handlerV1)  // registered as GET /v1/users
  done()
}
```

Resulting route tree:

```
GET /v1/users  --> handlerV1
GET /v2/users  --> handlerV2
```

Prefixes stack across nested `register` calls:

```js
fastify.register(function outerPlugin(app, opts, done) {
  // prefix so far: /api

  app.register(function innerPlugin(app, opts, done) {
    app.get('/items', handler)  // GET /api/v1/items
    done()
  }, { prefix: '/v1' })

  done()
}, { prefix: '/api' })
```

Route parameters inside prefixes are fully supported:

```js
fastify.register(userRoutes, { prefix: '/users/:userId' })
// every route inside userRoutes can access request.params.userId
```

### 7.2 prefixTrailingSlash Behaviour

When a route is declared as `/` inside a prefixed plugin, Fastify must decide
which URLs to register.  The `prefixTrailingSlash` route option controls this:

```
prefix = '/items'

prefixTrailingSlash | Registered URLs
--------------------|---------------------------
'both' (default)    | /items  AND  /items/
'slash'             | /items/
'no-slash'          | /items
```

```js
fastify.register(function (app, opts, done) {
  app.get('/', { prefixTrailingSlash: 'no-slash' }, handler)
  // only /items — never /items/
  done()
}, { prefix: '/items' })
```

### 7.3 Prefixing and fastify-plugin

`fastify-plugin` breaks encapsulation so that decorators and hooks leak out of
the scope — but it also **removes the prefix**.  Wrap the routes in an inner
plugin to regain prefixing:

```js
const fp = require('fastify-plugin')
const routes = require('./lib/routes')

module.exports = fp(async function (app, opts) {
  // fp wraps this plugin, so the prefix on register() below still works
  app.register(routes, { prefix: '/v1' })
}, { name: 'my-routes' })
```

---

## 8. Constraints — Overview

Constraints let multiple handlers occupy the **same method + URL** slot.
When a request arrives, find-my-way first matches method and URL, then runs
each active constraint strategy to find the single best handler.

```
GET /   (no constraints)     --> handler A
GET /   version: '2.0.0'     --> handler B
GET /   host: 'api.example'  --> handler C
GET /   version: '2.0.0'     --> handler D
        host: 'api.example'
```

All four handlers live in the router simultaneously.  A request for
`GET /` with `Accept-Version: 2.x` and `Host: api.example` matches D.

Built-in strategies provided by find-my-way:

```
Strategy   | Header read          | registered as
-----------|----------------------|------------------
version    | Accept-Version       | constraints.version
host       | Host                 | constraints.host
```

Custom strategies hook into the same mechanism and can read any part of the
request object.

---

## 9. Version Constraints

### 9.1 Declaring Versioned Routes

Pass `constraints: { version: 'x.y.z' }` with a full semver string:

```js
fastify.route({
  method: 'GET',
  url: '/api/resource',
  constraints: { version: '1.0.0' },
  handler: (req, reply) => reply.send({ v: 1 })
})

fastify.route({
  method: 'GET',
  url: '/api/resource',
  constraints: { version: '2.0.0' },
  handler: (req, reply) => reply.send({ v: 2 })
})
```

The shorthand equivalent:

```js
fastify.get('/api/resource', { constraints: { version: '1.0.0' } }, handler1)
fastify.get('/api/resource', { constraints: { version: '2.0.0' } }, handler2)
```

### 9.2 Matching Logic

The client sends `Accept-Version` with a semver range.  find-my-way picks the
**highest registered version that satisfies** the requested range:

```
Registered: 1.0.0, 1.2.0, 2.0.0

Accept-Version: 1.x   --> matches 1.2.0  (highest in major 1)
Accept-Version: 1.2.x --> matches 1.2.0
Accept-Version: 2.0.0 --> matches 2.0.0
Accept-Version: 3.x   --> 404
(no header)           --> 404  (versioned route requires the header)
```

A non-versioned handler for the same URL acts as a fallback when no version
header is sent (if it exists).

### 9.3 Cache Poisoning Protection

Because the response content varies by `Accept-Version`, intermediate caches
must see the header in the `Vary` response header.  Add a global `onSend` hook:

```js
const vary = require('vary')

fastify.addHook('onSend', (req, reply, payload, done) => {
  if (req.headers['accept-version']) {
    const current = reply.getHeader('Vary') || ''
    const header  = Array.isArray(current) ? current.join(', ') : String(current)
    const updated = vary.append(header, 'Accept-Version')
    if (updated) reply.header('Vary', updated)
  }
  done()
})
```

---

## 10. Host Constraints

### 10.1 Exact-Match Host

Restrict a route to a specific value of the `Host` request header:

```js
fastify.route({
  method: 'GET',
  url: '/',
  constraints: { host: 'admin.example.com' },
  handler: (req, reply) => reply.send('admin panel')
})

fastify.route({
  method: 'GET',
  url: '/',
  constraints: { host: 'www.example.com' },
  handler: (req, reply) => reply.send('public site')
})
```

A request with `Host: other.example.com` gets a 404 because neither handler
matches.

### 10.2 RegExp Host

Pass a `RegExp` to match a pattern of host names:

```js
fastify.route({
  method: 'GET',
  url: '/data',
  constraints: { host: /^tenant-[a-z]+\.example\.com$/ },
  handler: async (req, reply) => {
    const tenant = req.headers.host.split('.')[0]
    return { tenant }
  }
})
// matches tenant-acme.example.com, tenant-beta.example.com, etc.
```

> Host strings are compared against `request.headers.host` (which includes the
> port when non-standard, e.g. `api.example.com:8080`).  Ensure your pattern
> includes the port when running behind a non-standard port without a proxy.

---

## 11. Custom Constraint Strategies

### 11.1 Synchronous Custom Constraint

A constraint strategy is a plain object with four required fields:

```js
const tenantConstraint = {
  // (1) unique name — used as the key in route `constraints` objects
  name: 'tenant',

  // (2) storage factory — called once per route tree node
  //     returns a tiny map used by the router to store handlers per value
  storage () {
    const handlers = {}
    return {
      get (value)        { return handlers[value] ?? null },
      set (value, store) { handlers[value] = store }
    }
  },

  // (3) derive the constraint value from each incoming request
  //     sync version: just return the value
  deriveConstraint (req, ctx) {
    return req.headers['x-tenant-id'] ?? null
  },

  // (4) optional — when true, routes without this constraint will NOT match
  //     requests that carry a value for it
  mustMatchWhenDerived: true
}

// Register before fastify.listen()
fastify.addConstraintStrategy(tenantConstraint)
```

Use the strategy in a route by adding its `name` as a key in `constraints`:

```js
fastify.get('/settings', { constraints: { tenant: 'acme' } }, acmeHandler)
fastify.get('/settings', { constraints: { tenant: 'beta' } }, betaHandler)
```

Constraints can be combined:

```js
fastify.get('/', {
  constraints: {
    version: '2.0.0',
    host:    'api.example.com',
    tenant:  'acme'
  }
}, handler)
```

### 11.2 Asynchronous Custom Constraint

When the constraint value must be fetched from an external source (e.g., a
database or a cache), use the callback form of `deriveConstraint`:

```js
const dbTenantConstraint = {
  name: 'dbTenant',
  storage () {
    const handlers = {}
    return {
      get (v)    { return handlers[v] ?? null },
      set (v, s) { handlers[v] = s }
    }
  },
  // callback signature: (req, ctx, done)
  deriveConstraint (req, ctx, done) {
    db.getTenantByApiKey(req.headers['x-api-key'], (err, tenant) => {
      if (err) return done(err)
      done(null, tenant.id)
    })
  },
  mustMatchWhenDerived: true
}

fastify.addConstraintStrategy(dbTenantConstraint)
```

> Async constraints cause Fastify to use an internal async request-handling
> path which is slightly slower.  Prefer sync constraints whenever possible.

When an async constraint callback returns an error, provide a `frameworkErrors`
handler on the server to catch `FST_ERR_ASYNC_CONSTRAINT`:

```js
const fastify = require('fastify')({
  frameworkErrors (err, req, res) {
    const { FST_ERR_ASYNC_CONSTRAINT } = require('fastify').errorCodes
    if (err instanceof FST_ERR_ASYNC_CONSTRAINT) {
      res.statusCode = 400
      return res.end('Bad constraint value')
    }
    res.statusCode = 500
    res.end('Internal Server Error')
  }
})
```

### 11.3 mustMatchWhenDerived

`mustMatchWhenDerived` controls whether **unconstrained** routes are still
eligible when the constraint produces a value:

```
mustMatchWhenDerived = false  (default)
  A request with x-tenant-id: acme
  can still match a route that has no tenant constraint.

mustMatchWhenDerived = true
  A request with x-tenant-id: acme
  MUST match a route with tenant: 'acme'.
  Routes without tenant constraints are skipped.
```

Use `true` when the constraint is security-sensitive (e.g., per-tenant routing)
and you never want a tenant request falling through to a global handler.

---

## 12. Under the Hood — find-my-way

### 12.1 Router Internals

Fastify creates a single find-my-way instance in [lib/route.js](../../lib/route.js):

```js
const FindMyWay = require('find-my-way')
const router = FindMyWay(options)
```

Router-level options that Fastify forwards from its own config:

```
routerKeys = [
  'allowUnsafeRegex',
  'buildPrettyMeta',
  'caseSensitive',       // default: true
  'constraints',
  'defaultRoute',
  'ignoreDuplicateSlashes',
  'ignoreTrailingSlash',
  'maxParamLength',
  'onBadUrl',
  'querystringParser',
  'useSemicolonDelimiter'
]
```

When a request arrives, Fastify calls `router.lookup(req, res)` (exposed as
`routing`) which walks the radix tree to locate the correct `routeHandler` and
`Context` object.

### 12.2 Constraint Storage Model

find-my-way stores routes as nodes in a radix tree.  Each node holds a
`Constrainer` that manages multiple constraint strategies:

```
Radix tree node  (method=GET, path=/api/resource)
  |
  +-- Constrainer
        |
        +-- version strategy  { '1.0.0': handlerA, '2.0.0': handlerB }
        |
        +-- host    strategy  { 'admin.example.com': handlerC }
        |
        +-- tenant  strategy  { 'acme': handlerD }
```

On each request the constrainer iterates active strategies, derives values, and
intersects the sets of matching handlers until exactly one remains (or 404).

The `addConstraintStrategy` / `hasConstraintStrategy` helpers on the Fastify
instance delegate directly to `router.addConstraintStrategy` and
`router.hasConstraintStrategy` — they throw if called after the server has
started.

---

## 13. Route Config Object

`config` is a free-form object stored on the route context.  Read it at request
time via `reply.routeOptions.config`:

```js
function handler (req, reply) {
  const { featureFlag, rateLimitTier } = reply.routeOptions.config
  // ...
  reply.send({})
}

fastify.get('/search', {
  config: { featureFlag: 'newSearch', rateLimitTier: 'premium' }
}, handler)
```

`config` is also automatically populated with `url` and `method` by Fastify, so
you do not need to repeat those.

---

## 14. Per-Route Overrides

Several server-level behaviours can be overridden on individual routes:

```js
fastify.post('/upload', {
  bodyLimit:      10 * 1024 * 1024,  // 10 MiB for this route only
  handlerTimeout: 30_000,             // 30 s timeout
  logLevel:       'warn',
  errorHandler (err, req, reply) {
    reply.status(422).send({ error: err.message })
  },
  async handler (req, reply) {
    // ...
  }
})
```

Override resolution order (most specific wins):

```
route-level option
  --> plugin-level setErrorHandler / setChildLoggerFactory / …
    --> server-level default
```

---

## 15. printRoutes & hasRoute & findRoute

**`fastify.printRoutes()`** returns a visual ASCII tree of all registered routes
(delegates to find-my-way's `prettyPrint`):

```js
await fastify.ready()
console.log(fastify.printRoutes())
// └── /
//     ├── api/
//     │   ├── v1/users (GET)
//     │   └── v2/users (GET)
//     └── healthz (GET)
```

Pass `{ commonPrefix: false }` to print each full path on its own line.
Pass `{ includeHooks: true }` to include hook names beside each route.

**`fastify.hasRoute(options)`** returns `true` if a matching route exists:

```js
fastify.hasRoute({ method: 'GET', url: '/users/:id' })           // true / false
fastify.hasRoute({ method: 'GET', url: '/', constraints: { host: 'admin.example.com' } })
```

**`fastify.findRoute(options)`** returns the handler and matched params, or
`null`:

```js
const result = fastify.findRoute({ method: 'GET', url: '/users/42' })
// result === { handler: fn, params: { id: '42' }, searchParams: URLSearchParams {} }
// or null if no match
```

> `findRoute` exposes a reduced surface — it intentionally hides the internal
> `Context` and server objects to prevent runtime route mutation.

---

## 16. Performance Tips

1. **Prefer static over parametric.**  Static segments are resolved in O(1) by
   the radix tree.  Even a single static prefix before `:param` speeds things up.

2. **Avoid RegExp parameters on hot paths.**  They bypass the radix tree
   optimisation and fall back to linear matching.

3. **Minimise the number of parameters.**  Every extra capture group adds
   overhead.  If only one ID is needed, use `/:id` not `/:type/:id`.

4. **Avoid `fastify.all()` blindly.**  It registers a handler for every HTTP
   method, inflating the tree for methods you may not use.

5. **Use `ignoreTrailingSlash: true` at the server level** instead of
   registering both `/path` and `/path/`:

   ```js
   const fastify = require('fastify')({ ignoreTrailingSlash: true })
   ```

6. **Version and host constraints degrade performance** for the matched
   sub-tree.  Use them deliberately, not by default.

7. **Async constraints are slower than sync constraints.**  Only use the async
   form when the source of truth cannot be computed in-process.

8. **Response schemas improve throughput by 10–20 %** — Fastify uses
   fast-json-stringify to bypass `JSON.stringify`.  Always provide a response
   schema for hot routes.

---

## 17. Common Pitfalls

**Registering routes after `fastify.listen()`**

Fastify seals the router on startup.  Calling `fastify.get(...)` after
`await fastify.listen(...)` throws an error.

**Double-slash from prefix + leading slash**

```js
// prefix ends with /  AND  route starts with /  => double slash stripped
fastify.register(plugin, { prefix: '/api/' })
// inside plugin:
fastify.get('/users', h)  // becomes /api/users  (not /api//users)
```

Fastify handles this automatically, but it can be surprising — check
`fastify.printRoutes()` when URLs are unexpected.

**Duplicate routes**

```js
fastify.get('/users/:id', h1)
fastify.get('/users/:id', h2)  // throws FST_ERR_DUPLICATED_ROUTE
```

The same method+url without different constraints is always a duplicate.

**fastify-plugin removes the prefix**

```js
module.exports = fp(function plugin(app, opts, done) {
  app.get('/hello', h)  // ignores prefix — fp breaks encapsulation
  done()
})
```

Wrap the route registration in a nested `register` to re-apply the prefix.

**Version header required when any versioned handler exists**

If you register even one versioned handler at a given URL, requests without
`Accept-Version` that reach that URL return 404, not the un-versioned handler
(unless you also register one explicitly without constraints).

---

## 18. TypeScript Quick-Reference

```typescript
import Fastify, {
  FastifyInstance,
  FastifyRequest,
  FastifyReply,
  RouteShorthandOptions,
  RouteGenericInterface,
  FastifySchema,
  ConstraintStrategy,
} from 'fastify'

const app: FastifyInstance = Fastify({ logger: true })

// --- typed params / body / querystring ---

interface GetUserParams {
  userId: string
}

interface GetUserQuery {
  expand?: string
}

app.get<{ Params: GetUserParams; Querystring: GetUserQuery }>(
  '/users/:userId',
  async (request, reply) => {
    const { userId } = request.params   // string
    const { expand } = request.query    // string | undefined
    return { userId, expand }
  }
)

// --- full route declaration ---

interface CreateItemBody {
  name: string
  price: number
}

const schema: FastifySchema = {
  body: {
    type: 'object',
    required: ['name', 'price'],
    properties: {
      name:  { type: 'string' },
      price: { type: 'number' }
    }
  },
  response: {
    201: {
      type: 'object',
      properties: { id: { type: 'string' } }
    }
  }
}

app.route<{ Body: CreateItemBody }>({
  method: 'POST',
  url: '/items',
  schema,
  handler: async (req, reply) => {
    const { name, price } = req.body   // typed
    reply.status(201).send({ id: 'abc123' })
  }
})

// --- typed custom constraint strategy ---

const tenantStrategy: ConstraintStrategy<string> = {
  name: 'tenant',
  storage () {
    const map = new Map<string, unknown>()
    return {
      get (v)    { return map.get(v) ?? null },
      set (v, s) { map.set(v, s) }
    }
  },
  deriveConstraint (req) {
    return (req.headers as Record<string, string>)['x-tenant-id'] ?? null
  },
  mustMatchWhenDerived: true
}

app.addConstraintStrategy(tenantStrategy)

app.get(
  '/dashboard',
  { constraints: { tenant: 'acme' } },
  async (req, reply) => ({ tenant: 'acme' })
)

// --- prefix via register ---

app.register(async function v1Routes (app) {
  app.get('/users', async () => [])
  app.post('/users', async (req: FastifyRequest<{ Body: CreateItemBody }>, reply) => {
    reply.status(201).send({ id: 'new' })
  })
}, { prefix: '/v1' })

await app.listen({ port: 3000 })
```
