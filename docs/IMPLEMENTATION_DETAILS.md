# Implementation Details

Complete implementation documentation covering security, connection lifecycle, and configuration for the real-time messaging system.

## Table of Contents

1. [Security Implementation](#security-implementation)
2. [Connection Lifecycle](#connection-lifecycle)
3. [Configuration Reference](#configuration-reference)
4. [Performance Tuning](#performance-tuning)

---

## Security Implementation

### JWT Validation Process

The chat server validates JWT tokens using the same `JWT_SECRET` as the main backend, ensuring microservice compatibility.

#### Step-by-Step Validation

**1. Token Signature Verification**
```javascript
// Algorithm: HS256 (HMAC SHA-256)
// Secret: Base64-decoded JWT_SECRET (matches Java backend)
jwt.verify(token, JWT_SECRET, { algorithms: ['HS256'] });
```

**Process:**
- JWT_SECRET decoded from base64 to match Java backend format
- Signature verified using HMAC SHA-256
- Returns `false` if signature invalid or token malformed

**Error Handling:**
- `TokenExpiredError`: Token past expiration time
- `JsonWebTokenError`: Invalid signature or malformed token
- `NotBeforeError`: Token not yet active (nbf claim)

**2. Extract User Information**
```javascript
const decoded = jwt.verify(token, JWT_SECRET, { algorithms: ['HS256'] });
const userId = decoded.sub;  // Subject claim contains user ID
const jti = decoded.jti;     // JWT ID for revocation checking
```

**Token Structure:**
```json
{
  "sub": "123",              // User ID (subject)
  "jti": "uuid-token-id",    // JWT ID for revocation
  "exp": 1705316400,         // Expiration timestamp
  "iat": 1705312800          // Issued at timestamp
}
```

**3. Check Token Revocation**
```sql
SELECT 1 FROM jwt_revocation WHERE jti = $1 LIMIT 1
```

**Implementation:**
- Queries `jwt_revocation` table for JTI
- If JTI found → token revoked → authentication fails
- If JTI not found → token valid → continue
- Handles missing table gracefully (assumes not revoked)

**4. Fetch User Details**
```sql
SELECT ua.user_id, ua.username, ua.email, upi.first_name, upi.last_name
FROM users_auth ua
LEFT JOIN user_profile_info upi ON ua.user_id = upi.user_id
WHERE ua.user_id = $1
```

**Fallback Strategy:**
1. Try `users_auth` + `user_profile_info` (main backend tables)
2. If not found, try `chat_users` table (fallback)
3. If still not found, return null (authentication fails)
4. Handles type conversion errors (string to integer)

**5. Connection Pool Monitoring**
```javascript
// Check pool status before query
if (poolStatus.waiting > 0 || poolStatus.idle === 0) {
  console.warn('[JWT] Database connection pool status:', poolStatus);
}

// Query timeout: 5 seconds
const result = await Promise.race([
  dbPool.query(query, [userIdInt]),
  new Promise((_, reject) => 
    setTimeout(() => reject(new Error('Database query timeout')), 5000)
  )
]);
```

**Error Responses:**
```json
{
  "type": "auth_error",
  "message": "Token validation failed",
  "details": "Session expired. Please log in again."
}
```

**Complete Validation Flow:**
```
1. Client sends authenticate message with JWT token
   ↓
2. Server verifies signature (HS256)
   ↓ (if invalid → auth_error)
3. Server extracts userId and jti from token
   ↓ (if extraction fails → auth_error)
4. Server checks JTI in revocation table
   ↓ (if revoked → auth_error)
5. Server fetches user details from database
   ↓ (if user not found → auth_error)
6. Server establishes session
   ↓
7. Server delivers queued events
   ↓
8. Server sends connected response
```

---

### Encryption Algorithm and Key Management

#### Message Encryption

**Algorithm:** AES-256-CBC (Advanced Encryption Standard, 256-bit key, Cipher Block Chaining)

**Implementation:**
```javascript
const algorithm = 'aes-256-cbc';
const key = crypto.scryptSync(CHAT_ENCRYPTION_KEY, 'salt', 32);
const iv = crypto.randomBytes(16);  // Random IV for each message
const cipher = crypto.createCipheriv(algorithm, key, iv);
```

**Key Derivation:**
- Base key: `CHAT_ENCRYPTION_KEY` (environment variable)
- Derived key: 32 bytes via `scryptSync` with salt
- Salt: Static string `'salt'` (can be made configurable)

**Encryption Process:**
1. Generate random 16-byte IV (Initialization Vector)
2. Derive 32-byte encryption key from base key
3. Encrypt message content using AES-256-CBC
4. Store encrypted content as hex string
5. Store IV as hex string separately

**Storage:**
- `encrypted_content`: TEXT (hex-encoded encrypted message)
- `encryption_iv`: VARCHAR(255) (hex-encoded IV)
- `is_encrypted`: BOOLEAN (flag indicating encryption status)

**Decryption Process:**
```javascript
const key = crypto.scryptSync(CHAT_ENCRYPTION_KEY, 'salt', 32);
const decipher = crypto.createDecipheriv(algorithm, key, Buffer.from(iv, 'hex'));
const decrypted = decipher.update(encryptedData, 'hex', 'utf8') + 
                  decipher.final('utf8');
```

**Key Management:**
- Encryption key stored in environment variable: `CHAT_ENCRYPTION_KEY`
- Default key: `'super-secure-aes-encryption-key-32'` (development only)
- Production: Must be set via environment variable
- Key length: 32 bytes (256 bits) for AES-256
- IV: Random for each message (prevents pattern analysis)

**Security Considerations:**
- Each message uses unique IV (prevents identical plaintext → identical ciphertext)
- IV stored separately (required for decryption)
- Key derivation uses scrypt (key stretching, resistant to brute force)
- Fallback to unencrypted if encryption fails (logs warning)

---

### Rate Limiting Rules and Thresholds

#### IP-Based Rate Limiting

**Class:** `RateLimiter`

**Configuration:**
- **Max requests per IP:** 3000 requests per 15 minutes (900,000ms)
- **Window:** Sliding window (last 15 minutes)
- **Storage:** In-memory Map (`ip -> [timestamp, timestamp, ...]`)

**Implementation:**
```javascript
checkIPRateLimit(ip, maxRequests = 3000, windowMs = 900000) {
  const now = Date.now();
  const windowStart = now - windowMs;
  
  const requests = this.requests.get(ip) || [];
  const recentRequests = requests.filter(time => time > windowStart);
  
  if (recentRequests.length >= maxRequests) {
    return false;  // Rate limit exceeded
  }
  
  recentRequests.push(now);
  this.requests.set(ip, recentRequests);
  return true;  // Within limit
}
```

**Cleanup:**
- Old requests removed from Map (older than 15 minutes)
- Runs during periodic cleanup (every 30 seconds)
- Prevents Map growth over time

#### Message Rate Limiting

**Per-User Limits:**
- **Default:** 240 messages per 60 seconds
- **Configurable:** `MESSAGE_RATE_LIMIT` environment variable
- **Window:** Sliding window (last 60 seconds)
- **Storage:** In-memory Map (`userId -> [timestamp, timestamp, ...]`)

**Per-Client Limits:**
- Same limit applied per clientId (supports multiple devices)
- Tracks both user-level and client-level limits
- Enforced independently (both must pass)

**Implementation:**
```javascript
checkMessageRateLimit(userId, maxMessages = 240, windowMs = 60000) {
  const now = Date.now();
  const windowStart = now - windowMs;
  
  const messages = this.messageLimits.get(userId) || [];
  const recentMessages = messages.filter(time => time > windowStart);
  
  if (recentMessages.length >= maxMessages) {
    return false;  // Rate limit exceeded
  }
  
  recentMessages.push(now);
  this.messageLimits.set(userId, recentMessages);
  return true;
}
```

**Dynamic Limits:**
- Long messages (>500 chars): Limit doubled (480 messages per minute)
- Prevents abuse with large payloads

**Error Response:**
```json
{
  "type": "error",
  "message": "Rate limit exceeded",
  "details": "You are sending messages too quickly. Please wait before sending another message.",
  "code": "RATE_LIMIT_EXCEEDED",
  "retryAfter": 30
}
```

**Rate Limit Checks:**
1. IP rate limit (connection establishment)
2. User message rate limit (before processing message)
3. Client message rate limit (before processing message)
4. All checks must pass for message to be processed

---

### Blocking Check Implementation

#### Bidirectional Blocking

**Table:** `blocked_relationships`

**Query:**
```sql
SELECT blocker_id, blocked_id 
FROM blocked_relationships 
WHERE (blocker_id = $1 AND blocked_id = $2)
   OR (blocker_id = $2 AND blocked_id = $1)
LIMIT 1
```

**Logic:**
- If A blocks B → B cannot message A
- If B blocks A → A cannot message B
- Bidirectional: Either direction blocks communication
- Checked before every message send

**Implementation:**
```javascript
async function checkIfBlocked(userId1, userId2) {
  const result = await dbPool.query(`
    SELECT blocker_id, blocked_id 
    FROM blocked_relationships 
    WHERE (blocker_id = $1 AND blocked_id = $2)
       OR (blocker_id = $2 AND blocked_id = $1)
    LIMIT 1
  `, [user1Id, user2Id]);
  
  if (result.rows.length > 0) {
    return {
      isBlocked: true,
      blockerUserId: block.blocker_id.toString(),
      blockedUserId: block.blocked_id.toString()
    };
  }
  
  return { isBlocked: false };
}
```

**Error Handling:**
- Database errors: Fail open (assumes not blocked)
- Prevents blocking check from breaking messaging
- Logs errors for monitoring

**Error Response:**
```json
{
  "type": "error",
  "message": "Message blocked",
  "details": "You cannot send messages to this user.",
  "code": "USER_BLOCKED"
}
```

---

### Following Check Implementation

#### Following Relationship Check

**Table:** `user_follows`

**Query:**
```sql
SELECT EXISTS(
  SELECT 1 FROM user_follows 
  WHERE (follower_id = $1 AND followee_id = $2)
     OR (follower_id = $2 AND followee_id = $1)
) as either_follows
```

**Logic:**
- Checks if either user follows the other (bidirectional)
- Soft requirement (not strictly enforced)
- Warning logged if neither follows, but message still allowed
- Used for social features and recommendations

**Implementation:**
```javascript
async function checkIfEitherFollows(userId1, userId2) {
  const result = await dbPool.query(`
    SELECT EXISTS(
      SELECT 1 FROM user_follows 
      WHERE (follower_id = $1 AND followee_id = $2)
         OR (follower_id = $2 AND followee_id = $1)
    ) as either_follows
  `, [user1Id, user2Id]);
  
  return result.rows[0].either_follows;
}
```

**Usage:**
- Checked before sending messages
- Logs warning if neither follows
- Does not block message sending (soft check)
- Can be used for UI indicators

---

## Connection Lifecycle

### Connection Establishment

#### WebSocket Connection Flow

**1. Connection Request**
```
Client → WebSocket Server (ws:// or wss://)
```

**2. IP Validation**
```javascript
if (!SecurityUtils.validateIP(clientIp)) {
  ws.close(1008, 'Invalid IP address');
  return;
}
```

**3. Connection Limit Check**
```javascript
const MAX_CONNECTIONS_PER_IP = 10;  // Default
if (ipConnections.get(clientIp) >= MAX_CONNECTIONS_PER_IP) {
  ws.close(1008, 'Connection limit exceeded');
  return;
}
```

**4. IP Rate Limit Check**
```javascript
if (!rateLimiter.checkIPRateLimit(clientIp)) {
  ws.close(1008, 'Rate limit exceeded');
  return;
}
```

**5. Client Record Creation**
```javascript
const clientId = crypto.randomUUID();
const clientData = {
  ws,
  clientIp,
  lastActivity: Date.now()
};
clients.set(clientId, clientData);
```

**6. Welcome Message**
```json
{
  "type": "connected",
  "clientId": "uuid",
  "message": "Please authenticate to continue",
  "requiresAuth": true,
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

---

### Connection Timeout

#### Inactivity Timeout

**Configuration:**
- **Default:** 5 minutes (300,000ms)
- **Configurable:** `CONNECTION_TIMEOUT` environment variable
- **Resets:** On any message received (including ping)

**Implementation:**
```javascript
const CONNECTION_TIMEOUT = 300000;  // 5 minutes

let connectionTimeout = setTimeout(() => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.close(1000, 'Connection timeout');
  }
}, CONNECTION_TIMEOUT);

// Reset on any message
const resetConnectionTimeout = (messageType) => {
  clearTimeout(connectionTimeout);
  connectionTimeout = setTimeout(() => {
    ws.close(1000, 'Connection timeout');
  }, CONNECTION_TIMEOUT);
};
```

**Timeout Behavior:**
- Timer starts on connection
- Resets on every message (including ping/pong)
- Closes connection if no activity for timeout period
- Graceful close (code 1000)

---

### Reconnection Logic and Backoff

#### Reconnection Process

**1. New Connection Established**
- Client creates new WebSocket connection
- Server generates new clientId
- Client must re-authenticate

**2. Authentication on Reconnect**
```javascript
// Cancel any pending offline timer
if (pendingOfflineTimers.has(normalizedUserId)) {
  clearTimeout(pendingOfflineTimers.get(normalizedUserId));
  pendingOfflineTimers.delete(normalizedUserId);
}
```

**3. Event Queue Delivery**
- In-memory queue delivered first (fast)
- Database queue loaded and delivered (persistent)
- Events sorted by timestamp
- Delivered sequentially with 10ms delay

**Backoff Strategy:**
- **Client-side:** Not implemented server-side (client handles)
- **Recommended:** Exponential backoff (1s, 2s, 4s, 8s, max 30s)
- **Server-side:** No backoff (accepts reconnections immediately)

**Reconnection Flow:**
```
1. Client detects disconnection
   ↓
2. Client waits (exponential backoff)
   ↓
3. Client creates new WebSocket connection
   ↓
4. Server accepts connection
   ↓
5. Client authenticates with JWT token
   ↓
6. Server cancels offline timer
   ↓
7. Server delivers queued events
   ↓
8. Client receives all missed events
```

---

### Grace Period Implementation

#### Offline Detection Grace Period

**Configuration:**
- **Duration:** 60 seconds (60,000ms)
- **Hardcoded:** `OFFLINE_GRACE_PERIOD_MS = 60000`
- **Purpose:** Prevents false offline status during temporary disconnections

**Implementation:**
```javascript
const OFFLINE_GRACE_PERIOD_MS = 60000;  // 60 seconds

// On connection close
if (!otherActiveClientId) {
  // Cancel existing timer if any
  const existingTimer = pendingOfflineTimers.get(normalizedUserId);
  if (existingTimer) {
    clearTimeout(existingTimer);
  }
  
  // Start grace period timer
  const offlineTimer = setTimeout(() => {
    // Double-check no connections
    let stillNoConnection = true;
    for (const [otherClientId, otherClient] of clients.entries()) {
      if (otherClient.user && 
          otherClient.user.userId.toString() === normalizedUserId &&
          otherClient.ws.readyState === WebSocket.OPEN) {
        stillNoConnection = false;
        break;
      }
    }
    
    if (stillNoConnection) {
      // Mark user offline
      userConnections.delete(normalizedUserId);
      sessions.delete(normalizedUserId);
      
      // Broadcast offline event
      broadcastToAll({
        type: 'user_offline',
        user: { userId, username, displayName },
        timestamp: new Date().toISOString()
      });
    }
    
    pendingOfflineTimers.delete(normalizedUserId);
  }, OFFLINE_GRACE_PERIOD_MS);
  
  pendingOfflineTimers.set(normalizedUserId, offlineTimer);
}
```

**Grace Period Scenarios:**
- **Network hiccup:** User reconnects within 60s → stays online
- **Browser tab switch:** User returns within 60s → stays online
- **Mobile app backgrounding:** User returns within 60s → stays online
- **Actual disconnection:** No reconnection after 60s → marked offline

**Timer Cancellation:**
- Cancelled if user reconnects during grace period
- Cancelled if user has other active connections
- Prevents duplicate offline events

---

### Offline Detection Timing

#### Detection Process

**1. Connection Close Event**
```javascript
ws.on('close', async () => {
  // Check for other active connections
  let otherActiveClientId = null;
  for (const [otherClientId, otherClient] of clients.entries()) {
    if (otherClientId !== clientId && 
        otherClient.user && 
        otherClient.user.userId.toString() === normalizedUserId &&
        otherClient.ws.readyState === WebSocket.OPEN) {
      otherActiveClientId = otherClientId;
      break;
    }
  }
```

**2. Multiple Connection Handling**
- If other connections exist → user stays online
- If no other connections → start grace period timer

**3. Grace Period Expiration**
- After 60 seconds, double-check for connections
- If still no connections → mark offline
- Broadcast `user_offline` event to all clients

**Timing Breakdown:**
- **Immediate:** Connection removed from `clients` Map
- **0-60s:** Grace period (user still considered online)
- **60s+:** User marked offline (if no reconnection)

**Offline Event:**
```json
{
  "type": "user_offline",
  "user": {
    "id": "123",
    "userId": "123",
    "username": "john_doe",
    "displayName": "John Doe"
  },
  "timestamp": "2024-01-15T10:31:00.000Z"
}
```

---

### Event Queue Delivery Mechanism

#### Dual Queue System

**1. In-Memory Queue (Fast)**
- Stored in `eventQueues` Map: `userId -> Array of events`
- Delivered immediately if user on same worker
- Lost on server restart
- Limited to current worker process

**2. Database Queue (Persistent)**
- Stored in `offline_events` table
- Survives server restarts
- Cross-worker compatible
- Delivered on reconnection

#### Event Queuing

**When Events are Queued:**
- User offline (not in `userConnections` Map)
- User on different worker (cross-worker messaging)
- WebSocket connection closed (grace period)

**Event Types Queued:**
- `reaction_added` / `reaction_removed`
- `message_edited` / `message_deleted`
- `member_added` / `member_removed`
- `member_promoted` / `member_demoted`
- `group_info_updated` / `group_deleted`

**NOT Queued:**
- Messages (stored in `chat_messages` table, loaded separately)
- Typing indicators (not persisted)

#### Queue Storage

**In-Memory:**
```javascript
if (!eventQueues.has(normalizedUserId)) {
  eventQueues.set(normalizedUserId, []);
}
eventQueues.get(normalizedUserId).push({
  event: eventData,
  timestamp: Date.now()
});
```

**Database:**
```sql
INSERT INTO event_queue (user_id, event_type, event_data, created_at)
VALUES ($1, $2, $3, NOW())
```

#### Event Delivery on Reconnection

**1. Load In-Memory Queue**
```javascript
const memoryQueuedEvents = eventQueues.get(normalizedUserId);
if (memoryQueuedEvents && memoryQueuedEvents.length > 0) {
  const sortedEvents = [...memoryQueuedEvents].sort((a, b) => 
    a.timestamp - b.timestamp
  );
  
  for (const queuedEvent of sortedEvents) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(queuedEvent.event));
      await new Promise(resolve => setTimeout(resolve, 10));  // 10ms delay
    }
  }
  
  eventQueues.delete(normalizedUserId);
}
```

**2. Load Database Queue**
```javascript
const dbQueuedEvents = await getQueuedEventsForUser(normalizedUserId, dbPool);
if (dbQueuedEvents && dbQueuedEvents.length > 0) {
  const eventIds = [];
  
  for (const queuedEvent of dbQueuedEvents) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(queuedEvent.event));
      eventIds.push(queuedEvent.id);
      await new Promise(resolve => setTimeout(resolve, 10));  // 10ms delay
    }
  }
  
  await markEventsAsDelivered(eventIds, dbPool);
}
```

**Delivery Characteristics:**
- Events sorted by timestamp (chronological order)
- Delivered sequentially (10ms delay between events)
- Maximum 1000 events per user (prevents overwhelming)
- Events marked as delivered after sending
- In-memory queue cleared after delivery

#### Event Cleanup

**Delivered Events:**
- Cleaned up after 7 days
- Runs during periodic cleanup (every 30 seconds)
- Query: `DELETE FROM event_queue WHERE delivered_at < NOW() - INTERVAL '7 days'`

**Undelivered Events:**
- Cleaned up after 30 days
- Assumes user not coming back
- Query: `DELETE FROM event_queue WHERE delivered_at IS NULL AND created_at < NOW() - INTERVAL '30 days'`

**In-Memory Queue:**
- Cleaned up every 30 seconds
- Removes events older than 1 day
- Prevents memory leaks

---

## Configuration Reference

### Environment Variables

#### Server Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3001` | Server port number |
| `NODE_ENV` | `production` | Environment (development/production) |
| `MAX_CONNECTIONS` | `50` | Maximum WebSocket connections per worker |
| `MAX_PAYLOAD_SIZE` | `1048576` | Max message size in bytes (1MB) |
| `KEEP_ALIVE_TIMEOUT` | `60000` | WebSocket keep-alive timeout (ms) |
| `CONNECTION_TIMEOUT` | `300000` | Inactivity timeout (ms, 5 minutes) |
| `MAX_CONNECTIONS_PER_IP` | `10` | Maximum connections per IP address |
| `MESSAGE_RATE_LIMIT` | `240` | Messages per 60 seconds per user |

#### Database Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | - | Full database connection string (alternative to individual vars) |
| `DB_HOST` | - | Database host |
| `DB_PORT` | `5432` | Database port |
| `DB_NAME` | - | Database name |
| `DB_USER` | - | Database username |
| `DB_PASS` | - | Database password |
| `DB_SSL` | `true` | Enable SSL (set to `false` to disable) |
| `DB_MAX_CONNECTIONS` | `10` | Maximum connection pool size |
| `DB_MIN_CONNECTIONS` | `2` | Minimum connection pool size |
| `DB_IDLE_TIMEOUT` | `30000` | Idle connection timeout (ms) |
| `DB_CONNECTION_TIMEOUT` | `2000` | Connection establishment timeout (ms) |
| `DB_STATEMENT_TIMEOUT` | `30000` | SQL statement timeout (ms) |
| `DB_QUERY_TIMEOUT` | `30000` | Query execution timeout (ms) |

#### Redis Configuration (Optional)

| Variable | Default | Description |
|----------|---------|-------------|
| `REDIS_URL` | - | Redis connection URL |
| `REDISCLOUD_URL` | - | Heroku Redis URL (auto-set) |
| `REDISTOGO_URL` | - | RedisToGo URL (auto-set) |

#### Security Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `JWT_SECRET` | - | JWT secret key (base64-encoded, shared with main backend) |
| `CHAT_ENCRYPTION_KEY` | `'super-secure-aes-encryption-key-32'` | Message encryption key (32 bytes) |

#### TLS Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `TLS_KEY` | - | Path to TLS private key file |
| `TLS_CERT` | - | Path to TLS certificate file |
| `TLS_CA` | - | Path to TLS CA certificate file |

---

### Tuning Parameters

#### Connection Pool Tuning

**Recommended Settings:**
```javascript
DB_MAX_CONNECTIONS = 10      // Chat server share
DB_MIN_CONNECTIONS = 2        // Keep 2 connections warm
DB_IDLE_TIMEOUT = 30000       // 30 seconds
DB_CONNECTION_TIMEOUT = 2000  // 2 seconds
```

**Total App Limit:**
- **Chat server:** 10 connections
- **Main backend:** 30 connections
- **Total:** 40 connections (database limit)

**Pool Monitoring:**
- Warning at 90% capacity
- Warning if queries waiting for connections
- Logged every 5 minutes

#### Memory Tuning

**In-Memory Cache Limits:**
- Conversations: Last 50-100 messages per conversation
- Groups: Last 50-100 messages per group
- Event queue: Last 1 day (in-memory)
- Profile pictures: 1000 entries max

**Cleanup Intervals:**
- Memory cleanup: Every 30 seconds
- Inactive conversations: Removed after 30 minutes
- Event queue cleanup: Every 30 seconds

#### Performance Tuning

**Query Optimization:**
- CTE for username lookups (reduces N+1 queries)
- Pagination: Default 50, max 200
- Index usage: All foreign keys indexed
- Batch operations: Up to 50 items

**Connection Tuning:**
- Keep-alive: 60 seconds
- Inactivity timeout: 5 minutes
- Max payload: 1MB
- Compression: Per-message deflate (reduced memory settings)

---

### Limits and Thresholds

#### Connection Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Max connections per worker | `MAX_CONNECTIONS` (default: 50) | Total WebSocket connections |
| Max connections per IP | `MAX_CONNECTIONS_PER_IP` (default: 10) | Connections from single IP |
| Connection timeout | `CONNECTION_TIMEOUT` (default: 300000ms) | Inactivity timeout |
| Keep-alive timeout | `KEEP_ALIVE_TIMEOUT` (default: 60000ms) | WebSocket keep-alive |

#### Rate Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Messages per user (60s) | `MESSAGE_RATE_LIMIT` (default: 240) | Per-user message limit |
| Messages per user (long) | `MESSAGE_RATE_LIMIT * 2` (default: 480) | For messages >500 chars |
| Requests per IP (15min) | 3000 | IP-based rate limit |
| Batch size | 50 | Maximum items in batch operations |

#### Message Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Max payload size | `MAX_PAYLOAD_SIZE` (default: 1MB) | Maximum message size |
| Pagination default | 50 | Default messages per page |
| Pagination maximum | 200 | Maximum messages per page |
| Edit window | 5 minutes | Time limit for editing messages |

#### Database Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Max connections | `DB_MAX_CONNECTIONS` (default: 10) | Connection pool size |
| Min connections | `DB_MIN_CONNECTIONS` (default: 2) | Minimum pool size |
| Statement timeout | `DB_STATEMENT_TIMEOUT` (default: 30000ms) | SQL statement timeout |
| Query timeout | `DB_QUERY_TIMEOUT` (default: 30000ms) | Query execution timeout |

#### Event Queue Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Events per user | 1000 | Maximum events loaded on reconnect |
| Delivery delay | 10ms | Delay between event deliveries |
| Cleanup (delivered) | 7 days | Age before cleanup |
| Cleanup (undelivered) | 30 days | Age before cleanup |

#### Group Limits

| Limit | Value | Description |
|-------|-------|-------------|
| Max members | 50 | Default maximum group members |
| Max members (configurable) | Set on creation | Can be customized per group |

---

## Performance Tuning

### Database Connection Pool

**Optimal Configuration:**
```javascript
{
  max: 10,                    // Chat server share
  min: 2,                      // Keep warm connections
  idleTimeoutMillis: 30000,    // 30 seconds
  connectionTimeoutMillis: 2000,  // 2 seconds
  statement_timeout: 30000,    // 30 seconds
  query_timeout: 30000         // 30 seconds
}
```

**Monitoring:**
- Pool status logged every 5 minutes
- Warnings at 90% capacity
- Connection errors logged

### Memory Management

**Cache Sizes:**
- Conversations: 50-100 messages (configurable in code)
- Groups: 50-100 messages (configurable in code)
- Profile pictures: 1000 entries
- Event queue: 1 day retention

**Cleanup Strategy:**
- Periodic cleanup every 30 seconds
- Inactive conversations removed after 30 minutes
- Old events cleaned up automatically

### Query Optimization

**CTE Usage:**
- Username lookups pre-computed in CTE
- Reduces query time from 2-4s to <200ms
- Avoids N+1 query problem

**Index Strategy:**
- All foreign keys indexed
- Timestamp columns indexed for ordering
- Composite indexes for common queries

---

## Security Best Practices

### JWT Secret Management

**Requirements:**
- Must match main backend `JWT_SECRET`
- Base64-encoded format
- Minimum 32 bytes (256 bits) for HS256
- Stored in environment variable (never in code)

### Encryption Key Management

**Requirements:**
- 32-byte key for AES-256
- Stored in environment variable
- Different from JWT_SECRET
- Rotated periodically in production

### Rate Limiting

**Enforcement:**
- Applied at connection level (IP)
- Applied at user level (messages)
- Applied at client level (per device)
- All checks must pass

### Input Validation

**Validation:**
- JSON parsing with error handling
- Message size limits enforced
- User ID normalization
- SQL injection prevention (parameterized queries)

---

## Monitoring and Debugging

### Connection Monitoring

**Metrics Tracked:**
- Active connections per worker
- Peak concurrent users
- Connection timeouts
- IP connection counts

### Message Monitoring

**Metrics Tracked:**
- Messages sent/received
- Average response time
- Error count
- Rate limit hits

### Database Monitoring

**Metrics Tracked:**
- Connection pool usage
- Query performance
- Pool exhaustion warnings
- Connection timeouts

### Event Queue Monitoring

**Metrics Tracked:**
- Events queued per user
- Events delivered
- Queue cleanup counts
- Undelivered event age

---

## Troubleshooting

### Common Issues

**1. Authentication Failures**
- Check JWT_SECRET matches main backend
- Verify token expiration
- Check JTI revocation table
- Verify user exists in database

**2. Connection Pool Exhaustion**
- Monitor pool usage
- Increase DB_MAX_CONNECTIONS (within app limit)
- Check for connection leaks
- Review query timeouts

**3. Rate Limit Issues**
- Check MESSAGE_RATE_LIMIT setting
- Verify per-user vs per-client limits
- Review rate limit cleanup

**4. Event Queue Issues**
- Check database queue table
- Verify event delivery on reconnect
- Monitor queue cleanup
- Check for undelivered events

**5. Memory Issues**
- Monitor RSS and heap usage
- Review cache sizes
- Check cleanup intervals
- Verify memory limits

---

## Summary

This document provides complete implementation details for:

✅ **Security:** JWT validation, encryption, rate limiting, blocking/following  
✅ **Connection Lifecycle:** Reconnection, grace period, offline detection, event queue  
✅ **Configuration:** Environment variables, tuning parameters, limits  

All implementation details are documented with code examples, configuration values, and troubleshooting guidance.

