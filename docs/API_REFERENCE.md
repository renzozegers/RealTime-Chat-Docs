# API Reference

Complete reference for WebSocket message types and HTTP endpoints in the real-time messaging system.

## Table of Contents

1. [WebSocket API](#websocket-api)
2. [HTTP Endpoints](#http-endpoints)
3. [Message Formats](#message-formats)
4. [Error Handling](#error-handling)
5. [Authentication](#authentication)
6. [Rate Limiting](#rate-limiting)

---

## WebSocket API

All WebSocket messages are JSON objects with a `type` field indicating the message type.

### Connection Lifecycle

#### 1. Connect to WebSocket Server

**Connection:**
```
ws://localhost:3001 (development)
wss://chat.example.com (production)
```

**Initial Response:**
```json
{
  "type": "connected",
  "requiresAuth": true,
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

#### 2. Authenticate

**Message Type:** `authenticate` or `auth` (alias)

**Request:**
```json
{
  "type": "authenticate",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Success Response:**
```json
{
  "type": "connected",
  "user": {
    "userId": "123",
    "username": "john_doe",
    "displayName": "John Doe"
  },
  "queuedEvents": [...],
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

**Error Response:**
```json
{
  "type": "auth_error",
  "message": "Token validation failed",
  "details": "Session expired. Please log in again."
}
```

**Implementation Details:**
- JWT token validated against shared `JWT_SECRET`
- Token signature verified (HS256 algorithm)
- Expiration checked
- JTI (JWT ID) checked against revocation table
- User details fetched from database
- Existing connections closed if user reconnects
- Queued events delivered immediately after authentication

---

## Private Messaging

### Send Private Message

**Message Type:** `send_private_message`

**Request:**
```json
{
  "type": "send_private_message",
  "recipientId": "456",
  "content": "Hello!",
  "conversationId": "conv_123_456",  // Optional, auto-generated if not provided
  "replyToMessageId": "msg_789"      // Optional, for replying to messages
}
```

**Success Response (to sender):**
```json
{
  "type": "message_sent",
  "messageId": "msg_abc123",
  "message": {
    "id": "msg_abc123",
    "conversationId": "conv_123_456",
    "senderId": "123",
    "recipientId": "456",
    "username": "john_doe",
    "content": "Hello!",
    "timestamp": "2024-01-15T10:30:00.000Z",
    "isRead": false,
    "replyToMessageId": "msg_789"
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

**Recipient Receives:**
```json
{
  "type": "new_private_message",
  "message": {
    "id": "msg_abc123",
    "conversationId": "conv_123_456",
    "senderId": "123",
    "recipientId": "456",
    "username": "john_doe",
    "content": "Hello!",
    "timestamp": "2024-01-15T10:30:00.000Z",
    "isRead": false,
    "replyToMessageId": "msg_789"
  },
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

**Implementation Details:**
- Content encrypted with AES-256 before database storage
- Message saved to database in transaction
- Conversation metadata upserted
- Recipient lookup via `userConnections` Map
- If recipient offline, event queued in database
- Blocking and following relationships checked
- Rate limiting: 240 messages per 60 seconds per user

**Error Responses:**
```json
{
  "type": "error",
  "message": "Message blocked",
  "details": "You cannot send messages to this user.",
  "code": "USER_BLOCKED"
}
```

```json
{
  "type": "error",
  "message": "Rate limit exceeded",
  "details": "You are sending messages too quickly. Please wait before sending another message.",
  "retryAfter": 30
}
```

---

### Get Private Messages

**Message Type:** `get_private_messages`

**Request:**
```json
{
  "type": "get_private_messages",
  "conversationId": "conv_123_456",
  "limit": 50,        // Optional, default: 50, max: 200
  "offset": 0         // Optional, default: 0
}
```

**Response:**
```json
{
  "type": "private_messages",
  "messages": [
    {
      "id": "msg_abc123",
      "conversationId": "conv_123_456",
      "senderId": "123",
      "recipientId": "456",
      "username": "john_doe",
      "content": "Hello!",
      "timestamp": "2024-01-15T10:30:00.000Z",
      "isRead": true,
      "reactions": [
        {
          "userId": "456",
          "reaction": "👍",
          "username": "jane_doe"
        }
      ],
      "replyToMessageId": null
    }
  ],
  "pagination": {
    "limit": 50,
    "offset": 0,
    "hasMore": true
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

**Implementation Details:**
- First page (offset=0) checked in Redis cache
- Messages loaded from database with pagination
- Encrypted content decrypted on load
- Reactions loaded via subquery
- Deleted messages filtered per-user
- Usernames pre-computed in CTE for performance

---

### Mark Message as Read

**Message Type:** `mark_message_read`

**Request:**
```json
{
  "type": "mark_message_read",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456"
}
```

**Response:**
```json
{
  "type": "message_read",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

**Implementation Details:**
- Updates `is_read` flag in database
- Broadcasts read status to sender
- No response if message already read

---

### Edit Message

**Message Type:** `edit_message` or `edit_private_message`

**Request:**
```json
{
  "type": "edit_message",
  "messageId": "msg_abc123",
  "content": "Hello, updated!",
  "conversationId": "conv_123_456"
}
```

**Success Response:**
```json
{
  "type": "message_edited",
  "messageId": "msg_abc123",
  "message": {
    "id": "msg_abc123",
    "content": "Hello, updated!",
    "editedAt": "2024-01-15T10:35:00.000Z"
  },
  "timestamp": "2024-01-15T10:35:00.000Z"
}
```

**Error Response (timeout):**
```json
{
  "type": "error",
  "message": "Edit window expired",
  "details": "Messages can only be edited within 5 minutes of sending.",
  "code": "EDIT_WINDOW_EXPIRED"
}
```

**Implementation Details:**
- 5-minute edit window enforced
- Only sender can edit
- Content re-encrypted before database update
- Broadcast to all conversation participants
- Original content replaced (not versioned)

---

### Delete Message

**Message Type:** `delete_message` or `delete_private_message`

**Request:**
```json
{
  "type": "delete_message",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456",
  "deleteForEveryone": true  // Optional, default: true
}
```

**Response:**
```json
{
  "type": "message_deleted",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456",
  "deleteForEveryone": true,
  "timestamp": "2024-01-15T10:40:00.000Z"
}
```

**Implementation Details:**
- `deleteForEveryone=true`: Hard delete (removes from database)
- `deleteForEveryone=false`: Soft delete (per-user, stored in `message_deletions` table)
- Broadcast to all participants
- Reactions removed if message hard-deleted

---

### Add Reaction

**Message Type:** `add_reaction`

**Request:**
```json
{
  "type": "add_reaction",
  "messageId": "msg_abc123",
  "reaction": "👍",
  "conversationId": "conv_123_456"
}
```

**Response:**
```json
{
  "type": "reaction_added",
  "messageId": "msg_abc123",
  "reaction": {
    "userId": "123",
    "username": "john_doe",
    "reaction": "👍"
  },
  "timestamp": "2024-01-15T10:45:00.000Z"
}
```

**Implementation Details:**
- One reaction per user per message
- Replacing existing reaction updates it
- Same reaction removes it (toggle behavior)
- Broadcast to all conversation participants
- Saved to `message_reactions` table

---

### Remove Reaction

**Message Type:** `remove_reaction`

**Request:**
```json
{
  "type": "remove_reaction",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456"
}
```

**Response:**
```json
{
  "type": "reaction_removed",
  "messageId": "msg_abc123",
  "userId": "123",
  "timestamp": "2024-01-15T10:46:00.000Z"
}
```

---

### Typing Indicator

**Message Type:** `typing`

**Request:**
```json
{
  "type": "typing",
  "recipientId": "456",
  "isTyping": true
}
```

**Recipient Receives:**
```json
{
  "type": "user_typing",
  "userId": "123",
  "username": "john_doe",
  "isTyping": true,
  "timestamp": "2024-01-15T10:50:00.000Z"
}
```

**Implementation Details:**
- Automatically expires after 3 seconds of inactivity
- Broadcast only to recipient
- Not persisted to database

---

### Start Conversation

**Message Type:** `start_conversation`

**Request:**
```json
{
  "type": "start_conversation",
  "recipientId": "456"
}
```

**Response:**
```json
{
  "type": "conversation_started",
  "conversationId": "conv_123_456",
  "participant": {
    "userId": "456",
    "username": "jane_doe"
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

---

### Delete Conversation

**Message Type:** `delete_conversation`

**Request:**
```json
{
  "type": "delete_conversation",
  "conversationId": "conv_123_456"
}
```

**Response:**
```json
{
  "type": "conversation_deleted",
  "conversationId": "conv_123_456",
  "timestamp": "2024-01-15T10:55:00.000Z"
}
```

**Implementation Details:**
- Soft delete (per-user)
- Stored in `conversation_deletions` table
- Messages remain in database
- User won't see conversation in list

---

### Pin Message

**Message Type:** `pin_message`

**Request:**
```json
{
  "type": "pin_message",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456"
}
```

**Response:**
```json
{
  "type": "message_pinned",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456",
  "timestamp": "2024-01-15T11:00:00.000Z"
}
```

---

### Unpin Message

**Message Type:** `unpin_message`

**Request:**
```json
{
  "type": "unpin_message",
  "messageId": "msg_abc123",
  "conversationId": "conv_123_456"
}
```

---

## Group Messaging

### Create Group

**Message Type:** `create_group`

**Request:**
```json
{
  "type": "create_group",
  "groupName": "Project Team",
  "groupDescription": "Team chat for project collaboration",
  "initialMembers": ["456", "789"],  // Optional
  "maxMembers": 50                    // Optional, default: 50
}
```

**Response:**
```json
{
  "type": "group_created",
  "group": {
    "groupId": "group_xyz789",
    "groupName": "Project Team",
    "groupDescription": "Team chat for project collaboration",
    "createdBy": "123",
    "maxMembers": 50,
    "memberCount": 3,
    "createdAt": "2024-01-15T11:00:00.000Z"
  },
  "members": ["123", "456", "789"],
  "timestamp": "2024-01-15T11:00:00.000Z"
}
```

**Broadcast to Members:**
```json
{
  "type": "group_created",
  "groupId": "group_xyz789",
  "groupName": "Project Team",
  "createdBy": "123",
  "timestamp": "2024-01-15T11:00:00.000Z"
}
```

**Implementation Details:**
- Creator automatically added as admin
- Initial members added in batch transaction
- Group ID generated as `group_${UUID}`
- All operations in single database transaction

---

### Send Group Message

**Message Type:** `send_group_message`

**Request:**
```json
{
  "type": "send_group_message",
  "groupId": "group_xyz789",
  "content": "Hello team! @jane_doe",
  "replyToMessageId": "msg_abc123"  // Optional
}
```

**Response (to sender):**
```json
{
  "type": "group_message_sent",
  "message": {
    "id": "msg_group123",
    "groupId": "group_xyz789",
    "senderId": "123",
    "username": "john_doe",
    "content": "Hello team! @jane_doe",
    "timestamp": "2024-01-15T11:05:00.000Z",
    "mentions": [
      {
        "type": "user",
        "userId": "456",
        "username": "jane_doe"
      }
    ]
  },
  "timestamp": "2024-01-15T11:05:00.000Z"
}
```

**Broadcast to Members:**
```json
{
  "type": "new_group_message",
  "message": {
    "id": "msg_group123",
    "groupId": "group_xyz789",
    "senderId": "123",
    "username": "john_doe",
    "content": "Hello team! @jane_doe",
    "timestamp": "2024-01-15T11:05:00.000Z",
    "mentions": [...]
  },
  "timestamp": "2024-01-15T11:05:00.000Z"
}
```

**Mention Notification:**
```json
{
  "type": "mentioned_in_group",
  "messageId": "msg_group123",
  "groupId": "group_xyz789",
  "message": {...},
  "mentionedBy": "123",
  "mentionedByUsername": "john_doe",
  "timestamp": "2024-01-15T11:05:00.000Z"
}
```

**Implementation Details:**
- @mentions parsed from content
- Supports `@username`, `@everyone`, `@all`
- Broadcast to all group members
- Offline members receive via event queue
- Message saved to `private_messages` table with `group_id`

---

### Get Group Messages

**Message Type:** `get_group_messages`

**Request:**
```json
{
  "type": "get_group_messages",
  "groupId": "group_xyz789",
  "limit": 50,
  "offset": 0
}
```

**Response:**
```json
{
  "type": "group_messages",
  "messages": [...],
  "pagination": {
    "limit": 50,
    "offset": 0,
    "hasMore": true
  },
  "timestamp": "2024-01-15T11:10:00.000Z"
}
```

---

### Get User Groups

**Message Type:** `get_user_groups`

**Request:**
```json
{
  "type": "get_user_groups"
}
```

**Response:**
```json
{
  "type": "user_groups",
  "groups": [
    {
      "groupId": "group_xyz789",
      "groupName": "Project Team",
      "groupDescription": "Team chat",
      "createdBy": "123",
      "myRole": "admin",
      "memberCount": 5,
      "joinedAt": "2024-01-15T11:00:00.000Z"
    }
  ],
  "timestamp": "2024-01-15T11:15:00.000Z"
}
```

---

### Add Group Member

**Message Type:** `add_group_member`

**Request:**
```json
{
  "type": "add_group_member",
  "groupId": "group_xyz789",
  "userId": "999"
}
```

**Response:**
```json
{
  "type": "member_added",
  "groupId": "group_xyz789",
  "userId": "999",
  "username": "new_member",
  "addedBy": "123",
  "timestamp": "2024-01-15T11:20:00.000Z"
}
```

**Broadcast to Members:**
```json
{
  "type": "member_added",
  "groupId": "group_xyz789",
  "user": {
    "userId": "999",
    "username": "new_member"
  },
  "addedBy": "123",
  "timestamp": "2024-01-15T11:20:00.000Z"
}
```

**Implementation Details:**
- Requires admin role or `can_add_members` permission
- Checks max members limit
- Validates user exists and not blocked
- Broadcast to all members

---

### Remove Group Member

**Message Type:** `remove_group_member`

**Request:**
```json
{
  "type": "remove_group_member",
  "groupId": "group_xyz789",
  "userId": "999"
}
```

**Response:**
```json
{
  "type": "member_removed",
  "groupId": "group_xyz789",
  "userId": "999",
  "removedBy": "123",
  "timestamp": "2024-01-15T11:25:00.000Z"
}
```

---

### Leave Group

**Message Type:** `leave_group`

**Request:**
```json
{
  "type": "leave_group",
  "groupId": "group_xyz789"
}
```

**Response:**
```json
{
  "type": "member_left",
  "groupId": "group_xyz789",
  "userId": "123",
  "timestamp": "2024-01-15T11:30:00.000Z"
}
```

**Implementation Details:**
- Owner cannot leave (must delete group or transfer ownership)
- Broadcast to remaining members

---

### Promote Member

**Message Type:** `promote_member`

**Request:**
```json
{
  "type": "promote_member",
  "groupId": "group_xyz789",
  "userId": "456"
}
```

**Response:**
```json
{
  "type": "member_promoted",
  "groupId": "group_xyz789",
  "userId": "456",
  "newRole": "admin",
  "promotedBy": "123",
  "timestamp": "2024-01-15T11:35:00.000Z"
}
```

**Implementation Details:**
- Only owner can promote
- Promotes member → admin
- Grants `can_add_members` permission

---

### Demote Member

**Message Type:** `demote_member`

**Request:**
```json
{
  "type": "demote_member",
  "groupId": "group_xyz789",
  "userId": "456"
}
```

**Response:**
```json
{
  "type": "member_demoted",
  "groupId": "group_xyz789",
  "userId": "456",
  "newRole": "member",
  "demotedBy": "123",
  "timestamp": "2024-01-15T11:40:00.000Z"
}
```

---

### Update Group Info

**Message Type:** `update_group_info`

**Request:**
```json
{
  "type": "update_group_info",
  "groupId": "group_xyz789",
  "groupName": "Updated Team Name",      // Optional
  "groupDescription": "New description"  // Optional
}
```

**Response:**
```json
{
  "type": "group_info_updated",
  "groupId": "group_xyz789",
  "groupName": "Updated Team Name",
  "groupDescription": "New description",
  "updatedBy": "123",
  "timestamp": "2024-01-15T11:45:00.000Z"
}
```

**Implementation Details:**
- Requires admin role
- Broadcast to all members

---

### Send Announcement

**Message Type:** `send_announcement`

**Request:**
```json
{
  "type": "send_announcement",
  "groupId": "group_xyz789",
  "content": "Important announcement!"
}
```

**Response:**
```json
{
  "type": "announcement_sent",
  "message": {...},
  "timestamp": "2024-01-15T11:50:00.000Z"
}
```

**Implementation Details:**
- Only admins can send announcements
- Broadcast to all members
- Stored as regular group message with special flag

---

### Delete Group

**Message Type:** `delete_group`

**Request:**
```json
{
  "type": "delete_group",
  "groupId": "group_xyz789"
}
```

**Response:**
```json
{
  "type": "group_deleted",
  "groupId": "group_xyz789",
  "deletedBy": "123",
  "timestamp": "2024-01-15T11:55:00.000Z"
}
```

**Implementation Details:**
- Only owner can delete
- Soft delete (marks `is_active = false`)
- Broadcast to all members
- Messages remain in database

---

## System Messages

### Get Online Users

**Message Type:** `get_online_users`

**Request:**
```json
{
  "type": "get_online_users"
}
```

**Response:**
```json
{
  "type": "online_users",
  "users": [
    {
      "userId": "123",
      "username": "john_doe",
      "displayName": "John Doe"
    }
  ],
  "count": 1,
  "timestamp": "2024-01-15T12:00:00.000Z"
}
```

**Implementation Details:**
- Returns users connected to same worker process only
- Cross-worker presence requires Redis pub/sub

---

### Ping

**Message Type:** `ping`

**Request:**
```json
{
  "type": "ping"
}
```

**Response:**
```json
{
  "type": "pong",
  "timestamp": 1705312800000
}
```

**Implementation Details:**
- Resets connection timeout
- Used for keep-alive

---

### Test

**Message Type:** `test`

**Request:**
```json
{
  "type": "test"
}
```

**Response:**
```json
{
  "type": "test_response",
  "message": "Server is responding to test messages",
  "timestamp": 1705312800000
}
```

---

## HTTP Endpoints

### Health Check

**Endpoint:** `GET /health`

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T12:00:00.000Z",
  "uptime": 3600000,
  "connections": {
    "active": 150,
    "peak": 200,
    "limit": 50
  },
  "memory": {
    "rss": 250,
    "heapUsed": 180,
    "heapTotal": 200
  },
  "database": {
    "connected": true,
    "poolSize": 10,
    "idleConnections": 5
  },
  "redis": {
    "connected": true
  },
  "metrics": {
    "messagesSent": 5000,
    "messagesReceived": 4800,
    "errors": 2,
    "averageResponseTime": 15
  }
}
```

---

### Readiness Probe

**Endpoint:** `GET /ready`

**Response:**
```json
{
  "ready": true,
  "components": {
    "database": "healthy",
    "websocket": "healthy"
  },
  "timestamp": "2024-01-15T12:00:00.000Z"
}
```

**Implementation Details:**
- Returns 503 if database unavailable
- Used by Kubernetes/orchestration systems

---

### Liveness Probe

**Endpoint:** `GET /live`

**Response:**
```json
{
  "alive": true,
  "uptime": 3600000,
  "timestamp": "2024-01-15T12:00:00.000Z"
}
```

---

### Metrics

**Endpoint:** `GET /metrics`

**Response:**
```json
{
  "connections": 150,
  "messagesSent": 5000,
  "messagesReceived": 4800,
  "errors": 2,
  "startTime": "2024-01-15T11:00:00.000Z",
  "activeUsers": 120,
  "conversations": 80,
  "peakConcurrentUsers": 200,
  "averageResponseTime": 15,
  "totalDataTransferred": 104857600,
  "rateLimitHits": 5
}
```

---

### Get Conversations (HTTP)

**Endpoint:** `GET /api/conversations?limit=50&offset=0`

**Headers:**
```
Authorization: Bearer <JWT_TOKEN>
```

**Response:**
```json
{
  "status": "ok",
  "conversations": [
    {
      "conversationId": "conv_123_456",
      "participant": {
        "userId": "456",
        "username": "jane_doe"
      },
      "lastMessage": {
        "id": "msg_abc123",
        "content": "Hello!",
        "timestamp": "2024-01-15T10:30:00.000Z"
      },
      "unreadCount": 2
    }
  ],
  "pagination": {
    "limit": 50,
    "offset": 0,
    "count": 1
  },
  "timestamp": "2024-01-15T12:00:00.000Z"
}
```

---

### Batch Profile Pictures

**Endpoint:** `POST /api/profile-pictures/batch`

**Headers:**
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
```

**Request:**
```json
{
  "items": [
    {
      "type": "user",
      "id": 123
    },
    {
      "type": "community",
      "id": 456
    }
  ]
}
```

**Response:**
```json
{
  "status": "ok",
  "items": [
    {
      "type": "user",
      "id": 123,
      "hasPicture": true,
      "default": false,
      "url": "https://example.com/pfp/user/123"
    },
    {
      "type": "community",
      "id": 456,
      "hasPicture": false,
      "default": true
    }
  ],
  "timestamp": "2024-01-15T12:00:00.000Z"
}
```

**Implementation Details:**
- Max batch size: 50 items
- Uses in-memory cache (5-minute TTL)
- Falls back to HTTP request to main backend
- Reduces database queries by 80-95%

---

## Error Handling

### Error Response Format

All errors follow this format:

```json
{
  "type": "error",
  "message": "Human-readable error message",
  "details": "Detailed error information",
  "code": "ERROR_CODE",  // Optional
  "retryAfter": 30,      // Optional, seconds
  "requiresAuth": true,  // Optional
  "errorId": "uuid"      // Optional, for tracking
}
```

### Common Error Codes

- `AUTH_REQUIRED` - Authentication required
- `AUTH_FAILED` - Authentication failed
- `USER_BLOCKED` - User is blocked
- `RATE_LIMIT_EXCEEDED` - Rate limit exceeded
- `EDIT_WINDOW_EXPIRED` - Edit window expired
- `PERMISSION_DENIED` - Insufficient permissions
- `RESOURCE_NOT_FOUND` - Resource not found
- `MAX_CONNECTIONS_EXCEEDED` - Server at capacity
- `VALIDATION_ERROR` - Invalid input
- `INTERNAL_ERROR` - Internal server error

---

## Authentication

### JWT Token Structure

JWT tokens are issued by the main backend and validated by the chat server using shared `JWT_SECRET`.

**Token Payload:**
```json
{
  "userId": "123",
  "username": "john_doe",
  "jti": "token-id-uuid",
  "exp": 1705316400,
  "iat": 1705312800
}
```

**Validation Process:**
1. Verify signature (HS256)
2. Check expiration
3. Verify JTI not in revocation table
4. Fetch user details from database
5. Establish session

---

## Rate Limiting

### Limits

- **Messages per user:** 240 messages per 60 seconds
- **Connections per IP:** 10 connections per IP
- **Payload size:** 1MB max per message
- **Batch size:** 50 items max for batch operations

### Rate Limit Response

```json
{
  "type": "error",
  "message": "Rate limit exceeded",
  "details": "You are sending messages too quickly. Please wait before sending another message.",
  "code": "RATE_LIMIT_EXCEEDED",
  "retryAfter": 30
}
```

---

## Message Ordering

Messages are delivered in chronological order based on database `created_at` timestamp. Events from different workers may arrive out of order initially, but database queue ensures eventual consistency.

---

## Implementation Notes

### Connection Handling
- WebSocket connections tracked per worker process
- Multiple devices per user supported
- Grace period: 60 seconds before marking offline
- Automatic reconnection with event queue delivery

### Message Encryption
- AES-256 encryption for message content
- IV (Initialization Vector) stored separately
- Encryption key derived from environment variable
- Content encrypted before database storage

### Database Transactions
- All message operations use transactions
- Rollback on any error
- Connection pool: 10 connections per worker
- Shared database across all workers

### Caching Strategy
- Redis cache for first page of messages (TTL: 1 hour)
- In-memory cache for active conversations (last 50-100 messages)
- Cache invalidation on edit/delete
- Fallback to database if cache unavailable

### Cross-Worker Communication
- Messages to users on different workers queued in database
- Event queue loaded on reconnection
- Presence only visible within same worker (limitation)
- Redis pub/sub can be added for cross-worker presence

