# Testing Strategy

## Overview

The chat server includes a comprehensive test suite covering core functionality, edge cases, and performance characteristics. Tests are organized by feature area and can be run individually or as a complete suite.

## Test Structure

### Test Files

**Core Features:**
- Basic messaging, authentication, online users tests
- Message edit/delete workflow tests
- Community chat functionality tests

**Comprehensive Tests:**
- End-to-end feature validation tests
- Server initialization and module loading tests

**Performance Tests:**
- Lightweight load testing
- Full stress test with multiple scenarios

## Test Execution

### Prerequisites

1. **Server running**: Most tests require a running server instance
2. **Database setup**: PostgreSQL database with required tables
3. **Test tokens**: Valid JWT tokens for test users

### Running Tests

```bash
# Run all tests
npm test

# Core features (requires running server)
npm run test:comprehensive

# Message editing/deletion (requires running server)
npm run test:editing

# Community chat (requires running server)
npm run test:community

# Stress tests (requires running server)
npm run test:stress
npm run test:stress:full

# Server startup (stop server first)
npm run test:startup
```

### Test Token Management

**Refresh test tokens:**
```bash
npm run tokens:refresh
```

This script:
1. Generates new JWT tokens for test users
2. Copies tokens to test tokens file
3. Prints token details and usernames

**Important:** Refresh tokens whenever:
- JWT_SECRET changes
- Test users are recreated
- Tokens expire

## Test Categories

### 1. Authentication Tests

**Coverage:**
- Valid JWT token authentication
- Invalid token rejection
- Expired token handling
- Token revocation check
- User details loading


**Key assertions:**
- Authentication succeeds with valid token
- Authentication fails with invalid token
- User receives `connected` response
- Queued events delivered after authentication

### 2. Message Sending Tests

**Coverage:**
- Send private message
- Message encryption
- Database persistence
- Online delivery
- Offline queueing
- Message confirmation


**Key assertions:**
- Message saved to database
- Message encrypted in database
- Recipient receives message if online
- Event queued if recipient offline
- Sender receives confirmation

### 3. Message Loading Tests

**Coverage:**
- Load message history
- Pagination (limit/offset)
- Deleted message filtering
- Reaction loading
- Decryption of encrypted messages


**Key assertions:**
- Messages loaded in chronological order
- Pagination works correctly
- Deleted messages filtered per-user
- Reactions loaded for all messages
- Encrypted messages decrypted correctly

### 4. Message Editing Tests

**Coverage:**
- Edit message within 5-minute window
- Edit window expiration
- Permission checks (only sender can edit)
- Content re-encryption
- Broadcast to participants


**Key assertions:**
- Edit succeeds within 5 minutes
- Edit fails after 5 minutes
- Only sender can edit
- Content re-encrypted and saved
- Participants receive edit notification

### 5. Message Deletion Tests

**Coverage:**
- Delete for self (soft delete)
- Delete for everyone (hard delete)
- Permission checks
- Database cleanup
- Broadcast to participants


**Key assertions:**
- Soft delete creates deletion record
- Hard delete removes from database
- Only sender can delete for everyone
- Participants notified of deletion
- Deleted messages filtered from loads

### 6. Reaction Tests

**Coverage:**
- Add reaction
- Remove reaction
- Toggle reaction (add same reaction removes it)
- One reaction per user
- Broadcast to participants


**Key assertions:**
- Reaction saved to database
- Reaction removed from database
- Toggle behavior works correctly
- Only one reaction per user
- Participants receive reaction events

### 7. Group Chat Tests

**Coverage:**
- Create group
- Send group message
- Add/remove members
- Member permissions
- Group message loading
- @mentions


**Key assertions:**
- Group created with correct members
- Group messages delivered to all members
- Member management works correctly
- Permissions enforced
- Mentions resolved correctly

### 8. Reconnection Tests

**Coverage:**
- Grace period handling
- Event queue delivery
- In-memory queue delivery
- Database queue delivery
- Offline status broadcast

**Test approach:**
- Disconnect client
- Send message to disconnected user
- Reconnect client
- Verify queued events delivered

**Key assertions:**
- Events queued during disconnection
- Events delivered on reconnection
- Events delivered in chronological order
- Events marked as delivered in database

### 9. Stress Tests

**Coverage:**
- Multiple concurrent connections
- High message throughput
- Connection limits
- Memory usage
- Database connection pool

**Test files:**
- Lightweight stress tests (10-50 connections)
- Full stress tests (10-200 connections)

**Key metrics:**
- Messages per second
- Connection success rate
- Memory usage
- Database connection pool usage
- Error rate

## Test Data

### Test Users

Test users are created in the database with known credentials:

- `testuser1`, `testuser2`, etc.
- User IDs: Sequential integers
- JWT tokens: Generated with test secret

### Test Tokens


**Format:**
```json
{
  "testuser1": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "userId": "1",
    "username": "testuser1"
  }
}
```

**Generation:**
- Uses same JWT_SECRET as server
- Includes `sub` (userId), `jti`, `exp` claims
- Expires in 24 hours

## Test Patterns

### WebSocket Test Pattern

```javascript
const WebSocket = require('ws');

// Connect
const ws = new WebSocket('ws://localhost:3001');

// Authenticate
ws.on('open', () => {
  ws.send(JSON.stringify({
    type: 'connect',
    token: testTokens.testuser1.token
  }));
});

// Handle responses
ws.on('message', (data) => {
  const message = JSON.parse(data);
  // Assert message type and content
});

// Cleanup
ws.close();
```

### Database Test Pattern

```javascript
// Database connection pool (example pattern)
const dbPool = require('./database-config');

// Query database
const result = await dbPool.query('SELECT * FROM messages WHERE ...');

// Assert results
assert.strictEqual(result.rows.length, expectedCount);
```

## Continuous Integration

### Test Execution

Tests should be run:
- Before committing code
- In CI/CD pipeline
- After dependency updates
- After configuration changes

### Test Environment

**Required:**
- Node.js 18+
- PostgreSQL 12+
- Test database (separate from production)
- Test JWT tokens

**Optional:**
- Redis (for caching tests)
- Multiple test users

## Test Coverage

### Current Coverage

- **Authentication**: Complete
- **Message sending**: Complete
- **Message loading**: Complete
- **Message editing**: Complete
- **Message deletion**: Complete
- **Reactions**: Complete
- **Group chats**: Complete
- **Reconnection**: Partial (manual testing)
- **Stress testing**: Complete

### Gaps

- **Reconnection automation**: Needs automated test suite
- **Cross-worker communication**: Needs multi-worker test setup
- **Redis caching**: Needs Redis-specific tests
- **Error recovery**: Needs error scenario tests

## Best Practices

1. **Isolate tests**: Each test should be independent
2. **Clean up**: Remove test data after tests
3. **Use test database**: Never test against production
4. **Mock external services**: Mock main backend if possible
5. **Test error cases**: Test both success and failure paths
6. **Performance baselines**: Establish performance baselines
7. **Document test failures**: Log detailed failure information

## Debugging Tests

### Common Issues

**Token expiration:**
- Refresh tokens: `npm run tokens:refresh`
- Check token expiration time

**Server not running:**
- Start server instance
- Check server logs for errors

**Database connection:**
- Verify DATABASE_URL in .env
- Check database is running
- Verify connection pool limits

**Test failures:**
- Check server logs
- Verify test data exists
- Check database state
- Review test assertions

### Debug Mode

Run tests with verbose logging:
```bash
DEBUG=* npm test
```

## Future Improvements

1. **Automated reconnection tests**: Full test suite for reconnection scenarios
2. **Integration tests**: Test with real main backend
3. **Performance benchmarks**: Automated performance regression tests
4. **Load testing**: Continuous load testing in CI
5. **Coverage reporting**: Code coverage metrics
6. **Test documentation**: Detailed test documentation

