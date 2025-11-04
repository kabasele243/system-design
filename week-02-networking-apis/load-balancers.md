# Load Balancers

## Learning Objectives
- Understand what load balancers do
- Learn L4 vs L7 load balancing
- Understand load balancing algorithms

## Notes

### What is a Load Balancer?

**The Problem:**
- DNS returns one IP address (e.g., instagram.com → 157.240.23.35)
- 500 million users try to connect to that IP
- One server can't handle 500 million users!

**The Solution:**
Load balancer = Traffic director that distributes requests across multiple servers

**Analogy:** Coffee shop dispatcher telling customers which location to visit

```
[500M Users] → DNS → 157.240.23.35 (Load Balancer)
                           ↓
              Load Balancer distributes to:
                → Server 1 (10.0.1.1)
                → Server 2 (10.0.1.2)
                → Server 3 (10.0.1.3)
                → Server 4 (10.0.1.4)
                → ... more servers
```

**Load Balancer's Job:**
1. Receives all incoming requests
2. Picks a healthy server (health check)
3. Forwards request to that server
4. Returns response back to user

**User never knows which server they're talking to!**

---

### Load Balancing Algorithms

#### 1. Round Robin (Simplest)
**How it works:** Rotate through servers in order

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1 (start over)
```

**Pros:**
- Simple to implement
- Evenly distributes load

**Cons:**
- Doesn't consider server load
- Doesn't consider request complexity

**Best for:** Stateless servers with similar capacity

---

#### 2. Least Connections
**How it works:** Send to server with fewest active connections

```
Server 1: 100 connections
Server 2: 50 connections   ← Send here!
Server 3: 150 connections
```

**Pros:**
- Adapts to server load dynamically
- Better for long-lived connections

**Cons:**
- More complex tracking

**Best for:** Long file downloads, video streaming

---

#### 3. IP Hash
**How it works:** Hash user's IP address, always route to same server

```
Hash(192.168.1.100) % 4 = Server 2
→ This user always goes to Server 2
```

**Pros:**
- Session persistence (same user → same server)
- Good for caching per server

**Cons:**
- Uneven distribution (geographic clustering)
- Mobile users change IPs frequently
- Server failure requires rehashing

**Best for:** Legacy stateful applications

---

#### 4. Cookie-Based (Session Stickiness)
**How it works:** Load balancer sets cookie, routes based on cookie

```
First visit: Assign Server 3, set cookie: server=3
Next visit: Read cookie → Route to Server 3
```

**Better than IP Hash because:**
- Works across IP changes (mobile users)
- More control over distribution
- Can expire and reassign

---

### Modern Best Practice: Stateless Servers

**Instead of sticky sessions, make servers stateless!**

```
User session data stored in:
- Redis/Memcached (shared cache)
- Database

Any server can handle any request because:
→ Server pulls session from shared cache
→ No need to route to "same" server
```

**Instagram/Facebook approach:**
- Servers are stateless
- Session in Redis
- Load balancer uses Round Robin or Least Connections

---

### L4 (Transport Layer) Load Balancing

**What Layer 4 sees:** IP addresses and ports (TCP/UDP level)

```
Load balancer sees:
- Source IP: 192.168.1.1
- Destination IP: 157.240.23.35
- Port: 443 (HTTPS)

Load balancer: "Forward to Server 2"
```

**Cannot see:**
- HTTP URLs
- Headers
- Cookies
- Application data

**Characteristics:**
- **Very fast** (minimal processing)
- **Protocol agnostic** (works with HTTP, FTP, SSH, any TCP/UDP)
- **Simple routing** (based on IP/Port only)

**Example:** AWS Network Load Balancer (NLB)

---

### L7 (Application Layer) Load Balancing

**What Layer 7 sees:** Full HTTP request content

```
Load balancer sees:
- URL: /api/posts/12345
- Method: GET
- Headers: User-Agent: iPhone
- Cookies: session_id=abc123
```

**Can route based on:**
- URL path (`/api/*` → API servers)
- HTTP method (POST → Write servers, GET → Read servers)
- Headers (Mobile → Lightweight servers)
- Cookies (Premium users → Premium servers)

**Characteristics:**
- **Slower** (more processing)
- **HTTP/HTTPS only**
- **Smart routing** (content-based decisions)
- **Can do SSL termination** (decrypt HTTPS)

**Example:** AWS Application Load Balancer (ALB), NGINX

---

### L4 vs L7 Comparison

| Aspect | L4 (Transport) | L7 (Application) |
|--------|---------------|------------------|
| **Speed** | Very fast | Slower |
| **What it sees** | IP, Port | Full HTTP (URL, headers, cookies) |
| **Routing** | IP/Port based | Content-based (URL, method, headers) |
| **Protocols** | Any (HTTP, FTP, SSH, etc.) | HTTP/HTTPS only |
| **Use case** | Simple distribution | Smart routing |
| **Example** | Network Load Balancer | Application Load Balancer |

**Instagram example:**
```
L7 Load Balancer routes by URL:
- /api/* → API Servers
- /images/* → Image CDN
- /feed → Feed Service
- /stories → Stories Service
```

---

### Health Checks (Critical!)

**Problem:** What if Server 3 crashes? Load balancer keeps sending → All fail!

**Solution:** Regular health checks

```
Every 10 seconds:
Load Balancer → Server 1: GET /health
Server 1 → 200 OK ✅ (Healthy)

Load Balancer → Server 2: GET /health
Server 2 → 200 OK ✅ (Healthy)

Load Balancer → Server 3: GET /health
Server 3 → (timeout) ❌ (Dead)

Load Balancer: "Stop sending to Server 3!"
```

**Only route traffic to healthy servers**

---

### High Availability (Multiple Load Balancers)

**Problem:** Load balancer is a single point of failure!

**Solution:** Multiple load balancers

#### Active-Active (Both Running)
```
DNS returns two IPs:
- 157.240.23.35 (Load Balancer 1) → 50% traffic
- 157.240.23.36 (Load Balancer 2) → 50% traffic

If LB1 fails → LB2 handles 100%
```

**Pros:** Efficient resource use
**Cons:** Each must handle full load if one fails

#### Active-Passive (Primary + Backup)
```
Load Balancer 1 (Active): 100% traffic
Load Balancer 2 (Passive): Standby

If LB1 fails → LB2 takes over (failover)
```

**Pros:** LB2 ready for full load
**Cons:** LB2 sits idle (wasted capacity)

## Practice Questions

1. **When would you use L4 vs L7 load balancing?**
   - **L4:** When you need maximum speed and work with any protocol (FTP, SSH, HTTP)
   - **L7:** When you need smart routing based on URL, headers, cookies (HTTP only)
   - Example: L7 to route `/api/*` to API servers, `/images/*` to CDN

2. **What happens when a server behind a load balancer fails?**
   - Health checks detect failure (GET /health returns error or timeout)
   - Load balancer marks server as unhealthy
   - Stops sending new requests to that server
   - Only routes to healthy servers
   - When server recovers, health check passes → Resume routing

3. **How does a load balancer improve availability?**
   - Distributes load (no single server overwhelmed)
   - Health checks route around failures
   - If Server 1 fails → Servers 2, 3, 4 keep service running
   - Multiple load balancers prevent single point of failure
   - System stays up even when individual components fail

4. **For a mobile vs desktop routing use case, can L4 load balancer work?**
   - **No!** L4 can only see IP/Port, not User-Agent header
   - Need L7 to inspect HTTP headers and route based on device type

5. **Why is Round Robin better than IP Hash for Instagram?**
   - Mobile users change IPs → IP Hash breaks session
   - Better: Stateless servers + Round Robin
   - Session data in Redis (any server can serve any user)
   - Even load distribution across servers

## Hands-On Practice
Implement a basic round-robin load balancer in your preferred language.

**Pseudocode:**
```python
servers = ['server1.com', 'server2.com', 'server3.com']
current_index = 0

def get_next_server():
    global current_index
    server = servers[current_index]
    current_index = (current_index + 1) % len(servers)
    return server

# Request 1 → server1.com
# Request 2 → server2.com
# Request 3 → server3.com
# Request 4 → server1.com (loops back)
```

## Real-World Examples

### Instagram's Load Balancing
```
[Users] → DNS → L7 Load Balancer
                     ↓
            Routes by URL path:
            /api/* → API Servers (1-100)
            /images/* → Image CDN
            /feed → Feed Service
            /stories → Stories Service
```
- L7 for smart content routing
- Stateless servers (session in Redis)
- Round Robin within each service tier
- Multiple load balancers (Active-Active)

### Netflix Load Balancing
- Multiple layers of load balancing
- Geographic routing (users to nearest region)
- L7 for path-based routing
- Zuul (their custom load balancer) handles 100B+ requests/day

## Resources & References
- **Key insight:** Load balancer = traffic director for horizontal scaling
- **L4 vs L7:** Speed (L4) vs Smart routing (L7)
- **Best practice:** Stateless servers + Round Robin/Least Connections
- **Remember:** Health checks + Multiple load balancers = High availability
