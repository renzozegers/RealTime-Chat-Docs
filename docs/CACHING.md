# Caching Strategy

## Overview

The chat server uses a multi-layer caching strategy to optimize performance and reduce database load. Caching is implemented at both the in-memory level (within each worker process) and optionally at the Redis level (shared across workers).

## Caching Layers

### 1. In-Memory Caching


Each worker process maintains in-memory Maps for fast access:

- **clients**: Active WebSocket connections
- **userConnections**: User ID to client ID mapping
- **conversations**: Active conversation state (last 50-100 messages)
- **groups**: Active group state (last 50-100 messages)
- **eventQueues**: In-memory event queue (userId -> Array of events)

**Characteristics:**
- Fast access (no network latency)
- Limited to current worker process
- Lost on server restart
- Memory-limited (last 50-100 messages per conversation)

**Memory Management:**
- Conversations/groups removed after 30 minutes of inactivity
- Message arrays limited to last 50-100 messages
- Periodic cleanup every 30 seconds
- Memory warnings at 400MB, critical at 450MB (for 512MB limit)

### 2. Redis Caching (Optional)


Redis provides shared caching across worker processes:

- **Session data**: `session:{clientId}` (TTL: 1 hour)
- **Message cache**: `message:{messageId}` (TTL: 1 hour)
- **Conversation cache**: `conversation:{conversationId}` (last 100 messages)

**Characteristics:**
- Shared across workers
- Persistent (survives worker restarts)
- Network latency (~1-5ms)
- Configurable TTL

**Fallback:**
- If Redis unavailable, system continues in database-only mode
- In development, uses Redis Mock for faster iteration

## Cache Operations

### Message Caching


**On message save:**
```javascript
if (isRedisConnected()) {
  const redisClient = getRedisClient();
  await redisClient.setEx(`message:${message.id}`, 3600, JSON.stringify(message));
  await redisClient.lPush(`conversation:${message.conversationId}`, JSON.stringify(message));
  await redisClient.lTrim(`conversation:${message.conversationId}`, 0, 99);
}
```

**On message load:**
```javascript
if (offset === 0 && userId == null && isRedisConnected()) {
  const cachedMessages = await redisClient.lRange(`conversation:${conversationId}`, 0, limit - 1);
  if (cachedMessages && cachedMessages.length > 0) {
    return cachedMessages.map(msg => JSON.parse(msg));
  }
}
```

**Cache strategy:**
- Cache only first page (offset === 0)
- Cache last 100 messages per conversation
- TTL: 1 hour
- Cache failure doesn't affect message delivery

### Session Caching


**On connection:**
```javascript
if (isRedisConnected()) {
  await redisClient.setEx(`session:${clientId}`, 3600, JSON.stringify({
    clientIp,
    lastActivity: Date.now(),
    connected: true
  }));
}
```

**On authentication:**
```javascript
if (isRedisConnected()) {
  await redisClient.setEx(`user_session:${user.userId}`, 3600, clientId);
}
```

**Cache strategy:**
- TTL: 1 hour
- Stores client metadata and user session mapping
- Used for session recovery (if needed)

## Cache Invalidation

### Message Cache Invalidation

**On message edit:**
- Update cache with new content
- TTL remains same (1 hour)

**On message delete:**
- Remove from cache immediately
- Remove from conversation list

**On conversation load:**
- Cache miss triggers database load
- New messages added to cache

### Session Cache Invalidation

**On disconnect:**
- Remove `session:{clientId}` immediately
- Remove `user_session:{userId}` if no other connections

**On authentication:**
- Update session data with new user info
- TTL reset to 1 hour

## Cache Performance

### Hit Rates

- **Message cache**: ~60-80% hit rate (first page loads)
- **Session cache**: ~90-95% hit rate (active sessions)
- **Profile picture cache**: 85-95% hit rate (see Profile Picture Caching)

### Latency

- **In-memory access**: <1ms
- **Redis access**: ~1-5ms (depending on network)
- **Database access**: ~10-50ms (depending on load)

### Memory Usage

- **In-memory conversations**: ~50-100 messages × ~1KB = ~50-100KB per conversation
- **Redis cache**: ~100 messages × ~1KB = ~100KB per conversation
- **Session cache**: ~1KB per active session

## Profile Picture Caching


**Cache characteristics:**
- In-memory Map: `{type, id} -> {hasPicture, default, url, meta}`
- Max size: 1000 entries (configurable via `PFP_CACHE_MAX_SIZE`)
- TTL: 5 minutes (configurable via `PFP_CACHE_DURATION_MS`)
- Memory usage: ~100-150 bytes per entry = ~100-150 KB total

**Cache operations:**
- Check cache before database query
- Cache miss triggers HTTP request to main backend
- Cache hit returns immediately (no database query)

**Performance:**
- 85-95% cache hit rate
- Reduces database queries by 80-95%
- Batch API endpoint uses cache for all items

## Cache Configuration

### Redis Configuration

**Environment variables:**
- `REDIS_URL`: Redis connection URL
- `REDISCLOUD_URL`: Heroku Redis URL (auto-set)
- `REDISTOGO_URL`: RedisToGo URL (auto-set)

**Connection settings:**
- Connect timeout: 500ms
- Reconnect strategy: 2 retries max
- Lazy connect: true
- Max retries per request: 1

### In-Memory Configuration

**Environment variables:**
- `MAX_CONNECTIONS`: Maximum WebSocket connections (affects memory)
- `PFP_CACHE_MAX_SIZE`: Profile picture cache size (default: 1000)
- `PFP_CACHE_DURATION_MS`: Profile picture cache TTL (default: 5 minutes)

**Memory limits:**
- Conversations: Last 50-100 messages (configurable in code)
- Groups: Last 50-100 messages (configurable in code)
- Event queue: Last 1 day (configurable in code)

## Cache Monitoring

### Metrics

- **Cache hits/misses**: Tracked in application logs
- **Cache size**: Monitored via memory usage
- **Cache performance**: Query times logged

### Health Checks

- **Redis connection**: Checked on startup and periodically
- **Memory usage**: Warnings at 400MB, critical at 450MB
- **Cache cleanup**: Runs every 30 seconds

## Best Practices

1. **Cache warming**: Pre-load frequently accessed conversations
2. **Cache invalidation**: Invalidate on updates, not just on TTL
3. **Cache size limits**: Set appropriate limits to prevent memory issues
4. **Cache TTL**: Balance freshness with performance
5. **Fallback strategy**: Always have database fallback for cache misses

## Limitations

### Single Worker State

- In-memory cache only works within same worker process
- Users on different workers don't share in-memory cache
- Redis provides cross-worker caching

### Cache Consistency

- Redis cache may be stale (1 hour TTL)
- In-memory cache may be stale (until cleanup)
- Database is source of truth

### Memory Constraints

- Limited by available RAM (512MB for Heroku Eco)
- Cache size limits prevent memory exhaustion
- Periodic cleanup prevents memory leaks

