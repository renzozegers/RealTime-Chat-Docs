# Community Chat Documentation

## Overview

Community chats are **independent entities** that use group chat mechanics. They have ALL group chat features (messages, membership, roles, etc.) AND can have additional community-specific features.

### Key Architecture Principles

1. **Independent Entity**: `community_chat` table only has `id` and `community_id` - proves a community chat exists
2. **Uses Group Chat Mechanics**: `community_id` (as string) = `group_id` for all group operations
3. **Membership via Social Schema**: Uses `social_schema.community_memberships` for membership checks
4. **No Blocking**: Blocking relationships are ignored for community chats
5. **Automatic**: Infrastructure and members are auto-created and synced

## Database Structure

### Tables Involved

```
community_chat (chat service)
├── id (BIGSERIAL PRIMARY KEY)
└── community_id (BIGINT NOT NULL UNIQUE) → references social_schema.communities.id

social_schema.communities (social service)
├── id (BIGINT)
├── name, slug, visibility, etc.
└── (no chat_space_id - we use community_id directly)

social_schema.community_memberships (social service)
├── communityId (BIGINT)
├── userId (BIGINT)
├── status ('active')
├── role ('leader', 'member', etc.)
└── leftAt (NULL if active)

group_chats (chat service)
├── group_id (VARCHAR) = community_id.toString()
├── group_name, group_description, etc.
└── (all group chat fields)

group_members (chat service)
├── group_id (VARCHAR) = community_id.toString()
├── user_id (VARCHAR)
└── role, permissions, etc.
```

### The Direct Mapping

**Key Concept:** `community_id.toString()` = `group_id`

```
community_chat (community_id: 123)
    ↓
community_id.toString() = "123"
    ↓
group_chats.group_id = "123"
    ↓
All group operations use "123" as group_id
```

## How It Works: Step by Step

### 1. Community Creation (Social Service)

When a community is created:
```sql
-- Social service does:
INSERT INTO social_schema.communities (name, slug, ...) VALUES (...);
INSERT INTO community_chat (community_id) VALUES (123);  -- Just id and community_id
```

**Chat service does NOTHING** during community creation.

**Important:** The `community_chat` row exists, but the `group_chat` infrastructure does NOT exist yet.

### 2. Frontend Loads Community Chats

When the frontend loads, it calls `getUserGroups()`:

```javascript
// Backend handleGetUserGroups():
const communities = await getCommunitiesForUser(userId);
// Queries community_chat table (which EXISTS)
// Returns: [{ communityId: 123, groupId: "123", name: "My Community", ... }]
```

**Key Point:** The `groupId` comes from `community_id.toString()` - we know this from the `community_chat` table, even though `group_chat` doesn't exist yet!

**Frontend receives:**
```json
{
  "groups": [
    {
      "group_id": "123",  // ← community_id as string (from community_chat table)
      "group_name": "My Community",
      ...
    }
  ]
}
```

### 3. First Use of Community Chat

When user sends a message with `groupId = "123"`:

```javascript
// In handleSendGroupMessage or handleGetGroupMessages:
const actualGroupId = await ensureCommunityChatInfrastructure(groupId);
```

**What `ensureCommunityChatInfrastructure` does:**

1. Checks if `groupId` is numeric (e.g., "123")
   - If yes → it's a community chat
   - If no → regular group chat, return as-is

2. If community chat:
   - Calls `getCommunityChatByGroupId("123")` → finds `community_chat` row (EXISTS)
   - Calls `createCommunityChat({ communityId: 123 })`
   - **Creates `group_chat` entry** with `group_id = "123"` (if it doesn't exist)
   - **Syncs ALL existing community members** to `group_members` via `syncCommunityMembersToGroup()`
   - Returns `"123"` as the `group_id` to use

### 3. Using Community Chat

**All group operations use `community_id` as `group_id`:**

```javascript
// Sending a message
const groupId = "123";  // community_id as string
await saveGroupMessage({
  groupId: groupId,  // Uses community_id as group_id
  senderId: userId,
  content: "Hello community!"
});

// Checking membership
await isUserInGroup("123", userId);  // Checks group_members table

// Getting messages
await loadGroupMessages("123", limit, offset);  // Uses community_id as group_id
```

### 4. Membership Validation

**Two-level membership check for community chats:**

```javascript
// 1. Check community membership (via social_schema.community_memberships) - AUTHORITATIVE
if (communityChat) {
  const isCommunityMember = await isUserInCommunity(senderId, communityChat.communityId);
  if (!isCommunityMember) {
    // Reject - not a community member
    return;
  }
  
  // 2. Auto-sync if not in group_members yet
  const isMember = await isUserInGroup(actualGroupId, senderId);
  if (!isMember) {
    await syncUserToCommunityGroup(communityChat.communityId, senderId);
    // User is now in group_members!
  }
}
```

**Exact Query Used for Membership Check:**
```sql
SELECT m
FROM social_schema.community_memberships m
WHERE m."communityId" = $1
  AND m."userId" = $2
  AND m.status = 'active'
  AND m."leftAt" IS NULL
  AND m."communityId" IN (SELECT id FROM social_schema.communities)
```

**Why both checks?**
- `community_memberships` = social service membership (authoritative source)
- `group_members` = chat service membership (for group operations)
- Auto-sync ensures they stay in sync

### 5. Auto-Sync When User Joins

When a user joins a community (added to `social_schema.community_memberships`):

**On first chat use:**
- User tries to send message or load messages
- Backend checks: `isUserInCommunity()` → returns `true` (in `community_memberships`)
- Backend checks: `isUserInGroup()` → returns `false` (not in `group_members` yet)
- **Auto-syncs user to `group_members`** via `syncUserToCommunityGroup()`

### 6. Frontend Loading

**When frontend loads:**
1. After WebSocket authentication, frontend calls `getUserGroups()`
2. Backend `handleGetUserGroups()`:
   - Gets regular groups from `group_chats`
   - Gets community chats from `getCommunitiesForUser()` (queries `social_schema.community_memberships`)
   - Combines both into single `user_groups` event
3. Frontend receives all groups (regular + community) and displays them

**Result:** All community chats user belongs to appear in the sidebar automatically!

### 7. Blocking Behavior

**Community chats ignore blocking:**

```javascript
// In handleGetGroupMessages:
if (!communityChat) {
  // Regular group: filter blocked users' messages
  filteredMessages = messages.filter(msg => !allBlocked.has(msg.senderId));
} else {
  // Community chat: show all messages (blocking irrelevant)
  filteredMessages = messages;
}
```

## Key Functions

### `createCommunityChat(communityId)`
- Creates `group_chat` entry using `community_id` as `group_id`
- Called automatically when community chat is first used
- Does NOT create `community_chat` row (social service does that)
- Syncs all existing members to `group_members`
- **Conflict Prevention:** Validates that if a `group_chat` with this ID exists, it's actually a community chat (not a regular group)

### `getCommunityChatByGroupId(groupId)`
- Parses `groupId` as number
- If numeric → looks up in `community_chat` table
- Returns community data or `null` (if not a community)

### `ensureCommunityChatInfrastructure(groupId)`
- Called before using any group chat
- If community chat → ensures `group_chat` entry exists and syncs all members
- Returns `groupId` to use (community_id as string)

### `isUserInCommunity(userId, communityId)`
- Checks `social_schema.community_memberships` for valid membership
- **Exact Query:**
  ```sql
  SELECT m
  FROM social_schema.community_memberships m
  WHERE m."communityId" = $1
    AND m."userId" = $2
    AND m.status = 'active'
    AND m."leftAt" IS NULL
    AND m."communityId" IN (SELECT id FROM social_schema.communities)
  ```
- Used for message validation (authoritative source)

### `getCommunitiesForUser(userId)`
- Gets all communities user belongs to
- Joins: `community_chat` → `communities` → `community_memberships`
- Returns with `groupId = community_id.toString()`

### `isUserCommunityLeader(userId, communityId)`
- Checks if user is a leader of a community
- **Exact Query:**
  ```sql
  SELECT m
  FROM social_schema.community_memberships m
  WHERE m."communityId" = $1
    AND m."userId" = $2
    AND m.status = 'active'
    AND m."leftAt" IS NULL
    AND m.role = 'leader'
  ```
- Returns `true` if user is a leader, `false` otherwise

### `getCommunityName(communityId)`
- Gets community name by ID
- **Exact Query:**
  ```sql
  SELECT name
  FROM social_schema.communities
  WHERE id = $1
  ```
- Returns community name or `null` if not found

### `syncCommunityMembersToGroup(communityId, groupId)`
- Syncs ALL active community members to `group_members` table
- Called when `group_chat` infrastructure is first created
- Maps `role` ('leader' → 'admin', others → 'member')
- Returns count of synced members

### `syncUserToCommunityGroup(communityId, userId)`
- Syncs a SINGLE user to `group_members` when they join
- Called automatically when user tries to use chat but isn't in `group_members` yet
- Verifies user is in `social_schema.community_memberships` (active, not left)
- Returns `true` if synced, `false` if not a member

## Exact SQL Queries Used

All queries use the exact format specified for consistency with the social service:

### 1. Get Community Name
```sql
SELECT name
FROM social_schema.communities
WHERE id = $1
```
**Function:** `getCommunityName(communityId)`

### 2. Check User Membership
```sql
SELECT m
FROM social_schema.community_memberships m
WHERE m."communityId" = $1
  AND m."userId" = $2
  AND m.status = 'active'
  AND m."leftAt" IS NULL
  AND m."communityId" IN (SELECT id FROM social_schema.communities)
```
**Function:** `isUserInCommunity(userId, communityId)`

### 3. Check User Leadership
```sql
SELECT m
FROM social_schema.community_memberships m
WHERE m."communityId" = $1
  AND m."userId" = $2
  AND m.status = 'active'
  AND m."leftAt" IS NULL
  AND m.role = 'leader'
```
**Function:** `isUserCommunityLeader(userId, communityId)`

**Note:** PostgreSQL uses positional parameters (`$1`, `$2`) instead of named parameters (`:cid`, `:uid`), but the query logic matches exactly.

## HTTP API Endpoints

### `GET /api/community/user-communities`

Returns all communities user belongs to.

**Authentication:** Bearer token required

**Response:**
```json
{
  "status": "ok",
  "communities": [
    {
      "id": 1,
      "communityId": 123,
      "groupId": "123",
      "name": "Community Name",
      "slug": "community-slug",
      "visibility": "public",
      "isPro": false,
      "createdBy": 456,
      "createdAt": "2025-01-01T00:00:00Z",
      "userRole": "member",
      "joinedAt": "2025-01-01T00:00:00Z"
    }
  ],
  "count": 1,
  "timestamp": "2025-01-01T00:00:00Z"
}
```

### `GET /api/community/:communityId/members`

Returns all user IDs in a community.

**Authentication:** Bearer token required

**Response:**
```json
{
  "status": "ok",
  "communityId": 123,
  "userIds": ["user1", "user2", "user3"],
  "count": 3,
  "timestamp": "2025-01-01T00:00:00Z"
}
```

## Message Flow Example

**Complete flow from community creation to sending a message:**

### Step 1: Community Created (Social Service)
```sql
-- Social service creates:
INSERT INTO social_schema.communities (id, name) VALUES (123, 'My Community');
INSERT INTO community_chat (community_id) VALUES (123);
-- ✅ community_chat row EXISTS
-- ❌ group_chat row does NOT exist yet
```

### Step 2: User Joins Community
```sql
-- Social service adds:
INSERT INTO social_schema.community_memberships (communityId, userId, status) 
VALUES (123, 456, 'active');
-- ✅ User is in community_memberships
-- ❌ User is NOT in group_members yet
```

### Step 3: Frontend Loads (User Opens Chat App)
```javascript
// Frontend calls getUserGroups()
// Backend:
const communities = await getCommunitiesForUser(userId);
// Queries: community_chat JOIN community_memberships
// Returns: [{ communityId: 123, groupId: "123", name: "My Community" }]
// ✅ Frontend knows groupId = "123" (from community_chat table)
```

### Step 4: User Sends Message
```javascript
// Frontend sends: { type: "send_group_message", groupId: "123", content: "Hello" }
// Backend receives it:
const actualGroupId = await ensureCommunityChatInfrastructure("123");
// 1. Checks if "123" is numeric → YES (community chat)
// 2. Queries community_chat WHERE community_id = 123 → FOUND ✅
// 3. Checks if group_chat WHERE group_id = "123" exists → NOT FOUND ❌
// 4. Creates group_chat entry with group_id = "123" ✅
// 5. Syncs all community members to group_members ✅
// Returns "123"
```

### Step 5: Validate Membership
```javascript
const communityChat = await getCommunityChatByGroupId("123");
// Returns: { communityId: 123, name: "My Community", ... }

// Check authoritative membership using exact query:
await isUserInCommunity(userId, 123);
// Executes: SELECT m FROM social_schema.community_memberships m
//           WHERE m."communityId" = 123 AND m."userId" = userId
//           AND m.status = 'active' AND m."leftAt" IS NULL
//           AND m."communityId" IN (SELECT id FROM social_schema.communities)
// ✅ Returns true if user is in community_memberships

// Check group membership (auto-synced if needed)
await isUserInGroup("123", userId);  // ✅ Now in group_members (synced in step 4)
```

### Step 6: Save and Broadcast Message
```javascript
await saveGroupMessage({
  groupId: "123",  // community_id as string
  senderId: userId,
  content: "Hello"
});
// Message stored in private_messages with recipient_id = "123"

await broadcastToGroup("123", message);
// Uses group_members where group_id = "123" (now populated!)
```

**Key Insight:** The `groupId` exists conceptually (it's `community_id.toString()`), but the `group_chat` infrastructure is created lazily on first use!

## Member Sync Flow

```
User joins community
    ↓
Added to social_schema.community_memberships
    ↓
User opens chat app
    ↓
Frontend calls getUserGroups()
    ↓
Backend returns community chats (from community_memberships)
    ↓
User clicks on community chat
    ↓
Frontend requests messages
    ↓
Backend: ensureCommunityChatInfrastructure()
    ↓
Backend: syncUserToCommunityGroup() (if not already synced)
    ↓
User is now in group_members
    ↓
Messages load successfully!
```

## All Group Chat Features Work

Because `community_id` (as string) equals `group_id`, **ALL** group chat operations work:

- **Messages**: Send, receive, reply, mentions, edit, delete
- **Membership**: Join, leave, roles, permissions
- **Group Operations**: Update info, delete, promote/demote
- **Everything else**: All existing group handlers work without modification

## Visual Diagram

```
┌─────────────────────┐
│  community_chat     │
│  id: 1              │
│  community_id: 123  │──┐
└─────────────────────┘  │
                         │ community_id.toString()
                         │ = "123"
                         ▼
              ┌──────────────────────────┐
              │ social_schema.communities│
              │ id: 123                  │
              │ name: "My Community"     │
              └──────────────────────────┘
                         │
                         │
                         ▼
              ┌──────────────────┐
              │ group_chats      │
              │ group_id: "123"  │ ← community_id.toString()
              │ group_name: "..."│
              └──────────────────┘
                         │
                         │
                    ┌────┴────┬──────────────┬──────────────┐
                    │         │              │              │
                    ▼         ▼              ▼              ▼
            ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
            │group_members │ │group_messages│ │group_roles   │
            │group_id="123"│ │group_id="123"│ │group_id="123"│
            └──────────────┘ └──────────────┘ └──────────────┘
            
            All use community_id.toString() as group_id!
```

## ID Conflict Prevention

### Why Conflicts Are Extremely Rare

**Normal Operation (No Conflicts):**
- **Regular groups** use format: `group_${crypto.randomUUID()}` → e.g., `"group_abc-123-def-456-789"`
- **Community chats** use format: `community_id.toString()` → e.g., `"123"`
- These formats are **inherently different**, so conflicts cannot occur in normal operation

### Potential Conflict Scenarios (Very Rare)

**1. Manual Database Manipulation** ⚠️
   - **Scenario**: Someone manually runs SQL to create a regular group with numeric ID
   ```sql
   INSERT INTO group_chats (group_id, ...) VALUES ('123', ...);  -- Bad!
   ```
   - **Likelihood**: Very rare - requires direct database access and intentional action
   - **Protection**: System validates and throws error if detected

**2. Bug in Code** 🐛
   - **Scenario**: Code bug that creates regular groups with numeric IDs instead of UUIDs
   - **Likelihood**: Extremely rare - would require breaking the `createGroup()` function
   - **Current Code**: `createGroup()` always uses `group_${crypto.randomUUID()}` format
   - **Protection**: Code review, testing, and validation catch this

**3. Legacy Data Migration** 📦
   - **Scenario**: Migrating old data where groups had numeric IDs
   - **Likelihood**: Rare - only if migrating from a different system
   - **Protection**: Migration scripts should validate and convert IDs to UUID format

**4. Malicious Intent** 🔒
   - **Scenario**: Someone intentionally tries to create conflicts
   - **Likelihood**: Very rare - requires database access
   - **Protection**: System validates and prevents corruption

### Conflict Detection & Prevention

**What the system does:**
```javascript
// In createCommunityChat():
if (existingGroup.rows.length > 0) {
  // Verify this group_id belongs to a community chat
  const communityCheck = await dbPool.query(
    `SELECT community_id FROM community_chat WHERE community_id = $1`,
    [communityId]
  );
  
  if (communityCheck.rows.length === 0) {
    // Conflict detected! A regular group exists with this numeric ID
    throw new Error('Group ID conflict...');
  }
}
```

**Protection ensures:**
- ✅ Data integrity: Community chats can't accidentally use a regular group's ID
- ✅ Clear errors: If conflict occurs, you get a descriptive error message
- ✅ Prevention: Regular groups should use UUID format, not numeric IDs
- ✅ Validation: System checks before creating community chat infrastructure

### Summary: Conflict Rarity

| Scenario | Likelihood | Protection |
|----------|-----------|------------|
| Normal operation | **Never** | Different ID formats |
| Manual DB manipulation | **Very Rare** | Validation + error |
| Code bug | **Extremely Rare** | Code always uses UUIDs |
| Legacy migration | **Rare** | Migration validation |
| Malicious intent | **Very Rare** | Validation + error |

**Conclusion**: Conflicts are **extremely rare** in practice because:
1. Regular groups **always** use UUID format (`group_${uuid}`)
2. Community chats **always** use numeric format (`community_id.toString()`)
3. These formats are **mutually exclusive** by design
4. The validation is a **safety net** for edge cases

## Key Points Summary

1. **`community_id` = `group_id`**: Community ID (as string) is used directly as group_id
2. **Independent but uses group mechanics**: `community_chat` is separate, but uses `group_chats`/`group_members` for operations
3. **Membership from social schema**: Authoritative membership is in `social_schema.community_memberships`
4. **Automatic infrastructure**: Group chat infrastructure created on first use
5. **Automatic member sync**: All members synced when infrastructure created, new members synced on first use
6. **Frontend auto-load**: Community chats appear in sidebar via `getUserGroups()` response
7. **No blocking**: Community chats ignore user blocking relationships
8. **All group features work**: Messages, roles, permissions, mentions, etc. all work via group_id
9. **No manual steps**: Everything happens automatically!
10. **Conflict prevention**: System validates ID conflicts and prevents data corruption

## Summary

Community chats work by:
- Using `community_id` directly as `group_id` (converted to string)
- Creating `group_chat` entry automatically when first used
- Syncing all members automatically (bulk on creation, individual on use)
- Checking membership via `social_schema.community_memberships` (authoritative)
- Using all existing group chat handlers/operations without modification
- Ignoring blocking relationships
- Maintaining independence (not a child of group_chat)
- Auto-loading in frontend via `getUserGroups()`

This architecture allows community chats to have all group chat features while remaining independent and supporting additional community-specific features.

