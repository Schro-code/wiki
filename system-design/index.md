---
domain: system-design
type: index
updated: 2026-09-15

---

# System Design — Domain Index

Fundamentals of system design: networking, protocols, communication mechanisms, and infrastructure patterns.

## Pages

| Page | When to look | Sources |
|---|---|---|
| [[file-descriptors]] | Look when you need to understand what an fd is, why connections consume them, or how to tune server fd limits | — |
| [[ipc-transports]] | Look when comparing OS-level communication mechanisms (pipe, socket, shared memory) or understanding how RPC-over-pipe works | — |
| [[ip-addresses]] | Look when you need to understand IP addressing, DHCP, network interfaces, or why private/public IPs matter | — |
| [[dns]] | Look when you need to understand how domain names resolve to IPs, DNS server types, record types, or service discovery | — |
| [[acid-and-base]] | Look when choosing between consistency models, designing for scale, or understanding database trade-offs | — |
| [[cap-and-pacelc]] | Look when choosing a distributed database, understanding partition trade-offs, or confused about CAP vs ACID consistency | — |
| [[distributed-transactions]] | Look when you need operations to be atomic across multiple databases or services, or want to understand 2PC, 3PC, and Sagas | — |
| [[sagas]] | Look when you need distributed atomicity across microservices or third-party APIs without locking, or want to understand orchestration vs choreography and how Azure Durable Functions implements this | — |
| [[rest-api]] | Look when designing or consuming HTTP APIs, understanding REST constraints (stateless, cacheable), HTTP verbs, status codes, or how statelessness enables scaling | — |
| [[grpc]] | Look when designing internal service-to-service communication, comparing gRPC vs REST, understanding protobuf and code generation, or learning when to use each | — |
| [[http-grpc-websocket]] | Look when comparing HTTP, gRPC and WebSocket, understanding polling patterns (short, long polling), why clients must initiate connections (NAT), or choosing a protocol for real-time communication | — |
| [[rate-limiter]] | Look when designing traffic control, choosing a throttling algorithm, or understanding how Redis and cached rules fit into a rate limiting architecture | https://github.com/liquidslr/system-design-notes/blob/main/04.%20Rate%20Limiter/Readme.md |
| [[consistent-hashing]] | Look when distributing keys across servers, adding/removing nodes without global reshuffling, or understanding virtual nodes and the hash ring | https://github.com/liquidslr/system-design-notes/blob/main/05.%20Consistent%20Hashing/Readme.md |
