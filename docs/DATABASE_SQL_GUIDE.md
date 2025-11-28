# 🗄️ Database & SQL Guide

Complete guide to understanding databases and SQL in this chat service. Learn about PostgreSQL, connection pooling, SQL injection prevention, and more!

## 📖 Table of Contents

1. [What is a Database?](#what-is-a-database)
2. [PostgreSQL Basics](#postgresql-basics)
3. [SQL Basics](#sql-basics)
4. [Connection Pooling](#connection-pooling)
5. [SQL Injection Prevention](#sql-injection-prevention)
6. [Transactions](#transactions)
7. [Query Optimization](#query-optimization)
8. [Database Schema](#database-schema)

---

## What is a Database?

### Simple Analogy: Filing Cabinet

Think of a **database** like a filing cabinet:

- **Cabinet** = Database
- **Drawers** = Tables
- **Files** = Rows
- **File labels** = Columns

### Why Use a Database?

**Without Database:**
```
Data stored in memory
    ↓
Server restarts
    ↓
All data lost! 💥
```

**With Database:**
```
Data stored in database
    ↓
Server restarts
    ↓
Data still there! ✅
```

### Databases in Chat Service

**PostgreSQL** stores:
- Messages (permanent storage)
- Users and profiles
- Conversations
- Groups and members
- Reactions and mentions

**Memory** stores:
- Active connections
- Recent messages (cache)
- Online users

---

## PostgreSQL Basics

### What is PostgreSQL?

**PostgreSQL** = Relational database management system (RDBMS)

**Characteristics:**
- ✅ Open source
- ✅ ACID compliant (data integrity)
- ✅ Supports complex queries
- ✅ Handles concurrent connections
- ✅ Production-ready

### Database Connection

**Connection String:**
```
postgresql://user:password@host:port/database
```

**Example:**
```
postgresql://chat_user:secret123@localhost:5432/chat_db
```

**From Environment Variables:**
```javascript
// config/database.js
const dbConfig = {
  host: process.env.DB_HOST || 'localhost',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASS
};
```

---

## SQL Basics

### What is SQL?

**SQL** (Structured Query Language) = Language for querying databases

**Basic SQL Operations:**

**1. SELECT (Read Data):**
```sql
-- Get all users
SELECT * FROM users;

-- Get specific user
SELECT * FROM users WHERE user_id = '123';

-- Get specific columns
SELECT username, email FROM users WHERE user_id = '123';
```

**2. INSERT (Create Data):**
```sql
-- Insert new user
INSERT INTO users (user_id, username, email)
VALUES ('123', 'johndoe', 'john@example.com');
```

**3. UPDATE (Modify Data):**
```sql
-- Update user email
UPDATE users 
SET email = 'newemail@example.com' 
WHERE user_id = '123';
```

**4. DELETE (Remove Data):**
```sql
-- Delete user
DELETE FROM users WHERE user_id = '123';
```

### SQL in Chat Service

**Getting User Details:**
```javascript
// database/messageOperations.js
async function getUserDetails(userId, dbPool) {
  const result = await dbPool.query(
    'SELECT * FROM users WHERE user_id = $1',
    [userId]
  );
  
  return result.rows[0]; // First row (should be only one)
}
```

**Saving Message:**
```javascript
// database/messageOperations.js
async function saveMessage(message, dbPool) {
  await dbPool.query(
    `INSERT INTO private_messages 
     (message_id, conversation_id, sender_id, recipient_id, content) 
     VALUES ($1, $2, $3, $4, $5)`,
    [message.id, message.conversationId, message.senderId, 
     message.recipientId, message.content]
  );
}
```

---

## Connection Pooling

### What is Connection Pooling?

**Problem Without Pooling:**
```
Every query needs new connection
    ↓
Create connection (100-500ms)
    ↓
Execute query (10-50ms)
    ↓
Close connection (10-50ms)
    ↓
Total: 120-600ms per query! ❌
```

**Solution With Pooling:**
```
Reuse existing connections
    ↓
Get connection from pool (< 1ms)
    ↓
Execute query (10-50ms)
    ↓
Return connection to pool (< 1ms)
    ↓
Total: 10-50ms per query! ✅
```

### Connection Pool Configuration

**Pool Settings:**
```javascript
// config/database.js
const dbPool = new Pool({
  max: 10,              // Maximum connections in pool
  min: 2,               // Minimum connections (always open)
  idleTimeoutMillis: 30000,  // Close idle connections after 30s
  connectionTimeoutMillis: 2000,  // Timeout for getting connection
  maxUses: 7500,        // Recycle connection after 7500 uses
  statement_timeout: 30000,  // Query timeout (30 seconds)
  query_timeout: 30000  // Query timeout (30 seconds)
});
```

**Why Pool Settings Matter:**
- **max**: Maximum concurrent connections (prevents overload)
- **min**: Always-open connections (faster first queries)
- **idleTimeoutMillis**: Close unused connections (saves resources)
- **connectionTimeoutMillis**: Fail fast if pool is exhausted

### Pool Lifecycle

**1. Initialization:**
```javascript
// Create pool (no connections yet)
const dbPool = new Pool({ max: 10, min: 2 });
```

**2. First Query:**
```javascript
// Pool creates connections up to min (2)
const result = await dbPool.query('SELECT NOW()');
```

**3. More Queries:**
```javascript
// Pool creates more connections as needed (up to max: 10)
const users = await dbPool.query('SELECT * FROM users');
```

**4. Connection Reuse:**
```javascript
// Existing connections are reused (fast!)
const messages = await dbPool.query('SELECT * FROM messages');
```

**5. Idle Cleanup:**
```javascript
// After 30 seconds of inactivity, excess connections are closed
// (but keeps min: 2 connections open)
```

### Checking Pool Status

**Pool Metrics:**
```javascript
console.log('Pool status:', {
  totalCount: dbPool.totalCount,      // Total connections
  idleCount: dbPool.idleCount,        // Idle connections
  waitingCount: dbPool.waitingCount   // Queries waiting for connection
});
```

**Health Check:**
```javascript
// httpEndpoints.js
const health = {
  database: {
    connected: getDbConnected(),
    pool: {
      totalCount: dbPool.totalCount,
      idleCount: dbPool.idleCount,
      waitingCount: dbPool.waitingCount
    }
  }
};
```

---

## SQL Injection Prevention

### What is SQL Injection?

**SQL Injection** = Attacker injects malicious SQL code

**Vulnerable Code (DON'T DO THIS):**
```javascript
// ❌ BAD: Direct string concatenation
const userId = req.body.userId;
const query = `SELECT * FROM users WHERE user_id = '${userId}'`;

// If userId = "'; DROP TABLE users; --"
// Query becomes:
// SELECT * FROM users WHERE user_id = ''; DROP TABLE users; --'
// 💥 Database deleted!
```

### Solution: Parameterized Queries

**Safe Code (DO THIS):**
```javascript
// ✅ GOOD: Parameterized query
const userId = req.body.userId;
const query = `SELECT * FROM users WHERE user_id = $1`;
const result = await dbPool.query(query, [userId]);

// Database treats $1 as placeholder
// userId is safely escaped
// No SQL injection possible!
```

### How Parameterized Queries Work

**1. Query Template:**
```sql
SELECT * FROM users WHERE user_id = $1
                      -- ^^ Placeholder ($1, $2, $3, ...)
```

**2. Parameters Array:**
```javascript
['123']  // Values for placeholders
```

**3. Database Handles Safely:**
- Escapes special characters (`'` → `''`)
- Validates types (string stays string)
- Prevents SQL injection

### Examples from Chat Service

**Single Parameter:**
```javascript
// messageOperations.js
await dbPool.query(
  'SELECT * FROM users WHERE user_id = $1',
  [userId]
);
```

**Multiple Parameters:**
```javascript
// messageOperations.js
await dbPool.query(
  `INSERT INTO private_messages 
   (message_id, conversation_id, sender_id, recipient_id, content) 
   VALUES ($1, $2, $3, $4, $5)`,
  [messageId, conversationId, senderId, recipientId, content]
);
```

**Array Parameter (IN clause):**
```javascript
// Get multiple users
await dbPool.query(
  'SELECT * FROM users WHERE user_id = ANY($1)',
  [['123', '456', '789']]
);
```

### Always Use Parameterized Queries!

**Rule:** NEVER put user input directly in SQL strings!

**❌ Bad:**
```javascript
const query = `SELECT * FROM users WHERE username = '${username}'`;
```

**✅ Good:**
```javascript
const query = `SELECT * FROM users WHERE username = $1`;
await dbPool.query(query, [username]);
```

---

## Transactions

### What is a Transaction?

**Transaction** = Multiple operations that succeed or fail together

**Analogy: Bank Transfer:**
```
1. Deduct $100 from Account A
2. Add $100 to Account B

Both must succeed, or both fail!
```

### Transactions in Chat Service

**Saving Message (Atomic Operation):**
```javascript
// messageOperations.js
async function savePrivateMessageToDatabase(message, dbPool) {
  const client = await dbPool.connect(); // Get connection
  
  try {
    // Start transaction
    await client.query('BEGIN');
    
    // 1. Ensure conversation exists
    await client.query(/* ... */);
    
    // 2. Save message
    await client.query(
      `INSERT INTO private_messages (...) VALUES (...)`,
      [/* ... */]
    );
    
    // 3. Commit transaction (all succeed)
    await client.query('COMMIT');
    
    return true;
  } catch (error) {
    // Rollback transaction (all fail)
    await client.query('ROLLBACK');
    throw error;
  } finally {
    // Always release connection
    client.release();
  }
}
```

**Why Transactions?**
- ✅ **Atomicity**: All operations succeed or all fail
- ✅ **Consistency**: Database stays in valid state
- ✅ **Isolation**: Other queries don't see partial data
- ✅ **Durability**: Committed data is permanent

---

## Query Optimization

### Indexing

**Index** = Data structure that speeds up queries

**Without Index:**
```
Query: SELECT * FROM users WHERE username = 'john'
    ↓
Database scans ALL rows (slow!)
```

**With Index:**
```
Query: SELECT * FROM users WHERE username = 'john'
    ↓
Database uses index (fast!)
```

**Creating Indexes:**
```sql
-- Index on username (for fast lookups)
CREATE INDEX idx_users_username ON users(username);

-- Index on user_id (primary key, automatically indexed)
CREATE INDEX idx_users_user_id ON users(user_id);
```

### Limiting Results

**Pagination:**
```javascript
// Get messages with pagination
async function loadMessages(conversationId, limit = 50, offset = 0) {
  const result = await dbPool.query(
    `SELECT * FROM private_messages 
     WHERE conversation_id = $1 
     ORDER BY created_at DESC 
     LIMIT $2 OFFSET $3`,
    [conversationId, limit, offset]
  );
  
  return result.rows;
}
```

**Why Limit?**
- ✅ Faster queries (less data to process)
- ✅ Less memory usage
- ✅ Better performance

---

## Database Schema

### Main Tables

**1. users**
```sql
CREATE TABLE users (
  user_id BIGINT PRIMARY KEY,
  username VARCHAR(255) UNIQUE,
  email VARCHAR(255),
  display_name VARCHAR(255),
  profile_image_url TEXT,
  bio TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

**2. private_messages**
```sql
CREATE TABLE private_messages (
  message_id TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL,
  sender_id BIGINT NOT NULL,
  recipient_id BIGINT NOT NULL,
  content TEXT,
  encrypted_content TEXT,
  encryption_iv TEXT,
  is_encrypted BOOLEAN DEFAULT false,
  is_read BOOLEAN DEFAULT false,
  reply_to TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

**3. conversations**
```sql
CREATE TABLE conversations (
  conversation_id TEXT PRIMARY KEY,
  participant1_id BIGINT NOT NULL,
  participant2_id BIGINT NOT NULL,
  last_message_at TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### Relationships

**Foreign Keys:**
```sql
-- Messages reference users
ALTER TABLE private_messages
ADD CONSTRAINT fk_sender
FOREIGN KEY (sender_id) REFERENCES users(user_id);

ALTER TABLE private_messages
ADD CONSTRAINT fk_recipient
FOREIGN KEY (recipient_id) REFERENCES users(user_id);
```

**Why Foreign Keys?**
- ✅ Data integrity (can't reference non-existent user)
- ✅ Automatic validation
- ✅ Database enforces relationships

---

## Key Takeaways

1. **Databases store data permanently**
   - Survives server restarts
   - ACID compliant

2. **Connection pooling improves performance**
   - Reuses connections (faster)
   - Limits concurrent connections (prevents overload)

3. **Always use parameterized queries**
   - Prevents SQL injection
   - Safer and cleaner

4. **Transactions ensure data integrity**
   - All operations succeed or all fail
   - Prevents partial updates

5. **Indexes speed up queries**
   - Faster lookups
   - Essential for large tables

6. **Limit query results**
   - Pagination
   - Better performance

---

## Next Steps

- Read [Security Guide](./SECURITY_GUIDE.md) for SQL injection prevention
- Read [Memory & Caching Guide](./MEMORY_CACHING_GUIDE.md) for caching strategies
- Check [Database Schema](./DATABASE_SCHEMA.md) for complete schema documentation

---

**Questions?** Check the [Main Technical Guide](./TECH_GUIDE_MAIN.md) for more resources!

