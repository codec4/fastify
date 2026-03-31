# Fastify Lifecycle - Complete Tutorial

> Target audience: Developers familiar with Node.js / TypeScript who want to understand
> Fastify lifecycle behavior from internals and tests.

---

## Table of Contents

1. [What "Lifecycle" Means in Fastify](#1-what-lifecycle-means-in-fastify)
2. [Two Lifecycles You Must Know](#2-two-lifecycles-you-must-know)
3. [Boot Lifecycle (register -> ready -> listen -> close)](#3-boot-lifecycle-register---ready---listen---close)
4. [Request Lifecycle (onRequest -> onResponse)](#4-request-lifecycle-onrequest---onresponse)
5. [Lifecycle Hook Order and Signatures](#5-lifecycle-hook-order-and-signatures)
6. [Short-Circuiting, Errors, and Reply Flow](#6-short-circuiting-errors-and-reply-flow)
7. [Encapsulation and Hook Inheritance](#7-encapsulation-and-hook-inheritance)
8. [Route-Level Hooks vs Shared Hooks](#8-route-level-hooks-vs-shared-hooks)
9. [Timeout and Abort Paths](#9-timeout-and-abort-paths)
10. [Application Hooks: onReady, onListen, preClose, onClose](#10-application-hooks-onready-onlisten-preclose-onclose)
11. [TypeScript Patterns You Can Reuse](#11-typescript-patterns-you-can-reuse)
12. [End-to-End Example Plugin (Typed)](#12-end-to-end-example-plugin-typed)
13. [Common Pitfalls and Fast Fixes](#13-common-pitfalls-and-fast-fixes)

Related deep dives:

- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Streams — Complete Tutorial](./fastify-streams.tutorial.md)

---

## 1. What "Lifecycle" Means in Fastify

In Fastify, lifecycle means:

- The startup and shutdown phases of your app.
- The ordered request pipeline for each incoming HTTP request.

A lifecycle stage exists so you can place logic in the correct location (auth,
validation, serialization, metrics, cleanup) without mixing concerns in handlers.

Fastify implements this through:

- Hook arrays in [lib/hooks.js](../../lib/hooks.js).
- Request flow coordination in [lib/route.js](../../lib/route.js),
  [lib/handle-request.js](../../lib/handle-request.js), and [lib/reply.js](../../lib/reply.js).

---

## 2. Two Lifecycles You Must Know

There are two separate but related timelines.

### A) Application lifecycle (startup/shutdown)

Runs once per app boot cycle:

- plugin registration
- after/ready
- listen
- preClose/onClose

### B) Request lifecycle (per request)

Runs for every request that matches a route:

- onRequest
- preParsing
- preValidation
- preHandler
- handler
- preSerialization
- onSend
- onResponse
- plus side paths: onError, onTimeout, onRequestAbort

ASCII overview:

```text
App lifecycle:      create -> register plugins -> ready -> listen -> close
Request lifecycle:  recv request -> hooks -> handler -> send response -> complete
```

---

## 3. Boot Lifecycle (register -> ready -> listen -> close)

Fastify plugin loading is serial and deterministic (via avvio). Child plugin scopes
are created during registration unless encapsulation is skipped.

### Boot timeline

```text
Fastify()
  |
  +-- register(pluginA)
  |     +-- register(pluginA1)
  |
  +-- register(pluginB)
  |
  +-- ready()    -> runs onReady hooks once
  |
  +-- listen()   -> starts socket, then runs onListen hooks
  |
  +-- close()    -> preClose hooks, then onClose hooks
```

### Important guarantees

- `onReady` executes once, even if `ready()` is called multiple times.
- `onReady` runs before `listen()`.
- `onListen` is not executed by `ready()`.
- `onListen` errors are logged and hook execution continues.
- `onReady` errors fail readiness.

These behaviors are validated in:

- [test/hooks.on-ready.test.js](../test/hooks.on-ready.test.js)
- [test/hooks.on-listen.test.js](../test/hooks.on-listen.test.js)

---

## 4. Request Lifecycle (onRequest -> onResponse)

The core request flow is stitched together by:

- route match and setup in [lib/route.js](../../lib/route.js)
- preValidation/preHandler orchestration in [lib/handle-request.js](../../lib/handle-request.js)
- serialization/send/error/response hooks in [lib/reply.js](../../lib/reply.js)

### Happy path (no error)

```text
Incoming HTTP request
  |
  +-- onRequest
  |
  +-- preParsing
  |      (content-type parser reads body)
  |
  +-- preValidation
  |      (schema validation)
  |
  +-- preHandler
  |
  +-- route handler
  |
  +-- preSerialization   (if payload is not string/Buffer/stream/null)
  |
  +-- onSend
  |
  +-- write headers/body
  |
  +-- onResponse
```

### Error side path

```text
Any hook or handler throws/passes error
  |
  +-- onError
  |
  +-- error handler
  |
  +-- onSend
  |
  +-- onResponse
```

Notes:

- `onResponse` runs after the underlying response finishes.
- `onError` participates only on error flow.
- `reply.send()` inside `onError` is not allowed and throws `FST_ERR_SEND_INSIDE_ONERR`.

---

## 5. Lifecycle Hook Order and Signatures

Fastify supports callback or async style hooks, but do not mix styles in one function.

### Hook list (request lifecycle)

```text
1) onRequest
2) preParsing
3) preValidation
4) preHandler
5) handler
6) preSerialization
7) onSend
8) onResponse

Side hooks:
- onError
- onTimeout
- onRequestAbort
```

### TypeScript signatures (recommended)

```typescript
import {
  FastifyInstance,
  FastifyRequest,
  FastifyReply,
  HookHandlerDoneFunction
} from 'fastify'

function registerHooks(app: FastifyInstance): void {
  app.addHook('onRequest', async (request: FastifyRequest, reply: FastifyReply) => {
    // async style
  })

  app.addHook('preHandler', (
    request: FastifyRequest,
    reply: FastifyReply,
    done: HookHandlerDoneFunction
  ) => {
    // callback style
    done()
  })

  app.addHook('preSerialization', async (
    request: FastifyRequest,
    reply: FastifyReply,
    payload: unknown
  ) => {
    return payload
  })

  app.addHook('onSend', async (
    request: FastifyRequest,
    reply: FastifyReply,
    payload: string | Buffer | null
  ) => {
    return payload
  })

  app.addHook('onRequestAbort', async (request: FastifyRequest) => {
    // no reply in signature
  })
}
```

### Invalid async shape examples

These throw `FST_ERR_HOOK_INVALID_ASYNC_HANDLER`:

- async hook that still declares `done`
- wrong argument count for async route-level hooks

Validated in [test/hooks-async.test.js](../test/hooks-async.test.js).

---

## 6. Short-Circuiting, Errors, and Reply Flow

### Short-circuiting

Any early stage can terminate processing by sending a reply.

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.addHook('onRequest', async (request, reply) => {
  if (!request.headers.authorization) {
    return reply.code(401).send({ error: 'Unauthorized' })
  }
})

app.get('/private', async () => {
  return { ok: true }
})
```

If a hook sends a response early:

- later request hooks and handler do not run
- `onResponse` still runs when the response completes

### Error propagation

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.addHook('preHandler', async () => {
  throw new Error('boom')
})

app.addHook('onError', async (request, reply, error) => {
  request.log.error({ err: error }, 'hook failure')
  // DO NOT call reply.send() here
})

app.setErrorHandler(async (error, request, reply) => {
  return reply.code(500).send({ message: error.message })
})
```

Flow diagram:

```text
hook throws error
  |
  +-- onError hooks
  |
  +-- setErrorHandler/default error handler
  |
  +-- onSend hooks
  |
  +-- onResponse hooks
```

---

## 7. Encapsulation and Hook Inheritance

When a child plugin is created, Fastify clones parent hook arrays using `slice()`
(see `buildHooks` in [lib/hooks.js](../../lib/hooks.js)).

This gives you:

- Parent hooks are inherited by child scope.
- Child hooks do not leak to siblings or parent.

ASCII scope diagram:

```text
root
 |
 +-- pluginA
 |     +-- hook: onRequest(A)
 |     +-- route: /a
 |
 +-- pluginB
       +-- route: /b
```

`onRequest(A)` runs for `/a`, not for `/b`.

TypeScript example:

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.addHook('onRequest', async (request) => {
  ;(request as any).rootSeen = true
})

app.register(async (a) => {
  a.addHook('onRequest', async (request) => {
    ;(request as any).aSeen = true
  })

  a.get('/a', async (request) => {
    return { rootSeen: (request as any).rootSeen, aSeen: (request as any).aSeen }
  })
})

app.register(async (b) => {
  b.get('/b', async (request) => {
    return { rootSeen: (request as any).rootSeen, aSeen: (request as any).aSeen ?? false }
  })
})
```

---

## 8. Route-Level Hooks vs Shared Hooks

Route options can define per-route hooks (`onRequest`, `preHandler`, `onSend`, etc.).

Execution order is:

```text
shared hook(s) first -> route-level hook(s) next -> handler
```

And route-level arrays preserve declared order.

Validated in [test/route-hooks.test.js](../test/route-hooks.test.js).

TypeScript example:

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.addHook('preHandler', async (request) => {
  ;(request as any).order = ['shared']
})

app.get('/demo', {
  preHandler: [
    async (request) => {
      ;(request as any).order.push('route-1')
    },
    async (request) => {
      ;(request as any).order.push('route-2')
    }
  ]
}, async (request) => {
  return { order: (request as any).order }
})
```

Expected order:

```text
["shared", "route-1", "route-2"]
```

---

## 9. Timeout and Abort Paths

These are often confused because both are non-happy-path behaviors.

### onTimeout

Triggered when request/socket times out (for example with `connectionTimeout`).

```typescript
import Fastify from 'fastify'

const app = Fastify({ connectionTimeout: 500 })

app.addHook('onTimeout', async (request, reply) => {
  request.log.warn({ url: request.url }, 'request timed out')
})

app.get('/slow', async () => {
  // simulate hanging work
  await new Promise(() => {})
})
```

### onRequestAbort

Triggered when client aborts connection before completion.

```typescript
import Fastify from 'fastify'

const app = Fastify()

app.addHook('onRequestAbort', async (request) => {
  request.log.info({ id: request.id }, 'client disconnected')
})

app.get('/stream', async () => {
  await new Promise(resolve => setTimeout(resolve, 5000))
  return { done: true }
})
```

Behavior and error handling are covered in [test/hooks.test.js](../test/hooks.test.js).

---

## 10. Application Hooks: onReady, onListen, preClose, onClose

### Startup and shutdown sequence

```text
register plugins
  |
  +-- onReady (once)
  |
  +-- listen starts server
  |
  +-- onListen (after listening)
  |
  +-- close called
  |
  +-- preClose
  |
  +-- onClose
```

### TypeScript example

```typescript
import Fastify from 'fastify'

const app = Fastify({ logger: true })

app.addHook('onReady', async function () {
  this.log.info('all plugins loaded')
})

app.addHook('onListen', async function () {
  this.log.info('server is listening')
})

app.addHook('preClose', async function () {
  this.log.info('about to close')
})

app.addHook('onClose', async function () {
  this.log.info('closed')
})
```

Practical differences:

- `onReady`: can fail startup if it throws.
- `onListen`: errors are logged; remaining onListen hooks continue.
- `onClose`: run cleanup for resources.

---

## 11. TypeScript Patterns You Can Reuse

### Add typed request context in lifecycle hooks

```typescript
import Fastify, { FastifyRequest } from 'fastify'

declare module 'fastify' {
  interface FastifyRequest {
    authUserId: string | null
    startedAt: number
  }
}

const app = Fastify()

app.decorateRequest('authUserId', null)
app.decorateRequest('startedAt', 0)

app.addHook('onRequest', async (request: FastifyRequest) => {
  request.startedAt = Date.now()
  const auth = request.headers.authorization
  request.authUserId = auth ? auth.replace('Bearer ', '') : null
})
```

### Add typed reply helpers

```typescript
import Fastify, { FastifyReply } from 'fastify'

declare module 'fastify' {
  interface FastifyReply {
    sendOk: (value: unknown) => FastifyReply
  }
}

const app = Fastify()

app.decorateReply('sendOk', function (this: FastifyReply, value: unknown) {
  return this.code(200).send({ ok: true, data: value })
})
```

---

## 12. End-to-End Example Plugin (Typed)

This plugin records request timing and adds request IDs in lifecycle-safe places.

```typescript
import fp from 'fastify-plugin'
import {
  FastifyPluginAsync,
  FastifyRequest,
  FastifyReply
} from 'fastify'

declare module 'fastify' {
  interface FastifyRequest {
    startedAt: number
  }
}

const lifecyclePlugin: FastifyPluginAsync = async (app) => {
  app.decorateRequest('startedAt', 0)

  app.addHook('onRequest', async (request: FastifyRequest, reply: FastifyReply) => {
    request.startedAt = Date.now()
    reply.header('x-request-id', request.id)
  })

  app.addHook('onSend', async (request: FastifyRequest, reply: FastifyReply, payload) => {
    const elapsedMs = Date.now() - request.startedAt
    reply.header('x-response-time-ms', String(elapsedMs))
    return payload
  })

  app.addHook('onResponse', async (request: FastifyRequest, reply: FastifyReply) => {
    const elapsedMs = Date.now() - request.startedAt
    request.log.info({ elapsedMs, statusCode: reply.statusCode }, 'request complete')
  })

  app.addHook('onRequestAbort', async (request: FastifyRequest) => {
    request.log.warn('request aborted by client')
  })
}

export default fp(lifecyclePlugin, {
  name: 'lifecycle-plugin',
  fastify: '5.x'
})
```

Request path with this plugin:

```text
onRequest set start time + request id header
  -> handler work
  -> onSend set response time header
  -> response flush
  -> onResponse log final metrics
```

---

## 13. Common Pitfalls and Fast Fixes

1) Mixing async and callback in one hook

- Wrong:
  `async (req, reply, done) => { done() }`
- Right:
  `async (req, reply) => {}`
  or `(req, reply, done) => { done() }`

2) Calling `reply.send()` inside `onError`

- This throws `FST_ERR_SEND_INSIDE_ONERR`.
- Use `setErrorHandler` for response shaping.

3) Expecting `onListen` during `ready()`

- `ready()` triggers `onReady` only.
- `onListen` runs after successful `listen()`.

4) Assuming child hook leaks to siblings

- It does not. Encapsulation isolates child additions.

5) Forgetting route-level hook order

- Shared hooks run first, route-level hooks run after, in declared order.

---

## Final Mental Model

Use this compact model when deciding where logic belongs:

```text
Boot lifecycle:
  register -> onReady -> listen -> onListen -> close -> preClose/onClose

Request lifecycle:
  onRequest -> preParsing -> preValidation -> preHandler -> handler
  -> preSerialization -> onSend -> onResponse
  side paths: onError, onTimeout, onRequestAbort
```

If you keep this model in mind, you can place functionality correctly the first time,
with predictable behavior across plugins and routes.
