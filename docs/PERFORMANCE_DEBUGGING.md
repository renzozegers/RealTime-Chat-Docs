# Performance Debugging Guide

## Issue: Conversations Taking 4 Seconds to Load

### Backend Performance Logging Added

I've added detailed performance logging to `handleGetPrivateMessages` to identify where the delay is happening.

### What to Check

1. **Check Server Logs** - Look for `[PERF]` log messages:
   ```
   [PERF] loadPrivateMessagesFromDatabase took XXXms
   [PERF] COUNT query took XXXms
   [PERF] handleGetPrivateMessages total time: XXXms
   ```

2. **Expected Times (After Optimizations):**
   - `loadPrivateMessagesFromDatabase`: **<200ms** (should be fast now)
   - `COUNT query`: **<100ms** (should be fast now)
   - `JSON stringify`: **<50ms** (usually very fast)
   - **Total backend time**: **<350ms**

3. **If Backend is Fast (<500ms):**
   - The delay is likely in the frontend
   - Check browser DevTools Network tab for WebSocket message timing
   - Check if frontend is waiting for something

4. **If Backend is Slow (>1000ms):**
   - Database query might still be slow (despite optimizations)
   - Check if indexes are properly applied
   - Check database connection pool status

### Frontend Potential Issues

1. **Conversation Not in Store** - Line 248-251 in `ChatLayout.tsx`:
   ```typescript
   if (!conversation) {
     console.log("[DEBUG] Conversation not yet in store, waiting...")
     return // Waits indefinitely until conversation exists
   }
   ```
   - If conversation summary hasn't loaded yet, it will wait
   - Check if `loadConversationSummariesForUser` is being called

2. **Missing recipientId** - Line 258 in `ChatLayout.tsx`:
   ```typescript
   if (activeConvId !== lastLoadedConvRef.current && recipientId) {
     // Only loads if recipientId exists
   }
   ```
   - If `recipientId` can't be extracted, it won't load
   - Check `getRecipientIdFromConversation` function

3. **WebSocket Not Connected** - Line 322/336:
   - If WebSocket is closed, `getPrivateMessages` will queue the command
   - Check WebSocket connection status
   - Messages won't load until WebSocket reconnects

### How to Debug

1. **Open Browser DevTools:**
   - Network tab → WS (WebSocket) filter
   - Look for `get_private_messages` command
   - Check timing of request and response

2. **Check Console Logs:**
   - Look for `[DEBUG]` messages from frontend
   - Look for `[PERF]` messages from backend
   - Look for errors or warnings

3. **Check Server Logs:**
   - Look for `[PERF]` timing messages
   - Look for database query errors
   - Check connection pool status

### Common Issues

1. **Conversation Summary Not Loaded:**
   - Frontend waits for conversation to exist in store
   - If conversation list hasn't loaded, clicking conversation will wait
   - **Fix:** Ensure `loadConversationSummariesForUser` is called on mount

2. **WebSocket Queue:**
   - If WebSocket is closed, commands are queued
   - Won't execute until WebSocket reconnects
   - **Fix:** Ensure WebSocket is connected before clicking conversations

3. **Slow Database Query:**
   - Despite optimizations, query might still be slow
   - **Fix:** Check if indexes are applied, check EXPLAIN ANALYZE

4. **Frontend Rendering Delay:**
   - React might be slow to render messages
   - **Fix:** Check React DevTools Profiler for slow renders

### Next Steps

1. Restart server to apply performance logging
2. Open a conversation
3. Check server logs for `[PERF]` messages
4. Report back with timing breakdown
5. Check browser DevTools Network tab for WebSocket timing

