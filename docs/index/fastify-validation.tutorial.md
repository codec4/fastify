# Fastify Validation — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> Fastify's validation and serialization system from first principles.

---

## Table of Contents

1. [What is Validation?](#1-what-is-validation)
2. [Schema Slots](#2-schema-slots)
   - 2.1 [params](#21-params)
   - 2.2 [body](#22-body)
   - 2.3 [querystring (and `query` alias)](#23-querystring-and-query-alias)
   - 2.4 [headers](#24-headers)
   - 2.5 [response](#25-response)
   - 2.6 [Under the Hood — Schema Slot Internals](#26-under-the-hood--schema-slot-internals)
3. [The Compile Pipeline](#3-the-compile-pipeline)
   - 3.1 [addSchema and $ref](#31-addschema-and-ref)
   - 3.2 [fluent-json-schema](#32-fluent-json-schema)
   - 3.3 [Under the Hood — Compile Pipeline Internals](#33-under-the-hood--compile-pipeline-internals)
4. [Input Validation at Request Time](#4-input-validation-at-request-time)
   - 4.1 [Default Error Shape](#41-default-error-shape)
   - 4.2 [attachValidation — Soft Mode](#42-attachvalidation--soft-mode)
   - 4.3 [Under the Hood — Request Validation Internals](#43-under-the-hood--request-validation-internals)
5. [Output Serialization](#5-output-serialization)
   - 5.1 [Status-Code Keys](#51-status-code-keys)
   - 5.2 [Field Stripping](#52-field-stripping)
   - 5.3 [Under the Hood — Serialization Internals](#53-under-the-hood--serialization-internals)
6. [Custom Validators](#6-custom-validators)
   - 6.1 [TypeBox](#61-typebox)
   - 6.2 [Zod](#62-zod)
   - 6.3 [Raw AJV with coerceTypes](#63-raw-ajv-with-coercetypes)
   - 6.4 [Server-Level vs Route-Level vs compilersFactory](#64-server-level-vs-route-level-vs-compilersfactory)
   - 6.5 [Under the Hood — Custom Compiler Internals](#65-under-the-hood--custom-compiler-internals)
7. [Error Handling and Formatting](#7-error-handling-and-formatting)
   - 7.1 [schemaErrorFormatter](#71-schemaerrorformatter)
   - 7.2 [Error Code Reference](#72-error-code-reference)
8. [Advanced Patterns](#8-advanced-patterns)
   - 8.1 [Content-Type–Scoped Body Schemas (OpenAPI 3)](#81-content-typescoped-body-schemas-openapi-3)
   - 8.2 [Content-Type–Scoped Response Schemas](#82-content-typescoped-response-schemas)
   - 8.3 [reply.compileSerializationSchema](#83-replycompileserializationschema)
   - 8.4 [reply.serializeInput](#84-replyserializeinput)
9. [Performance Deep Dive](#9-performance-deep-dive)
10. [Pros, Cons, and Trade-offs](#10-pros-cons-and-trade-offs)
11. [TypeScript Quick-Reference](#11-typescript-quick-reference)

Related deep dives:

- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Plugins — Complete Tutorial](./fastify-plugins.tutorial.md)
- [Fastify Lifecycle — Complete Tutorial](./fastify-lifecycle.tutorial.md)

---

## 1. What is Validation?

Fastify's validation system is split into **two entirely separate sub-systems** that both
operate on schemas, but at different lifecycle stages:

| Sub-system | Tool | Stage | Purpose |
|---|---|---|---|
| **Input validation** | AJV v8 (via `@fastify/ajv-compiler`) | Between `preValidation` and `preHandler` hooks | Reject bad requests before they reach your handler |
| **Output serialization** | `fast-json-stringify` (via `@fastify/fast-json-stringify-compiler`) | Between `preSerialization` and `onSend` hooks | Encode response payloads and strip undeclared fields |

The defining performance characteristic of both sub-systems is **compile-once, run-many**.
Schema compilation — which involves building optimised JavaScript closures from JSON Schema
definitions — happens exactly once during server startup, inside Fastify's `preReady` phase.
Zero compilation work occurs at request time.

```
  Incoming request
       |
       v
  onRequest hooks
       |
       v
  preParsing hooks
       |
       v
  body parsing (content-type parsers)
       |
       v
  preValidation hooks  <-- can modify request.body/params/query/headers
       |
       v
  +---------------------------------------------+
  |  INPUT VALIDATION  (AJV compiled function)  |  <- this tutorial
  |  params -> body -> querystring -> headers   |
  +---------------------------------------------+
       |  if fails -> 400 / attachValidation
       v
  preHandler hooks
       |
       v
  route handler
       |
       v
  preSerialization hooks  <-- can transform payload
       |
       v
  +----------------------------------------------+
  |  OUTPUT SERIALIZATION  (fast-json-stringify) |  <- this tutorial
  |  status-code schema lookup -> encode         |
  +----------------------------------------------+
       |
       v
  onSend hooks  <-- payload is already a string/Buffer
       |
       v
  Response sent
```

> **Rule:** Validation and serialization are opt-in per route. A route without a `schema`
> option uses `JSON.stringify` for the response and skips all input validation.

---

## 2. Schema Slots

A route can carry a `schema` object with up to five named slots:

```typescript
import Fastify from 'fastify'

const fastify = Fastify()

fastify.post<{
  Params: { id: string }
  Body: { name: string; age: number }
  Querystring: { verbose?: string }
  Headers: { 'x-api-key': string }
  Reply: { id: string; name: string }
}>('/users/:id', {
  schema: {
    params: {
      type: 'object',
      properties: {
        id: { type: 'string', pattern: '^[0-9]+$' }
      },
      required: ['id']
    },
    body: {
      type: 'object',
      properties: {
        name: { type: 'string', minLength: 1 },
        age:  { type: 'integer', minimum: 0 }
      },
      required: ['name', 'age']
    },
    querystring: {
      type: 'object',
      properties: {
        verbose: { type: 'string', enum: ['true', 'false'] }
      }
    },
    headers: {
      type: 'object',
      properties: {
        'x-api-key': { type: 'string', minLength: 32 }
      },
      required: ['x-api-key']
    },
    response: {
      200: {
        type: 'object',
        properties: {
          id:   { type: 'string' },
          name: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => {
  return { id: request.params.id, name: request.body.name }
})
```

### 2.1 params

Validates route path parameters (`:id`, `:userId`, etc.). The value passed to the validator
is `request.params`, which is always an object even for single-param routes.

```typescript
fastify.get('/items/:id/:version', {
  schema: {
    params: {
      type: 'object',
      properties: {
        id:      { type: 'integer' },   // AJV coerces ':42' string → 42 integer by default
        version: { type: 'string', pattern: '^v\\d+$' }
      },
      required: ['id', 'version']
    }
  }
}, async (request) => {
  // request.params.id is a string from the URL; coercion depends on AJV config
  return request.params
})
```

### 2.2 body

Validates the parsed request body. Only applies to methods that carry a body (`POST`, `PUT`,
`PATCH`, `DELETE` with body, etc.). Fastify passes `request.body` to the validator; if no
body was sent, `request.body` is `undefined` and the validator receives `null`.

```typescript
fastify.post('/articles', {
  schema: {
    body: {
      type: 'object',
      properties: {
        title:   { type: 'string', minLength: 1, maxLength: 200 },
        content: { type: 'string' },
        tags:    { type: 'array', items: { type: 'string' }, maxItems: 10 }
      },
      required: ['title', 'content'],
      additionalProperties: false   // reject unknown keys at input
    }
  }
}, async (request) => {
  return { ok: true }
})
```

### 2.3 querystring (and `query` alias)

Validates `request.query` (parsed query string object). The slot name is `querystring` but
Fastify also accepts `query` as an alias for convenience.

```typescript
fastify.get('/search', {
  schema: {
    querystring: {           // or: query: { ... }
      type: 'object',
      properties: {
        q:        { type: 'string', minLength: 1 },
        page:     { type: 'integer', minimum: 1, default: 1 },
        pageSize: { type: 'integer', minimum: 1, maximum: 100, default: 20 }
      },
      required: ['q']
    }
  }
}, async (request) => {
  // request.query.page is a string from URL; default injection requires AJV useDefaults
  return { q: request.query.q }
})
```

> **Rule:** Using both `query` and `querystring` in the same route schema throws
> `FST_ERR_SCH_DUPLICATE` at route registration time — not silently at runtime.

### 2.4 headers

Validates `request.headers`. Header property names in the schema must be lowercase because
HTTP/1.1 headers are case-insensitive and Fastify normalizes them to lowercase before
validation (see §2.6 Under the Hood).

```typescript
fastify.post('/secure', {
  schema: {
    headers: {
      type: 'object',
      properties: {
        'x-api-key':       { type: 'string', minLength: 32 },
        'content-type':    { type: 'string' },
        'x-request-id':   { type: 'string', format: 'uuid' }
      },
      required: ['x-api-key']
    }
  }
}, async (request) => {
  return { ok: true }
})
```

### 2.5 response

Drives output serialization via `fast-json-stringify`. Each key is an HTTP status code
pattern. The schema controls which fields appear in the response body:

```typescript
fastify.get('/products/:id', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          id:    { type: 'string' },
          name:  { type: 'string' },
          price: { type: 'number' }
          // NOTE: fields NOT listed here are stripped from the response
        }
      },
      404: {
        type: 'object',
        properties: {
          message: { type: 'string' }
        }
      },
      '5xx': {
        type: 'object',
        properties: {
          error: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => {
  const product = await db.find(request.params.id)
  if (!product) {
    return reply.status(404).send({ message: 'Not found' })
  }
  return product   // extra fields like `createdAt` are silently stripped
})
```

Valid status-code key formats:

| Format | Example | Matches |
|---|---|---|
| Exact 3-digit | `200`, `404` | That specific status code |
| Generic pattern | `2xx`, `4xx`, `5xx` | Any code in that range |
| Default | `default` | Any status code with no other match |

### 2.6 Under the Hood — Schema Slot Internals

**Header Key Normalization**

Fastify normalizes header schema keys to lowercase in
[lib/validation.js](../../lib/validation.js) before compiling the AJV validator. This means
writing `X-Api-Key` in your schema works — Fastify rewrites it to `x-api-key`:

```javascript
// lib/validation.js (simplified)
if (headers.properties) {
  headersSchemaLowerCase.properties = {}
  Object.keys(headers.properties).forEach(k => {
    headersSchemaLowerCase.properties[k.toLowerCase()] = headers.properties[k]
  })
  if (headers.required) {
    headersSchemaLowerCase.required = headers.required.map(k => k.toLowerCase())
  }
}
```

This normalization is **skipped** when a custom `validatorCompiler` is in use
(`isCustom === true`), so Joi/Zod/TypeBox receive the schema verbatim.

**Schema Slot Symbols**

Compiled validators are stored on the route context object under V8-optimised Symbol keys
(defined in [lib/symbols.js](../../lib/symbols.js)):

| Symbol | Purpose |
|---|---|
| `kSchemaParams` (`'params-schema'`) | Compiled AJV function for `params` |
| `kSchemaBody` (`'body-schema'`) | Compiled AJV function (or MIME map) for `body` |
| `kSchemaQuerystring` (`'querystring-schema'`) | Compiled AJV function for `querystring` |
| `kSchemaHeaders` (`'headers-schema'`) | Compiled AJV function for `headers` |
| `kSchemaResponse` (`'response-schema'`) | Map: status-code string → `fast-json-stringify` function |

These Symbols live on the route's `Context` object; accessing them is a plain property
read — the fastest possible JavaScript dispatch path.

**Empty Schema Warning**

If you write `schema: { body: null }` or `schema: { body: {} }` — present but falsy/empty
— Fastify emits `FSTWRN001` process warning at startup so you discover the mistake early
rather than silently skipping validation.

---

## 3. The Compile Pipeline

Understanding when schemas are compiled prevents several common production issues.

All schema compilation happens during the `preReady` phase — after `fastify.listen()` (or
`fastify.ready()`) is called but before any request is accepted. The pipeline per route is:

```
  fastify.register(plugin)  or  fastify.post('/route', { schema }, handler)
       |
       v
  normalizeSchema()
  +-- alias: query -> querystring
  +-- unwrap: fluent-json-schema .valueOf()
  +-- validate: content[mediaType] has .schema
  +-- stamp: kSchemaVisited = true  (idempotent guard)
       |
       v  (preReady phase)
  SchemaController.setupValidator()
  +-- if no compiler yet, or addedSchemas dirty flag set:
  |     AJV instance = ajvCompilerFactory(allSchemas, ajvOpts)
  +-- validatorCompiler = ajvInstance.compile
       |
       v
  compileSchemasForValidation(context, validatorCompiler)
  +-- compileHeadersSchema   -> context[kSchemaHeaders]
  +-- compileParamsSchema    -> context[kSchemaParams]
  +-- compileQuerySchema     -> context[kSchemaQuerystring]
  +-- compileBodySchema      -> context[kSchemaBody]  (or MIME type map)
       |
       v
  SchemaController.setupSerializer()
       |
       v
  compileSchemasForSerialization(context, serializerCompiler)
  +-- for each response status key -> context[kSchemaResponse][statusCode]
```

### 3.1 addSchema and $ref

`fastify.addSchema()` registers a shared schema with a `$id` that any route in the same
scope (or child scopes) can reference via `$ref`:

```typescript
import Fastify from 'fastify'

const fastify = Fastify()

// Register shared schemas once
fastify.addSchema({
  $id: 'Address',
  type: 'object',
  properties: {
    street:  { type: 'string' },
    city:    { type: 'string' },
    country: { type: 'string', minLength: 2, maxLength: 2 }
  },
  required: ['street', 'city', 'country']
})

fastify.addSchema({
  $id: 'User',
  type: 'object',
  properties: {
    id:      { type: 'string', format: 'uuid' },
    name:    { type: 'string' },
    address: { $ref: 'Address' }   // reference by $id
  },
  required: ['id', 'name']
})

fastify.post('/users', {
  schema: {
    body: { $ref: 'User' },        // use shared schema directly
    response: {
      201: { $ref: 'User' }
    }
  }
}, async (request, reply) => {
  return reply.status(201).send(request.body)
})
```

> **Rule:** `addSchema()` requires a `$id` property — omitting it throws
> `FST_ERR_SCH_MISSING_ID`. Duplicate `$id` values in the same scope throw
> `FST_ERR_SCH_ALREADY_PRESENT`.

**Schema scope inheritance:** Schemas registered with `addSchema()` in a parent scope are
visible in all child scopes, but child schemas do not leak upward.

```
  root scope -- addSchema({ $id: 'GlobalUser' })
  |
  +-- /api/v1 scope -- addSchema({ $id: 'V1Config' })
  |   |
  |   +-- /api/v1/users  <- can use 'GlobalUser' AND 'V1Config'
  |
  +-- /api/v2 scope
      +-- /api/v2/users  <- can use 'GlobalUser', but NOT 'V1Config'
```

### 3.2 fluent-json-schema

`fluent-json-schema` objects are treated as first-class citizens — Fastify detects them
automatically and calls `.valueOf()` to extract the plain JSON Schema object before
compilation:

```typescript
import Fastify from 'fastify'
import S from 'fluent-json-schema'

const fastify = Fastify()

fastify.addSchema(
  S.object()
    .id('https://myapp.com/schemas/address')
    .prop('street', S.string().required())
    .prop('city',   S.string().required())
)

fastify.post('/checkout', {
  schema: {
    body: S.object()
      .prop('email',   S.string().format('email').required())
      .prop('address', S.ref('https://myapp.com/schemas/address#')),
    response: {
      200: S.object()
        .prop('orderId', S.string())
    }
  }
}, async (request) => {
  return { orderId: 'ord_123' }
})
```

### 3.3 Under the Hood — Compile Pipeline Internals

**`@fastify/ajv-compiler` Factory**

The default validator builder is the `@fastify/ajv-compiler` package. It exposes a factory
function `buildAjvValidator(schemas, ajvOptions)` that constructs a single AJV v8 instance
loaded with all registered shared schemas, then returns it as the `validatorCompiler`
function that Fastify calls per route:

```javascript
// lib/schema-controller.js (simplified)
setupValidator (serverOptions) {
  const isReady = this.validatorCompiler !== undefined && !this.addedSchemas
  if (isReady) return

  this.validatorCompiler = this.getValidatorBuilder()(
    this.schemaBucket.getSchemas(),  // all addSchema() entries
    serverOptions.ajv                // forwarded to AJV constructor
  )
}
```

**`addedSchemas` Dirty Flag**

Every `fastify.addSchema()` call sets `this.addedSchemas = true` on the
`SchemaController`. `setupValidator()` detects this flag and **rebuilds** the AJV instance
so the new schemas are included. Once rebuilt the flag resets. This means calling
`addSchema()` inside a plugin — which is the only valid place — works correctly because
all plugins complete before `preReady` triggers the compile pass.

**`Schemas` Bucket and `rfdc` Deep Clone**

[lib/schemas.js](../../lib/schemas.js) stores each `$id`-keyed schema as a **deep clone**
(via `rfdc`) of the object you passed. This prevents mutation of your original schema
objects from affecting the compiled validator or other routes referencing the same schema.

**Schema Inheritance Across Plugin Scopes**

Each plugin scope gets a new `SchemaController` via
`buildSchemaController(parentCtrl, opts)`. The child controller is initialised with the
parent's bucket contents — inheriting all schemas — but has its own bucket for new
`addSchema()` calls. Any child `addSchema()` dirties only the child's `addedSchemas` flag,
not the parent's.

---

## 4. Input Validation at Request Time

Once schemas are compiled at startup, validation at request time is a pure function call
against the pre-compiled AJV closure. The route context holds a reference to that closure
under the Symbol key; Fastify calls it, checks the return value, and either passes the
request to the next hook or sends a 400 response.

The four input slots are validated in this fixed order:

```
  params  ->  body  ->  querystring  ->  headers
```

### 4.1 Default Error Shape

When validation fails, Fastify sends a `400 Bad Request` with code `FST_ERR_VALIDATION`:

```json
{
  "statusCode": 400,
  "code": "FST_ERR_VALIDATION",
  "error": "Bad Request",
  "message": "body/age must be integer"
}
```

The `message` field is the AJV error message for the first failing constraint. You can
customise this — see §7.

### 4.2 attachValidation — Soft Mode

Setting `attachValidation: true` on a route disables automatic 400 responses. Instead,
the validation error is stored on `request.validationError` and the handler is still
called. This is useful for building custom error responses or for graceful degradation:

```typescript
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

const fastify = Fastify()

fastify.post('/lenient', {
  attachValidation: true,
  schema: {
    body: {
      type: 'object',
      properties: {
        email: { type: 'string', format: 'email' },
        age:   { type: 'integer', minimum: 0 }
      },
      required: ['email']
    }
  }
}, async (request: FastifyRequest, reply: FastifyReply) => {
  if (request.validationError) {
    // Decide what to do with the invalid input
    return reply.status(422).send({
      status: 'validation_failed',
      details: request.validationError.validation,
      received: request.body
    })
  }
  return { ok: true }
})
```

### 4.3 Under the Hood — Request Validation Internals

**Execution entry point**

Validation runs inside `preValidationCallback` in
[lib/handle-request.js](../../lib/handle-request.js):

```javascript
// lib/handle-request.js (simplified)
const validationErr = validateSchema(reply[kRouteContext], request)
const isAsync = validationErr && typeof validationErr.then === 'function'

if (isAsync) {
  validationErr.then(validationCompleted.bind(null, request, reply), ...)
} else {
  validationCompleted(request, reply, validationErr)
}
```

**`validateParam` — three possible outcomes**

For each slot, `validateParam` calls the compiled function and interprets the return:

| Return value | Meaning | Action |
|---|---|---|
| `false` | Validation passed | Continue to next slot |
| Array of errors | Validation failed | Build error and return it |
| `{ error }` | Validation failed (Joi/Zod style) | Build error and return it |
| `{ value: newVal }` | Validation passed with coercion | Replace `request[slot]` with `newVal` |
| `Promise` | Async validator | Pause, await, resume with skip-flags |
| Throws synchronously | Programmer error in validator | Catch, set `statusCode = 500`, return |

The `{ value }` coercion path is how AJV's `coerceTypes` option and Yup's
`validateSync` surface their transformed values back into `request.body` /
`request.params` etc.

**Async Validator Continuation Chain**

If a validator returns a Promise, the continuation reconstructs the remaining validation
slots using a skip-flag mechanism to avoid double-validating already-passed slots:

```
validate(ctx, req)                  -> params returns Promise
  validationErr.then(resume)
    resume -> validate(ctx, req, { skipParams: true })
      -> body returns Promise
        validationErr.then(resume)
          resume -> validate(ctx, req, { skipParams: true, skipBody: true })
            -> ...
```

**Sync throw → 500**

If your compiled validator throws synchronously (a programmer error — e.g. the schema
itself is invalid but slipped through), `validateParam` catches it and sets
`err.statusCode = 500`. This distinguishes programmer errors (500) from invalid user
input (400).

---

## 5. Output Serialization

When `reply.send(payload)` is called, Fastify looks up the `fast-json-stringify` function
compiled for the response's HTTP status code and runs the payload through it. This replaces
`JSON.stringify` for any route that declares a `schema.response`.

### 5.1 Status-Code Keys

The lookup is more nuanced than an exact match. The resolver walks through these fallbacks
in order until a schema is found:

```
  1.  Exact match:    statusCode === '200'
  2.  Generic match:  statusCode[0] + 'xx' === '2xx'
  3.  Default:        'default' key exists
  4.  No match:       fall back to JSON.stringify
```

```typescript
fastify.get('/resource', {
  schema: {
    response: {
      200:     { /* exact schema for 200 */ },
      '2xx':   { /* fallback for 201, 202, 204, etc */ },
      '4xx':   { /* fallback for 400, 401, 403, 404, etc */ },
      default: { /* final fallback for everything else */ }
    }
  }
}, handler)
```

### 5.2 Field Stripping

`fast-json-stringify` only serializes fields declared in the response schema. Any field
your handler returns that is not in the schema is **silently dropped** from the response:

```typescript
interface DbUser {
  id: string
  name: string
  email: string
  passwordHash: string    // sensitive — should NOT be in the response
  createdAt: string
  updatedAt: string
}

fastify.get<{ Reply: Pick<DbUser, 'id' | 'name'> }>('/users/me', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          id:   { type: 'string' },
          name: { type: 'string' }
          // passwordHash, createdAt, updatedAt are not listed → stripped
        }
      }
    }
  }
}, async (request): Promise<Pick<DbUser, 'id' | 'name'>> => {
  const user: DbUser = await db.getUser(request.user.id)
  return user   // returning full DbUser is safe — extra fields are stripped
})
```

> **Rule:** Field stripping is implicit and silent. A field returned by your handler
> but not in the response schema will disappear without error. Use this deliberately
> to prevent data leakage (like `passwordHash` above), but add `additionalProperties: false`
> to input schemas when you want *rejecting* unknown fields on write.

### 5.3 Under the Hood — Serialization Internals

**`compileSchemasForSerialization()`**

[lib/validation.js](../../lib/validation.js) iterates the `schema.response` keys and builds
a `fast-json-stringify` function per status code:

```javascript
// lib/validation.js (simplified)
context[responseSchema] = Object.keys(schema.response).reduce((acc, statusCode) => {
  statusCode = statusCode.toLowerCase()
  // validates format: /^[1-5](?:\d{2}|xx)$|^default$/
  acc[statusCode] = serializerCompiler({ schema, url, method, httpStatus: statusCode })
  return acc
}, {})
```

Invalid status-code key formats throw `FST_ERR_SCH_RESPONSE_SCHEMA_NOT_NESTED_2XX` at
registration time.

**`getSchemaSerializer` Lookup**

[lib/schemas.js](../../lib/schemas.js) implements the fallback chain:

```javascript
// lib/schemas.js (simplified)
function getSchemaSerializer (context, statusCode, contentType) {
  const response = context[kSchemaResponse]
  if (!response) return false

  return (
    response[statusCode] ||           // '200'
    response[statusCode[0] + 'xx'] || // '2xx'
    response.default ||               // 'default'
    false
  )
}
```

**`reply.send()` Dispatch Path**

Inside [lib/reply.js](../../lib/reply.js) `preSerializationHookEnd()`, the serializer is
resolved in priority order:

```
  1. reply.serializer(fn) was called -> use that fn          (per-reply override)
  2. fastify({ replySerializer: fn }) was set -> use that fn (server-wide override)
  3. getSchemaSerializer() -> fast-json-stringify fn          (schema-based)
  4. No match -> JSON.stringify                               (fallback)
```

**Why `fast-json-stringify` is Faster**

`JSON.stringify` must inspect each value dynamically — its type, its properties, whether
they're enumerable — every time. `fast-json-stringify` generates a dedicated JavaScript
function from the schema that directly accesses known property paths by name, skipping
type checks for declared types, and avoids the dynamic enumeration loop entirely. The
performance advantage over `JSON.stringify` is typically 2–4× for typical API payloads.

---

## 6. Custom Validators

Fastify's `validatorCompiler` API lets you replace AJV with any schema validation library.
The interface is simple: provide a function that accepts `{ schema, method, url, httpPart }`
and returns a **validator function**. The validator function accepts the value to test and
must return:

- `false` — valid
- Array of errors — invalid
- `{ error }` — invalid (object wrapping errors)
- `{ value: coercedValue }` — valid, replace the original value with `coercedValue`
- A `Promise` resolving to any of the above — async validation

### 6.1 TypeBox

[TypeBox](https://github.com/sinclairzx81/typebox) generates JSON Schema–compatible types
that are valid both as TypeScript types and as AJV schemas, giving you end-to-end type
safety with zero gap between the runtime validator and the compile-time types:

```typescript
import Fastify from 'fastify'
import { Type, Static } from '@sinclair/typebox'
import Ajv from 'ajv'
import addFormats from 'ajv-formats'

// Create a single AJV instance with TypeBox-specific configuration
const ajv = addFormats(new Ajv({
  removeAdditional: true,
  useDefaults: true,
  coerceTypes: false,
  allErrors: true
}))

// TypeBox schemas double as TypeScript types
const CreateUserBody = Type.Object({
  name:  Type.String({ minLength: 1, maxLength: 100 }),
  email: Type.String({ format: 'email' }),
  age:   Type.Optional(Type.Integer({ minimum: 0, maximum: 150 }))
})

const UserResponse = Type.Object({
  id:    Type.String({ format: 'uuid' }),
  name:  Type.String(),
  email: Type.String()
})

type TCreateUserBody = Static<typeof CreateUserBody>
type TUserResponse   = Static<typeof UserResponse>

const fastify = Fastify()

// Install a server-wide TypeBox/AJV validator
fastify.setValidatorCompiler(({ schema }) => ajv.compile(schema))

fastify.post<{ Body: TCreateUserBody; Reply: TUserResponse }>('/users', {
  schema: {
    body:     CreateUserBody,
    response: { 201: UserResponse }
  }
}, async (request, reply) => {
  const user = await createUser(request.body)  // body is typed as TCreateUserBody
  return reply.status(201).send(user)           // return is typed as TUserResponse
})
```

> **Rule:** When using TypeBox schemas as both the JSON Schema (for AJV) and TypeScript
> generics, you get a single source of truth: the TypeBox definition drives runtime
> validation, compile-time types, and the `fast-json-stringify` serializer simultaneously.

### 6.2 Zod

[Zod](https://zod.dev) uses a different schema format, so you need to convert Zod schemas
to JSON Schema (for the serializer) and use Zod's own `.safeParse()` as the validator:

```typescript
import Fastify from 'fastify'
import { z } from 'zod'
import { zodToJsonSchema } from 'zod-to-json-schema'

const fastify = Fastify()

// Zod schema definitions
const CreateOrderBody = z.object({
  productId: z.string().uuid(),
  quantity:  z.number().int().positive(),
  notes:     z.string().max(500).optional()
})

const OrderResponse = z.object({
  orderId:    z.string().uuid(),
  total:      z.number(),
  status:     z.enum(['pending', 'confirmed', 'shipped'])
})

type TCreateOrderBody = z.infer<typeof CreateOrderBody>
type TOrderResponse   = z.infer<typeof OrderResponse>

// Route-level Zod validator
fastify.post<{ Body: TCreateOrderBody; Reply: TOrderResponse }>('/orders', {
  schema: {
    body:     zodToJsonSchema(CreateOrderBody),   // converted for fast-json-stringify
    response: { 201: zodToJsonSchema(OrderResponse) }
  },
  // Override the validator for this route to use Zod
  validatorCompiler: ({ schema, httpPart }) => {
    // We use Zod only for body; fall back for other parts
    if (httpPart === 'body') {
      return (data: unknown) => {
        const result = CreateOrderBody.safeParse(data)
        if (!result.success) {
          return { error: result.error.format() }  // return { error } on failure
        }
        return { value: result.data }              // return { value } for coercion (strips extras)
      }
    }
    // For params/query/headers use a no-op (always valid)
    return () => false
  }
}, async (request, reply) => {
  const order = await processOrder(request.body)
  return reply.status(201).send(order)
})
```

> **Rule:** `zodToJsonSchema()` is needed to give `fast-json-stringify` a JSON Schema for
> the response serializer. Zod and `fast-json-stringify` speak different schema languages;
> the conversion bridges them.

### 6.3 Raw AJV with coerceTypes

Use a custom AJV instance when you need AJV features not enabled in Fastify's default
configuration, such as `coerceTypes` (converts URL string params to their declared type)
or `useDefaults` (injects default values declared in the schema):

```typescript
import Fastify from 'fastify'
import Ajv, { ValidateFunction } from 'ajv'
import addFormats from 'ajv-formats'

// Custom AJV with coercion and default injection
const ajv = addFormats(
  new Ajv({
    removeAdditional: true,   // strip undeclared input fields
    useDefaults: true,        // inject default values from schema
    coerceTypes: 'array',     // coerce string params to declared types; 'array' handles multi-value query params
    allErrors: false          // stop at first error for performance
  })
)

const fastify = Fastify()

// Install as server-wide validator
fastify.setValidatorCompiler<ValidateFunction>(({ schema }) => {
  return ajv.compile(schema)
})

fastify.get('/items', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        page:     { type: 'integer', minimum: 1, default: 1 },   // default injected
        pageSize: { type: 'integer', minimum: 1, maximum: 100, default: 20 },
        active:   { type: 'boolean' }   // coerced: '?active=true' → true
      }
    }
  }
}, async (request) => {
  // With coerceTypes + useDefaults:
  //   request.query.page     is number (not string), defaults to 1
  //   request.query.pageSize is number (not string), defaults to 20
  //   request.query.active   is boolean (not string)
  return { items: [], page: request.query.page }
})
```

### 6.4 Server-Level vs Route-Level vs compilersFactory

There are three places to install a custom validator/serializer, with different scopes:

```typescript
import Fastify from 'fastify'

// ── Option 1: Server-level (applies to all routes) ───────────────────────
const fastify = Fastify()
fastify.setValidatorCompiler(myValidatorCompiler)
fastify.setSerializerCompiler(mySerializerCompiler)

// ── Option 2: Route-level (overrides server-level for this route only) ───
fastify.post('/special', {
  schema: { body: mySchema },
  validatorCompiler:  myRouteValidatorCompiler,
  serializerCompiler: myRouteSerializerCompiler
}, handler)

// ── Option 3: compilersFactory (most powerful — controls AJV instance creation) ───
const fastify2 = Fastify({
  schemaController: {
    compilersFactory: {
      buildValidator:   (schemas, ajvOpts) => {
        // schemas = all addSchema() entries; return a validatorCompiler fn
        const ajv = new Ajv({ ...ajvOpts })
        schemas.forEach(s => ajv.addSchema(s))
        return ({ schema }) => ajv.compile(schema)
      },
      buildSerializer:  (schemas, serializerOpts) => {
        // return a serializerCompiler fn
        return ({ schema }) => fastJsonStringify(schema, { schema: schemas })
      }
    }
  }
})
```

### 6.5 Under the Hood — Custom Compiler Internals

**`isCustomValidatorCompiler` Flag**

When you set a custom validator via `setValidatorCompiler()` or `validatorCompiler` route
option, Fastify sets `isCustomValidatorCompiler = true` on the schema controller context.
This flag **disables the automatic header key lowercase normalization** described in §2.6.
Custom validators receive the schema exactly as written — including header keys in their
original casing.

**`compilersFactory` Adapter Pattern**

`setValidatorCompiler(fn)` internally wraps your function as:

```javascript
compilersFactory: { buildValidator: () => fn }
```

This makes the single-function API and the full factory API interchangeable internally,
preserving the `setupValidator` rebuild-on-`addSchema` path for both.

---

## 7. Error Handling and Formatting

### 7.1 schemaErrorFormatter

The default error message is the AJV error text for the first failing constraint. To
customise the shape or message of validation errors, provide a `schemaErrorFormatter`
either globally (as a Fastify option) or per-route:

```typescript
import Fastify, { FastifySchemaValidationError } from 'fastify'

const fastify = Fastify({
  // Global formatter — receives AJV errors and the data part that failed
  schemaErrorFormatter (errors: FastifySchemaValidationError[], dataVar: string) {
    const messages = errors.map(e => `${e.instancePath || dataVar} ${e.message}`)
    const err = new Error(messages.join('; '))
    ;(err as any).statusCode = 400
    return err
  }
})

// Or per-route:
fastify.post('/strict', {
  schema: {
    body: {
      type: 'object',
      properties: { name: { type: 'string' } },
      required: ['name']
    }
  },
  schemaErrorFormatter (errors, dataVar) {
    return Object.assign(new Error('Validation failed'), {
      statusCode: 422,
      validationErrors: errors
    })
  }
}, handler)
```

The returned error is passed directly to `reply.send()`, which invokes the `onError`
lifecycle hook. You can intercept it there for structured logging or further transformation.

### 7.2 Error Code Reference

| Code | Stage | HTTP Status | Description |
|---|---|---|---|
| `FST_ERR_VALIDATION` | Request time | `400` | Input params / body / querystring / headers failed validation |
| `FST_ERR_SCH_VALIDATION_BUILD` | Startup | — | AJV failed to compile an input schema |
| `FST_ERR_SCH_SERIALIZATION_BUILD` | Startup | — | `fast-json-stringify` failed to compile a response schema |
| `FST_ERR_SCH_MISSING_ID` | `addSchema()` | — | Schema passed to `addSchema()` has no `$id` |
| `FST_ERR_SCH_ALREADY_PRESENT` | `addSchema()` | — | Duplicate `$id` in the same scope |
| `FST_ERR_SCH_DUPLICATE` | Route registration | — | Both `query` and `querystring` declared on the same route |
| `FST_ERR_SCH_CONTENT_MISSING_SCHEMA` | Route registration | — | `schema.body.content['application/json']` missing a `.schema` sub-key |
| `FST_ERR_SCH_RESPONSE_SCHEMA_NOT_NESTED_2XX` | Route registration | — | Response schema key doesn't match `/^[1-5](\d{2}\|xx)$\|^default$/` |
| `FST_ERR_MISSING_SERIALIZATION_FN` | Request time | `500` | `reply.serializeInput()` called with a status code that has no registered schema |
| `FSTWRN001` | Startup (warning) | — | Schema key exists but is falsy/empty (process warning, not an error) |

---

## 8. Advanced Patterns

### 8.1 Content-Type–Scoped Body Schemas (OpenAPI 3)

When your endpoint accepts different body formats depending on `content-type`, use the
OpenAPI 3 `schema.body.content` map. Fastify compiles a separate validator per MIME type
and dispatches at request time based on the incoming `Content-Type` header:

```typescript
import Fastify from 'fastify'

const fastify = Fastify()

fastify.post('/upload', {
  schema: {
    body: {
      content: {
        'application/json': {
          schema: {
            type: 'object',
            properties: {
              name: { type: 'string' },
              data: { type: 'string', contentEncoding: 'base64' }
            },
            required: ['name', 'data']
          }
        },
        'application/x-www-form-urlencoded': {
          schema: {
            type: 'object',
            properties: {
              name: { type: 'string' },
              file: { type: 'string' }
            },
            required: ['name']
          }
        }
      }
    }
  }
}, async (request) => {
  // Fastify matched the validator based on request.headers['content-type']
  return { ok: true }
})
```

If the incoming `Content-Type` has no matching entry in the map, validation is skipped
for the body (no match = no validator compiled for that type).

### 8.2 Content-Type–Scoped Response Schemas

Response schemas can also be keyed by both status code and content type when different
response formats require different schemas:

```typescript
fastify.get('/data', {
  schema: {
    response: {
      200: {
        content: {
          'application/json': {
            schema: {
              type: 'object',
              properties: {
                count: { type: 'integer' },
                items: { type: 'array', items: { type: 'object' } }
              }
            }
          },
          'text/csv': {
            schema: { type: 'string' }
          },
          '*/*': {
            // Wildcard: matches any content type not matched above
            schema: {
              type: 'object',
              properties: { raw: { type: 'string' } }
            }
          }
        }
      }
    }
  }
}, async (request, reply) => {
  const accept = request.headers.accept ?? 'application/json'
  if (accept.includes('text/csv')) {
    return reply.type('text/csv').send('id,name\n1,Alice')
  }
  return { count: 1, items: [{ id: 1 }] }
})
```

The `*/*` wildcard entry acts as a catch-all for any content type not explicitly declared.

### 8.3 reply.compileSerializationSchema

For cases where you need to serialize a value using an ad-hoc schema that was not
pre-registered in the route's `schema.response`, use `reply.compileSerializationSchema()`.
The compiled function is cached in a `WeakMap` on the route context so that repeated calls
with the same schema object incur no recompilation cost:

```typescript
import Fastify from 'fastify'

const fastify = Fastify()

// Schema defined once, referenced by identity (WeakMap key)
const PartialUserSchema = {
  type: 'object',
  properties: {
    id:   { type: 'string' },
    name: { type: 'string' }
  }
} as const

fastify.get('/users/:id', async (request, reply) => {
  const user = await db.getUser(request.params.id)

  // `serialize` is cached after the first call — same schema object = same WeakMap key
  const serialize = reply.compileSerializationSchema(PartialUserSchema)
  return serialize(user)   // fast-json-stringify output
})
```

> **Rule:** For `reply.compileSerializationSchema()` caching to work, always reference
> the **same schema object** (by identity). Creating a new object literal `{}` on every
> request bypasses the cache and compiles a new serializer on every request.

### 8.4 reply.serializeInput

`reply.serializeInput()` serializes a value against a pre-registered response schema
without going through `reply.send()`. This is useful in hooks or middleware where you want
to produce an encoded payload without committing the response:

```typescript
fastify.addHook('preSerialization', async (request, reply, payload) => {
  // Wrap the payload in an envelope using the registered schema for status 200
  const serialized = reply.serializeInput(
    payload,
    undefined,  // schema (optional — use pre-registered if null)
    reply.statusCode
  )
  return { data: serialized, timestamp: Date.now() }
})
```

If you pass a `statusCode` for which no schema is registered, Fastify throws
`FST_ERR_MISSING_SERIALIZATION_FN` rather than silently falling back to `JSON.stringify`.

---

## 9. Performance Deep Dive

The validation system is deliberately designed so that **all expensive work happens at startup**
and **request handling is zero-cost beyond the function call itself**.

| Phase | Work done | Cost |
|---|---|---|
| **Startup** | Parse JSON Schema → compile AJV JS closure | ~0.1–5 ms per schema (CPU), done once |
| **Startup** | Build `fast-json-stringify` serializer per response schema | ~0.1–2 ms per schema, done once |
| **Request time (input)** | Call pre-compiled AJV closure with `request.body` | ~1–10 µs (closure call only) |
| **Request time (output)** | Symbol lookup + `fast-json-stringify` call | ~2–15 µs (faster than `JSON.stringify`) |

**AJV v8 Code Generation**

AJV v8 compiles JSON Schema to JavaScript source code and evaluates it with `new Function()`.
The resulting closure uses direct property accesses (`data.name`), has the schema constraints
inlined as literals, and avoids prototype-chain traversal. Accessing a compiled AJV function
via a Symbol key on the route context is effectively:

```
context[Symbol('body-schema')](request.body)
```

This is a single property read on a known-shape object — V8 can optimise this to a direct
memory offset. There is no `Map` lookup, no `if/else` dispatch tree, no property
enumeration.

**`fast-json-stringify` vs `JSON.stringify`**

`JSON.stringify` uses a generic algorithm that must introspect each value at runtime:
check if it's a string, number, boolean, array, object, then enumerate properties. For a
known schema shape, `fast-json-stringify` generates code that directly writes the opening
brace, writes `"id":`, reads `obj.id`, writes the closing `"`, etc. — no type checking,
no enumeration.

**WeakMap Cache for Ad-hoc Serializers**

`reply.compileSerializationSchema()` uses a `WeakMap<schema, fn>` stored as
`kReplyCacheSerializeFns` on the route context. WeakMap lookups are `O(1)` and GC-friendly
— the entry is automatically released when the schema object is garbage-collected. The
WeakMap itself is created lazily (only when the method is first called), so routes that
never use it pay zero overhead.

**Startup Trade-offs**

The compile-once model shifts cost to startup. Practical guidelines:

- Each `addSchema()` call that's made **after** the first `setupValidator()` run triggers
  a full AJV instance rebuild for that scope. Keep all `addSchema()` calls in plugin
  registration bodies (before `ready()`), not in application code paths.
- Applications with hundreds of schemas see measurable startup time increases. Profile
  with `--prof` if startup time is critical in a serverless context.
- AJV `allErrors: true` (collect all errors in one pass) is slightly slower than
  `allErrors: false` (stop at first error). Use `false` in production for throughput.

---

## 10. Pros, Cons, and Trade-offs

### Pros

| Feature | Detail |
|---|---|
| **Zero runtime overhead** | All schema compilation happens at startup; request-time validation is a pure closure call |
| **AJV v8 (one of the fastest validators)** | Code-generation approach; compiles to native JS; consistently top benchmarks |
| **`fast-json-stringify` response encoding** | 2–4× faster than `JSON.stringify` for typical payloads; implicit data-leak protection via field stripping |
| **Pluggable compiler API** | Replace AJV with TypeBox, Zod, Joi, custom logic via `validatorCompiler` / `compilersFactory` |
| **Schema scope inheritance** | Child plugin scopes inherit parent `addSchema()` entries; no manual wiring |
| **`fluent-json-schema` first-class** | Auto-detected and unwrapped via `.valueOf()`; no adapter code needed |
| **`attachValidation` soft mode** | Opt-in per-route to receive invalid input in the handler rather than auto-rejecting |
| **Async validators** | Validators can return Promises; continuation chain reconstructed correctly with skip-flags |
| **Content-type routing** | Per-MIME-type body and response schemas for OpenAPI 3–style APIs |
| **Status-code wildcards** | `2xx`, `4xx`, `5xx`, `default` fallback keys reduce schema duplication |
| **One source of truth with TypeBox** | TypeBox schemas are simultaneously valid AJV input, `fast-json-stringify` input, and TypeScript types |

### Cons

| Limitation | Detail |
|---|---|
| **Startup cost scales with schema count** | Large applications with hundreds of `addSchema()` entries see slower cold starts; critical in serverless environments |
| **Silent field stripping** | Extra response fields disappear without warning; devs coming from `JSON.stringify` are surprised when fields vanish |
| **Header lowercase coupling** | Schema properties for headers must be lowercase; using uppercase or mixed-case (without auto-normalization) fails silently |
| **Custom validators disable normalization** | Setting `isCustom = true` turns off Fastify's automatic header lowercase normalization; custom validators must handle casing themselves |
| **Cross-field validation requires AJV keywords** | AJV validates fields in isolation; multi-field constraints (e.g. `endDate > startDate`) require custom AJV keywords or a post-validation hook |
| **`addSchema()` after init triggers rebuild** | Calling `addSchema()` programmatically after the first validator build triggers a full AJV instance rebuild for that scope |
| **`schemaErrorFormatter` is global or per-route** | No built-in per-`httpPart` error formatting; formatting `params` errors differently from `body` errors requires manual inspection of `error.validationContext` |
| **Zod requires schema conversion for serializer** | The `fast-json-stringify` serializer always needs JSON Schema; Zod users must convert with `zod-to-json-schema` |

---

## 11. TypeScript Quick-Reference

### `FastifySchema` — All Slot Types

```typescript
import {
  FastifySchema,
  FastifyRequest,
  FastifyReply,
  RouteShorthandOptions
} from 'fastify'

// The full schema interface:
interface MySchema extends FastifySchema {
  params?:      unknown  // JSON Schema object
  body?:        unknown
  querystring?: unknown
  headers?:     unknown
  response?:    { [statusCode: string]: unknown }
}
```

### Route Generic Interface — Typed `request.body`

```typescript
import Fastify, { FastifyRequest, FastifyReply } from 'fastify'

interface CreateUserBody {
  name:  string
  email: string
  age?:  number
}

interface UserReply {
  id:   string
  name: string
}

const fastify = Fastify()

fastify.post<{
  Body:  CreateUserBody
  Reply: UserReply
}>('/users', {
  schema: {
    body: {
      type: 'object',
      properties: {
        name:  { type: 'string' },
        email: { type: 'string', format: 'email' },
        age:   { type: 'integer' }
      },
      required: ['name', 'email']
    },
    response: {
      201: {
        type: 'object',
        properties: {
          id:   { type: 'string' },
          name: { type: 'string' }
        }
      }
    }
  }
}, async (
  request: FastifyRequest<{ Body: CreateUserBody }>,
  reply:   FastifyReply
): Promise<UserReply> => {
  const body = request.body   // typed as CreateUserBody
  return { id: 'usr_1', name: body.name }
})
```

### `ValidatorCompiler` and `SerializerCompiler` Types

```typescript
import {
  FastifySchemaCompiler,
  FastifySerializerCompiler,
  FastifyTypeProvider
} from 'fastify'

// ValidatorCompiler: receives schema context, returns a validator function
type MyValidatorCompiler = FastifySchemaCompiler<unknown>
// = ({ schema, method, url, httpPart }) => (data: unknown) => boolean | ...

// SerializerCompiler: receives schema context, returns a serializer function
type MySerializerCompiler = FastifySerializerCompiler<unknown>
// = ({ schema, method, url, httpStatus }) => (data: unknown) => string
```

### TypeBox Type Provider

TypeBox integrates with Fastify via a `TypeProvider` to close the loop between runtime
schema validation and TypeScript type inference:

```typescript
import Fastify from 'fastify'
import { TypeBoxTypeProvider } from '@fastify/type-provider-typebox'
import { Type } from '@sinclair/typebox'

const fastify = Fastify().withTypeProvider<TypeBoxTypeProvider>()

// Now schema inference flows directly to handler types — no separate interface needed
fastify.post('/items', {
  schema: {
    body:     Type.Object({ name: Type.String() }),
    response: { 201: Type.Object({ id: Type.String() }) }
  }
}, async (request) => {
  const name = request.body.name   // ← TypeScript knows this is `string`
  return { id: 'itm_1' }
})
```

### Zod Type Provider

```typescript
import Fastify from 'fastify'
import { ZodTypeProvider } from 'fastify-type-provider-zod'
import { z } from 'zod'

const fastify = Fastify().withTypeProvider<ZodTypeProvider>()

fastify.post('/items', {
  schema: {
    body:     z.object({ name: z.string() }),
    response: { 201: z.object({ id: z.string() }) }
  }
}, async (request) => {
  const name = request.body.name   // ← TypeScript infers from Zod schema
  return { id: 'itm_1' }
})
```

### `schemaErrorFormatter` Type Signature

```typescript
import Fastify, { FastifySchemaValidationError } from 'fastify'

const fastify = Fastify({
  schemaErrorFormatter: (
    errors:  FastifySchemaValidationError[],
    dataVar: string    // e.g. 'body', 'params'
  ): Error => {
    return new Error(
      errors.map(e => `[${dataVar}${e.instancePath}] ${e.message}`).join(', ')
    )
  }
})
```

---

*Tutorial last updated: 2026-03-02*
