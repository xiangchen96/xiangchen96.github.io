---
title: "📝 Domain Driven Design notes"
date: 2022-08-27
draft: false

tags: ["Software Engineering", "Programming", "Python"]
---

Some notes about DDD

<!--more-->

## References

- [Awesome Domain-Driven Design](https://github.com/heynickc/awesome-ddd)
- [Cosmic Python](https://www.cosmicpython.com/)
- [Domain-Driven Design](https://www.goodreads.com/book/show/179133.Domain_Driven_Design)
- [Implementing Domain-Driven Design](https://www.goodreads.com/book/show/15756865-implementing-domain-driven-design)

## Domain

- Use `@dataclass(frozen=True)` for [value objects](https://en.wikipedia.org/wiki/Value_object)
- For entities (objects with long lived identity, aka Reference Objects) use a normal class and implement `__eq__` and `__hash__`
- Don't use `FooBuilder`, `BarManager` classes, just functions

## Repository

- Use `Protocol` instead of `abc`.
- Leave `.commit()` as a responsibility for the caller.
- For simple cases this abstraction is unnecessary, you can just stick with ORMs.
- Create an in-memory `FakeRepository` for testing.

{{< notice info>}}
Some people consider "monkeypatching" a code smell and prefer using Fakes.
{{< /notice >}}

## Service

- Pass the repository as a parameter to service-layer functions and use primitives instead of Domain objects.
- Leave endpoints as a thin wrapper for parsing and returning HTTP responses, but you can combine controller/views if it is just a web app.
- Write tests for the service layer.

## Unit of Work

- Use context manager and initialize the repository on `__enter__`.
- Explicit `commit` is recommended for easier state flushing.
- Add `uow` to the service layer.

## Aggregates and Consistency Boundaries

> An AGGREGATE is a cluster of associated objects that we treat as a unit for the purpose of data changes.

- Configure [version counters](https://docs.sqlalchemy.org/en/14/orm/versioning.html) for Optimistic Concurrency Control.
- Retry on conflict.
- The only repository that should be used is the one for the aggregate.

Part 1: https://github.com/cosmicpython/code/tree/chapter_07_aggregate

## Events and Message Bus

- To avoid violating the [single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle), use events instead of commands.
- A message bus is a dict that maps event types to a list of handlers.
- Rethink API calls as capturing events.
- Services will have only 2 params: the event and the uow.

## Commands

> If at first you don’t succeed, retry the operation with an exponentially increasing back-off period.

- Commands are usually imperative verbs and expect a response.
- Events are usually past-tense verbs and broadcasted.

## Event-Driven Architecture

- Use an external message broker (redis, kafka, rabbitmq).
- Two services should accept eventual consistency between them.
- Avoid the Distributed Big Ball of Mud, think in terms of verbs / business processes with asynchronous communication.

https://martinfowler.com/articles/201701-event-driven.html

## Command-Query Responsibility Separation (CQRS)

- Create thin views for our data: via raw SQL?? (repository are clunky and ORMs slow).

## Dependency Injection (DI)

- Dependency Injection vs Monkeypatching.

Part 2: https://github.com/cosmicpython/code
