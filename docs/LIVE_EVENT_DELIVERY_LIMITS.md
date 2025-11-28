# Live Event Delivery - Limiting Factors

## Overview
This document explains what limits whether events (messages, reactions, etc.) are received live in both DMs and group chats.

## Critical Requirements for Live Delivery

### 1. **WebSocket Connection State**
**Status**: ✅ Connection must be OPEN
- **Location**: `connectionHandler.js`, `broadcastToGroup()`, `handleSendPrivateMessage()`
- **Check**: `client.ws.readyState === WebSocket.OPEN`
- **Failure**: If connection is CLOSED, CONNECTING, or CLOSING, events are NOT sent
- **Impact**: HIGH - No events delivered if connection is down

### 2. **User Authentication**
**Status**: ✅ User must be authenticated
- **Location**: All handlers check `client.user` exists
- **Check**: `if (!client || !client.user) return`
- **Failure**: Unauthenticated users cannot send or receive events
- **Impact**: HIGH - Blocks all event delivery

### 3. **User Connection Mapping**
**Status**: ✅ User ID must be in `userConnections` Map
- **Location**: `handleAuthentication()` sets `userConnections.set(normalizedUserId, clientId)`
- **Check**: `userConnections.get(normalizedUserId)`
- **Failure**: If user ID not mapped, events cannot find the connection
- **Impact**: CRITICAL - This is the #1 reason events fail to deliver

### 4. **User ID Normalization**
**Status**: ✅ User IDs must be normalized consistently
- **Location**: `normalizeUserId()` converts to string
- **Check**: Both sender and recipient must use same normalization
- **Failure**: Mismatched normalization = connection lookup fails
- **Impact**: HIGH - Silent failures if IDs don't match

### 5. **Group Membership (Groups Only)**
**Status**: ✅ User must be in group's member list
- **Location**: `broadcastToGroup()` calls `getGroupMembers(groupId)`
- **Check**: User must exist in database `group_members` table
- **Failure**: Non-members don't receive group events
- **Impact**: HIGH - Blocks group event delivery

### 6. **Conversation Existence (DMs Only)**
**Status**: ✅ Conversation must exist in memory
- **Location**: `handleSendPrivateMessage()` checks `conversations.has(convId)`
- **Check**: Conversation created on first message
- **Failure**: If conversation not in memory, recipient lookup may fail
- **Impact**: MEDIUM - Usually auto-created, but can fail on edge cases

### 7. **Connection Timeout**
**Status**: ⚠️ Connection must not timeout
- **Location**: `connectionHandler.js` - 5 minute timeout (configurable)
- **Check**: Heartbeat (ping/pong) every 15 seconds resets timeout
- **Failure**: If no activity for 5 minutes, connection closes
- **Impact**: MEDIUM - Only affects idle connections

### 8. **Grace Period for Offline Status**
**Status**: ⚠️ 10-second grace period before marking offline
- **Location**: `connectionHandler.js` - `OFFLINE_GRACE_PERIOD_MS = 10000`
- **Check**: Reconnection within 10 seconds cancels offline timer
- **Failure**: If reconnect takes >10 seconds, user marked offline
- **Impact**: LOW - Only affects reconnection timing

## Event-Specific Requirements

### Direct Messages (DMs)
1. **Sender Requirements**:
   - ✅ Authenticated
   - ✅ WebSocket OPEN
   - ✅ In `userConnections` Map

2. **Recipient Requirements**:
   - ✅ Authenticated
   - ✅ WebSocket OPEN
   - ✅ In `userConnections` Map
   - ✅ User ID normalized correctly
   - ✅ Connection found via `getConnectionId(userConnections, recipientKey)`

3. **Delivery Path**:
   ```
   Sender → handleSendPrivateMessage()
     ↓
   Lookup recipient: getConnectionId(userConnections, recipientKey)
     ↓
   Check: recipientClient.ws.readyState === OPEN
     ↓
   Send: new_private_message event
   ```

### Group Messages
1. **Sender Requirements**:
   - ✅ Authenticated
   - ✅ WebSocket OPEN
   - ✅ Member of group (in database)

2. **Recipient Requirements** (for each member):
   - ✅ Authenticated
   - ✅ WebSocket OPEN
   - ✅ In `userConnections` Map
   - ✅ Member of group (in database)
   - ✅ User ID normalized correctly

3. **Delivery Path**:
   ```
   Sender → handleSendGroupMessage()
     ↓
   Get all members: getGroupMembers(groupId)
     ↓
   For each member:
     - Normalize user ID
     - Lookup: userConnections.get(normalizedUserId)
     - Check: client.ws.readyState === OPEN
     - Send: new_group_message event
   ```

### Group Reactions
1. **Sender Requirements**:
   - ✅ Authenticated
   - ✅ WebSocket OPEN
   - ✅ Member of group

2. **Recipient Requirements**:
   - ✅ Authenticated
   - ✅ WebSocket OPEN
   - ✅ In `userConnections` Map
   - ✅ Member of group

3. **Delivery Path**:
   ```
   Sender → handleAddGroupReaction()
     ↓
   Broadcast: broadcastToGroup(groupId, group_reaction_added)
     ↓
   For each member (except sender):
     - Lookup connection
     - Check WebSocket OPEN
     - Send event
   ```

## Common Failure Points

### 1. **User ID Normalization Mismatch** (Most Common)
**Problem**: Sender uses `"402"`, recipient stored as `402` (number)
**Solution**: Always normalize to string before lookup
**Location**: `normalizeUserId()` must be used consistently

### 2. **Connection Not in Map**
**Problem**: User authenticated but `userConnections` doesn't have entry
**Solution**: Ensure `handleAuthentication()` always updates `userConnections`
**Location**: `messageHandlers.js` - `handleAuthentication()`

### 3. **WebSocket Closed but Not Cleaned Up**
**Problem**: Connection closed but still in `clients` Map
**Solution**: `ws.on('close')` handler removes from Map
**Location**: `connectionHandler.js` - cleanup logic

### 4. **Group Member Not in Database**
**Problem**: User added to group but not in `group_members` table
**Solution**: Ensure `addGroupMember()` saves to database
**Location**: `groupHandlers.js` - `handleAddGroupMember()`

### 5. **Race Condition: Message Before Auth**
**Problem**: Message sent before recipient finishes authentication
**Solution**: Queue messages until authenticated (already implemented)
**Location**: `chatSocketClient.ts` - `queuedAfterAuth`

## Debugging Checklist

When events aren't received live, check:

1. **Connection State**:
   ```javascript
   // Check if WebSocket is OPEN
   client.ws.readyState === WebSocket.OPEN
   ```

2. **User in Connections Map**:
   ```javascript
   // Check if user ID is mapped
   userConnections.has(normalizedUserId)
   ```

3. **Group Membership**:
   ```javascript
   // Check if user is in group
   const members = await getGroupMembers(groupId);
   members.find(m => normalizeUserId(m.userId) === normalizedUserId)
   ```

4. **Normalization**:
   ```javascript
   // Ensure consistent normalization
   normalizeUserId(senderId) === normalizeUserId(recipientId)
   ```

5. **Backend Logs**:
   - `[broadcastToGroup]` logs show connection lookups
   - `[handleSendPrivateMessage]` logs show recipient lookup
   - Check `sentCount` vs `totalMembers` in broadcast logs

## Solutions to Improve Reliability

### 1. **Add Connection Health Check**
- Periodically verify connections are still valid
- Remove stale connections from Map

### 2. **Retry Failed Deliveries**
- Queue failed deliveries
- Retry when connection restored

### 3. **Better Normalization**
- Ensure all user IDs normalized at entry points
- Add validation to catch mismatches

### 4. **Connection Re-sync**
- On reconnect, re-sync all active conversations/groups
- Request latest messages if connection was down

### 5. **Event Acknowledgment**
- Require clients to acknowledge receipt
- Retry unacknowledged events

