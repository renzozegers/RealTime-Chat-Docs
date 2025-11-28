# Database Optimization

## Overview

The messaging system uses a relational database with optimized queries, strategic indexing, and connection pooling to handle high message throughput while maintaining low latency.

## Query Optimizations

### CTE-Based User Metadata Lookups

**Problem:** Loading messages with user metadata required N subqueries (one per message), causing 2-4 second query times.

**Solution:** Pre-compute all user metadata in a Common Table Expression (CTE), then use simple subquery lookups.

**Before:**
```sql
SELECT 
  m.*,
  (SELECT display_name FROM users WHERE user_id = m.sender_id) as sender_name
FROM messages m
WHERE m.thread_id = $1
-- N subqueries = slow
```

**After:**
```sql
WITH sender_metadata AS (
  SELECT DISTINCT
    m.sender_id,
    COALESCE(
      (SELECT display_name FROM users WHERE user_id = m.sender_id),
      m.sender_id
    ) as sender_name
  FROM messages m
  WHERE m.thread_id = $1
)
SELECT 
  m.*,
  (SELECT sender_name FROM sender_metadata WHERE sender_id = m.sender_id) as sender_name
FROM messages m
WHERE m.thread_id = $1
-- 1 CTE + N simple lookups = fast (<200ms)
```

**Performance improvement:** 2-4s → <200ms

### Parallel Query Execution

**Problem:** Loading messages and counting total required sequential queries.

**Solution:** Run message loading and COUNT query in parallel using Promise.all or similar concurrency patterns.

**Implementation Pattern:**
```javascript
const [messagesResult, countResult] = await Promise.all([
  loadMessages(threadId, limit, offset),
  dbPool.query(`SELECT COUNT(*) FROM messages WHERE thread_id = $1`, [threadId])
]);
```

**Performance improvement:** ~50ms faster (parallel execution)

### Reaction Loading Optimization

**Problem:** Loading reactions required N subqueries for user metadata.

**Solution:** Pre-compute reaction user metadata in CTE.

**Implementation Pattern:**
```sql
WITH reaction_user_metadata AS (
  SELECT DISTINCT
    r.user_id,
    COALESCE(
      (SELECT display_name FROM users WHERE user_id = r.user_id),
      r.user_id::text
    ) AS user_name
  FROM reactions r
  WHERE r.message_id = ANY($1::text[])
)
SELECT
  r.*,
  (SELECT user_name FROM reaction_user_metadata WHERE user_id = r.user_id) AS user_name
FROM reactions r
WHERE r.message_id = ANY($1::text[])
```

**Performance improvement:** 1-2s → <100ms

## Indexes

### Primary Indexes

**Messages table:**
- Primary key on message identifier
- Index on thread identifier
- Index on sender identifier
- Index on recipient identifier
- Index on created timestamp (DESC)
- Composite index on (thread_id, created_at DESC)

**Group memberships:**
- Primary key on (group_id, user_id)
- Index on user_id
- Index on group_id

**Event queue:**
- Primary key on identifier
- Index on user_id
- Composite index on (user_id, delivered_at)
- Index on created_at

**Reactions:**
- Primary key on (message_id, user_id, reaction_type)
- Index on message_id

**Note:** Index strategies are described conceptually without exposing actual schema definitions.

## Connection Pooling

### Pool Configuration

**Settings:**
- Max connections: Configurable based on application needs
- Min connections: Maintains persistent connections
- Idle timeout: 30 seconds
- Connection timeout: 2 seconds
- Statement timeout: 30 seconds

**Configuration Pattern:**
```javascript
const dbPool = new Pool({
  max: MAX_CONNECTIONS,
  min: MIN_CONNECTIONS,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  statement_timeout: 30000,
  query_timeout: 30000
});
```

### Pool Monitoring

**Monitoring Metrics:**
- Total connections: `dbPool.totalCount`
- Idle connections: `dbPool.idleCount`
- Waiting queries: `dbPool.waitingCount`
- Active connections: `totalCount - idleCount`

**Warning Thresholds:**
- Pool nearly exhausted: >90% of max connections
- Queries waiting: `waitingCount > 0`

## Transaction Management

### Atomic Operations

**Message saving pattern:**
```javascript
const client = await dbPool.connect();
try {
  await client.query('BEGIN');
  await updateThreadMetadata(...);
  await client.query('INSERT INTO messages ...');
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release();
}
```

**Benefits:**
- All-or-nothing operations
- Data consistency
- Error recovery

### Connection Release

**Critical:** Always release connections immediately after transaction completes.

**Pattern:**
```javascript
const client = await dbPool.connect();
try {
  // ... transaction operations ...
} finally {
  client.release();  // Always release, even on error
}
```

**Why:** Prevents connection pool exhaustion.

## Schema Design

### Normalization

**Table Structure:**
- Messages table: Message content and metadata
- Threads: Thread metadata (denormalized for performance)
- Reactions: Reactions (separate table for many-to-many relationships)
- Deletions: Per-user deletion records (soft delete)

### Denormalization

**Threads table:**
- Stores participant identifiers (sorted)
- Stores last_message_at for sorting
- Reduces JOINs when loading thread list

**Benefits:**
- Faster thread list queries
- Simpler queries
- Reduced JOIN complexity

### Soft Deletes

**Deletions table:**
- Per-user deletion records
- Allows "delete for me" functionality
- Messages remain in database for other participants

**Benefits:**
- Data preservation
- Per-user deletion
- Audit trail

## Query Patterns

### Thread List Loading

**Optimized query pattern:**
```sql
WITH user_threads AS (
  SELECT thread_id, other_participant_id, sort_timestamp
  FROM threads
  WHERE (participant1_id = $1 OR participant2_id = $1)
    AND NOT EXISTS (SELECT 1 FROM thread_deletions ...)
),
latest_message_per_thread AS (
  SELECT DISTINCT ON (thread_id) ...
  FROM messages
  WHERE thread_id IN (SELECT thread_id FROM user_threads)
  ORDER BY thread_id, created_at DESC
),
unread_count_per_thread AS (
  SELECT thread_id, COUNT(*)::int AS unread_count
  FROM messages
  WHERE thread_id IN (SELECT thread_id FROM user_threads)
    AND recipient_id = $1 AND is_read = false
  GROUP BY thread_id
)
SELECT ... FROM user_threads ...
```

**Performance:** <1s for 50 threads (was 10-30s)

### Message Loading with Deletions

**Optimized query pattern:**
```sql
WITH deleted_message_ids AS (
  SELECT message_id 
  FROM deletions 
  WHERE user_id = $4
)
SELECT m.*
FROM messages m
WHERE m.thread_id = $1
  AND m.is_deleted = false
  AND NOT EXISTS (SELECT 1 FROM deleted_message_ids WHERE message_id = m.message_id)
ORDER BY m.created_at DESC
LIMIT $2 OFFSET $3
```

**Benefits:**
- Efficient deletion filtering
- Uses index on deletions table
- Fast execution

## Performance Monitoring

### Query Timing

**Logging pattern:**
```javascript
const startTime = Date.now();
const result = await dbPool.query(...);
const queryTime = Date.now() - startTime;
console.log(`[PERF] Query took ${queryTime}ms`);
```

### Connection Pool Monitoring

**Monitoring pattern:**
```javascript
setInterval(() => {
  const total = dbPool.totalCount || 0;
  const idle = dbPool.idleCount || 0;
  const waiting = dbPool.waitingCount || 0;
  
  if (total >= MAX_CONNECTIONS * 0.9) {
    console.warn(`[DATABASE] Pool nearly exhausted: ${total}/${MAX_CONNECTIONS}`);
  }
  
  if (waiting > 0) {
    console.warn(`[DATABASE] ${waiting} queries waiting for connections`);
  }
}, 5 * 60 * 1000);
```

## Best Practices

1. **Use CTEs**: Pre-compute data once instead of N subqueries
2. **Parallel queries**: Run independent queries in parallel
3. **Indexes**: Create indexes on frequently queried columns
4. **Connection pooling**: Reuse connections, release immediately
5. **Transactions**: Use transactions for atomic operations
6. **Query timeouts**: Set appropriate timeouts to prevent hanging
7. **Monitor pool**: Track connection pool usage and warnings

## Optimization Checklist

- CTE-based metadata lookups implemented
- Parallel queries for message loading and COUNT
- Indexes on all frequently queried columns
- Connection pool properly configured
- Transactions used for atomic operations
- Connections released immediately after use
- Query timeouts set appropriately
- Pool monitoring enabled
- Query timing logged for optimization
