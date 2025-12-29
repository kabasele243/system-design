# Monolith vs Microservices

## Monolith
A single, unified codebase where all modules (Auth, Payments, Inventory) run in the same process.
*   **Pros**:
    *   **Simplicity**: Easy to develop, test, and deploy (one binary).
    *   **Performance**: Function calls are in-memory (fast).
    *   **Transactions**: Easy ACID transactions across domains.
*   **Cons**:
    *   **Coupling**: A bug in "Inventory" can crash "Payments".
    *   **Scaling Limit**: You have to scale the *entire* app, even if only one part is hot.
    *   **Tech Stack Lock-in**: Hard to use Python for ML and Go for API if it's all one Java app.

## Microservices
Breaking the application into small, independent services communicating via network (HTTP/gRPC).
*   **Pros**:
    *   **Independent Scaling**: Scale only the service that needs it.
    *   **Isolation**: If "Inventory" crashes, "Payments" stays up.
    *   **Organizational Scale**: Teams can work independently (Decoupled deployment cycles).
    *   **Tech Freedom**: Use the right tool for the job per service.
*   **Cons**:
    *   **Distributed Complexity**: Network failure, Latency, Serialization overhead.
    *   **Operational Cost**: Monitoring, Logging, Deployment pipelines for 50+ services is hard.
    *   **Data Consistency**: No distributed transactions (Sagas/Two-phase commit required).

## The "Citadel" Pattern (Hybrid)
**Modular Monolith**: Code is structured like microservices (clear boundaries) but deployed as a monolith. This is often the best starting point.
Split to Microservices **only when**:
1.  A specific module needs independent scaling.
2.  Team size grows too large (> 20 engineers).
