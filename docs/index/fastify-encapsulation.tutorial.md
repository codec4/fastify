# Fastify Encapsulation — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to
> master Fastify encapsulation from internals and tests.

---

## Table of Contents

1. [What Encapsulation Means in Fastify](#1-what-encapsulation-means-in-fastify)
2. [The Plugin Tree Mental Model](#2-the-plugin-tree-mental-model)
3. [How `register()` Creates Scope Internally](#3-how-register-creates-scope-internally)
4. [Visibility Rules (Parent, Child, Sibling)](#4-visibility-rules-parent-child-sibling)
5. [Route Prefix Encapsulation and Prefix Math](#5-route-prefix-encapsulation-and-prefix-math)
6. [Decorators in Encapsulated Scopes](#6-decorators-in-encapsulated-scopes)
7. [Hooks in Encapsulated Scopes](#7-hooks-in-encapsulated-scopes)
8. [Error and NotFound Handler Scoping](#8-error-and-notfound-handler-scoping)
9. [Logger and Request ID Scoping](#9-logger-and-request-id-scoping)
10. [Breaking Encapsulation with `fastify-plugin`](#10-breaking-encapsulation-with-fastify-plugin)
11. [Encapsulated vs `fastify-plugin` — Comparison Matrix](#11-encapsulated-vs-fastify-plugin--comparison-matrix)
12. [Real-World Architecture Pattern (Detailed)](#12-real-world-architecture-pattern-detailed)
13. [TypeScript Quick Reference](#13-typescript-quick-reference)
14. [Best Practices and Common Pitfalls](#14-best-practices-and-common-pitfalls)

---

## 1. What Encapsulation Means in Fastify

In Fastify, encapsulation means each `register()` call creates a **new plugin scope**.

Inside that scope, you can add:

- routes
- decorators
- hooks
- error handlers
- not-found handlers
- parser/compiler/logger/request-id customizations

Those additions are visible inside the scope and its descendants, but not in siblings
or parent (unless encapsulation is explicitly bypassed).

In one sentence:

```text
register(plugin) => child scope (inherits parent, isolated from siblings/parent writes)
```

---

## 2. The Plugin Tree Mental Model

Think of your app as a tree, not a flat list.

```text
root
|
+-- auth-plugin
|   |
|   +-- admin-plugin
|
+-- billing-plugin
```

Rules:

1. Parent state is readable by children.
2. Child additions are not visible to parent.
3. Siblings cannot see each other's additions.

Example (TypeScript):

```typescript
import Fastify, { FastifyInstance, FastifyPluginAsync } from 'fastify'

const app: FastifyInstance = Fastify()

type AuthUser = { id: string }

declare module 'fastify' {
  interface FastifyInstance {
    authSecret?: string
    billingSecret?: string
  }
}

const authPlugin: FastifyPluginAsync = async (instance) => {
  instance.decorate('authSecret', 'auth-only')

  instance.get('/auth/secret', async () => ({ authSecret: instance.authSecret }))
}

const billingPlugin: FastifyPluginAsync = async (instance) => {
  instance.decorate('billingSecret', 'billing-only')

  instance.get('/billing/secret', async () => ({
    billingSecret: instance.billingSecret,
    seesAuthSecret: instance.hasDecorator('authSecret')
  }))
}

app.register(authPlugin)
app.register(billingPlugin)
```

`billingPlugin` cannot access `authSecret` because it is a sibling scope.

---

## 3. How `register()` Creates Scope Internally

At registration time Fastify's override path creates a child context via prototypal
inheritance and scope-local copies of mutable internals.

Important internal behaviors (simplified):

- child instance is created from parent (`Object.create(parent)`)
- child receives copied hook arrays
- child computes its own route prefix
- child gets scoped route parser/schema/logger-related state
- `onRegister` hooks execute for that registration event

Conceptual flow:

```text
parent.register(plugin)
    |
    +-- create child scope
    +-- copy scope-sensitive internals
    +-- run onRegister(parent -> child)
    +-- execute plugin(child)
```

Practical implication: encapsulation is deterministic and structural, not dynamic.

---

## 4. Visibility Rules (Parent, Child, Sibling)

### 4.1 Parent -> Child visibility

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

app.decorate('serviceName', 'root-service')

const childPlugin: FastifyPluginAsync = async (instance) => {
  instance.get('/child', async () => ({
    serviceName: instance.getDecorator<string>('serviceName')
  }))
}

app.register(childPlugin)
```

The child can read root decorators.

### 4.2 Child !-> Parent visibility

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

const childPlugin: FastifyPluginAsync = async (instance) => {
  instance.decorate('childOnlyValue', 42)
}

app.register(childPlugin)

app.get('/root', async () => ({
  hasChildOnlyValue: app.hasDecorator('childOnlyValue')
}))
```

Root cannot see child's new decorator.

### 4.3 Sibling isolation

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

const pluginA: FastifyPluginAsync = async (instance) => {
  instance.decorate('fromA', true)
}

const pluginB: FastifyPluginAsync = async (instance) => {
  instance.get('/b', async () => ({
    seesFromA: instance.hasDecorator('fromA')
  }))
}

app.register(pluginA)
app.register(pluginB)
```

`pluginB` does not see `fromA`.

### 4.4 Quick visibility table

```text
Where decorator is declared: child A

Can root read it?         no
Can child A read it?      yes
Can descendants of A?     yes
Can sibling child B?      no
```

---

## 5. Route Prefix Encapsulation and Prefix Math

Prefix is scope-local and composes with parent prefixes.

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

const v1Plugin: FastifyPluginAsync = async (instance) => {
  instance.get('/users', async () => [{ id: 1 }])

  instance.register(async function adminPlugin (admin) {
    admin.get('/stats', async () => ({ ok: true }))
  }, { prefix: '/admin' })
}

app.register(v1Plugin, { prefix: '/api/v1' })
```

Resulting routes:

```text
GET /api/v1/users
GET /api/v1/admin/stats
```

Pattern: use prefixes as bounded contexts.

- `/api/v1/auth/*`
- `/api/v1/billing/*`
- `/api/v1/admin/*`

Each subtree can carry its own hooks/handlers/decorators.

---

## 6. Decorators in Encapsulated Scopes

Decorators are one of the most direct ways to observe encapsulation.

### 6.1 Instance decorator in child scope

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

declare module 'fastify' {
  interface FastifyInstance {
    formatCurrency: (value: number) => string
  }
}

const billingPlugin: FastifyPluginAsync = async (instance) => {
  instance.decorate('formatCurrency', (value: number) => `$${value.toFixed(2)}`)

  instance.get('/billing/preview', async () => ({
    total: instance.formatCurrency(10)
  }))
}

app.register(billingPlugin)
```

Outside `billingPlugin`, this decorator is unavailable.

### 6.2 Request/reply decorators remain scope-aware

```typescript
import Fastify, {
  FastifyPluginAsync,
  FastifyReply,
  FastifyRequest
} from 'fastify'

declare module 'fastify' {
  interface FastifyRequest {
    tenantId: string | null
  }
  interface FastifyReply {
    sendTenantAware: (payload: unknown) => FastifyReply
  }
}

const tenantPlugin: FastifyPluginAsync = async (instance) => {
  instance.decorateRequest('tenantId', null)

  instance.decorateReply('sendTenantAware', function (this: FastifyReply, payload: unknown) {
    return this.send({ tenant: this.request.tenantId, payload })
  })

  instance.addHook('onRequest', async (request: FastifyRequest) => {
    request.tenantId = (request.headers['x-tenant-id'] as string | undefined) ?? null
  })

  instance.get('/tenant/data', async (request, reply) => {
    return reply.sendTenantAware({ hello: 'world' })
  })
}
```

### 6.3 Dependency checks are scoped

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

const pluginA: FastifyPluginAsync = async (instance) => {
  instance.decorate('db', { query: async () => [] })
}

const pluginB: FastifyPluginAsync = async (instance) => {
  instance.decorate('repo', {
    findAll: async () => instance.getDecorator<{ query: () => Promise<unknown[]> }>('db').query()
  }, ['db'])
}

app.register(pluginA)
app.register(pluginB)
```

If `db` is not visible in the current scope chain, dependency validation fails.

---

## 7. Hooks in Encapsulated Scopes

Hook arrays are copied into child scopes. This creates inheritance + isolation.

### 7.1 Parent hook inherited by child

```typescript
import Fastify, { FastifyPluginAsync, FastifyRequest } from 'fastify'

const app = Fastify()

app.decorateRequest('traceRoot', null)

app.addHook('onRequest', async (request: FastifyRequest) => {
  request.traceRoot = 'root'
})

const childPlugin: FastifyPluginAsync = async (instance) => {
  instance.get('/child', async (request) => ({ traceRoot: request.traceRoot }))
}

app.register(childPlugin)
```

`/child` receives root hook behavior.

### 7.2 Child hook only affects child subtree

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

const adminPlugin: FastifyPluginAsync = async (instance) => {
  instance.addHook('preHandler', async (request, reply) => {
    const token = request.headers.authorization
    if (!token) {
      reply.code(401).send({ error: 'Unauthorized' })
    }
  })

  instance.get('/admin/dashboard', async () => ({ ok: true }))
}

app.register(adminPlugin)
app.get('/public', async () => ({ ok: true }))
```

`/public` does not execute admin `preHandler`.

### 7.3 Hook ordering intuition

For a route inside child scope:

```text
parent hooks (in order) -> child hooks (in order) -> route-level hooks
```

---

## 8. Error and NotFound Handler Scoping

Both error and 404 handlers are encapsulated, and both are central to bounded APIs.

### 8.1 `setErrorHandler` scope chain

Inner scope handler gets first chance. If it rethrows, parent error handler can catch.

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

app.setErrorHandler(async (error, request, reply) => {
  reply.code(500).send({ where: 'root', message: error.message })
})

const adminPlugin: FastifyPluginAsync = async (instance) => {
  instance.setErrorHandler(async (error, request, reply) => {
    if (error.message === 'admin-only') {
      reply.code(403).send({ where: 'admin', message: error.message })
      return
    }

    throw error
  })

  instance.get('/admin/fail', async () => {
    throw new Error('admin-only')
  })

  instance.get('/admin/fail-up', async () => {
    throw new Error('bubble-up')
  })
}

app.register(adminPlugin)
```

### 8.2 `setNotFoundHandler` is also scoped

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

app.setNotFoundHandler(async (request, reply) => {
  reply.code(404).send({ area: 'root', url: request.url })
})

const adminPlugin: FastifyPluginAsync = async (instance) => {
  instance.setNotFoundHandler(async (request, reply) => {
    reply.code(404).send({ area: 'admin', url: request.url })
  })

  instance.get('/admin/known', async () => ({ ok: true }))
}

app.register(adminPlugin, { prefix: '/admin' })
```

Requests under `/admin/*` resolve through admin not-found behavior.

### 8.3 `setNotFoundHandler(options, handler)` and `reply.callNotFound()`

Use this to run pre-validation/pre-handler logic in the not-found flow.

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify()

const docsPlugin: FastifyPluginAsync = async (instance) => {
  instance.setNotFoundHandler(
    {
      preHandler: async (request, reply) => {
        reply.header('x-docs-scope', '1')
      }
    },
    async (request, reply) => {
      reply.code(404).send({ message: 'Docs endpoint not found' })
    }
  )

  instance.get('/forward', async (request, reply) => {
    reply.callNotFound()
  })
}

app.register(docsPlugin, { prefix: '/docs' })
```

### 8.4 Full 404 scope constraints

Key constraints demonstrated by tests:

- one active not-found handler per 404 level/scope
- nested scope can define its own
- root and child handlers can coexist
- prefix-scoped 404 behavior is deterministic

ASCII view:

```text
root 404 handler
|
+-- /admin scope with its own 404
|
+-- /docs scope with its own 404 (+ preHandler)
```

---

## 9. Logger and Request ID Scoping

Encapsulation applies to logging and request identity strategy as well.

### 9.1 Scoped `setChildLoggerFactory`

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify({ logger: true })

const apiPlugin: FastifyPluginAsync = async (instance) => {
  instance.setChildLoggerFactory(function (logger, bindings, opts, rawReq) {
    return logger.child({
      ...bindings,
      area: 'api',
      method: rawReq.method
    })
  })

  instance.get('/api/ping', async () => ({ pong: true }))
}

app.register(apiPlugin)
```

Only the plugin subtree gets this child logger factory.

### 9.2 Scoped `setGenReqId`

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'

const app = Fastify({ logger: true })

app.setGenReqId(() => `root-${Date.now()}`)

const internalPlugin: FastifyPluginAsync = async (instance) => {
  instance.setGenReqId(() => `internal-${Math.random().toString(36).slice(2)}`)
  instance.get('/internal/check', async (request) => ({ id: request.id }))
}

app.register(internalPlugin)
app.get('/public/check', async (request) => ({ id: request.id }))
```

Requests in `/internal/*` can use a different id strategy from root/public routes.

---

## 10. Breaking Encapsulation with `fastify-plugin`

`fastify-plugin` intentionally runs a plugin in the current instance scope (skip
child override), so the plugin's additions become shared at that level.

```typescript
import Fastify, { FastifyPluginAsync } from 'fastify'
import fp from 'fastify-plugin'

const app = Fastify()

declare module 'fastify' {
  interface FastifyInstance {
    platformVersion: string
  }
}

const platformPlugin: FastifyPluginAsync = async (instance) => {
  instance.decorate('platformVersion', '2026.03')
}

app.register(fp(platformPlugin, { name: 'platform-plugin' }))

app.register(async function pluginA (instance) {
  instance.get('/a', async () => ({ platformVersion: instance.platformVersion }))
})

app.register(async function pluginB (instance) {
  instance.get('/b', async () => ({ platformVersion: instance.platformVersion }))
})
```

Both sibling plugins can read `platformVersion`.

Guideline:

- use encapsulated plugins for feature isolation
- use `fastify-plugin` only for intentionally shared platform wiring

---

## 11. Encapsulated vs `fastify-plugin` — Comparison Matrix

```text
+-------------------------+------------------------------+------------------------------+
| Capability              | Plain register(plugin)       | register(fp(plugin))         |
+-------------------------+------------------------------+------------------------------+
| Child instance created  | yes                          | no (same instance)           |
| Decorators leak upward  | no                           | yes (shared at level)        |
| Sibling visibility      | isolated                     | shared                       |
| Hook scope              | subtree only                 | shared at level              |
| Error/notFound handlers | subtree chain                | shared at level              |
| Logger factory scope    | subtree only                 | shared at level              |
| Request-id strategy     | subtree only                 | shared at level              |
+-------------------------+------------------------------+------------------------------+
```

Inline reminder for all sections:

- If you need reuse with boundaries: keep encapsulation.
- If you need global platform behavior: use `fastify-plugin` deliberately.

---

## 12. Real-World Architecture Pattern (Detailed)

Goal: one shared platform layer + encapsulated feature modules.

### 12.1 Architecture

```text
root app
|
+-- fp(platformPlugin)      // shared logger/db/config decorators
|
+-- register(authModule, { prefix: /auth })
+-- register(billingModule, { prefix: /billing })
+-- register(adminModule, { prefix: /admin })
```

### 12.2 Shared platform plugin (intentionally global)

```typescript
import fp from 'fastify-plugin'
import {
  FastifyInstance,
  FastifyPluginAsync
} from 'fastify'

type DbClient = {
  query: (sql: string) => Promise<unknown[]>
}

declare module 'fastify' {
  interface FastifyInstance {
    db: DbClient
    config: {
      env: 'development' | 'staging' | 'production'
    }
  }
}

const platformPlugin: FastifyPluginAsync = async (instance: FastifyInstance) => {
  instance.decorate('db', {
    query: async (_sql: string) => []
  })

  instance.decorate('config', {
    env: 'development'
  })
}

export default fp(platformPlugin, { name: 'platform' })
```

### 12.3 Encapsulated auth module

```typescript
import {
  FastifyPluginAsync,
  FastifyReply,
  FastifyRequest
} from 'fastify'

type User = { id: string; role: 'user' | 'admin' }

declare module 'fastify' {
  interface FastifyRequest {
    user: User | null
  }
  interface FastifyInstance {
    authenticate: (request: FastifyRequest, reply: FastifyReply) => Promise<void>
  }
}

export const authModule: FastifyPluginAsync = async (instance) => {
  instance.decorateRequest('user', null)

  instance.decorate('authenticate', async (request, reply) => {
    const token = request.headers.authorization
    if (!token) {
      reply.code(401).send({ error: 'Unauthorized' })
      return
    }

    request.user = { id: token, role: 'user' }
  })

  instance.addHook('preHandler', async (request, reply) => {
    await instance.authenticate(request, reply)
  })

  instance.get('/private/me', async (request) => ({ user: request.user }))
}
```

### 12.4 Encapsulated billing module with local error/not-found behavior

```typescript
import { FastifyPluginAsync } from 'fastify'

export const billingModule: FastifyPluginAsync = async (instance) => {
  instance.setErrorHandler(async (error, request, reply) => {
    request.log.error({ err: error }, 'billing error')
    reply.code(500).send({ area: 'billing', message: 'Billing failure' })
  })

  instance.setNotFoundHandler(async (request, reply) => {
    reply.code(404).send({ area: 'billing', message: 'Billing endpoint not found' })
  })

  instance.get('/invoices', async () => {
    const rows = await instance.db.query('SELECT * FROM invoices')
    return { rows }
  })

  instance.get('/fail', async () => {
    throw new Error('billing-boom')
  })
}
```

### 12.5 Root assembly

```typescript
import Fastify from 'fastify'
import platformPlugin from './platform-plugin'
import { authModule } from './auth-module'
import { billingModule } from './billing-module'

const app = Fastify({ logger: true })

app.register(platformPlugin)
app.register(authModule, { prefix: '/auth' })
app.register(billingModule, { prefix: '/billing' })

app.setNotFoundHandler(async (request, reply) => {
  reply.code(404).send({ area: 'root', message: 'Route not found' })
})

await app.listen({ port: 3000 })
```

### 12.6 Why this pattern scales

- shared platform concerns remain explicit and centralized
- feature modules keep isolated hooks/handlers/decorators
- bounded prefixes map cleanly to ownership boundaries
- local error/404 behavior avoids global branching complexity

---

## 13. TypeScript Quick Reference

### 13.1 Typed plugin signatures

```typescript
import {
  FastifyPluginAsync,
  FastifyPluginCallback,
  FastifyPluginOptions
} from 'fastify'

interface BillingOptions extends FastifyPluginOptions {
  currency: 'USD' | 'EUR'
}

export const billingAsyncPlugin: FastifyPluginAsync<BillingOptions> = async (instance, opts) => {
  instance.decorate('currency', opts.currency)
}

export const billingCallbackPlugin: FastifyPluginCallback<BillingOptions> = (instance, opts, done) => {
  instance.decorate('currency', opts.currency)
  done()
}
```

### 13.2 Module augmentation template

```typescript
declare module 'fastify' {
  interface FastifyInstance {
    db: { query: (sql: string) => Promise<unknown[]> }
    authenticate: (request: FastifyRequest, reply: FastifyReply) => Promise<void>
  }

  interface FastifyRequest {
    user: { id: string; role: 'user' | 'admin' } | null
    tenantId: string | null
  }

  interface FastifyReply {
    sendUnauthorized: () => FastifyReply
  }
}
```

### 13.3 Typed decorator accessors

```typescript
const db = fastify.getDecorator<{ query: (sql: string) => Promise<unknown[]> }>('db')
const exists = fastify.hasDecorator('db')
```

---

## 14. Best Practices and Common Pitfalls

### Best practices

1. Design plugin boundaries first (ownership -> scope -> prefix).
2. Keep feature hooks local; avoid global hooks unless truly cross-cutting.
3. Use `fastify-plugin` only for intentional shared infrastructure.
4. Prefer local error/not-found handlers for bounded APIs.
5. Use decorator dependencies to enforce module contracts.
6. Keep request/reply decorator defaults non-reference (`null` + getters/setters if needed).

### Common pitfalls

- Expecting child decorators to be visible in parent or siblings.
- Mixing async and callback signatures in plugins/hooks.
- Defining decorators after startup.
- Using object/array defaults for request/reply decorators.
- Overusing `fastify-plugin`, accidentally flattening module boundaries.
- Attempting multiple not-found handlers in the same 404 level.

### Error codes you should recognize

```text
FST_ERR_DEC_ALREADY_PRESENT
FST_ERR_DEC_MISSING_DEPENDENCY
FST_ERR_DEC_DEPENDENCY_INVALID_TYPE
FST_ERR_DEC_REFERENCE_TYPE
FST_ERR_DEC_AFTER_START
FST_ERR_DEC_UNDECLARED
FST_ERR_PLUGIN_INVALID_ASYNC_HANDLER
```

---

### Source grounding

This tutorial is aligned with behavior demonstrated in:

- `fastify.js`
- `lib/plugin-override.js`
- `lib/plugin-utils.js`
- `lib/decorate.js`
- `lib/hooks.js`
- `lib/route.js`
- `lib/four-oh-four.js`
- `lib/error-handler.js`
- `lib/logger-factory.js`
- `test/plugin.2.test.js`
- `test/decorator.test.js`
- `test/hooks.test.js`
- `test/hooks-async.test.js`
- `test/route-prefix.test.js`
- `test/404s.test.js`
- `test/encapsulated-error-handler.test.js`
- `test/encapsulated-child-logger-factory.test.js`
- `test/genReqId.test.js`

*End of tutorial.*
