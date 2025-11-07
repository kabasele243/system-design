# Encryption

## Learning Objectives
- Understand encryption at rest vs in transit
- Learn symmetric vs asymmetric encryption
- Understand hashing and salting

## Notes

### Encryption at Rest

**Definition:** Encrypting data stored on disk (database, files, backups)

**Why needed:** Protects data if physical storage is stolen or accessed

**Example:**
```javascript
// Store encrypted password
const bcrypt = require('bcrypt');
const hashedPassword = await bcrypt.hash('mypassword123', 10);
await db.users.insert({email, password: hashedPassword});

// Verify password
const valid = await bcrypt.compare(inputPassword, user.password);
```

**Real-world:**
- Database encryption (PostgreSQL, MySQL with encryption enabled)
- File encryption (encrypted S3 buckets)
- Backup encryption (encrypted database dumps)

---

### Encryption in Transit

**Definition:** Encrypting data while traveling over the network

**Why needed:** Prevents man-in-the-middle attacks, eavesdropping

**HTTPS/TLS:**
```
User                    Server
  |                       |
  |--Client Hello-------->|  (TLS handshake starts)
  |<--Server Certificate--|  (Server sends public key)
  |--Encrypted Session--->|  (Client encrypts with public key)
  |<--Data (encrypted)----|
  |--Data (encrypted)---->|
```

**Real-world:**
- HTTPS for web traffic
- WSS (WebSocket Secure) for real-time
- SFTP for file transfers
- VPN for network traffic

---

### Symmetric vs Asymmetric Encryption

**Symmetric Encryption:**

**Definition:** Same key for encryption and decryption

**Analogy:** Single key for a lock

```javascript
const crypto = require('crypto');

const algorithm = 'aes-256-cbc';
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

// Encrypt
function encrypt(text) {
    const cipher = crypto.createCipheriv(algorithm, key, iv);
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
}

// Decrypt
function decrypt(encrypted) {
    const decipher = crypto.createDecipheriv(algorithm, key, iv);
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
}

const encrypted = encrypt('Secret message');
const decrypted = decrypt(encrypted);  // "Secret message"
```

**Pros:**
- Fast (AES encrypts GB/sec)
- Low overhead

**Cons:**
- Key distribution problem (how to share key securely?)

**Use cases:**
- Encrypting large files
- Database encryption
- Disk encryption
- Session encryption (after key exchange)

**Common algorithms:**
- AES-256 (recommended)
- ChaCha20
- 3DES (deprecated)

---

**Asymmetric Encryption:**

**Definition:** Public key for encryption, private key for decryption

**Analogy:** Mailbox (anyone can drop mail, only owner has key)

```javascript
const crypto = require('crypto');

// Generate key pair
const {publicKey, privateKey} = crypto.generateKeyPairSync('rsa', {
    modulusLength: 2048
});

// Encrypt with public key
function encrypt(text) {
    return crypto.publicEncrypt(publicKey, Buffer.from(text)).toString('base64');
}

// Decrypt with private key
function decrypt(encrypted) {
    return crypto.privateDecrypt(privateKey, Buffer.from(encrypted, 'base64')).toString();
}

const encrypted = encrypt('Secret message');
const decrypted = decrypt(encrypted);  // "Secret message"
```

**Pros:**
- No key distribution problem
- Can verify sender (digital signatures)

**Cons:**
- Slow (1000x slower than symmetric)
- Limited data size

**Use cases:**
- HTTPS/TLS handshake
- SSH key authentication
- Digital signatures
- PGP email encryption

**Common algorithms:**
- RSA-2048 (recommended)
- ECC (Elliptic Curve)
- Ed25519 (fast, modern)

---

**When to use each:**

| Scenario | Use |
|----------|-----|
| **Encrypt large file** | Symmetric (AES) |
| **Send public message that only recipient can read** | Asymmetric (RSA) |
| **HTTPS connection** | Both! (Asymmetric for key exchange, then Symmetric for data) |
| **Database encryption** | Symmetric (AES) |
| **SSH login** | Asymmetric (key pair) |
| **Password hashing** | Neither! Use hashing (bcrypt) |

---

### Hashing

**What is hashing?**

Hashing is **one-way**: You can't decrypt it!

```
Hash("password123") → "5f4dcc3b..."
Hash("5f4dcc3b...") → IMPOSSIBLE! ❌
```

**Analogy:** Meat grinder
- Put steak in → Ground meat comes out
- Can you reconstruct the steak from ground meat? No! ✅

**Hashing vs Encryption:**

| Feature | Hashing | Encryption |
|---------|---------|------------|
| **Reversible** | No (one-way) | Yes (two-way) |
| **Use case** | Passwords, checksums | Secure communication |
| **Output size** | Fixed (256 bits) | Variable |
| **Key needed** | No | Yes |

---

**Common hashing algorithms:**

**1. MD5 (INSECURE ❌):**
```javascript
// DO NOT USE FOR PASSWORDS! ❌
const hash = crypto.createHash('md5').update('password').digest('hex');
```
- Fast → Easy to brute force
- Collisions found (different inputs → same hash)

**2. SHA-256 (OK for checksums, NOT for passwords):**
```javascript
const hash = crypto.createHash('sha256').update('password').digest('hex');
```
- Fast → Still brute-forceable
- Use for: File integrity, checksums
- Don't use for: Passwords

**3. bcrypt (RECOMMENDED for passwords ✅):**
```javascript
const bcrypt = require('bcrypt');

// Hash password (takes ~100ms intentionally!)
const hashedPassword = await bcrypt.hash('password123', 10);
// $2b$10$N9qo8uLOickgx2ZMRZoMye...

// Verify password
const valid = await bcrypt.compare('password123', hashedPassword);
```

**Why bcrypt:**
- Slow by design (prevents brute force)
- Includes salt automatically
- Adaptive (can increase rounds as computers get faster)

**Other good options:**
- Argon2 (newer, more secure)
- scrypt (memory-hard)
- PBKDF2 (older but acceptable)

---

**Salting:**

**Problem:** Same password → Same hash

```javascript
// WITHOUT salt ❌
hash("password123") → "5f4dcc3b..."
hash("password123") → "5f4dcc3b..."  (same!)

// Attacker can use rainbow table:
// "password123" → "5f4dcc3b..."
// If hash matches, password found! ❌
```

**Solution:** Add random salt before hashing

```javascript
// WITH salt ✅
salt1 = "a3f8"
hash("password123" + "a3f8") → "9d3e7a1b..."

salt2 = "k9z2"
hash("password123" + "k9z2") → "1f6c2e8d..."  (different!)

// Rainbow tables useless! ✅
```

**bcrypt handles salting automatically:**
```javascript
const hash1 = await bcrypt.hash('password123', 10);
// $2b$10$N9qo8uLOickgx2ZMRZoMye...
//        ^^^^^^^^^^^^^^^^^^^ ← salt

const hash2 = await bcrypt.hash('password123', 10);
// $2b$10$K3m2nL9pQrtHdx8VmNzRde...
//        ^^^^^^^^^^^^^^^^^^^ ← different salt

// Same password, different hashes! ✅
```

---

### HTTPS/TLS

**How HTTPS works:**

**1. TLS Handshake (Asymmetric):**
```
Client                                Server
  |                                     |
  |--"Hello, I speak TLS 1.3"--------->|
  |<--Server Certificate (public key)--|
  |                                     |
  |- Verify certificate with CA ✅     |
  |                                     |
  |--Generate session key------------->|
  |  (encrypted with server's public key)
  |                                     |
  |  Server decrypts with private key ✅
  |                                     |
  [Both now have shared session key]
```

**2. Data Transfer (Symmetric):**
```
Client                                Server
  |                                     |
  |--GET /api/users------------------->|
  |  (encrypted with session key: AES)
  |                                     |
  |<--[user data]----------------------|
  |  (encrypted with session key: AES)
```

**Why hybrid approach:**
- Asymmetric: Secure key exchange (slow, but only once)
- Symmetric: Fast encryption (for all data transfer)

**TLS Handshake details:**
1. Client Hello (supported ciphers)
2. Server Hello (chosen cipher, certificate)
3. Client verifies certificate (is it signed by trusted CA?)
4. Key exchange (ECDHE or RSA)
5. Finished messages (handshake complete)

---

### Real-World Example: User Registration

```javascript
// Scenario 1: Store password ❌
await db.users.insert({
    email: 'alice@example.com',
    password: 'mypassword123'  // NEVER DO THIS! ❌
});

// Scenario 2: Hash password (still vulnerable) ❌
const hash = crypto.createHash('sha256').update('mypassword123').digest('hex');
await db.users.insert({
    email: 'alice@example.com',
    password: hash  // Brute-forceable! ❌
});

// Scenario 3: Hash + Salt (manual, error-prone) ⚠️
const salt = crypto.randomBytes(16).toString('hex');
const hash = crypto.createHash('sha256').update('mypassword123' + salt).digest('hex');
await db.users.insert({
    email: 'alice@example.com',
    password: hash,
    salt: salt
});

// Scenario 4: bcrypt (CORRECT ✅)
const bcrypt = require('bcrypt');
const hashedPassword = await bcrypt.hash('mypassword123', 10);
await db.users.insert({
    email: 'alice@example.com',
    password: hashedPassword  // Includes salt automatically! ✅
});

// Login verification
const user = await db.users.findOne({email: 'alice@example.com'});
const valid = await bcrypt.compare(inputPassword, user.password);
if (valid) {
    // Login successful
}
```

---

## Practice Questions

1. **What's the difference between encryption and hashing?**

Answer:
- **Encryption:** Two-way (encrypt ⇔ decrypt), requires key, used for secure communication
- **Hashing:** One-way (hash → CANNOT unhash), no key, used for passwords and checksums

2. **Why do we salt passwords before hashing?**

Answer:
- Without salt: Same password → same hash → rainbow table attack works
- With salt: Same password → different hash → rainbow tables useless
- Salt is random, stored with hash
- Example: bcrypt automatically includes salt in output

3. **Explain how HTTPS keeps data secure**

Answer:
```
1. TLS Handshake (Asymmetric Encryption):
   - Client connects to server
   - Server sends certificate (contains public key)
   - Client verifies certificate with CA
   - Client generates session key, encrypts with server's public key
   - Server decrypts with private key
   - Both now have shared session key

2. Data Transfer (Symmetric Encryption):
   - All data encrypted with session key (AES-256)
   - Fast and secure

Why hybrid:
- Asymmetric: Secure key exchange (slow, used once)
- Symmetric: Fast data encryption (used for all traffic)
```

---

## Real-World Examples

**Stripe:**
- HTTPS for API calls
- Card numbers encrypted at rest (AES-256)
- PCI DSS compliant
- Tokenization (card → token, never store real card)

**Signal (Messaging App):**
- End-to-end encryption (E2EE)
- Public key cryptography (X3DH)
- Double Ratchet algorithm
- Messages encrypted on device, server can't read

**AWS:**
- S3: Encryption at rest (AES-256, optional)
- RDS: Database encryption (AES-256)
- KMS: Key management service
- TLS for all API calls

---

## Resources & References

- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [bcrypt npm package](https://www.npmjs.com/package/bcrypt)
- [Node.js Crypto documentation](https://nodejs.org/api/crypto.html)
- [How HTTPS works (comic)](https://howhttps.works/)
