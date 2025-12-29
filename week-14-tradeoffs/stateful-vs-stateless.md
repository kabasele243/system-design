# Stateful vs Stateless Architecture

## Definition

### Stateless Architecture
*   **Concept**: The server does NOT store any user state (session data) between requests. Every request must contain all necessary info (e.g., JWT Token) to be processed.
*   **Scaling**: Easy. Any server can handle any request. You can spin up 100 new servers and add them to the Load Balancer immediately.
*   **Failure Recovery**: Easy. If a server dies, requests just go to another one. No data is lost.
*   **Complexity**: Lower on the server-side, slightly higher on client (must store tokens).

### Stateful Architecture
*   **Concept**: The server stores client session data in memory or local disk. Subsequent requests *must* go to the *same* server to access that state.
*   **Scaling**: Hard. Requires "Sticky Sessions" (Session Affinity) at the Load Balancer level. Adding new servers doesn't immediately help existing active sessions.
*   **Failure Recovery**: Hard. If a server dies, all active sessions on that machine are lost (users logged out, shopping carts emptied), unless state is replicated (complex).
*   **Use Case**: Real-time gaming (low latency state), WebSocket servers, large legacy apps.

## The Modern Standard
**Stateless Application + Stateful Store**
*   Keep the App Servers **Stateless**.
*   Store the State in a external **Stateful Store** (Redis/Memcached/Database).
*   **Benefit**: You get the scaling benefits of Stateless (App layer) with the functionality of Stateful.

## Trade-off Summary
| Feature | Stateless | Stateful |
| :--- | :--- | :--- |
| **Scalability** | Horizontal (Easy) | Limited (Stickiness required) |
| **Resilience** | High (Failover is instant) | Low (Session loss on crash) |
| **Complexity** | Low | High |
| **Connection** | HTTP (Short lived) | TCP/WebSocket (Long lived) |
