# Performance Monitoring and Metrics

## Overview

This document describes how to monitor the chat server's performance, measure latency, identify bottlenecks, and use profiling tools to optimize the system.

## Latency Measurement

### Message Latency Components

**End-to-End Latency Breakdown:**

1. **Network latency**: Client to server (varies by location, typically 10-50ms)
2. **WebSocket processing**: Message parsing and routing (~1ms)
3. **Authentication check**: JWT validation (~1ms)
4. **Database write**: Message persistence (~5-10ms)
5. **Encryption**: Message content encryption (~1ms)
6. **Delivery**: WebSocket send to recipient (~1ms)
7. **Network latency**: Server to recipient client (10-50ms)

**Total typical latency**: 30-70ms for online delivery

### Measuring Latency

**Server-Side Measurement:**

```javascript
// Message handler with latency tracking
async function handleSendMessage(message, clientId) {
  const startTime = process.hrtime.bigint();
  
  // Process message
  await processMessage(message);
  
  const endTime = process.hrtime.bigint();
  const latency = Number(endTime - startTime) / 1e6; // Convert to milliseconds
  
  metrics.recordLatency('message_processing', latency);
  
  return response;
}
```

**Client-Side Measurement:**

```javascript
// Client sends message with timestamp
const message = {
  type: 'send_message',
  content: 'Hello',
  timestamp: Date.now()  // Client timestamp
};

ws.send(JSON.stringify(message));

// Server includes processing time in response
// Client calculates round-trip time
const rtt = Date.now() - message.timestamp;
```

### Latency Metrics

**Key Latency Metrics:**

- **P50 (median)**: 50% of messages processed in this time or less
- **P95**: 95% of messages processed in this time or less
- **P99**: 99% of messages processed in this time or less
- **P99.9**: 99.9% of messages processed in this time or less (tail latency)

**Example Metrics:**

```json
{
  "latency": {
    "p50": 25,
    "p95": 80,
    "p99": 150,
    "p99.9": 500,
    "max": 1200
  }
}
```

## Highest Latency Parts

### Database Operations

**Database writes are the highest latency component:**

- **Message save**: 5-10ms average
- **Conversation update**: 2-5ms average
- **Transaction commit**: 1-2ms average
- **Total database latency**: 8-17ms per message

**Optimization:**

- Connection pooling reduces connection acquisition time
- Indexes optimize query performance
- Batch operations reduce round-trips

### Network Latency

**Client-to-server latency:**

- **Local network**: 1-5ms
- **Same region**: 10-30ms
- **Cross-region**: 50-200ms
- **International**: 100-500ms

**Optimization:**

- Deploy servers in multiple regions
- Use CDN for static assets
- WebSocket compression reduces payload size

### JSON Serialization

**JSON parsing and stringification:**

- **Small messages (< 1KB)**: < 1ms
- **Medium messages (1-10KB)**: 1-5ms
- **Large messages (> 10KB)**: 5-20ms

**Optimization:**

- Limit message size (1MB max payload)
- Use binary protocol for high-throughput scenarios
- Minimize message payload size

## Event Loop Monitoring

### Measuring Event Loop Lag

**Event Loop Lag:**

The delay between when an event is scheduled and when it's actually processed.

**Measurement:**

```javascript
// Event loop lag monitor
let lastCheck = process.hrtime.bigint();

function measureEventLoopLag() {
  const now = process.hrtime.bigint();
  const lag = Number(now - lastCheck) / 1e6; // Convert to milliseconds
  lastCheck = now;
  
  // Expected lag should be ~1ms (setImmediate scheduling)
  const actualLag = lag - 1;
  
  if (actualLag > 10) {
    console.warn(`Event loop lag: ${actualLag.toFixed(2)}ms`);
    metrics.recordEventLoopLag(actualLag);
  }
  
  setImmediate(measureEventLoopLag);
}

// Start monitoring
setImmediate(measureEventLoopLag);
```

**Healthy Event Loop:**

- **Lag < 5ms**: Healthy, no blocking operations
- **Lag 5-10ms**: Acceptable, minor blocking
- **Lag 10-50ms**: Degraded, blocking operations present
- **Lag > 50ms**: Critical, significant blocking

### Event Loop Utilization

**Utilization Calculation:**

```javascript
function calculateEventLoopUtilization() {
  const start = process.hrtime.bigint();
  
  setImmediate(() => {
    const end = process.hrtime.bigint();
    const duration = Number(end - start) / 1e6;
    const utilization = duration / 100; // Assuming 100ms measurement window
    
    metrics.recordEventLoopUtilization(utilization);
  });
}
```

**Target Utilization:**

- **< 50%**: Healthy
- **50-70%**: Acceptable
- **70-90%**: High, consider optimization
- **> 90%**: Critical, system overloaded

## Profiling Tools

### Node.js Built-in Profiler

**CPU Profiling:**

```bash
# Start server with CPU profiling
node --prof server.js

# Generate report
node --prof-process isolate-*.log > profile.txt
```

**Heap Profiling:**

```bash
# Start server with heap profiling
node --heapsnapshot-signal=SIGUSR2 server.js

# Trigger heap snapshot
kill -USR2 <pid>

# Analyze with Chrome DevTools
# chrome://inspect -> Memory -> Load snapshot
```

### Clinic.js

**Performance Profiling:**

```bash
# Install
npm install -g clinic

# Profile CPU
clinic doctor -- node server.js

# Profile event loop
clinic bubbleprof -- node server.js

# Profile memory
clinic heapprofiler -- node server.js
```

### 0x

**Flame Graph Generation:**

```bash
# Install
npm install -g 0x

# Generate flame graph
0x server.js

# Open flame graph in browser
```

### APM Tools

**Application Performance Monitoring:**

- **New Relic**: Full-stack APM
- **Datadog**: Infrastructure and APM
- **Elastic APM**: Open-source APM
- **Prometheus + Grafana**: Custom metrics and dashboards

## JSON Serialization Optimization

### Current Approach

**JSON.stringify() overhead:**

- **Small objects**: < 1ms
- **Large objects**: 5-20ms
- **Repeated serialization**: Can be optimized

### Optimization Strategies

**1. Cache Serialized Messages:**

```javascript
// Cache serialized message for repeated sends
const messageCache = new Map();

function serializeMessage(message) {
  const cacheKey = message.id;
  if (messageCache.has(cacheKey)) {
    return messageCache.get(cacheKey);
  }
  
  const serialized = JSON.stringify(message);
  messageCache.set(cacheKey, serialized);
  
  // Clear cache after 1 minute
  setTimeout(() => messageCache.delete(cacheKey), 60000);
  
  return serialized;
}
```

**2. Minimize Payload Size:**

```javascript
// Use shorter field names
const message = {
  t: 'msg',        // type
  id: '123',      // id
  c: 'Hello',     // content
  ts: 1234567890  // timestamp
};

// Instead of:
const message = {
  type: 'message',
  id: '123',
  content: 'Hello',
  timestamp: 1234567890
};
```

**3. Binary Protocol (Advanced):**

For ultra-high throughput, use binary protocol instead of JSON:

```javascript
// Binary message format
const buffer = Buffer.alloc(32);
buffer.writeUInt8(1, 0);  // Message type
buffer.writeUInt32BE(messageId, 1);  // Message ID
buffer.write(messageContent, 5);  // Content

ws.send(buffer);  // Binary send
```

## Throughput Improvement

### Current Throughput

**Estimated throughput per worker:**

- **Messages per second**: 100-500 (depending on message size)
- **Concurrent connections**: 100-500 per worker
- **Total system**: 400-4000 messages/second (with 8 workers)

### Optimization Strategies

**1. Database Write Optimization:**

```javascript
// Batch database writes
const messages = [...]; // Array of messages
await db.query('INSERT INTO messages VALUES $1', [messages]); // Single query
```

**2. Connection Pool Tuning:**

```javascript
// Increase pool size for higher throughput
const pool = new Pool({
  max: 20,  // Increased from 10
  min: 5,
  idleTimeoutMillis: 30000
});
```

**3. Async Database Writes:**

```javascript
// Don't wait for database write before delivery
async function handleSendMessage(message) {
  // Deliver immediately
  deliverToRecipient(message);
  
  // Save to database in background
  setImmediate(() => {
    messageOperations.savePrivateMessage(message).catch(err => {
      // Queue for retry on failure
      queueForRetry(message);
    });
  });
}
```

**4. Message Batching:**

```javascript
// Batch multiple messages into single database transaction
const messageBatch = [];
const batchSize = 10;
const batchTimeout = 100; // ms

function addToBatch(message) {
  messageBatch.push(message);
  
  if (messageBatch.length >= batchSize) {
    flushBatch();
  } else {
    setTimeout(flushBatch, batchTimeout);
  }
}

async function flushBatch() {
  if (messageBatch.length === 0) return;
  
  const batch = messageBatch.splice(0);
  await db.query('BEGIN');
  for (const message of batch) {
    await db.query('INSERT INTO messages ...', [message]);
  }
  await db.query('COMMIT');
}
```

## Monitoring Endpoints

### Health Endpoint

**GET /health**

Returns server health and basic metrics:

```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "uptime": 3600,
  "connections": 150,
  "memory": {
    "rss": 53687091,
    "heapTotal": 13631488,
    "heapUsed": 10485760
  },
  "database": {
    "connected": true,
    "pool": {
      "totalCount": 10,
      "idleCount": 3,
      "waitingCount": 0
    }
  },
  "eventLoop": {
    "lag": 2.5,
    "utilization": 0.15
  }
}
```

### Metrics Endpoint

**GET /metrics**

Returns detailed performance metrics:

```json
{
  "connections": 150,
  "messagesSent": 5000,
  "messagesReceived": 4800,
  "errors": 5,
  "latency": {
    "p50": 25,
    "p95": 80,
    "p99": 150
  },
  "database": {
    "queries": 10000,
    "avgQueryTime": 8,
    "slowQueries": 5
  },
  "eventLoop": {
    "lag": 2.5,
    "utilization": 0.15
  }
}
```

## Summary

Performance monitoring is essential for maintaining system health and identifying bottlenecks. The highest latency components are database operations and network latency. Event loop monitoring helps identify blocking operations. Profiling tools like Clinic.js and 0x help identify performance issues. JSON serialization can be optimized through caching and payload minimization. Throughput can be improved through database write optimization, connection pool tuning, and message batching.

