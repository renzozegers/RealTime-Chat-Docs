# Database Schema

Complete database schema documentation for the real-time messaging system. Table names have been modified for documentation purposes.

## Table of Contents

1. [Schema Overview](#schema-overview)
2. [Core Tables](#core-tables)
3. [Group Tables](#group-tables)
4. [Event Queue Tables](#event-queue-tables)
5. [User Management Tables](#user-management-tables)
6. [Indexes and Performance](#indexes-and-performance)
7. [Relationships](#relationships)
8. [Data Types](#data-types)

---

## Schema Overview

The database uses PostgreSQL and is shared between the chat server and main backend. All chat-related tables are managed by the chat server, while user management tables are shared.

### Database Connection

- **Connection Pool:** 10 connections per worker (configurable)
- **Total App Limit:** 40 connections (chat server: 10, main backend: 30)
- **Transaction Support:** All write operations use transactions
- **Isolation Level:** READ COMMITTED (default)

---

## Core Tables

### chat_messages

Stores all messages (both direct messages and group messages).

**Table Name:** `chat_messages` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `message_id` | VARCHAR(255) | PRIMARY KEY | Unique message identifier (UUID format) |
| `conversation_id` | VARCHAR(255) | NOT NULL, INDEXED | Conversation identifier (DM: `conv_{id1}_{id2}`, Group: `group_{uuid}`) |
| `sender_id` | VARCHAR(255) | NOT NULL, INDEXED | User ID of message sender |
| `recipient_id` | VARCHAR(255) | NULL, INDEXED | User ID of recipient (NULL for group messages) |
| `group_id` | VARCHAR(255) | NULL, INDEXED | Group ID (NULL for direct messages) |
| `content` | TEXT | NOT NULL | Original message content (unencrypted, for display) |
| `encrypted_content` | TEXT | NULL | AES-256 encrypted message content |
| `encryption_iv` | VARCHAR(255) | NULL | Initialization Vector for decryption |
| `is_encrypted` | BOOLEAN | NOT NULL, DEFAULT true | Whether message is encrypted |
| `is_read` | BOOLEAN | NOT NULL, DEFAULT false | Read status (per message) |
| `reply_to` | VARCHAR(255) | NULL, INDEXED | Message ID this message replies to |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW(), INDEXED | Message creation timestamp |
| `updated_at` | TIMESTAMP | NULL | Last update timestamp (for edits) |

**Indexes:**
- Primary key on `message_id`
- Index on `conversation_id` (for loading conversation history)
- Index on `sender_id` (for user message queries)
- Index on `recipient_id` (for recipient message queries)
- Index on `group_id` (for group message queries)
- Index on `created_at` (for chronological ordering)
- Index on `reply_to` (for reply chains)

**Implementation Details:**
- Messages encrypted with AES-256 before storage
- IV stored separately for decryption
- Content field stores unencrypted version for quick access (encrypted version is primary)
- Group messages have `group_id` set, `recipient_id` is NULL
- Direct messages have `recipient_id` set, `group_id` is NULL
- `reply_to` supports message threading
- Soft deletes handled via `message_removals` table

**Query Patterns:**
```sql
-- Load conversation messages (paginated)
SELECT * FROM chat_messages 
WHERE conversation_id = $1 
ORDER BY created_at DESC 
LIMIT $2 OFFSET $3;

-- Load group messages
SELECT * FROM chat_messages 
WHERE group_id = $1 
ORDER BY created_at DESC 
LIMIT $2 OFFSET $3;
```

---

### dm_conversations

Stores metadata for direct message conversations.

**Table Name:** `dm_conversations` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `conversation_id` | VARCHAR(255) | PRIMARY KEY | Unique conversation identifier (`conv_{id1}_{id2}`) |
| `participant1_id` | VARCHAR(255) | NOT NULL, INDEXED | First participant user ID (sorted) |
| `participant2_id` | VARCHAR(255) | NOT NULL, INDEXED | Second participant user ID (sorted) |
| `last_message_at` | TIMESTAMP | NOT NULL | Timestamp of last message in conversation |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Conversation creation timestamp |
| `updated_at` | TIMESTAMP | NULL | Last update timestamp |

**Indexes:**
- Primary key on `conversation_id`
- Index on `participant1_id` (for user conversation queries)
- Index on `participant2_id` (for user conversation queries)
- Composite index on `(participant1_id, participant2_id)` (for lookup)

**Implementation Details:**
- Conversation ID generated as `conv_{sorted_id1}_{sorted_id2}`
- Participants sorted to ensure consistent conversation ID
- Upserted on every message (updates `last_message_at`)
- Used for conversation list queries

**Query Patterns:**
```sql
-- Find conversation between two users
SELECT * FROM dm_conversations 
WHERE (participant1_id = $1 AND participant2_id = $2)
   OR (participant1_id = $2 AND participant2_id = $1);

-- Get user's conversations (ordered by last message)
SELECT * FROM dm_conversations 
WHERE participant1_id = $1 OR participant2_id = $1
ORDER BY last_message_at DESC;
```

---

## Group Tables

### chat_groups

Stores group chat metadata.

**Table Name:** `chat_groups` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `group_id` | VARCHAR(255) | PRIMARY KEY | Unique group identifier (`group_{uuid}`) |
| `group_name` | VARCHAR(255) | NOT NULL | Group display name |
| `group_description` | TEXT | NULL | Group description |
| `created_by_user_id` | VARCHAR(255) | NOT NULL, INDEXED | User ID of group creator |
| `max_members` | INTEGER | NOT NULL, DEFAULT 50 | Maximum number of members |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true, INDEXED | Whether group is active (soft delete) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Group creation timestamp |
| `updated_at` | TIMESTAMP | NULL | Last update timestamp |

**Indexes:**
- Primary key on `group_id`
- Index on `created_by_user_id` (for creator queries)
- Index on `is_active` (for filtering active groups)

**Implementation Details:**
- Group ID generated as `group_{UUID}`
- Soft delete via `is_active` flag
- Creator automatically added as admin in `group_participants`
- Max members enforced on member addition

---

### group_participants

Stores group membership and roles.

**Table Name:** `group_participants` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `group_id` | VARCHAR(255) | NOT NULL, INDEXED | Group identifier |
| `user_id` | VARCHAR(255) | NOT NULL, INDEXED | User identifier |
| `role` | VARCHAR(50) | NOT NULL, DEFAULT 'member' | User role: 'owner', 'admin', 'member' |
| `can_add_members` | BOOLEAN | NOT NULL, DEFAULT false | Permission to add members |
| `added_by_user_id` | VARCHAR(255) | NULL | User who added this member |
| `joined_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | When user joined group |
| `updated_at` | TIMESTAMP | NULL | Last update timestamp |

**Indexes:**
- Composite primary key on `(group_id, user_id)`
- Index on `group_id` (for group member queries)
- Index on `user_id` (for user group queries)
- Index on `role` (for role-based queries)

**Implementation Details:**
- Unique constraint on `(group_id, user_id)` prevents duplicates
- Roles: `owner` (can delete group, promote/demote), `admin` (can add members, send announcements), `member` (basic permissions)
- `can_add_members` flag grants permission independently of role
- Creator automatically set as `owner` with `can_add_members=true`

**Query Patterns:**
```sql
-- Get all members of a group
SELECT * FROM group_participants 
WHERE group_id = $1 
ORDER BY joined_at ASC;

-- Get all groups a user belongs to
SELECT g.* FROM chat_groups g
JOIN group_participants p ON g.group_id = p.group_id
WHERE p.user_id = $1 AND g.is_active = true;

-- Count members per group
SELECT group_id, COUNT(*) as member_count
FROM group_participants
GROUP BY group_id;
```

---

## Event Queue Tables

### offline_events

Stores events queued for offline users (reactions, edits, deletions, group changes).

**Table Name:** `offline_events` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `id` | BIGSERIAL | PRIMARY KEY | Auto-incrementing event ID |
| `user_id` | VARCHAR(255) | NOT NULL, INDEXED | User ID to deliver event to |
| `event_type` | VARCHAR(100) | NOT NULL, INDEXED | Event type: 'reaction_added', 'member_added', etc. |
| `event_data` | JSONB | NOT NULL | Event payload (JSON) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW(), INDEXED | Event creation timestamp |
| `delivered_at` | TIMESTAMP | NULL, INDEXED | When event was delivered (NULL = undelivered) |

**Indexes:**
- Primary key on `id`
- Index on `user_id` (for user event queries)
- Index on `event_type` (for event type filtering)
- Index on `created_at` (for chronological ordering)
- Index on `delivered_at` (for cleanup queries)
- Composite index on `(user_id, delivered_at)` (for undelivered event queries)

**Implementation Details:**
- Events queued when user is offline or on different worker
- Delivered on reconnection in chronological order
- Marked as delivered with `delivered_at` timestamp
- Cleaned up after 7 days (delivered) or 30 days (undelivered)
- NOT for messages - messages stored in `chat_messages` table

**Event Types:**
- `reaction_added` - Reaction added to message
- `reaction_removed` - Reaction removed from message
- `message_edited` - Message was edited
- `message_deleted` - Message was deleted
- `member_added` - User added to group
- `member_removed` - User removed from group
- `member_promoted` - Member promoted to admin
- `member_demoted` - Admin demoted to member
- `group_info_updated` - Group name/description updated
- `group_deleted` - Group was deleted

**Query Patterns:**
```sql
-- Get undelivered events for user
SELECT * FROM offline_events 
WHERE user_id = $1 AND delivered_at IS NULL
ORDER BY created_at ASC
LIMIT 1000;

-- Mark events as delivered
UPDATE offline_events 
SET delivered_at = NOW() 
WHERE id = ANY($1::bigint[]);
```

---

## User Management Tables

These tables are shared with the main backend and managed by the Java backend.

### user_accounts

User authentication and account information.

**Table Name:** `user_accounts` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `user_id` | VARCHAR(255) | PRIMARY KEY | Unique user identifier |
| `username` | VARCHAR(255) | NOT NULL, UNIQUE | Username |
| `email` | VARCHAR(255) | NULL | User email |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Account creation timestamp |

**Usage:**
- Username lookups for message display
- User existence validation
- JWT token validation

---

### user_profiles

User profile information.

**Table Name:** `user_profiles` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `user_id` | VARCHAR(255) | PRIMARY KEY | User identifier (FK to user_accounts) |
| `display_name` | VARCHAR(255) | NULL | User display name |
| `profile_picture_url` | VARCHAR(500) | NULL | Profile picture URL |
| `updated_at` | TIMESTAMP | NULL | Last update timestamp |

---

### blocked_relationships

User blocking relationships (bidirectional).

**Table Name:** `blocked_relationships` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `blocker_id` | VARCHAR(255) | NOT NULL, INDEXED | User who blocked |
| `blocked_id` | VARCHAR(255) | NOT NULL, INDEXED | User who was blocked |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Block creation timestamp |

**Indexes:**
- Composite primary key on `(blocker_id, blocked_id)`
- Index on `blocker_id` (for blocker queries)
- Index on `blocked_id` (for blocked user queries)

**Implementation Details:**
- Bidirectional blocking: If A blocks B, B cannot message A
- Checked before every message send
- Query: `SELECT 1 FROM blocked_relationships WHERE (blocker_id = $1 AND blocked_id = $2) OR (blocker_id = $2 AND blocked_id = $1)`

---

### user_follows

Follow relationships (optional, for social features).

**Table Name:** `user_follows` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `follower_id` | VARCHAR(255) | NOT NULL, INDEXED | User who follows |
| `followed_id` | VARCHAR(255) | NOT NULL, INDEXED | User who is followed |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Follow creation timestamp |

**Implementation Details:**
- Optional check before messaging (soft requirement)
- Not strictly enforced (messages allowed even without follow)

---

### token_revocations

Revoked JWT tokens (JTI-based).

**Table Name:** `token_revocations` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `jti` | VARCHAR(255) | PRIMARY KEY | JWT ID (JTI claim) |
| `revoked_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Revocation timestamp |
| `expires_at` | TIMESTAMP | NOT NULL, INDEXED | Token expiration timestamp |

**Implementation Details:**
- Checked during JWT validation
- Expired tokens cleaned up periodically
- Prevents use of logged-out tokens

---

## Deletion Tables

### message_removals

Per-user message deletion records (soft delete).

**Table Name:** `message_removals` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `message_id` | VARCHAR(255) | NOT NULL, INDEXED | Message identifier |
| `user_id` | VARCHAR(255) | NOT NULL, INDEXED | User who deleted message |
| `deleted_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Deletion timestamp |

**Indexes:**
- Composite primary key on `(message_id, user_id)`
- Index on `message_id` (for message queries)
- Index on `user_id` (for user deletion queries)

**Implementation Details:**
- Per-user soft delete: Message hidden from user's view but remains in database
- Used to filter messages in `get_private_messages` and `get_group_messages`
- Hard delete (deleteForEveryone) removes message from `chat_messages` table

---

### conversation_removals

Per-user conversation deletion records.

**Table Name:** `conversation_removals` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `conversation_id` | VARCHAR(255) | NOT NULL, INDEXED | Conversation identifier |
| `user_id` | VARCHAR(255) | NOT NULL, INDEXED | User who deleted conversation |
| `deleted_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Deletion timestamp |

**Indexes:**
- Composite primary key on `(conversation_id, user_id)`
- Index on `conversation_id`
- Index on `user_id`

**Implementation Details:**
- Hides conversation from user's conversation list
- Messages remain in database
- Conversation can be "undeleted" by removing record

---

## Reaction Tables

### chat_reactions

Message reactions (emoji).

**Table Name:** `chat_reactions` (documented name, actual name differs)

**Columns:**

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `message_id` | VARCHAR(255) | NOT NULL, INDEXED | Message identifier |
| `user_id` | VARCHAR(255) | NOT NULL, INDEXED | User who reacted |
| `reaction` | VARCHAR(10) | NOT NULL | Emoji reaction (unicode) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Reaction timestamp |
| `updated_at` | TIMESTAMP | NULL | Last update timestamp |

**Indexes:**
- Composite primary key on `(message_id, user_id)`
- Index on `message_id` (for message reaction queries)
- Index on `user_id` (for user reaction queries)

**Implementation Details:**
- One reaction per user per message
- Replacing existing reaction updates `updated_at`
- Same reaction removes it (toggle behavior)
- Loaded with messages via subquery

**Query Patterns:**
```sql
-- Get all reactions for a message
SELECT * FROM chat_reactions 
WHERE message_id = $1;

-- Get reactions for multiple messages (used in message loading)
SELECT * FROM chat_reactions 
WHERE message_id = ANY($1::text[]);
```

---

## Indexes and Performance

### Index Strategy

**Primary Indexes:**
- All tables have primary keys on unique identifiers
- Composite primary keys for junction tables

**Query Optimization Indexes:**
- `conversation_id` on `chat_messages` (most common query)
- `created_at` on `chat_messages` (chronological ordering)
- `user_id` on all user-related tables
- `group_id` on group-related tables

**Composite Indexes:**
- `(group_id, user_id)` on `group_participants` (member lookup)
- `(user_id, delivered_at)` on `offline_events` (undelivered events)
- `(participant1_id, participant2_id)` on `dm_conversations` (conversation lookup)

### Query Performance

**Optimized Queries:**
- Username lookups pre-computed in CTE (Common Table Expression)
- Pagination with `LIMIT/OFFSET`
- Subqueries instead of JOINs for readability
- Batch operations for reactions and mentions

**Performance Characteristics:**
- Message loading: <200ms (with CTE optimization)
- Conversation list: <100ms
- Group member queries: <50ms
- Event queue loading: <100ms

---

## Relationships

### Entity Relationship Diagram

```
user_accounts (1) ──┐
                    ├── (M) chat_messages (sender)
                    ├── (M) chat_messages (recipient)
                    ├── (M) group_participants
                    ├── (M) blocked_relationships (blocker)
                    ├── (M) blocked_relationships (blocked)
                    └── (M) offline_events

chat_groups (1) ────┐
                    ├── (M) chat_messages
                    └── (M) group_participants

chat_messages (1) ──┐
                    ├── (M) chat_reactions
                    ├── (M) message_removals
                    └── (1) chat_messages (reply_to, self-reference)

dm_conversations (1) ─── (M) chat_messages
```

### Foreign Key Relationships

**Explicit Foreign Keys:**
- `group_participants.group_id` → `chat_groups.group_id`
- `group_participants.user_id` → `user_accounts.user_id`
- `chat_messages.sender_id` → `user_accounts.user_id`
- `chat_messages.recipient_id` → `user_accounts.user_id`
- `chat_reactions.message_id` → `chat_messages.message_id`
- `chat_reactions.user_id` → `user_accounts.user_id`

**Logical Relationships (no FK constraints):**
- `chat_messages.conversation_id` → `dm_conversations.conversation_id`
- `chat_messages.group_id` → `chat_groups.group_id`
- `chat_messages.reply_to` → `chat_messages.message_id`
- `offline_events.user_id` → `user_accounts.user_id`

**Note:** Some relationships don't have explicit foreign key constraints due to:
- Cross-schema references (chat server vs main backend)
- Performance considerations
- Soft delete patterns

---

## Data Types

### Common Data Types

**VARCHAR(255):** User IDs, message IDs, conversation IDs, group IDs
- All IDs stored as strings for consistency
- UUIDs formatted as `{prefix}_{uuid}`

**TEXT:** Message content, descriptions
- No length limit (handled by application)
- Encryption may increase size

**JSONB:** Event data in `offline_events`
- Efficient JSON storage and querying
- Indexed for performance

**TIMESTAMP:** All timestamps
- Stored in UTC
- Default: `NOW()`
- Used for ordering and filtering

**BOOLEAN:** Flags and status fields
- Default values specified
- Indexed for filtering

---

## Implementation Notes

### Transaction Management

All write operations use database transactions:

```sql
BEGIN;
  -- Multiple operations
  INSERT INTO chat_messages ...;
  UPDATE dm_conversations ...;
COMMIT;
```

**Rollback on Error:**
- Any error triggers `ROLLBACK`
- Ensures data consistency
- Prevents partial writes

### Encryption

**Message Encryption:**
- AES-256 encryption for message content
- IV (Initialization Vector) stored in `encryption_iv` column
- Encryption key from environment variable
- Content encrypted before database insert

**Encryption Process:**
1. Generate random IV
2. Encrypt content with AES-256 using IV
3. Store encrypted content and IV separately
4. Original content also stored (for quick access, encrypted is primary)

### Soft Deletes

**Per-User Deletes:**
- Messages: `message_removals` table
- Conversations: `conversation_removals` table
- Filtered in queries: `WHERE NOT EXISTS (SELECT 1 FROM message_removals ...)`

**Global Deletes:**
- Groups: `is_active = false` flag
- Messages: Hard delete (removed from `chat_messages`)

### Cleanup Operations

**Automatic Cleanup:**
- Delivered events: Cleaned up after 7 days
- Undelivered events: Cleaned up after 30 days
- Expired tokens: Cleaned up periodically
- Runs every 30 seconds via scheduled task

**Manual Cleanup:**
- Inactive conversations removed from memory after 30 minutes
- Old messages trimmed from in-memory cache (last 50-100 kept)

---

## Migration Strategy

### Schema Changes

**Version Control:**
- Schema changes tracked in migration files
- Applied via `ensure-schema.js` script
- Idempotent operations (safe to run multiple times)

**Common Migrations:**
- Adding new columns (with defaults)
- Creating new indexes
- Adding new tables
- Modifying constraints

### Backup Strategy

**Database Backups:**
- Daily automated backups
- Point-in-time recovery supported
- Transaction logs retained for 7 days

**Data Retention:**
- Messages: Permanent (never auto-deleted)
- Events: 7-30 days (as specified)
- Deleted records: Permanent (for audit)

---

## Performance Considerations

### Query Optimization

**CTE Usage:**
- Username lookups pre-computed in CTE
- Reduces N+1 query problem
- Improves query time from 2-4s to <200ms

**Pagination:**
- Always use `LIMIT/OFFSET` for large result sets
- Default limit: 50, max: 200
- Offset-based pagination (cursor-based can be added)

**Index Usage:**
- All foreign key columns indexed
- Timestamp columns indexed for ordering
- Composite indexes for common query patterns

### Connection Pooling

**Pool Configuration:**
- Max connections: 10 per worker
- Idle timeout: 30 seconds
- Connection timeout: 5 seconds
- Shared across all workers

**Pool Management:**
- Connections released immediately after transaction
- Prevents pool exhaustion
- Automatic retry on connection failure

---

## Security Considerations

### Data Protection

**Encryption:**
- Message content encrypted at rest
- Encryption keys stored in environment variables
- IV stored separately for security

**Access Control:**
- JWT token validation for all operations
- User ID extracted from token (not trusted from client)
- Permission checks for group operations

### SQL Injection Prevention

**Parameterized Queries:**
- All queries use parameterized statements
- No string concatenation in SQL
- Prepared statements for repeated queries

**Example:**
```sql
-- ✅ Safe
SELECT * FROM chat_messages WHERE conversation_id = $1;

-- ❌ Unsafe (never used)
SELECT * FROM chat_messages WHERE conversation_id = '${conversationId}';
```

---

## Monitoring and Maintenance

### Health Checks

**Database Health:**
- Connection pool status monitored
- Query performance tracked
- Slow query logging enabled (>1 second)

**Table Statistics:**
- Table sizes monitored
- Index usage tracked
- Vacuum operations scheduled

### Maintenance Tasks

**Regular Maintenance:**
- VACUUM ANALYZE weekly
- Index rebuilds monthly
- Statistics updates daily
- Connection pool monitoring continuous

