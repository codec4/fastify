# Fastify Plugins — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's plugin system from first principles.

---

## Table of Contents

1. [What is a Plugin?](#1-what-is-a-plugin)
2. [Plugin Anatomy](#2-plugin-anatomy)
3. [Encapsulation — The Core Concept](#3-encapsulation--the-core-concept)
4. [fastify-plugin — Breaking Encapsulation](#4-fastify-plugin--breaking-encapsulation)
5. [Plugin Options](#5-plugin-options)
6. [Route Prefixes](#6-route-prefixes)
7. [Decorators](#7-decorators)
  - 7.1 [decorate](#71-decorate)
   - 7.2 [decorateRequest](#72-decoraterequest)
   - 7.3 [decorateReply](#73-decoratereply)
   - 7.4 [hasDecorator Guards](#74-hasdecorator-guards)
   - 7.5 [Decorator Dependencies](#75-decorator-dependencies)
8. [Plugin Metadata & fastify-plugin](#8-plugin-metadata--fastify-plugin)
   - 8.1 [Name](#81-name)
   - 8.2 [Version Constraints](#82-version-constraints)
   - 8.3 [Decorator Requirements](#83-decorator-requirements)
   - 8.4 [Plugin Dependencies](#84-plugin-dependencies)
   - 8.5 [hasPlugin()](#85-hasplugin)
9. [Lifecycle Callbacks](#9-lifecycle-callbacks)
   - 9.1 [after()](#91-after)
   - 9.2 [ready()](#92-ready)
10. [Error Handling During Registration](#10-error-handling-during-registration)
11. [Plugin Load Order](#11-plugin-load-order)
12. [Real-World Patterns](#12-real-world-patterns)
    - 12.1 [Database Connection Plugin](#121-database-connection-plugin)
    - 12.2 [Authentication Plugin](#122-authentication-plugin)
    - 12.3 [Scoped API Version Plugin](#123-scoped-api-version-plugin)
13. [TypeScript Quick-Reference](#13-typescript-quick-reference)

Related deep dive:

- [Fastify Encapsulation — Complete Tutorial](./fastify-encapsulation.tutorial.md)

---

## 1. What is a Plugin?

A **plugin** is a plain function with the signature:

```
(instance, options, done) => void    // callback style
(instance, options) => Promise<void> // async style
```

`fastify.register(plugin, opts?)` hands that function a **child instance** — a
prototypal copy of the current Fastify instance — so anything defined inside the
plugin lives only inside that scope.

```
  fastify.register(plugin)
         |
         v
  plugin(childInstance, opts, done)
             |
             v
         childInstance
         (inherits from fastify, isolated by default)
```

This is the **entire mental model**: every `register` call creates a new scope;
everything added in that scope stays inside it unless you opt out.

---

## 2. Plugin Anatomy

```typescript
import Fastify, { FastifyInstance, FastifyPluginOptions } from 'fastify'

const app: FastifyInstance = Fastify({ logger: true })

// ---- Minimal plugin (callback style) ----------------------------------------
function myPlugin(
  instance: FastifyInstance,
  opts:     FastifyPluginOptions,
  done:     (err?: Error) => void
): void {
  instance.get('/hello', async () => ({ hello: 'world' }))
  done()             // MUST be called — forgetting it hangs the boot process
}

// ---- Minimal plugin (async style) -------------------------------------------
async function myAsyncPlugin(
  instance: FastifyInstance,
  opts:     FastifyPluginOptions
): Promise<void> {
  // no done() needed — returning the promise is enough
  instance.get('/ping', async () => 'pong')
}

app.register(myPlugin)
app.register(myAsyncPlugin)

await app.listen({ port: 3000 })
```

> **Rule:** callback-style plugins MUST call `done([err])`.
> Async plugins MUST NOT accept a third `done` argument;
> if a function is async and has three parameters, Fastify throws
> `FST_ERR_PLUGIN_INVALID_ASYNC_HANDLER`.

---

## 3. Encapsulation — The Core Concept

When Fastify calls `register`, the `override` function in
[lib/plugin-override.js](../../lib/plugin-override.js) creates a **child instance**
via `Object.create(old)`.  Each child gets its own copies of:

- hook arrays (`kHooks`)
- content-type parsers (`kContentTypeParser`)
- route prefix (`kRoutePrefix`)
- log level (`kLogLevel`)
- schema controller (`kSchemaController`)
- custom `Request` / `Reply` constructors (`kRequest` / `kReply`)

This means that anything set inside a plugin **does not leak to the parent or siblings**:

> Want the full deep dive? See
> [Fastify Encapsulation — Complete Tutorial](./fastify-encapsulation.tutorial.md).

```
  fastify (root)
  |
  +-- register(pluginA)
  |     |
  |     +-- decorate('dbA', ...)     <-- only visible inside pluginA and its children
  |     +-- addHook('onRequest', ..) <-- only fires for routes inside pluginA
  |     +-- GET /a
  |
  +-- register(pluginB)
        |
        +-- GET /b    // cannot see dbA decorator
```

### Hands-on example

```typescript
import Fastify, { FastifyInstance } from 'fastify'

const app = Fastify()

// Plugin A adds a decorator
app.register(async (instanceA: FastifyInstance) => {
  instanceA.decorate('secretKey', 'abc123')

  instanceA.get('/a', async (req) => {
    // OK: secretKey lives in instanceA's scope
    return { key: (instanceA as any).secretKey }
  })
})

// Plugin B cannot see secretKey
app.register(async (instanceB: FastifyInstance) => {
  instanceB.get('/b', async (req) => {
    // (instanceB as any).secretKey === undefined
    return { key: null }
  })
})

await app.listen({ port: 3000 })
```

**ASCII scope diagram:**

```
+--------------------------------------------------+
|  fastify (root)                                  |
|                                                  |
|  +--------------------+  +--------------------+  |
|  |  plugin A scope    |  |  plugin B scope    |  |
|  |  .secretKey='abc'  |  |  .secretKey = ???  |  |
|  |  GET /a            |  |  GET /b            |  |
|  +--------------------+  +--------------------+  |
+--------------------------------------------------+
```

---

## 4. fastify-plugin — Breaking Encapsulation

`fastify-plugin` (npm package `fastify-plugin`, alias `fp`) sets
`Symbol.for('skip-override') = true` on the plugin function.  When the
`override` function in `lib/plugin-override.js` sees that symbol:

```javascript
// lib/plugin-override.js (simplified)
const shouldSkipOverride = pluginUtils.registerPlugin.call(old, fn)
if (shouldSkipOverride) {
  old[kPluginNameChain].push(fnName)
  return old          // <-- returns the SAME instance, no child created
}
```

The plugin then runs **in the parent's scope**, making its decorators, hooks,
and parsers visible to every sibling and ancestor.

For a full treatment of scope boundaries and visibility rules, see
[Fastify Encapsulation — Complete Tutorial](./fastify-encapsulation.tutorial.md).

```
  WITHOUT fastify-plugin          WITH fastify-plugin
  ----------------------          ---------------------
  root                            root
   |                               |
   +-- register(plugin)            +-- register(fp(plugin))
         |                               |
         v                               v
      child instance              SAME instance (root)
      (isolated)                  (decorators visible everywhere)
```

### Example

```typescript
import Fastify, { FastifyInstance } from 'fastify'
import fp from 'fastify-plugin'

const app = Fastify()

// Wrapped with fp -> runs in root scope
app.register(fp(async (instance: FastifyInstance) => {
  instance.decorate('db', { query: async (sql: string) => [] })
}))

// Can see `db` because fp broke encapsulation
app.register(async (instance: FastifyInstance) => {
  instance.get('/users', async (req) => {
    // (instance as any).db is available here
    return (instance as any).db.query('SELECT * FROM users')
  })
})

await app.listen({ port: 3000 })
```

> **Rule of thumb:**
> - Share infrastructure (DB, cache, auth) across the whole app → use `fp`.
> - Scope routes / middleware to a sub-tree → omit `fp`.

---

## 5. Plugin Options

The second argument to `register` is passed verbatim as `opts` to the plugin:

```typescript
import Fastify, { FastifyInstance, FastifyPluginOptions } from 'fastify'

interface CacheOptions extends FastifyPluginOptions {
  ttl:  number
  size: number
}

async function cachePlugin(
  instance: FastifyInstance,
  opts:     CacheOptions
): Promise<void> {
  const cache = new Map<string, { value: unknown; expires: number }>()

  instance.decorate('cache', {
    get (key: string) {
      const entry = cache.get(key)
      if (!entry || Date.now() > entry.expires) return undefined
      return entry.value
    },
    set (key: string, value: unknown) {
      cache.set(key, { value, expires: Date.now() + opts.ttl })
    }
  })
}

const app = Fastify()
app.register(cachePlugin, { ttl: 5_000, size: 100 })
```

Options available on every `register` call (set by Fastify itself):

| Option           | Type     | Description                                     |
|------------------|----------|-------------------------------------------------|
| `prefix`         | `string` | URL prefix applied to all routes in the plugin  |
| `logLevel`       | `string` | Override log level for this plugin scope        |
| `logSerializers` | `object` | Custom log serializers for this scope           |

---

## 6. Route Prefixes

Passing `{ prefix: '/v1' }` makes [lib/plugin-override.js](../../lib/plugin-override.js)
call `buildRoutePrefix(instancePrefix, '/v1')`, which concatenates the prefixes:

```
  root prefix:   ""
  plugin prefix: "/v1"  ->  effective prefix: "/v1"

  nested plugin prefix: "/v2"  ->  effective prefix: "/v1/v2"
```

Fastify handles slash normalisation automatically (`/api/` + `/users` → `/api/users`).

```typescript
import Fastify, { FastifyInstance } from 'fastify'

const app = Fastify()

// GET /v1/users, GET /v1/users/:id
app.register(
  async (v1: FastifyInstance) => {
    v1.get('/users',     async () => [])
    v1.get('/users/:id', async (req) => ({ id: (req.params as any).id }))

    // Nested: GET /v1/admin/stats
    v1.register(
      async (admin: FastifyInstance) => {
        admin.get('/stats', async () => ({ ok: true }))
      },
      { prefix: '/admin' }
    )
  },
  { prefix: '/v1' }
)

await app.listen({ port: 3000 })
```

**Prefix nesting:**

```
  /
  +-- /v1              (plugin boundary)
       +-- /users
       +-- /users/:id
       +-- /admin       (nested plugin boundary)
            +-- /stats
```

---

## 7. Decorators

Decorators attach custom properties to the Fastify instance, every `Request`, or every
`Reply`.  Internally they use `Object.defineProperty` (for getter/setter) or direct
assignment on `instance` (see [lib/decorate.js](../../lib/decorate.js)).

### 7.1 `decorate`

```typescript
import Fastify from 'fastify'

const app = Fastify()

// Decorate with a plain value
app.decorate('appName', 'my-service')

// Decorate with a function
app.decorate('greet', (name: string) => `Hello, ${name}!`)

// Decorate with a getter/setter pair
app.decorate('timestamp', {
  getter () { return Date.now() }
})
```

> **Important:** `decorate` throws `FST_ERR_DEC_ALREADY_PRESENT` if the name already
> exists.  Use `hasDecorator` to guard.

### 7.2 `decorateRequest`

Adds a property to **every incoming request object**. For mutable state, initialise to
`null` and assign per request in a hook.

Do not use object/array defaults directly with `decorateRequest`/`decorateReply` because
that can create shared mutable state and Fastify rejects it with
`FST_ERR_DEC_REFERENCE_TYPE`:

```typescript
import Fastify, { FastifyRequest } from 'fastify'

// Augment the FastifyRequest type
declare module 'fastify' {
  interface FastifyRequest {
    currentUser: { id: string; roles: string[] } | null
  }
}

const app = Fastify()

// Initialise to null, then set a request-local value in hooks/handlers
app.decorateRequest('currentUser', null)

app.addHook('preHandler', async (request: FastifyRequest) => {
  request.currentUser = await loadUser(request.headers['x-user-id'] as string)
})

async function loadUser (id: string) {
  return { id, roles: ['user'] }
}
```

### 7.3 `decorateReply`

Same pattern for the reply object:

```typescript
import Fastify, { FastifyReply } from 'fastify'

declare module 'fastify' {
  interface FastifyReply {
    sendSuccess (data: unknown): FastifyReply
  }
}

const app = Fastify()

app.decorateReply('sendSuccess', function (this: FastifyReply, data: unknown) {
  return this.code(200).send({ success: true, data })
})

app.get('/ok', async (req, reply) => {
  return reply.sendSuccess({ message: 'all good' })
})
```

### 7.4 `hasDecorator` Guards

```typescript
import Fastify from 'fastify'
import fp from 'fastify-plugin'

const app = Fastify()

app.register(fp(async (instance) => {
  // Only decorate once across the whole app
  if (!instance.hasDecorator('config')) {
    instance.decorate('config', { env: process.env.NODE_ENV })
  }

  if (!instance.hasRequestDecorator('requestId')) {
    instance.decorateRequest('requestId', null)
  }

  if (!instance.hasReplyDecorator('sendError')) {
    instance.decorateReply('sendError', function (this: any, msg: string) {
      return this.code(500).send({ error: msg })
    })
  }
}))
```

### 7.5 Decorator Dependencies

You can declare that a decorator requires another decorator to exist first:

```typescript
import Fastify from 'fastify'

const app = Fastify()

// Must exist first
app.decorate('db', { query: async (sql: string) => [] })

// Declares dependency on 'db' — throws FST_ERR_DEC_MISSING_DEPENDENCY if absent
app.decorate('repo', { findUser: async (id: string) => null }, ['db'])
```

---

## 8. Plugin Metadata & fastify-plugin

`fastify-plugin` reads metadata from the `Symbol.for('plugin-meta')` property.
Pass a metadata object as the second argument to `fp(fn, meta)`:

### 8.1 Name

Naming your plugin helps with debugging — `fastify.pluginName` traces the chain:

```typescript
import fp from 'fastify-plugin'

export default fp(
  async (instance) => {
    instance.decorate('pg', { /* ... */ })
  },
  { name: 'postgres-connector' }
)
// fastify.pluginName => "fastify -> postgres-connector"
```

Name resolution order: `fastify.display-name` symbol → module filename (from `require.cache`)
→ function name → a short function preview. Anonymous plugins may show an
approximated preview, and when using `fp` the chain accumulates (e.g.,
`fastify -> plugin-A -> plugin-B`).

### 8.2 Version Constraints

```typescript
import fp from 'fastify-plugin'

export default fp(
  async (instance) => { /* ... */ },
  {
    name:    'my-plugin',
    fastify: '5.x'   // semver range — throws FST_ERR_PLUGIN_VERSION_MISMATCH if violated
  }
)
```

### 8.3 Decorator Requirements

Declare which decorators your plugin requires so Fastify validates them at load time:

```typescript
import fp from 'fastify-plugin'

export default fp(
  async (instance) => {
    // Can safely use instance.db — Fastify verified it exists before calling this
    const db = (instance as any).db
    instance.decorate('userRepo', { find: (id: string) => db.query(`SELECT * FROM users WHERE id=${id}`) })
  },
  {
    name:       'user-repository',
    decorators: {
      fastify: ['db'],      // instance.db must exist
      request: [],
      reply:   []
    }
  }
)
```

Missing required decorators cause Fastify to throw
`FST_ERR_PLUGIN_NOT_PRESENT_IN_INSTANCE` during registration.

### 8.4 Plugin Dependencies

Declare that your plugin requires another named plugin to have been registered first:

```typescript
import fp from 'fastify-plugin'

export default fp(
  async (instance) => {
    // postgres-connector ran first, so instance.pg is available
    const pg = (instance as any).pg
    instance.decorate('userRepo', { find: (id: string) => pg.query(`SELECT 1`) })
  },
  {
    name:         'user-repository',
    dependencies: ['postgres-connector']  // throws "The dependency 'postgres-connector' of plugin 'user-repository' is not registered" if not found
  }
)
```

### 8.5 hasPlugin()

Use `fastify.hasPlugin(name)` to check whether a named plugin has been registered
in the current scope.

Encapsulation rules apply:
- A plugin can see names registered in its own scope and ancestor scopes.
- Parent and sibling scopes cannot see names from encapsulated children.
- If you wrap with `fastify-plugin`, names are visible across the shared scope.

---

## 9. Lifecycle Callbacks

### 9.1 `after()`

`after` fires once the **current** registration batch has loaded, before `ready`:

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.register(async (instance) => {
  instance.decorate('db', {})
})

// Runs synchronously after the register above settles
app.after((err) => {
  if (err) throw err
  console.log('plugin batch loaded, db =', (app as any).db)
})

// Awaitable form
await app.register(async (instance) => {
  instance.decorate('cache', {})
})
// At this point, cache is already decorated on the child scope
```

### 9.2 `ready()`

`ready` fires once **all** plugins and hooks have been registered (avvio finalisation):

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.register(async (instance) => {
  instance.decorate('db', { pool: null as any })
})

app.ready((err) => {
  if (err) throw err
  // All plugins have initialised
  console.log('server ready')
})

// Async / promise style (recommended)
await app.ready()
```

**Timing diagram:**

```
  register(A)  register(B)  register(C)
       |             |             |
       v             v             v
    [batch queue]--[batch queue]--[batch queue]
                                       |
                                       v
                                 after() fires
                                       |
                                       v
                                 ready() fires
                                       |
                                       v
                                 listen() opens socket
```

---

## 10. Error Handling During Registration

Any error thrown (or passed via `done(err)`) inside a plugin aborts the entire load:

```typescript
import Fastify from 'fastify'

const app = Fastify()

// Callback style error propagation
app.register((instance, opts, done) => {
  const connected = false  // pretend DB connection failed
  if (!connected) {
    done(new Error('DB connection failed'))
    return
  }
  done()
})

// Async style — just throw
app.register(async (instance) => {
  const connected = false
  if (!connected) {
    throw new Error('DB connection failed')
  }
})

try {
  await app.ready()
} catch (err) {
  console.error('Boot failed:', (err as Error).message)
  process.exit(1)
}
```

`after` can intercept errors from the immediately preceding batch:

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.register((instance, opts, done) => {
  done(new Error('oops'))
})

app.after((err) => {
  // err.message === 'oops'
  // return without re-throwing to suppress the error
  console.error('Handled:', err.message)
})

await app.ready()  // does NOT throw because after() swallowed the error
```

---

## 11. Plugin Load Order

Fastify uses [avvio](https://github.com/mcollina/avvio) for boot sequencing.
Plugins are initialised **serially** in registration order, even if they are async:

```
  Register order:     A -> B -> C
  Init order:         A -> B -> C     (always serial)
  After A resolves:   B starts
  After B resolves:   C starts
  After C resolves:   ready() fires
```

Nested plugins are initialised **depth-first**:

```
  register(A)
    register(A1)
    register(A2)
  register(B)
    register(B1)
```

```
  Init order: A -> A1 -> A2 -> B -> B1
```

**Practical consequence:** if plugin B depends on a decorator added by plugin A,
registering A before B is sufficient — no need for explicit dependency declarations
(although declaring them is good practice for documentation and safety).

---

## 12. Real-World Patterns

### 12.1 Database Connection Plugin

A reusable, encapsulation-breaking database plugin:

```typescript
// plugins/db.ts
import fp from 'fastify-plugin'
import { FastifyInstance } from 'fastify'

export interface DbClient {
  query: (sql: string, params?: unknown[]) => Promise<unknown[]>
  close: () => Promise<void>
}

declare module 'fastify' {
  interface FastifyInstance {
    db: DbClient
  }
}

async function dbPlugin(instance: FastifyInstance): Promise<void> {
  // In a real app: use pg, mysql2, knex, prisma, etc.
  const client: DbClient = {
    query: async (sql, params) => {
      console.log('query:', sql, params)
      return []
    },
    close: async () => {}
  }

  instance.decorate('db', client)

  // Close the connection when the app shuts down
  instance.addHook('onClose', async () => {
    await client.close()
  })
}

export default fp(dbPlugin, {
  name:    'db',
  fastify: '5.x'
})
```

```typescript
// app.ts
import Fastify from 'fastify'
import dbPlugin from './plugins/db.js'

const app = Fastify({ logger: true })

app.register(dbPlugin)

app.get('/users', async (req, reply) => {
  const users = await app.db.query('SELECT * FROM users')
  return users
})

await app.listen({ port: 3000 })
```

---

### 12.2 Authentication Plugin

```typescript
// plugins/auth.ts
import fp from 'fastify-plugin'
import { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify'

interface AuthenticatedUser {
  id:    string
  roles: string[]
}

declare module 'fastify' {
  interface FastifyRequest {
    user: AuthenticatedUser | null
  }
  interface FastifyInstance {
    authenticate: (req: FastifyRequest, reply: FastifyReply) => Promise<void>
  }
}

async function authPlugin(instance: FastifyInstance): Promise<void> {
  instance.decorateRequest('user', null)

  // Shared authenticate helper — call from preHandler on protected routes
  instance.decorate('authenticate', async (req: FastifyRequest, reply: FastifyReply) => {
    const token = req.headers['authorization']?.replace('Bearer ', '')
    if (!token) {
      reply.code(401).send({ error: 'Unauthorized' })
      return
    }
    // Replace with real JWT verification
    req.user = { id: token, roles: ['user'] }
  })
}

export default fp(authPlugin, { name: 'auth' })
```

```typescript
// routes/protected.ts
import { FastifyInstance } from 'fastify'

export default async function protectedRoutes(instance: FastifyInstance): Promise<void> {
  // Apply authenticate to every route in this plugin scope
  instance.addHook('preHandler', instance.authenticate)

  instance.get('/profile', async (req) => {
    return { user: req.user }
  })

  instance.get('/settings', async (req) => {
    return { id: req.user?.id, theme: 'dark' }
  })
}
```

```typescript
// app.ts
import Fastify from 'fastify'
import authPlugin     from './plugins/auth.js'
import protectedRoutes from './routes/protected.js'

const app = Fastify({ logger: true })

app.register(authPlugin)
app.register(protectedRoutes, { prefix: '/api' })

await app.listen({ port: 3000 })
```

---

### 12.3 Scoped API Version Plugin

Different API versions in isolated scopes — each version can have its own
content-type parsers, hooks, and error handlers:

```typescript
import Fastify, { FastifyInstance } from 'fastify'

const app = Fastify({ logger: true })

// Shared infrastructure (no fp needed — still root scope here)
app.decorate('config', { dbUrl: process.env.DATABASE_URL ?? '' })

// ----- v1 ---------------------------------------------------------------
async function v1Routes(instance: FastifyInstance): Promise<void> {
  instance.addHook('onRequest', async (req) => {
    req.headers['x-api-version'] = '1'
  })

  instance.get('/users', async () => [{ id: 1, name: 'Alice (v1)' }])
}

// ----- v2 ---------------------------------------------------------------
async function v2Routes(instance: FastifyInstance): Promise<void> {
  instance.addHook('onRequest', async (req) => {
    req.headers['x-api-version'] = '2'
  })

  instance.get('/users', async () => ({
    data:  [{ id: 1, name: 'Alice (v2)' }],
    total: 1
  }))
}

app.register(v1Routes, { prefix: '/v1' })
app.register(v2Routes, { prefix: '/v2' })

await app.listen({ port: 3000 })
// GET /v1/users  ->  [{id:1, name:"Alice (v1)"}]
// GET /v2/users  ->  {data:[...], total:1}
```

**Plugin tree:**

```
  app (root)
  |
  +-- decorate('config', ...)
  |
  +-- register(v1Routes, prefix='/v1')
  |     |
  |     +-- addHook('onRequest', ...)  -- only for /v1/* routes
  |     +-- GET /v1/users
  |
  +-- register(v2Routes, prefix='/v2')
        |
        +-- addHook('onRequest', ...)  -- only for /v2/* routes
        +-- GET /v2/users
```

---

## 13. TypeScript Quick-Reference

### Plugin signature type helpers

```typescript
import {
  FastifyPluginAsync,
  FastifyPluginCallback,
  FastifyPluginOptions
} from 'fastify'

// Async plugin with typed options
interface MyOptions extends FastifyPluginOptions {
  secret: string
}

const myPlugin: FastifyPluginAsync<MyOptions> = async (instance, opts) => {
  instance.decorate('secret', opts.secret)
}

// Callback plugin
const myCallbackPlugin: FastifyPluginCallback<MyOptions> = (instance, opts, done) => {
  instance.decorate('secret', opts.secret)
  done()
}
```

### Module augmentation

Always augment `fastify` module types so TypeScript can see your decorators:

```typescript
declare module 'fastify' {
  interface FastifyInstance {
    db:         DbClient
    config:     AppConfig
    authenticate: (req: FastifyRequest, reply: FastifyReply) => Promise<void>
  }
  interface FastifyRequest {
    user:      AuthenticatedUser | null
    requestId: string
  }
  interface FastifyReply {
    sendSuccess: (data: unknown) => FastifyReply
    sendError:   (msg: string, code?: number) => FastifyReply
  }
}
```

### Using `fp` with TypeScript

```typescript
import fp from 'fastify-plugin'
import { FastifyPluginAsync } from 'fastify'

const plugin: FastifyPluginAsync = async (instance) => {
  instance.decorate('hello', () => 'world')
}

export default fp(plugin, {
  name:    'hello-plugin',
  fastify: '5.x'
})
```

### Common error codes

| Code                                    | Cause                                                      |
|-----------------------------------------|------------------------------------------------------------|
| `FST_ERR_DEC_ALREADY_PRESENT`           | `decorate()` called twice with the same name               |
| `FST_ERR_DEC_MISSING_DEPENDENCY`        | Required decorator not present before this decorator        |
| `FST_ERR_DEC_AFTER_START`              | `decorate()` called after `listen()` / `ready()`           |
| `FST_ERR_DEC_REFERENCE_TYPE`           | Object/array used as default for `decorateRequest`/`decorateReply` |
| `FST_ERR_PLUGIN_VERSION_MISMATCH`       | Plugin's `fastify` semver range doesn't match runtime       |
| `FST_ERR_PLUGIN_NOT_PRESENT_IN_INSTANCE`| Required decorator not present during plugin registration    |
| `FST_ERR_PLUGIN_INVALID_ASYNC_HANDLER`  | Async plugin function has three arguments (`done`)          |

---

*End of tutorial.*
