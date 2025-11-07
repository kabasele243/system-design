# Authentication vs Authorization

## Learning Objectives
- Understand the difference between authentication and authorization
- Learn common authentication methods
- Understand role-based access control (RBAC)

## Notes

### The Fundamental Difference

**Authentication = "Who are you?"**
- Proving your identity
- Like showing ID at airport security
- Examples: Password, fingerprint, face scan

**Authorization = "What can you do?"**
- Permissions and access rights
- Like checking if your ticket allows lounge access
- Happens AFTER authentication

**Coffee Shop Analogy:**
```
Authentication: Show your Starbucks membership card ✅
Authorization: Can you order the secret menu? (Gold member only) ✅
```

---

### Authentication (Who are you?)

**Definition:** Process of verifying identity

**Real-world example:**
```
User: "I'm alice@email.com"
System: "Prove it! Enter password"
User: *enters password*
System: ✅ "You are Alice"
```

**Common Methods:**

**1. Password-based (Something you know):**
```javascript
// Login flow
const user = await db.users.findOne({email: 'alice@email.com'});
const isValid = await bcrypt.compare(password, user.hashedPassword);
if (isValid) {
    return generateJWT(user);
} else {
    throw new Error('Invalid credentials');
}
```

**2. Multi-Factor Authentication (MFA):**
- Something you know (password)
- Something you have (phone, security key)
- Something you are (fingerprint, face)

**Example: Banking app**
```
Step 1: Enter password ✅
Step 2: Enter 6-digit code from SMS ✅
Step 3: Logged in! ✅
```

**3. Biometric:**
- Fingerprint (Touch ID)
- Face recognition (Face ID)
- Iris scan

**4. Token-based (OAuth, JWT):**
```
User logs in → Server generates JWT → User sends JWT with every request
```

---

### Authorization (What can you do?)

**Definition:** Determining what authenticated user can access

**Real-world example:**
```
User (authenticated): "I want to delete this post"
System (authorization): "Are you the author or admin?"
- If yes: ✅ Delete allowed
- If no: ❌ 403 Forbidden
```

---

### Authorization Models

**1. Role-Based Access Control (RBAC):**

**Definition:** Permissions based on roles

**Example: Uber**
```
Roles:
- Driver: Can accept rides, view earnings
- Passenger: Can request rides, rate drivers
- Admin: Can ban users, view all data
```

**Implementation:**
```javascript
// Check permission
function canDeletePost(user, post) {
    if (user.role === 'admin') return true;
    if (user.role === 'author' && post.authorId === user.id) return true;
    return false;
}

// Middleware
app.delete('/posts/:id', requireAuth, async (req, res) => {
    const post = await db.posts.findById(req.params.id);
    if (!canDeletePost(req.user, post)) {
        return res.status(403).json({error: 'Forbidden'});
    }
    await post.delete();
    res.json({success: true});
});
```

**2. Attribute-Based Access Control (ABAC):**

**Definition:** Permissions based on attributes (user, resource, environment)

**Example: Medical records**
```
Rule: Doctor can view patient records IF:
- Doctor.department === Patient.department
- Time is business hours (9am-5pm)
- Location is hospital network
```

**3. Access Control Lists (ACL):**

**Definition:** Specific permissions per resource

**Example: Google Drive**
```
Document: "Q4 Report.docx"
ACL:
- alice@company.com: Owner (read, write, delete, share)
- bob@company.com: Editor (read, write)
- charlie@company.com: Viewer (read only)
```

---

### Real-World Example: Uber

**Authentication:**
```
1. Passenger opens app
2. Enters phone number: +1-555-1234
3. Enters SMS code: 123456
4. System verifies code ✅
5. Logged in as "Alice"
```

**Authorization:**
```
Alice tries to:
- Request ride ✅ (Passenger role)
- Accept ride ❌ (Driver role required)
- Ban user ❌ (Admin role required)
```

**Implementation:**
```javascript
// Authentication
app.post('/login', async (req, res) => {
    const {phone, code} = req.body;
    const valid = await twilioVerify(phone, code);
    if (!valid) return res.status(401).json({error: 'Invalid code'});

    const user = await db.users.findOne({phone});
    const token = generateJWT({userId: user.id, role: user.role});
    res.json({token});
});

// Authorization
app.post('/rides/accept', requireAuth, async (req, res) => {
    if (req.user.role !== 'driver') {
        return res.status(403).json({error: 'Drivers only'});
    }
    // Accept ride logic...
});
```

---

### Real-World Example: Google Sign-In

**Why OAuth exists:**
```
Bad way (security risk):
Spotify: "Enter your Google password so we can access your email"
You: *gives Spotify your Google password* ❌
→ Spotify now has full access to your Google account!

Good way (OAuth):
Spotify: "Sign in with Google"
Google: "Spotify wants to: Read your email (gmail.readonly)"
You: "Allow" ✅
→ Spotify gets LIMITED access, NOT your password!
```

---

### Common Authentication Flows

**1. Session-based (Traditional):**
```javascript
// Login
app.post('/login', async (req, res) => {
    const user = await authenticate(req.body.email, req.body.password);
    req.session.userId = user.id;  // Store in session
    res.json({success: true});
});

// Protected route
app.get('/profile', requireAuth, (req, res) => {
    // req.session.userId is available
    res.json(req.user);
});
```

**Pros:** Simple, server controls sessions
**Cons:** Doesn't scale (sessions stored on server)

**2. Token-based (JWT):**
```javascript
// Login
app.post('/login', async (req, res) => {
    const user = await authenticate(req.body.email, req.body.password);
    const token = jwt.sign({userId: user.id, role: user.role}, SECRET);
    res.json({token});
});

// Protected route
app.get('/profile', requireAuth, (req, res) => {
    // Token validated in middleware
    res.json(req.user);
});

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
```

**Pros:** Scalable (stateless), works across services
**Cons:** Can't revoke easily, payload visible (base64)

---

### Security Best Practices

**1. Never store passwords in plain text:**
```javascript
// BAD ❌
await db.users.insert({email, password: 'mypassword123'});

// GOOD ✅
const hashedPassword = await bcrypt.hash('mypassword123', 10);
await db.users.insert({email, password: hashedPassword});
```

**2. Always use HTTPS:**
```
HTTP: Password sent in plain text ❌
HTTPS: Password encrypted in transit ✅
```

**3. Implement MFA for sensitive operations:**
```javascript
// Transfer money (requires MFA)
app.post('/transfer', requireAuth, requireMFA, async (req, res) => {
    // Process transfer
});
```

**4. Use short-lived tokens:**
```javascript
const accessToken = jwt.sign({userId}, SECRET, {expiresIn: '15m'});
const refreshToken = jwt.sign({userId}, SECRET, {expiresIn: '7d'});
```

**5. Implement rate limiting:**
```javascript
// Prevent brute force attacks
app.post('/login', rateLimit({max: 5, windowMs: 15 * 60 * 1000}), async (req, res) => {
    // Login logic
});
```

---

## Practice Questions

1. **What's the difference between authentication and authorization?**
   - Authentication = Who you are (identity verification)
   - Authorization = What you can do (permission check)

2. **Why is MFA more secure than passwords alone?**
   - Requires multiple factors (password + phone + biometric)
   - Even if password leaked, attacker needs phone/fingerprint
   - Example: Banking apps require password + SMS code

3. **Design an RBAC system for a blogging platform (admin, editor, author, reader)**

**Roles & Permissions:**
```
Admin:
- Create/edit/delete ANY post
- Create/edit/delete ANY comment
- Manage users (ban, promote)
- View analytics

Editor:
- Create/edit/delete own posts
- Edit others' posts
- Delete comments
- Publish posts

Author:
- Create/edit own posts
- Delete own comments
- Submit posts for review

Reader:
- View published posts
- Create comments on own account
```

**Implementation:**
```javascript
const permissions = {
    admin: ['post:*', 'comment:*', 'user:*', 'analytics:read'],
    editor: ['post:write', 'post:edit:any', 'comment:delete', 'post:publish'],
    author: ['post:write', 'post:edit:own', 'comment:write'],
    reader: ['post:read', 'comment:write']
};

function hasPermission(user, permission, resource) {
    const userPerms = permissions[user.role];

    // Check wildcard
    if (userPerms.includes(`${permission.split(':')[0]}:*`)) return true;

    // Check exact match
    if (userPerms.includes(permission)) {
        // Check ownership for "own" permissions
        if (permission.includes(':own')) {
            return resource.authorId === user.id;
        }
        return true;
    }

    return false;
}

// Usage
app.delete('/posts/:id', requireAuth, async (req, res) => {
    const post = await db.posts.findById(req.params.id);

    if (!hasPermission(req.user, 'post:delete', post)) {
        return res.status(403).json({error: 'Forbidden'});
    }

    await post.delete();
    res.json({success: true});
});
```

---

## Real-World Examples

**Spotify:**
- Authentication: Email/password, Google Sign-In, Facebook Login
- Authorization:
  - Free users: Ad-supported, shuffle only
  - Premium users: No ads, download songs
  - Family plan admin: Manage family members

**GitHub:**
- Authentication: Username/password, 2FA, SSH keys
- Authorization:
  - Public repos: Read access
  - Collaborators: Write access
  - Organization owners: Admin access

**AWS:**
- Authentication: Access keys, IAM roles
- Authorization: IAM policies (S3 read, EC2 write, etc.)

---

## Resources & References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [JWT.io - Introduction to JWT](https://jwt.io/introduction)
- [OAuth 2.0 RFC](https://oauth.net/2/)
- [NIST Digital Identity Guidelines](https://pages.nist.gov/800-63-3/)
