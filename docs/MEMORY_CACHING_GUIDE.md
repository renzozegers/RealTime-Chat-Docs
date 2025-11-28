# 💾 Memory Usage & Caching Guide

Understanding how memory and caching work in this chat service. Learn about in-memory data structures, Redis caching, and memory management!

## 📖 Table of Contents

1. [What is Caching?](#what-is-caching)
2. [Why Cache Data?](#why-cache-data)
3. [In-Memory Data Structures](#in-memory-data-structures)
4. [Memory Management](#memory-management)
5. [Redis Caching](#redis-caching)
6. [Cache Invalidation](#cache-invalidation)
7. [Memory Optimization](#memory-optimization)
8. [Monitoring Memory](#monitoring-memory)

---

## What is Caching?

### Simple Analogy: Library

Think of a **library**:

**Without Cache:**
- Every time you need a book → Go to storage room
- Walk 5 minutes, find book, walk back
- Repeat for every book

**With Cache:**
- Keep popular books at desk (cache)
- Instant access for popular books
- Only go to storage for rare books

### In Technical Terms

**Cache** = Temporary storage of frequently accessed data

**Benefits:**
- ✅ Faster access (memory is faster than database)
- ✅ Less database load (fewer queries)
- ✅ Better performance (instant responses)

**Trade-offs:**
- ❌ Uses memory (limited resource)
- ❌ Can become stale (outdated data)
- ❌ Must manage invalidation

---

## Why Cache Data?

### The Problem: Database Latency

**Without Cache:**
```
User requests conversation
    ↓
Server queries database (50-200ms)
    ↓
Database responds
    ↓
Server sends to user
    ↓
Total: 50-200ms
```

**With Cache:**
```
User requests conversation
    ↓
Server checks cache (< 1ms) ✅ Found!
    ↓
Server sends to user
    ↓
Total: < 1ms (200x faster!)
```

### What We Cache in Chat Service

**1. Active Conversations:**
- Recently opened conversations
- Last 50 messages per conversation
- Participant lists

**2. Online Users:**
- List of currently online users
- User status (online/offline)
- Last seen timestamps

**3. Group Information:**
- Active groups
- Group members
- Group settings

**4. User Details:**
- Frequently accessed user profiles
- Profile pictures metadata
- User roles and permissions

---

## In-Memory Data Structures

### Maps (Key-Value Storage)

**Maps** store data with keys for fast lookup:

```javascript
// Store clients by clientId
const clients = new Map();
clients.set('client_123', {
  ws: websocket,
  userId: '456',
  lastActivity: Date.now()
});

// Fast lookup: O(1)
const client = clients.get('client_123');
```

**Example from Chat Service:**
```javascript
// connectionHandler.js
const clients = new Map(); // All WebSocket connections

clients.set(clientId, {
  ws: websocket,
  clientIp: '192.168.1.1',
  lastActivity: Date.now()
});
```

**Why Maps?**
- ✅ Fast lookups (O(1))
- ✅ Easy to add/remove
- ✅ Can use any type as key

### Sets (Unique Values)

**Sets** store unique values:

```javascript
// Store participants (unique user IDs)
const participants = new Set();
participants.add('user_123');
participants.add('user_456');
participants.add('user_123'); // Ignored (already exists)

// Fast check: O(1)
if (participants.has('user_123')) {
  console.log('User is participant');
}
```

**Example from Chat Service:**
```javascript
// messageHandlers.js
conversations.set(conversationId, {
  participants: new Set([senderId, recipientId]),
  messages: []
});
```

**Why Sets?**
- ✅ Prevents duplicates automatically
- ✅ Fast membership check (O(1))
- ✅ Perfect for participants, online users

### Arrays (Ordered Lists)

**Arrays** store ordered data:

```javascript
// Store messages in order
const messages = [];
messages.push(message1);
messages.push(message2);

// Keep only last 100 messages
if (messages.length > 100) {
  messages = messages.slice(-100); // Keep last 100
}
```

**Example from Chat Service:**
```javascript
// messageHandlers.js
const conversation = {
  participants: new Set([senderId, recipientId]),
  messages: [] // Array of messages
};

// Add message
conversation.messages.push(newMessage);

// Keep last 500 messages in memory
if (conversation.messages.length > 500) {
  conversation.messages = conversation.messages.slice(-500);
}
```

---

## Memory Management

### Tracking Active Connections

**Connection Storage:**
```javascript
// connectionHandler.js
const clients = new Map(); // All active connections

// When client connects
clients.set(clientId, {
  ws: websocket,
  userId: '123',
  lastActivity: Date.now()
});

// When client disconnects
clients.delete(clientId);
```

**Memory Usage:**
- Each connection: ~200-500 bytes
- 1000 connections: ~200-500 KB
- Manageable even for large deployments

### Caching Conversations

**In-Memory Conversations:**
```javascript
// messageHandlers.js
const conversations = new Map(); // Active conversations

// Create or get conversation
if (!conversations.has(conversationId)) {
  conversations.set(conversationId, {
    participants: new Set([senderId, recipientId]),
    messages: [] // Last 50-500 messages
  });
}

// Load messages from database
const dbMessages = await loadMessagesFromDatabase(conversationId, 50);
conversation.messages = dbMessages;
```

**Memory Usage Per Conversation:**
- Conversation metadata: ~200 bytes
- Participants Set: ~50 bytes per participant
- Messages (50 messages): ~10-50 KB
- Total per conversation: ~10-50 KB

**Total Memory:**
- 100 active conversations: ~1-5 MB
- 1000 active conversations: ~10-50 MB
- Still manageable!

### Limiting Memory Growth

**Message Limit Per Conversation:**
```javascript
// Keep only last 500 messages in memory
if (conversation.messages.length > 500) {
  conversation.messages = conversation.messages.slice(-500);
}
```

**Conversation Cleanup:**
```javascript
// Remove inactive conversations (not accessed in 1 hour)
function cleanupInactiveConversations() {
  const now = Date.now();
  const ONE_HOUR = 3600000;
  
  for (const [conversationId, conversation] of conversations) {
    if (now - conversation.lastAccessed > ONE_HOUR) {
      conversations.delete(conversationId);
    }
  }
}

// Run cleanup every hour
setInterval(cleanupInactiveConversations, 3600000);
```

---

## Redis Caching

### What is Redis?

**Redis** = In-memory database (faster than PostgreSQL)

**Benefits:**
- ✅ Very fast (in-memory, sub-millisecond)
- ✅ Shared across multiple servers
- ✅ Persists to disk (survives restarts)
- ✅ Supports expiration (auto-cleanup)

**Use Cases:**
- Session storage
- Message caching
- Rate limiting
- Online user tracking

### Redis in Chat Service

**Message Caching:**
```javascript
// messageOperations.js
async function saveMessage(message, dbPool) {
  // 1. Save to database
  await dbPool.query(/* ... */);
  
  // 2. Cache in Redis (after successful save)
  if (isRedisConnected()) {
    const redisClient = getRedisClient();
    
    // Cache message for 1 hour
    await redisClient.setEx(
      `message:${message.id}`,
      3600, // 1 hour
      JSON.stringify(message)
    );
    
    // Cache conversation messages (last 100)
    await redisClient.lPush(
      `conversation:${message.conversationId}`,
      JSON.stringify(message)
    );
    await redisClient.lTrim(`conversation:${message.conversationId}`, 0, 99);
  }
}
```

**Reading from Cache:**
```javascript
async function getMessage(messageId) {
  // 1. Try Redis cache first
  if (isRedisConnected()) {
    const redisClient = getRedisClient();
    const cached = await redisClient.get(`message:${messageId}`);
    
    if (cached) {
      return JSON.parse(cached); // ✅ Found in cache!
    }
  }
  
  // 2. Fall back to database
  const result = await dbPool.query(
    'SELECT * FROM private_messages WHERE message_id = $1',
    [messageId]
  );
  
  return result.rows[0];
}
```

### Profile Picture Caching

**Profile Picture Metadata Cache:**
```javascript
// Profile picture cache (in-memory)
const PFP_CACHE_MAX_SIZE = 1000; // Max 1000 entries
const pfpCache = new Map(); // LRU-like cache

async function getCachedProfilePicture(userId) {
  // Check cache
  const cached = pfpCache.get(`user:${userId}`);
  
  if (cached && Date.now() - cached.timestamp < 300000) {
    return cached.metadata; // ✅ Found in cache!
  }
  
  // Not in cache, fetch from database
  const metadata = await getProfilePictureFromDatabase(userId);
  
  // Add to cache
  if (pfpCache.size >= PFP_CACHE_MAX_SIZE) {
    // Remove oldest entry
    const firstKey = pfpCache.keys().next().value;
    pfpCache.delete(firstKey);
  }
  
  pfpCache.set(`user:${userId}`, {
    metadata,
    timestamp: Date.now()
  });
  
  return metadata;
}
```

**Memory Usage:**
- Per entry: ~150 bytes
- 1000 entries: ~150 KB
- Negligible memory impact!

---

## Cache Invalidation

### When to Invalidate Cache?

**Cache becomes stale when:**
1. Data is updated (message edited, user updated)
2. Data is deleted (message deleted, user removed)
3. Time expires (TTL - Time To Live)

### Invalidation Strategies

**1. Time-Based Expiration (TTL):**
```javascript
// Cache expires after 5 minutes
await redisClient.setEx('key', 300, JSON.stringify(data));
// After 300 seconds, cache is automatically deleted
```

**2. Event-Based Invalidation:**
```javascript
// When message is updated
async function updateMessage(messageId, newContent) {
  // 1. Update database
  await dbPool.query(/* ... */);
  
  // 2. Invalidate cache
  if (isRedisConnected()) {
    await redisClient.del(`message:${messageId}`);
    await redisClient.del(`conversation:${conversationId}`);
  }
  
  // 3. Update in-memory cache
  const conversation = conversations.get(conversationId);
  const message = conversation.messages.find(m => m.id === messageId);
  if (message) {
    message.content = newContent;
  }
}
```

**3. Manual Invalidation:**
```javascript
// Clear all cached conversations
async function clearConversationCache() {
  if (isRedisConnected()) {
    const keys = await redisClient.keys('conversation:*');
    for (const key of keys) {
      await redisClient.del(key);
    }
  }
  
  // Clear in-memory cache
  conversations.clear();
}
```

---

## Memory Optimization

### Limiting Cache Size

**Profile Picture Cache:**
```javascript
// Limit cache to 1000 entries
const PFP_CACHE_MAX_SIZE = 1000;

// When cache is full, remove oldest entry
if (pfpCache.size >= PFP_CACHE_MAX_SIZE) {
  const firstKey = pfpCache.keys().next().value;
  pfpCache.delete(firstKey);
}
```

**Message Cache:**
```javascript
// Keep only last 100 messages per conversation in Redis
await redisClient.lTrim(`conversation:${conversationId}`, 0, 99);
```

### Periodic Cleanup

**Cleanup Inactive Data:**
```javascript
// Clean up expired entries every minute
setInterval(() => {
  const now = Date.now();
  const FIVE_MINUTES = 300000;
  
  for (const [key, entry] of pfpCache) {
    if (now - entry.timestamp > FIVE_MINUTES) {
      pfpCache.delete(key);
    }
  }
}, 60000); // Every minute
```

### Garbage Collection

**Node.js handles garbage collection automatically:**
- Deleted Map/Set entries are garbage collected
- Unreferenced objects are freed
- Memory is reclaimed automatically

**Manual cleanup helps:**
```javascript
// When conversation ends
function endConversation(conversationId) {
  // Remove from memory
  conversations.delete(conversationId);
  
  // This allows garbage collector to free memory
}
```

---

## Monitoring Memory

### Checking Memory Usage

**Node.js Memory Info:**
```javascript
const memoryUsage = process.memoryUsage();
console.log({
  heapUsed: `${(memoryUsage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
  heapTotal: `${(memoryUsage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
  rss: `${(memoryUsage.rss / 1024 / 1024).toFixed(2)} MB`
});
```

**Cache Statistics:**
```javascript
// Count cache entries
console.log('Active connections:', clients.size);
console.log('Active conversations:', conversations.size);
console.log('Cached profile pictures:', pfpCache.size);
```

### Health Endpoint

**Memory stats in health check:**
```javascript
// httpEndpoints.js
server.on('request', (req, res) => {
  if (url.pathname === '/health') {
    const health = {
      memory: process.memoryUsage(),
      cache: {
        connections: clients.size,
        conversations: conversations.size,
        profilePictures: pfpCache.size
      },
      redis: {
        connected: isRedisConnected()
      }
    };
    
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(health));
  }
});
```

---

## Key Takeaways

1. **Caching improves performance**
   - Memory is 100x+ faster than database
   - Reduces database load

2. **Use appropriate data structures**
   - Maps for key-value lookups
   - Sets for unique collections
   - Arrays for ordered lists

3. **Limit cache size**
   - Prevent unbounded memory growth
   - Use LRU eviction when full

4. **Invalidate stale data**
   - Time-based expiration (TTL)
   - Event-based invalidation
   - Manual cleanup

5. **Monitor memory usage**
   - Track cache sizes
   - Check Node.js memory stats
   - Set up alerts

6. **Redis for distributed caching**
   - Shared across multiple servers
   - Persists to disk
   - Auto-expiration support

---

## Next Steps

- Read [Database & SQL Guide](./DATABASE_SQL_GUIDE.md) for database caching
- Read [Client-Server Guide](./CLIENT_SERVER_GUIDE.md) for architecture
- Check [Cache Memory Analysis](./CACHE_MEMORY_ANALYSIS.md) for detailed metrics

---

**Questions?** Check the [Main Technical Guide](./TECH_GUIDE_MAIN.md) for more resources!

