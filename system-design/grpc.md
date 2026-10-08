---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-03
---

# gRPC

## What it is

gRPC is a high-performance RPC framework developed by Google. RPC (Remote Procedure Call) is the idea of calling a function on a remote machine as if it were a local function — the network is invisible to the caller. gRPC implements this over HTTP/2 using Protocol Buffers (protobuf) as the serialization format.

## What problem it solves

REST forces you to model everything as resources and HTTP verbs. gRPC lets you model operations as function calls — natural for action-oriented operations and internal service-to-service communication. It also solves the contract problem: both client and server are generated from the same `.proto` file, so mismatches are caught at compile time rather than in production.

## Notes

### RPC — the core concept

RPC = Remote Procedure Call. Call a function on another machine, get a result back. From the caller's perspective it looks like a local call:

```python
# looks local, runs on a remote server
response = orders_client.create_order(user_id=123, items=["shirt"])
```

Under the hood: arguments are serialized → sent over the network → server deserializes and runs the function → result is serialized and returned. The serialization and transport are invisible to the caller.

### How gRPC works

**Step 1 — write a `.proto` file (the contract):**

```proto
syntax = "proto3";

service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (CreateOrderResponse);
}

message CreateOrderRequest {
  int32 user_id = 1;
  repeated string items = 2;
}

message CreateOrderResponse {
  string order_id = 1;
}
```

**Step 2 — generate client and server code:**
```bash
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. orders.proto
```

**Step 3 — implement the server (only business logic, boilerplate is generated):**
```python
class OrderServicer(orders_pb2_grpc.OrderServiceServicer):
    def CreateOrder(self, request, context):
        order_id = db.create(request.user_id, request.items)
        return orders_pb2.CreateOrderResponse(order_id=order_id)
```

**Step 4 — call from the client:**
```python
stub = orders_pb2_grpc.OrderServiceStub(channel)
response = stub.CreateOrder(CreateOrderRequest(user_id=123, items=["shirt"]))
```

### The most common use case: microservices internal communication

A browser or external client talks REST. Everything behind the API gateway talks gRPC:

```
Browser
  │ REST
  ▼
API Gateway
  ├─ gRPC ──► Order Service
  │                ├─ gRPC ──► Inventory Service
  │                ├─ gRPC ──► Payment Service
  │                └─ gRPC ──► Notification Service
  └─ gRPC ──► User Service
```

Internal services call each other thousands of times per second — that's where performance and type safety pay off. Companies like Netflix, Uber, and Google use this pattern.

### REST vs gRPC

| | REST | gRPC |
|---|---|---|
| Mental model | Resources + HTTP verbs | Functions + arguments |
| Format | JSON (text, human-readable) | Protobuf (binary, ~3x smaller) |
| Browser support | Native | Needs gRPC-web library |
| Streaming | SSE / WebSocket (workaround) | Native (server, client, bidirectional) |
| Contract | Docs (Swagger/OpenAPI) — not enforced | `.proto` file — enforced at compile time |
| Coupling | Loose — anyone can call your URL | Tighter — need generated stubs |
| Best for | Public APIs, browser clients, CRUD | Internal service-to-service, high throughput, action-heavy |

### When to use which

- **Browser is involved** → REST. Browsers don't support raw gRPC.
- **You control both sides** → gRPC is worth considering.
- **Public API** → REST (any client can consume it without setup).
- **Internal microservices** → gRPC is the standard choice.
- **Mobile app calling your backend** → gRPC possible (you control the app; Google Maps uses it).

The typical pattern at scale: REST on the outside (public-facing), gRPC on the inside (between services).

### Workflow difference vs REST

| Step | REST | gRPC |
|---|---|---|
| Design | Define URL + HTTP verb | Write function signature in `.proto` |
| Serialization | Parse JSON manually | Generated — you never touch it |
| Documentation | Write Swagger/OpenAPI | The `.proto` file IS the docs |
| Client code | Write HTTP calls manually | Generated from `.proto` |
| Contract enforcement | Runtime (prod failures) | Compile time (breaks the build) |

### Debugging

```bash
# REST
curl -X POST http://orders-service/orders -d '{"user_id": 123}'

# gRPC
grpcurl -d '{"user_id": 123}' orders-service:50051 OrderService/CreateOrder
```

### Why ChatGPT streaming is not gRPC

The OpenAI API uses SSE (Server-Sent Events) — HTTP/REST with a long-lived connection that pushes tokens as they are generated. Not gRPC, because the client is a browser. gRPC may be used internally between OpenAI's own services, but the public-facing API is REST + SSE.

## Relations

- [[rest-api]] — REST is the alternative for public and browser-facing APIs; most systems use REST externally and gRPC internally
- [[ipc-transports]] — gRPC runs over TCP (HTTP/2); the RPC-over-stdin/stdout pattern used by MCP servers is the same concept applied to local pipes instead of network sockets

## Tags

#system-design
