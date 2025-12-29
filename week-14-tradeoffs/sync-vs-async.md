# Sync vs Async Processing

## Overview
How tightly coupled are your services?

## 1. Synchronous (Blocking)
Caller waits for the "Done" signal.
*   **Flow**: User -> Service A -> Service B -> DB.
*   **Pros**:
    *   **Simplicity**: Easy to reason about (Linear code).
    *   **Consistency**: You know immediately if it failed.
*   **Cons**:
    *   **Cascading Failure**: If Service B is slow, Service A becomes slow.
    *   **Latency**: Response time = Sum of all downstream calls.
    *   **Loss of Availability**: If Service B is down, the whole request fails.

## 2. Asynchronous (Non-Blocking)
Caller fires an event and returns immediately. Processing happens in background.
*   **Flow**: User -> Service A -> Queue -> Service B.
*   **Pros**:
    *   **Resilience**: If Service B is down, message sits in Queue. Service A is still up.
    *   **Latency**: Response time is fast (only enqueue time).
    *   **Throttling**: Service B can process at its own pace (Load Leveling).
*   **Cons**:
    *   **Complexity**: Handling deadlock, retries, eventual consistency.
    *   **User Experience**: User doesn't know if it actually finished. Need polling or websockets for updates.

## Decision Guide
*   **User waiting on screen?** -> Sync (or Async with websocket push).
*   **Long running job (Email, PDF Gen)?** -> Async everywhere.
*   **Inter-service communication?** -> Prefer Async for decoupling unless data is needed immediately.
