# Authentication Troubleshooting Guide

## Quick Diagnostic Tool

Run the diagnostic script to check your authentication setup:

```bash
# Basic check
npm run diagnose:auth

# Check specific user
npm run diagnose:auth 402

# Check specific user with token
npm run diagnose:auth 402 "your-jwt-token-here"
```

This will check:
- ✅ JWT_SECRET configuration
- ✅ Database connection
- ✅ User tables existence
- ✅ User lookup (if userId provided)
- ✅ Token validation (if token provided)
- ✅ Connection pool status
- ✅ Recommendations

## Common Causes of Authentication Failures

### 1. JWT_SECRET Mismatch

**Symptom:** `Invalid token signature` or `JsonWebTokenError`

**Cause:** The `JWT_SECRET` in the chat server's `.env` file doesn't match the Java backend's JWT secret.

**Solution:**
1. Check Java backend's `JwtUtil.java` for the exact secret string
2. Ensure both services use the **exact same** Base64-encoded secret
3. Verify `.env` file has `JWT_SECRET` set correctly
4. Restart chat server after updating `.env`

**Debug:**
```javascript
// In jwtUtils.js, check:
console.log('JWT_SECRET length:', JWT_SECRET?.length);
console.log('JWT_SECRET preview:', JWT_SECRET?.substring(0, 20));
```

### 2. User Not Found in Database

**Symptom:** `User not found in database` error

**Cause:** The user ID from the JWT token doesn't exist in the `users_auth` or `chat_users` table.

**Check:**
```sql
-- Check if user exists
SELECT user_id, username FROM users_auth WHERE user_id = <userId>;
SELECT user_id, username FROM chat_users WHERE user_id = <userId>;
```

**Solution:**
- Ensure user exists in `users_auth` table (created by Java backend)
- Check for type mismatch: JWT might have `sub` as string, but database has `user_id` as INTEGER
- Verify user ID conversion in `getUserDetails()` function

### 3. Token Expired

**Symptom:** `Token expired` or `TokenExpiredError`

**Cause:** JWT token's `exp` claim is in the past.

**Check:**
```javascript
// Decode token to check expiration
const payload = JSON.parse(atob(token.split('.')[1]));
console.log('Token expires:', new Date(payload.exp * 1000));
console.log('Current time:', new Date());
```

**Solution:**
- User needs to log in again to get a fresh token
- Check Java backend's token expiration settings

### 4. JTI Revoked

**Symptom:** `Token has been revoked`

**Cause:** The token's `jti` (JWT ID) is in the `jwt_revocation` table.

**Check:**
```sql
-- Check if JTI is revoked
SELECT * FROM jwt_revocation WHERE jti = '<jti_from_token>';
```

**Solution:**
- User needs to log in again (token was revoked, likely due to logout)
- Clear revocation if needed: `DELETE FROM jwt_revocation WHERE jti = '<jti>';`

### 5. Database Connection Pool Exhaustion ⚠️ **MOST COMMON**

**Symptom:** `Database connection unavailable` or `Database query timeout`

**Cause:** Connection pool exhausted due to app-wide limit of 40 connections.

**Connection Allocation:**
- **Total App Limit:** 40 connections
- **Chat Server:** 10 connections (fixed)
- **Java Backend:** 30 connections (should be configured)

**Why This Happens:**
- Chat server uses 10 connections (authentication, queries, transactions)
- Java backend may be using more than 30 connections
- Total exceeds 40, causing connection pool exhaustion
- Authentication queries timeout waiting for available connections

**Check:**
```bash
# Check chat server pool status
curl http://localhost:3001/health
# Look for database.pool.waitingCount > 0

# Or run diagnostic
npm run diagnose:auth

# Check total database connections
psql -c "SELECT COUNT(*) FROM pg_stat_activity WHERE datname = 'your_db';"
```

**Solution:**

1. **Verify Java backend limit:**
   ```properties
   # In Java application.properties
   spring.datasource.hikari.maximum-pool-size=30  # Must be ≤ 30
   ```

2. **Check total connections:**
   ```sql
   SELECT COUNT(*) FROM pg_stat_activity;
   -- Should be < 40
   ```

3. **If chat server pool exhausted:**
   - Check if Java backend is using > 30 connections
   - Reduce Java backend pool size if needed
   - Optimize queries to release connections faster

4. **If Java backend needs more:**
   - Temporarily reduce chat server: `DB_MAX_CONNECTIONS=5`
   - Increase Java backend: `maximum-pool-size=35`
   - **Total must stay ≤ 40**

**Configuration:**
```env
# Chat Server (.env)
DB_MAX_CONNECTIONS=10   # Fixed limit (10 of 40 total)
DB_MIN_CONNECTIONS=2    # Minimum persistent connections
```

```properties
# Java Backend (application.properties)
spring.datasource.hikari.maximum-pool-size=30  # Fixed limit (30 of 40 total)
spring.datasource.hikari.minimum-idle=5
```

**See:** [Database Connection Limits](./DATABASE_CONNECTION_LIMITS.md) for detailed guide

### 6. User ID Type Mismatch

**Symptom:** User exists but authentication fails silently

**Cause:** JWT has `sub` as string `"402"`, but database query expects integer `402`.

**Check:**
```javascript
// In jwtUtils.js, getUserDetails function
console.log('userId from token:', userId, typeof userId);
console.log('userIdInt after conversion:', userIdInt, typeof userIdInt);
```

**Solution:**
- The code already handles this with `parseInt()`, but verify conversion is working
- Check database column type: `users_auth.user_id` should be INTEGER/BIGINT

### 7. MAX_CONNECTIONS Limit Reached

**Symptom:** Connection rejected before authentication

**Cause:** `MAX_CONNECTIONS=10` limit reached, new connections are rejected.

**Check:**
```bash
# Check current connections
curl http://localhost:3001/health
# Look for connections count
```

**Solution:**
- Close inactive connections
- Increase `MAX_CONNECTIONS` if needed
- Check for connection leaks (connections not being cleaned up)

## Diagnostic Steps

### Step 1: Check Server Logs

Look for these log messages:
```
[JWT] Extracted userId: <userId> jti: <jti>
[JWT] Authentication successful for user: <userId> <username>
[AUTH] Authentication failed for client <clientId>: { error: '...' }
```

### Step 2: Verify JWT Token

```javascript
// Decode token (client-side or server-side)
const token = 'your-jwt-token';
const parts = token.split('.');
const payload = JSON.parse(atob(parts[1]));

console.log('Token payload:', payload);
console.log('User ID (sub):', payload.sub);
console.log('JTI:', payload.jti);
console.log('Expires:', new Date(payload.exp * 1000));
console.log('Current time:', new Date());
console.log('Is expired:', new Date(payload.exp * 1000) < new Date());
```

### Step 3: Test Database Query

```sql
-- Test user lookup
SELECT ua.user_id, ua.username, ua.email, upi.first_name, upi.last_name
FROM users_auth ua
LEFT JOIN user_profile_info upi ON ua.user_id = upi.user_id
WHERE ua.user_id = 402;  -- Replace with actual user ID

-- Test JTI revocation
SELECT * FROM jwt_revocation WHERE jti = '<jti_from_token>';
```

### Step 4: Check Environment Variables

```bash
# Verify JWT_SECRET is set
echo $JWT_SECRET  # Should match Java backend

# Verify database connection
echo $DATABASE_URL  # Or DB_HOST, DB_USER, etc.
```

### Step 5: Test Authentication Manually

```javascript
// Test JWT verification
const { verifyJWTWithBackend } = require('./security/jwtUtils');
const { dbPool } = require('./config/database');

const result = await verifyJWTWithBackend('your-token', '127.0.0.1', dbPool);
console.log('Verification result:', result);
```

## Quick Fixes

### Fix 1: JWT_SECRET Mismatch
```bash
# 1. Get secret from Java backend
# 2. Update .env file
JWT_SECRET=your-exact-secret-from-java-backend

# 3. Restart server
node server.js
```

### Fix 2: User Not Found
```sql
-- Ensure user exists
INSERT INTO users_auth (user_id, username, email, password_hash)
VALUES (402, 'testuser', 'test@example.com', 'hashed_password')
ON CONFLICT (user_id) DO NOTHING;
```

### Fix 3: Database Connection Pool Exhausted
```bash
# Increase pool size in .env
DB_MAX_CONNECTIONS=50  # Instead of 10
DB_MIN_CONNECTIONS=5   # Instead of 0

# Restart server
```

### Fix 4: Clear Revoked Tokens
```sql
-- If token was incorrectly revoked
DELETE FROM jwt_revocation WHERE jti = '<jti>';
```

## Enhanced Error Logging

The server now logs detailed error information:

```
[JWT] Token validation failed: Invalid signature or expired
[JWT] Token expired: { expiredAt: '...', currentTime: '...' }
[JWT] User not found in database: 402
[JWT] Token revoked: <jti>
[AUTH] Authentication failed for client <clientId>: { error: '...', clientIp: '...' }
```

Check server console for these messages to identify the exact issue.

## Common Scenarios

### Scenario 1: "User does not exist" but user is valid
- **Check:** User ID type (string vs integer)
- **Fix:** Ensure `getUserDetails()` properly converts string to integer
- **Verify:** User exists in `users_auth` table with correct `user_id`

### Scenario 2: Authentication works sometimes but fails other times
- **Check:** Database connection pool exhaustion
- **Fix:** Increase `DB_MAX_CONNECTIONS` or reduce concurrent connections
- **Verify:** Check `/health` endpoint for database connection status

### Scenario 3: Token works in Java backend but fails in chat server
- **Check:** JWT_SECRET mismatch
- **Fix:** Ensure both services use identical secret
- **Verify:** Decode token and verify signature manually

### Scenario 4: Authentication fails after server restart
- **Check:** Environment variables loaded correctly
- **Fix:** Verify `.env` file is in correct location and loaded
- **Verify:** Check server startup logs for JWT_SECRET loading

## Still Having Issues?

1. **Enable debug logging:**
   ```bash
   LOG_LEVEL=debug node server.js
   ```

2. **Check all logs:**
   - Server console output
   - Database query logs
   - WebSocket connection logs

3. **Test with known good token:**
   - Use a token that works in Java backend
   - Test same token in chat server
   - Compare results

4. **Verify database schema:**
   - Ensure `users_auth` table exists
   - Ensure `user_profile_info` table exists
   - Ensure `jwt_revocation` table exists

