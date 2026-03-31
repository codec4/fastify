# Fastify Decorators — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to
> master Fastify decorators from first principles.

---

## Table of Contents

1. [What is a Decorator?](#1-what-is-a-decorator)
2. [Decorator Types](#2-decorator-types)
3. [Encapsulation and Scope](#3-encapsulation-and-scope)
4. [Instance Decorators with `decorate`](#4-instance-decorators-with-decorate)
5. [Request Decorators with `decorateRequest`](#5-request-decorators-with-decoraterequest)
6. [Reply Decorators with `decorateReply`](#6-reply-decorators-with-decoratereply)
7. [Reference Types and Safe Per-Request State](#7-reference-types-and-safe-per-request-state)
8. [Dependencies](#8-dependencies)
9. [Introspection APIs (`has*Decorator`, `getDecorator`, `setDecorator`)](#9-introspection-apis-hasdecorator-getdecorator-setdecorator)
10. [Startup Constraints and Common Errors](#10-startup-constraints-and-common-errors)
11. [Real-World Pattern](#11-real-world-pattern)
12. [TypeScript Quick Reference](#12-typescript-quick-reference)

---

## 1. What is a Decorator?

A Fastify decorator is a named property attached to one of three targets:

- the Fastify instance
- every request object
- every reply object

Decorators are Fastify's official extension mechanism for shared utilities and
request-scoped state.

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.decorate('serviceName', 'billing-api')

app.get('/health', async () => {
  return { service: app.serviceName, ok: true }
})
```

---

## 2. Decorator Types

Fastify exposes three APIs:

```typescript
fastify.decorate(name, valueOrFunction, dependencies?)
fastify.decorateRequest(name, valueOrFunctionOrAccessor, dependencies?)
fastify.decorateReply(name, valueOrFunctionOrAccessor, dependencies?)
```

ASCII overview:

```text
+----------------------+-----------------------------+
| API                  | Target                      |
+----------------------+-----------------------------+
| decorate             | Fastify instance            |
| decorateRequest      | FastifyRequest prototype    |
| decorateReply        | FastifyReply prototype      |
+----------------------+-----------------------------+
```

---

## 3. Encapsulation and Scope

Decorators follow Fastify plugin encapsulation rules. A decorator added in a child
plugin is visible to that plugin and its descendants, but not to parent/sibling scopes.

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.register(async function pluginA (instance) {
  instance.decorate('fromA', true)

  instance.get('/a', async () => ({ hasFromA: instance.hasDecorator('fromA') }))
})

app.get('/root', async () => ({ hasFromA: app.hasDecorator('fromA') }))
```

Scope diagram (ASCII only):

```text
root
|
+-- pluginA
|   |
|   +-- decorate('fromA')
|
+-- pluginB

Visibility:
- pluginA sees fromA: yes
- children of pluginA see fromA: yes
- root sees fromA: no
- pluginB sees fromA: no
```

---

## 4. Instance Decorators with `decorate`

Use `decorate` for app-level helpers and shared services.

```typescript
import Fastify, { FastifyInstance } from 'fastify'

declare module 'fastify' {
  interface FastifyInstance {
    greet: (name: string) => string
  }
}

const app: FastifyInstance = Fastify()

app.decorate('greet', function (name: string): string {
  return `Hello ${name}`
})

app.get('/hello/:name', async function (request) {
  const params = request.params as { name: string }
  return { message: this.greet(params.name) }
})
```

Notes:

- `decorate` allows object/array values on the instance.
- duplicate names throw `FST_ERR_DEC_ALREADY_PRESENT`.
- function decorators returned via `getDecorator` are bound to the owning instance.

---

## 5. Request Decorators with `decorateRequest`

Use request decorators for data needed during one request lifecycle.

```typescript
import Fastify, { FastifyRequest } from 'fastify'

declare module 'fastify' {
  interface FastifyRequest {
    userId: string | null
  }
}

const app = Fastify()

app.decorateRequest('userId', null)

app.addHook('onRequest', async (request: FastifyRequest) => {
  request.userId = (request.headers['x-user-id'] as string | undefined) ?? null
})

app.get('/me', async (request) => {
  return { userId: request.userId }
})
```

---

## 6. Reply Decorators with `decorateReply`

Use reply decorators for reusable response helpers.

```typescript
import Fastify, { FastifyReply } from 'fastify'

declare module 'fastify' {
  interface FastifyReply {
    sendOk: (data: unknown) => FastifyReply
  }
}

const app = Fastify()

app.decorateReply('sendOk', function (this: FastifyReply, data: unknown) {
  return this.code(200).send({ ok: true, data })
})

app.get('/items', async (request, reply) => {
  return reply.sendOk([{ id: 1 }])
})
```

---

## 7. Reference Types and Safe Per-Request State

Important behavior from `lib/decorate.js` and `test/decorator.test.js`:

- `decorateRequest`/`decorateReply` reject plain object/array defaults.
- this throws `FST_ERR_DEC_REFERENCE_TYPE`.
- use `null` or a getter/setter pattern for mutable per-request data.

Unsafe (will throw):

```typescript
app.decorateRequest('session', {})
app.decorateReply('items', [])
```

Safe pattern (getter with per-request holder):

```typescript
import Fastify from 'fastify'

declare module 'fastify' {
  interface FastifyRequest {
    sessionHolder: Record<string, unknown> | undefined
    session: Record<string, unknown>
  }
}

const app = Fastify()

app.decorateRequest('sessionHolder')
app.decorateRequest('session', {
  getter (this: { sessionHolder?: Record<string, unknown> }) {
    this.sessionHolder ??= {}
    return this.sessionHolder
  }
})

app.get('/set', async (request) => {
  request.session.user = 'alice'
  return request.session
})
```

State model diagram:

```text
request #1 --> session getter --> holder object A
request #2 --> session getter --> holder object B

A and B are different objects.
```

---

## 8. Dependencies

Decorators can require other decorators with the third argument.

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.decorate('db', { query: async () => [] })
app.decorate('repo', {
  findUsers: async function () {
    return app.db.query()
  }
}, ['db'])
```

Rules:

- dependencies must be an array, else `FST_ERR_DEC_DEPENDENCY_INVALID_TYPE`.
- missing dependencies throw `FST_ERR_DEC_MISSING_DEPENDENCY`.

---

## 9. Introspection APIs (`has*Decorator`, `getDecorator`, `setDecorator`)

Fastify provides safe APIs to inspect and access decorators.

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.decorate('answer', 42)
app.decorateRequest('traceId', null)

app.get('/', async (request, reply) => {
  const hasAnswer = app.hasDecorator('answer')
  const answer = app.getDecorator<number>('answer')

  request.setDecorator('traceId', 'req-1')
  const traceId = request.getDecorator<string>('traceId')

  return { hasAnswer, answer, traceId }
})
```

Key points:

- `getDecorator(name)` throws `FST_ERR_DEC_UNDECLARED` if not found in scope.
- function values returned by `getDecorator` are context-bound.
- `setDecorator` updates an already-declared request/reply decorator; undeclared names throw.

---

## 10. Startup Constraints and Common Errors

Decorators must be declared before startup completes.

```typescript
import Fastify from 'fastify'

const app = Fastify()

await app.listen({ port: 3000 })

// throws FST_ERR_DEC_AFTER_START
app.decorate('tooLate', true)
```

Common decorator errors:

```text
FST_ERR_DEC_ALREADY_PRESENT
FST_ERR_DEC_MISSING_DEPENDENCY
FST_ERR_DEC_DEPENDENCY_INVALID_TYPE
FST_ERR_DEC_REFERENCE_TYPE
FST_ERR_DEC_AFTER_START
FST_ERR_DEC_UNDECLARED
```

Error flow (ASCII):

```text
decorate* call
    |
    +-- invalid state/type/dependency?
            |
            +-- yes -> throw Fastify error code
            |
            +-- no  -> decorator installed
```

---

## 11. Real-World Pattern

A small auth plugin combining all three decorator types.

```typescript
import fp from 'fastify-plugin'
import {
  FastifyPluginAsync,
  FastifyRequest,
  FastifyReply
} from 'fastify'

type User = { id: string; role: 'user' | 'admin' }

declare module 'fastify' {
  interface FastifyInstance {
    authenticate: (request: FastifyRequest, reply: FastifyReply) => Promise<void>
  }
  interface FastifyRequest {
    user: User | null
  }
  interface FastifyReply {
    sendUnauthorized: () => FastifyReply
  }
}

const authPlugin: FastifyPluginAsync = async (app) => {
  app.decorateRequest('user', null)

  app.decorateReply('sendUnauthorized', function (this: FastifyReply) {
    return this.code(401).send({ error: 'Unauthorized' })
  })

  app.decorate('authenticate', async (request, reply) => {
    const token = request.headers.authorization
    if (!token) {
      reply.sendUnauthorized()
      return
    }
    request.user = { id: token, role: 'user' }
  })
}

export default fp(authPlugin, { name: 'auth-plugin' })
```

---

## 12. TypeScript Quick Reference

Typed plugin signatures:

```typescript
import {
  FastifyPluginAsync,
  FastifyPluginCallback,
  FastifyPluginOptions
} from 'fastify'

interface Options extends FastifyPluginOptions {
  headerName: string
}

const asyncPlugin: FastifyPluginAsync<Options> = async (app, opts) => {
  app.decorate('headerName', opts.headerName)
}

const callbackPlugin: FastifyPluginCallback<Options> = (app, opts, done) => {
  app.decorate('headerName', opts.headerName)
  done()
}
```

Minimal augmentation template:

```typescript
declare module 'fastify' {
  interface FastifyInstance {
    myService: { ping: () => string }
  }
  interface FastifyRequest {
    userId: string | null
  }
  interface FastifyReply {
    sendOk: (data: unknown) => FastifyReply
  }
}
```

---

### Source grounding

This tutorial is aligned with behavior demonstrated in:

- `lib/decorate.js`
- `lib/errors.js`
- `test/decorator.test.js`

*End of tutorial.*
