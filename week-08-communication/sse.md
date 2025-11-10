# Server-Sent Events (SSE)

## Learning Objectives
- Understand Server-Sent Events protocol
- Learn when to use SSE vs WebSockets
- Master real-time notifications

## Notes

### What are Server-Sent Events (SSE)?

**The Use Case:**
- Live sports scoreboard
- Stock price updates
- Server logs streaming
- Social media feed updates

**Key characteristic:** Server pushes updates, client only receives (one-way)

### Why SSE Instead of WebSockets?

**WebSocket (bidirectional, more complex):**
```javascript
const ws = new WebSocket('wss://api.example.com/updates');
ws.onmessage = (event) => { /* handle update */ };
// But client never sends anything back! Overkill?
```

**SSE (one-way, simpler):**
```javascript
const eventSource = new EventSource('/api/updates');
eventSource.onmessage = (event) => { /* handle update */ };
```

**Benefits of SSE:**
- ✅ Simpler than WebSockets (just HTTP, no protocol upgrade)
- ✅ Browser native (all browsers)
- ✅ **Auto-reconnect built-in** (huge advantage!)
- ✅ **Event IDs for resuming** (never miss events!)
- ✅ Text-based (easier debugging)

**Limitations:**
- ❌ One-way only (server → client)
- ❌ Text only (no binary data)
- ❌ Browser connection limit (6 per domain)

### How SSE Works

**SSE is just HTTP with special content type:**

**Client initiates:**
```http
GET /api/live-updates HTTP/1.1
Accept: text/event-stream
```

**Server responds and keeps connection open:**
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"restaurant": "Pizza Palace", "rating": 4.5}

data: {"restaurant": "Burger Joint", "rating": 4.2}

data: {"restaurant": "Taco Stand", "rating": 4.8}
```

**Key points:**
- Still HTTP (not WebSocket upgrade!)
- Text-based format
- Each message ends with **two newlines** (`\n\n`)

### SSE Message Format

**Simple message:**
```
data: Hello World

```

**JSON message:**
```
data: {"type": "rating_update", "rating": 4.5}

```

**Multi-line message:**
```
data: Line 1
data: Line 2
data: Line 3

```

**Message with ID (for reconnection):**
```
id: 12345
data: {"rating": 4.5}

```

**Named events:**
```
event: rating_update
data: {"rating": 4.5}

event: new_restaurant
data: {"name": "Sushi Place"}

```

### Auto-Reconnection (Killer Feature!)

**What happens when connection drops:**

1. **Connection lost** (server crash, network failure)
2. **Browser detects** connection closed
3. **Browser automatically retries** after 3 seconds (default)
4. **Reconnects** with `Last-Event-ID` header

**Server implementation:**
```javascript
app.get('/api/live-updates', (req, res) => {
  const lastEventId = req.headers['last-event-id'];

  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  // Client is reconnecting, send missed events
  if (lastEventId) {
    const missedEvents = getEventsSince(lastEventId);
    missedEvents.forEach(event => {
      res.write(`id: ${event.id}\n`);
      res.write(`data: ${JSON.stringify(event.data)}\n\n`);
    });
  }

  // Continue with live updates...
});
```

**Client never misses events!** This is huge for reliability.

### Custom Reconnection Timing

**Change retry delay:**
```
retry: 10000
data: Update

```
Browser waits 10 seconds before reconnecting (instead of default 3)

### Client Implementation

```javascript
const eventSource = new EventSource('/api/live-updates');

// Handle messages
eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Update:', data);
  updateUI(data);
};

// Handle named events
eventSource.addEventListener('rating_update', (event) => {
  const data = JSON.parse(event.data);
  updateRating(data);
});

eventSource.addEventListener('new_restaurant', (event) => {
  const data = JSON.parse(event.data);
  showNotification(data);
});

// Handle connection open
eventSource.onopen = () => {
  console.log('Connected!');
};

// Handle errors
eventSource.onerror = (error) => {
  console.error('SSE error:', error);
  // Browser auto-reconnects!
};

// Close connection
eventSource.close();
```

### Server Implementation (Node.js)

```javascript
const express = require('express');
const app = express();

app.get('/api/live-updates', (req, res) => {
  // Set SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Send initial message
  res.write('data: Connected\n\n');

  // Send updates every 5 seconds
  const interval = setInterval(() => {
    const update = {
      restaurant: 'Pizza Palace',
      rating: (Math.random() * 2 + 3).toFixed(1),
      timestamp: Date.now()
    };

    res.write(`data: ${JSON.stringify(update)}\n\n`);
  }, 5000);

  // Cleanup on disconnect
  req.on('close', () => {
    clearInterval(interval);
    console.log('Client disconnected');
  });
});
```

### Real-World Example: Live Dashboard

**Server subscribes to Redis:**
```javascript
app.get('/dashboard/live', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  // Subscribe to Redis pub/sub
  const subscriber = redis.createClient();
  await subscriber.subscribe('restaurant_updates');

  subscriber.on('message', (channel, message) => {
    const update = JSON.parse(message);

    if (update.type === 'rating') {
      res.write(`event: rating_update\n`);
      res.write(`id: ${update.id}\n`);
      res.write(`data: ${JSON.stringify(update)}\n\n`);
    }
  });

  req.on('close', () => {
    subscriber.unsubscribe();
    subscriber.quit();
  });
});
```

### SSE vs WebSockets - Decision Matrix

| Feature | SSE | WebSockets |
|---------|-----|------------|
| Direction | Server → Client only | Bidirectional |
| Protocol | HTTP | WebSocket (upgraded) |
| Browser support | ✅ All | ✅ All modern |
| Auto-reconnect | ✅ Built-in | ❌ Must implement |
| Event IDs / Resume | ✅ Built-in | ❌ Must implement |
| Binary data | ❌ Text only | ✅ Binary |
| Implementation | ✅ Simpler | More complex |
| Connection limit | 6 per domain | Higher |

### When to Use SSE

#### ✅ Perfect for:

**1. Live feeds/notifications**
- Social media feeds (new posts)
- Stock tickers
- Sports scores
- System monitoring dashboards

**2. Progress updates**
- File upload progress
- Long-running job status
- Build/deployment progress

**3. Server-initiated updates**
- New message notifications
- Order status changes
- Breaking news alerts

**Key:** Server pushes, client rarely/never sends back

#### ❌ Don't use SSE for:

**1. Bidirectional communication**
- Live chat (user types messages)
- Multiplayer games
- Collaborative editing
→ Use WebSockets

**2. Binary data**
- Video streaming
- File transfers
→ Use WebSockets

**3. Backend services**
- Service-to-service
→ Use gRPC streaming

### SSE Scaling (Same as WebSockets)

**Horizontal scaling + Redis Pub/Sub:**

```
         Load Balancer
         /     |     \
    Server1  Server2  Server3
        \      |      /
         Redis Pub/Sub
            ↑
     (Backend publishes)
```

Each server:
- Holds subset of SSE connections
- Subscribes to Redis
- Forwards events to connected clients

### SSE Limitations & Workarounds

#### Limitation 1: Browser Connection Limit
**Problem:** 6 SSE connections per domain
```
Tab 1-6: Each opens EventSource
Tab 7: Blocked! ❌
```

**Workarounds:**
- Use SharedWorker (share one connection across tabs)
- Use subdomains (api1.example.com, api2.example.com)

#### Limitation 2: No Request Headers
**Problem:** Can't send auth token in headers
```javascript
// Doesn't work:
const es = new EventSource('/live', {
  headers: { 'Authorization': 'Bearer token' } // ❌
});
```

**Workaround:** Auth in URL query param
```javascript
const token = getAuthToken();
const es = new EventSource(`/live?token=${token}`);
```

Or use cookies (sent automatically)

#### Limitation 3: Text Only
**Problem:** Can't send binary efficiently

**Workaround:** Base64 encode (33% size increase)
```javascript
// Server
const base64 = imageBuffer.toString('base64');
res.write(`data: ${base64}\n\n`);
```

If binary is frequent → use WebSockets

### Decision Tree: Real-Time Communication

```
Need real-time browser updates?
  ├─ NO → Use REST
  │
  └─ YES → Does client send data back?
      ├─ YES → Use WebSockets
      │         (chat, games, collaboration)
      │
      └─ NO → Use SSE
                (feeds, notifications, monitoring)

Backend service communication?
  └─ Use gRPC streaming
```

## Practice Questions

1. **What are the main advantages of SSE over WebSockets?**
   - Simpler implementation (just HTTP)
   - Built-in auto-reconnect
   - Event IDs for resuming (never miss events)
   - Easier debugging (text-based)
   - Perfect when only server needs to push

2. **How does SSE handle reconnection after network failure?**
   - Browser detects connection closed
   - Auto-retries after 3 seconds (configurable)
   - Sends `Last-Event-ID` header
   - Server sends missed events since that ID
   - Client never misses updates

3. **When would you choose WebSockets over SSE?**
   - Need bidirectional communication (client sends messages)
   - Need binary data transfer
   - Client needs to send frequently
   - Example: Live chat, multiplayer games

4. **How do you scale SSE connections?**
   - Same as WebSockets: horizontal scaling
   - Use Redis Pub/Sub for cross-server messaging
   - Load balancer with sticky sessions
   - Separate connection handling from business logic

## Real-World Examples

- **Twitter**: SSE for live tweet stream updates
- **Stock trading platforms**: SSE for live price updates
- **GitHub**: SSE for real-time notifications
- **CI/CD platforms**: SSE for build logs streaming

## Resources & References

- Server-Sent Events W3C Specification
- EventSource API Documentation
- Redis Pub/Sub for SSE Broadcasting
- SSE vs WebSockets: When to Use Each
