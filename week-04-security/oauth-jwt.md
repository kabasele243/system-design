# OAuth 2.0 & JWT

## Learning Objectives
- Understand OAuth 2.0 flow
- Learn how JWT tokens work
- Understand when to use each

## Notes

### OAuth 2.0

**What is OAuth?**

OAuth 2.0 is an **authorization framework** that allows third-party apps to access user data **without sharing passwords**.

**Real-world analogy:**
```
Hotel key card system:
- You check in at hotel (authentication)
- Hotel gives you key card (access token)
- Key card opens your room ONLY (limited scope)
- Key card expires after checkout (token expiration)
- Hotel can deactivate card anytime (revoke token)
```

**Problem OAuth solves:**
```
Bad way (pre-OAuth):
Spotify: "Give us your Google password so we can access your email"
You: *gives password* ❌
→ Spotify has FULL access to your Google account!
→ Can read emails, delete files, everything!

Good way (OAuth):
Spotify: "Sign in with Google"
Google: "Spotify wants to: Read your email (gmail.readonly)"
You: "Allow" ✅
→ Spotify gets LIMITED access (read-only)
→ Spotify NEVER sees your password!
→ You can revoke access anytime!
```

---

### OAuth 2.0 Flow

**The 4 Players:**

1. **Resource Owner** = You (the user)
2. **Client** = Spotify (the app wanting access)
3. **Authorization Server** = Google's OAuth server
4. **Resource Server** = Google's API (Gmail, Drive, etc.)

**Complete Flow:**

```
1. User clicks "Sign in with Google" on Spotify
   ↓
2. Spotify redirects to Google with:
   - client_id: "spotify-12345"
   - scope: "gmail.readonly profile"
   - redirect_uri: "https://spotify.com/callback"
   ↓
3. User logs into Google (authentication)
   ↓
4. Google shows: "Spotify wants to: Read your email, See your profile"
   User clicks "Allow" ✅
   ↓
5. Google redirects back to Spotify with authorization code:
   https://spotify.com/callback?code=ABC123
   ↓
6. Spotify exchanges code for access token (server-to-server):
   POST https://oauth.google.com/token
   {
     code: "ABC123",
     client_id: "spotify-12345",
     client_secret: "SECRET_KEY",
     redirect_uri: "https://spotify.com/callback"
   }
   ↓
7. Google returns access token:
   {
     access_token: "ya29.a0AfH6SMC...",
     token_type: "Bearer",
     expires_in: 3600,
     scope: "gmail.readonly profile"
   }
   ↓
8. Spotify uses access token to fetch your Gmail:
   GET https://gmail.googleapis.com/gmail/v1/users/me/messages
   Authorization: Bearer ya29.a0AfH6SMC...
   ↓
9. Google returns your emails (read-only!) ✅
```

**Visual Timeline:**
```
User          Spotify        Google Auth     Google API
 |               |               |              |
 |--Click "Sign in with Google"->|              |
 |               |--Redirect---->|              |
 |<------Login page--------------|              |
 |--Enter password-------------->|              |
 |<--"Allow Spotify?"------------|              |
 |--"Allow"--------------------->|              |
 |               |<--Code "ABC123"|             |
 |               |--Exchange code->|            |
 |               |<--Access token--|            |
 |               |--GET /emails---------------->|
 |               |<--Email data-----------------|
 |<--Show emails-|               |              |
```

---

### OAuth Roles Explained

**Resource Owner (You):**
- Owns the data (Gmail, Google Drive)
- Decides what to share
- Can revoke access anytime

**Client (Spotify):**
- Wants to access your data
- Registered with Google (has client_id and client_secret)
- Never sees your password

**Authorization Server (Google OAuth):**
- Authenticates you
- Shows consent screen ("Allow Spotify to...")
- Issues access tokens

**Resource Server (Gmail API):**
- Hosts the data
- Validates access tokens
- Returns requested data

---

### OAuth Scopes

**Scopes = Permissions the app is requesting**

```
gmail.readonly        → Read emails only
gmail.send           → Send emails
drive.file           → Access files created by this app
drive                → Full Drive access
profile              → Read profile info (name, email)
openid               → Unique user ID (for login)
```

**Example:**
```
User sees:
"Spotify wants to:
✅ Read your email (gmail.readonly)
✅ See your profile (profile)
❌ Delete your files (not requested)"

User thinks: "Why does Spotify need my email?!" 🤔
→ Always read the scopes carefully!
```

---

### JWT (JSON Web Tokens)

**What is JWT?**

JWT is a **self-contained token** that carries user info + signature.

**Structure:**
```
Header.Payload.Signature

Example:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEyMzQ1LCJyb2xlIjoiYWRtaW4iLCJleHAiOjE2ODAwMDAwMDB9.5Y2Q9Z8K1xF7N3P6M9L0R2J4T8V5H1G3S6W9C0B4A7D

Base64 decoded:
{
  "alg": "HS256",
  "typ": "JWT"
}
.
{
  "userId": 12345,
  "role": "admin",
  "exp": 1680000000
}
.
<signature>
```

---

### JWT Structure

**1. Header:**
```json
{
  "alg": "HS256",        // Signing algorithm (HMAC SHA256)
  "typ": "JWT"           // Token type
}
```

**2. Payload (Claims):**
```json
{
  "userId": 12345,       // Custom claim
  "role": "admin",       // Custom claim
  "email": "alice@example.com",
  "iat": 1677000000,     // Issued at (timestamp)
  "exp": 1677086400      // Expires at (timestamp)
}
```

**Standard claims:**
- `iss` = Issuer (who created the token)
- `sub` = Subject (user ID)
- `aud` = Audience (who can use this token)
- `exp` = Expiration time
- `iat` = Issued at
- `nbf` = Not before (token not valid until this time)

**3. Signature:**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  SECRET_KEY
)
```

**Why signature matters:**
```
If attacker changes payload:
{
  "userId": 12345,
  "role": "admin"   →   "role": "superadmin"  ❌
}

Signature becomes invalid!
Server rejects token ✅
```

---

### How JWT Works

**Login Flow:**
```javascript
// 1. User logs in
app.post('/login', async (req, res) => {
    const {email, password} = req.body;

    const user = await db.users.findOne({email});
    const valid = await bcrypt.compare(password, user.hashedPassword);

    if (!valid) return res.status(401).json({error: 'Invalid credentials'});

    // 2. Generate JWT
    const token = jwt.sign(
        {
            userId: user.id,
            role: user.role,
            email: user.email
        },
        process.env.JWT_SECRET,
        {
            expiresIn: '1h'
        }
    );

    // 3. Return token to client
    res.json({token});
});
```

**Using JWT in requests:**
```javascript
// Client stores token
localStorage.setItem('token', token);

// Client sends token in every request
fetch('/api/profile', {
    headers: {
        'Authorization': `Bearer ${token}`
    }
});
```

**Server validates JWT:**
```javascript
// Middleware
function requireAuth(req, res, next) {
    const authHeader = req.headers.authorization;
    if (!authHeader) return res.status(401).json({error: 'No token'});

    const token = authHeader.split(' ')[1];  // "Bearer <token>"

    try {
        // Verify signature and decode payload
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;  // {userId, role, email}
        next();
    } catch (error) {
        res.status(401).json({error: 'Invalid token'});
    }
}

// Protected route
app.get('/api/profile', requireAuth, (req, res) => {
    res.json({
        userId: req.user.userId,
        role: req.user.role
    });
});
```

---

### JWT Security

**Pros:**
✅ **Stateless** - No database lookup needed
✅ **Scalable** - Works across multiple servers
✅ **Self-contained** - All info in the token
✅ **Fast** - No session storage required

**Cons:**
❌ **Can't revoke easily** - Valid until expiration
❌ **Payload visible** - Base64 encoded, not encrypted
❌ **Token theft** - If stolen, attacker has access until expiration

**Best Practices:**

**1. Use short expiration:**
```javascript
const accessToken = jwt.sign({userId}, SECRET, {expiresIn: '15m'});
const refreshToken = jwt.sign({userId}, SECRET, {expiresIn: '7d'});
```

**2. Store securely:**
```javascript
// BAD: localStorage (vulnerable to XSS) ❌
localStorage.setItem('token', token);

// GOOD: HttpOnly cookie (not accessible by JavaScript) ✅
res.cookie('token', token, {
    httpOnly: true,
    secure: true,     // HTTPS only
    sameSite: 'strict'
});
```

**3. Never put sensitive data in payload:**
```javascript
// BAD ❌
jwt.sign({userId, password: 'secret123'}, SECRET);

// GOOD ✅
jwt.sign({userId, role: 'admin'}, SECRET);
```

**4. Validate on every request:**
```javascript
app.get('/api/*', requireAuth, (req, res, next) => {
    // All /api/* routes require valid JWT
});
```

---

### OAuth vs JWT

**OAuth 2.0:**
- **Purpose:** Authorization framework
- **Use case:** "Sign in with Google", third-party access
- **Token type:** Opaque access tokens (random strings)
- **Validation:** Server asks authorization server "Is this token valid?"
- **Revocation:** Can revoke tokens centrally

**JWT:**
- **Purpose:** Token format
- **Use case:** Stateless authentication
- **Token type:** Self-contained (has user info)
- **Validation:** Server verifies signature locally
- **Revocation:** Hard to revoke (need blacklist)

**Can use together:**
```
OAuth 2.0 issues access tokens → Those tokens can BE JWTs!

Example: Google OAuth returns:
{
  "access_token": "eyJhbGci..." ← This IS a JWT!
}
```

---

### Real-World Example: Spotify + Google

**Scenario:** Spotify wants to import your YouTube playlists

**OAuth Flow:**
```
1. User: "Import from YouTube"
   ↓
2. Spotify: Redirects to Google with scope="youtube.readonly"
   ↓
3. Google: "Spotify wants to: View your YouTube playlists"
   User: "Allow" ✅
   ↓
4. Google: Issues access token (JWT format!)
   {
     "access_token": "eyJhbGci...",
     "scope": "youtube.readonly",
     "expires_in": 3600
   }
   ↓
5. Spotify: Decodes JWT (optional, for info)
   {
     "userId": "google-user-123",
     "scope": "youtube.readonly",
     "exp": 1680000000
   }
   ↓
6. Spotify: Uses token to call YouTube API
   GET https://youtube.googleapis.com/youtube/v3/playlists
   Authorization: Bearer eyJhbGci...
   ↓
7. YouTube API: Verifies token signature ✅
   Returns playlists
```

---

### When to Use OAuth vs Custom JWT

**Use OAuth when:**
- Third-party login ("Sign in with Google/Facebook")
- Delegated access (app accessing user's Google Drive)
- Need to revoke access easily
- Multi-service ecosystem

**Use Custom JWT when:**
- Your own authentication system
- Stateless API authentication
- Microservices communication
- Mobile app authentication

**Example: Uber**
```
Login:
- Custom JWT (Uber's own auth system)
- JWT contains: {userId, role, exp}

Payment:
- OAuth (Stripe payment processing)
- User authorizes Uber to charge their card
```

---

### Refresh Token Pattern

**Problem:** Access tokens expire quickly (15 min)
**Solution:** Use refresh tokens (long-lived)

```javascript
// Login returns both tokens
app.post('/login', async (req, res) => {
    const user = await authenticate(req.body);

    const accessToken = jwt.sign({userId: user.id}, SECRET, {expiresIn: '15m'});
    const refreshToken = jwt.sign({userId: user.id}, SECRET, {expiresIn: '7d'});

    // Store refresh token in database (for revocation)
    await db.refreshTokens.insert({userId: user.id, token: refreshToken});

    res.json({accessToken, refreshToken});
});

// Refresh endpoint
app.post('/refresh', async (req, res) => {
    const {refreshToken} = req.body;

    // Verify refresh token
    const decoded = jwt.verify(refreshToken, SECRET);

    // Check if revoked
    const exists = await db.refreshTokens.findOne({token: refreshToken});
    if (!exists) return res.status(401).json({error: 'Refresh token revoked'});

    // Issue new access token
    const accessToken = jwt.sign({userId: decoded.userId}, SECRET, {expiresIn: '15m'});
    res.json({accessToken});
});
```

**Flow:**
```
Client              Server
  |                   |
  |--Login----------->|
  |<--Access (15m)----|
  |<--Refresh (7d)----|
  |                   |
  |--API call-------->| (access token valid ✅)
  |<--Data------------|
  |                   |
 [16 minutes later]
  |                   |
  |--API call-------->| (access token expired ❌)
  |<--401 Unauthorized|
  |                   |
  |--Refresh-------->| (send refresh token)
  |<--New Access-----|
  |                   |
  |--API call-------->| (new access token ✅)
  |<--Data------------|
```

---

## Practice Questions

1. **Explain the "Sign in with Google" flow using OAuth**

Answer:
```
1. User clicks "Sign in with Google" on App
2. App redirects to Google with client_id and requested scopes
3. User logs into Google (if not already logged in)
4. Google shows consent: "App wants to: Read your profile"
5. User clicks "Allow"
6. Google redirects back with authorization code
7. App exchanges code for access token (server-to-server)
8. App uses access token to fetch user's Google profile
9. App creates session for user
```

2. **What information goes in a JWT payload?**

Good to include:
- `userId`: Identify the user
- `role`: User's role (admin, user, etc.)
- `email`: User's email (non-sensitive)
- `exp`: Expiration timestamp
- `iat`: Issued at timestamp

Never include:
- Passwords
- Credit card numbers
- Social security numbers
- Private encryption keys

3. **How does JWT prevent tampering?**

Answer:
```
JWT has 3 parts: Header.Payload.Signature

Signature = HMAC(Header + Payload, SECRET_KEY)

If attacker changes payload:
1. Attacker modifies: {"role": "user"} → {"role": "admin"}
2. Attacker can't regenerate signature (doesn't have SECRET_KEY)
3. Server recalculates signature: HMAC(Header + Modified Payload, SECRET_KEY)
4. Signatures don't match → Token rejected ❌

Only server with SECRET_KEY can create valid signatures ✅
```

---

## Real-World Examples

**GitHub OAuth:**
- Scopes: `repo`, `user`, `admin:org`
- Use case: VS Code accessing your repos
- Token format: Opaque (not JWT)

**Auth0 (Identity Provider):**
- Issues JWTs for OAuth access tokens
- Includes custom claims: `email_verified`, `locale`
- Can be verified without calling Auth0 (stateless)

**Stripe:**
- API keys (not OAuth for direct API usage)
- OAuth for Connect (third-party platform access)
- Refresh tokens never expire (revoke manually)

---

## Resources & References

- [OAuth 2.0 Simplified](https://www.oauth.com/)
- [JWT.io - Decode and verify JWTs](https://jwt.io/)
- [RFC 6749 - OAuth 2.0 Specification](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7519 - JWT Specification](https://datatracker.ietf.org/doc/html/rfc7519)
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
