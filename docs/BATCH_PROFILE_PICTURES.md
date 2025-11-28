# Batch Profile Pictures API

## Overview
The chat server now provides a batch endpoint for fetching profile picture metadata, reducing the number of HTTP requests from the frontend to the main backend.

## Current Architecture

### Before (No Batching)
- **Frontend**: Makes 2 HTTP requests per profile picture (metadata + raw) to main backend
- **20 conversations**: 40 HTTP requests to main backend
- **Each request**: ~50-200ms latency
- **Total time**: 2-8 seconds for all profile pictures

### After (With Batching)
- **Frontend**: Makes 1 batch request to chat server
- **Chat server**: Batches requests to main backend in parallel
- **20 conversations**: 1 batch request + N parallel requests to main backend
- **Total time**: ~200-500ms for all profile pictures (parallel execution)

## Endpoint: `POST /api/profile-pictures/batch`

### Request
```json
{
  "items": [
    { "type": "user", "id": 402 },
    { "type": "user", "id": 435 },
    { "type": "community", "id": 34 }
  ]
}
```

### Response
```json
{
  "status": "ok",
  "items": [
    {
      "type": "user",
      "id": 402,
      "hasPicture": true,
      "default": false,
      "url": "/api/profile-pictures/user/402/raw"
    },
    {
      "type": "user",
      "id": 435,
      "hasPicture": false,
      "default": true,
      "url": null
    },
    {
      "type": "community",
      "id": 34,
      "hasPicture": true,
      "default": false,
      "url": "/api/profile-pictures/community/34/raw"
    }
  ],
  "count": 3,
  "timestamp": "2025-01-20T12:00:00.000Z"
}
```

## Raw Image Endpoints

After getting batch metadata, frontend can fetch raw images using:

- `GET /api/profile-pictures/user/:userId/raw` - User profile picture
- `GET /api/profile-pictures/community/:communityId/raw` - Community profile picture

These endpoints proxy to the main backend and return the raw image bytes with proper MIME types.

## Performance Impact

### Database Queries
**Current**: Profile pictures are NOT stored in chat server database. They're fetched from main backend API.

**Each profile picture request**:
- 1 HTTP request to main backend (metadata)
- 1 HTTP request to main backend (raw image)
- Main backend makes database queries (not chat server)

**With batching**:
- 1 batch request to chat server
- Chat server makes N parallel HTTP requests to main backend
- Main backend still makes database queries, but requests are parallelized
- **Reduction**: From 2N sequential requests to 1 batch + N parallel requests

### Latency Reduction
- **Before (No Caching)**: 20 conversations × 2 requests × 100ms = 4000ms (sequential)
- **After (With Caching)**:
  - **First request**: 1 batch request + (20/5 chunks × 100ms) = ~400-600ms (chunked parallel)
  - **Subsequent requests**: 1 batch request + (only uncached items/5 chunks × 100ms) = ~50-200ms
  - **If all cached**: ~10-50ms (instant from cache)
- **Improvement**: 
  - First load: ~85-90% faster
  - Subsequent loads: ~95-99% faster (most items cached)
  - Database queries: Reduced by 80-95% on subsequent requests

## Implementation Details

### Chat Server Side
- **In-memory caching**: Profile picture metadata cached for 5 minutes
- **Cache-first strategy**: Checks cache before making HTTP requests to main backend
- **Smart batching**: Separates cached items (instant) from uncached items (chunked)
- Batches uncached metadata requests in **chunks of 5** to avoid overwhelming main backend's database connection pool
- Limits batch size to 50 items to prevent abuse
- Processes chunks sequentially with 10ms delay between chunks
- Handles errors gracefully (returns error for failed items, continues with others)
- Proxies raw image requests to main backend
- **Cache cleanup**: Automatically removes expired entries every minute

### Main Backend Side
- Each metadata request still makes a database query (only for uncached items)
- **With caching**: Most requests are served from cache (0 database queries)
- **Without caching**: Requests are processed in chunks of 5 to prevent database connection pool exhaustion
- With MAX_CONNECTIONS=10, this ensures we don't exhaust the main backend's database connections
- **Concurrency**: Max 5 parallel database queries at a time (instead of 20+)
- **Database query reduction**: 80-95% reduction on subsequent requests (most items cached)

## Frontend Integration

Update `useProfilePictureCache.ts` to use batch endpoint:

```typescript
// Batch fetch metadata for all visible conversations
const items = conversations.map(c => ({
  type: c.isGroup ? 'community' : 'user',
  id: c.isGroup ? getCommunityIdFromGroupId(c.id) : c.otherUser.userId
}));

const response = await fetch(`${CHAT_HTTP_BASE_URL}/api/profile-pictures/batch`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ items })
});

const data = await response.json();
// Cache metadata, then fetch raw images for items with hasPicture: true
```

## Caching Strategy

### Cache Details
- **Duration**: 5 minutes TTL
- **Max Size**: 1000 entries (LRU eviction)
- **Scope**: In-memory on chat server (shared across all requests)
- **Cache Key**: `user:123` or `community:456`
- **Automatic Cleanup**: Expired entries removed every minute

### Cache Benefits
- **First Request**: Normal latency (~400-600ms for 20 items)
- **Subsequent Requests**: 
  - If all cached: ~10-50ms (instant)
  - If partially cached: Only uncached items fetched (~50-200ms)
- **Database Query Reduction**: 80-95% reduction on subsequent requests

## Limitations

1. **Main Backend Still Makes Individual Queries**: The main backend (Java) still makes one database query per uncached profile picture. To fully optimize, the main backend would need a batch endpoint.

2. **Concurrency Limited to 5**: To prevent database connection pool exhaustion (especially with MAX_CONNECTIONS=10), uncached requests are processed in chunks of 5. This is a trade-off between speed and resource usage.

3. **Cache Memory Usage**: In-memory cache uses ~1-2MB for 1000 entries (negligible).

4. **Raw Images Still Individual**: Raw images are still fetched individually (but can be cached on frontend).

5. **Batch Size Limit**: Limited to 50 items per batch to prevent abuse.

## Future Optimization

If the main backend implements a true batch endpoint:
- Single database query for all profile pictures
- Single HTTP request from chat server to main backend
- Even greater performance improvement

## Security

- Requires JWT authentication
- Validates token before processing
- Limits batch size to prevent abuse
- Handles errors gracefully

