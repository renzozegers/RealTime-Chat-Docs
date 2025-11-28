# System Architecture

## Overview

The real-time messaging system is a Node.js-based microservice that handles WebSocket connections, message routing, persistence, and delivery for direct messages, group chats, and community chats.

## Core Components

### 1. WebSocket Server Layer

The server uses the `ws` library to handle WebSocket connections. Multiple worker processes (via Node.js cluster) each run their own WebSocket server instance.

**Key characteristics:**
- Per-message deflate compression (reduced memory settings for 512MB RAM)
- Max payload size: 1MB
- Keep-alive timeout: 60 seconds
- Connection tracking: `wss.clients.size` for connection limits


### 2. Connection Handler

Manages the WebSocket connection lifecycle:

- **Connection establishment**: Validates IP, checks connection limits, creates client record
- **Authentication**: Validates JWT tokens, extracts user information
- **Message processing**: Routes messages to message router
- **Timeout handling**: 5-minute inactivity timeout (resets on activity)
- **Cleanup**: Removes client on disconnect, handles grace period for offline status

**Key data structures:**
- `clients` Map: `clientId -> {ws, user, clientIp, lastActivity, connectionTimeout}`
- `userConnections` Map: `userId -> clientId`
- `sessions` Map: `userId -> clientId` (for single connection enforcement)


### 3. Message Router

Routes incoming WebSocket messages to appropriate handlers based on message type.

**Message types:**
- Authentication: `connect`
- Direct messaging: `send_message`, `get_messages`, `update_message`, etc.
- Group chats: `create_group`, `send_group_message`, `add_group_member`, etc.
- System: `ping`, `test`, `get_online_users`


### 4. Message Handlers

Process private messaging operations:

- **Sending messages**: Validates relationships, encrypts content, saves to database, delivers to recipient
- **Loading messages**: Loads from database with pagination, decrypts content, loads reactions
- **Editing messages**: Validates permissions, checks 5-minute edit window, re-encrypts content
- **Deleting messages**: Soft delete (per-user) or hard delete (for everyone)
- **Reactions**: Adds/removes reactions, broadcasts to participants
- **Typing indicators**: Real-time typing status


### 5. Group Handlers

Process group chat operations:

- **Group management**: Create, update, delete groups
- **Member management**: Add, remove, promote, demote members
- **Group messaging**: Send messages with @mentions, handle permissions
- **Group features**: Pin messages, send announcements, manage roles


### 6. In-Memory State

Each worker process maintains Maps for fast access:

- **clients**: Active WebSocket connections
- **userConnections**: User ID to client ID mapping
- **conversations**: Active conversation state (last 50-100 messages)
- **groups**: Active group state (last 50-100 messages)
- **eventQueues**: In-memory event queue (userId -> Array of events)
- **sessions**: User session tracking
- **ipConnections**: IP-based connection tracking
- **messageRateLimit**: Per-user rate limiting
- **pendingOfflineTimers**: Grace period timers for offline detection

**Memory management:**
- Conversations/groups removed from memory after 30 minutes of inactivity
- Message arrays limited to last 50-100 messages in memory
- Old messages loaded from database on demand
- Periodic cleanup every 30 seconds


## Data Persistence

### PostgreSQL Database

**Database Tables:**
- Messages table: All messages (encrypted content, metadata)
- Groups table: Group metadata (name, description, max members)
- Group members table: Group membership and roles (owner, admin, member)
- Event queue table: Queued events for offline users
- Conversations table: Conversation metadata (participants, last message)
- Reactions table: Message reactions (user, reaction, timestamp)
- Deletion records: Per-user message and conversation deletion records

**Connection Pool:**
- Max connections: 10 (configurable, total app limit: 40)
- Min connections: 2 (configurable)
- Idle timeout: 30 seconds
- Connection timeout: 2 seconds
- Statement timeout: 30 seconds


### Redis Cache (Optional)

**Usage:**
- Session data: `session:{clientId}` (TTL: 1 hour)
- Message cache: `message:{messageId}` (TTL: 1 hour)
- Conversation cache: `conversation:{conversationId}` (last 100 messages)

**Fallback:**
- If Redis unavailable, system continues in database-only mode
- In development, uses Redis Mock for faster iteration


## Security Layer

### JWT Authentication

- Validates JWT tokens independently (microservice approach)
- Checks token signature and expiration
- Verifies JTI (JWT ID) against revocation table
- Fetches user details from database
- No server-side caching (fresh data on each request)


### Message Encryption

- Algorithm: AES-256-CBC
- Key derivation: scrypt from CHAT_ENCRYPTION_KEY
- IV: Random 16 bytes per message
- Storage: Encrypted content and IV stored in database


### Rate Limiting

- IP-based: 3000 requests per 15 minutes
- User-based: 240 messages per minute (configurable)
- Per-client tracking in memory
- Cleanup of old entries every 30 seconds


### Relationship Checks

- **Blocking**: Bidirectional blocking check (prevents messaging)
- **Following**: Optional check (soft warning, doesn't block)
- Batch checks for efficiency (group member additions)


## Message Flow

### Sending a Message

1. Client sends `send_message` via WebSocket
2. Connection handler validates authentication and rate limits
3. Message router routes to message handler
4. Handler checks access permissions and user relationships
5. Handler generates message ID (UUID)
6. Handler encrypts message content
7. Handler begins database transaction
8. Handler inserts message into database
9. Handler upserts conversation metadata
10. Handler commits transaction
11. Handler caches message in Redis (if available)
12. Handler stores message in memory (conversations Map)
13. Handler sends confirmation to sender
14. Handler delivers to recipient if online, or queues for offline delivery

### Loading Messages

1. Client sends `get_messages` with pagination
2. Handler loads from Redis cache (if available, first page only)
3. Handler loads from database with optimized query (CTE for username lookups)
4. Handler decrypts encrypted messages
5. Handler loads reactions for fetched messages
6. Handler filters deleted messages (per-user)
7. Handler returns messages in chronological order

## Reconnection Handling

### Disconnection

1. WebSocket close event received
2. Connection handler checks for other active connections
3. If no other connections, starts 60-second grace period timer
4. If grace period expires, user marked offline and `user_offline` event broadcast

### Reconnection

1. New WebSocket connection established
2. Client authenticates with JWT token
3. Connection handler cancels any pending offline timer
4. Handler loads in-memory queued events (fast)
5. Handler loads database queued events (persistent)
6. Handler delivers events in chronological order
7. Handler marks events as delivered in database
8. Handler clears in-memory queue

## Scaling Model

### Single Process

- Development mode: Single process execution
- All connections handled by single process
- In-memory state shared across all connections
- Suitable for small deployments (< 50 concurrent users)

### Cluster Mode

- Production mode: Multi-process execution
- Spawns multiple worker processes (based on CPU cores, max 8)
- Each worker handles subset of connections
- Each worker maintains separate in-memory state
- All workers share same PostgreSQL database
- Optional Redis for cross-worker caching

**Limitation**: Users on different workers cannot see each other's real-time presence. Requires Redis pub/sub or message broker for true multi-worker presence.

## Performance Optimizations

### Database Queries

- **CTE-based queries**: Pre-compute usernames once instead of N subqueries
- **Parallel queries**: Run message loading and COUNT in parallel
- **Indexes**: Optimized indexes on frequently queried columns
- **Connection pooling**: Reuse database connections

### Memory Management

- **Limited in-memory messages**: Last 50-100 messages per conversation/group
- **Inactive cleanup**: Remove conversations/groups after 30 minutes of inactivity
- **Periodic cleanup**: Every 30 seconds, clean up old data
- **Memory monitoring**: Log warnings when approaching limits (512MB for Heroku Eco)

### Caching

- **Redis caching**: Cache recent messages and session data
- **In-memory caching**: Fast access to active conversations
- **Profile picture caching**: 85-95% cache hit rate

## Error Handling

- **Database errors**: Transactions rolled back on error
- **Redis errors**: Graceful fallback to database-only mode
- **WebSocket errors**: Connection closed, client notified
- **Validation errors**: Detailed error messages sent to client
- **Rate limit errors**: Client notified with retry-after time

## Monitoring

- **Health endpoint**: Server status, metrics, memory usage
- **Readiness probe**: Database connection check
- **Liveness probe**: Server running check
- **Metrics endpoint**: Detailed performance metrics

## Deployment Considerations

- **Database connections**: Total app limit of 40 (chat server: 10, main backend: 30)
- **Memory**: Minimum 512MB RAM (1GB recommended)
- **WebSocket support**: Platform must support WebSocket (wss:// for production)
- **Environment variables**: All required variables must be set
- **TLS/SSL**: Required for production (wss://)

