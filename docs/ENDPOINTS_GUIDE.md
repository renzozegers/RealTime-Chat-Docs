# 📡 Endpoints Guide

Complete guide to understanding how endpoints work in this chat service. Learn about WebSocket endpoints, HTTP endpoints, message routing, and request/response flow!

## 📖 Table of Contents

1. [What are Endpoints?](#what-are-endpoints)
2. [WebSocket Endpoints vs HTTP Endpoints](#websocket-endpoints-vs-http-endpoints)
3. [Message Routing](#message-routing)
4. [Request/Response Flow](#requestresponse-flow)
5. [WebSocket Endpoints](#websocket-endpoints)
6. [HTTP Endpoints](#http-endpoints)
7. [Error Handling](#error-handling)
8. [Complete Endpoint List](#complete-endpoint-list)

---

## What are Endpoints?

### Simple Analogy: Restaurant Menu

Think of **endpoints** like a restaurant menu:

- **Menu Item** = Endpoint (what you can order)
- **Order** = Request (what you send)
- **Food** = Response (what you receive)

### In Technical Terms

**Endpoint** = Specific operation the server can perform

**Example:**
```
Client: "I want to send a message"
    ↓
Endpoint: send_private_message
    ↓
Server: "Message sent! Here's confirmation"
```

### Types of Endpoints

**1. WebSocket Endpoints:**
- Real-time bidirectional communication
- Messages sent as JSON
- Server can push data anytime

**2. HTTP Endpoints:**
- Request-response model
- GET, POST, PUT, DELETE methods
- Traditional REST API

---

## WebSocket Endpoints vs HTTP Endpoints

### WebSocket Endpoints (Real-Time)

**Characteristics:**
- ✅ Persistent connection
- ✅ Bidirectional (both sides can send)
- ✅ Real-time (instant delivery)
- ✅ Lower latency

**Use Cases:**
- Sending messages
- Receiving messages
- Typing indicators
- Online status updates

**Example:**
```javascript
// Client sends message
ws.send(JSON.stringify({
  type: 'send_private_message',
  recipientId: '456',
  content: 'Hello!'
}));

// Server responds immediately
ws.on('message', (data) => {
  const response = JSON.parse(data);
  console.log('Received:', response);
});
```

### HTTP Endpoints (Traditional REST)

**Characteristics:**
- ✅ Standard request-response
- ✅ Stateless (no connection maintained)
- ✅ Cacheable
- ✅ Works with load balancers

**Use Cases:**
- Health checks
- Profile pictures
- Community sync
- One-time operations

**Example:**
```javascript
// Client makes HTTP request
fetch('https://chat.example.com/api/profile-picture/123')
  .then(response => response.json())
  .then(data => console.log(data));

// Server responds
server.on('request', (req, res) => {
  if (req.url === '/api/profile-picture/123') {
    res.json({ url: 'https://...' });
  }
});
```

---

## Message Routing

### How Messages Are Routed

**1. Client Sends Message:**
```javascript
// Client sends JSON message
ws.send(JSON.stringify({
  type: 'send_private_message',
  recipientId: '456',
  content: 'Hello!'
}));
```

**2. Server Receives Message:**
```javascript
// connectionHandler.js
ws.on('message', async (data) => {
  try {
    const message = JSON.parse(data);
    // Route to appropriate handler
    await handleMessage(ws, message, clientId, clients);
  } catch (error) {
    // Handle error
  }
});
```

**3. Router Determines Handler:**
```javascript
// messageRouter.js
async function handleMessage(ws, message, clientId, clients) {
  const { type, ...data } = message;
  
  switch (type) {
    case 'send_private_message':
      await handleSendPrivateMessage(ws, data, clientId, clients);
      break;
      
    case 'get_online_users':
      await handleGetOnlineUsers(ws, data, clientId, clients);
      break;
      
    case 'add_reaction':
      await handleAddReaction(ws, data, clientId, clients);
      break;
      
    default:
      ws.send(JSON.stringify({
        type: 'error',
        message: `Unknown message type: ${type}`
      }));
  }
}
```

**4. Handler Processes Request:**
```javascript
// messageHandlers.js
async function handleSendPrivateMessage(ws, data, clientId, clients) {
  // 1. Validate authentication
  if (!clients.get(clientId).authenticated) {
    ws.send(JSON.stringify({ type: 'error', message: 'Not authenticated' }));
    return;
  }
  
  // 2. Validate input
  if (!data.recipientId || !data.content) {
    ws.send(JSON.stringify({ type: 'error', message: 'Missing required fields' }));
    return;
  }
  
  // 3. Process message
  const message = await saveMessageToDatabase(data);
  
  // 4. Send response
  ws.send(JSON.stringify({
    type: 'message_sent',
    messageId: message.id
  }));
  
  // 5. Notify recipient
  const recipientClient = findClientByUserId(data.recipientId);
  if (recipientClient) {
    recipientClient.ws.send(JSON.stringify({
      type: 'new_message',
      message: message
    }));
  }
}
```

---

## Request/Response Flow

### WebSocket Endpoint Flow

**1. Client Connects:**
```
Client → Server: WebSocket connection
Server → Client: Connection established
```

**2. Client Authenticates:**
```
Client → Server: { type: 'authenticate', token: '...' }
Server → Client: { type: 'auth_success', user: {...} }
```

**3. Client Sends Message:**
```
Client → Server: { type: 'send_private_message', ... }
    ↓
Server processes message
    ↓
Server → Client: { type: 'message_sent', ... }
Server → Recipient: { type: 'new_message', ... }
```

### HTTP Endpoint Flow

**1. Client Makes Request:**
```
Client → Server: GET /api/health
```

**2. Server Processes Request:**
```javascript
// httpEndpoints.js
server.on('request', (req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);
  
  if (url.pathname === '/health') {
    // Process request
    const health = {
      status: 'healthy',
      timestamp: new Date().toISOString()
    };
    
    // Send response
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(health));
  }
});
```

**3. Server Sends Response:**
```
Server → Client: { status: 'healthy', ... }
```

---

## WebSocket Endpoints

### Authentication

**Endpoint:** `authenticate`

**Request:**
```json
{
  "type": "authenticate",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response (Success):**
```json
{
  "type": "auth_success",
  "userId": "123",
  "username": "johndoe",
  "email": "john@example.com"
}
```

**Response (Error):**
```json
{
  "type": "auth_error",
  "error": "Invalid token signature or expired"
}
```

### Send Private Message

**Endpoint:** `send_private_message`

**Request:**
```json
{
  "type": "send_private_message",
  "recipientId": "456",
  "content": "Hello!",
  "replyToMessageId": "msg_123"  // Optional
}
```

**Response (to Sender):**
```json
{
  "type": "message_sent",
  "messageId": "msg_789",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Response (to Recipient):**
```json
{
  "type": "new_private_message",
  "messageId": "msg_789",
  "conversationId": "conv_123_456",
  "senderId": "123",
  "content": "Hello!",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Get Online Users

**Endpoint:** `get_online_users`

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
      "userId": "456",
      "username": "janedoe",
      "displayName": "Jane Doe",
      "lastSeen": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### Add Reaction

**Endpoint:** `add_reaction`

**Request:**
```json
{
  "type": "add_reaction",
  "messageId": "msg_123",
  "reaction": "👍"
}
```

**Response:**
```json
{
  "type": "reaction_added",
  "messageId": "msg_123",
  "reaction": {
    "userId": "123",
    "username": "johndoe",
    "reaction": "👍",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

---

## HTTP Endpoints

### Health Check

**Endpoint:** `GET /health`

**Request:**
```
GET /health HTTP/1.1
Host: chat.example.com
```

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00Z",
  "uptime": 3600,
  "connections": 42,
  "memory": {
    "heapUsed": 50.2,
    "heapTotal": 100.5,
    "rss": 150.8
  },
  "database": {
    "connected": true,
    "pool": {
      "totalCount": 5,
      "idleCount": 3,
      "waitingCount": 0
    }
  }
}
```

### Profile Picture

**Endpoint:** `GET /api/profile-picture/:userId`

**Request:**
```
GET /api/profile-picture/123 HTTP/1.1
Host: chat.example.com
```

**Response:**
```json
{
  "userId": "123",
  "profileImageUrl": "https://example.com/profile/123.jpg",
  "hasDefault": false
}
```

### Community Sync

**Endpoint:** `POST /api/community/sync`

**Request:**
```
POST /api/community/sync HTTP/1.1
Host: chat.example.com
Content-Type: application/json
Authorization: Bearer sync_token_here

{
  "communityId": "comm_123",
  "action": "member_joined",
  "userId": "456"
}
```

**Response:**
```json
{
  "status": "success",
  "message": "Community sync completed"
}
```

---

## Error Handling

### Error Response Format

**All errors follow this format:**
```json
{
  "type": "error",
  "message": "Human-readable error message",
  "details": "Additional details (optional)",
  "field": "field_name (optional)",
  "code": "ERROR_CODE (optional)"
}
```

### Common Error Scenarios

**1. Authentication Required:**
```json
{
  "type": "error",
  "message": "Authentication required",
  "code": "AUTH_REQUIRED"
}
```

**2. Invalid Input:**
```json
{
  "type": "error",
  "message": "Missing required field",
  "field": "recipientId",
  "details": "recipientId is required"
}
```

**3. Permission Denied:**
```json
{
  "type": "error",
  "message": "Permission denied",
  "details": "You are not authorized to perform this action",
  "code": "PERMISSION_DENIED"
}
```

**4. Rate Limit Exceeded:**
```json
{
  "type": "error",
  "message": "Rate limit exceeded",
  "details": "Too many requests. Please try again later.",
  "code": "RATE_LIMIT_EXCEEDED"
}
```

### Error Handling in Code

**Client-Side:**
```javascript
ws.on('message', (data) => {
  const message = JSON.parse(data);
  
  if (message.type === 'error') {
    console.error('Error:', message.message);
    // Display error to user
    showError(message.message);
  } else {
    // Handle success
    handleSuccess(message);
  }
});
```

**Server-Side:**
```javascript
try {
  // Process request
  await handleRequest(ws, data, clientId);
} catch (error) {
  // Send error to client
  ws.send(JSON.stringify({
    type: 'error',
    message: error.message,
    details: error.details || ''
  }));
}
```

---

## Complete Endpoint List

### WebSocket Endpoints (26+)

**Authentication:**
- `authenticate` - Authenticate with JWT token

**Direct Messages:**
- `get_online_users` - Get list of online users
- `start_conversation` - Start or retrieve conversation
- `send_private_message` - Send direct message
- `get_private_messages` - Get message history
- `edit_private_message` - Edit your message
- `delete_private_message` - Delete your message
- `delete_conversation` - Delete entire conversation
- `mark_message_read` - Mark message as read
- `typing` - Send typing indicator
- `pin_message` - Pin message to top
- `unpin_message` - Unpin message

**Reactions:**
- `add_reaction` - Add emoji reaction
- `remove_reaction` - Remove emoji reaction

**Group Chats:**
- `create_group` - Create new group
- `send_group_message` - Send group message
- `get_group_messages` - Get group message history
- `get_user_groups` - Get user's groups
- `get_group_members` - Get group members
- `get_group_info` - Get group information
- `add_group_member` - Add member to group
- `remove_group_member` - Remove member from group
- `leave_group` - Leave group
- `update_group_info` - Update group settings
- `delete_group` - Delete group
- `promote_member` - Promote member to admin
- `demote_member` - Demote admin to member
- `send_announcement` - Send group announcement
- `get_pinned_messages` - Get pinned messages

**Communities:**
- `get_user_communities` - Get user's communities
- `get_community_members_list` - Get community members
- `get_community_progress` - Get community progress

**Utilities:**
- `ping` - Keep connection alive
- `test` - Test server response

### HTTP Endpoints (5+)

**Health & Status:**
- `GET /health` - Server health check

**Profile Pictures:**
- `GET /api/profile-picture/:userId` - Get profile picture
- `GET /api/profile-picture/batch` - Get multiple profile pictures

**Community Sync:**
- `POST /api/community/sync` - Sync community changes

---

## Key Takeaways

1. **Endpoints define what the server can do**
   - WebSocket endpoints for real-time operations
   - HTTP endpoints for traditional REST API

2. **Messages are routed by type**
   - Router examines `type` field
   - Calls appropriate handler function

3. **Request/Response flow is consistent**
   - Client sends request
   - Server validates and processes
   - Server sends response

4. **Errors follow standard format**
   - Always include `type: 'error'`
   - Include helpful error messages
   - Optional details and codes

5. **All endpoints require authentication** (except health check)
   - JWT token must be valid
   - User must be authenticated

6. **Input validation is crucial**
   - Validate all user input
   - Return clear error messages

---

## Next Steps

- Read [API Reference](./API_REFERENCE.md) for complete endpoint documentation
- Read [WebSocket Guide](./WEBSOCKET_GUIDE.md) for WebSocket details
- Read [Security Guide](./SECURITY_GUIDE.md) for authentication and authorization
- Check [Code Examples](./CODE_EXAMPLES.md) for implementation examples

---

**Questions?** Check the [Main Technical Guide](./TECH_GUIDE_MAIN.md) for more resources!

