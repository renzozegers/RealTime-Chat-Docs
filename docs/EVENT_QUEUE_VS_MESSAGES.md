# Event Queue vs Messages - Understanding the Difference

## Critical Distinction

**MESSAGES** and **EVENT QUEUE** are TWO SEPARATE SYSTEMS:

### 1. MESSAGES (Actual Chat Messages)
- **Stored in**: `private_messages` table (permanent storage)
- **Purpose**: The actual text messages users send to each other
- **Persistence**: **FOREVER** - messages are never deleted (unless user deletes them)
- **How they're loaded**: 
  - When user opens a conversation → `get_private_messages` endpoint loads from database
  - Can load messages from ANY time period (days, weeks, months, years ago)
  - Messages are loaded via pagination (limit/offset)

**Example**: 
- User sends "Hello" → Saved to `private_messages` table
- User can load this message 6 months later → Still there!

### 2. EVENT QUEUE (Live Events Only)
- **Stored in**: `event_queue` table (temporary storage)
- **Purpose**: LIVE EVENTS that happened while user was offline
- **What gets queued**:
  - Reactions (emoji reactions to messages)
  - Group membership changes (member added/removed)
  - Group info updates
  - Message edits/deletes (the edit/delete event, not the message itself)
  - Typing indicators (if we implement them)
- **Persistence**: 
  - Delivered events: Cleaned up after 7 days
  - Undelivered events: Cleaned up after 30 days
- **How they're loaded**: 
  - Only when user RECONNECTS (not when opening conversation)
  - Delivered automatically via WebSocket
  - Then deleted from queue

**Example**:
- User is offline
- Someone reacts to their message → Reaction event queued
- User reconnects → Reaction event delivered → Event deleted from queue
- The MESSAGE itself is still in `private_messages` table (permanent)

## Why Two Systems?

### Messages (Permanent)
```
User sends message → Saved to database → Can load anytime
```
- Messages are the core content
- Users need to see message history when opening conversations
- Messages are loaded on-demand (not pushed)

### Event Queue (Temporary)
```
User offline → Reaction happens → Event queued → User reconnects → Event delivered → Event deleted
```
- Events are "notifications" about things that happened
- Only needed if user was offline when it happened
- Once delivered, we don't need to keep them

## Cleanup Explained

### `cleanupOldDeliveredEvents()` - 7 days
- **ONLY cleans `event_queue` table**
- **Does NOT touch `private_messages` table**
- **Why 7 days?** 
  - Event was already delivered to user
  - User already saw the reaction/change
  - No need to keep it forever
  - Saves database space

### `cleanupOldUndeliveredEvents()` - 30 days
- **ONLY cleans `event_queue` table**
- **Does NOT touch `private_messages` table**
- **Why 30 days?**
  - User hasn't been online for 30 days
  - Likely not coming back soon
  - Prevents database bloat

## How Messages Are Loaded (Past 7 Days)

Messages are **NOT** affected by event queue cleanup!

When user opens a conversation:
1. Frontend calls `get_private_messages` WebSocket command
2. Backend loads from `private_messages` table (NOT event_queue)
3. Can load messages from ANY time period
4. Messages are permanent - never auto-deleted

**Example Timeline**:
```
Day 1: User sends message "Hello" → Saved to private_messages
Day 8: Event queue cleanup runs → Does NOT affect "Hello" message
Day 30: User opens conversation → Loads "Hello" from private_messages ✅
```

## In-Memory vs Database Events

### In-Memory Events (eventQueues Map)
- **Storage**: JavaScript Map in server memory
- **Speed**: Very fast (no database query)
- **Persistence**: Lost on server restart
- **Use case**: Recent disconnects (user reconnects within minutes)
- **Retention**: 7 days (then cleaned up)

### Database Events (event_queue table)
- **Storage**: PostgreSQL table
- **Speed**: Slower (database query)
- **Persistence**: Survives server restarts ✅
- **Use case**: Long disconnects (user offline for days/weeks)
- **Retention**: 30 days for undelivered, 7 days for delivered

### Why Both?
- **In-memory**: Fast delivery for quick reconnects
- **Database**: Reliable delivery even after server restarts
- **Redundancy**: If one fails, other still works

## Summary

| Aspect | Messages | Event Queue |
|--------|----------|-------------|
| **Table** | `private_messages` | `event_queue` |
| **Purpose** | Actual chat content | Live event notifications |
| **Persistence** | Forever | 7-30 days |
| **Loaded when** | Opening conversation | Reconnecting |
| **Cleanup affects** | Never auto-deleted | Yes, after delivery |
| **Example** | "Hello" message | Reaction to "Hello" |

**Key Point**: Event queue cleanup does NOT affect messages. Messages are permanent and loaded separately!

