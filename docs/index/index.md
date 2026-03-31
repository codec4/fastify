# Fastify Tutorials Index

This folder contains long-form Fastify deep dives. These tutorials are best used
as a guided study track after you already know the basic Fastify API surface.

If you are completely new to Fastify, start with
[Getting Started](../docs/Guides/Getting-Started.md) first, then come back here.

## How To Read These Tutorials

Use this folder in layers instead of reading files at random.

1. Read the core mental-model tutorials first.
2. Read the API-focused tutorials for request and reply behavior.
3. Read the infrastructure tutorials for validation, parsing, and errors.
4. Finish with advanced routing or ecosystem-specific material.

## Recommended Reading Order

### Phase 1: Core Concepts

1. [Fastify Lifecycle](./fastify-lifecycle.tutorial.md)
   Learn startup, shutdown, and per-request execution flow.
2. [Fastify Hooks](./fastify-hooks.tutorial.md)
   Understand where to place cross-cutting logic.
3. [Fastify Encapsulation](./fastify-encapsulation.tutorial.md)
   Learn scope boundaries between parent and child plugins.
4. [Fastify Plugins](./fastify-plugins.tutorial.md)
   See how Fastify applications are composed.
5. [Fastify Decorators](./fastify-decorators.tutorial.md)
   Learn how shared services and request/reply extensions are exposed.

### Phase 2: Request/Response Flow

6. [Fastify Request API](./fastify-request-api.tutorial.md)
   Focus on what Fastify adds to the incoming request object.
7. [Fastify Reply API](./fastify-reply-api.tutorial.md)
   Learn response control, serialization, headers, and sending patterns.
8. [Fastify Validation](./fastify-validation.tutorial.md)
   Understand schema validation and response serialization.
9. [Fastify Content-Type Parser](./fastify-content-type-parser.tutorial.md)
   Learn how bodies are parsed before validation and handlers.
10. [Fastify Error Handling](./fastify-error-handling.tutorial.md)
    Study error propagation, custom handlers, and failure paths.

### Phase 3: Advanced Topics

11. [Fastify Server Configuration](./fastify-server-configuration.tutorial.md)
    Review important boot-time and runtime server options.
12. [Fastify Streams](./fastify-streams.tutorial.md)
    Understand streaming request/response patterns.
13. [Fastify Routing & Constraints](./fastify-routing-constraints.tutorial.md)
    Learn advanced matching and routing behavior.
14. [Fastify + DynamoDB](./fastify-dynamodb.tutorial.md)
    Finish with an integration-oriented example.

## Fast Paths By Goal

### I want the minimum foundation

Read these first:

1. [Fastify Lifecycle](./fastify-lifecycle.tutorial.md)
2. [Fastify Hooks](./fastify-hooks.tutorial.md)
3. [Fastify Encapsulation](./fastify-encapsulation.tutorial.md)
4. [Fastify Plugins](./fastify-plugins.tutorial.md)

### I want to understand handler behavior

Read these first:

1. [Fastify Lifecycle](./fastify-lifecycle.tutorial.md)
2. [Fastify Request API](./fastify-request-api.tutorial.md)
3. [Fastify Reply API](./fastify-reply-api.tutorial.md)
4. [Fastify Error Handling](./fastify-error-handling.tutorial.md)

### I want to understand schemas and body parsing

Read these first:

1. [Fastify Validation](./fastify-validation.tutorial.md)
2. [Fastify Content-Type Parser](./fastify-content-type-parser.tutorial.md)
3. [Fastify Error Handling](./fastify-error-handling.tutorial.md)

### I want to write reusable plugins

Read these first:

1. [Fastify Encapsulation](./fastify-encapsulation.tutorial.md)
2. [Fastify Plugins](./fastify-plugins.tutorial.md)
3. [Fastify Decorators](./fastify-decorators.tutorial.md)
4. [Fastify Hooks](./fastify-hooks.tutorial.md)

## Tutorial Map

| Tutorial | Best read when you want to learn | Depends on |
|---|---|---|
| [Lifecycle](./fastify-lifecycle.tutorial.md) | boot and request execution order | none |
| [Hooks](./fastify-hooks.tutorial.md) | hook timing and cross-cutting behavior | lifecycle |
| [Encapsulation](./fastify-encapsulation.tutorial.md) | plugin scope boundaries | lifecycle |
| [Plugins](./fastify-plugins.tutorial.md) | app composition patterns | encapsulation |
| [Decorators](./fastify-decorators.tutorial.md) | shared APIs on app/request/reply | plugins, encapsulation |
| [Request API](./fastify-request-api.tutorial.md) | request object behavior | lifecycle |
| [Reply API](./fastify-reply-api.tutorial.md) | response control and sending | lifecycle |
| [Validation](./fastify-validation.tutorial.md) | schemas, AJV, serialization | request/reply basics |
| [Content-Type Parser](./fastify-content-type-parser.tutorial.md) | body parsing internals | lifecycle |
| [Error Handling](./fastify-error-handling.tutorial.md) | operational failure paths | hooks, reply |
| [Server Configuration](./fastify-server-configuration.tutorial.md) | server options and behavior | core concepts |
| [Streams](./fastify-streams.tutorial.md) | stream-based request and response handling | request/reply basics |
| [Routing & Constraints](./fastify-routing-constraints.tutorial.md) | advanced route matching | lifecycle |
| [DynamoDB](./fastify-dynamodb.tutorial.md) | real integration patterns | plugins, decorators, validation |

## Practical Advice

- Read one tutorial at a time and run the code examples locally.
- Keep [lib](../lib/) and [test](../test/) open while reading; these tutorials often
  explain behavior by referencing implementation details.
- If a tutorial feels too low-level, move back to the previous one in the reading order.
- Use the tutorial links between files to jump sideways once the core model is clear.
