# Message Delivery Pipeline

## Overview

The message delivery pipeline handles the complete flow of messages from sender to recipient, including validation, persistence, encryption, caching, and delivery (real-time or queued).

## Pipeline Stages

### Stage 1: Message Reception


1. **WebSocket message received**: Raw JSON data from client
2. **JSON parsing**: Parse message, handle malformed JSON errors
3. **Connection timeout reset**: Reset inactivity timer
4. **Activity tracking**: Update client's `lastActivity` timestamp
5. **Rate limit check**: Validate per-user and per-IP rate limits
6. **Authentication check**: Verify user is authenticated for protected operations
7. **Message routing**: Forward to message router

**Protected operations** require authentication:
- `send_message`, `get_messages`, `update_message`, `remove_message`
- `create_group`, `send_group_message`, `add_group_member`
- `get_online_users`, `typing`, `add_reaction`, etc.

### Stage 2: Message Validation


1. **Input validation**: Validate required fields (recipientId, content)
2. **Content sanitization**: Sanitize message content (XSS prevention)
3. **Blocking check**: Verify users are not blocked (bidirectional)
4. **Following check**: Optional check (soft warning, doesn't block)
5. **Conversation ID generation**: Generate deterministic conversation ID from user IDs

**Blocking logic:**
- Checks blocking relationships table for bidirectional blocks
- If blocked, message rejected with access denied error
- Batch checks for efficiency (group member additions)

**Following logic:**
- Checks following relationships table
- Soft warning if no follow relationship (doesn't block)
- Required for adding users to groups

### Stage 3: Message Creation


1. **Message ID generation**: Generate UUID for message
2. **Message object creation**: Create message object with metadata:
   - `id`: UUID
   - `conversationId`: Deterministic conversation ID
   - `senderId`: Normalized user ID
   - `recipientId`: Normalized recipient ID
   - `content`: Original content (before encryption)
   - `timestamp`: ISO timestamp
   - `isRead`: false (initially)
   - `replyTo`: Optional reply-to message ID

3. **In-memory storage**: Store in `conversations` Map
4. **Memory limit**: Keep last 50-100 messages in memory, older messages loaded from DB

### Stage 4: Database Persistence


1. **Transaction start**: Begin database transaction
2. **Conversation metadata**: Upsert conversation metadata in `conversations` table
3. **Message encryption**: Encrypt message content (AES-256-CBC)
4. **Message insertion**: Insert into messages table:
   - `message_id`: UUID
   - `conversation_id`: Conversation ID
   - `sender_id`: Sender user ID
   - `recipient_id`: Recipient user ID
   - `content`: Plain text (for search/backup)
   - `encrypted_content`: Encrypted content (hex)
   - `encryption_iv`: Initialization vector (hex)
   - `is_encrypted`: true
   - `is_read`: false
   - `reply_to`: Optional reply-to message ID

5. **Transaction commit**: Commit transaction (atomic operation)

**Error handling:**
- Transaction rollback on any error
- Error logged with full context
- Error propagated to caller

### Stage 5: Caching


1. **Redis cache check**: If Redis available, cache message
2. **Individual message cache**: `message:{messageId}` (TTL: 1 hour)
3. **Conversation cache**: `conversation:{conversationId}` (last 100 messages)
4. **Cache failure handling**: Graceful fallback (doesn't fail operation)

**Cache strategy:**
- Cache only after successful database write
- Cache failure doesn't affect message delivery
- Cache TTL: 1 hour (configurable)

### Stage 6: Delivery


#### Online Delivery

1. **Recipient lookup**: Find recipient's clientId in `userConnections` Map
2. **Connection check**: Verify WebSocket is open
3. **Message delivery**: Send `message_received` event via WebSocket
4. **Delivery delay**: 100ms delay to avoid race with mark-as-read

**Message format:**
```json
{
  "type": "message_received",
  "message": {
    "id": "uuid",
    "conversationId": "conv_user1_user2",
    "senderId": "user1",
    "recipientId": "user2",
    "content": "Hello",
    "timestamp": "2024-01-01T00:00:00.000Z",
    "isRead": false,
    "replyTo": null
  },
  "messageId": "uuid",
  "conversationId": "conv_user1_user2",
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

#### Offline Delivery

1. **In-memory queue**: Add event to `eventQueues` Map (userId -> Array)
2. **Database queue**: Insert into event queue table:
   - `user_id`: Recipient user ID
   - `event_type`: 'message_received'
   - `event_data`: JSON stringified event object
   - `created_at`: Current timestamp
   - `delivered_at`: NULL (until delivered)

3. **Event persistence**: Event survives server restarts (database-backed)

**Event format:**
```json
{
  "type": "message_received",
  "message": { /* full message object */ },
  "messageId": "uuid",
  "conversationId": "conv_user1_user2",
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

### Stage 7: Confirmation


1. **Sender confirmation**: Send `message_sent` event to sender
2. **Message object included**: Full message object in confirmation
3. **Metrics update**: Increment `messagesSent` counter

**Confirmation format:**
```json
{
  "type": "message_sent",
  "message": { /* full message object */ },
  "messageId": "uuid",
  "conversationId": "conv_user1_user2",
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

## Offline Delivery on Reconnection


When user reconnects:

1. **In-memory queue first**: Deliver events from `eventQueues` Map (fast)
2. **Database queue second**: Load events from event queue table (persistent)
3. **Chronological order**: Sort events by timestamp
4. **Sequential delivery**: Send events one by one with 10ms delay
5. **Mark as delivered**: Update event queue table with `delivered_at` timestamp
6. **Clear memory**: Remove events from in-memory queue

**Delivery order:**
- In-memory events first (fastest)
- Database events second (persistent)
- Both sorted by timestamp

## Error Handling

### Validation Errors

- **Missing fields**: Return error with field name
- **Invalid format**: Return error with details
- **Rate limit exceeded**: Return error with retry-after time

### Database Errors

- **Transaction rollback**: All changes reverted
- **Error logged**: Full context logged
- **Error returned**: Client receives error message

### Delivery Errors

- **WebSocket closed**: Event queued for later delivery
- **Redis unavailable**: Graceful fallback to database-only
- **Network errors**: Event remains queued

## Performance Characteristics

### Latency

- **Database write**: ~10-50ms (depending on load)
- **Encryption**: ~1-5ms (AES-256-CBC)
- **Online delivery**: <10ms (in-memory lookup + WebSocket send)
- **Offline queueing**: ~5-10ms (in-memory + database write)

### Throughput

- **Messages per second**: Limited by rate limits (240 msg/min per user)
- **Concurrent messages**: Limited by database connection pool (10 connections)
- **Offline queueing**: Handles thousands of queued events per user

### Memory Usage

- **In-memory messages**: ~50-100 messages per conversation (limited)
- **Event queue**: ~100-1000 events per user (limited by cleanup)
- **Connection state**: ~1KB per active connection

## Optimizations

1. **Batch operations**: Group member additions use batch SQL
2. **Parallel queries**: Message loading and COUNT run in parallel
3. **CTE-based queries**: Pre-compute usernames once instead of N subqueries
4. **Connection pooling**: Reuse database connections
5. **In-memory caching**: Fast access to active conversations
6. **Redis caching**: Cache recent messages and session data

## Monitoring

- **Metrics tracked**: `messagesSent`, `messagesReceived`, `errors`
- **Performance logging**: Query times logged for optimization
- **Error tracking**: All errors logged with full context
- **Health checks**: Health, readiness, and liveness endpoints

