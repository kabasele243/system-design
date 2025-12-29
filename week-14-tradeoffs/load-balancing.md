# Load Balancing

## Overview
Distributing traffic across healthy servers.

## Types

### 1. Layer 4 (Transport Layer - TCP/UDP)
Routes based on IP and Port.
*   **Pros**:
    *   **Extremely Fast**: Doesn't look at packet content. Hardware supported.
    *   **Secure**: Doesn't decrypt SSL (Passthrough).
*   **Cons**:
    *   **Dumb**: Can't route "User A to Server 1". Can't see URL path.
*   **Examples**: AWS Network Load Balancer (NLB), IPVS.

### 2. Layer 7 (Application Layer - HTTP)
Routes based on Content (URL, Headers, Cookies).
*   **Pros**:
    *   **Smart**: Can route `/api/video` to Video Servers and `/api/user` to User Servers.
    *   **Features**: Sticky Sessions, SSL Termination, Compression, Rate Limiting.
*   **Cons**:
    *   **Slower**: CPU intensive (must inspect packets).
*   **Examples**: NGINX, HAProxy, AWS Application Load Balancer (ALB).

## Algorithms
1.  **Round Robin**: Cycle through list. (Simple).
2.  **Least Connections**: Send to server with fewest open connections. (Adapts to load).
3.  **Weighted**: Server A gets 2x traffic of Server B. (For mixed hardware).
4.  **IP Hash**: Client IP determines Server. (Provides session stickiness without cookies).
