# Fault Tolerance and Reliability

## Overview

This document describes how the chat server handles failures, crashes, and partial system outages. The system is designed to maintain message delivery guarantees and graceful degradation under failure conditions.

## Failure Scenarios

### 1. Worker Process Crash

**Scenario:** A single worker process crashes (out of memory, uncaught exception, etc.)

**Impact:**

- **Connections on crashed worker**: All WebSocket connections on that worker are lost
- **In-memory state**: All in-memory Maps (clients, conversations, eventQueues) are lost
- **Other workers**: Continue operating normally
- **Database**: Unaffected (all messages persisted)

**Recovery Process:**

1. **Cluster module detects crash**: Master process receives `exit` event
2. **Automatic restart**: Master process spawns new worker
3. **New worker initializes**: Creates new WebSocket server, initializes Maps
4. **Clients reconnect**: Load balancer routes reconnection attempts to new worker
5. **Event delivery**: New worker loads queued events from database on reconnection

**Code Example:**

```javascript
// Cluster master process
cluster.on('exit', (worker, code, signal) => {
  console.log(`Worker ${worker.process.pid} died (${signal || code})`);
  // Spawn new worker to replace crashed one
  cluster.fork();
});
```

**Message Guarantee:**

- **Messages saved to database**: Delivered when user reconnects
- **Messages in memory only**: Lost (but rare, as messages are saved immediately)
- **In-flight messages**: May be lost if crash occurs during processing

**Mitigation:**

- **Immediate database persistence**: All messages saved to database before delivery
- **Transactional saves**: Database transactions ensure atomicity
- **Event queue in database**: Offline events persisted to database

### 2. Database Connection Failure

**Scenario:** Database becomes unavailable (network issue, database server crash)

**Impact:**

- **Message sending**: Fails (cannot save to database)
- **Message loading**: Fails (cannot load from database)
- **WebSocket connections**: Remain active
- **In-memory state**: Unaffected

**Recovery Process:**

1. **Connection pool detects failure**: `pg.Pool` throws connection error
2. **Error handling**: Message handler catches error, returns error to client
3. **Retry logic**: Client can retry message send
4. **Connection retry**: Pool automatically retries connection (with backoff)
5. **Service restoration**: When database recovers, operations resume

**Error Response:**

```javascript
// Message handler error handling
try {
  await messageOperations.savePrivateMessage(message);
} catch (error) {
  if (error.code === 'ECONNREFUSED' || error.code === 'ETIMEDOUT') {
    // Database unavailable
    ws.send(JSON.stringify({
      type: 'error',
      message: 'Service temporarily unavailable',
      retryAfter: 30
    }));
  }
}
```

**Message Guarantee:**

- **Messages not saved**: Returned to client with error
- **Client responsibility**: Client should retry failed messages
- **No message loss**: Messages not saved until database is available

### 3. Redis Failure (Optional Component)

**Scenario:** Redis becomes unavailable (if Redis is used for caching)

**Impact:**

- **Caching**: Cache misses, all queries go to database
- **Performance**: Slightly slower (database queries instead of cache)
- **Functionality**: System continues operating (Redis is optional)

**Recovery Process:**

1. **Redis client detects failure**: Connection error
2. **Fallback to database**: All operations use database directly
3. **Graceful degradation**: System operates without caching
4. **Automatic reconnection**: Redis client reconnects when available
5. **Cache rebuild**: Cache repopulated as data is accessed

**Code Example:**

```javascript
// Redis with fallback
async function getCachedMessage(messageId) {
  try {
    const cached = await redisClient.get(`message:${messageId}`);
    if (cached) return JSON.parse(cached);
  } catch (error) {
    // Redis unavailable, fallback to database
    console.warn('Redis unavailable, using database');
  }
  // Fallback to database
  return await db.query('SELECT * FROM messages WHERE id = $1', [messageId]);
}
```

### 4. Network Partition

**Scenario:** Network connectivity issues between server and clients

**Impact:**

- **WebSocket connections**: May timeout or disconnect
- **Message delivery**: Fails for affected connections
- **Reconnection**: Clients attempt to reconnect

**Recovery Process:**

1. **Connection timeout**: WebSocket timeout (60 seconds) detects disconnection
2. **Grace period**: 60-second grace period prevents false offline status
3. **Event queuing**: Messages queued in database for offline users
4. **Reconnection**: Client reconnects when network restored
5. **Event delivery**: Queued events delivered on reconnection

**Message Guarantee:**

- **Messages during partition**: Queued in database
- **Delivery on reconnection**: All queued events delivered
- **No message loss**: Database persistence ensures delivery

### 5. Partial System Failure

**Scenario:** Some workers crash, others continue operating

**Impact:**

- **Affected workers**: Connections lost, in-memory state lost
- **Healthy workers**: Continue operating normally
- **Load balancer**: Routes new connections to healthy workers
- **Database**: Unaffected

**Recovery Process:**

1. **Crashed workers restart**: Cluster module spawns new workers
2. **Load redistribution**: Load balancer distributes connections to healthy workers
3. **Client reconnection**: Clients reconnect to available workers
4. **Event delivery**: Events loaded from database on reconnection

**Message Guarantee:**

- **Messages on healthy workers**: Delivered normally
- **Messages on crashed workers**: Queued in database, delivered on reconnection
- **No message loss**: Database persistence ensures delivery

## Message Delivery Guarantees

### At-Least-Once Delivery

The system provides at-least-once delivery guarantees:

- **Messages saved to database**: Guaranteed delivery
- **Event queue in database**: Ensures offline delivery
- **Reconnection delivery**: All queued events delivered on reconnect

**Potential for duplicates:**

- **Network retries**: Client may retry message send
- **Reconnection delivery**: Events may be delivered multiple times if reconnection occurs during delivery

**Mitigation:**

- **Idempotent operations**: Message handlers check for duplicate message IDs
- **Event deduplication**: Event queue marks events as delivered to prevent duplicates

### Message Persistence

**Immediate Persistence:**

All messages are saved to database immediately upon receipt:

```javascript
// Message handler
async function handleSendMessage(message) {
  // 1. Save to database (transactional)
  await db.query('BEGIN');
  await db.query('INSERT INTO messages ...');
  await db.query('UPDATE conversations ...');
  await db.query('COMMIT');
  
  // 2. Then deliver to recipient
  deliverToRecipient(message);
}
```

**Transaction Guarantees:**

- **Atomicity**: Message and conversation update in single transaction
- **Durability**: Database ensures data survives crashes
- **Consistency**: Database constraints ensure data integrity

### Offline Message Delivery

**Event Queue System:**

1. **Message sent to offline user**: Saved to database event queue
2. **User reconnects**: Event queue loaded from database
3. **Events delivered**: All queued events delivered in chronological order
4. **Events marked delivered**: Prevents duplicate delivery

**Code Example:**

```javascript
// On reconnection
async function deliverQueuedEvents(userId) {
  // Load undelivered events from database
  const events = await db.query(
    'SELECT * FROM event_queue WHERE user_id = $1 AND delivered = false ORDER BY created_at',
    [userId]
  );
  
  // Deliver events in order
  for (const event of events) {
    await sendEventToClient(event);
    // Mark as delivered
    await db.query('UPDATE event_queue SET delivered = true WHERE id = $1', [event.id]);
  }
}
```

## Retry Mechanisms

### Database Operation Retries

**Connection Retry:**

The database connection pool automatically retries failed connections:

```javascript
const pool = new Pool({
  max: 10,
  connectionTimeoutMillis: 2000,
  // Automatic retry on connection failure
});
```

**Query Retry (Manual):**

For critical operations, implement retry logic:

```javascript
async function saveMessageWithRetry(message, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await messageOperations.savePrivateMessage(message);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      // Exponential backoff
      await new Promise(resolve => setTimeout(resolve, Math.pow(2, i) * 100));
    }
  }
}
```

### Client-Side Retries

**Message Send Retry:**

Clients should implement retry logic for failed message sends:

```javascript
// Client-side retry
async function sendMessageWithRetry(message, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await ws.send(JSON.stringify(message));
      if (response.type === 'error' && response.retryAfter) {
        await new Promise(resolve => setTimeout(resolve, response.retryAfter * 1000));
        continue;
      }
      return response;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, Math.pow(2, i) * 100));
    }
  }
}
```

## Graceful Degradation

### Service Levels

**Full Service:**

- All components operational (database, Redis optional, all workers)
- Normal performance and functionality

**Degraded Service (Redis unavailable):**

- Caching disabled, all queries go to database
- Slightly slower performance
- Full functionality maintained

**Degraded Service (Some workers crashed):**

- Reduced capacity (fewer workers)
- Load balancer routes to healthy workers
- Full functionality maintained

**Minimal Service (Database slow):**

- Increased latency on database operations
- Timeout errors possible
- Core functionality maintained (messages queued)

### Health Checks

**Health Endpoint:**

```javascript
// GET /health
{
  "status": "healthy" | "degraded" | "unhealthy",
  "database": {
    "connected": true,
    "responseTime": 5
  },
  "redis": {
    "connected": false,  // Optional, system still healthy
    "responseTime": null
  },
  "workers": {
    "total": 4,
    "healthy": 3,
    "crashed": 1
  }
}
```

**Readiness Probe:**

```javascript
// GET /ready
{
  "ready": true,  // false if database unavailable
  "checks": {
    "database": "healthy",
    "websocket": "healthy"
  }
}
```

## Monitoring and Alerting

### Key Metrics

**1. Worker Health:**

- Worker process uptime
- Worker crash frequency
- Worker restart count

**2. Database Health:**

- Connection pool utilization
- Query response times
- Connection failures

**3. Message Delivery:**

- Messages sent successfully
- Messages queued (offline users)
- Messages failed
- Delivery latency

**4. System Health:**

- Event loop lag
- Memory usage
- CPU usage
- Connection count

### Alerting Thresholds

**Critical Alerts:**

- Database unavailable: Immediate alert
- All workers crashed: Immediate alert
- High message failure rate (> 5%): Alert within 1 minute

**Warning Alerts:**

- Single worker crash: Alert within 5 minutes
- High event loop lag (> 50ms): Alert within 5 minutes
- High memory usage (> 80%): Alert within 10 minutes

## Best Practices

**1. Immediate Persistence:**

Always save critical data to database before processing:

```javascript
// Good: Save first, then process
await db.save(message);
await processMessage(message);

// Bad: Process first, save later (risk of data loss)
await processMessage(message);
await db.save(message);  // If crash here, data lost
```

**2. Transactional Operations:**

Use database transactions for atomic operations:

```javascript
await db.query('BEGIN');
await db.query('INSERT INTO messages ...');
await db.query('UPDATE conversations ...');
await db.query('COMMIT');
```

**3. Error Handling:**

Always handle errors gracefully:

```javascript
try {
  await criticalOperation();
} catch (error) {
  // Log error
  logger.error('Operation failed', error);
  // Return error to client
  ws.send(JSON.stringify({ type: 'error', message: 'Operation failed' }));
  // Queue for retry if appropriate
  await queueForRetry(operation);
}
```

**4. Health Monitoring:**

Implement comprehensive health checks:

```javascript
// Health check endpoint
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    database: await checkDatabase(),
    redis: await checkRedis(),  // Optional
    workers: await checkWorkers()
  };
  res.json(health);
});
```

## Summary

The chat server is designed to handle various failure scenarios gracefully. Worker process crashes are automatically recovered through cluster module restart. Database failures are handled with error responses and retry logic. The system maintains message delivery guarantees through immediate database persistence and event queue systems. All critical operations are transactional and include comprehensive error handling.


