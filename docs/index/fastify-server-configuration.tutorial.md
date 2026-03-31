# Fastify Server Configuration — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript who want to master
> every configuration option Fastify exposes at instantiation time and understand what
> happens internally when each option is applied.

---

## Table of Contents

1. [The Factory Function](#1-the-factory-function)
2. [HTTP Protocol Options](#2-http-protocol-options)
   - 2.1 [`http`](#21-http)
   - 2.2 [`http2`](#22-http2)
   - 2.3 [`https`](#23-https)
   - 2.4 [`serverFactory`](#24-serverfactory)
3. [Connection and Timeout Options](#3-connection-and-timeout-options)
   - 3.1 [`connectionTimeout`](#31-connectiontimeout)
   - 3.2 [`keepAliveTimeout`](#32-keepalivetimeout)
   - 3.3 [`requestTimeout`](#33-requesttimeout)
   - 3.4 [`handlerTimeout`](#34-handlertimeout)
   - 3.5 [`maxRequestsPerSocket`](#35-maxrequestspersocket)
   - 3.6 [`forceCloseConnections`](#36-forcecloseconnections)
   - 3.7 [`http2SessionTimeout`](#37-http2sessiontimeout)
4. [Request Body Options](#4-request-body-options)
   - 4.1 [`bodyLimit`](#41-bodylimit)
   - 4.2 [`onProtoPoisoning`](#42-onprotopoisoning)
   - 4.3 [`onConstructorPoisoning`](#43-onconstructorpoisoning)
5. [Logging Options](#5-logging-options)
   - 5.1 [`logger`](#51-logger)
   - 5.2 [`loggerInstance`](#52-loggerinstance)
   - 5.3 [`disableRequestLogging`](#53-disablerequestlogging)
   - 5.4 [`requestIdHeader`](#54-requestidheader)
   - 5.5 [`requestIdLogLabel`](#55-requestidloglabel)
   - 5.6 [`genReqId`](#56-genreqid)
6. [Router Options](#6-router-options)
   - 6.1 [`caseSensitive`](#61-casesensitive)
   - 6.2 [`ignoreTrailingSlash`](#62-ignoretrailingslash)
   - 6.3 [`ignoreDuplicateSlashes`](#63-ignoreduplicateslashes)
   - 6.4 [`maxParamLength`](#64-maxparamlength)
   - 6.5 [`allowUnsafeRegex`](#65-allowunsaferegex)
   - 6.6 [`useSemicolonDelimiter`](#66-usesemicolondelimiter)
   - 6.7 [`querystringParser`](#67-querystringparser)
   - 6.8 [`constraints`](#68-constraints)
7. [Plugin and Startup Options](#7-plugin-and-startup-options)
   - 7.1 [`pluginTimeout`](#71-plugintimeout)
   - 7.2 [`trustProxy`](#72-trustproxy)
   - 7.3 [`exposeHeadRoutes`](#73-exposeheadroutes)
   - 7.4 [`return503OnClosing`](#74-return503onclosing)
   - 7.5 [`rewriteUrl`](#75-rewriteurl)
8. [Validation and Serialization Options](#8-validation-and-serialization-options)
   - 8.1 [`ajv`](#81-ajv)
   - 8.2 [`serializerOpts`](#82-serializeropts)
   - 8.3 [`schemaController`](#83-schemacontroller)
9. [Error Handling Options](#9-error-handling-options)
   - 9.1 [`frameworkErrors`](#91-frameworkerrors)
   - 9.2 [`clientErrorHandler`](#92-clienterrorhandler)
   - 9.3 [`allowErrorHandlerOverride`](#93-allowerrorhandleroverride)
10. [The `listen()` Call](#10-the-listen-call)
    - 10.1 [Listen Options](#101-listen-options)
    - 10.2 [Unix Socket and File Descriptor](#102-unix-socket-and-file-descriptor)
    - 10.3 [AbortSignal Integration](#103-abortsignal-integration)
11. [`initialConfig` — The Sealed Snapshot](#11-initialconfig--the-sealed-snapshot)
12. [How Options Are Processed Internally](#12-how-options-are-processed-internally)
13. [TypeScript Quick-Reference](#13-typescript-quick-reference)
14. [Full Configuration Example](#14-full-configuration-example)
15. [Common Pitfalls](#15-common-pitfalls)

Related deep dives:

- [Fastify Lifecycle — Complete Tutorial](./fastify-lifecycle.tutorial.md)
- [Fastify Plugins — Complete Tutorial](./fastify-plugins.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)
- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)

---

## 1. The Factory Function

Every Fastify application starts with a single call to the factory:

```javascript
const Fastify = require('fastify')

const app = Fastify({ /* options */ })
```

Or with ES modules / TypeScript:

```typescript
import Fastify, { FastifyInstance } from 'fastify'

const app: FastifyInstance = Fastify({ /* options */ })
```

Under the hood, calling `Fastify(options)` runs `processOptions(options, ...)` in
[fastify.js](../fastify.js), which:

1. Deep-validates the options object against a compiled AJV schema (see
   [lib/config-validator.js](../../lib/config-validator.js)).
2. Creates the Node.js HTTP / HTTPS / HTTP2 server via `createServer` in
   [lib/server.js](../../lib/server.js).
3. Builds the router via `buildRouting` from [lib/route.js](../../lib/route.js).
4. Returns a frozen `initialConfig` snapshot so your startup options can never
   be mutated after boot.

```text
Fastify(opts)
  |
  +-- processOptions(opts)
  |     |-- validate schema (AJV, config-validator.js)
  |     |-- createLogger(opts)
  |     |-- reqIdGenFactory(...)
  |     |-- getSecuredInitialConfig(opts)  <-- returns frozen copy
  |     '-- buildRouterOptions(opts)
  |
  +-- createServer(opts, httpHandler)  ---> Node.js http/https/http2 server
  |
  +-- build404(opts)
  |
  +-- buildRouting(routerOptions)      ---> find-my-way router
  |
  '-- returns FastifyInstance (public API)
```

---

## 2. HTTP Protocol Options

### 2.1 `http`

```javascript
const app = Fastify({
  http: {
    // any option accepted by Node.js http.createServer()
    keepAlive: true,
    noDelay: true
  }
})
```

- **Default:** `null`
- Passes raw options to Node.js `http.createServer(options)`.
- Ignored when `http2` or `https` is set.
- Useful for enabling `IncomingMessage` subclasses or adjusting socket-level
  behaviour.

### 2.2 `http2`

```javascript
const app = Fastify({ http2: true })
```

- **Default:** `false`
- Switches the underlying server to Node.js `http2.createServer()` (cleartext
  HTTP/2).
- Combine with `https` to use `http2.createSecureServer()`.

### 2.3 `https`

```javascript
const fs = require('node:fs')

const app = Fastify({
  https: {
    key:  fs.readFileSync('./server.key'),
    cert: fs.readFileSync('./server.cert')
  }
})
```

- **Default:** `null`
- When set, Fastify uses `https.createServer(opts)` for HTTP/1.1 TLS, or
  `http2.createSecureServer(opts)` when `http2: true` is also set.
- The TLS options mirror the Node.js `tls.createServer()` API
  (`key`, `cert`, `ca`, `requestCert`, `SNICallback`, …).

```text
http2: false, https: null   --> http.createServer()
http2: false, https: {...}  --> https.createServer()
http2: true,  https: null   --> http2.createServer()   (cleartext H2)
http2: true,  https: {...}  --> http2.createSecureServer()
```

### 2.4 `serverFactory`

```javascript
const http = require('node:http')

const app = Fastify({
  serverFactory (handler, opts) {
    const server = http.createServer((req, res) => {
      handler(req, res)
    })
    // attach extra event listeners, metrics, etc.
    server.on('error', (err) => console.error(err))
    return server
  }
})
```

- **Default:** `null` (Fastify creates its own server)
- Gives full control over the Node.js server instance.
- When `serverFactory` is used, `connectionTimeout`, `keepAliveTimeout`,
  `requestTimeout`, and `maxRequestsPerSocket` options are **ignored** because
  Fastify cannot set those properties on an externally supplied server.
- The factory **must** return a server that exposes the Node.js HTTP server API.

---

## 3. Connection and Timeout Options

### 3.1 `connectionTimeout`

```javascript
const app = Fastify({ connectionTimeout: 30_000 }) // 30 s
```

- **Default:** `0` (no timeout)
- Maps directly to `server.timeout` in Node.js: the socket is destroyed if no
  data is received within this window (in milliseconds).
- Applies only when Fastify manages the server (ignored with `serverFactory`).

### 3.2 `keepAliveTimeout`

```javascript
const app = Fastify({ keepAliveTimeout: 5_000 }) // 5 s
```

- **Default:** `72000` (72 seconds)
- Maps to `server.keepAliveTimeout` — how long an idle keep-alive socket stays
  open between requests.
- HTTP/1 only; ignored for HTTP/2 (use `http2SessionTimeout` instead).
- Ignored when `serverFactory` is provided.

> Set this slightly higher than your load-balancer's keep-alive so the server
> never drops a connection before the proxy does.

### 3.3 `requestTimeout`

```javascript
const app = Fastify({ requestTimeout: 120_000 }) // 2 min
```

- **Default:** `0` (no limit)
- Maps to `server.requestTimeout`: maximum milliseconds to receive the **full
  incoming request** (headers + body) from the client.
- Protects against slow-loris style attacks when you are **not** behind a
  reverse proxy.
- Ignored when `serverFactory` is provided.

### 3.4 `handlerTimeout`

```javascript
const app = Fastify({ handlerTimeout: 10_000 }) // 10 s default for all routes
```

- **Default:** `0` (no timeout)
- An **application-level** timeout that covers the entire Fastify route
  lifecycle: hooks + handler + serialization.
- When the timeout fires, Fastify sends a `503 Service Unavailable` response
  and aborts `request.signal`.
- Unlike socket-level timeouts, this works correctly with keep-alive
  connections.
- Can be overridden **per route** in route options; the route value takes
  precedence.

```javascript
// Per-route override
app.get('/slow', { handlerTimeout: 60_000 }, async (request) => {
  const data = await db.query(sql, { signal: request.signal }) // respects abort
  return data
})
```

```text
Global handlerTimeout = 10 s
  |
  +-- /fast  (no route override)  --> inherits 10 s
  +-- /slow  (handlerTimeout: 60s) --> 60 s
  +-- /report (handlerTimeout: 0)  --> ERROR: cannot opt out once global is set
```

### 3.5 `maxRequestsPerSocket`

```javascript
const app = Fastify({ maxRequestsPerSocket: 100 })
```

- **Default:** `0` (unlimited)
- After this many requests have been handled on a single keep-alive socket,
  the socket is closed.
- HTTP/1.1 only; ignored with HTTP/2 and with `serverFactory`.

### 3.6 `forceCloseConnections`

```javascript
// Destroy all connections on close
const app = Fastify({ forceCloseConnections: true })

// Destroy only idle connections on close (default when supported)
const app = Fastify({ forceCloseConnections: 'idle' })
```

- **Default:** `"idle"` if `server.closeIdleConnections` is available (Node.js
  >= 18.2), `false` otherwise.
- `true` — calls `server.closeAllConnections()` on `fastify.close()`,
  destroying **all** sockets including active ones.
- `"idle"` — calls `server.closeIdleConnections()`, only destroying sockets
  with no in-flight request or response.
- `false` — Fastify waits for all connections to drain naturally.

```text
fastify.close()
  |
  +-- forceCloseConnections: true   --> closeAllConnections()
  +-- forceCloseConnections: 'idle' --> closeIdleConnections()
  +-- forceCloseConnections: false  --> wait for natural drain
```

### 3.7 `http2SessionTimeout`

```javascript
const app = Fastify({
  http2: true,
  http2SessionTimeout: 120_000 // 2 min
})
```

- **Default:** `72000` (72 seconds)
- Sets a session-level timeout for every HTTP/2 connection.
- The low default is intentional — it limits the attack surface of long-lived
  idle sessions under load.
- Increase it when Fastify is behind a trusted load balancer.

---

## 4. Request Body Options

### 4.1 `bodyLimit`

```javascript
const app = Fastify({ bodyLimit: 5 * 1024 * 1024 }) // 5 MiB
```

- **Default:** `1048576` (1 MiB)
- Maximum size of the request body in bytes.
- Applies **after** any `preParsing` transform — so if you decompress a stream
  in `preParsing`, the limit is enforced on the decompressed bytes.
- When exceeded, Fastify sends a `413 Content Too Large` response
  (`FST_ERR_CTP_BODY_TOO_LARGE`).
- Can also be set per-route:
  ```javascript
  app.post('/upload', { bodyLimit: 50 * 1024 * 1024 }, handler)
  ```

### 4.2 `onProtoPoisoning`

```javascript
const app = Fastify({ onProtoPoisoning: 'remove' })
```

- **Default:** `'error'`
- Controls what happens when a JSON body contains a `__proto__` key.
- Values:
  - `'error'` — throw `FST_ERR_CTP_INVALID_JSON_BODY` (safest).
  - `'remove'` — silently strip the `__proto__` key from the parsed object.
  - `'ignore'` — leave the key in place (dangerous, use only if you know why).
- Powered by [`secure-json-parse`](https://github.com/fastify/secure-json-parse).

### 4.3 `onConstructorPoisoning`

```javascript
const app = Fastify({ onConstructorPoisoning: 'remove' })
```

- **Default:** `'error'`
- Same semantics as `onProtoPoisoning` but for the `constructor` key.
- Both `onProtoPoisoning` and `onConstructorPoisoning` defend against
  **prototype pollution** attacks in JSON payloads.

---

## 5. Logging Options

### 5.1 `logger`

```javascript
// Disabled (default)
const app = Fastify({ logger: false })

// Enabled with defaults
const app = Fastify({ logger: true })

// Pino options object
const app = Fastify({
  logger: {
    level: 'debug',
    transport: {
      target: 'pino-pretty',
      options: { colorize: true }
    }
  }
})
```

- **Default:** `false`
- When `false`, Fastify installs a no-op logger (`abstract-logging`).
- When `true` or an object, Fastify creates a Pino logger.
- If an object is provided, it is passed directly to the Pino constructor.
  Fastify always injects default serializers for `req`, `res`, and `err`
  (custom serializers provided in the object **merge** with these defaults).
- Handled in [lib/logger-factory.js](../../lib/logger-factory.js).

### 5.2 `loggerInstance`

```javascript
import pino from 'pino'

const logger = pino({ level: 'info' })

const app = Fastify({ loggerInstance: logger })
```

- **Default:** `null`
- Pass a pre-built Pino-compatible logger instance.
- The instance must expose: `info`, `error`, `debug`, `fatal`, `warn`, `trace`,
  `child`.
- `logger` and `loggerInstance` are mutually exclusive — providing both throws
  `FST_ERR_LOG_LOGGER_AND_LOGGER_INSTANCE_PROVIDED`.

### 5.3 `disableRequestLogging`

```javascript
// Statically suppress all request/response logs
const app = Fastify({ logger: true, disableRequestLogging: true })

// Conditionally suppress based on the request
const app = Fastify({
  logger: true,
  disableRequestLogging: (request) => request.url === '/health'
})
```

- **Default:** `false`
- When `true`, suppresses the automatic `info` log on request received and on
  response sent.
- Accepts a function `(FastifyRequest) => boolean` for selective suppression.
- Use this when you want fine-grained control via custom `onRequest` /
  `onResponse` hooks.

### 5.4 `requestIdHeader`

```javascript
const app = Fastify({ requestIdHeader: 'x-request-id' })
```

- **Default:** `false`
- When set to a non-empty string, Fastify reads the request ID from that header
  name (lowercased) instead of generating one.
- When `false` (default), `genReqId` is always called.
- When `true`, uses the header name `"request-id"`.

> **Security:** No validation is performed on the incoming header value. An
> attacker can inject arbitrary strings into your log `reqId`. Only enable this
> behind a trusted proxy that strips or rewrites the header.

### 5.5 `requestIdLogLabel`

```javascript
const app = Fastify({ requestIdLogLabel: 'traceId' })
```

- **Default:** `'reqId'`
- The key used when logging the request ID.
- Each per-request child logger will carry `{ [requestIdLogLabel]: id }` in
  its bindings.

### 5.6 `genReqId`

```javascript
import { v4 as uuidv4 } from 'uuid'

const app = Fastify({
  genReqId: (req) => uuidv4()
})
```

- **Default:** auto-incrementing integer IDs formatted as `req-<base36>` (resets
  at 2,147,483,647).
- Receives the raw `IncomingMessage`; must be synchronous and error-free.
- Called only when `requestIdHeader` is not set, or no matching header is
  present on the request.

See [lib/req-id-gen-factory.js](../../lib/req-id-gen-factory.js) for the full
factory logic.

---

## 6. Router Options

All router options are passed through `routerOptions` to the internal
[`find-my-way`](https://github.com/delvedor/find-my-way) instance. Several
can also be placed at the top-level for backwards compatibility — Fastify will
merge them into `routerOptions` via `buildRouterOptions()` during
`processOptions`.

```javascript
const app = Fastify({
  routerOptions: {
    caseSensitive: false,
    ignoreTrailingSlash: true
  }
})
```

### 6.1 `caseSensitive`

- **Default:** `true`
- When `false`, routes match regardless of letter case (`/Foo` === `/foo`).
- Route parameters retain their original casing even when the route is matched
  case-insensitively.

### 6.2 `ignoreTrailingSlash`

- **Default:** `false`
- When `true`, `/users` and `/users/` match the same route.

### 6.3 `ignoreDuplicateSlashes`

- **Default:** `false`
- When `true`, `//users//list` is treated the same as `/users/list`.

### 6.4 `maxParamLength`

```javascript
const app = Fastify({ routerOptions: { maxParamLength: 500 } })
```

- **Default:** `100`
- Maximum number of characters in a URL parameter value.
- Requests whose parameter values exceed this length receive a 404 response.

### 6.5 `allowUnsafeRegex`

```javascript
const app = Fastify({ routerOptions: { allowUnsafeRegex: true } })

app.get('/item/:id(^[a-z0-9]{8,32}$)', handler)
```

- **Default:** `false`
- By default `find-my-way` rejects regex patterns that could cause ReDoS.
- Set to `true` only if you have verified the regex is safe.

### 6.6 `useSemicolonDelimiter`

```javascript
const app = Fastify({ routerOptions: { useSemicolonDelimiter: true } })
// /search;q=hello;page=2  ->  req.params = { q: 'hello', page: '2' }
```

- **Default:** `false`
- When `true`, semicolons in the URL path are treated as parameter delimiters
  (legacy matrix parameters style).

### 6.7 `querystringParser`

```javascript
import qs from 'qs'

const app = Fastify({
  routerOptions: {
    querystringParser: (str) => qs.parse(str)
  }
})
```

- **Default:** [`fast-querystring`](https://github.com/anonrig/fast-querystring)
- Replace or wrap the default parser for full control over how
  `request.query` is populated.
- The function receives the raw query string (without the `?`) and must return
  an object.

### 6.8 `constraints`

```javascript
const app = Fastify({
  routerOptions: {
    constraints: {
      myHeader: {
        name: 'myHeader',
        storage () {
          const store = {}
          return {
            get: (v) => store[v] ?? null,
            set: (v, fn) => { store[v] = fn }
          }
        },
        validate (v) { return typeof v === 'string' },
        deriveConstraint (req) { return req.headers['x-my-header'] }
      }
    }
  }
})

app.get('/', { constraints: { myHeader: 'v1' } }, handlerV1)
app.get('/', { constraints: { myHeader: 'v2' } }, handlerV2)
```

- **Default:** `{}` (only built-in `version` and `host` strategies from
  `find-my-way`)
- Registers custom constraint strategies.
- Each strategy must implement `name`, `storage`, `validate`, and
  `deriveConstraint`.
- See [Routing Constraints Tutorial](./fastify-routing-constraints.tutorial.md)
  for a full deep dive.

---

## 7. Plugin and Startup Options

### 7.1 `pluginTimeout`

```javascript
const app = Fastify({ pluginTimeout: 30_000 })
```

- **Default:** `10000` (10 seconds)
- Maximum time in milliseconds that a plugin can take to complete its
  registration (i.e., call `done()` or resolve its promise).
- When exceeded, `app.ready()` rejects with an error code
  `ERR_AVVIO_PLUGIN_TIMEOUT`.
- Set to `0` to disable the check entirely.
- Increase it when plugins do slow async work during registration (database
  connections, loading configs from remotes, etc.).

### 7.2 `trustProxy`

```javascript
// Trust all proxies
const app = Fastify({ trustProxy: true })

// Trust specific CIDR
const app = Fastify({ trustProxy: '10.0.0.0/8' })

// Custom function
const app = Fastify({
  trustProxy: (address, hop) => hop === 1
})
```

- **Default:** `false`
- Tells Fastify to trust `X-Forwarded-*` headers populated by an upstream
  proxy.
- When trusted, `request.ip`, `request.ips`, `request.host`, and
  `request.protocol` are derived from these headers instead of raw socket info.
- Accepts: `boolean | string | string[] | number | Function`.
- Powered by [`@fastify/proxy-addr`](https://www.npmjs.com/package/@fastify/proxy-addr).

> **Security:** Never set `trustProxy: true` on a public-facing server without
> a verified, hardened proxy in front. Doing so lets any client spoof their IP
> address.

### 7.3 `exposeHeadRoutes`

```javascript
const app = Fastify({ exposeHeadRoutes: false })
```

- **Default:** `true`
- When `true`, Fastify automatically creates a `HEAD` sibling for every `GET`
  route registered.
- To supply a custom `HEAD` handler for a specific route, register it **before**
  the corresponding `GET`.

### 7.4 `return503OnClosing`

```javascript
const app = Fastify({ return503OnClosing: false })
```

- **Default:** `true`
- When `true`, any request arriving after `fastify.close()` has been called
  receives `503 Service Unavailable` with `Connection: close`. This signals load
  balancers to stop routing traffic here.
- When `false`, in-flight and new requests are processed normally, but still
  receive `Connection: close`.

### 7.5 `rewriteUrl`

```javascript
const app = Fastify({
  rewriteUrl (req) {
    if (req.url.startsWith('/api/v1')) {
      return req.url.replace('/api/v1', '')
    }
    return req.url
  }
})
```

- **Default:** `null`
- A sync function called **before routing**, receiving the raw
  `IncomingMessage`.
- Must return the string URL that the router should use.
- `this` inside the function is bound to the root Fastify instance.
- Apply this when you are behind a proxy that adds a prefix, and you want
  Fastify routes to remain prefix-free.
- Not encapsulated — always applied to the entire instance.

---

## 8. Validation and Serialization Options

### 8.1 `ajv`

```javascript
const app = Fastify({
  ajv: {
    customOptions: {
      removeAdditional: 'all',
      coerceTypes: 'array',
      useDefaults: true
    },
    plugins: [
      require('ajv-formats'),
      [require('ajv-keywords'), 'instanceof']
    ],
    onCreate (ajv) {
      ajv.addFormat('objectid', /^[0-9a-f]{24}$/)
    }
  }
})
```

- **Default:** `{ customOptions: {}, plugins: [] }`
- Configures the shared Ajv v8 instance used for all route schema validation.
- `customOptions` — any AJV constructor option
  ([AJV docs](https://ajv.js.org/options.html)).
- `plugins` — array of `[plugin, opts?]` tuples; plugins are applied before any
  schemas are compiled.
- `onCreate` — callback to further mutate the AJV instance (e.g. add custom
  formats or keywords).
- For a fully custom validator compiler, use `fastify.setValidatorCompiler()`.

### 8.2 `serializerOpts`

```javascript
const app = Fastify({
  serializerOpts: {
    rounding: 'ceil',
    ajv: { 
      customOptions: { strict: false }
    }
  }
})
```

- **Default:** `{}`
- Passed directly to
  [`fast-json-stringify`](https://github.com/fastify/fast-json-stringify#options),
  which Fastify uses for response serialization.
- Useful for custom number rounding, AJV options for the serializer, or schema
  reference resolution.

### 8.3 `schemaController`

```javascript
const app = Fastify({
  schemaController: {
    bucket (parentSchemas) {
      // return a custom schema bucket
      return {
        add (schema) { /* store schema */ },
        getSchema (id) { /* retrieve by $id */ },
        getSchemas () { /* return all */ }
      }
    },
    compilersFactory: {
      buildValidator (externalSchemas, ajvOptions) { /* ... */ },
      buildSerializer (externalSchemas, serializerOptions) { /* ... */ }
    }
  }
})
```

- **Default:** built-in schema controller
- Provides hooks to supply custom validator and serializer compilers at the
  schema storage layer, below even `setValidatorCompiler`.
- `bucket` — factory function for the schema store (receives parent schemas).
- `compilersFactory.buildValidator` — factory for the AJV-based validator.
- `compilersFactory.buildSerializer` — factory for the fast-json-stringify based
  serializer.

---

## 9. Error Handling Options

### 9.1 `frameworkErrors`

```javascript
const { FST_ERR_BAD_URL } = require('fastify').errorCodes

const app = Fastify({
  frameworkErrors (error, req, res) {
    if (error.code === 'FST_ERR_BAD_URL') {
      res.statusCode = 400
      res.end(JSON.stringify({ error: 'Malformed URL', url: req.url }))
    } else {
      res.statusCode = 500
      res.end(JSON.stringify({ error: 'Internal Server Error' }))
    }
  }
})
```

- **Default:** `null` (uses Fastify built-in handlers)
- Overrides how Fastify handles framework-level errors that occur before the
  route handler runs.
- Currently applicable error codes: `FST_ERR_BAD_URL`,
  `FST_ERR_ASYNC_CONSTRAINT`.
- Receives the **raw** `(err, req, res)` — not Fastify request/reply objects.

### 9.2 `clientErrorHandler`

```javascript
const app = Fastify({
  clientErrorHandler (err, socket) {
    if (err.code === 'ECONNRESET' || socket.destroyed) return

    const body = JSON.stringify({
      statusCode: 400,
      error: 'Bad Request',
      message: err.message
    })

    this.log.warn({ err }, 'client error')

    if (socket.writable) {
      socket.write(
        `HTTP/1.1 400 Bad Request\r\nContent-Length: ${body.length}\r\nContent-Type: application/json\r\n\r\n${body}`
      )
    }
    socket.destroy(err)
  }
})
```

- **Default:** handles `ECONNRESET`, `ERR_HTTP_REQUEST_TIMEOUT`, and
  `HPE_HEADER_OVERFLOW` with appropriate 400/408/431 responses.
- Listens to Node.js `clientError` events on the HTTP server — fires when the
  TCP socket emits an error (malformed request, header overflow, etc.).
- Must write a valid HTTP/1.1 response manually (raw socket, no Fastify
  request/reply).
- `this` is bound to the Fastify root instance.

### 9.3 `allowErrorHandlerOverride`

```javascript
const app = Fastify({ allowErrorHandlerOverride: true })

// In a child plugin:
app.register(async (child) => {
  child.setErrorHandler((err, req, reply) => {
    reply.send({ ok: false, reason: err.message })
  })

  // Override again inside same scope (only works with allowErrorHandlerOverride: true)
  child.setErrorHandler((err, req, reply) => {
    reply.send({ ok: false, reason: 'overridden' })
  })
})
```

- **Default:** `false`
- By default, calling `setErrorHandler` twice in the same scope throws
  `FST_ERR_ERROR_HANDLER_ALREADY_SET`.
- Set to `true` to allow re-registration in the same scope, which is useful
  in testing or hot-reload scenarios.

---

## 10. The `listen()` Call

`fastify.listen(options[, callback])` is not a constructor option but is closely
related to server configuration. It completes the startup sequence.

### 10.1 Listen Options

```javascript
await app.listen({
  port: 3000,
  host: '0.0.0.0',
  backlog: 511,
  exclusive: false,
  readableAll: false,
  writableAll: false,
  ipv6Only: false
})
```

| Option | Default | Description |
|---|---|---|
| `port` | `0` | TCP port (0 = random available) |
| `host` | `'localhost'` | Bind address |
| `backlog` | OS default | Max queued connections |
| `exclusive` | `false` | Prevent port sharing |
| `ipv6Only` | `false` | Dual-stack or IPv6-only |

When `host` is `'localhost'`, Fastify uses DNS to resolve both `127.0.0.1` and
`::1` and binds to both via secondary server instances:

```text
host: 'localhost'
  |
  +--dns.lookup--> ['127.0.0.1', '::1']
  |
  +-- primary server   listens on 127.0.0.1:3000
  '-- secondary server listens on ::1:3000
```

### 10.2 Unix Socket and File Descriptor

```javascript
// Unix domain socket
await app.listen({ path: '/tmp/fastify.sock' })

// File descriptor (pre-bound socket, useful in cluster/systemd scenarios)
await app.listen({ fd: 3 })
```

When `path` is specified, `host` is ignored.

### 10.3 AbortSignal Integration

```javascript
const ac = new AbortController()

app.listen({ port: 3000, signal: ac.signal })

// Later — triggers graceful close:
ac.abort()
```

- Fastify subscribes to the signal's `abort` event and calls `fastify.close()`
  automatically.
- Useful for process-level shutdown orchestration via `AbortController`.

---

## 11. `initialConfig` — The Sealed Snapshot

After Fastify boots, the options used to configure it are available as a **deeply
frozen** object:

```javascript
const app = Fastify({
  logger: true,
  bodyLimit: 2 * 1024 * 1024
})

await app.ready()

console.log(app.initialConfig.bodyLimit)         // 2097152
console.log(app.initialConfig.logger)            // pino instance (not original bool)

// Mutations are silently ignored (strict mode throws):
app.initialConfig.bodyLimit = 0  // no effect
```

The freeze is applied recursively by `deepFreezeObject` in
[lib/initial-config-validation.js](../../lib/initial-config-validation.js).

Properties visible on `initialConfig` are the **schema-validated** subset:

```text
connectionTimeout     keepAliveTimeout      forceCloseConnections
maxRequestsPerSocket  requestTimeout        handlerTimeout
bodyLimit             caseSensitive         allowUnsafeRegex
http2                 https                 ignoreTrailingSlash
ignoreDuplicateSlashes disableRequestLogging maxParamLength
onProtoPoisoning      onConstructorPoisoning pluginTimeout
requestIdHeader       requestIdLogLabel      http2SessionTimeout
exposeHeadRoutes      useSemicolonDelimiter  routerOptions
constraints
```

Function-typed options (`genReqId`, `serverFactory`, `querystringParser`, …)
are **not** in `initialConfig` because they cannot be safely frozen or
serialized.

---

## 12. How Options Are Processed Internally

```text
Fastify(opts)
  |
  v
processOptions(opts)                         [fastify.js]
  |
  +-- validate & coerce via AJV schema       [lib/config-validator.js]
  |     sets defaults for missing fields
  |
  +-- createLogger(opts)                     [lib/logger-factory.js]
  |     mutually exclusive: logger vs loggerInstance
  |     installs abstract-logging if both false
  |
  +-- reqIdGenFactory(requestIdHeader, genReqId)
  |     [lib/req-id-gen-factory.js]
  |     wraps user fn or builds default counter
  |
  +-- getSecuredInitialConfig(opts)          [lib/initial-config-validation.js]
  |     deepClone + deepFreeze -> frozen snapshot
  |
  +-- buildRouterOptions(opts, defaults)     [lib/route.js]
  |     merges top-level router opts into routerOptions
  |
  '-- returns { options, genReqId, disableRequestLogging, hasLogger, initialConfig }

createServer(options, httpHandler)           [lib/server.js]
  |
  +-- getServerInstance(options)
  |     selects http / https / http2 / serverFactory
  |
  +-- sets connectionTimeout, keepAliveTimeout,
  |   requestTimeout, maxRequestsPerSocket on server
  |
  +-- resolves forceCloseConnections strategy
  |   (closeAllConnections / closeIdleConnections / internal Set)
  |
  '-- returns { server, listen, forceCloseConnections, ... }
```

---

## 13. TypeScript Quick-Reference

```typescript
import Fastify, {
  FastifyInstance,
  FastifyServerOptions,
  FastifyBaseLogger,
  FastifyTypeProviderDefault
} from 'fastify'

// Typed options object
const opts: FastifyServerOptions = {
  logger: { level: 'info' },
  bodyLimit: 1024 * 1024,
  trustProxy: true,
  ajv: {
    customOptions: { removeAdditional: 'all' }
  }
}

const app: FastifyInstance = Fastify(opts)

// Access the frozen initial config
const config = app.initialConfig
//    ^? Readonly<FastifyInitialConfig>

// Custom genReqId
const appWithId = Fastify({
  genReqId (req: import('http').IncomingMessage): string {
    return req.headers['x-request-id'] as string ?? crypto.randomUUID()
  }
})

// Custom server factory
import http from 'node:http'

const appCustomServer = Fastify({
  serverFactory (handler, opts) {
    return http.createServer((req, res) => handler(req, res))
  }
})
```

---

## 14. Full Configuration Example

```javascript
'use strict'

const Fastify = require('fastify')
const fs = require('node:fs')
const qs = require('qs')

const app = Fastify({
  // --- Protocol ---
  https: {
    key:  fs.readFileSync('./tls/server.key'),
    cert: fs.readFileSync('./tls/server.cert')
  },

  // --- Timeouts ---
  connectionTimeout:    30_000,   // 30 s socket idle
  keepAliveTimeout:      5_000,   // 5 s keep-alive
  requestTimeout:      120_000,   // 2 min to receive full request
  handlerTimeout:       30_000,   // 30 s for the entire route lifecycle
  maxRequestsPerSocket:    100,   // close socket after 100 requests

  // --- Body ---
  bodyLimit:          5_242_880,  // 5 MiB
  onProtoPoisoning:    'remove',
  onConstructorPoisoning: 'error',

  // --- Logging ---
  logger: {
    level: 'info',
    serializers: {
      req: (req) => ({ method: req.method, url: req.url })
    }
  },
  requestIdHeader:     'x-request-id',
  requestIdLogLabel:   'requestId',
  genReqId: () => crypto.randomUUID(),
  disableRequestLogging: (req) => req.url === '/health',

  // --- Router ---
  routerOptions: {
    caseSensitive:          true,
    ignoreTrailingSlash:    true,
    ignoreDuplicateSlashes: true,
    maxParamLength:           500,
    querystringParser:  (str) => qs.parse(str)
  },

  // --- Startup ---
  pluginTimeout:      20_000,
  trustProxy:         '10.0.0.0/8',
  exposeHeadRoutes:    true,
  return503OnClosing:  true,
  forceCloseConnections: 'idle',

  // --- Validation ---
  ajv: {
    customOptions: {
      removeAdditional: 'all',
      coerceTypes:      'array',
      useDefaults:       true
    },
    plugins: [require('ajv-formats')]
  },

  // --- Error handling ---
  clientErrorHandler (err, socket) {
    if (err.code === 'ECONNRESET' || socket.destroyed) return
    const body = JSON.stringify({ statusCode: 400, error: err.message })
    if (socket.writable) {
      socket.write(
        `HTTP/1.1 400 Bad Request\r\nContent-Length: ${body.length}\r\nContent-Type: application/json\r\n\r\n${body}`
      )
    }
    socket.destroy(err)
  }
})

await app.listen({ port: 8443, host: '0.0.0.0' })
```

---

## 15. Common Pitfalls

### Mutating options after boot

Options are frozen. Changes after `Fastify()` returns are silently discarded.
Compute all dynamic values (from env vars, config files) **before** calling the
factory.

```javascript
// Wrong
const app = Fastify({ bodyLimit: 1024 })
app.initialConfig.bodyLimit = 2048  // no effect

// Right
const app = Fastify({ bodyLimit: Number(process.env.BODY_LIMIT) || 1048576 })
```

### Setting `requestIdHeader` without a trusted proxy

Any caller can set the header to an arbitrary value. Without a proxy stripping
or rewriting it, this allows clients to inject arbitrary strings into your logs.

### `handlerTimeout: 0` cannot opt out after a global is set

Once `handlerTimeout` has been set globally (non-zero), individual routes
cannot set it back to `0` to disable it. They can only override it with a
different positive integer.

### `logger: true` vs `disableRequestLogging: true`

`disableRequestLogging: true` only suppresses the automatic
`received request` / `request completed` log lines. It does **not** disable the
Pino logger itself — manual `request.log.info(...)` calls still work.

### Passing `logger` and `loggerInstance` together

```javascript
// Throws FST_ERR_LOG_LOGGER_AND_LOGGER_INSTANCE_PROVIDED
const app = Fastify({
  logger: true,
  loggerInstance: myPinoInstance
})
```

Use one or the other, never both.

### `trustProxy` in production without a real proxy

Enabling `trustProxy: true` without a verified load balancer in front lets
clients forge `X-Forwarded-For` headers and appear as any IP address, breaking
rate-limiting, geo-routing, and access control.

### Plugin timeout too low for slow startup

If a plugin connects to external services during `register`, the default 10 s
timeout might fire before the connection is established. Increase
`pluginTimeout` to match the expected startup latency plus a safety margin.

```javascript
const app = Fastify({ pluginTimeout: 60_000 }) // 60 s for slow DB connect
```

### `querystringParser` in wrong location

Prior to routerOptions consolidation, `querystringParser` was placed at the
top-level. Always nest it under `routerOptions`:

```javascript
// Deprecated (still works but raises a warning)
const app = Fastify({ querystringParser: qs.parse })

// Correct
const app = Fastify({ routerOptions: { querystringParser: qs.parse } })
```

---

*See also:*

- [lib/initial-config-validation.js](../../lib/initial-config-validation.js) — option schema and freeze logic
- [lib/config-validator.js](../../lib/config-validator.js) — AJV-compiled validator with all defaults
- [lib/server.js](../../lib/server.js) — HTTP server creation and listen logic
- [lib/logger-factory.js](../../lib/logger-factory.js) — logger creation and validation
- [lib/req-id-gen-factory.js](../../lib/req-id-gen-factory.js) — request ID generation
- [fastify.js](../fastify.js) — `processOptions` and the public factory
