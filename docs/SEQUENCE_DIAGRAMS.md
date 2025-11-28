# Sequence Diagrams

Detailed sequence diagrams for key system interactions in a real-time messaging system.

## Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant WS as WebSocket Server
    participant CH as Connection Handler
    participant Auth as Authentication Service
    participant DB as Database
    participant Backend as Auth Backend

    Client->>WS: WebSocket Connection
    WS->>CH: Handle connection
    CH->>CH: Generate client ID
    CH->>CH: Validate IP, check limits
    CH->>Client: {type: "connected", requiresAuth: true}
    
    Client->>WS: {type: "connect", token: "auth-token"}
    WS->>CH: Route message
    CH->>Auth: Verify token
    Auth->>Auth: Validate signature & expiration
    Auth->>DB: Check token revocation
    Auth->>DB: Get user details
    DB-->>Auth: User data
    Auth-->>CH: {valid: true, user: {...}}
    
    CH->>CH: Check existing session
    alt User already connected
        CH->>CH: Close old connection
    end
    
    CH->>CH: Update connection mappings
    CH->>CH: Cancel offline timer (if any)
    CH->>CH: Load queued events
    CH->>Client: Send in-memory queued events
    CH->>DB: Load database queued events
    CH->>Client: Send database queued events
    CH->>DB: Mark events as delivered
    CH->>Client: {type: "connected", user: {...}}
    CH->>Client: {type: "online_users", users: [...]}
```

## Message Sending Flow (Online Recipient)

```mermaid
sequenceDiagram
    participant Sender
    participant WS as WebSocket Server
    participant CH as Connection Handler
    participant MR as Message Router
    participant MH as Message Handler
    participant Security as Security Layer
    participant DB as Database
    participant Cache as Distributed Cache
    participant Recipient

    Sender->>WS: {type: "send_message", recipientId, content}
    WS->>CH: Route message
    CH->>CH: Validate auth, rate limits
    CH->>MR: Process message
    MR->>MH: Handle message send
    
    MH->>Security: Check access permissions
    Security->>DB: Query user relationships
    DB-->>Security: Access status
    Security-->>MH: {allowed: true}
    
    MH->>MH: Generate message ID
    MH->>MH: Encrypt content
    MH->>DB: BEGIN TRANSACTION
    MH->>DB: INSERT message
    MH->>DB: UPDATE conversation metadata
    MH->>DB: COMMIT
    
    MH->>Cache: Cache message (if available)
    MH->>MH: Store in memory
    MH->>Sender: {type: "message_sent"}
    
    MH->>MH: Find recipient connection
    MH->>Recipient: {type: "message_received", message: {...}}
```

## Message Sending Flow (Offline Recipient)

```mermaid
sequenceDiagram
    participant Sender
    participant WS as WebSocket Server
    participant MH as Message Handler
    participant DB as Database
    participant EQ_MEM as In-Memory Queue
    participant EQ_DB as Database Queue

    Sender->>WS: {type: "send_message", recipientId, content}
    WS->>MH: Handle message send
    MH->>MH: Save message to database
    MH->>MH: Check recipient online status
    
    alt Recipient offline
        MH->>EQ_MEM: Add to in-memory queue
        MH->>EQ_DB: Queue event for user
        EQ_DB->>DB: INSERT event to queue
        Note over EQ_DB: Event persisted for later delivery
    end
    
    MH->>Sender: {type: "message_sent"}
    
    Note over EQ_DB: User reconnects later...
    EQ_DB->>MH: Load queued events
    MH->>Sender: Deliver queued events
    EQ_DB->>DB: UPDATE queue SET delivered_at
```

## Reconnection and Event Delivery

```mermaid
sequenceDiagram
    participant Client
    participant WS as WebSocket Server
    participant CH as Connection Handler
    participant MH as Message Handler
    participant EQ_MEM as In-Memory Queue
    participant EQ_DB as Database Queue
    participant DB as Database

    Note over Client: User disconnects
    Client->>WS: WebSocket close
    WS->>CH: Connection closed handler
    CH->>CH: Start 60s grace period timer
    
    Note over Client: User reconnects within grace period
    Client->>WS: New WebSocket connection
    WS->>CH: Handle connection
    Client->>WS: {type: "connect", token: "auth-token"}
    WS->>MH: Process authentication
    
    MH->>CH: Cancel offline timer
    MH->>EQ_MEM: Get in-memory queued events
    EQ_MEM-->>MH: [event1, event2, ...]
    
    loop For each in-memory event
        MH->>Client: Send event
    end
    
    MH->>EQ_DB: Get queued events for user
    EQ_DB->>DB: SELECT FROM event_queue WHERE delivered_at IS NULL
    DB-->>EQ_DB: [event3, event4, ...]
    EQ_DB-->>MH: Queued events
    
    loop For each database event
        MH->>Client: Send event
        MH->>EQ_DB: Mark event for delivery
    end
    
    MH->>EQ_DB: Mark events as delivered
    EQ_DB->>DB: UPDATE event_queue SET delivered_at = NOW()
    MH->>EQ_MEM: Clear in-memory queue
    MH->>Client: {type: "connected"}
```

## Group Message Flow

```mermaid
sequenceDiagram
    participant Sender
    participant WS as WebSocket Server
    participant GH as Group Handler
    participant DB as Database
    participant Members as Group Members

    Sender->>WS: {type: "send_group_message", groupId, content}
    WS->>GH: Handle group message
    
    GH->>DB: Check group membership
    GH->>DB: Check user permissions
    GH->>GH: Parse mentions from content
    GH->>DB: Resolve mentioned users
    GH->>GH: Generate message ID
    GH->>DB: INSERT message (group flag=true)
    GH->>GH: Store in groups memory
    
    GH->>Sender: {type: "group_message_sent"}
    
    loop For each group member
        alt Member is online
            GH->>Member: {type: "group_message_received", message: {...}}
        else Member is offline
            GH->>DB: Queue event for member
        end
    end
    
    alt Mentions found
        loop For each mentioned user
            GH->>MentionedUser: {type: "mentioned_in_group", ...}
        end
    end
```

## Message Edit Flow

```mermaid
sequenceDiagram
    participant Sender
    participant WS as WebSocket Server
    participant MH as Message Handler
    participant DB as Database
    participant Recipient

    Sender->>WS: {type: "update_message", messageId, newContent}
    WS->>MH: Handle message update
    
    MH->>MH: Find message in memory or DB
    MH->>MH: Verify sender is message owner
    MH->>MH: Check edit time window
    
    alt Edit window expired
        MH->>Sender: {type: "error", message: "Edit time limit exceeded"}
    else Edit allowed
        MH->>MH: Encrypt new content
        MH->>DB: UPDATE message SET content, edited_at
        MH->>MH: Update message in memory
        MH->>Sender: {type: "message_updated", newContent, oldContent}
        
        MH->>Recipient: {type: "message_updated", messageId, newContent}
    end
```

## Reaction Flow

```mermaid
sequenceDiagram
    participant User
    participant WS as WebSocket Server
    participant MH as Message Handler
    participant DB as Database
    participant Participants as Conversation Participants

    User->>WS: {type: "add_reaction", messageId, reaction}
    WS->>MH: Handle reaction
    
    MH->>DB: Load message (if not in memory)
    MH->>DB: Check existing reaction
    
    alt User already has this reaction
        MH->>DB: DELETE reaction
        MH->>User: {type: "reaction_removed"}
        MH->>Participants: {type: "reaction_removed"}
    else New reaction
        MH->>DB: DELETE existing reaction (one per user)
        MH->>DB: INSERT reaction
        MH->>MH: Update message in memory
        MH->>User: {type: "reaction_added"}
        MH->>Participants: {type: "reaction_added"}
    end
```

## Connection Lifecycle

```mermaid
sequenceDiagram
    participant Client
    participant WS as WebSocket Server
    participant CH as Connection Handler
    participant Timer as Inactivity Timer
    participant DB as Database

    Client->>WS: WebSocket Connection
    WS->>CH: Handle connection
    CH->>CH: Generate client ID
    CH->>CH: Create client record
    CH->>Timer: Start inactivity timer
    CH->>Client: {type: "connected"}
    
    loop While connected
        Client->>WS: Any message
        WS->>CH: Reset inactivity timer
        Timer->>Timer: Clear and restart timer
    end
    
    alt Inactivity timeout
        Timer->>CH: Timeout fired
        CH->>Client: Close connection (timeout)
    else Client disconnects
        Client->>WS: WebSocket close
        WS->>CH: Connection closed handler
        CH->>CH: Cleanup client record
        CH->>Timer: Clear inactivity timer
        CH->>DB: Cleanup session data
    end
```
