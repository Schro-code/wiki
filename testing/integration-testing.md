---
domain: testing
tags: [testing]
sources: []
updated: 2026-10-19
---

# Integration Testing

## What it is

Tests that verify two or more components work correctly together, without going through the full UI. They sit in the middle layer of the test pyramid: more realistic than unit tests (no mocks for the components under test), less expensive than e2e tests (no browser, no full deployment).

## What problem it solves

Unit tests mock dependencies, so they can't catch bugs at the boundaries between components. E2e tests catch boundary bugs but are slow and expensive. Integration tests fill the gap: they test real interactions between components at a fraction of the e2e cost.

## Notes

### What "integration" means in practice

The "two or more components" can be:

| Integration | What it tests | Example |
|---|---|---|
| **Service + Database** | Queries, migrations, transactions | "Does `UserRepository.find_by_email()` return the right user from a real PostgreSQL?" |
| **Service + Service** | API contracts, serialization | "Does the Order service correctly call the Payment service and handle its response?" |
| **Service + Message Queue** | Event publishing and consumption | "When an order is created, does the event arrive in the queue with the right schema?" |
| **API endpoint** | HTTP routing, validation, serialization | "Does `POST /api/users` with valid JSON return 201 and the created user?" |

### How they differ from unit and e2e

```
Unit test:        [Function] → mock everything else → assert output
Integration test: [Service A] → [Real Service B] → assert interaction works
E2E test:         [Browser] → [Full App] → [All Services] → [All DBs] → assert user sees result
```

### Common patterns

- **Test containers**: spin up a real database in Docker for the test, tear it down after. The test hits a real PostgreSQL, not a mock.
- **In-memory alternatives**: use SQLite instead of PostgreSQL for speed (trade-off: less realistic).
- **API testing**: send real HTTP requests to a running server, assert on status codes and response bodies. No browser needed.

### When to use

- Every time you have a database query that's non-trivial (joins, transactions, migrations)
- When two services communicate via API and you want to verify the contract
- When you're not sure if your ORM/query builder generates the right SQL
- When unit tests pass but the feature doesn't work (the bug is at the boundary)

## Quiz

**1.** All unit tests for `OrderService` pass — every function returns the right value with a mocked database. But in staging, creating an order fails with a SQL constraint violation. What went wrong?

- a) This is a deployment problem, not a testing problem
- b) The unit tests have a bug — they should have caught this
- c) Unit tests mocked the database, so they couldn't catch a real SQL constraint. An integration test with a real database would have caught it
- d) I don't know

**2.** You need to verify that when a user places an order, a message lands in a queue with the correct schema. An e2e test would need a browser, a running app, and a user clicking "Buy." Is there a cheaper way?

- a) Yes — an integration test that calls the order service directly and checks the queue, no browser needed
- b) No — if you want to test the full flow you need an e2e test
- c) Yes — a unit test that mocks the queue and checks the message format
- d) I don't know

**3.** Your team wants to use SQLite instead of real PostgreSQL for integration tests because it's faster. What's the trade-off?

- a) SQLite is always wrong for integration tests — you must use the real database
- b) Faster tests, but SQLite has different SQL behavior — some queries that work in SQLite will fail in real PostgreSQL (or vice versa), so you might miss production bugs
- c) No trade-off — SQLite and PostgreSQL behave the same for testing purposes
- d) I don't know

**4.** A test sends POST to `/api/users` on a running server and checks the 201 response and JSON body. No browser involved. A colleague calls it an e2e test. Is that right?

- a) No — it's an integration test. It verifies real components (HTTP routing + service + DB) but doesn't simulate a user interacting with the UI
- b) No — it's a unit test because it only checks one endpoint
- c) Yes — it hits a real server, so it's e2e
- d) I don't know

**5.** Your team only has unit tests and e2e tests — nothing in between. E2e tests are slow and flaky, unit tests can't catch boundary bugs. What's missing?

- a) Better unit test mocks that simulate real infrastructure more accurately
- b) More e2e tests to compensate for the flakiness
- c) Integration tests — they test real component boundaries (service + DB, service + API) at a fraction of the e2e cost and without the flakiness
- d) I don't know

## Relations

- [[test-pyramid]] — integration tests are the middle layer
- [[unit-testing]] — the layer below, where dependencies are mocked instead of real
- [[test-doubles]] — integration tests deliberately drop test doubles for the components under test
- [[end-to-end-testing]] — integration tests verify boundaries without the full UI overhead
- [[smoke-testing]] — some smoke checks are integration-level ("can the app reach the database?")

## Tags

#testing
