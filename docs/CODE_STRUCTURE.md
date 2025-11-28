# 🏗️ Code Structure & Implementation Guide

Complete guide to understanding the codebase structure, how components connect, and the execution flow. Perfect for understanding the actual implementation!

## 📖 Table of Contents

1. [Codebase Overview](#codebase-overview)
2. [File Structure](#file-structure)
3. [Entry Point: server.js](#entry-point-serverjs)
4. [Connection Flow](#connection-flow)
5. [Message Flow](#message-flow)
6. [Component Connections](#component-connections)
7. [Data Structures](#data-structures)
8. [Execution Flow Examples](#execution-flow-examples)

---

## Codebase Overview

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    server.js (Entry Point)               │
│  - Creates HTTP/HTTPS server                             │
│  - Creates WebSocket server                              │
│  - Initializes database & Redis                          │
│  - Sets up HTTP endpoints                                │
│  - Handles WebSocket connections                         │
└──────────────┬──────────────────────────────────────────┘
               │
               ├──────────────────────────────────────────┐
               │                                          │
               ↓                                          ↓
┌──────────────────────────────┐    ┌──────────────────────────────┐
│  connectionHandler.js        │    │  httpEndpoints.js            │
│  - Handles new connections   │    │  - Health check              │
│  - Validates IP/rate limits  │    │  - Profile pictures          │
│  - Manages timeouts          │    │  - Community sync            │
│  - Routes to messageRouter   │    │                              │
└──────────────┬───────────────┘    └──────────────────────────────┘
               │
               ↓
┌──────────────────────────────┐
│  messageRouter.js            │
│  - Routes by message type    │
│  - Calls appropriate handler │
└──────────────┬───────────────┘
               │
       ┌───────┴───────┐
       │               │
       ↓               ↓
┌──────────────┐  ┌──────────────┐
│messageHandlers│  │groupHandlers │
│- Private DMs │  │- Group chats │
│- Reactions   │  │- Group mgmt  │
│- Typing      │  │- Announcements│
└──────┬───────┘  └──────┬───────┘
       │                 │
       └────────┬────────┘
                │
                ↓
┌──────────────────────────────┐
│  database/                   │
│  - messageOperations.js      │
│  - groupOperations.js        │
│  - communityOperations.js    │
└──────────────────────────────┘
```

---

## File Structure

### Root Directory

```
chat-server/
├── server.js                    # Main entry point
├── config/                      # Configuration files
│   ├── database.js             # Database connection pool
│   └── redis.js                # Redis connection
├── handlers/                    # Request handlers
│   ├── connectionHandler.js    # WebSocket connection management
│   ├── messageRouter.js        # Routes messages to handlers
│   ├── messageHandlers.js     # Private message handlers
│   ├── groupHandlers.js       # Group chat handlers
│   └── httpEndpoints.js       # HTTP REST endpoints
├── database/                    # Database operations
│   ├── messageOperations.js    # Message CRUD operations
│   ├── groupOperations.js      # Group CRUD operations
│   └── communityOperations.js  # Community operations
├── security/                    # Security utilities
│   ├── jwtUtils.js             # JWT verification
│   ├── inputValidator.js      # Input validation
│   ├── rateLimiter.js          # Rate limiting
│   └── encryption.js           # Message encryption
└── utils/                       # Utility functions
    └── profilePictureUtils.js  # Profile picture caching
```

---

## Entry Point: server.js

### What It Does

**server.js** is the main entry point that:
1. Creates the HTTP/HTTPS server
2. Creates the WebSocket server
3. Initializes database and Redis
4. Sets up all handlers
5. Starts listening for connections

### Code Breakdown

**1. Imports and Configuration:**
```javascript
// server.js (lines 1-22)
require('dotenv').config();

const WebSocket = require('ws');
const http = require('http');
const https = require('https');

// Import database and Redis
const { dbPool, initializeDatabase } = require('./config/database');
const { initializeRedis } = require('./config/redis');

// Import handlers
const { setupHttpEndpoints } = require('./handlers/httpEndpoints');
const { handleWebSocketConnection } = require('./handlers/connectionHandler');

// Import security
const RateLimiter = require('./security/rateLimiter');
```

**What this does:**
- Loads environment variables (`.env` file)
- Imports required modules
- Imports database and Redis configs
- Imports handler functions
- Creates rate limiter instance

**2. Create HTTP/HTTPS Server:**
```javascript
// server.js (lines 59-83)
let server;

if (TLS_KEY && TLS_CERT) {
  // Create HTTPS server (secure)
  server = https.createServer(tlsOptions);
} else {
  // Create HTTP server (development)
  server = http.createServer(HTTP_SERVER_CONFIG);
}
```

**What this does:**
- Checks if TLS certificates exist
- Creates HTTPS server if certificates found
- Falls back to HTTP server if not
- Applies performance tuning

**3. Create WebSocket Server:**
```javascript
// server.js (lines 85-105)
const wss = new WebSocket.Server({ 
  server,                    // Attach to HTTP server
  clientTracking: true,      // Track all clients
  perMessageDeflate: true,   // Enable compression
  maxPayload: 1048576        // 1MB max message size
});
```

**What this does:**
- Creates WebSocket server attached to HTTP server
- Enables client tracking
- Enables message compression
- Sets maximum message size

**4. Initialize Data Structures:**
```javascript
// server.js (lines 107-116)
const clients = new Map();              // clientId -> { ws, user, clientIp }
const userConnections = new Map();      // userId -> clientId
const conversations = new Map();        // conversationId -> { participants, messages }
const groups = new Map();               // groupId -> { name, members, messages }
const ipConnections = new Map();        // ip -> connection count
const messageRateLimit = new Map();     // userId -> message timestamps
const sessions = new Map();             // sessionId -> session data
const pendingOfflineTimers = new Map(); // userId -> timeout timer
const eventQueues = new Map();          // userId -> queued events
```

**What this does:**
- Creates in-memory data structures
- All handlers share these Maps
- Stores active connections, conversations, groups
- Tracks rate limits and sessions

**5. Initialize Services:**
```javascript
// server.js (lines 133-150)
async function initializeServices() {
  await initializeDatabase();  // Connect to PostgreSQL
  await initializeRedis();     // Connect to Redis (optional)
}
```

**What this does:**
- Connects to PostgreSQL database
- Connects to Redis cache (if available)
- Sets up connection pools

**6. Set Up HTTP Endpoints:**
```javascript
// server.js (lines 160-165)
setupHttpEndpoints(
  server,           // HTTP server
  wss,              // WebSocket server
  dbPool,           // Database pool
  getDbConnected,   // Database status function
  metrics,          // Metrics object
  clients,          // Clients Map
  userConnections   // User connections Map
);
```

**What this does:**
- Registers HTTP endpoints (health, profile pictures, etc.)
- Passes shared data structures to HTTP handler

**7. Handle WebSocket Connections:**
```javascript
// server.js (lines 167-180)
wss.on('connection', (ws, req) => {
  handleWebSocketConnection(
    ws,                    // WebSocket connection
    req,                   // HTTP request
    clients,               // Clients Map
    userConnections,       // User connections Map
    conversations,         // Conversations Map
    groups,                // Groups Map
    ipConnections,         // IP connections Map
    messageRateLimit,      // Rate limit Map
    sessions,              // Sessions Map
    metrics,               // Metrics object
    rateLimiter,           // Rate limiter instance
    pendingOfflineTimers,  // Offline timers Map
    eventQueues            // Event queues Map
  );
});
```

**What this does:**
- Listens for new WebSocket connections
- Calls `handleWebSocketConnection` for each new connection
- Passes all shared data structures

**8. Start Server:**
```javascript
// server.js (lines 182-190)
server.listen(PORT, () => {
  logger.info(`Server running on port ${PORT}`);
  logger.info(`WebSocket server ready`);
});
```

**What this does:**
- Starts HTTP server listening on port
- WebSocket server automatically starts (attached to HTTP server)

---

## Connection Flow

### Step-by-Step: New Client Connects

**1. Client Initiates Connection:**
```javascript
// Client-side
const ws = new WebSocket('ws://localhost:3001');
```

**2. server.js Receives Connection:**
```javascript
// server.js (line 167)
wss.on('connection', (ws, req) => {
  // ws = WebSocket connection object
  // req = HTTP request object
  handleWebSocketConnection(ws, req, ...);
});
```

**3. connectionHandler.js Processes Connection:**
```javascript
// connectionHandler.js (lines 48-79)
async function handleWebSocketConnection(ws, req, clients, ...) {
  // 1. Generate unique client ID
  const clientId = crypto.randomUUID();
  
  // 2. Get client IP address
  const clientIp = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
  
  // 3. Validate IP address
  if (!SecurityUtils.validateIP(clientIp)) {
    ws.close(1008, 'Invalid IP address');
    return;
  }
  
  // 4. Check connection limit per IP
  if (!checkConnectionLimit(clientIp, ipConnections)) {
    ws.close(1008, 'Connection limit exceeded');
    return;
  }
  
  // 5. Check rate limit
  if (!rateLimiter.checkIPRateLimit(clientIp)) {
    ws.close(1008, 'Rate limit exceeded');
    return;
  }
  
  // 6. Store client
  const clientData = {
    ws,                    // WebSocket connection
    clientIp,              // Client's IP address
    lastActivity: Date.now() // Last activity timestamp
  };
  clients.set(clientId, clientData);
}
```

**4. Set Up Message Handler:**
```javascript
// connectionHandler.js (lines 119-180)
ws.on('message', async (data) => {
  // 1. Parse JSON message
  const message = JSON.parse(data);
  
  // 2. Reset connection timeout
  resetConnectionTimeout(message.type);
  
  // 3. Route to messageRouter
  await handleMessage(ws, message, clientId, clients, ...);
});
```

**5. messageRouter.js Routes Message:**
```javascript
// messageRouter.js (lines 43-48)
async function handleMessage(ws, message, clientId, clients, ...) {
  const { type, ...data } = message;
  
  switch (type) {
    case 'authenticate':
      await handleAuthentication(ws, data, clientId, clients, ...);
      break;
    case 'send_private_message':
      await handleSendPrivateMessage(ws, data, clientId, clients, ...);
      break;
    // ... more cases
  }
}
```

---

## Message Flow

### Example: Sending a Private Message

**1. Client Sends Message:**
```javascript
// Client-side
ws.send(JSON.stringify({
  type: 'send_private_message',
  recipientId: '456',
  content: 'Hello!'
}));
```

**2. connectionHandler Receives:**
```javascript
// connectionHandler.js (line 119)
ws.on('message', async (data) => {
  const message = JSON.parse(data);
  // message = { type: 'send_private_message', recipientId: '456', content: 'Hello!' }
  
  await handleMessage(ws, message, clientId, clients, ...);
});
```

**3. messageRouter Routes:**
```javascript
// messageRouter.js (line 62)
case 'send_private_message':
  await handleSendPrivateMessage(ws, data, clientId, clients, ...);
  break;
```

**4. messageHandlers Processes:**
```javascript
// messageHandlers.js
async function handleSendPrivateMessage(ws, data, clientId, clients, ...) {
  // 1. Get client data
  const client = clients.get(clientId);
  
  // 2. Check authentication
  if (!client || !client.user) {
    ws.send(JSON.stringify({ type: 'error', message: 'Not authenticated' }));
    return;
  }
  
  // 3. Validate input
  const validation = InputValidator.validateMessage(data);
  if (!validation.isValid) {
    ws.send(JSON.stringify({ type: 'error', message: validation.errors[0] }));
    return;
  }
  
  // 4. Check blocking
  const isBlocked = await checkIfBlocked(client.user.userId, data.recipientId);
  if (isBlocked) {
    ws.send(JSON.stringify({ type: 'error', message: 'User is blocked' }));
    return;
  }
  
  // 5. Create message object
  const message = {
    id: crypto.randomUUID(),
    conversationId: generateConversationId(client.user.userId, data.recipientId),
    senderId: client.user.userId,
    recipientId: data.recipientId,
    content: data.content,
    timestamp: new Date().toISOString()
  };
  
  // 6. Save to database
  await savePrivateMessageToDatabase(message, dbPool);
  
  // 7. Store in memory cache
  const conversation = conversations.get(message.conversationId);
  if (conversation) {
    conversation.messages.push(message);
  }
  
  // 8. Send confirmation to sender
  ws.send(JSON.stringify({
    type: 'message_sent',
    messageId: message.id
  }));
  
  // 9. Send to recipient if online
  const recipientClientId = userConnections.get(data.recipientId);
  if (recipientClientId) {
    const recipientClient = clients.get(recipientClientId);
    if (recipientClient && recipientClient.ws.readyState === WebSocket.OPEN) {
      recipientClient.ws.send(JSON.stringify({
        type: 'new_private_message',
        message: message
      }));
    }
  }
}
```

**5. Database Operation:**
```javascript
// database/messageOperations.js
async function savePrivateMessageToDatabase(message, dbPool) {
  const client = await dbPool.connect();
  try {
    await client.query('BEGIN');
    
    // Save message
    await client.query(
      `INSERT INTO private_messages 
       (message_id, conversation_id, sender_id, recipient_id, content) 
       VALUES ($1, $2, $3, $4, $5)`,
      [message.id, message.conversationId, message.senderId, 
       message.recipientId, message.content]
    );
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

## Component Connections

### How Files Connect

**1. server.js → connectionHandler.js:**
```javascript
// server.js
const { handleWebSocketConnection } = require('./handlers/connectionHandler');

wss.on('connection', (ws, req) => {
  handleWebSocketConnection(ws, req, clients, userConnections, ...);
});
```

**2. connectionHandler.js → messageRouter.js:**
```javascript
// connectionHandler.js
const { handleMessage } = require('./messageRouter');

ws.on('message', async (data) => {
  const message = JSON.parse(data);
  await handleMessage(ws, message, clientId, clients, ...);
});
```

**3. messageRouter.js → messageHandlers.js:**
```javascript
// messageRouter.js
const {
  handleAuthentication,
  handleSendPrivateMessage,
  handleAddReaction,
  // ... more handlers
} = require('./messageHandlers');

switch (type) {
  case 'send_private_message':
    await handleSendPrivateMessage(ws, data, clientId, clients, ...);
    break;
}
```

**4. messageHandlers.js → database/messageOperations.js:**
```javascript
// messageHandlers.js
const { savePrivateMessageToDatabase } = require('../database/messageOperations');

async function handleSendPrivateMessage(...) {
  await savePrivateMessageToDatabase(message, dbPool);
}
```

**5. messageHandlers.js → security/jwtUtils.js:**
```javascript
// messageHandlers.js
const { verifyJWTWithBackend } = require('../security/jwtUtils');

async function handleAuthentication(...) {
  const result = await verifyJWTWithBackend(token, clientIp, dbPool);
  if (result.valid) {
    client.user = result.user;
  }
}
```

---

## Data Structures

### Shared Maps (Passed to All Handlers)

**1. clients Map:**
```javascript
// Structure: Map<clientId, { ws, user, clientIp, lastActivity, connectionTimeout }>
clients.set('client_123', {
  ws: WebSocket,              // WebSocket connection
  user: { id: '456', ... },   // Authenticated user (null if not authenticated)
  clientIp: '192.168.1.1',   // Client's IP address
  lastActivity: 1234567890,  // Timestamp of last activity
  connectionTimeout: Timer   // Timeout timer
});
```

**2. userConnections Map:**
```javascript
// Structure: Map<userId, clientId>
// Maps user ID to their active client connection
userConnections.set('456', 'client_123');
// User 456 is connected via client_123
```

**3. conversations Map:**
```javascript
// Structure: Map<conversationId, { participants: Set, messages: [] }>
conversations.set('conv_123_456', {
  participants: new Set(['123', '456']),  // User IDs in conversation
  messages: [                             // Last 50-500 messages
    { id: 'msg_1', content: 'Hello', ... },
    { id: 'msg_2', content: 'Hi', ... }
  ]
});
```

**4. groups Map:**
```javascript
// Structure: Map<groupId, { name, members: Set, messages: [] }>
groups.set('group_abc', {
  name: 'My Group',
  members: new Set(['123', '456', '789']),  // Member user IDs
  messages: [                                // Last 50-500 messages
    { id: 'msg_1', content: 'Hello', ... }
  ]
});
```

### How Data Flows

**Example: Finding Recipient's Connection:**
```javascript
// 1. Get sender's user ID from client
const senderClient = clients.get(clientId);
const senderUserId = senderClient.user.userId; // '123'

// 2. Find recipient's client ID
const recipientClientId = userConnections.get(recipientId); // 'client_456'

// 3. Get recipient's client
const recipientClient = clients.get(recipientClientId);

// 4. Send message to recipient
if (recipientClient && recipientClient.ws.readyState === WebSocket.OPEN) {
  recipientClient.ws.send(JSON.stringify({
    type: 'new_private_message',
    message: message
  }));
}
```

---

## Execution Flow Examples

### Example 1: User Authenticates

```
1. Client connects
   ↓
2. server.js → connectionHandler.js
   ↓
3. connectionHandler.js stores client in clients Map
   ↓
4. Client sends: { type: 'authenticate', token: '...' }
   ↓
5. connectionHandler.js → messageRouter.js
   ↓
6. messageRouter.js → messageHandlers.js (handleAuthentication)
   ↓
7. messageHandlers.js → security/jwtUtils.js (verifyJWTWithBackend)
   ↓
8. jwtUtils.js → config/database.js (query database)
   ↓
9. jwtUtils.js returns user data
   ↓
10. messageHandlers.js stores user in client object
   ↓
11. messageHandlers.js updates userConnections Map
   ↓
12. messageHandlers.js sends auth_success to client
```

### Example 2: Send Group Message

```
1. Client sends: { type: 'send_group_message', groupId: '...', content: '...' }
   ↓
2. connectionHandler.js receives message
   ↓
3. connectionHandler.js → messageRouter.js
   ↓
4. messageRouter.js → groupHandlers.js (handleSendGroupMessage)
   ↓
5. groupHandlers.js checks:
   - Authentication
   - Group membership
   - Permissions
   ↓
6. groupHandlers.js → database/groupOperations.js (saveGroupMessage)
   ↓
7. groupOperations.js saves to PostgreSQL
   ↓
8. groupHandlers.js stores in groups Map (in-memory)
   ↓
9. groupHandlers.js finds all group members from groups Map
   ↓
10. groupHandlers.js sends message to all online members:
    - Loop through group.members Set
    - Find each member's clientId from userConnections Map
    - Get client from clients Map
    - Send message via client.ws
```

### Example 3: HTTP Health Check

```
1. Client makes: GET /health
   ↓
2. server.js HTTP server receives request
   ↓
3. server.js → httpEndpoints.js (setupHttpEndpoints)
   ↓
4. httpEndpoints.js checks URL pathname === '/health'
   ↓
5. httpEndpoints.js reads:
   - metrics object (from server.js)
   - clients Map size
   - dbPool status
   - process.memoryUsage()
   ↓
6. httpEndpoints.js sends JSON response:
   {
     status: 'healthy',
     connections: clients.size,
     memory: process.memoryUsage(),
     database: { connected: true, ... }
   }
```

---

## Key Takeaways

1. **server.js is the entry point**
   - Creates servers
   - Initializes data structures
   - Connects all components

2. **Data structures are shared**
   - Passed to all handlers
   - Stored in memory
   - Accessed by multiple files

3. **Message flow is linear**
   - connectionHandler → messageRouter → specific handler
   - Each step processes and passes along

4. **Handlers are modular**
   - Each handler has specific responsibility
   - Can be tested independently
   - Easy to add new handlers

5. **Database operations are separate**
   - Handlers call database functions
   - Database functions handle SQL
   - Transactions ensure data integrity

6. **Security is layered**
   - Connection-level (IP validation, rate limits)
   - Message-level (authentication, authorization)
   - Data-level (input validation, SQL injection prevention)

---

## Next Steps

- Read [WebSocket Guide](./WEBSOCKET_GUIDE.md) for protocol details
- Read [Endpoints Guide](./ENDPOINTS_GUIDE.md) for all available endpoints
- Read [Security Guide](./SECURITY_GUIDE.md) for security implementation
- Check [API Reference](./API_REFERENCE.md) for endpoint specifications

---

**Questions?** Check the [Main Technical Guide](./TECH_GUIDE_MAIN.md) for more resources!

