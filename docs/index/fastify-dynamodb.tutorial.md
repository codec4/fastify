# Fastify + DynamoDB — Complete Tutorial

> **Target audience:** Developers familiar with Node.js / TypeScript and Fastify basics
> who want to integrate Amazon DynamoDB as a data layer using best practices.

---

## Table of Contents

1. [Why DynamoDB with Fastify?](#1-why-dynamodb-with-fastify)
2. [Prerequisites & Setup](#2-prerequisites--setup)
3. [Project Structure](#3-project-structure)
4. [DynamoDB Primer — Key Concepts](#4-dynamodb-primer--key-concepts)
5. [Connecting to DynamoDB via a Fastify Plugin](#5-connecting-to-dynamodb-via-a-fastify-plugin)
6. [Table Design & Schema Planning](#6-table-design--schema-planning)
7. [CRUD Operations](#7-crud-operations)
   - 7.1 [PutItem — Create](#71-putitem--create)
   - 7.2 [GetItem — Read by Key](#72-getitem--read-by-key)
   - 7.3 [UpdateItem — Partial Update](#73-updateitem--partial-update)
   - 7.4 [DeleteItem](#74-deleteitem)
   - 7.5 [Query — Range Scans](#75-query--range-scans)
   - 7.6 [Scan — Full Table](#76-scan--full-table)
8. [Route Handlers with Validation](#8-route-handlers-with-validation)
9. [Error Handling](#9-error-handling)
10. [Pagination](#10-pagination)
11. [Transactions](#11-transactions)
12. [Using a Single-Table Design](#12-using-a-single-table-design)
13. [Local Development with DynamoDB Local](#13-local-development-with-dynamodb-local)
14. [Testing](#14-testing)
15. [Real-World Patterns](#15-real-world-patterns)
    - 15.1 [Repository Pattern](#151-repository-pattern)
    - 15.2 [Optimistic Locking](#152-optimistic-locking)
    - 15.3 [TTL-Based Expiry](#153-ttl-based-expiry)
16. [TypeScript Quick-Reference](#16-typescript-quick-reference)

Related deep dives:

- [Fastify Plugins — Complete Tutorial](./fastify-plugins.tutorial.md)
- [Fastify Hooks — Complete Tutorial](./fastify-hooks.tutorial.md)
- [Fastify Validation — Complete Tutorial](./fastify-validation.tutorial.md)

---

## 1. Why DynamoDB with Fastify?

DynamoDB is a fully managed, serverless key-value and document database.  It pairs
naturally with Fastify because:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Fastify                       DynamoDB                              │
├──────────────────────────────────────────────────────────────────────┤
│  Near-zero overhead per req    Single-digit ms latency at any scale  │
│  Schema validation built-in    Schema-on-read (flexible attributes)  │
│  Plugin-based encapsulation    Per-table access patterns             │
│  Async-first                   AWS SDK v3 is async / stream-first    │
│  Lightweight process           Serverless — no connection pool       │
└──────────────────────────────────────────────────────────────────────┘
```

Unlike relational databases, DynamoDB has **no connection pool** to manage.  Each
AWS SDK call is an independent HTTPS request, which means startup is instantaneous
and you never hit "too many connections" errors under Lambda or container autoscaling.

---

## 2. Prerequisites & Setup

### 2.1 Install dependencies

```
npm install fastify @fastify/sensible fastify-plugin
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
npm install @aws-sdk/util-dynamodb          # optional — marshalling helpers
npm install --save-dev @types/node typescript tsx
```

| Package | Purpose |
|---|---|
| `@aws-sdk/client-dynamodb` | Low-level DynamoDB client (typed AttributeValues) |
| `@aws-sdk/lib-dynamodb` | `DynamoDBDocumentClient` — auto-marshals JS ↔ DynamoDB types |
| `@aws-sdk/util-dynamodb` | `marshall` / `unmarshall` utilities |

### 2.2 AWS credentials

The SDK resolves credentials in this order:

```
1. Explicit constructor options  (never hardcode in source!)
2. Environment variables         AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
3. ~/.aws/credentials file       (local dev via aws configure)
4. IAM role / ECS task role      (recommended for production)
5. SSO / Web Identity Token      (for CI / GitHub Actions OIDC)
```

For local development, run `aws configure` or export environment variables.
In production, attach an **IAM role** to your ECS task or Lambda — never ship
credentials in code.

### 2.3 TypeScript config

```
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "outDir": "dist",
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

---

## 3. Project Structure

```
my-fastify-dynamo/
├── src/
│   ├── app.ts                   ← Fastify instance factory
│   ├── server.ts                ← Entry-point (listen)
│   ├── plugins/
│   │   └── dynamodb.ts          ← DynamoDB client plugin
│   ├── repositories/
│   │   └── user.repository.ts   ← Data-access layer
│   ├── routes/
│   │   └── users.ts             ← Route definitions
│   ├── schemas/
│   │   └── user.schema.ts       ← JSON Schema shapes
│   └── types/
│       └── index.d.ts           ← Module augmentation
├── test/
│   └── users.test.ts
├── tsconfig.json
└── package.json
```

Separating **plugins** (infrastructure) from **repositories** (data access) from
**routes** (HTTP surface) keeps each concern independently testable.

---

## 4. DynamoDB Primer — Key Concepts

Before writing code, you need to understand the DynamoDB data model:

```
┌──────────────────────────────────────────────────────────────────────┐
│  TABLE                                                               │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Item (= document / row)                                       │  │
│  │  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────┐  │  │
│  │  │ Partition Key│  │  Sort Key (opt) │  │ Other attributes │  │  │
│  │  │  (PK / Hash) │  │   (SK / Range)  │  │  (any type)      │  │  │
│  │  └──────────────┘  └─────────────────┘  └──────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

| Concept | Description |
|---|---|
| **Partition Key (PK)** | Hash key; determines the physical partition. Mandatory. |
| **Sort Key (SK)** | Range key; enables range queries within a partition. Optional. |
| **Item** | A collection of attributes. Max 400 KB. |
| **GSI** | Global Secondary Index — alternate PK/SK pair for different query patterns. |
| **LSI** | Local Secondary Index — alternate SK on the same PK. Must be defined at creation. |
| **Capacity modes** | On-demand (pay per request) or Provisioned (RCU/WCU). |
| **Streams** | Ordered log of item changes; powers event-driven architectures. |

### DynamoDB wire types vs JavaScript

The raw client uses typed attribute objects.  `DynamoDBDocumentClient` from
`@aws-sdk/lib-dynamodb` handles the mapping automatically:

```
DynamoDB wire type   JS value (DocumentClient)
─────────────────    ──────────────────────────
{ S: "Alice" }       "Alice"
{ N: "42" }          42
{ BOOL: true }       true
{ NULL: true }       null
{ L: [...] }         Array
{ M: {...} }         Object (plain)
{ SS: [...] }        Set<string>  (via marshall)
{ NS: [...] }        Set<number>  (via marshall)
{ B: Buffer }        Uint8Array
```

---

## 5. Connecting to DynamoDB via a Fastify Plugin

### 5.1 The plugin

```
// src/plugins/dynamodb.ts
import fp from 'fastify-plugin'
import {
  FastifyInstance,
  FastifyPluginOptions,
} from 'fastify'
import { DynamoDBClient, DynamoDBClientConfig } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient, TranslateConfig } from '@aws-sdk/lib-dynamodb'

export interface DynamoDBPluginOptions extends FastifyPluginOptions {
  region?: string
  endpoint?: string          // set to http://localhost:8000 for DynamoDB Local
  credentials?: {
    accessKeyId: string
    secretAccessKey: string
  }
}

// Extend the Fastify instance type so every route sees `fastify.dynamo`
declare module 'fastify' {
  interface FastifyInstance {
    dynamo: DynamoDBDocumentClient
    dynamoRaw: DynamoDBClient
  }
}

async function dynamoDBPlugin(
  fastify: FastifyInstance,
  opts: DynamoDBPluginOptions
): Promise<void> {
  const clientConfig: DynamoDBClientConfig = {
    region: opts.region ?? process.env.AWS_REGION ?? 'us-east-1',
  }

  if (opts.endpoint) {
    clientConfig.endpoint = opts.endpoint
  }

  if (opts.credentials) {
    clientConfig.credentials = opts.credentials
  }

  const translateConfig: TranslateConfig = {
    marshallOptions: {
      // Convert typeof undefined to null so missing optional fields are stored
      convertEmptyValues:       false,
      removeUndefinedValues:    true,
      convertClassInstanceToMap: false,
    },
    unmarshallOptions: {
      wrapNumbers: false, // return JS number (watch for > 2^53 integers!)
    },
  }

  const rawClient = new DynamoDBClient(clientConfig)
  const docClient = DynamoDBDocumentClient.from(rawClient, translateConfig)

  // Decorate with both clients so power-users can drop to the raw client
  fastify.decorate('dynamoRaw', rawClient)
  fastify.decorate('dynamo', docClient)

  // Validate connectivity during startup (will throw and abort boot on error)
  fastify.addHook('onReady', async () => {
    const { ListTablesCommand } = await import('@aws-sdk/client-dynamodb')
    try {
      await rawClient.send(new ListTablesCommand({ Limit: 1 }))
      fastify.log.info('DynamoDB connection verified')
    } catch (err) {
      fastify.log.error({ err }, 'DynamoDB connectivity check failed')
      throw err
    }
  })

  // Clean up the underlying HTTP connections on shutdown
  fastify.addHook('onClose', async () => {
    rawClient.destroy()
    fastify.log.info('DynamoDB client destroyed')
  })
}

export default fp(dynamoDBPlugin, {
  fastify: '>=4.0.0',
  name: 'fastify-dynamodb',
})
```

### 5.2 Registering the plugin in your app

```
// src/app.ts
import Fastify, { FastifyInstance } from 'fastify'
import sensible from '@fastify/sensible'
import dynamoDBPlugin from './plugins/dynamodb.js'
import userRoutes from './routes/users.js'

export async function buildApp(): Promise<FastifyInstance> {
  const app = Fastify({
    logger: {
      level: process.env.LOG_LEVEL ?? 'info',
    },
  })

  // Infrastructure
  await app.register(sensible)           // httpErrors, assert, etc.
  await app.register(dynamoDBPlugin, {
    region:   process.env.AWS_REGION   ?? 'us-east-1',
    endpoint: process.env.DYNAMO_ENDPOINT,  // undefined in production
  })

  // Routes
  await app.register(userRoutes, { prefix: '/users' })

  return app
}
```

---

## 6. Table Design & Schema Planning

Designing a DynamoDB table is **access-pattern-first**, not entity-first.
Write down every query your application needs, then derive the key schema.

### Example: Users service

Access patterns:

```
1.  Get user by ID                      → PK=USER#<id>   SK=PROFILE
2.  List all posts by a user            → PK=USER#<id>   SK begins_with POST#
3.  Get a specific post                 → PK=USER#<id>   SK=POST#<postId>
4.  Get user by email (login flow)      → GSI: email-index PK=email
5.  List most-recently active users     → GSI: status-index PK=status SK=lastActiveAt
```

Table definition (IaC-style pseudocode):

```
TableName:  AppTable

KeySchema:
  PartitionKey:  PK   (String)
  SortKey:       SK   (String)

GlobalSecondaryIndexes:
  - IndexName:   email-index
    KeySchema:   PK=email  (projects ALL)

  - IndexName:   status-index
    KeySchema:   PK=status  SK=lastActiveAt  (projects ALL)

BillingMode: PAY_PER_REQUEST
```

### Creating the table with the SDK

```
// scripts/create-table.ts  (run once, or use CDK/Terraform in production)
import { DynamoDBClient, CreateTableCommand } from '@aws-sdk/client-dynamodb'

const client = new DynamoDBClient({
  region:   'us-east-1',
  endpoint: process.env.DYNAMO_ENDPOINT, // local dev
})

await client.send(
  new CreateTableCommand({
    TableName: 'AppTable',
    AttributeDefinitions: [
      { AttributeName: 'PK',          AttributeType: 'S' },
      { AttributeName: 'SK',          AttributeType: 'S' },
      { AttributeName: 'email',       AttributeType: 'S' },
      { AttributeName: 'status',      AttributeType: 'S' },
      { AttributeName: 'lastActiveAt', AttributeType: 'S' },
    ],
    KeySchema: [
      { AttributeName: 'PK', KeyType: 'HASH' },
      { AttributeName: 'SK', KeyType: 'RANGE' },
    ],
    GlobalSecondaryIndexes: [
      {
        IndexName: 'email-index',
        KeySchema: [{ AttributeName: 'email', KeyType: 'HASH' }],
        Projection: { ProjectionType: 'ALL' },
      },
      {
        IndexName: 'status-index',
        KeySchema: [
          { AttributeName: 'status',       KeyType: 'HASH' },
          { AttributeName: 'lastActiveAt', KeyType: 'RANGE' },
        ],
        Projection: { ProjectionType: 'ALL' },
      },
    ],
    BillingMode: 'PAY_PER_REQUEST',
  })
)

console.log('Table created')
```

---

## 7. CRUD Operations

All examples use `DynamoDBDocumentClient` (auto-marshalling).
The `TABLE` constant is `'AppTable'`.

```
// src/repositories/constants.ts
export const TABLE = process.env.DYNAMO_TABLE ?? 'AppTable'
```

### 7.1 PutItem — Create

`PutItem` unconditionally writes (or replaces) a full item.  Use a
`ConditionExpression` to enforce uniqueness.

```
// src/repositories/user.repository.ts  (excerpt)
import {
  PutCommand,
  PutCommandInput,
} from '@aws-sdk/lib-dynamodb'
import { ConditionalCheckFailedException } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'
import { randomUUID } from 'node:crypto'

export interface User {
  id:           string
  email:        string
  name:         string
  status:       'active' | 'suspended'
  lastActiveAt: string   // ISO-8601
  createdAt:    string
  version:      number
}

export async function createUser(
  dynamo: DynamoDBDocumentClient,
  input: Pick<User, 'email' | 'name'>
): Promise<User> {
  const now  = new Date().toISOString()
  const user: User = {
    id:           randomUUID(),
    email:        input.email,
    name:         input.name,
    status:       'active',
    lastActiveAt: now,
    createdAt:    now,
    version:      1,
  }

  const params: PutCommandInput = {
    TableName:           TABLE,
    Item: {
      PK:    `USER#${user.id}`,
      SK:    'PROFILE',
      _type: 'User',
      ...user,
    },
    // Prevent overwriting an existing item with the same PK+SK
    ConditionExpression: 'attribute_not_exists(PK)',
  }

  try {
    await dynamo.send(new PutCommand(params))
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new Error(`User ${user.id} already exists`)
    }
    throw err
  }

  return user
}
```

> **Tip:** `attribute_not_exists(PK)` is the DynamoDB idiom for "INSERT OR FAIL".
> If the item already exists the call throws `ConditionalCheckFailedException`.

### 7.2 GetItem — Read by Key

`GetItem` retrieves **exactly one** item by its full primary key.
It is the cheapest and fastest read operation — always prefer it over Query/Scan when
you know both keys.

```
import {
  GetCommand,
  GetCommandOutput,
} from '@aws-sdk/lib-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'
import type { User } from './user.repository.js'

export async function getUserById(
  dynamo: DynamoDBDocumentClient,
  id: string
): Promise<User | null> {
  const result: GetCommandOutput = await dynamo.send(
    new GetCommand({
      TableName: TABLE,
      Key: {
        PK: `USER#${id}`,
        SK: 'PROFILE',
      },
      // Strong consistency ensures you always read the latest value.
      // Omit (or set false) for eventually consistent reads (cheaper, ~50% cost).
      ConsistentRead: true,
    })
  )

  if (!result.Item) return null

  // Strip DynamoDB-internal keys before returning the domain object
  const { PK, SK, _type, ...user } = result.Item
  return user as User
}
```

### 7.3 UpdateItem — Partial Update

`UpdateItem` modifies specific attributes **in place** without fetching the full item
first.  This is the preferred way to update — it is atomic and cheaper than a
Get → mutate → Put round trip.

```ts
import {
  UpdateCommand,
  UpdateCommandInput,
} from '@aws-sdk/lib-dynamodb'
import { ConditionalCheckFailedException } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'
import type { User } from './user.repository.js'

export async function updateUserName(
  dynamo: DynamoDBDocumentClient,
  id:     string,
  name:   string,
  currentVersion: number
): Promise<void> {
  const params: UpdateCommandInput = {
    TableName: TABLE,
    Key: {
      PK: `USER#${id}`,
      SK: 'PROFILE',
    },
    // UpdateExpression uses SET, REMOVE, ADD, DELETE clauses
    UpdateExpression:
      'SET #name = :name, lastActiveAt = :now, version = :newVersion',
    // Optimistic locking — only update if version has not changed
    ConditionExpression: 'version = :currentVersion',
    ExpressionAttributeNames: {
      '#name': 'name',   // 'name' is not a reserved word but '#' is idiomatic
    },
    ExpressionAttributeValues: {
      ':name':           name,
      ':now':            new Date().toISOString(),
      ':newVersion':     currentVersion + 1,
      ':currentVersion': currentVersion,
    },
  }

  try {
    await dynamo.send(new UpdateCommand(params))
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new Error('Concurrent update detected — please retry')
    }
    throw err
  }
}
```

> **UpdateExpression cheat-sheet:**
>
> | Clause   | Effect                                      |
> |----------|---------------------------------------------|
> | `SET`    | Add / overwrite attribute values            |
> | `REMOVE` | Delete one or more attributes               |
> | `ADD`    | Increment a Number / add items to a Set     |
> | `DELETE` | Remove items from a Set                     |
>
> Clauses can be combined: `SET a = :a REMOVE b ADD c :n`

### 7.4 DeleteItem

```ts
import { DeleteCommand } from '@aws-sdk/lib-dynamodb'
import { ConditionalCheckFailedException } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'

export async function deleteUser(
  dynamo: DynamoDBDocumentClient,
  id:     string
): Promise<void> {
  try {
    await dynamo.send(
      new DeleteCommand({
        TableName: TABLE,
        Key: {
          PK: `USER#${id}`,
          SK: 'PROFILE',
        },
        // Guard against deleting a non-existent item if you need that signal
        ConditionExpression: 'attribute_exists(PK)',
        // ReturnValues: 'ALL_OLD'   ← uncomment to get the deleted item back
      })
    )
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new Error(`User ${id} not found`)
    }
    throw err
  }
}
```

### 7.5 Query — Range Scans

`Query` retrieves **all items sharing the same partition key**, optionally filtered by
sort key conditions.  It reads only one partition and is far cheaper than `Scan`.

```ts
import {
  QueryCommand,
  QueryCommandInput,
} from '@aws-sdk/lib-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'

export interface Post {
  postId:    string
  userId:    string
  title:     string
  body:      string
  createdAt: string
}

// Returns all posts for a user, newest first
export async function getPostsByUser(
  dynamo:  DynamoDBDocumentClient,
  userId:  string,
  limit?:  number
): Promise<Post[]> {
  const params: QueryCommandInput = {
    TableName:              TABLE,
    KeyConditionExpression: 'PK = :pk AND begins_with(SK, :skPrefix)',
    ExpressionAttributeValues: {
      ':pk':       `USER#${userId}`,
      ':skPrefix': 'POST#',
    },
    ScanIndexForward: false,   // false = descending sort key order (newest first)
    Limit:            limit,
  }

  const result = await dynamo.send(new QueryCommand(params))
  return (result.Items ?? []).map(({ PK, SK, _type, ...post }) => post as Post)
}

// Query a GSI — get user by email
export async function getUserByEmail(
  dynamo: DynamoDBDocumentClient,
  email:  string
): Promise<import('./user.repository.js').User | null> {
  const result = await dynamo.send(
    new QueryCommand({
      TableName:              TABLE,
      IndexName:              'email-index',
      KeyConditionExpression: 'email = :email',
      ExpressionAttributeValues: { ':email': email },
      Limit: 1,
    })
  )

  const item = result.Items?.[0]
  if (!item) return null
  const { PK, SK, _type, ...user } = item
  return user as import('./user.repository.js').User
}
```

### 7.6 Scan — Full Table

`Scan` reads **every item** in the table (or index).  It is expensive and slow;
use it only for administrative tooling or data migrations, never in hot paths.

```ts
import { ScanCommand } from '@aws-sdk/lib-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'

// Example: find all suspended users (admin script — NOT a hot path)
export async function scanSuspendedUsers(
  dynamo: DynamoDBDocumentClient
): Promise<import('./user.repository.js').User[]> {
  const users: import('./user.repository.js').User[] = []
  let lastKey: Record<string, unknown> | undefined

  do {
    const result = await dynamo.send(
      new ScanCommand({
        TableName:        TABLE,
        FilterExpression: '#status = :status AND #type = :type',
        ExpressionAttributeNames: {
          '#status': 'status',
          '#type':   '_type',
        },
        ExpressionAttributeValues: {
          ':status': 'suspended',
          ':type':   'User',
        },
        ExclusiveStartKey: lastKey,
      })
    )

    for (const item of result.Items ?? []) {
      const { PK, SK, _type, ...user } = item
      users.push(user as import('./user.repository.js').User)
    }

    lastKey = result.LastEvaluatedKey as Record<string, unknown> | undefined
  } while (lastKey)

  return users
}
```

> **Warning:** `FilterExpression` does **not** reduce consumed capacity — items are
> read first, then filtered.  For frequent access patterns, model the filter as a
> GSI key instead.

---

## 8. Route Handlers with Validation

### 8.1 JSON Schemas

Define schemas once and share them between validation and documentation:

```ts
// src/schemas/user.schema.ts
import { FastifySchema } from 'fastify'

export const createUserBody = {
  type: 'object',
  required: ['email', 'name'],
  additionalProperties: false,
  properties: {
    email: { type: 'string', format: 'email', maxLength: 320 },
    name:  { type: 'string', minLength: 1, maxLength: 100 },
  },
} as const

export const userParams = {
  type: 'object',
  required: ['id'],
  properties: {
    id: { type: 'string', format: 'uuid' },
  },
} as const

export const userResponse = {
  type: 'object',
  properties: {
    id:           { type: 'string' },
    email:        { type: 'string' },
    name:         { type: 'string' },
    status:       { type: 'string', enum: ['active', 'suspended'] },
    lastActiveAt: { type: 'string' },
    createdAt:    { type: 'string' },
    version:      { type: 'number' },
  },
} as const

export const createUserSchema: FastifySchema = {
  body:     createUserBody,
  response: { 201: userResponse },
}

export const getUserSchema: FastifySchema = {
  params:   userParams,
  response: { 200: userResponse },
}

export const updateUserSchema: FastifySchema = {
  params: userParams,
  body: {
    type: 'object',
    minProperties: 1,
    additionalProperties: false,
    properties: {
      name:    { type: 'string', minLength: 1, maxLength: 100 },
      version: { type: 'integer', minimum: 1 },
    },
    required: ['version'],
  },
  response: { 204: { type: 'null' } },
}
```

### 8.2 Route file

```ts
// src/routes/users.ts
import { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify'
import {
  createUser,
  getUserById,
  deleteUser,
} from '../repositories/user.repository.js'
import { updateUserName } from '../repositories/user.update.js'
import {
  createUserSchema,
  getUserSchema,
  updateUserSchema,
} from '../schemas/user.schema.js'

interface CreateBody {
  email: string
  name:  string
}

interface UserParams {
  id: string
}

interface UpdateBody {
  name?:   string
  version: number
}

export default async function userRoutes(fastify: FastifyInstance): Promise<void> {
  // POST /users
  fastify.post<{ Body: CreateBody }>(
    '/',
    { schema: createUserSchema },
    async (request: FastifyRequest<{ Body: CreateBody }>, reply: FastifyReply) => {
      const user = await createUser(fastify.dynamo, request.body)
      return reply.code(201).send(user)
    }
  )

  // GET /users/:id
  fastify.get<{ Params: UserParams }>(
    '/:id',
    { schema: getUserSchema },
    async (request, reply) => {
      const user = await getUserById(fastify.dynamo, request.params.id)
      if (!user) {
        throw fastify.httpErrors.notFound(`User ${request.params.id} not found`)
      }
      return user
    }
  )

  // PATCH /users/:id
  fastify.patch<{ Params: UserParams; Body: UpdateBody }>(
    '/:id',
    { schema: updateUserSchema },
    async (request, reply) => {
      const { id }      = request.params
      const { name, version } = request.body

      if (name) {
        await updateUserName(fastify.dynamo, id, name, version)
      }

      return reply.code(204).send()
    }
  )

  // DELETE /users/:id
  fastify.delete<{ Params: UserParams }>(
    '/:id',
    { schema: { params: getUserSchema.params } },
    async (request, reply) => {
      await deleteUser(fastify.dynamo, request.params.id)
      return reply.code(204).send()
    }
  )
}
```

---

## 9. Error Handling

### 9.1 Map DynamoDB errors to HTTP errors

```ts
// src/lib/dynamo-errors.ts
import {
  ConditionalCheckFailedException,
  ProvisionedThroughputExceededException,
  RequestLimitExceeded,
  ResourceNotFoundException,
  TransactionConflictException,
} from '@aws-sdk/client-dynamodb'
import { FastifyError } from 'fastify'
import createError from '@fastify/error'

const ConflictError  = createError('FST_DYNAMO_CONFLICT',   '%s', 409)
const TooManyReqs    = createError('FST_DYNAMO_THROTTLE',   '%s', 429)
const NotFoundError  = createError('FST_DYNAMO_NOT_FOUND',  '%s', 404)
const TxConflict     = createError('FST_DYNAMO_TX_CONFLICT','%s', 409)

export function mapDynamoError(err: unknown, context?: string): FastifyError | unknown {
  const msg = context ? `${context}: ${(err as Error).message}` : (err as Error).message

  if (err instanceof ConditionalCheckFailedException) return new ConflictError(msg)
  if (err instanceof ProvisionedThroughputExceededException) return new TooManyReqs(msg)
  if (err instanceof RequestLimitExceeded) return new TooManyReqs(msg)
  if (err instanceof ResourceNotFoundException) return new NotFoundError(msg)
  if (err instanceof TransactionConflictException) return new TxConflict(msg)

  return err   // unknown — let Fastify's default handler deal with it
}
```

### 9.2 Global error hook

```ts
// in src/app.ts — inside buildApp()
app.setErrorHandler((err, request, reply) => {
  // Log 5xx as error, 4xx as warn
  const level = (err.statusCode ?? 500) >= 500 ? 'error' : 'warn'
  request.log[level]({ err }, err.message)

  const statusCode = err.statusCode ?? 500
  return reply.code(statusCode).send({
    error:   err.code    ?? 'INTERNAL_SERVER_ERROR',
    message: err.message ?? 'An unexpected error occurred',
    ...(process.env.NODE_ENV !== 'production' && { stack: err.stack }),
  })
})
```

---

## 10. Pagination

DynamoDB paginates via **`LastEvaluatedKey`** / **`ExclusiveStartKey`** — a token
that represents the last evaluated item.  Expose this as a cursor to API consumers.

```ts
// src/repositories/user.paginate.ts
import { QueryCommand } from '@aws-sdk/lib-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { marshall, unmarshall } from '@aws-sdk/util-dynamodb'
import { TABLE } from './constants.js'
import type { Post } from './user.repository.js'

export interface PageResult<T> {
  items:      T[]
  nextCursor: string | null   // base64-encoded LastEvaluatedKey
}

function encodeCursor(key: Record<string, unknown>): string {
  return Buffer.from(JSON.stringify(key)).toString('base64url')
}

function decodeCursor(cursor: string): Record<string, unknown> {
  return JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8'))
}

export async function listUserPosts(
  dynamo:  DynamoDBDocumentClient,
  userId:  string,
  limit:   number = 20,
  cursor?: string
): Promise<PageResult<Post>> {
  const exclusiveStartKey = cursor ? decodeCursor(cursor) : undefined

  const result = await dynamo.send(
    new QueryCommand({
      TableName:              TABLE,
      KeyConditionExpression: 'PK = :pk AND begins_with(SK, :skPrefix)',
      ExpressionAttributeValues: {
        ':pk':       `USER#${userId}`,
        ':skPrefix': 'POST#',
      },
      ScanIndexForward:  false,
      Limit:             limit,
      ExclusiveStartKey: exclusiveStartKey,
    })
  )

  const items = (result.Items ?? []).map(
    ({ PK, SK, _type, ...post }) => post as Post
  )

  const nextCursor = result.LastEvaluatedKey
    ? encodeCursor(result.LastEvaluatedKey as Record<string, unknown>)
    : null

  return { items, nextCursor }
}
```

### Route with cursor pagination

```ts
// GET /users/:id/posts?limit=20&cursor=<token>
fastify.get<{
  Params: { id: string }
  Querystring: { limit?: number; cursor?: string }
}>(
  '/:id/posts',
  {
    schema: {
      params:      { type: 'object', properties: { id: { type: 'string' } } },
      querystring: {
        type: 'object',
        properties: {
          limit:  { type: 'integer', minimum: 1, maximum: 100, default: 20 },
          cursor: { type: 'string' },
        },
      },
    },
  },
  async (request, reply) => {
    const { id }             = request.params
    const { limit, cursor }  = request.query

    const page = await listUserPosts(fastify.dynamo, id, limit, cursor)

    return reply.send({
      data:       page.items,
      nextCursor: page.nextCursor,
    })
  }
)
```

---

## 11. Transactions

DynamoDB **TransactWrite** executes up to **100 actions** atomically across multiple
items (even across tables).  **TransactGet** reads up to 100 items atomically.

```ts
// src/repositories/order.repository.ts
import {
  TransactWriteCommand,
  TransactWriteCommandInput,
} from '@aws-sdk/lib-dynamodb'
import {
  TransactionCanceledException,
} from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'
import { randomUUID } from 'node:crypto'

export interface Order {
  orderId:   string
  userId:    string
  productId: string
  quantity:  number
  createdAt: string
}

/**
 * Place an order and decrement product inventory in one atomic write.
 * Fails if the product does not have enough stock (version check).
 */
export async function placeOrder(
  dynamo:    DynamoDBDocumentClient,
  userId:    string,
  productId: string,
  quantity:  number,
  currentStock: number
): Promise<Order> {
  const orderId = randomUUID()
  const now     = new Date().toISOString()

  const order: Order = { orderId, userId, productId, quantity, createdAt: now }

  const params: TransactWriteCommandInput = {
    TransactItems: [
      // 1. Write the new order item
      {
        Put: {
          TableName:           TABLE,
          Item: {
            PK:    `ORDER#${orderId}`,
            SK:    'DETAIL',
            _type: 'Order',
            ...order,
          },
          ConditionExpression: 'attribute_not_exists(PK)',
        },
      },
      // 2. Decrement stock atomically, guarded by current stock value
      {
        Update: {
          TableName:        TABLE,
          Key: { PK: `PRODUCT#${productId}`, SK: 'DETAIL' },
          UpdateExpression: 'SET stock = stock - :qty',
          ConditionExpression:
            'stock = :currentStock AND stock >= :qty',
          ExpressionAttributeValues: {
            ':qty':          quantity,
            ':currentStock': currentStock,
          },
        },
      },
    ],
    // Idempotency token — safe to retry on network errors
    ClientRequestToken: `order-${orderId}`,
  }

  try {
    await dynamo.send(new TransactWriteCommand(params))
  } catch (err) {
    if (err instanceof TransactionCanceledException) {
      const reasons = err.CancellationReasons ?? []
      const stockFail = reasons[1]?.Code === 'ConditionalCheckFailed'
      if (stockFail) {
        throw new Error('Insufficient stock or concurrent order conflict')
      }
    }
    throw err
  }

  return order
}
```

---

## 12. Using a Single-Table Design

Single-table design (STD) stores **all entity types in one table**, using prefixed
PK/SK values to keep them apart.  This minimises read costs and enables
relationship queries without joins.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Single AppTable — item examples                                        │
├──────────────────┬──────────────────┬──────────────────────────────────-┤
│  PK              │  SK              │  Other attributes                 │
├──────────────────┼──────────────────┼──────────────────────────────────-┤
│  USER#abc        │  PROFILE         │  name, email, status, version     │
│  USER#abc        │  POST#2024-01-01 │  title, body, tags                │
│  USER#abc        │  POST#2024-02-15 │  title, body, tags                │
│  PRODUCT#xyz     │  DETAIL          │  name, price, stock               │
│  ORDER#def       │  DETAIL          │  userId, productId, quantity      │
│  SESSION#tok123  │  USER#abc        │  expiresAt (TTL)                  │
└──────────────────┴──────────────────┴──────────────────────────────────-┘
```

### Generic item helpers

```ts
// src/lib/dynamo-helpers.ts
import {
  PutCommand,
  GetCommand,
  QueryCommand,
  DeleteCommand,
} from '@aws-sdk/lib-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from '../repositories/constants.js'

type ItemKey = { PK: string; SK: string }
type DynamoItem = Record<string, unknown>

// Strip internal keys from a raw DynamoDB item
export function stripKeys<T>(item: DynamoItem): T {
  const { PK, SK, _type, ...rest } = item
  return rest as T
}

export async function putItem(
  dynamo: DynamoDBDocumentClient,
  item:   DynamoItem & ItemKey,
  opts?: { conditionExpression?: string }
): Promise<void> {
  await dynamo.send(
    new PutCommand({
      TableName:           TABLE,
      Item:                item,
      ConditionExpression: opts?.conditionExpression,
    })
  )
}

export async function getItem<T>(
  dynamo: DynamoDBDocumentClient,
  key:    ItemKey
): Promise<T | null> {
  const result = await dynamo.send(
    new GetCommand({ TableName: TABLE, Key: key, ConsistentRead: true })
  )
  return result.Item ? stripKeys<T>(result.Item) : null
}

export async function queryItems<T>(
  dynamo:  DynamoDBDocumentClient,
  pk:      string,
  skPrefix?: string
): Promise<T[]> {
  const result = await dynamo.send(
    new QueryCommand({
      TableName:              TABLE,
      KeyConditionExpression: skPrefix
        ? 'PK = :pk AND begins_with(SK, :sk)'
        : 'PK = :pk',
      ExpressionAttributeValues: {
        ':pk': pk,
        ...(skPrefix ? { ':sk': skPrefix } : {}),
      },
    })
  )
  return (result.Items ?? []).map(stripKeys<T>)
}
```

---

## 13. Local Development with DynamoDB Local

Amazon provides an official local DynamoDB Docker image for offline development.

### docker-compose.yml

```yaml
version: '3.9'

services:
  dynamodb-local:
    image: amazon/dynamodb-local:latest
    container_name: dynamodb-local
    ports:
      - '8000:8000'
    command: '-jar DynamoDBLocal.jar -sharedDb -inMemory'
    healthcheck:
      test: ['CMD-SHELL', 'curl -sf http://localhost:8000 || exit 1']
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build: .
    environment:
      AWS_REGION:         us-east-1
      AWS_ACCESS_KEY_ID:  local
      AWS_SECRET_ACCESS_KEY: local
      DYNAMO_ENDPOINT:    http://dynamodb-local:8000
      DYNAMO_TABLE:       AppTable
    ports:
      - '3000:3000'
    depends_on:
      dynamodb-local:
        condition: service_healthy
```

### Environment-aware plugin registration

```ts
// in src/app.ts
await app.register(dynamoDBPlugin, {
  region:   process.env.AWS_REGION        ?? 'us-east-1',
  endpoint: process.env.DYNAMO_ENDPOINT,     // undefined in production
  ...(process.env.DYNAMO_ENDPOINT
    ? {
        credentials: {
          accessKeyId:     process.env.AWS_ACCESS_KEY_ID     ?? 'local',
          secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY ?? 'local',
        },
      }
    : {}),
})
```

### Seed script

```ts
// scripts/seed.ts  — run after create-table.ts
import { buildApp } from '../src/app.js'

const app = await buildApp()
await app.ready()

const { createUser } = await import('../src/repositories/user.repository.js')
await createUser(app.dynamo, { name: 'Alice',   email: 'alice@example.com' })
await createUser(app.dynamo, { name: 'Bob',     email: 'bob@example.com' })
await createUser(app.dynamo, { name: 'Charlie', email: 'charlie@example.com' })

console.log('Seeded 3 users')
await app.close()
```

---

## 14. Testing

### 14.1 Integration tests with DynamoDB Local

```ts
// test/users.test.ts
import { describe, it, before, after } from 'node:test'
import assert from 'node:assert/strict'
import { buildApp } from '../src/app.js'
import type { FastifyInstance } from 'fastify'

describe('User routes', () => {
  let app: FastifyInstance

  before(async () => {
    process.env.AWS_REGION             = 'us-east-1'
    process.env.AWS_ACCESS_KEY_ID      = 'local'
    process.env.AWS_SECRET_ACCESS_KEY  = 'local'
    process.env.DYNAMO_ENDPOINT        = 'http://localhost:8000'
    process.env.DYNAMO_TABLE           = 'AppTable-test'

    app = await buildApp()
    await app.ready()
  })

  after(async () => {
    await app.close()
  })

  it('creates a user and retrieves it', async () => {
    // Create
    const createRes = await app.inject({
      method:  'POST',
      url:     '/users',
      payload: { name: 'Alice', email: 'alice@test.com' },
    })
    assert.equal(createRes.statusCode, 201)

    const user = createRes.json<{ id: string; name: string; email: string }>()
    assert.equal(user.name,  'Alice')
    assert.equal(user.email, 'alice@test.com')
    assert.ok(user.id, 'id should be set')

    // Retrieve
    const getRes = await app.inject({
      method: 'GET',
      url:    `/users/${user.id}`,
    })
    assert.equal(getRes.statusCode, 200)
    assert.equal(getRes.json<{ name: string }>().name, 'Alice')
  })

  it('returns 404 for an unknown user', async () => {
    const res = await app.inject({
      method: 'GET',
      url:    '/users/00000000-0000-0000-0000-000000000000',
    })
    assert.equal(res.statusCode, 404)
  })

  it('deletes a user', async () => {
    const createRes = await app.inject({
      method:  'POST',
      url:     '/users',
      payload: { name: 'Bob', email: 'bob@test.com' },
    })
    const { id } = createRes.json<{ id: string }>()

    const deleteRes = await app.inject({ method: 'DELETE', url: `/users/${id}` })
    assert.equal(deleteRes.statusCode, 204)

    const getRes = await app.inject({ method: 'GET', url: `/users/${id}` })
    assert.equal(getRes.statusCode, 404)
  })
})
```

Run tests with DynamoDB Local running:

```
docker compose up dynamodb-local -d
npx tsx scripts/create-table.ts   # run once per test table
node --test --require tsx/esm test/**/*.test.ts
```

### 14.2 Unit tests with mocked DocumentClient

For pure unit tests you can mock the `dynamo` decorator:

```
// test/unit/user.repository.test.ts
import { describe, it, mock } from 'node:test'
import assert from 'node:assert/strict'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'

describe('getUserById', () => {
  it('returns null when item is missing', async () => {
    // Minimal mock — only `.send()` needs to be faked
    const mockSend = mock.fn(async () => ({ Item: undefined }))
    const fakeDynamo = { send: mockSend } as unknown as DynamoDBDocumentClient

    const { getUserById } = await import('../../src/repositories/user.repository.js')
    const result = await getUserById(fakeDynamo, 'non-existent-id')

    assert.equal(result, null)
    assert.equal(mockSend.mock.calls.length, 1)
  })
})
```

---

## 15. Real-World Patterns

### 15.1 Repository Pattern

Encapsulate all DynamoDB access behind a repository interface so routes are never
aware of the underlying storage technology.  This also makes mocking trivial.

```
// src/repositories/user.repository.ts  (interface + class)
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import type { User } from '../types/index.js'

export interface IUserRepository {
  create(input: Pick<User, 'email' | 'name'>): Promise<User>
  findById(id: string): Promise<User | null>
  findByEmail(email: string): Promise<User | null>
  update(id: string, patch: Partial<User>, version: number): Promise<void>
  delete(id: string): Promise<void>
}

export class DynamoUserRepository implements IUserRepository {
  constructor(private readonly dynamo: DynamoDBDocumentClient) {}

  async create(input: Pick<User, 'email' | 'name'>): Promise<User> {
    return createUser(this.dynamo, input)
  }

  async findById(id: string): Promise<User | null> {
    return getUserById(this.dynamo, id)
  }

  async findByEmail(email: string): Promise<User | null> {
    return getUserByEmail(this.dynamo, email)
  }

  async update(id: string, patch: Partial<User>, version: number): Promise<void> {
    if (patch.name) await updateUserName(this.dynamo, id, patch.name, version)
  }

  async delete(id: string): Promise<void> {
    return deleteUser(this.dynamo, id)
  }
}
```

Register the repository as a Fastify decorator so routes access it via
`fastify.userRepo`:

```
// src/plugins/repositories.ts
import fp from 'fastify-plugin'
import { FastifyInstance } from 'fastify'
import { DynamoUserRepository, IUserRepository } from '../repositories/user.repository.js'

declare module 'fastify' {
  interface FastifyInstance {
    userRepo: IUserRepository
  }
}

export default fp(async function repositoriesPlugin(fastify: FastifyInstance) {
  fastify.decorate('userRepo', new DynamoUserRepository(fastify.dynamo))
}, { name: 'repositories', dependencies: ['fastify-dynamodb'] })
```

### 15.2 Optimistic Locking

Use a `version` attribute to prevent lost updates when multiple processes may write
the same item concurrently.

```
// src/lib/optimistic-lock.ts
import { UpdateCommand } from '@aws-sdk/lib-dynamodb'
import { ConditionalCheckFailedException } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from '../repositories/constants.js'

export class OptimisticLockError extends Error {
  readonly code = 'OPTIMISTIC_LOCK_CONFLICT'
  constructor(entity: string, id: string) {
    super(`${entity} ${id} was modified by another process — please retry`)
  }
}

/**
 * Generic version-guarded update.
 * Pass any SET clause and this function enforces the version check.
 */
export async function versionedUpdate(
  dynamo:   DynamoDBDocumentClient,
  pk:       string,
  sk:       string,
  setClauses: string,
  values:   Record<string, unknown>,
  version:  number
): Promise<void> {
  try {
    await dynamo.send(
      new UpdateCommand({
        TableName:        TABLE,
        Key:              { PK: pk, SK: sk },
        UpdateExpression: `SET ${setClauses}, version = :newVersion`,
        ConditionExpression: 'version = :version',
        ExpressionAttributeValues: {
          ...values,
          ':version':    version,
          ':newVersion': version + 1,
        },
      })
    )
  } catch (err) {
    if (err instanceof ConditionalCheckFailedException) {
      throw new OptimisticLockError(pk.split('#')[0], pk.split('#')[1])
    }
    throw err
  }
}
```

### 15.3 TTL-Based Expiry

DynamoDB can automatically delete items when a **TTL attribute** (a Unix epoch
timestamp in seconds) passes.  Use this for sessions, caches, and temporary tokens.

```
// src/repositories/session.repository.ts
import { PutCommand, DeleteCommand } from '@aws-sdk/lib-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from './constants.js'
import { randomBytes } from 'node:crypto'

const SESSION_TTL_SECONDS = 60 * 60 * 24 * 7  // 7 days

export interface Session {
  token:     string
  userId:    string
  createdAt: string
  expiresAt: number  // Unix epoch seconds — DynamoDB TTL attribute
}

export async function createSession(
  dynamo: DynamoDBDocumentClient,
  userId: string
): Promise<Session> {
  const token     = randomBytes(32).toString('hex')
  const now       = Math.floor(Date.now() / 1000)
  const expiresAt = now + SESSION_TTL_SECONDS

  const session: Session = {
    token,
    userId,
    createdAt: new Date().toISOString(),
    expiresAt,
  }

  await dynamo.send(
    new PutCommand({
      TableName: TABLE,
      Item: {
        PK:    `SESSION#${token}`,
        SK:    `USER#${userId}`,
        _type: 'Session',
        ...session,
        // DynamoDB reads this attribute name from the table's TTL configuration.
        // Enable TTL on the table: aws dynamodb update-time-to-live \
        //   --table-name AppTable --time-to-live-specification \
        //   "Enabled=true, AttributeName=expiresAt"
        expiresAt,
      },
    })
  )

  return session
}

export async function deleteSession(
  dynamo: DynamoDBDocumentClient,
  token:  string,
  userId: string
): Promise<void> {
  await dynamo.send(
    new DeleteCommand({
      TableName: TABLE,
      Key: {
        PK: `SESSION#${token}`,
        SK: `USER#${userId}`,
      },
    })
  )
}
```

> **Note:** TTL deletions happen within 48 hours of the expiry time — they are
> **not** guaranteed to be instant.  Always check `expiresAt` in your application
> logic for security-sensitive use cases.

---

## 16. TypeScript Quick-Reference

### Plugin type augmentation

```
// src/types/index.d.ts
import 'fastify'
import type { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import type { DynamoDBClient }         from '@aws-sdk/client-dynamodb'
import type { IUserRepository }        from '../repositories/user.repository.js'

declare module 'fastify' {
  interface FastifyInstance {
    dynamo:     DynamoDBDocumentClient
    dynamoRaw:  DynamoDBClient
    userRepo:   IUserRepository
  }
}
```

### Typed command helpers

```
// Typed wrappers to avoid repeating TableName everywhere
import {
  GetCommand, PutCommand, UpdateCommand, DeleteCommand, QueryCommand,
  GetCommandInput, PutCommandInput, UpdateCommandInput,
  DeleteCommandInput, QueryCommandInput,
} from '@aws-sdk/lib-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { TABLE } from '../repositories/constants.js'

type Omit_TableName<T> = Omit<T, 'TableName'>

export const db = (dynamo: DynamoDBDocumentClient) => ({
  get:    (input: Omit_TableName<GetCommandInput>) =>
            dynamo.send(new GetCommand({ TableName: TABLE, ...input })),
  put:    (input: Omit_TableName<PutCommandInput>) =>
            dynamo.send(new PutCommand({ TableName: TABLE, ...input })),
  update: (input: Omit_TableName<UpdateCommandInput>) =>
            dynamo.send(new UpdateCommand({ TableName: TABLE, ...input })),
  delete: (input: Omit_TableName<DeleteCommandInput>) =>
            dynamo.send(new DeleteCommand({ TableName: TABLE, ...input })),
  query:  (input: Omit_TableName<QueryCommandInput>) =>
            dynamo.send(new QueryCommand({ TableName: TABLE, ...input })),
})

// Usage:
// const ops = db(fastify.dynamo)
// const result = await ops.get({ Key: { PK: 'USER#1', SK: 'PROFILE' } })
```

### Common error codes reference

```
┌──────────────────────────────────────────────────────────────────────────┐
│  AWS SDK Error Class                    HTTP  Retry?  Meaning            │
├──────────────────────────────────────────────────────────────────────────┤
│  ConditionalCheckFailedException         400   No     Condition failed   │
│  ProvisionedThroughputExceededException  400   Yes    Throttled          │
│  RequestLimitExceeded                    400   Yes    API call throttled │
│  ResourceNotFoundException               400   No     Table missing      │
│  TransactionCanceledException            400   Maybe  TX rolled back     │
│  TransactionConflictException            400   Yes    Concurrent TX      │
│  ItemCollectionSizeLimitExceededException400   No     LSI partition full │
│  ValidationException                     400   No     Bad request shape  │
│  InternalServerError                     500   Yes    AWS-side error     │
└──────────────────────────────────────────────────────────────────────────┘
```

### Recommended `ExpressionAttributeNames` — reserved words

DynamoDB has ~573 reserved words.  Always alias attributes that collide:

```
// Common ones you WILL hit
ExpressionAttributeNames: {
  '#name':      'name',       // RESERVED
  '#status':    'status',     // RESERVED
  '#data':      'data',       // RESERVED
  '#role':      'role',       // RESERVED
  '#timestamp': 'timestamp',  // RESERVED
  '#count':     'count',      // RESERVED
  '#type':      '_type',      // custom convention
}
```

---

*Tutorial complete.*  You now have a full working pattern for connecting Fastify to
DynamoDB — from plugin setup and table design through CRUD, pagination, transactions,
single-table design, local development, testing, and production-ready patterns.

For further reading, see the
[AWS DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
and the
[AWS SDK v3 for JavaScript reference](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/).
