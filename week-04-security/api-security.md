# API Security

## Learning Objectives
- Learn common API security vulnerabilities
- Understand API authentication methods
- Learn best practices for securing APIs

## Notes

### Common API Vulnerabilities

**1. Injection Attacks**

**SQL Injection:**
```javascript
// VULNERABLE ❌
app.get('/users', (req, res) => {
    const query = `SELECT * FROM users WHERE email = '${req.query.email}'`;
    db.query(query);  // Attacker can inject: ' OR '1'='1
});

// SECURE ✅
app.get('/users', (req, res) => {
    const query = 'SELECT * FROM users WHERE email = ?';
    db.query(query, [req.query.email]);  // Parameterized query
});
```

**NoSQL Injection:**
```javascript
// VULNERABLE ❌
app.post('/login', (req, res) => {
    db.users.findOne({
        email: req.body.email,    // Attacker sends: {"$ne": null}
        password: req.body.password
    });
});

// SECURE ✅
app.post('/login', (req, res) => {
    const email = String(req.body.email);  // Force string type
    const password = String(req.body.password);
    db.users.findOne({email, password});
});
```

**Command Injection:**
```javascript
// VULNERABLE ❌
app.get('/ping', (req, res) => {
    exec(`ping -c 4 ${req.query.host}`);  // Attacker: "; rm -rf /"
});

// SECURE ✅
app.get('/ping', (req, res) => {
    const host = req.query.host.replace(/[^a-zA-Z0-9.-]/g, '');  // Whitelist
    exec(`ping -c 4 ${host}`);
});
```

---

**2. Broken Authentication**

**Problem:** Weak session management, no MFA, predictable tokens

**Examples:**
```javascript
// VULNERABLE: Predictable session IDs ❌
const sessionId = userId + timestamp;  // Guessable!

// SECURE: Random session IDs ✅
const sessionId = crypto.randomBytes(32).toString('hex');

// VULNERABLE: No brute force protection ❌
app.post('/login', authenticate);

// SECURE: Rate limiting ✅
app.post('/login', rateLimit({max: 5, windowMs: 15 * 60 * 1000}), authenticate);

// VULNERABLE: Long-lived tokens ❌
jwt.sign({userId}, SECRET, {expiresIn: '30d'});

// SECURE: Short-lived access + refresh ✅
const accessToken = jwt.sign({userId}, SECRET, {expiresIn: '15m'});
const refreshToken = jwt.sign({userId}, SECRET, {expiresIn: '7d'});
```

---

**3. Excessive Data Exposure**

**Problem:** API returns more data than needed

```javascript
// VULNERABLE: Returns password hash! ❌
app.get('/users/:id', (req, res) => {
    const user = await db.users.findById(req.params.id);
    res.json(user);  // {id, email, password, ssn, creditCard...}
});

// SECURE: Return only needed fields ✅
app.get('/users/:id', (req, res) => {
    const user = await db.users.findById(req.params.id);
    res.json({
        id: user.id,
        email: user.email,
        name: user.name
    });
});

// OR: Use serializer
class UserSerializer {
    static public(user) {
        return {id: user.id, email: user.email, name: user.name};
    }

    static private(user) {
        return {...this.public(user), phone: user.phone};
    }
}
```

---

**4. Lack of Rate Limiting**

**Problem:** API can be abused by bots, scrapers, brute-force attacks

```javascript
// VULNERABLE ❌
app.post('/api/*', handler);

// SECURE ✅
const rateLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,                   // Limit per window
    message: 'Too many requests'
});

app.post('/api/*', rateLimiter, handler);

// PER-USER rate limiting
const userLimiter = rateLimit({
    keyGenerator: (req) => req.user.id,
    max: 1000
});

app.get('/api/tweets', requireAuth, userLimiter, getTweets);
```

---

**5. Security Misconfiguration**

**Examples:**
```javascript
// VULNERABLE: Debug mode in production ❌
app.use(errorHandler({debug: true}));

// VULNERABLE: CORS allows all origins ❌
app.use(cors({origin: '*'}));

// VULNERABLE: Verbose error messages ❌
app.use((err, req, res, next) => {
    res.json({error: err.stack});  // Leaks stack trace!
});

// SECURE ✅
app.use(cors({
    origin: ['https://myapp.com'],
    credentials: true
}));

app.use((err, req, res, next) => {
    console.error(err);  // Log internally
    res.status(500).json({error: 'Internal server error'});
});
```

---

### API Authentication Methods

**1. API Keys**

**Use case:** Server-to-server, simple apps

```javascript
// Implementation
app.get('/api/data', (req, res) => {
    const apiKey = req.headers['x-api-key'];

    const valid = await db.apiKeys.findOne({key: apiKey, active: true});
    if (!valid) return res.status(401).json({error: 'Invalid API key'});

    // Process request
});

// Usage
fetch('https://api.example.com/data', {
    headers: {
        'X-API-Key': 'sk_live_abc123xyz'
    }
});
```

**Pros:** Simple, good for server-to-server
**Cons:** No expiration, hard to rotate, single key per user

---

**2. Bearer Tokens (JWT)**

**Use case:** User authentication, stateless APIs

```javascript
// Login
app.post('/login', async (req, res) => {
    const user = await authenticate(req.body);
    const token = jwt.sign({userId: user.id}, SECRET, {expiresIn: '1h'});
    res.json({token});
});

// Protected endpoint
app.get('/api/profile', requireAuth, (req, res) => {
    res.json(req.user);
});

function requireAuth(req, res, next) {
    const token = req.headers.authorization?.split(' ')[1];
    try {
        req.user = jwt.verify(token, SECRET);
        next();
    } catch {
        res.status(401).json({error: 'Unauthorized'});
    }
}

// Usage
fetch('https://api.example.com/profile', {
    headers: {
        'Authorization': 'Bearer eyJhbGci...'
    }
});
```

**Pros:** Stateless, scalable, self-contained
**Cons:** Can't revoke easily, payload visible

---

**3. OAuth 2.0**

**Use case:** Third-party access, delegated authorization

```javascript
// Authorization endpoint
app.get('/oauth/authorize', (req, res) => {
    const {client_id, scope, redirect_uri} = req.query;

    // Show consent screen
    res.render('consent', {client_id, scope});
});

// Token exchange
app.post('/oauth/token', async (req, res) => {
    const {code, client_id, client_secret} = req.body;

    const valid = await verifyAuthorizationCode(code, client_id);
    if (!valid) return res.status(401).json({error: 'Invalid code'});

    const accessToken = generateAccessToken();
    res.json({access_token: accessToken, expires_in: 3600});
});
```

**Pros:** Secure, revocable, limited scope
**Cons:** Complex, requires OAuth server

---

**4. mTLS (Mutual TLS)**

**Use case:** Microservices, high-security environments

```javascript
// Server requires client certificate
const https = require('https');
const fs = require('fs');

const options = {
    key: fs.readFileSync('server-key.pem'),
    cert: fs.readFileSync('server-cert.pem'),
    ca: fs.readFileSync('ca-cert.pem'),
    requestCert: true,
    rejectUnauthorized: true
};

https.createServer(options, (req, res) => {
    const cert = req.socket.getPeerCertificate();
    if (req.client.authorized) {
        res.writeHead(200);
        res.end('Authenticated!');
    } else {
        res.writeHead(401);
        res.end('Unauthorized');
    }
}).listen(443);
```

**Pros:** Very secure, no passwords
**Cons:** Complex setup, certificate management

---

### API Security Best Practices

**1. HTTPS Only**

```javascript
// Redirect HTTP to HTTPS
app.use((req, res, next) => {
    if (!req.secure && req.get('x-forwarded-proto') !== 'https') {
        return res.redirect('https://' + req.get('host') + req.url);
    }
    next();
});

// Set HSTS header
app.use((req, res, next) => {
    res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    next();
});
```

**Why:** Encrypts data in transit, prevents man-in-the-middle attacks

---

**2. Input Validation**

```javascript
const {body, validationResult} = require('express-validator');

app.post('/users',
    body('email').isEmail(),
    body('age').isInt({min: 0, max: 120}),
    body('website').optional().isURL(),
    (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({errors: errors.array()});
        }

        // Process valid input
    }
);
```

**Validate:**
- Data types (string, int, email, URL)
- Length limits
- Format (regex patterns)
- Allowed values (enums)

---

**3. Rate Limiting**

```javascript
// Global rate limit
const globalLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 100,
    standardHeaders: true,
    legacyHeaders: false
});

// Endpoint-specific limits
const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5,
    skipSuccessfulRequests: true
});

app.use('/api/', globalLimiter);
app.post('/login', loginLimiter, handleLogin);
```

**Response headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1677686400
```

---

**4. CORS Configuration**

```javascript
const cors = require('cors');

// VULNERABLE: Allow all ❌
app.use(cors({origin: '*'}));

// SECURE: Whitelist origins ✅
app.use(cors({
    origin: function (origin, callback) {
        const allowedOrigins = ['https://myapp.com', 'https://app.myapp.com'];
        if (!origin || allowedOrigins.indexOf(origin) !== -1) {
            callback(null, true);
        } else {
            callback(new Error('Not allowed by CORS'));
        }
    },
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization']
}));
```

**Preflight request:**
```
OPTIONS /api/users HTTP/1.1
Origin: https://malicious.com

Response:
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST
```

---

**5. API Versioning**

```javascript
// URL versioning
app.use('/api/v1/users', usersV1Router);
app.use('/api/v2/users', usersV2Router);

// Header versioning
app.use('/api/users', (req, res, next) => {
    const version = req.headers['api-version'] || '1';
    if (version === '2') {
        usersV2Router(req, res, next);
    } else {
        usersV1Router(req, res, next);
    }
});

// Deprecation warning
app.use('/api/v1/*', (req, res, next) => {
    res.setHeader('Warning', '299 - "Deprecated API version. Upgrade to v2"');
    next();
});
```

**Why:** Allows breaking changes without affecting existing clients

---

### Real-World Example: Securing Banking API

```javascript
const express = require('express');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');

const app = express();

// 1. Security headers
app.use(helmet());

// 2. HTTPS only
app.use((req, res, next) => {
    if (!req.secure) return res.redirect('https://' + req.get('host') + req.url);
    next();
});

// 3. CORS
app.use(cors({
    origin: 'https://bank.com',
    credentials: true
}));

// 4. Rate limiting
const loginLimiter = rateLimit({windowMs: 15 * 60 * 1000, max: 5});
const transferLimiter = rateLimit({windowMs: 60 * 1000, max: 3});

// 5. Input validation
const {body, validationResult} = require('express-validator');

// Login endpoint
app.post('/api/v1/login',
    loginLimiter,
    body('email').isEmail(),
    body('password').isLength({min: 8}),
    async (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) return res.status(400).json({errors});

        const user = await authenticate(req.body);
        if (!user) return res.status(401).json({error: 'Invalid credentials'});

        const token = jwt.sign({userId: user.id}, SECRET, {expiresIn: '15m'});
        res.json({token});
    }
);

// Transfer endpoint
app.post('/api/v1/transfer',
    transferLimiter,
    requireAuth,
    requireMFA,
    body('amount').isFloat({min: 0.01, max: 10000}),
    body('toAccount').isLength({min: 10, max: 10}).isNumeric(),
    async (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) return res.status(400).json({errors});

        // Check balance
        const balance = await getBalance(req.user.id);
        if (balance < req.body.amount) {
            return res.status(400).json({error: 'Insufficient funds'});
        }

        // Process transfer
        await processTransfer({
            from: req.user.id,
            to: req.body.toAccount,
            amount: req.body.amount
        });

        res.json({success: true});
    }
);

// Middleware
function requireAuth(req, res, next) {
    const token = req.headers.authorization?.split(' ')[1];
    try {
        req.user = jwt.verify(token, SECRET);
        next();
    } catch {
        res.status(401).json({error: 'Unauthorized'});
    }
}

function requireMFA(req, res, next) {
    const mfaCode = req.headers['x-mfa-code'];
    const valid = verifyMFA(req.user.id, mfaCode);
    if (!valid) return res.status(401).json({error: 'Invalid MFA code'});
    next();
}
```

---

## Practice Questions

1. **Why should APIs use HTTPS instead of HTTP?**

Answer:
- HTTPS encrypts data in transit (passwords, tokens, sensitive data)
- Prevents man-in-the-middle attacks
- Prevents eavesdropping on public networks
- Required for modern authentication (OAuth, JWT)
- Builds user trust (padlock icon in browser)

2. **What's the difference between API keys and OAuth tokens?**

| Feature | API Keys | OAuth Tokens |
|---------|----------|--------------|
| **Purpose** | Simple auth | Delegated access |
| **Expiration** | No expiration | Expires (1-24 hours) |
| **Scope** | Full access | Limited scope |
| **Revocation** | Hard | Easy (centralized) |
| **Use case** | Server-to-server | Third-party apps |

3. **Design a rate limiting strategy for a public API**

```
Endpoint: POST /api/search

Rate limits:
1. IP-based: 100 requests / 15 minutes (prevent scraping)
2. User-based: 1000 requests / hour (authenticated users)
3. Burst limit: 10 requests / second (prevent spikes)

Implementation:
- Token Bucket algorithm (allows bursts)
- Redis for distributed rate limiting
- Return headers: X-RateLimit-Limit, X-RateLimit-Remaining
- 429 status code when exceeded

Tiered limits:
- Free tier: 100/hour
- Pro tier: 1000/hour
- Enterprise: 10000/hour
```

---

## Real-World Examples

**Stripe API:**
- API keys for server-to-server
- Rate limit: 100 req/sec in test mode
- HTTPS required
- Webhook signatures (HMAC)

**GitHub API:**
- OAuth for third-party apps
- Rate limit: 5000 req/hour authenticated, 60 req/hour unauthenticated
- Versioning via Accept header
- Conditional requests (ETag)

**Twitter API:**
- OAuth 2.0 for user context
- Rate limit per endpoint (300 tweets/3 hours)
- Token Bucket algorithm
- Webhook for real-time events

---

## Resources & References

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [REST API Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html)
- [Express Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [Helmet.js Security Headers](https://helmetjs.github.io/)
