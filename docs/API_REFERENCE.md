# Real-Time Messaging API Reference

High-level API concepts and patterns for a real-time messaging system.

## Overview

The messaging system uses WebSocket connections for real-time bidirectional communication and HTTP endpoints for RESTful operations. All messages are JSON objects with a `type` field indicating the message type.

## WebSocket Message Patterns

### Authentication Pattern

**Connection Flow:**
- Client establishes WebSocket connection
- Client sends authentication message with token
- Server validates token and establishes session
- Server delivers any queued events
- Client receives connection confirmation

**Key Concepts:**
- Token-based authentication
- Session establishment
- Queued event delivery on connection
- Single or multi-session support

### Direct Messaging Pattern

**Message Sending:**
- Client sends message with recipient identifier and content
- Server validates permissions and relationships
- Server encrypts and persists message
- Server delivers to recipient if online, or queues for offline delivery
- Client receives confirmation

**Message Loading:**
- Client requests message history with pagination
- Server loads from cache or database
- Server decrypts messages
- Server includes reactions and metadata
- Client receives paginated message list

**Key Concepts:**
- Real-time delivery for online users
- Offline queueing for offline users
- Message encryption at rest
- Pagination support
- Reaction support

### Message Modification Patterns

**Editing:**
- Time-limited edit window
- Permission checks (sender only)
- Content re-encryption
- Broadcast to participants

**Deletion:**
- Per-user deletion (soft delete)
- Global deletion (hard delete)
- Permission-based access control
- Broadcast to participants

### Reaction Pattern

**Adding Reactions:**
- One reaction per user (replaces previous)
- Toggle behavior (same reaction removes it)
- Broadcast to all participants
- Persistent storage

### Typing Indicators

**Pattern:**
- Client sends typing status
- Server broadcasts to recipient
- Automatic timeout after inactivity
- Real-time updates

### Group Messaging Pattern

**Group Management:**
- Create groups with initial members
- Member management (add/remove)
- Permission-based access control
- Role-based permissions

**Group Messaging:**
- Send messages to group
- @mention support
- Broadcast to all members
- Offline queueing per member

## HTTP Endpoint Patterns

### Health Check Pattern

**Purpose:** Server health status and system metrics

**Response Structure:**
- Status indicator
- Timestamp
- System metrics (connections, memory, database)
- Performance counters

**Use Cases:**
- Monitoring and alerting
- Load balancer health checks
- Orchestration system probes

### Readiness Probe Pattern

**Purpose:** Check if service is ready to accept traffic

**Response Structure:**
- Ready status
- Component health checks
- Timestamp

**Use Cases:**
- Kubernetes readiness probes
- Deployment verification
- Service dependency checks

### Liveness Probe Pattern

**Purpose:** Check if service is running

**Response Structure:**
- Alive status
- Uptime information
- Timestamp

**Use Cases:**
- Kubernetes liveness probes
- Service monitoring
- Automatic restart triggers

### Resource Listing Pattern

**Purpose:** Retrieve paginated lists of resources

**Request Parameters:**
- Pagination (limit, offset)
- Filtering options
- Sorting options

**Response Structure:**
- Resource list
- Pagination metadata
- Total count
- Timestamp

**Use Cases:**
- Conversation lists
- Message history
- User lists

### Batch Operations Pattern

**Purpose:** Process multiple items in a single request

**Request Structure:**
- Array of items to process
- Batch size limits
- Item types

**Response Structure:**
- Processed items
- Success/failure status per item
- Batch metadata
- Timestamp

**Use Cases:**
- Batch metadata fetching
- Bulk operations
- Resource loading

## Error Handling Patterns

**Error Response Structure:**
- Error type identifier
- Human-readable message
- Detailed error information
- Error code (generic)
- Retry information (if applicable)

**Common Error Categories:**
- Authentication errors
- Authorization errors
- Rate limiting errors
- Resource not found errors
- Permission errors
- Capacity errors

**Error Handling Principles:**
- Consistent error format
- Actionable error messages
- Appropriate HTTP status codes
- Retry guidance where applicable

## Design Principles

- **Real-time Delivery**: WebSocket-based for instant message delivery
- **Offline Support**: Events queued for offline users, delivered on reconnect
- **Security**: End-to-end encryption, rate limiting, access control
- **Scalability**: Horizontal scaling with worker processes
- **Reliability**: Persistent storage with transaction support
- **Performance**: Multi-layer caching and optimized queries
