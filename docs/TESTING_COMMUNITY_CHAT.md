# Testing Community Chat

## Quick Start

### Prerequisites

1. **Start the server:**
   ```bash
   node server.js
   # Or in background:
   node server.js &
   ```

2. **Ensure test data exists:**
   - Test JWT tokens in `tests/test-jwt-tokens.json`
   - Test community in `social_schema.communities` (optional, for full testing)
   - Test users in `users_auth` table

### Run Community Chat Tests

```bash
cd tests
node test-community-chat.js
```

## Test Coverage

### 1. Database Operations Tests

Tests the core database functions:

- ✅ `getCommunityName(communityId)` - Get community name
- ✅ `getCommunityChatByGroupId(groupId)` - Detect community chats
- ✅ `ensureCommunityChatInfrastructure(groupId)` - Create infrastructure
- ✅ `isUserInCommunity(userId, communityId)` - Membership check
- ✅ `isUserCommunityLeader(userId, communityId)` - Leadership check
- ✅ `getCommunitiesForUser(userId)` - Get user's communities

**Run:**
```bash
node tests/test-community-chat.js
```

### 2. HTTP API Tests

Tests the REST endpoints:

- ✅ `GET /api/community/user-communities` - Get user's communities
- ✅ `GET /api/community/:communityId/members` - Get community members

**Run:**
```bash
# Make sure server is running first
node tests/test-community-chat.js
```

### 3. WebSocket Tests

Tests real-time chat operations:

- ✅ WebSocket connection and authentication
- ✅ `get_user_groups` - Should include community chats
- ✅ `send_group_message` - Send to community chat
- ✅ `get_group_messages` - Load community messages

**Run:**
```bash
# Make sure server is running first
node tests/test-community-chat.js
```

### 4. Membership Validation Tests

Tests membership checks:

- ✅ `isUserInCommunity()` - Valid membership check
- ✅ `isUserCommunityLeader()` - Leadership check
- ✅ Auto-sync to `group_members` when needed

**Run:**
```bash
node tests/test-community-chat.js
```

## Manual Testing Guide

### Test 1: Create Community Chat Infrastructure

**Setup:**
1. Create a community in `social_schema.communities`:
   ```sql
   INSERT INTO social_schema.communities (id, name, slug, visibility, created_by)
   VALUES (123, 'Test Community', 'test-community', 'public', 1);
   ```

2. Insert into `community_chat`:
   ```sql
   INSERT INTO community_chat (community_id) VALUES (123);
   ```

**Test:**
```bash
node -e "
const { ensureCommunityChatInfrastructure } = require('./database/communityOperations');
ensureCommunityChatInfrastructure('123').then(id => {
  console.log('Group ID:', id);
  process.exit(0);
});
"
```

**Expected:** Creates `group_chat` entry with `group_id = "123"`

### Test 2: Send Message to Community Chat

**Setup:**
1. Ensure community exists (from Test 1)
2. Add user to `social_schema.community_memberships`:
   ```sql
   INSERT INTO social_schema.community_memberships 
   ("communityId", "userId", status, role, "joinedAt")
   VALUES (123, 1, 'active', 'member', NOW());
   ```

**Test via WebSocket:**
```javascript
const WebSocket = require('ws');
const ws = new WebSocket('ws://localhost:3001');

ws.on('open', () => {
  ws.send(JSON.stringify({
    type: 'authenticate',
    token: 'YOUR_JWT_TOKEN'
  }));
});

ws.on('message', (data) => {
  const msg = JSON.parse(data);
  if (msg.type === 'auth_success') {
    // Send message to community chat
    ws.send(JSON.stringify({
      type: 'send_group_message',
      groupId: '123',  // community_id as string
      content: 'Hello community!'
    }));
  }
  console.log('Received:', msg.type);
});
```

**Expected:** Message sent successfully, received by all community members

### Test 3: Get User Communities

**Test via HTTP:**
```bash
curl -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  http://localhost:3001/api/community/user-communities
```

**Expected Response:**
```json
{
  "status": "ok",
  "communities": [
    {
      "id": 1,
      "communityId": 123,
      "groupId": "123",
      "name": "Test Community",
      ...
    }
  ],
  "count": 1
}
```

### Test 4: Membership Validation

**Test:**
```bash
node -e "
const { isUserInCommunity, isUserCommunityLeader } = require('./database/communityOperations');
isUserInCommunity(1, 123).then(result => {
  console.log('Is member:', result);
  return isUserCommunityLeader(1, 123);
}).then(result => {
  console.log('Is leader:', result);
  process.exit(0);
});
"
```

**Expected:** Returns `true` if user is member/leader, `false` otherwise

### Test 5: Auto-Sync Members

**Setup:**
1. User in `social_schema.community_memberships` but NOT in `group_members`
2. User tries to send message

**Test:**
- Send message to community chat
- Check `group_members` table - user should be auto-added

**Expected:** User automatically synced to `group_members` on first use

## Integration Testing

### Full Flow Test

1. **Create Community** (social service)
   ```sql
   INSERT INTO social_schema.communities (id, name, ...) VALUES (123, 'Test', ...);
   INSERT INTO community_chat (community_id) VALUES (123);
   ```

2. **User Joins** (social service)
   ```sql
   INSERT INTO social_schema.community_memberships 
   ("communityId", "userId", status, role) 
   VALUES (123, 1, 'active', 'member');
   ```

3. **Frontend Loads**
   - Call `getUserGroups()` via WebSocket
   - Should return community with `groupId = "123"`

4. **User Sends Message**
   - Send message with `groupId = "123"`
   - Infrastructure auto-created
   - User auto-synced to `group_members`
   - Message saved and broadcast

5. **Verify**
   ```sql
   -- Check infrastructure created
   SELECT * FROM group_chats WHERE group_id = '123';
   
   -- Check user synced
   SELECT * FROM group_members WHERE group_id = '123' AND user_id = '1';
   
   -- Check message saved
   SELECT * FROM private_messages 
   WHERE recipient_id = '123' AND is_group_message = true;
   ```

## Test Data Setup

### Create Test Community

```sql
-- 1. Create community
INSERT INTO social_schema.communities (id, name, slug, visibility, created_by, created_at)
VALUES (999, 'Test Community', 'test-community', 'public', 1, NOW());

-- 2. Create community_chat entry
INSERT INTO community_chat (community_id) VALUES (999);

-- 3. Add test users to membership
INSERT INTO social_schema.community_memberships 
("communityId", "userId", status, role, "joinedAt")
VALUES 
  (999, 1, 'active', 'leader', NOW()),
  (999, 2, 'active', 'member', NOW()),
  (999, 3, 'active', 'member', NOW());
```

### Cleanup Test Data

```sql
-- Remove test community
DELETE FROM social_schema.community_memberships WHERE "communityId" = 999;
DELETE FROM community_chat WHERE community_id = 999;
DELETE FROM group_members WHERE group_id = '999';
DELETE FROM group_chats WHERE group_id = '999';
DELETE FROM private_messages WHERE recipient_id = '999';
DELETE FROM social_schema.communities WHERE id = 999;
```

## Troubleshooting

### Tests Fail to Connect

**Problem:** WebSocket connection fails
**Solution:**
- Check server is running: `node server.js`
- Check port 3001 is available
- Check firewall settings

### Authentication Fails

**Problem:** `auth_error` received
**Solution:**
- Check JWT token is valid in `test-jwt-tokens.json`
- Check `JWT_SECRET` in `.env` matches backend
- Check token hasn't expired

### Community Not Found

**Problem:** `getCommunityChatByGroupId` returns null
**Solution:**
- Ensure community exists in `social_schema.communities`
- Ensure `community_chat` row exists for that community
- Check `community_id` matches

### Membership Check Fails

**Problem:** `isUserInCommunity` returns false
**Solution:**
- Check user exists in `social_schema.community_memberships`
- Check `status = 'active'` and `leftAt IS NULL`
- Check `communityId` matches

### Infrastructure Not Created

**Problem:** `group_chat` entry not created
**Solution:**
- Check `ensureCommunityChatInfrastructure` is called
- Check `community_chat` row exists
- Check database permissions
- Check for ID conflicts (regular group with same ID)

## Expected Test Results

### Successful Test Run

```
🧪 Community Chat Test Suite
==================================================
Server: ws://localhost:3001
HTTP: http://localhost:3001

📊 Testing Database Operations
==================================================
✅ getCommunityName: Skipped - requires test community
✅ getCommunityChatByGroupId (non-existent): Correctly returns null
✅ ensureCommunityChatInfrastructure (regular group): Returns groupId as-is

🌐 Testing HTTP Endpoints
==================================================
✅ GET /api/community/user-communities: Found 2 communities

🔌 Testing WebSocket Operations
==================================================
✅ WebSocket Connection: Connected and authenticated
✅ getUserGroups: Found 3 groups/communities

👥 Testing Membership Validation
==================================================
✅ isUserInCommunity: Skipped - requires test IDs
✅ isUserCommunityLeader: Skipped - requires test IDs

==================================================
📊 Test Results Summary
==================================================
✅ Passed: 6
❌ Failed: 0
📈 Success Rate: 100.0%

🎉 All tests passed!
```

## Next Steps

1. **Add test community to database** for full testing
2. **Create test users** in `users_auth` if needed
3. **Run comprehensive tests** with real data
4. **Test edge cases:**
   - User not in community tries to send message
   - Community with no members
   - Multiple communities for same user
   - Leadership permissions

## Related Documentation

- [COMMUNITY_CHAT.md](./COMMUNITY_CHAT.md) - Full community chat documentation
- [README_TESTS.md](../tests/README_TESTS.md) - General testing guide

