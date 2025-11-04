# DNS (Domain Name System)

## Learning Objectives
- Understand how DNS translates URLs to IP addresses
- Learn the DNS resolution process
- Understand DNS caching and TTL

## Notes

### What is DNS?

**DNS = Domain Name System = The Internet's Phone Book**

**The Problem:**
- Computers use IP addresses to communicate (e.g., `157.240.23.35`)
- Humans can't remember IP addresses
- We want to type `instagram.com`, not `157.240.23.35`

**The Solution:**
DNS translates human-readable domain names → machine-readable IP addresses

**Analogy:** Like a phone book that translates "John Smith" → "555-1234"

---

### DNS Resolution Process (Step-by-Step)

#### Step 1: Check Browser/Computer Cache
```
You type: instagram.com
Browser: "Did I visit this recently?"

IF CACHED:
  Browser: "Yes! instagram.com = 157.240.23.35"
  → Connect immediately ✅
  → DONE (< 1ms)

IF NOT CACHED:
  Browser: "Need to look this up"
  → Continue to Step 2
```

#### Step 2: Ask ISP's DNS Resolver
```
Computer → ISP DNS Resolver: "Where is instagram.com?"

Resolver checks ITS cache:

IF CACHED:
  Resolver: "instagram.com = 157.240.23.35"
  → Return to computer → Cache it
  → DONE (~5-10ms)

IF NOT CACHED:
  Resolver: "Need to query DNS hierarchy"
  → Continue to Step 3
```

#### Step 3: DNS Hierarchy Lookup (3 Levels)

**Level 1: Root DNS Server**
```
Resolver → Root DNS: "Where is instagram.com?"
Root: "I don't know exactly, but ask the .COM TLD server"
Root: "Here's the .COM server address: [IP]"
```

**Level 2: TLD Server (Top-Level Domain .com)**
```
Resolver → .COM TLD: "Where is instagram.com?"
TLD: "Ask Instagram's authoritative DNS server"
TLD: "Here's Instagram's DNS address: [IP]"
```

**Level 3: Authoritative DNS (Instagram's Server)**
```
Resolver → Instagram DNS: "Where is instagram.com?"
Instagram DNS: "instagram.com = 157.240.23.35"
→ This is the SOURCE OF TRUTH
```

#### Step 4: Return Answer & Cache
```
Instagram DNS → Resolver: "157.240.23.35"
Resolver: Cache for 1 hour (TTL)
Resolver → Your Computer: "157.240.23.35"
Computer: Cache for 1 hour
Browser: Connect to 157.240.23.35 ✅
```

**Total time (uncached):** 20-100ms
**Total time (cached):** < 1ms

---

### Caching & TTL (Time To Live)

**Why Caching Matters:**
- Makes lookups instant (< 1ms vs 50ms)
- Reduces load on DNS servers (80-90% served from cache)
- Without caching: DNS servers would crash from billions of requests

**Multi-Level Caching:**
1. Browser cache (your browser)
2. OS cache (your computer)
3. ISP DNS Resolver cache
4. Each level reduces the need to query upstream

**TTL (Time To Live):**
- How long to cache the DNS record
- Typical values: 5 minutes to 24 hours
- After TTL expires → Must look up again
- Example: TTL = 1 hour means cache expires after 60 minutes

**Trade-off:**
- **High TTL (24 hours):** Fast, but takes long to update if IP changes
- **Low TTL (5 minutes):** Flexible, but more DNS queries

---

### DNS Record Types

| Record Type | Purpose | Example |
|-------------|---------|---------|
| **A Record** | Domain → IPv4 address | `instagram.com` → `157.240.23.35` |
| **AAAA Record** | Domain → IPv6 address | `instagram.com` → `2a03:2880:...` |
| **CNAME** | Alias (points to another domain) | `www.instagram.com` → `instagram.com` |
| **MX Record** | Mail server | Email to `@instagram.com` → `mail.instagram.com` |
| **NS Record** | Name server (who manages DNS) | `instagram.com` → `ns1.instagram.com` |

**Most important for system design:** A Record (domain → IP)

---

### Domain Registrar vs DNS Provider

**Domain Registrar (Where you BUY the domain):**
- Examples: GoDaddy, Namecheap, Route 53
- You register ownership of "myapp.com"
- **You only buy from ONE registrar**

**DNS Provider (Who ANSWERS lookups):**
- Examples: Route 53, Cloudflare, Google DNS
- Hosts your DNS records (the "phone book")
- **You CAN use MULTIPLE DNS providers for redundancy**

**How to use multiple DNS providers:**
1. Set up DNS records on both providers (keep in sync)
2. Configure nameservers to point to BOTH providers
3. If one fails, the other answers

**Example:**
```
Domain: instagram.com
Nameservers:
- ns1.route53.com (Route 53)
- ns2.route53.com (Route 53)
- ns1.cloudflare.com (Cloudflare)
- ns2.cloudflare.com (Cloudflare)
```

If Route 53 fails, Cloudflare still answers → High availability

## Practice Questions

1. **How does typing "google.com" get translated to an IP address?**
   - Check browser/computer cache (< 1ms if cached)
   - Ask ISP DNS resolver (5-10ms if resolver cached)
   - If not cached: Resolver queries Root → TLD (.com) → Authoritative (Google's DNS)
   - Authoritative returns IP address
   - Cache at multiple levels for future lookups

2. **What is DNS caching and why is it important?**
   - Caching stores DNS lookups temporarily at multiple levels
   - **Why critical:** Without caching, billions of DNS queries would crash servers
   - Makes lookups instant (< 1ms vs 50-100ms)
   - 80-90% of DNS requests served from cache
   - Reduces load on DNS infrastructure

3. **What happens if a DNS server fails?**
   - **Users with cached DNS:** Can still access the site (until cache expires)
   - **Users without cache:** Get "Server not found" error
   - **Even if website servers are running fine!**
   - **Real example:** 2021 Facebook outage - DNS records deleted, site unreachable for 6+ hours
   - **Solution:** Multiple DNS providers and nameservers for redundancy

4. **If you visit instagram.com in the morning and again 2 hours later, will your computer ask the DNS resolver again?**
   - Depends on TTL (Time To Live)
   - If TTL < 2 hours (e.g., 1 hour): Cache expired → Ask resolver again
   - If TTL ≥ 2 hours (e.g., 24 hours): Still cached → Use cached IP instantly

## Real-World Examples

### DNS Failures
**2021 Facebook/Instagram/WhatsApp Outage:**
- Duration: 6+ hours
- Cause: Facebook accidentally deleted their DNS records
- Impact: Billions of users couldn't access any Facebook services
- **The websites were running fine** - but DNS made them unreachable
- Cost: ~$100 million in lost revenue

**2016 Dyn DNS Attack:**
- Major DNS provider Dyn hit with massive DDoS attack
- Twitter, Netflix, Reddit, GitHub all became unreachable
- Services themselves were operational
- DNS failure = entire internet appeared broken

### Multi-Provider DNS (Enterprise)
**Netflix:**
- Uses multiple DNS providers: UltraDNS, Route 53, Dyn
- If one DNS provider fails, others keep Netflix accessible
- Cost: Extra complexity but ensures 99.99%+ availability

**Instagram:**
- Multiple authoritative nameservers (ns1, ns2, ns3, ns4)
- Geographic distribution (servers worldwide)
- If one nameserver fails, others answer

## Resources & References
- **Key insight:** DNS = Internet's phone book (human names → computer addresses)
- **Remember:** 3-level hierarchy (Root → TLD → Authoritative)
- **Critical:** Caching at multiple levels prevents DNS overload
- **Pro tip:** DNS failure makes entire service appear offline, even if servers are fine
