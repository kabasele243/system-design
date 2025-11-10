# WebSockets

## Learning Objectives
- Understand WebSocket protocol and handshake
- Learn bidirectional real-time communication
- Master scaling WebSocket connections

## Notes

### What are WebSockets?

**Problem REST polling solves poorly:**
- Live delivery tracking: User wants real-time driver location updates
- REST polling: Client requests location every 2 seconds
  - ❌ 30 requests/minute even if driver hasn't moved
  - ❌ Wastes bandwidth (90% return "no change")
  - ❌ Battery drain on mobile
  - ❌ Not truly real-time (2 second delay)

**WebSocket solution:**
- **Persistent bidirectional connection** between client and server
- Server pushes updates instantly (< 50ms latency)
- Browser native support
- One connection for entire session

### How WebSockets Work

#### The Handshake (HTTP Upgrade)

**Step 1: Client sends HTTP upgrade request**
```
GET /driver/track HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
```

**Step 2: Server responds with upgrade**
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

**Step 3: Now WebSocket protocol (not HTTP!)**
- No more HTTP headers
- Just raw data frames
- Connection stays open minutes/hours
- Bidirectional communication

### Bidirectional Communication

Unlike REST (request → response), **both sides can send anytime:**

**Server → Client (push):**
```javascript
ws.send(JSON.stringify({
  lat: 40.7128,
  lng: -74.0060,
  eta: '3 minutes'
}));
```

**Client → Server (send):**
```javascript
ws.send(JSON.stringify({
  action: 'contact_driver',
  message: "I'm waiting outside"
}));
```

Both directions work simultaneously, no request/response cycle!

### Client Implementation

```javascript
// Connect to WebSocket
const ws = new WebSocket('wss://api.delivery.com/track?orderId=123');

// Handle connection open
ws.onopen = () => {
  console.log('Connected! Tracking driver...');
};

// Receive real-time updates
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  updateMap(data.lat, data.lng);
  showETA(data.eta);
};

// Handle disconnection
ws.onclose = () => {
  console.log('Connection closed');
};

// Handle errors
ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

// Close connection
ws.close();
```

### The Scaling Challenge

**Problem: Connection State**
- REST: Connection closes immediately (stateless)
- WebSocket: Connection stays open (stateful)

**1 million concurrent users:**
- Each connection: ~10KB memory + 1 file descriptor
- 1M × 10KB = **10GB RAM just for connections!**
- Most servers max out at 65K file descriptors
- **One server can't handle it!**

### Scaling Solutions

#### 1. Horizontal Scaling + Sticky Sessions

```
              Load Balancer (sticky sessions)
              /      |      \
    Server 1    Server 2    Server 3
    (333K)      (333K)      (333K)
```

**Sticky Sessions:**
- User connects → Assigned to Server 2
- ALL messages from that user → Server 2
- Can't switch servers mid-connection (stateful!)

#### 2. Broadcasting with Redis Pub/Sub

**Problem:** Driver on Server 1, User on Server 3

```
Driver (Server 1): Updates location
User (Server 3): Needs to see update
```

**Solution: Message Queue**
```
Driver → Server 1 → Redis publish("driver_123", data)
                         ↓
                    All servers subscribe
                         ↓
                    Server 3 → Sends to User's WebSocket
```

#### 3. Room/Channel Pattern

**Don't broadcast to everyone!**

```javascript
// Group connections by order
const rooms = {
  'order_123': [ws1, ws2],     // 2 users tracking
  'order_456': [ws3],          // 1 user
  'order_789': [ws4, ws5, ws6] // 3 users
};

// Broadcast only to specific room
function broadcastToOrder(orderId, data) {
  rooms[orderId].forEach(ws => {
    ws.send(JSON.stringify(data));
  });
}

// Only 2 users get update, not 1 million!
```

#### 4. Dedicated WebSocket Gateway

```
Users → WebSocket Gateway (only manages connections)
            ↓
        Redis Pub/Sub
            ↓
        Backend Services (business logic)
```

- Gateway: Scales horizontally for connections
- Backend: Handles business logic, publishes to Redis
- Separation of concerns

### Best Practices

#### 1. Heartbeat/Ping-Pong
Keep connection alive, detect dead connections:

```javascript
// Server pings every 30 seconds
setInterval(() => {
  ws.ping();
}, 30000);

ws.on('pong', () => {
  // Connection alive
});

// If no pong in 5 seconds → connection dead, cleanup
```

**Why?** Mobile users switch networks (WiFi → 4G), connection dies silently

#### 2. Reconnection Logic
Connections drop frequently, handle gracefully:

```javascript
function connectWebSocket() {
  const ws = new WebSocket('wss://api.delivery.com/track');

  ws.onclose = () => {
    console.log('Reconnecting in 3 seconds...');
    setTimeout(connectWebSocket, 3000); // Exponential backoff
  };

  return ws;
}
```

#### 3. Message Acknowledgment
Ensure critical messages aren't lost:

```javascript
// Client sends with ID
ws.send(JSON.stringify({
  id: 'msg_123',
  action: 'contact_driver'
}));

// Server acknowledges
ws.send(JSON.stringify({ ack: 'msg_123' }));

// If no ack in 5 seconds → retry
```

#### 4. Compression
Enable WebSocket compression (60-80% reduction):

```javascript
const wss = new WebSocket.Server({
  perMessageDeflate: true
});
```

### When NOT to Use WebSockets

❌ **Simple request/response** - Use REST
```
// Bad: WebSocket for one-time fetch
ws.send('get_user_profile');

// Good: REST
fetch('/user/profile');
```

❌ **Infrequent updates** - Use Server-Sent Events
```
// If server only pushes, client never sends → Use SSE
```

❌ **Backend service communication** - Use gRPC
```
// Order Service → Payment Service → Use gRPC streaming
```

### WebSockets vs Other Protocols

| Feature | REST | gRPC | WebSockets |
|---------|------|------|------------|
| Real-time push | ✗ | ✅ | ✅ |
| Browser support | ✅ | ✗ | ✅ |
| Bidirectional | ✗ | ✅ | ✅ |
| Connection | New per request | Persistent | Persistent |
| Best for | Client APIs | Service-to-service | Browser real-time |

## Practice Questions

1. **Why do WebSockets use "sticky sessions" in load balancing?**
   - WebSocket connections are stateful (stay open)
   - Can't switch servers mid-connection
   - User must always route to same server that holds their connection
   - Unlike REST which is stateless (any server can handle request)

2. **How do you broadcast messages when users are on different servers?**
   - Use Redis Pub/Sub as message queue
   - Driver updates → Server 1 publishes to Redis
   - All servers subscribe to Redis
   - Server 3 receives message → Forwards to user's WebSocket

3. **What is the heartbeat/ping-pong mechanism?**
   - Server pings client every 30 seconds
   - Client responds with pong
   - Detects dead connections (network switches, crashes)
   - Allows server to cleanup zombie connections

4. **When would you use REST polling instead of WebSockets?**
   - Updates are infrequent (every few minutes)
   - Don't need sub-second latency
   - Simpler implementation (no connection management)
   - Example: Weather updates every 10 minutes

## Real-World Examples

- **Slack**: WebSockets for live chat, message updates
- **Uber**: WebSockets for live driver location tracking
- **WhatsApp Web**: WebSockets for real-time messaging
- **Trading platforms**: WebSockets for live stock prices

**Scale numbers:**
- Slack: Millions of WebSocket connections across 100+ servers
- WhatsApp: 10M+ concurrent WebSocket connections

## Resources & References

- WebSocket Protocol RFC 6455
- Socket.io Documentation (WebSocket library)
- Scaling WebSocket Architecture Patterns
- Redis Pub/Sub for WebSocket Broadcasting
