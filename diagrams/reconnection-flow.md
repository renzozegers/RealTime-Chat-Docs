# Reconnection and Offline State Resynchronization Flow

```mermaid
sequenceDiagram
    participant Client as Client
    participant WS as WebSocket Server
    participant CH as Connection Handler
    participant MH as Message Handler
    participant DB as Database
    participant EQ_MEM as In-Memory<br/>Event Queue
    participant EQ_DB as Database<br/>Event Queue
    participant Timer as Offline Timer

    Note over Client: User disconnects<br/>(network issue, app closed)

    Client->>WS: WebSocket close event
    WS->>CH: Connection closed handler
    
    CH->>CH: Check for other connections<br/>for same userId
    
    alt No other active connections
        CH->>Timer: Start grace period<br/>(60 seconds)
        Note over Timer: Grace period allows<br/>quick reconnection without<br/>marking offline
        
        alt User reconnects within grace period
            Client->>WS: New WebSocket connection
            WS->>CH: Handle connection
            CH->>CH: Authenticate user
            CH->>Timer: Cancel offline timer
            CH->>MH: Process authentication
            MH->>EQ_MEM: Get in-memory queued events
            MH->>EQ_DB: Get database queued events
            
            loop For each queued event
                MH->>Client: Send queued event<br/>(new message,<br/>reaction, etc.)
            end
            
            MH->>EQ_DB: Mark events as delivered
            MH->>EQ_MEM: Clear in-memory queue
            
            Note over Client: User receives all<br/>missed events
        else Grace period expires
            Timer->>CH: Timeout fired
            CH->>CH: Remove from active connections
            CH->>CH: Broadcast user offline event
            Note over CH: User marked as offline
        end
    else Other active connection exists
        CH->>CH: Keep user online<br/>(point to other connection)
        Note over CH: User remains online<br/>via other device/tab
    end
```

## Reconnection Flow Details

### Disconnection Detection

When a WebSocket connection closes:

1. **Connection Handler** receives close event
2. Checks if user has other active connections (multiple tabs/devices)
3. If no other connections, starts a **grace period timer** (60 seconds)

### Grace Period

The 60-second grace period serves multiple purposes:

- **Quick Reconnection**: If user reconnects quickly (network hiccup), they remain "online" without interruption
- **Prevents False Offline**: Temporary network issues don't immediately mark user as offline
- **Allows Reconnection**: User has time to reconnect before being marked offline

### Reconnection Process

When user reconnects (within or after grace period):

1. **New WebSocket Connection**: Client establishes new WebSocket connection
2. **Authentication**: Client sends authentication message with token
3. **Connection Handler**:
   - Validates token with authentication service
   - Cancels any pending offline timer
   - Updates connection mappings with new connection ID
   - Updates session data
4. **Event Queue Delivery**:
   - **In-Memory Queue First**: Delivers events from memory queue (fast, immediate)
   - **Database Queue Second**: Loads events from database queue (persistent, survives restarts)
   - Events sent in chronological order (sorted by timestamp)
   - Small delay between events to prevent overwhelming client
5. **Mark as Delivered**: Updates database queue with delivery timestamp
6. **Clear Memory**: Removes events from in-memory queue after delivery

### Offline State

If grace period expires without reconnection:

1. User removed from active connections
2. User removed from session data
3. User offline event broadcast to all connected clients
4. Events remain queued in database for delivery on next connection

### Event Queue Types

Events queued for offline users include:

- **New messages**: Messages sent while offline
- **Reactions**: Message reactions added/removed
- **Edits**: Message edits
- **Deletions**: Message deletions
- **Membership changes**: Group membership changes
- **Group updates**: Group information updates

### Key Design Decisions

1. **Dual Queue System**: In-memory queue for speed, database queue for persistence
2. **Grace Period**: Prevents false offline status during temporary disconnections
3. **Chronological Delivery**: Events delivered in order they occurred
4. **Automatic Cleanup**: Delivered events cleaned up after retention period
5. **No Message Loss**: All events persisted to database, survive server restarts

### Limitations

- **Single Worker State**: In-memory event queue only works within same worker process
- **Cross-Worker Presence**: Users on different workers cannot see each other's real-time presence (would require message broker)
- **Event Ordering**: Events from different workers may arrive out of order (database queue ensures eventual consistency)
