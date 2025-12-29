# Communication Patterns

## Overview
How services talk to each other impacts latency, coupling, and complexity.

## 1. Request-Response (Synchronous)
The client sends a request and waits for a response.

### REST (Representational State Transfer)
*   **Protocol**: HTTP/1.1 or HTTP/2.
*   **Format**: JSON (usually).
*   **Pros**: Simple, Universal, Human-readable, Cacheable.
*   **Cons**: Text-based (slower), No built-in types, Over-fetching/Under-fetching.

### gRPC (Google Remote Procedure Call)
*   **Protocol**: HTTP/2 (Protobuf).
*   **Format**: Binary.
*   **Pros**: High performance, Strongly typed, Code generation, Bidirectional streaming.
*   **Cons**: Binary (not human readable), Browser support requires proxy.

### GraphQL
*   **Protocol**: HTTP.
*   **Format**: JSON.
*   **Pros**: Client asks for exactly what it needs (No over-fetching), Single endpoint.
*   **Cons**: Complexity in caching, N+1 query problem on server.

## 2. Asynchronous (Event-Driven)
The client sends a message and doesn't wait.

### Message Queues (Point-to-Point)
*   **Examples**: RabbitMQ, SQS.
*   **Pattern**: Producer -> Queue -> Consumer.
*   **Pros**: Decoupling, Load leveling (Backpressure).

### Pub/Sub (Publish-Subscribe)
*   **Examples**: PC/Kafka, SNS.
*   **Pattern**: Publisher -> Topic -> Multiple Subscribers.
*   **Pros**: Decoupling, Broadcast capability.

## 3. Real-Time (Bidirectional)

| Pattern | Description | Best For |
| :--- | :--- | :--- |
| **Short Polling** | Client requests every N seconds. | Simple, non-critical updates. |
| **Long Polling** | Client requests, Server holds open until data available. | Better than short polling, widely supported. |
| **WebSockets** | Persistent, full-duplex TCP connection. | Chat apps, Games, Trading platforms. |
| **SSE (Server-Sent Events)** | Server pushes to client (One-way). | Tickers, Feed updates. |

## Trade-offs
*   **Latency vs Coupling**: Sync is faster for the user but couples services. Async is resilient but slower feedback.
*   **Complexity**: WebSockets are harder to scale (stateful connections) than stateless REST.
