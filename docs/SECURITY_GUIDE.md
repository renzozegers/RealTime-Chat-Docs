# 🔒 Security Guide

Comprehensive guide to security in this chat service. Learn about authentication, authorization, SQL injection prevention, CORS, and more!

## 📖 Table of Contents

1. [Authentication vs Authorization](#authentication-vs-authorization)
2. [JWT (JSON Web Tokens)](#jwt-json-web-tokens)
3. [SQL Injection Prevention](#sql-injection-prevention)
4. [CORS (Cross-Origin Resource Sharing)](#cors-cross-origin-resource-sharing)
5. [Input Validation](#input-validation)
6. [Rate Limiting](#rate-limiting)
7. [Encryption](#encryption)
8. [Connection Security](#connection-security)

---

## Authentication vs Authorization

### Authentication (Who Are You?)

**Authentication** = Verifying identity (proving you are who you say you are)

**Example:**
```
User: "I'm John, here's my password"
Server: "Password correct? Yes. You are authenticated as John ✅"
```

**In Chat Service:**
```javascript
// User sends JWT token
ws.send(JSON.stringify({
  type: 'authenticate',
  token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
}));

// Server verifies token
const user = await verifyJWTToken(token);
if (user) {
  // ✅ Authenticated!
  client.user = user;
  client.authenticated = true;
}
```

### Authorization (What Can You Do?)

**Authorization** = Checking permissions (what you're allowed to do)

**Example:**
```
User: "I want to delete this message"
Server: "Are you authenticated? Yes ✅"
Server: "Are you the message owner or admin? Yes ✅"
Server: "You're authorized to delete! ✅"
```

**In Chat Service:**
```javascript
// Check if user can delete message
async function handleDeleteMessage(ws, messageId, client) {
  // 1. Authentication check
  if (!client.authenticated) {
    ws.send(JSON.stringify({ type: 'error', message: 'Not authenticated' }));
    return;
  }
  
  // 2. Authorization check
  const message = await getMessage(messageId);
  const isOwner = message.senderId === client.user.id;
  const isAdmin = client.user.role === 'admin';
  
  if (!isOwner && !isAdmin) {
    ws.send(JSON.stringify({ type: 'error', message: 'Not authorized' }));
    return;
  }
  
  // ✅ Authorized! Delete message
  await deleteMessage(messageId);
}
```

---

## JWT (JSON Web Tokens)

### What is JWT?

**JWT** = Secure way to prove identity without storing session data

**Analogy:**
- **Session Cookies** = Club membership card (stored at club)
- **JWT** = Signed ticket (you carry it, club verifies signature)

### JWT Structure

**Three parts separated by dots:**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjMifQ.signature
   └─ Header ─────────────┘ └─ Payload ─────┘ └─ Signature ─┘
```

**1. Header** (algorithm and type):
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**2. Payload** (user data):
```json
{
  "userId": "123",
  "username": "johndoe",
  "exp": 1735689600,  // Expiration time
  "jti": "token-id-abc"  // Token ID (for revocation)
}
```

**3. Signature** (verification):
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### How JWT Works

**1. User Logs In:**
```
User → Auth Server: "Username: john, Password: secret123"
Auth Server → User: "Here's your JWT token"
```

**2. User Sends Token:**
```
User → Chat Server: "I want to send a message, here's my token"
```

**3. Server Verifies Token:**
```javascript
// jwtUtils.js
async function verifyJWTToken(token) {
  // 1. Verify signature
  const decoded = jwt.verify(token, JWT_SECRET);
  
  // 2. Check expiration
  if (decoded.exp < Date.now() / 1000) {
    throw new Error('Token expired');
  }
  
  // 3. Check if revoked
  const isRevoked = await checkJTIRevoked(decoded.jti);
  if (isRevoked) {
    throw new Error('Token revoked');
  }
  
  // 4. Get user details
  const user = await getUserDetails(decoded.userId);
  
  return user;
}
```

**4. Server Uses Token:**
```javascript
// Token verified, user is authenticated
const user = await verifyJWTToken(token);
// user = { id: '123', username: 'johndoe', email: 'john@example.com' }
```

### Why JWT for Chat Service?

**Benefits:**
- ✅ No session storage needed (stateless)
- ✅ Works across multiple servers (microservices)
- ✅ Can include user data (no database lookup needed)
- ✅ Secure (signature prevents tampering)

**In Chat Service:**
- User authenticates once with JWT token
- Server verifies token on connection
- User remains authenticated for connection duration
- Token can be revoked (stored in database)

---

## SQL Injection Prevention

### What is SQL Injection?

**SQL Injection** = Attacker tricks server into executing malicious SQL

**Vulnerable Code (DON'T DO THIS):**
```javascript
// ❌ BAD: Direct string concatenation
const userId = req.body.userId; // From user input
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// If userId = "'; DROP TABLE users; --"
// Query becomes:
// SELECT * FROM users WHERE id = ''; DROP TABLE users; --'
// 💥 Database deleted!
```

### Solution: Parameterized Queries

**Safe Code (DO THIS):**
```javascript
// ✅ GOOD: Parameterized query
const userId = req.body.userId; // From user input
const query = `SELECT * FROM users WHERE id = $1`;
const result = await dbPool.query(query, [userId]);

// userId is treated as DATA, not code
// Even if userId = "'; DROP TABLE users; --"
// Database knows it's just a string value
```

### How Parameterized Queries Work

**1. Query Template:**
```sql
SELECT * FROM users WHERE id = $1
                      -- ^^ Placeholder
```

**2. Parameters:**
```javascript
['123']  // Array of values
```

**3. Database Handles Safely:**
- Escapes special characters
- Treats as data, not code
- Prevents SQL injection

### Examples from Chat Service

**Safe Query:**
```javascript
// messageOperations.js
async function getUserDetails(userId, dbPool) {
  const result = await dbPool.query(
    'SELECT * FROM users WHERE user_id = $1',
    [userId]  // ✅ Parameterized
  );
  return result.rows[0];
}
```

**Safe Query with Multiple Parameters:**
```javascript
// messageOperations.js
async function saveMessage(messageId, conversationId, senderId, content, dbPool) {
  await dbPool.query(
    `INSERT INTO private_messages 
     (message_id, conversation_id, sender_id, content) 
     VALUES ($1, $2, $3, $4)`,
    [messageId, conversationId, senderId, content]  // ✅ All parameterized
  );
}
```

**Safe Query with LIKE (Search):**
```javascript
// Safe search query
const searchTerm = '%' + userInput + '%';
await dbPool.query(
  'SELECT * FROM users WHERE username LIKE $1',
  [searchTerm]  // ✅ Still parameterized
);
```

### Always Use Parameterized Queries

**Rule:** NEVER put user input directly in SQL strings!

**❌ Bad:**
```javascript
const query = `SELECT * FROM users WHERE username = '${username}'`;
```

**✅ Good:**
```javascript
const query = `SELECT * FROM users WHERE username = $1`;
await dbPool.query(query, [username]);
```

---

## CORS (Cross-Origin Resource Sharing)

### What is CORS?

**CORS** = Browser security feature that controls which websites can make requests to your server

**The Problem:**
```
Website A (evil.com) → Tries to make request to → Website B (chat.example.com)
Browser: "Wait, is this allowed? Let me check CORS headers..."
```

### Same-Origin Policy

**Same Origin:**
- Same protocol (http/https)
- Same domain (example.com)
- Same port (3001)

**Different Origins (Need CORS):**
- Different domain: `evil.com` → `chat.example.com`
- Different port: `localhost:3000` → `localhost:3001`
- Different protocol: `http://` → `https://`

### CORS Headers

**Server sends CORS headers to tell browser:**
- Which origins are allowed
- Which methods are allowed (GET, POST, etc.)
- Which headers are allowed

**Example from Chat Service:**
```javascript
// httpEndpoints.js
function setupHttpEndpoints(server) {
  server.on('request', (req, res) => {
    // Set CORS headers
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Requested-With');
    
    // Handle preflight request
    if (req.method === 'OPTIONS') {
      res.writeHead(200);
      res.end();
      return;
    }
    
    // Handle actual request
    // ...
  });
}
```

### CORS Headers Explained

**1. Access-Control-Allow-Origin:**
```javascript
// Allow all origins (less secure)
res.setHeader('Access-Control-Allow-Origin', '*');

// Allow specific origin (more secure)
res.setHeader('Access-Control-Allow-Origin', 'https://app.example.com');
```

**2. Access-Control-Allow-Methods:**
```javascript
// Which HTTP methods are allowed
res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
```

**3. Access-Control-Allow-Headers:**
```javascript
// Which headers client can send
res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
```

**4. Preflight Request (OPTIONS):**
```javascript
// Browser sends OPTIONS request before actual request
if (req.method === 'OPTIONS') {
  // Just return CORS headers, no data needed
  res.writeHead(200);
  res.end();
}
```

### CORS Flow

**1. Browser sends preflight request:**
```
OPTIONS /api/messages HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
```

**2. Server responds with CORS headers:**
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type
```

**3. Browser sends actual request:**
```
POST /api/messages HTTP/1.1
Origin: https://app.example.com
Content-Type: application/json
```

---

## Input Validation

### Why Validate Input?

**Invalid input can cause:**
- Security vulnerabilities
- Application crashes
- Database errors
- Unexpected behavior

### Validation Examples

**1. Check Type:**
```javascript
// inputValidator.js
static validateMessage(message) {
  if (typeof message.content !== 'string') {
    return { isValid: false, error: 'Content must be a string' };
  }
  
  if (typeof message.recipientId !== 'string' && typeof message.recipientId !== 'number') {
    return { isValid: false, error: 'Recipient ID must be string or number' };
  }
  
  return { isValid: true };
}
```

**2. Check Required Fields:**
```javascript
if (!message.content) {
  return { isValid: false, error: 'Content is required' };
}
```

**3. Check Length:**
```javascript
if (message.content.length > 10000) {
  return { isValid: false, error: 'Message too long (max 10000 characters)' };
}
```

**4. Check Format:**
```javascript
// Validate user ID format
const userIdPattern = /^[a-zA-Z0-9_-]+$/;
if (!userIdPattern.test(userId)) {
  return { isValid: false, error: 'Invalid user ID format' };
}
```

### Sanitization

**Remove dangerous content:**
```javascript
// Remove HTML tags to prevent XSS
function sanitizeContent(content) {
  return content.replace(/<[^>]*>/g, ''); // Remove HTML tags
}

// Escape special characters
function escapeSQL(content) {
  // Not needed if using parameterized queries!
  // But good to know
  return content.replace(/'/g, "''");
}
```

---

## Rate Limiting

### What is Rate Limiting?

**Rate Limiting** = Limiting how many requests a user can make

**Prevents:**
- Spam/abuse
- DDoS attacks
- Resource exhaustion

### Rate Limiting in Chat Service

**1. Per-IP Rate Limiting:**
```javascript
// connectionHandler.js
const rateLimiter = new RateLimiter();

// Check rate limit on connection
if (!rateLimiter.checkIPRateLimit(clientIp)) {
  ws.close(1008, 'Rate limit exceeded');
  return;
}
```

**2. Per-User Rate Limiting:**
```javascript
// Limit messages per minute
const MESSAGE_RATE_LIMIT = 240; // 240 messages per minute

function checkRateLimit(userId, messageRateLimit) {
  const now = Date.now();
  const userKey = `rate_limit:${userId}`;
  const messages = messageRateLimit.get(userKey) || [];
  
  // Remove messages older than 1 minute
  const recentMessages = messages.filter(time => now - time < 60000);
  
  if (recentMessages.length >= MESSAGE_RATE_LIMIT) {
    return false; // Rate limit exceeded
  }
  
  // Add current message
  recentMessages.push(now);
  messageRateLimit.set(userKey, recentMessages);
  
  return true;
}
```

**3. Connection Limit Per IP:**
```javascript
// Limit connections per IP
const MAX_CONNECTIONS_PER_IP = 10;

function checkConnectionLimit(ip, ipConnections) {
  const currentConnections = ipConnections.get(ip) || 0;
  if (currentConnections >= MAX_CONNECTIONS_PER_IP) {
    return false; // Too many connections
  }
  ipConnections.set(ip, currentConnections + 1);
  return true;
}
```

---

## Encryption

### Message Encryption

**Encrypting messages before storing:**
```javascript
// encryption.js
function encryptMessage(content) {
  const algorithm = 'aes-256-cbc';
  const key = crypto.scryptSync(ENCRYPTION_KEY, 'salt', 32);
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(algorithm, key, iv);
  
  let encrypted = cipher.update(content, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  return {
    encrypted: encrypted,
    iv: iv.toString('hex')
  };
}
```

**Storing encrypted messages:**
```javascript
const encryptionResult = encryptMessage(message.content);
await dbPool.query(
  `INSERT INTO private_messages 
   (content, encrypted_content, encryption_iv, is_encrypted) 
   VALUES ($1, $2, $3, $4)`,
  [message.content, encryptionResult.encrypted, encryptionResult.iv, true]
);
```

---

## Connection Security

### TLS/SSL (HTTPS/WSS)

**Encrypted connections:**
- **HTTPS** = HTTP over TLS (secure)
- **WSS** = WebSocket over TLS (secure)

**Setup in Chat Service:**
```javascript
// server.js
if (TLS_KEY && TLS_CERT) {
  const tlsOptions = {
    key: fs.readFileSync(TLS_KEY),
    cert: fs.readFileSync(TLS_CERT)
  };
  
  server = https.createServer(tlsOptions);
  // Now connections are encrypted!
}
```

**Why use TLS?**
- ✅ Encrypts data in transit
- ✅ Prevents man-in-the-middle attacks
- ✅ Required for production

---

## Key Takeaways

1. **Always use parameterized queries** - Prevents SQL injection
2. **Validate all input** - Prevents invalid data
3. **Use JWT for authentication** - Secure, stateless
4. **Implement rate limiting** - Prevents abuse
5. **Set proper CORS headers** - Controls access
6. **Encrypt sensitive data** - Protects privacy
7. **Use TLS in production** - Encrypts connections

---

## Next Steps

- Read [WebSocket Guide](./WEBSOCKET_GUIDE.md) for connection details
- Read [Database Guide](./DATABASE_SQL_GUIDE.md) for SQL details
- Read [Endpoints Guide](./ENDPOINTS_GUIDE.md) for API security

---

**Questions?** Check the [Main Technical Guide](./TECH_GUIDE_MAIN.md) for more resources!

