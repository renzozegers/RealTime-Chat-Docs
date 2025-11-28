# 🌐 WebSocket Technical Guide

A comprehensive guide to understanding WebSockets in this chat service. Perfect for beginners and intermediate developers.

## 📖 Table of Contents

1. [What are WebSockets?](#what-are-websockets)
2. [WebSocket vs HTTP](#websocket-vs-http)
3. [How WebSockets Work](#how-websockets-work)
4. [Connection Lifecycle](#connection-lifecycle)
5. [Ping/Pong Keep-Alive](#pingpong-keep-alive)
6. [Message Format](#message-format)
7. [Error Handling](#error-handling)
8. [Implementation in This Chat Service](#implementation-in-this-chat-service)

---

## What are WebSockets?

### The Problem with HTTP

Think of HTTP like sending letters through the mail:
- You send a letter (request)
- You wait for a response (letter back)
- The connection ends
- To send another letter, you start over

**Problem**: For a chat app, you'd need to constantly ask "Do I have new messages?" - this is inefficient and slow!

### WebSocket Solution

WebSockets are like a **phone call**:
- You dial once (connect)
- The line stays open (persistent connection)
- Both sides can talk at any time (bidirectional)
- You hang up when done (disconnect)

**Result**: Instant, real-time communication!

---

## WebSocket vs HTTP

### HTTP Request-Response Model

```
Client: "Give me messages" → [Connection opens]
Server: "Here are messages" → [Connection closes]
Client: "Any new messages?" → [Connection opens again]
Server: "No" → [Connection closes]
Client: "Any new messages?" → [Connection opens again]
... (inefficient!)
```

### WebSocket Persistent Connection

```
Client: [Connects once]
Server: [Connection stays open]
Client: "Any new messages?"
Server: "No"
[Connection still open...]
Server: "Hey! New message for you!" [Server can send anytime!]
Client: "Thanks!"
[Connection stays open...]
```

### Key Differences

| Feature | HTTP | WebSocket |
|---------|------|-----------|
| **Connection** | Opens and closes per request | Stays open |
| **Direction** | Client → Server only | Both directions |
| **Latency** | High (reconnect each time) | Low (always connected) |
| **Use Case** | Web pages, APIs | Chat, gaming, live updates |
| **Overhead** | High (headers every time) | Low (once at start) |

---

## How WebSockets Work

### The Handshake

When a client wants to connect, it first makes an **HTTP request** with special headers:

```
GET /chat HTTP/1.1
Host: chat.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

The server responds:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Status 101** means "I'm switching from HTTP to WebSocket protocol"

After this, the connection is **upgraded** and stays open!

### Connection States

WebSocket connections have 4 states:

1. **CONNECTING (0)**: Still establishing connection
2. **OPEN (1)**: Connection open, ready to send/receive
3. **CLOSING (2)**: Connection is closing
4. **CLOSED (3)**: Connection is closed

```javascript
// In JavaScript:
if (ws.readyState === WebSocket.OPEN) {
  // Connection is ready!
  ws.send("Hello!");
}
```

---

## Connection Lifecycle

### 1. Client Initiates Connection

```javascript
// Client-side JavaScript
const ws = new WebSocket('ws://localhost:3001');

ws.onopen = () => {
  console.log('Connected!');
  // Send authentication
  ws.send(JSON.stringify({
    type: 'authenticate',
    token: 'user_jwt_token_here'
  }));
};
```

### 2. Server Accepts Connection

```javascript
// Server-side (Node.js with 'ws' library)
wss.on('connection', (ws, req) => {
  console.log('New connection!');
  
  // Generate unique client ID
  const clientId = crypto.randomUUID();
  
  // Store connection
  clients.set(clientId, { ws, lastActivity: Date.now() });
});
```

### 3. Message Exchange

**Client sends message:**
```javascript
ws.send(JSON.stringify({
  type: 'send_private_message',
  recipientId: '123',
  content: 'Hello!'
}));
```

**Server receives and processes:**
```javascript
ws.on('message', (data) => {
  const message = JSON.parse(data);
  
  if (message.type === 'send_private_message') {
    // Process message
    handlePrivateMessage(message);
  }
});
```

**Server sends response:**
```javascript
ws.send(JSON.stringify({
  type: 'message_sent',
  messageId: 'msg_123',
  timestamp: Date.now()
}));
```

### 4. Connection Close

```javascript
// Client closes
ws.close();

// Server detects close
ws.on('close', () => {
  console.log('Client disconnected');
  // Clean up resources
  clients.delete(clientId);
});
```

---

## Ping/Pong Keep-Alive

### The Problem: Idle Connections

Imagine you're on a phone call, but neither person speaks for 5 minutes. Is the line still connected? How do you know?

Similarly, WebSocket connections can appear "alive" but actually be dead (network issues, firewalls, etc.).

### Solution: Ping/Pong

**Ping/Pong** is like saying "Are you there?" periodically:

```
Client: [sends ping every 30 seconds]
Server: [responds with pong]
Client: [receives pong - connection is alive!]
```

If the server doesn't respond to ping, the connection is dead.

### Implementation in Chat Service

**Client sends ping:**
```javascript
// Client-side
setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'ping' }));
  }
}, 30000); // Every 30 seconds
```

**Server responds with pong:**
```javascript
// Server-side (messageRouter.js)
case 'ping':
  // Respond immediately
  ws.send(JSON.stringify({ 
    type: 'pong', 
    timestamp: Date.now() 
  }));
  break;
```

**Server also uses ping/pong to reset timeout:**
```javascript
// If server doesn't receive ANY message (including ping) for 5 minutes
// It closes the connection
const CONNECTION_TIMEOUT = 300000; // 5 minutes

let timeout = setTimeout(() => {
  ws.close(); // Close idle connection
}, CONNECTION_TIMEOUT);

// Reset timeout on any message (ping, message, etc.)
ws.on('message', () => {
  clearTimeout(timeout);
  timeout = setTimeout(() => ws.close(), CONNECTION_TIMEOUT);
});
```

### Why Both Sides Use Ping/Pong?

- **Client → Server**: Keeps connection alive, detects server issues
- **Server → Client**: Resets timeout, ensures client is responsive

---

## Message Format

### JSON Messages

All messages in this chat service use **JSON format**:

```json
{
  "type": "send_private_message",
  "recipientId": "456",
  "content": "Hello there!",
  "replyTo": "msg_123"
}
```

### Message Types

**Authentication:**
```json
{
  "type": "authenticate",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Send Message:**
```json
{
  "type": "send_private_message",
  "recipientId": "456",
  "content": "Hello!"
}
```

**Reaction:**
```json
{
  "type": "add_reaction",
  "messageId": "msg_123",
  "reaction": "👍"
}
```

**Ping:**
```json
{
  "type": "ping"
}
```

### Server Responses

**Success:**
```json
{
  "type": "message_sent",
  "messageId": "msg_789",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Error:**
```json
{
  "type": "error",
  "message": "User not found",
  "details": "The recipient ID does not exist"
}
```

---

## Error Handling

### Connection Errors

**Network failure:**
```javascript
ws.onerror = (error) => {
  console.error('WebSocket error:', error);
  // Attempt to reconnect
  setTimeout(connect, 5000);
};
```

**Connection closed unexpectedly:**
```javascript
ws.onclose = (event) => {
  if (event.code === 1006) { // Abnormal closure
    console.log('Connection lost, reconnecting...');
    reconnect();
  }
};
```

### Message Errors

**Invalid JSON:**
```javascript
ws.on('message', (data) => {
  try {
    const message = JSON.parse(data);
    // Process message
  } catch (error) {
    // Send error back to client
    ws.send(JSON.stringify({
      type: 'error',
      message: 'Invalid JSON format'
    }));
  }
});
```

**Invalid message type:**
```javascript
switch (message.type) {
  case 'send_private_message':
    // Handle it
    break;
  default:
    ws.send(JSON.stringify({
      type: 'error',
      message: `Unknown message type: ${message.type}`
    }));
}
```

---

## Implementation in This Chat Service

### Server Setup

```javascript
// server.js
const WebSocket = require('ws');
const http = require('http');

// Create HTTP server
const server = http.createServer();

// Create WebSocket server attached to HTTP server
const wss = new WebSocket.Server({ 
  server,
  clientTracking: true, // Track all clients
  perMessageDeflate: true // Enable compression
});

// Listen for new connections
wss.on('connection', handleWebSocketConnection);
```

### Connection Handler

```javascript
// connectionHandler.js
async function handleWebSocketConnection(ws, req) {
  const clientId = crypto.randomUUID();
  const clientIp = req.connection.remoteAddress;
  
  // Store client
  clients.set(clientId, {
    ws,           // WebSocket connection
    clientIp,     // Client's IP address
    lastActivity: Date.now()
  });
  
  // Set up message handler
  ws.on('message', async (data) => {
    try {
      const message = JSON.parse(data);
      await handleMessage(ws, message, clientId, clients);
    } catch (error) {
      // Handle error
    }
  });
  
  // Handle disconnection
  ws.on('close', () => {
    // Clean up
    clients.delete(clientId);
  });
  
  // Set timeout
  setTimeout(() => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.close(1000, 'Connection timeout');
    }
  }, 300000); // 5 minutes
}
```

### Message Router

```javascript
// messageRouter.js
async function handleMessage(ws, message, clientId, clients) {
  switch (message.type) {
    case 'authenticate':
      await handleAuthenticate(ws, message, clientId);
      break;
      
    case 'send_private_message':
      await handleSendPrivateMessage(ws, message, clientId);
      break;
      
    case 'ping':
      ws.send(JSON.stringify({ type: 'pong' }));
      break;
      
    default:
      ws.send(JSON.stringify({
        type: 'error',
        message: 'Unknown message type'
      }));
  }
}
```

### Client Implementation Example

```javascript
// Client-side JavaScript
class ChatClient {
  constructor(url) {
    this.url = url;
    this.ws = null;
    this.reconnectAttempts = 0;
  }
  
  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onopen = () => {
      console.log('Connected!');
      this.reconnectAttempts = 0;
      // Authenticate
      this.authenticate();
    };
    
    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this.handleMessage(message);
    };
    
    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
    
    this.ws.onclose = () => {
      console.log('Disconnected, reconnecting...');
      this.reconnect();
    };
  }
  
  authenticate() {
    this.ws.send(JSON.stringify({
      type: 'authenticate',
      token: localStorage.getItem('jwt_token')
    }));
  }
  
  sendMessage(recipientId, content) {
    this.ws.send(JSON.stringify({
      type: 'send_private_message',
      recipientId,
      content
    }));
  }
  
  reconnect() {
    const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
    this.reconnectAttempts++;
    setTimeout(() => this.connect(), delay);
  }
}

// Usage
const client = new ChatClient('ws://localhost:3001');
client.connect();
```

---

## Key Takeaways

1. **WebSockets provide persistent, bidirectional connections**
   - Unlike HTTP, the connection stays open
   - Both client and server can send anytime

2. **Ping/Pong keeps connections alive**
   - Prevents idle timeouts
   - Detects dead connections

3. **Messages use JSON format**
   - Easy to parse and debug
   - Type-based routing

4. **Error handling is crucial**
   - Network can fail anytime
   - Always validate messages
   - Implement reconnection logic

5. **Connection lifecycle management**
   - Track all connections
   - Clean up on disconnect
   - Handle timeouts

---

## Next Steps

- Read [Client-Server Guide](./CLIENT_SERVER_GUIDE.md) to understand the architecture
- Read [Endpoints Guide](./ENDPOINTS_GUIDE.md) to see all available endpoints
- Read [Security Guide](./SECURITY_GUIDE.md) to understand authentication

---

**Questions?** Check the [Main Technical Guide](./TECH_GUIDE_MAIN.md) for more resources!

