# Proxies

## Learning Objectives
- Understand Forward Proxy vs Reverse Proxy
- Learn when to use each type
- Understand API Gateway as a reverse proxy

## Notes

### Forward Proxy (Client-Side Proxy)

**Definition:** A proxy server that sits between clients and the internet, making requests on behalf of clients.

**Analogy:** Your assistant who makes requests on your behalf

**How It Works:**
```
[Client] → [Forward Proxy] → [Internet/Server]
            ↑
       "Pretends to be the client"
```

**Key Point:** The client KNOWS they're using a proxy (user configures it)

**Use Cases:**

1. **Bypass Restrictions/Censorship:**
```
User (in blocked country) → Forward Proxy (VPN in USA) → blocked-website.com
Website sees: Request from USA, not from blocked country
```

2. **Corporate Network Control:**
```
Employee → Corporate Forward Proxy → Internet
Proxy: "Block social media, allow work sites only"
```

3. **Privacy/Anonymity:**
```
User → Forward Proxy → Website
Website can't see user's real IP address
```

4. **Caching:**
```
Corporate proxy caches common websites
Multiple employees request same site → Served from cache (faster)
```

**Examples:**
- VPN (NordVPN, ExpressVPN)
- Squid Proxy
- Corporate web proxy

**What Forward Proxy Protects:** Client identity and privacy

---

### Reverse Proxy (Server-Side Proxy)

**Definition:** A proxy server that sits in front of web servers, intercepting requests from clients.

**Analogy:** A receptionist who directs visitors to the right office

**How It Works:**
```
[Client] → [Reverse Proxy] → [Internal Servers 1, 2, 3...]
                              ↑
                         "Hidden from client"
```

**Key Point:** The client DOESN'T KNOW there's a proxy (transparent)

**Use Cases:**

1. **Load Balancing (What We Just Learned!):**
```
Users → Reverse Proxy → Distributes to Server 1, 2, 3, 4
```

2. **SSL Termination:**
```
User (HTTPS) → Reverse Proxy → Decrypt HTTPS → HTTP → Servers
                    ↑
           "Handles encryption so servers don't have to"
```
Benefit: Servers don't need SSL certificates, less CPU load

3. **Caching:**
```
User → Reverse Proxy: "Give me /home"
Reverse Proxy: "I have /home cached!" → Return immediately
No need to hit backend servers
```

4. **Security/Firewall:**
```
User → Reverse Proxy → Check for attacks/DDoS → Forward to Server
         ↑
   "Hides internal server IPs and infrastructure"
```

5. **Compression:**
```
Server → Large response → Reverse Proxy → Compress → Smaller response → User
```

**Examples:**
- NGINX
- HAProxy
- AWS Application Load Balancer
- Cloudflare

**What Reverse Proxy Protects:** Server infrastructure and improves performance

---

### Forward vs Reverse Proxy Comparison

| Aspect | Forward Proxy | Reverse Proxy |
|--------|--------------|---------------|
| **Location** | Client side (near user) | Server side (near servers) |
| **Protects** | Client identity | Server infrastructure |
| **User awareness** | User knows and configures it | Transparent to user |
| **Purpose** | Anonymity, bypass restrictions | Load balancing, security, performance |
| **Example** | VPN, corporate proxy | Load balancer, CDN, NGINX |
| **Who configures?** | User/Admin | Server admin |

**Memory Trick:**
- **Forward Proxy:** Forwards requests FOR the client (hides client)
- **Reverse Proxy:** Reverses the flow TO servers (hides servers)

---

### API Gateway (Advanced Reverse Proxy)

**Definition:** A reverse proxy with additional application-level features

**API Gateway = Reverse Proxy + Extra Superpowers**

```
[Mobile App] ─┐
[Web App]     ├─→ [API Gateway] → Routes to:
[Partner API] ─┘                  → Auth Service
                                  → User Service
                                  → Payment Service
                                  → Order Service
```

**Features Beyond Basic Reverse Proxy:**

1. **Routing/Aggregation:**
   - Route `/users/*` → User Service
   - Route `/orders/*` → Order Service
   - Aggregate multiple service calls into one response

2. **Authentication/Authorization:**
   - Check API keys
   - Validate JWT tokens
   - OAuth 2.0 flows

3. **Rate Limiting:**
   - Max 100 requests/minute per user
   - Prevent API abuse

4. **Request/Response Transformation:**
   - Convert XML to JSON
   - Add/remove headers
   - Modify request body

5. **API Versioning:**
   - Route `/v1/*` → Old API
   - Route `/v2/*` → New API

6. **Monitoring/Analytics:**
   - Track API usage
   - Log requests
   - Performance metrics

**Examples:**
- Kong
- AWS API Gateway
- Azure API Management
- Apigee

**When to Use API Gateway:**
- Microservices architecture (many backend services)
- Need centralized authentication
- Want to expose internal APIs to partners/public
- Need rate limiting and analytics

**Instagram Example:**
```
[Mobile App] → API Gateway → Routes:
                             /feed → Feed Service
                             /profile → User Service
                             /upload → Media Service
                             /messages → Chat Service
```
All authentication, rate limiting handled at gateway level

## Practice Questions
1. What's the difference between a forward proxy and a reverse proxy?
2. When would you use a forward proxy?
3. Why do companies use reverse proxies?

## Real-World Examples


## Resources & References
