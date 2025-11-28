# 📘 JavaScript Basics for Chat Service

Essential JavaScript concepts you need to understand this chat service codebase. Perfect for beginners!

## 📖 Table of Contents

1. [Variables: const, let, var](#variables-const-let-var)
2. [Objects and Maps](#objects-and-maps)
3. [Arrays and Sets](#arrays-and-sets)
4. [Functions: Regular, Arrow, Async](#functions)
5. [Promises and Async/Await](#promises-and-asyncawait)
6. [Modules: require() and module.exports](#modules)
7. [Event Handling](#event-handling)
8. [Error Handling: try/catch](#error-handling)
9. [Destructuring](#destructuring)
10. [Template Literals](#template-literals)

---

## Variables: const, let, var

### const (Constant - Cannot Change)

```javascript
const clientId = crypto.randomUUID();
const MAX_CONNECTIONS = 50000;

// ❌ ERROR: Cannot reassign
// clientId = 'new-id'; // TypeError!
```

**Use for:** Values that never change (config, IDs, constants)

### let (Variable - Can Change)

```javascript
let messageCount = 0;
messageCount = messageCount + 1; // ✅ OK

let userName = 'John';
userName = 'Jane'; // ✅ OK
```

**Use for:** Values that change (counters, user data)

### var (Old Style - Avoid in Modern Code)

```javascript
var oldStyle = 'not recommended';
```

**Why avoid:** Has scope issues. Use `const` or `let` instead.

---

## Objects and Maps

### Objects (Key-Value Pairs)

```javascript
// Creating an object
const user = {
  id: '123',
  username: 'johndoe',
  email: 'john@example.com'
};

// Accessing properties
console.log(user.username); // 'johndoe'
console.log(user['email']); // 'john@example.com'

// Adding properties
user.online = true;

// Object with nested properties
const message = {
  id: 'msg_123',
  sender: {
    id: '123',
    username: 'johndoe'
  },
  content: 'Hello!',
  timestamp: Date.now()
};
```

### Maps (Better for Dynamic Keys)

```javascript
// Creating a Map
const clients = new Map();

// Storing data
clients.set('client_123', { ws: socket, userId: '456' });
clients.set('client_456', { ws: socket2, userId: '789' });

// Getting data
const clientData = clients.get('client_123');
console.log(clientData.userId); // '456'

// Checking if exists
if (clients.has('client_123')) {
  console.log('Client exists!');
}

// Deleting
clients.delete('client_123');

// Size
console.log(clients.size); // 1

// Iterating
for (const [clientId, data] of clients) {
  console.log(`${clientId}: ${data.userId}`);
}
```

**Why Maps in Chat Service?**
- Better performance for frequent additions/deletions
- Keys can be any type (not just strings)
- Easy to iterate
- Size property included

**Example from codebase:**
```javascript
// connectionHandler.js
const clients = new Map(); // Store all WebSocket connections

clients.set(clientId, {
  ws: websocket,
  clientIp: '192.168.1.1',
  lastActivity: Date.now()
});
```

---

## Arrays and Sets

### Arrays (Ordered Lists)

```javascript
// Creating arrays
const messages = ['Hello', 'Hi', 'Hey'];
const numbers = [1, 2, 3, 4, 5];

// Accessing elements
console.log(messages[0]); // 'Hello'
console.log(messages.length); // 3

// Adding elements
messages.push('Goodbye'); // Add to end
messages.unshift('First'); // Add to beginning

// Removing elements
messages.pop(); // Remove from end
messages.shift(); // Remove from beginning

// Finding elements
const index = messages.indexOf('Hi'); // 1

// Filtering
const longMessages = messages.filter(msg => msg.length > 5);

// Mapping (transform each element)
const upperMessages = messages.map(msg => msg.toUpperCase());

// Iterating
messages.forEach(msg => {
  console.log(msg);
});
```

**Example from codebase:**
```javascript
// messageHandlers.js
const conversation = {
  participants: new Set(['user1', 'user2']),
  messages: [] // Array of messages
};

conversation.messages.push(newMessage);
```

### Sets (Unique Values Only)

```javascript
// Creating a Set
const participants = new Set();

// Adding values (duplicates ignored)
participants.add('user_123');
participants.add('user_456');
participants.add('user_123'); // Ignored (already exists)

console.log(participants.size); // 2

// Checking if exists
if (participants.has('user_123')) {
  console.log('User is a participant!');
}

// Removing
participants.delete('user_123');

// Iterating
for (const userId of participants) {
  console.log(userId);
}
```

**Why Sets in Chat Service?**
- Automatically prevents duplicates
- Fast lookup (has())
- Perfect for tracking participants, online users

**Example from codebase:**
```javascript
// messageHandlers.js
conversations.set(conversationId, {
  participants: new Set([senderId, recipientId]),
  messages: []
});
```

---

## Functions

### Regular Functions

```javascript
function sendMessage(content) {
  console.log('Sending:', content);
}

// Call it
sendMessage('Hello!');
```

### Arrow Functions (Modern Style)

```javascript
// One parameter - parentheses optional
const sendMessage = (content) => {
  console.log('Sending:', content);
};

// One line - braces and return optional
const add = (a, b) => a + b;

// No parameters
const connect = () => {
  console.log('Connecting...');
};
```

**Example from codebase:**
```javascript
// connectionHandler.js
const normalizeUserId = (value) => {
  return value != null ? value.toString() : value;
};

// Used as callback
ws.on('message', (data) => {
  handleMessage(data);
});
```

### Async Functions (Handle Asynchronous Operations)

```javascript
// Regular async function
async function getUserDetails(userId) {
  const user = await database.query('SELECT * FROM users WHERE id = $1', [userId]);
  return user;
}

// Arrow async function
const getUserDetails = async (userId) => {
  const user = await database.query('SELECT * FROM users WHERE id = $1', [userId]);
  return user;
};
```

**The `async` keyword:**
- Makes a function return a Promise
- Allows you to use `await` inside
- Automatically wraps return value in Promise

**Example from codebase:**
```javascript
// messageHandlers.js
async function handleSendPrivateMessage(ws, data, clientId) {
  const user = await getUserDetails(data.recipientId);
  // ... rest of code
}
```

---

## Promises and Async/Await

### The Problem: Callback Hell

```javascript
// ❌ BAD: Nested callbacks (hard to read)
getUser(userId, (user) => {
  getMessages(user.id, (messages) => {
    getReactions(messages[0].id, (reactions) => {
      // Too nested!
    });
  });
});
```

### Solution: Promises

```javascript
// Promise-based (better)
getUser(userId)
  .then(user => getMessages(user.id))
  .then(messages => getReactions(messages[0].id))
  .then(reactions => {
    console.log(reactions);
  })
  .catch(error => {
    console.error('Error:', error);
  });
```

### Better Solution: Async/Await

```javascript
// ✅ BEST: async/await (easiest to read)
async function loadUserData(userId) {
  try {
    const user = await getUser(userId);
    const messages = await getMessages(user.id);
    const reactions = await getReactions(messages[0].id);
    console.log(reactions);
  } catch (error) {
    console.error('Error:', error);
  }
}
```

### Understanding Promises

```javascript
// Creating a Promise
const fetchUser = (userId) => {
  return new Promise((resolve, reject) => {
    // Simulate async operation
    setTimeout(() => {
      if (userId) {
        resolve({ id: userId, name: 'John' }); // Success
      } else {
        reject(new Error('Invalid user ID')); // Error
      }
    }, 1000);
  });
};

// Using it with async/await
async function getUser(userId) {
  try {
    const user = await fetchUser(userId);
    console.log(user.name); // 'John'
  } catch (error) {
    console.error(error.message); // 'Invalid user ID'
  }
}
```

**Example from codebase:**
```javascript
// database.js
const dbPool = new Pool({ /* config */ });

// query() returns a Promise
async function getUserDetails(userId) {
  const result = await dbPool.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
  );
  return result.rows[0];
}
```

---

## Modules

### Exporting (Making Code Available)

```javascript
// messageHandlers.js
function handleSendMessage(ws, data) {
  // ... code
}

function handleReceiveMessage(ws, data) {
  // ... code
}

// Export multiple functions
module.exports = {
  handleSendMessage,
  handleReceiveMessage
};

// Or export single function
module.exports = handleSendMessage;
```

### Importing (Using Code from Other Files)

```javascript
// server.js
// Import single function
const { handleWebSocketConnection } = require('./handlers/connectionHandler');

// Import entire module
const messageHandlers = require('./handlers/messageHandlers');
messageHandlers.handleSendMessage(ws, data);

// Import with alias
const { handleWebSocketConnection: handleConnection } = require('./handlers/connectionHandler');
```

**Example from codebase:**
```javascript
// server.js
const { handleWebSocketConnection } = require('./handlers/connectionHandler');
const { setupHttpEndpoints } = require('./handlers/httpEndpoints');

// connectionHandler.js
const { handleMessage } = require('./messageRouter');
const { broadcastToAll } = require('./messageHandlers');
```

### ES6 Modules (Alternative Syntax)

```javascript
// Modern syntax (not used in this codebase, but good to know)
export function handleSendMessage() { }
export default function handler() { }

// Import
import { handleSendMessage } from './handlers/messageHandlers';
import handler from './handlers/messageHandlers';
```

---

## Event Handling

### Browser Events (Frontend)

```javascript
// Click event
button.addEventListener('click', () => {
  console.log('Button clicked!');
});

// WebSocket events
websocket.addEventListener('open', () => {
  console.log('Connected!');
});

websocket.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
});
```

### Node.js Events (Backend)

```javascript
// WebSocket events (using 'ws' library)
ws.on('message', (data) => {
  console.log('Received:', data);
});

ws.on('close', () => {
  console.log('Connection closed');
});

ws.on('error', (error) => {
  console.error('Error:', error);
});

// Custom events
const EventEmitter = require('events');
const emitter = new EventEmitter();

emitter.on('userJoined', (userId) => {
  console.log(`User ${userId} joined!`);
});

emitter.emit('userJoined', '123');
```

**Example from codebase:**
```javascript
// connectionHandler.js
ws.on('message', async (data) => {
  try {
    const message = JSON.parse(data);
    await handleMessage(ws, message, clientId, clients);
  } catch (error) {
    console.error('Error processing message:', error);
  }
});

ws.on('close', () => {
  console.log(`Client ${clientId} disconnected`);
  clients.delete(clientId);
});
```

---

## Error Handling

### try/catch Blocks

```javascript
try {
  // Code that might throw an error
  const result = riskyOperation();
  console.log(result);
} catch (error) {
  // Handle the error
  console.error('Something went wrong:', error.message);
}

// Example: Parsing JSON
try {
  const message = JSON.parse(userInput);
  console.log(message);
} catch (error) {
  console.error('Invalid JSON:', error.message);
}
```

### Async/Await with try/catch

```javascript
async function fetchUserData(userId) {
  try {
    const user = await database.query('SELECT * FROM users WHERE id = $1', [userId]);
    return user;
  } catch (error) {
    console.error('Database error:', error.message);
    throw error; // Re-throw to let caller handle it
  }
}

// Using it
try {
  const user = await fetchUserData('123');
} catch (error) {
  console.error('Failed to fetch user:', error);
}
```

**Example from codebase:**
```javascript
// connectionHandler.js
ws.on('message', async (data) => {
  try {
    const message = JSON.parse(data);
    // ... process message
  } catch (jsonError) {
    console.warn('Malformed JSON:', jsonError.message);
    ws.send(JSON.stringify({
      type: 'error',
      message: 'Invalid JSON format'
    }));
  }
});
```

---

## Destructuring

### Object Destructuring

```javascript
const user = {
  id: '123',
  username: 'johndoe',
  email: 'john@example.com'
};

// Extract properties
const { id, username } = user;
console.log(id); // '123'
console.log(username); // 'johndoe'

// Rename while destructuring
const { id: userId, username: name } = user;
console.log(userId); // '123'

// With default values
const { id, username, role = 'user' } = user;
```

### Array Destructuring

```javascript
const messages = ['Hello', 'Hi', 'Hey'];

const [first, second] = messages;
console.log(first); // 'Hello'
console.log(second); // 'Hi'

// Skip elements
const [first, , third] = messages;
console.log(third); // 'Hey'
```

**Example from codebase:**
```javascript
// messageHandlers.js
const { handleWebSocketConnection } = require('./handlers/connectionHandler');
const { dbPool } = require('./config/database');

// Function parameters
async function handleMessage({ type, content, recipientId }) {
  // Instead of message.type, message.content, etc.
}
```

---

## Template Literals

### Regular Strings (Old Way)

```javascript
const name = 'John';
const greeting = 'Hello, ' + name + '!';
```

### Template Literals (New Way)

```javascript
const name = 'John';
const greeting = `Hello, ${name}!`;
```

### Multi-line Strings

```javascript
// Old way (ugly)
const message = 'Line 1\n' +
                'Line 2\n' +
                'Line 3';

// New way (clean)
const message = `Line 1
Line 2
Line 3`;
```

### Expression Evaluation

```javascript
const userId = '123';
const count = 5;

const text = `User ${userId} has ${count} messages`;

// Complex expressions
const text = `User ${user.id} has ${messages.length} message${messages.length !== 1 ? 's' : ''}`;
```

**Example from codebase:**
```javascript
// connectionHandler.js
console.log(`New WebSocket connection: ${clientId} from ${clientIp}`);
console.log(`Connection timeout for client: ${clientId} (no activity for ${CONNECTION_TIMEOUT}ms)`);
```

---

## Key Takeaways

1. **Use `const` for values that don't change, `let` for values that do**
2. **Maps and Sets are better for dynamic collections than Objects/Arrays**
3. **Async/await makes asynchronous code much easier to read**
4. **Always wrap risky operations in try/catch blocks**
5. **Template literals are cleaner than string concatenation**
6. **Modules (`require`/`module.exports`) organize code into separate files**

---

## Practice Exercises

1. Create a Map to store user connections
2. Write an async function to fetch user data
3. Use destructuring to extract properties from an object
4. Handle errors when parsing JSON
5. Create a module that exports multiple functions

---

## Next Steps

- Read [Client-Server Guide](./CLIENT_SERVER_GUIDE.md) to see how these concepts work together
- Read [WebSocket Guide](./WEBSOCKET_GUIDE.md) to understand real-time communication
- Check [Code Examples](./CODE_EXAMPLES.md) for practical examples

---

**Questions?** Check the [Main Technical Guide](./TECH_GUIDE_MAIN.md) for more resources!

