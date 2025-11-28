# System Architecture Diagram

```mermaid
graph LR
    subgraph Clients["Client Layer"]
        C1[WebSocket<br/>Client 1]
        C2[WebSocket<br/>Client 2]
        C3[WebSocket<br/>Client N]
        HTTP[HTTP<br/>Client]
    end

    LB[Load Balancer<br/>Reverse Proxy]

    subgraph Workers["Application Server Cluster"]
        subgraph W1["Worker Process 1"]
            WS1[WebSocket<br/>Server]
            CH1[Connection<br/>Handler]
            MR1[Message<br/>Router]
            MH1[Message<br/>Handlers]
            GH1[Group<br/>Handlers]
        end
        
        subgraph W2["Worker Process 2"]
            WS2[WebSocket<br/>Server]
            CH2[Connection<br/>Handler]
            MR2[Message<br/>Router]
            MH2[Message<br/>Handlers]
            GH2[Group<br/>Handlers]
        end
        
        subgraph WN["Worker Process N"]
            WSN[WebSocket<br/>Server]
            CHN[Connection<br/>Handler]
            MRN[Message<br/>Router]
            MHN[Message<br/>Handlers]
            GHN[Group<br/>Handlers]
        end
    end

    subgraph Memory["In-Memory State<br/>(Per Worker)"]
        CLIENTS[Active Connections]
        USER_CONN[User Mappings]
        CONV[Conversations]
        GROUPS[Groups]
        EVENT_QUEUE[Event Queues]
    end

    subgraph Security["Security Layer"]
        JWT[Token Validation]
        ENC[Encryption]
        RATE[Rate Limiter]
        BLOCK[Access Control]
        FOLLOW[Relationship Checks]
    end

    subgraph Database["Relational Database"]
        MSG[Messages]
        GRP[Groups]
        MEM[Memberships]
        EVT[Event Queue]
        CONV_TBL[Conversations]
        REACT[Reactions]
        USERS[User Data]
    end

    subgraph Cache["Distributed Cache<br/>(Optional)"]
        REDIS[Session Cache<br/>Message Cache]
    end

    subgraph External["External Services"]
        AUTH_SVC[Authentication Service<br/>Token Verification]
    end

    C1 -->|WebSocket| LB
    C2 -->|WebSocket| LB
    C3 -->|WebSocket| LB
    HTTP -->|HTTP/REST| LB

    LB --> WS1
    LB --> WS2
    LB --> WSN

    WS1 --> CH1
    CH1 --> MR1
    MR1 --> MH1
    MR1 --> GH1

    WS2 --> CH2
    CH2 --> MR2
    MR2 --> MH2
    MR2 --> GH2

    WSN --> CHN
    CHN --> MRN
    MRN --> MHN
    MRN --> GHN

    MH1 --> Memory
    GH1 --> Memory

    MH1 --> Security
    GH1 --> Security

    MH1 --> Database
    GH1 --> Database

    MH1 -.->|Cache| Cache
    GH1 -.->|Cache| Cache

    JWT --> External
    JWT --> USERS

    style Clients fill:#000000,color:#ffffff
    style Workers fill:#000000,color:#ffffff
    style Memory fill:#000000,color:#ffffff
    style Security fill:#000000,color:#ffffff
    style Database fill:#000000,color:#ffffff
    style Cache fill:#000000,color:#ffffff
    style External fill:#000000,color:#ffffff
```

## Architecture Overview

The real-time messaging system is a WebSocket-based application designed for horizontal scaling. The architecture consists of:

### Core Components

1. **WebSocket Server Layer**: Multiple worker processes handle WebSocket connections. Each worker maintains its own WebSocket server instance.

2. **Connection Handler**: Manages WebSocket lifecycle - connection establishment, authentication, timeout handling, and cleanup. Tracks client connections in memory.

3. **Message Router**: Routes incoming WebSocket messages to appropriate handlers based on message type (direct messages, group messages, authentication, etc.).

4. **Message Handlers**: Process direct messaging operations - sending, receiving, editing, deleting messages, managing reactions, and handling typing indicators.

5. **Group Handlers**: Process group chat operations - creating groups, sending group messages, managing members, handling group-specific reactions and announcements.

6. **In-Memory State**: Each worker process maintains data structures for:
   - Active connections tracking
   - User to connection mappings
   - Active conversation state
   - Active group state
   - Queued events for offline users

### Data Persistence

- **Relational Database**: Primary database for all persistent data:
  - Messages and metadata
  - Groups and memberships
  - Event queue for offline delivery
  - User relationships
  - Reactions

- **Distributed Cache (Optional)**: Caching layer for:
  - Session data
  - Recent messages
  - User metadata

### Security Layer

- **Token Authentication**: Verifies tokens with authentication service, checks revocation status
- **Message Encryption**: Encryption for message content at rest
- **Rate Limiting**: IP-based and user-based rate limits
- **Access Control**: Validates user relationships before allowing messaging

### Scaling Model

The system uses a cluster module to spawn multiple worker processes. Each worker:
- Handles a subset of WebSocket connections
- Maintains its own in-memory state
- Shares the same relational database
- Can optionally share distributed cache for cross-worker caching

Since each worker maintains separate in-memory state, users connected to different workers cannot see each other's real-time presence. This is a limitation of the current architecture that would require a message broker or pub/sub system to resolve for true multi-worker presence.
