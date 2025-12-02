# Scaling Guide

## Overview

The chat server is designed to scale horizontally using Node.js cluster module. Each worker process handles a subset of WebSocket connections while sharing the same PostgreSQL database.

## Scaling Model

### Single Process Mode

**Mode:** Single process execution

**Characteristics:**
- Single worker process
- All connections handled by one process
- In-memory state shared across all connections
- Suitable for: Development, small deployments (< 50 concurrent users)

**Limitations:**
- Single point of failure
- Limited by single CPU core
- Memory constraints (all connections in one process)

### Cluster Mode

**Mode:** Multi-process cluster execution

**Characteristics:**
- Multiple worker processes (based on CPU cores, max 8)
- Each worker handles subset of connections
- Each worker maintains separate in-memory state
- All workers share same PostgreSQL database
- Suitable for: Production, medium to large deployments

**Worker count:**
```javascript
const numWorkers = process.env.NODE_ENV === 'production' 
  ? Math.min(numCPUs, 8) 
  : 1;
```


## Worker Process Architecture

### Process Isolation

Each worker process:
- Has its own WebSocket server instance
- Maintains its own in-memory Maps (clients, conversations, groups)
- Shares the same PostgreSQL database connection pool
- Can optionally share Redis for cross-worker caching

### Connection Distribution

Connections are distributed across workers by the load balancer:
- Round-robin distribution (default)
- Sticky sessions recommended for WebSocket connections
- Each worker handles ~(total_connections / num_workers) connections

### State Isolation

**In-memory state (per worker):**
- `clients` Map: Only connections handled by this worker
- `userConnections` Map: Only users connected to this worker
- `conversations` Map: Only conversations with active participants on this worker
- `groups` Map: Only groups with active members on this worker
- `eventQueues` Map: Only events for users connected to this worker

**Shared state:**
- PostgreSQL database: All workers read/write to same database
- Redis cache: All workers can read/write to same Redis instance

## Scaling Limitations

### Real-Time Presence

**Problem:** Users connected to different workers cannot see each other's real-time presence.

**Example:**
- User A connected to Worker 1
- User B connected to Worker 2
- User A sends message to User B
- User B receives message, but User A doesn't see User B's online status

**Solution:** Requires Redis pub/sub or message broker:
- Broadcast presence events via Redis pub/sub
- All workers subscribe to presence channel
- Workers forward presence events to connected clients

### Event Queue

**Problem:** In-memory event queue only works within same worker process.

**Example:**
- User A connected to Worker 1
- User B (offline) has events queued in Worker 2's memory
- User B reconnects to Worker 1
- Worker 1 doesn't have User B's in-memory events

**Solution:** Database event queue ensures cross-worker delivery:
- All events persisted to database
- Workers load events from database on reconnection
- In-memory queue provides fast delivery within same worker

### Message Delivery

**Problem:** Messages sent to users on different workers require database lookup.

**Example:**
- User A (Worker 1) sends message to User B (Worker 2)
- Worker 1 doesn't have User B's clientId in its `userConnections` Map
- Worker 1 queues event in database
- Worker 2 loads event from database and delivers to User B

**Solution:** Database event queue handles cross-worker delivery automatically.

## Horizontal Scaling Strategies

### 1. Load Balancer Configuration

**Requirements:**
- WebSocket support (wss:// for production)
- Sticky sessions (recommended)
- Health check endpoints (health, readiness, liveness)

**Configuration:**
```nginx
upstream chat_backend {
    ip_hash;  # Sticky sessions
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 443 ssl;
    server_name chat.example.com;
    
    location / {
        proxy_pass http://chat_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 2. Redis Pub/Sub for Presence

**Implementation:**
- Workers publish presence events to Redis channels
- Workers subscribe to presence channels
- Workers forward presence events to connected clients

**Example:**
```javascript
// Publish presence event
await redisClient.publish('presence', JSON.stringify({
  type: 'user_online',
  userId: userId,
  timestamp: Date.now()
}));

// Subscribe to presence events
redisClient.subscribe('presence', (message) => {
  const event = JSON.parse(message);
  // Forward to connected clients
  broadcastToAll(event, null, clients, metrics);
});
```

### 3. Message Broker for Cross-Worker Communication

**Options:**
- Redis pub/sub (lightweight)
- RabbitMQ (feature-rich)
- Apache Kafka (high throughput)

**Use cases:**
- Real-time presence synchronization
- Cross-worker message delivery
- Event broadcasting

## Database Scaling

### Connection Pool Management

**Total app limit:** 40 connections
- Chat server: 10 connections (`DB_MAX_CONNECTIONS=10`)
- Main backend: 30 connections (configure HikariCP)

**Per worker:**
- Workers share the same connection pool
- Pool size: 10 connections total (not per worker)
- Connection acquisition timeout: 2 seconds

**Configuration:**
```javascript
const dbPool = new Pool({
  max: DB_MAX_CONNECTIONS,  // 10
  min: DB_MIN_CONNECTIONS,  // 2
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  statement_timeout: 30000
});
```

### Query Optimization

- **CTE-based queries**: Pre-compute usernames once
- **Parallel queries**: Run message loading and COUNT in parallel
- **Indexes**: Optimized indexes on frequently queried columns
- **Connection pooling**: Reuse database connections

## Memory Scaling

### Per-Worker Memory

**Memory usage per worker:**
- Base: ~50MB (Node.js runtime)
- Per connection: ~1KB (client metadata)
- Per conversation: ~50-100KB (last 50-100 messages)
- Per group: ~50-100KB (last 50-100 messages)

**Example:**
- 10 connections: ~60MB
- 50 connections: ~100MB
- 100 connections: ~150MB

### Memory Limits

**Heroku Eco (512MB):**
- Single worker: ~150MB (safe)
- Multiple workers: ~100MB per worker (4-5 workers max)

**Larger VPS (2GB):**
- Single worker: ~200MB (safe)
- Multiple workers: ~150MB per worker (10-12 workers max)

## Monitoring and Metrics

### Health Checks

- **Health endpoint**: Server health, metrics, memory usage
- **Readiness endpoint**: Readiness probe (database connection)
- **Liveness endpoint**: Liveness probe (server running)

### Metrics

- **Connections**: `wss.clients.size` per worker
- **Messages**: `metrics.messagesSent`, `metrics.messagesReceived`
- **Memory**: `process.memoryUsage()` per worker
- **Database**: Connection pool status

### Logging

- Worker process ID in logs
- Connection distribution across workers
- Memory usage per worker
- Database connection pool status

## Best Practices

1. **Start with single worker**: Scale up only when needed
2. **Monitor memory usage**: Set alerts at 80% of limit
3. **Use Redis for caching**: Reduces database load
4. **Implement sticky sessions**: Better WebSocket connection handling
5. **Monitor connection distribution**: Ensure even distribution across workers
6. **Set appropriate limits**: MAX_CONNECTIONS, DB_MAX_CONNECTIONS
7. **Use health checks**: For load balancer and monitoring

## Deployment Considerations

### Platform Requirements

- **WebSocket support**: Required (wss:// for production)
- **Process management**: PM2, systemd, or platform-specific
- **Load balancing**: Platform load balancer or external (nginx, HAProxy)
- **Health checks**: Platform health check endpoints

### Environment Variables

- `NODE_ENV=production`: Enables cluster mode
- `MAX_CONNECTIONS`: Per-worker connection limit
- `DB_MAX_CONNECTIONS`: Total database connection pool size
- `REDIS_URL`: Optional Redis connection for caching

### Scaling Triggers

- **CPU usage**: > 70% average
- **Memory usage**: > 80% of limit
- **Connection count**: > 80% of MAX_CONNECTIONS
- **Database connections**: > 80% of DB_MAX_CONNECTIONS

