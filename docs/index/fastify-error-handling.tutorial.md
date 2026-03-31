# Fastify Error Handling — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's error handling system from first principles.

---

## Table of Contents

1. [What is Error Handling in Fastify?](#1-what-is-error-handling-in-fastify)
2. [Default Error Response Shape](#2-default-error-response-shape)
3. [Throwing Errors From Handlers](#3-throwing-errors-from-handlers)
   - 3.1 [Plain `Error`](#31-plain-error)
   - 3.2 [`http-errors` Compatible Objects](#32-http-errors-compatible-objects)
   - 3.3 [`@fastify/error` — Structured FST Errors](#33-fastifyerror--structured-fst-errors)
4. [Setting a Custom Error Handler](#4-setting-a-custom-error-handler)
   - 4.1 [Global Error Handler](#41-global-error-handler)
   - 4.2 [Scoped Error Handlers and Handler Chaining](#42-scoped-error-handlers-and-handler-chaining)
   - 4.3 [Async Error Handlers](#43-async-error-handlers)
   - 4.4 [Under the Hood — `buildErrorHandler` Internals](#44-under-the-hood--builderrorhandler-internals)
5. [The `onError` Hook](#5-the-onerror-hook)
   - 5.1 [Difference from `setErrorHandler`](#51-difference-from-seterrorhandler)
   - 5.2 [Under the Hood — `onErrorHook`](#52-under-the-hood--onerrorhook)
6. [Validation Errors (400 Bad Request)](#6-validation-errors-400-bad-request)
   - 6.1 [Default Validation Error Shape](#61-default-validation-error-shape)
   - 6.2 [`attachValidation` — Soft Mode](#62-attachvalidation--soft-mode)
   - 6.3 [Custom `schemaErrorFormatter`](#63-custom-schemaerrorformatter)
7. [Not Found and Bad URL Errors](#7-not-found-and-bad-url-errors)
   - 7.1 [`setNotFoundHandler`](#71-setnotfoundhandler)
   - 7.2 [Scoped 404 Handlers](#72-scoped-404-handlers)
8. [Error Propagation Through the Lifecycle](#8-error-propagation-through-the-lifecycle)
9. [Error Serialization](#9-error-serialization)
   - 9.1 [Default Serializer](#91-default-serializer)
   - 9.2 [Response Schema for Error Status Codes](#92-response-schema-for-error-status-codes)
   - 9.3 [Under the Hood — `fallbackErrorHandler`](#93-under-the-hood--fallbackerrorhandler)
10. [Fastify Built-in Error Codes Reference](#10-fastify-built-in-error-codes-reference)
11. [Real-World Patterns](#11-real-world-patterns)
    - 11.1 [Domain Error → HTTP Status Mapping](#111-domain-error--http-status-mapping)
    - 11.2 [Centralized Error Logging Plugin](#112-centralized-error-logging-plugin)
    - 11.3 [Error Normalization for External APIs](#113-error-normalization-for-external-apis)
12. [TypeScript Quick-Reference](#12-typescript-quick-reference)
13. [Common Pitfalls](#13-common-pitfalls)

Related deep dives:

- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Lifecycle — Complete Tutorial](./fastify-lifecycle.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)

---

## 1. What is Error Handling in Fastify?

Fastify centralises error handling so that any unhandled rejection, thrown exception, or
explicit `reply.send(error)` call is routed through a single point: the **error handler
chain**.

```
  Route handler / Hook throws or rejects
        |
        v
  onError hooks (observability, augmentation)
        |
        v
  Error handler (setErrorHandler -- can be scoped per plugin)
        |
        v
  Fallback error handler (last resort -- always succeeds)
        |
        v
  onSend hooks  ->  onResponse hooks
```

Three mechanisms are available:

| Mechanism | Purpose |
|---|---|
| `throw` / `reply.send(error)` | Signal an error from any handler or hook |
| `fastify.setErrorHandler(fn)` | Transform errors into HTTP responses |
| `fastify.addHook('onError', fn)` | Observe errors (e.g. logging, metrics) without owning the response |

---

## 2. Default Error Response Shape

When no custom error handler is set, Fastify serialises errors with
[`lib/error-serializer.js`](../../lib/error-serializer.js) into a fixed JSON structure:

```json
{
  "statusCode": 500,
  "code":       "FST_ERR_SOME_CODE",
  "error":      "Internal Server Error",
  "message":    "Something went wrong"
}
```

Field mapping:

| Field | Source |
|---|---|
| `statusCode` | `reply.statusCode` (derived from the error's `.statusCode` / `.status`, or 500) |
| `error` | `http.STATUS_CODES[statusCode]` — the standard HTTP reason phrase |
| `code` | `error.code` (present on `@fastify/error` instances and `http-errors`) |
| `message` | `error.message` |

`code` is omitted from the output when the error does not carry one.

---

## 3. Throwing Errors From Handlers

### 3.1 Plain `Error`

The simplest approach — throw a regular JavaScript `Error`. Fastify catches it, sets the
status code to **500**, and runs the error handler.

```javascript
fastify.get('/boom', async (request, reply) => {
  throw new Error('database offline')
  // → 500 { statusCode: 500, error: 'Internal Server Error', message: 'database offline' }
})
```

To control the status code without a library, attach `.statusCode` directly:

```javascript
fastify.get('/forbidden', async () => {
  const err = new Error('access denied')
  err.statusCode = 403
  throw err
  // → 403 { statusCode: 403, error: 'Forbidden', message: 'access denied' }
})
```

Fastify checks `error.statusCode` and `error.status` (in that order) and uses the
value when it is `>= 400`. See [`lib/error-handler.js` — `setErrorHeaders`](../../lib/error-handler.js).

### 3.2 `http-errors` Compatible Objects

Any object with a numeric `statusCode` / `status` property `>= 400` is treated as an
intentional HTTP error. The popular `http-errors` package meets this interface:

```javascript
import createError from 'http-errors'

fastify.get('/not-found', async () => {
  throw createError(404, 'user not found')
  // → 404 { statusCode: 404, error: 'Not Found', message: 'user not found' }
})
```

### 3.3 `@fastify/error` — Structured FST Errors

Fastify itself uses `@fastify/error` for all of its own internal errors. You can use the
same factory to define typed, reusable error classes with stable codes:

```javascript
import createError from '@fastify/error'

// createError(code, message [, statusCode [, Base]])
const NotFoundError    = createError('APP_NOT_FOUND',    'Resource %s was not found', 404)
const ConflictError    = createError('APP_CONFLICT',     'Resource %s already exists', 409)
const UnauthorizedError = createError('APP_UNAUTHORIZED', 'Missing or invalid token', 401)

fastify.get('/users/:id', async (request) => {
  const user = await db.findUser(request.params.id)
  if (!user) throw new NotFoundError(request.params.id)
  return user
})
```

The `%s` placeholders use `printf`-style formatting — arguments are passed to the
constructor:

```javascript
const err = new NotFoundError('user/42')
// err.message  → 'Resource user/42 was not found'
// err.code     → 'APP_NOT_FOUND'
// err.statusCode → 404
```

You can also subclass from a native error type (fourth argument):

```javascript
const ValidationFailure = createError(
  'APP_VALIDATION',
  'Validation failed: %s',
  422,
  TypeError                   // err instanceof TypeError === true
)
```

---

## 4. Setting a Custom Error Handler

### 4.1 Global Error Handler

`fastify.setErrorHandler(fn)` replaces the default error handler at the **root scope**.
The function receives `(error, request, reply)` and is responsible for sending the
response.

```javascript
import Fastify from 'fastify'

const fastify = Fastify()

fastify.setErrorHandler(function (error, request, reply) {
  request.log.error(error)                    // structured pino log

  const statusCode = error.statusCode ?? 500
  reply.status(statusCode).send({
    ok: false,
    code:    error.code    ?? 'INTERNAL_ERROR',
    message: error.message ?? 'An unexpected error occurred',
  })
})

fastify.get('/fail', async () => { throw new Error('oops') })
// → 500 { ok: false, code: 'INTERNAL_ERROR', message: 'oops' }
```

> **Important:** You **must** call `reply.send()` (or return a value) inside the error
> handler, otherwise the request will hang.

### 4.2 Scoped Error Handlers and Handler Chaining

Error handlers are **encapsulated**. A handler registered inside a plugin scope only
applies to routes defined in that same scope or its children.  When a route inside an
inner scope throws an error:

1. Fastify first looks for an error handler in the **innermost** scope that has a route.
2. If that handler rethrows, Fastify walks up the prototype chain to the next outer
   handler.
3. If no custom handler is found, the built-in `defaultErrorHandler` runs.

```javascript
fastify.register(async (instance) => {
  // This handler only covers routes in this plugin
  instance.setErrorHandler((error, request, reply) => {
    if (error.statusCode === 401) {
      return reply.status(401).send({ error: 'please log in' })
    }
    // Re-throw: walks up to the parent scope's handler
    throw error
  })

  instance.get('/secret', async () => {
    const err = new Error('not authenticated')
    err.statusCode = 401
    throw err
  })
})

// Root-scope handler handles everything that fell through
fastify.setErrorHandler((error, request, reply) => {
  reply.status(error.statusCode ?? 500).send({ message: error.message })
})
```

### 4.3 Async Error Handlers

Error handlers can be `async`. Return a value to send it as the response body, or call
`reply.send()` explicitly.

```javascript
fastify.setErrorHandler(async (error, request, reply) => {
  await metrics.increment('errors', { code: error.code })

  // returning a value calls reply.send() internally
  return reply.status(error.statusCode ?? 500).send({
    message: error.message
  })
})
```

If the async error handler itself throws, Fastify catches that too and walks up the
prototype chain to the next outer error handler.

### 4.4 Under the Hood — `buildErrorHandler` Internals

Fastify builds the error handler chain using JavaScript prototype inheritance
([`lib/error-handler.js`](../../lib/error-handler.js)):

```javascript
// Simplified from lib/error-handler.js
const rootErrorHandler = { func: defaultErrorHandler }

function buildErrorHandler (parent = rootErrorHandler, func) {
  if (!func) return parent
  const errorHandler = Object.create(parent)   // prototype = parent handler
  errorHandler.func = func
  return errorHandler
}
```

When `handleError(reply, error, cb)` runs:

```
reply[kReplyNextErrorHandler] = Object.getPrototypeOf(currentHandler)

  +- currentHandler.func(error, request, reply)
  |
  |  if it throws -> reply.send(err) again
  |                -> next iteration uses kReplyNextErrorHandler (the parent)
  |
  +- if kReplyNextErrorHandler === false -> fallbackErrorHandler (last resort)
```

The fallback always produces a valid HTTP response and never throws.

---

## 5. The `onError` Hook

### 5.1 Difference from `setErrorHandler`

| | `setErrorHandler` | `onError` hook |
|---|---|---|
| Owns the response | Yes — must call `reply.send()` | No — must **not** call `reply.send()` |
| Can change status code | Yes | Yes (via `reply.code()`) |
| Can suppress error | Yes (by sending a 2xx) | No |
| Runs per-scope | Yes (encapsulated) | Yes (encapsulated) |
| Runs after the handler | Yes | Yes — runs before `setErrorHandler` |

Use `onError` for **side effects**: logging, metrics, alerting.  Use `setErrorHandler`
to **shape** the response.

```javascript
// Observe, do not respond
fastify.addHook('onError', async (request, reply, error) => {
  await alerting.notify({
    code:    error.code,
    url:     request.url,
    traceId: request.headers['x-trace-id'],
  })
})
```

> **Rule:** Never call `reply.send()` inside `onError`. Fastify will throw
> `FST_ERR_SEND_INSIDE_ONERR` if you do.

```javascript
// ✗ Wrong — will throw FST_ERR_SEND_INSIDE_ONERR
fastify.addHook('onError', async (request, reply, error) => {
  reply.send({ my: 'response' })   // ← forbidden
})
```

You can, however, mutate `reply.statusCode`, add headers, or augment the error object:

```javascript
fastify.addHook('onError', async (request, reply, error) => {
  // Attach a request-id to every error response header for correlation
  reply.header('x-request-id', request.id)
  // Enrich the error for downstream handlers
  error.requestUrl = request.url
})
```

### 5.2 Under the Hood — `onErrorHook`

When `reply.send(error)` is called with an `Error` object, Fastify routes through
`onErrorHook` before reaching the error handler
(see [`lib/reply.js`](../../lib/reply.js#L149)):

```
reply.send(error)
  |
  +-- kReplyIsRunningOnErrorHook? -> throw FST_ERR_SEND_INSIDE_ONERR
  |
  +-- payload instanceof Error or kReplyIsError?
  |     +-- onErrorHook(reply, payload, onSendHook)
  |           |
  |           +-- run all onError hooks (async or callback)
  |           |
  |           +-- handleError(reply, error, cb)       <- error handler chain
  |                 |
  |                 +-- fallbackErrorHandler           <- always succeeds
  |
  +-- (not an error) -> proceed to serialization / onSend
```

---

## 6. Validation Errors (400 Bad Request)

### 6.1 Default Validation Error Shape

When an incoming request fails JSON Schema validation, Fastify throws
`FST_ERR_VALIDATION` (status 400) before the route handler runs.

```json
{
  "statusCode": 400,
  "code":       "FST_ERR_VALIDATION",
  "error":      "Bad Request",
  "message":    "body/name must be string"
}
```

The `message` is the AJV error message text (e.g. `body/name must be string`,
`querystring/limit must be >= 1`).

### 6.2 `attachValidation` — Soft Mode

Set `attachValidation: true` on a route to prevent Fastify from automatically sending
the 400 response. Instead, validation errors are attached to `request.validationError`
and the handler runs normally.

```javascript
fastify.post('/items', {
  attachValidation: true,
  schema: {
    body: {
      type: 'object',
      properties: { name: { type: 'string' } },
      required: ['name']
    }
  }
}, async (request, reply) => {
  if (request.validationError) {
    // Custom response for validation failures
    return reply.status(422).send({
      ok: false,
      details: request.validationError.validation
    })
  }
  return { ok: true, name: request.body.name }
})
```

### 6.3 Custom `schemaErrorFormatter`

`schemaErrorFormatter` is a server-level option that transforms the raw AJV error array
into a custom `Error` object. It runs synchronously.

```javascript
const fastify = Fastify({
  schemaErrorFormatter (errors, dataVar) {
    // errors: AJV ValidationError[]
    // dataVar: 'body' | 'params' | 'querystring' | 'headers'
    const details = errors.map(e => `${e.instancePath} ${e.message}`).join('; ')
    const err = new Error(`Validation failed on ${dataVar}: ${details}`)
    err.statusCode = 400
    err.code = 'APP_VALIDATION_ERROR'
    return err
  }
})
```

The error returned by `schemaErrorFormatter` is passed to the error handler like any
other error.

---

## 7. Not Found and Bad URL Errors

### 7.1 `setNotFoundHandler`

`setNotFoundHandler` registers a handler specifically for **404 Not Found** responses.
It accepts the same `(request, reply)` signature as a regular route handler.

```javascript
fastify.setNotFoundHandler((request, reply) => {
  reply.status(404).send({
    statusCode: 404,
    error:      'Not Found',
    message:    `Route ${request.method}:${request.url} not found`,
  })
})
```

You can also pass `preHandler` hooks as the first argument (e.g. for auth on 404):

```javascript
fastify.setNotFoundHandler({
  preHandler: fastify.auth([fastify.verifyJWT])
}, (request, reply) => {
  reply.status(404).send({ message: 'not found' })
})
```

### 7.2 Scoped 404 Handlers

Like `setErrorHandler`, `setNotFoundHandler` is **encapsulated** per plugin scope.
An inner plugin can register its own 404 handler that only fires for routes under its
prefix.

```javascript
fastify.register(async (instance) => {
  instance.setNotFoundHandler((request, reply) => {
    reply.status(404).send({ message: 'API endpoint not found', area: 'v2' })
  })

  instance.get('/users', async () => [])
}, { prefix: '/v2' })

// Root 404 handler applies everywhere else
fastify.setNotFoundHandler((request, reply) => {
  reply.status(404).send({ message: 'not found' })
})
```

---

## 8. Error Propagation Through the Lifecycle

The diagram below shows every path an error can take through the request lifecycle.

```
  Incoming HTTP Request
        |
        v
  onRequest hooks
        |  throws / done(err)
        +-----------------------------------------------------+
        v                                                     |
  preParsing hooks                                           |
        |  throws / done(err)                                 |
        +-----------------------------------------------------+
        v                                                     |
  Body parsing (content-type parser)                         |
        |  unsupported content-type -> 415                    |
        |  body too large           -> 413                    |
        |  invalid JSON             -> 400                    |
        +-----------------------------------------------------+
        v                                                     |
  preValidation hooks                                        |
        |  throws / done(err)                                 |
        +-----------------------------------------------------+
        v                                                     |
  JSON Schema validation                                     |
        |  fails -> FST_ERR_VALIDATION (400)                  |
        +-----------------------------------------------------+
        v                                                     |
  preHandler hooks                                           |
        |  throws / done(err)                                 |
        +-----------------------------------------------------+
        v                                                     |
  Route Handler                                              |
        |  throws / rejects                                   |
        +-----------------------------------------------------+
        v                                                     |
  preSerialization hooks                                     |
        |  throws / done(err)                                 |
        +-----------------------------------------------------+
                                                             |
        <----------------------------------------------------+
                    ALL error paths merge here
        |
        v
  onError hooks  (observe only, no reply.send)
        |
        v
  setErrorHandler chain
        |
        v (error -> serialized payload)
  onSend hooks
        |
        v
  onResponse hooks
```

Key rules:

- An error from any hook or handler jumps **directly** to the `onError` /
  `setErrorHandler` path. Subsequent non-error hooks in the happy path are skipped.
- `onSend` and `onResponse` **always** run, even on the error path.
- The `onError` hook runs after the error handler starts processing, not before.

---

## 9. Error Serialization

### 9.1 Default Serializer

The default error serializer is generated by `build/build-error-serializer.js` using
`fast-json-stringify` — it is a compiled, allocation-efficient function that outputs:

```
{ statusCode, code, error, message }
```

It is faster than `JSON.stringify` because it emits field names as string literals
and uses typed serialization for each field.

### 9.2 Response Schema for Error Status Codes

When a route defines a `schema.response` entry for an error status code, Fastify uses
the compiled schema serializer for that status code instead of the default error
serializer. This lets you produce custom error shapes for known error cases.

```javascript
fastify.get('/items/:id', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: { id: { type: 'integer' }, name: { type: 'string' } }
      },
      404: {
        type: 'object',
        properties: {
          statusCode: { type: 'integer' },
          message:    { type: 'string' },
          hint:       { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => {
  const item = await db.findItem(request.params.id)
  if (!item) {
    return reply.status(404).send({
      statusCode: 404,
      message:    'item not found',
      hint:       'check the id parameter'
    })
  }
  return item
})
```

### 9.3 Under the Hood — `fallbackErrorHandler`

The `fallbackErrorHandler` in [`lib/error-handler.js`](../../lib/error-handler.js) is the
ultimate safety net. It runs when:

- The error handler chain is exhausted (`kReplyNextErrorHandler === false`), or
- No `func` is found on the current error handler object.

It follows this decision tree:

```javascript
// Simplified
function fallbackErrorHandler (error, reply, cb) {
  const statusCode = reply.statusCode

  // Does the route schema define a serializer for this status code?
  const serializerFn = getSchemaSerializer(context, statusCode, contentType)

  let payload
  if (serializerFn === false) {
    // No schema serializer → use the compiled default error serializer
    payload = serializeError({
      error:      STATUS_CODES[statusCode],
      code:       error.code,
      message:    error.message,
      statusCode
    })
  } else {
    // Use the route-level schema serializer
    payload = serializerFn(/* error with extra props */)
  }

  cb(reply, payload)
}
```

If even the schema serializer throws (e.g. a bug in a custom compiler), Fastify catches
that too and falls back to `FST_ERR_FAILED_ERROR_SERIALIZATION` with a 500 response.

---

## 10. Fastify Built-in Error Codes Reference

The table below lists the most common errors from [`lib/errors.js`](../../lib/errors.js):

| Code | Status | When |
|---|---|---|
| `FST_ERR_NOT_FOUND` | 404 | No route matched |
| `FST_ERR_VALIDATION` | 400 | JSON Schema validation failed |
| `FST_ERR_CTP_BODY_TOO_LARGE` | 413 | Body exceeds `bodyLimit` |
| `FST_ERR_CTP_INVALID_MEDIA_TYPE` | 415 | Content-Type has no registered parser |
| `FST_ERR_CTP_INVALID_CONTENT_LENGTH` | 400 | Content-Length mismatch |
| `FST_ERR_CTP_EMPTY_JSON_BODY` | 400 | Empty body with `application/json` |
| `FST_ERR_CTP_INVALID_JSON_BODY` | 400 | Malformed JSON body |
| `FST_ERR_REP_ALREADY_SENT` | 500 | `reply.send()` called twice |
| `FST_ERR_SEND_INSIDE_ONERR` | 500 | `reply.send()` called inside `onError` |
| `FST_ERR_FAILED_ERROR_SERIALIZATION` | 500 | Error serializer itself threw |
| `FST_ERR_HOOK_TIMEOUT` | — | Hook callback timed out (missing `done()`) |
| `FST_ERR_HANDLER_TIMEOUT` | — | Route handler timed out |
| `FST_ERR_BAD_STATUS_CODE` | 500 | Invalid HTTP status code passed to `reply.code()` |
| `FST_ERR_BAD_URL` | — | Router received a bad URL |
| `FST_ERR_ERROR_HANDLER_ALREADY_SET` | 500 | Duplicate `setErrorHandler` in same scope without `allowErrorHandlerOverride: true` |

All of these are instances created by `@fastify/error` and carry `.code`, `.statusCode`,
and `.message`.

---

## 11. Real-World Patterns

### 11.1 Domain Error → HTTP Status Mapping

Define a catalogue of domain errors and map them to HTTP status codes in a single
central error handler, keeping route handlers free of HTTP concerns.

```javascript
// errors/domain.js
import createError from '@fastify/error'

export const NotFoundError    = createError('APP_NOT_FOUND',    '%s not found', 404)
export const ConflictError    = createError('APP_CONFLICT',     '%s already exists', 409)
export const ForbiddenError   = createError('APP_FORBIDDEN',    'Access denied: %s', 403)
export const UnprocessableError = createError('APP_UNPROCESSABLE', '%s', 422)
```

```javascript
// plugins/error-handler.js
import fp from 'fastify-plugin'
import { NotFoundError, ConflictError, ForbiddenError } from '../errors/domain.js'

export default fp(async (fastify) => {
  fastify.setErrorHandler((error, request, reply) => {
    // Known domain errors — already carry the right statusCode
    if (error.code?.startsWith('APP_')) {
      return reply.status(error.statusCode).send({
        ok: false,
        code:    error.code,
        message: error.message,
      })
    }

    // Fastify built-in errors (validation, body limit, etc.)
    if (error.code?.startsWith('FST_')) {
      return reply.status(error.statusCode ?? 400).send({
        ok: false,
        code:    error.code,
        message: error.message,
      })
    }

    // Unexpected errors
    request.log.error(error)
    reply.status(500).send({ ok: false, message: 'Internal Server Error' })
  })
})
```

### 11.2 Centralized Error Logging Plugin

Separate the concern of logging from the concern of shaping the response using
`onError`:

```javascript
import fp from 'fastify-plugin'

export default fp(async (fastify) => {
  fastify.addHook('onError', async (request, reply, error) => {
    if (error.statusCode == null || error.statusCode >= 500) {
      // Unexpected — emit to an external error tracker
      await errorTracker.capture(error, {
        url:    request.url,
        method: request.method,
        userId: request.user?.id,
      })
    }
  })
})
```

### 11.3 Error Normalization for External APIs

When calling external services, wrap unknown errors with domain errors before they
propagate back through Fastify:

```javascript
fastify.get('/orders/:id', async (request) => {
  let order
  try {
    order = await paymentsService.getOrder(request.params.id)
  } catch (err) {
    if (err.response?.status === 404) {
      throw new NotFoundError(`order/${request.params.id}`)
    }
    if (err.response?.status === 503) {
      const wrapped = new Error('payments service unavailable')
      wrapped.statusCode = 503
      throw wrapped
    }
    throw err  // let the global handler deal with truly unexpected errors
  }
  return order
})
```

---

## 12. TypeScript Quick-Reference

```typescript
import Fastify, {
  FastifyInstance,
  FastifyError,
  FastifyRequest,
  FastifyReply,
  FastifyPluginAsync,
} from 'fastify'
import createError from '@fastify/error'

// Typed domain error factory
const NotFoundError = createError<[resource: string]>(
  'APP_NOT_FOUND',
  '%s not found',
  404
)

// Typed error handler
const fastify: FastifyInstance = Fastify()

fastify.setErrorHandler(
  (error: FastifyError, request: FastifyRequest, reply: FastifyReply) => {
    const statusCode = error.statusCode ?? 500
    reply.status(statusCode).send({
      ok:      false,
      code:    error.code,
      message: error.message,
    })
  }
)

// Typed not-found handler
fastify.setNotFoundHandler(
  (request: FastifyRequest, reply: FastifyReply) => {
    reply.status(404).send({ ok: false, message: 'not found' })
  }
)

// Typed onError hook
fastify.addHook(
  'onError',
  async (request: FastifyRequest, reply: FastifyReply, error: FastifyError) => {
    if ((error.statusCode ?? 500) >= 500) {
      request.log.error({ err: error }, 'unexpected error')
    }
  }
)

// Scoped error handler inside a typed plugin
const usersPlugin: FastifyPluginAsync = async (instance) => {
  instance.setErrorHandler(
    (error: FastifyError, request: FastifyRequest, reply: FastifyReply) => {
      if (error.code === 'APP_NOT_FOUND') {
        return reply.status(404).send({ message: error.message })
      }
      throw error  // bubble up to parent scope
    }
  )

  instance.get('/:id', async (request) => {
    const { id } = request.params as { id: string }
    const user = await db.find(id)
    if (!user) throw new NotFoundError(`user/${id}`)
    return user
  })
}
```

---

## 13. Common Pitfalls

### Forgetting to send a response in `setErrorHandler`

```javascript
// ✗ Wrong — request hangs
fastify.setErrorHandler((error, request, reply) => {
  console.error(error)        // logs it but never sends a response
})

// ✓ Correct
fastify.setErrorHandler((error, request, reply) => {
  console.error(error)
  reply.status(500).send({ message: error.message })
})
```

### Calling `reply.send()` inside `onError`

```javascript
// ✗ Wrong — throws FST_ERR_SEND_INSIDE_ONERR
fastify.addHook('onError', async (request, reply, error) => {
  reply.send({ ok: false })   // ← forbidden
})
```

### Using `reply.send()` with an error after the response is already sent

If `onSend` or a previous `reply.send()` call completed, calling `reply.send` again
throws `FST_ERR_REP_ALREADY_SENT`. Guard using `reply.sent`:

```javascript
fastify.addHook('onSend', async (request, reply, payload) => {
  if (!reply.sent) {
    // safe to modify headers
  }
  return payload
})
```

### Async error handler throwing synchronously

If an async error handler itself throws before returning a promise (synchronous throw
inside an async function before the first `await`), Fastify still catches it because the
handler wraps the result with `wrapThenable`.  But be aware: if you `return` a plain
value (non-promise) from a synchronous `function` error handler, Fastify calls
`reply.send(result)` for you — so returning `undefined` is a silent no-op that hangs.

### Scope confusion with `setErrorHandler`

An error handler registered at the **child plugin** scope does not cover routes on the
**parent** scope. Register global handlers on the root instance before any plugins.

```javascript
// ✗ Wrong — the handler is scoped to the plugin, not the root
fastify.register(async (instance) => {
  instance.setErrorHandler(globalHandler)   // only affects routes inside this plugin
})

// ✓ Correct — register on the root fastify instance
fastify.setErrorHandler(globalHandler)
fastify.register(myPlugin)
```

### `attachValidation` still runs the handler on invalid input

When `attachValidation: true`, the handler always executes regardless of whether
validation passed. Always check `request.validationError` before processing
`request.body`.
