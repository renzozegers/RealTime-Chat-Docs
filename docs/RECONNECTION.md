# Reconnection and Offline State Resynchronization

## Overview

The reconnection system handles user disconnections gracefully, preventing false offline status during temporary network issues and ensuring all missed events are delivered when users reconnect.

## Disconnection Detection

### WebSocket Close Event


When a WebSocket connection closes:

1. **Close event received**: Connection handler processes close event
2. **Connection cleanup**: Remove client from `clients` Map
3. **IP tracking cleanup**: Decrement IP connection count
4. **Redis cleanup**: Remove session data from Redis (if available)
5. **Offline detection**: Check if user should be marked offline

### Multiple Connection Handling

The system supports multiple connections per user (multiple tabs/devices):

1. **Check for other connections**: Look for other active connections for same userId
2. **If other connections exist**: User remains online, point `userConnections` to other connection
3. **If no other connections**: Start grace period timer

**Code logic:**
```javascript
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

## Grace Period

### Purpose

The 60-second grace period prevents false offline status during:
- Temporary network hiccups
- Mobile app backgrounding
- Browser tab switching
- Slow authentication

### Implementation


1. **Timer creation**: Create 60-second timeout
2. **Timer storage**: Store in `pendingOfflineTimers` Map (userId -> timer)
3. **Timer cancellation**: Cancel if user reconnects within grace period
4. **Timer expiration**: Mark user offline if grace period expires

**Code:**
```javascript
const OFFLINE_GRACE_PERIOD_MS = 60000; // 60 seconds

const offlineTimer = setTimeout(() => {
  // Double check no connections
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
    userConnections.delete(normalizedUserId);
    sessions.delete(normalizedUserId);
    broadcastToAll(offlineEvent, clientId, clients, metrics);
  }
  
  pendingOfflineTimers.delete(normalizedUserId);
}, OFFLINE_GRACE_PERIOD_MS);
```

## Reconnection Process

### Connection Establishment


1. **New WebSocket connection**: Client establishes new WebSocket connection
2. **Client ID generation**: Generate unique clientId (UUID)
3. **Connection validation**: Validate IP, check connection limits
4. **Client record creation**: Create entry in `clients` Map
5. **Welcome message**: Send `connected` event with authentication requirement

### Authentication


1. **JWT token validation**: Verify token signature and expiration
2. **JTI revocation check**: Check if token is revoked in database
3. **User details fetch**: Load user details from database
4. **Session conflict check**: If user already connected, close old connection
5. **Session update**: Update `sessions` and `userConnections` Maps
6. **Grace period cancellation**: Cancel any pending offline timer

**Code:**
```javascript
// Cancel offline timer if reconnecting
if (pendingOfflineTimers) {
  const pendingTimer = pendingOfflineTimers.get(normalizedUserId);
  if (pendingTimer) {
    clearTimeout(pendingTimer);
    pendingOfflineTimers.delete(normalizedUserId);
    console.log("[DEBUG] Canceled pending offline timer - user reconnected");
  }
}
```

### Event Queue Delivery


#### In-Memory Queue (Fast)

1. **Load from memory**: Get events from `eventQueues` Map
2. **Sort by timestamp**: Ensure chronological order
3. **Sequential delivery**: Send events one by one with 10ms delay
4. **Clear memory**: Remove events from Map after delivery

**Code:**
```javascript
const memoryQueuedEvents = eventQueues.get(normalizedUserId);
if (memoryQueuedEvents && memoryQueuedEvents.length > 0) {
  const sortedEvents = [...memoryQueuedEvents].sort((a, b) => a.timestamp - b.timestamp);
  for (const queuedEvent of sortedEvents) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(queuedEvent.event));
      await new Promise(resolve => setTimeout(resolve, 10));
    }
  }
  eventQueues.delete(normalizedUserId);
}
```

#### Database Queue (Persistent)

1. **Load from database**: Query event queue table for undelivered events
2. **Sort by timestamp**: ORDER BY created_at ASC
3. **Sequential delivery**: Send events one by one with 10ms delay
4. **Mark as delivered**: Update `delivered_at` timestamp
5. **Limit**: Maximum 1000 events per user (prevents overwhelming)

**Code:**
```javascript
const dbQueuedEvents = await getQueuedEventsForUser(normalizedUserId, dbPool);
if (dbQueuedEvents && dbQueuedEvents.length > 0) {
  const eventIds = [];
  for (const queuedEvent of dbQueuedEvents) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(queuedEvent.event));
      eventIds.push(queuedEvent.id);
      await new Promise(resolve => setTimeout(resolve, 10));
    }
  }
  await markEventsAsDelivered(eventIds, dbPool);
}
```

## Event Queue System

### Event Types

Events queued for offline users include:

- **new_private_message**: Private messages sent while offline
- **reaction_added / reaction_removed**: Message reactions
- **message_edited**: Message edits
- **message_deleted**: Message deletions
- **member_added / member_removed**: Group membership changes
- **group_info_updated**: Group information updates
- **message_pinned / message_unpinned**: Message pinning

### Dual Queue System

**In-Memory Queue:**
- Fast delivery (no database query)
- Limited to current worker process
- Lost on server restart
- Used for recent events

**Database Queue:**
- Persistent (survives restarts)
- Cross-worker compatible
- Slower (database query)
- Used for all events

### Queue Operations


**Queue event:**
```javascript
await queueEventForUser(userId, eventType, eventData, dbPool);
```

**Get queued events:**
```javascript
const events = await getQueuedEventsForUser(userId, dbPool);
```

**Mark as delivered:**
```javascript
await markEventsAsDelivered(eventIds, dbPool);
```

## Offline Status Broadcast

### User Offline Event


When grace period expires:

1. **User removed**: Remove from `userConnections` and `sessions` Maps
2. **Offline event created**: Create `user_offline` event
3. **Broadcast**: Broadcast to all connected clients

**Event format:**
```json
{
  "type": "user_offline",
  "user": {
    "id": "userId",
    "userId": "userId",
    "username": "username",
    "displayName": "Display Name"
  },
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

### User Online Event


When user authenticates (new connection or reconnect):

1. **Check existing connection**: Verify if user already has active connection
2. **Broadcast online**: If new connection, broadcast `user_online` event
3. **Skip if duplicate**: If user has other active connection, skip broadcast

**Event format:**
```json
{
  "type": "user_online",
  "user": {
    "id": "userId",
    "userId": "userId",
    "username": "username",
    "displayName": "Display Name"
  },
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

## Cleanup

### Event Queue Cleanup


**Delivered events:**
- Cleaned up after 7 days
- Runs during periodic cleanup (every 30 seconds)

**Undelivered events:**
- Cleaned up after 30 days
- Assumes user not coming back

**Code:**
```javascript
// Clean up delivered events (7 days)
await cleanupOldDeliveredEvents(dbPool);

// Clean up undelivered events (30 days)
await cleanupOldUndeliveredEvents(dbPool);
```

### In-Memory Queue Cleanup


- Cleaned up every 30 seconds
- Removes events older than 1 day
- Prevents memory leaks

## Limitations

### Single Worker State

- In-memory event queue only works within same worker process
- Users on different workers cannot see each other's real-time presence
- Database queue ensures cross-worker event delivery

### Event Ordering

- Events from different workers may arrive out of order
- Database queue ensures eventual consistency
- In-memory queue provides fast delivery within same worker

### Grace Period

- 60 seconds may be too short for some network conditions
- Configurable via `OFFLINE_GRACE_PERIOD_MS` (currently hardcoded)
- Mobile apps may need longer grace period

## Best Practices

1. **Client reconnection**: Implement exponential backoff for reconnection attempts
2. **Event handling**: Process events in order, handle duplicates gracefully
3. **State synchronization**: Reload conversation state after reconnection
4. **Error handling**: Handle WebSocket errors and reconnection failures
5. **Monitoring**: Track reconnection rates and event delivery times

