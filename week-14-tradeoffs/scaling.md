# Vertical vs Horizontal Scaling

## Vertical Scaling (Scale Up)
Adding more power (CPU, RAM, Disk) to an **existing** server.
*   **Analogy**: Buying a bigger engine for your car.
*   **Pros**:
    *   **Simplicity**: No code changes required.
    *   **Consistency**: Data lives in one place (no distributed consistency issues).
    *   **Communication**: Fast (in-memory, no network calls).
*   **Cons**:
    *   **Hard Limit**: There is a max limit to hardware (e.g., 128 cores, 4TB RAM). You hit a ceiling.
    *   **Exponential Cost**: High-end hardware is disproportionately expensive.
    *   **Single Point of Failure**: If that one monster server dies, you are down.

## Horizontal Scaling (Scale Out)
Adding **more** servers to the pool.
*   **Analogy**: Buying more cars (a fleet).
*   **Pros**:
    *   **Infinite Scale**: Theoretically limitless (Google/Amazon style).
    *   **Cost Effective**: Uses cheap, commodity hardware.
    *   **Resilience**: If one node dies, others take over.
*   **Cons**:
    *   **Complexity**: Requires Load Balancers, Distributed Systems logic, Partitioning/Sharding.
    *   **Consistency**: Harder to maintain data consistency across nodes (CAP Theorem applies).
    *   **Network Overhead**: Services talk over the network (slower than in-memory).

## Decision Framework
1.  **Start with Vertical** (for DBs usually): It's easier and fine for MVP/startups.
2.  **Move to Horizontal** (Stateless Apps): Web servers are easy to scale out.
3.  **Sharding** (Horizontal DB): Only do this when Vertical Scaling hits the hardware ceiling or cost becomes prohibitive.
