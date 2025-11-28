# Database Connection Limits

## Overview

The application has a **total database connection limit of 40 connections** shared across all services.

### Connection Allocation

| Service | Max Connections | Purpose |
|---------|----------------|---------|
| **Chat Server** | 10 | Authentication, queries, transactions |
| **Java Backend** | 30 | REST API, business logic, user management |
| **Total** | **40** | **App-wide limit** |

## Chat Server Configuration

### Default Settings

```env
DB_MAX_CONNECTIONS=10   # Chat server limit (10 of 40 total)
DB_MIN_CONNECTIONS=2    # Minimum persistent connections
```

### Why 10 Connections?

**Chat Server Usage:**
- Authentication queries: ~2-3 connections
- Message operations: ~2-3 connections
- Profile picture queries: ~1-2 connections
- Transactions: ~1-2 connections
- Buffer for spikes: ~2-3 connections
- **Total: ~10 connections**

This leaves **30 connections** for the Java backend and other services.

## Java Backend Configuration

**IMPORTANT:** Ensure your Java backend is configured to use **maximum 30 connections**.

### HikariCP Configuration (Java)

```properties
# In application.properties or application.yml
spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=2000
```

### Why 30 Connections?

**Java Backend Usage:**
- REST API endpoints: ~5-10 connections
- User management: ~3-5 connections
- Business logic: ~5-10 connections
- Buffer for spikes: ~5-10 connections
- **Total: ~30 connections**

## Monitoring Connection Usage

### Check Chat Server Pool Status

```bash
# Via health endpoint
curl http://localhost:3001/health | jq '.database.pool'

# Or run diagnostic
npm run diagnose:auth
```

**Output:**
```json
{
  "totalCount": 10,
  "idleCount": 5,
  "waitingCount": 0
}
```

### Check Total Database Connections

```sql
-- Check all active connections
SELECT 
  datname,
  usename,
  application_name,
  state,
  COUNT(*) as connection_count
FROM pg_stat_activity
WHERE datname = 'your_database_name'
GROUP BY datname, usename, application_name, state
ORDER BY connection_count DESC;
```

**Expected:**
- Chat server: ~5-10 connections
- Java backend: ~10-30 connections
- **Total: < 40 connections**

## Troubleshooting

### Issue: Connection Pool Exhausted

**Symptom:**
```
Database connection pool exhausted
Database query timeout
```

**Causes:**
1. Chat server using more than 10 connections
2. Java backend using more than 30 connections
3. Total exceeds 40 connections

**Solutions:**

1. **Check chat server usage:**
   ```bash
   curl http://localhost:3001/health | jq '.database.pool'
   ```
   - If `waitingCount > 0`: Pool is exhausted
   - If `totalCount > 10`: Configuration issue

2. **Check Java backend configuration:**
   - Verify `maximum-pool-size=30` in Java config
   - Check Java backend logs for connection pool warnings

3. **Check total database connections:**
   ```sql
   SELECT COUNT(*) FROM pg_stat_activity WHERE datname = 'your_db';
   ```
   - Should be < 40

4. **Optimize connection usage:**
   - Reduce `DB_MIN_CONNECTIONS` if idle connections are high
   - Ensure connections are released promptly
   - Check for connection leaks

### Issue: Authentication Failures

**Symptom:** Valid users can't authenticate

**Cause:** Connection pool exhausted, authentication queries timeout

**Solution:**
1. Check if chat server pool is exhausted
2. Verify Java backend isn't using too many connections
3. Temporarily increase `DB_MAX_CONNECTIONS` to 15 (if Java backend uses < 25)
4. **Better:** Optimize Java backend to use fewer connections

### Issue: Slow Queries

**Symptom:** Database queries are slow

**Cause:** Waiting for available connections

**Solution:**
1. Check `waitingCount` in pool status
2. If > 0, connections are waiting
3. Optimize query performance (add indexes)
4. Reduce connection acquisition timeouts

## Best Practices

### 1. Connection Pool Sizing

**Chat Server:**
- `DB_MAX_CONNECTIONS=10` (fixed)
- `DB_MIN_CONNECTIONS=2-5` (adjust based on idle usage)

**Java Backend:**
- `maximum-pool-size=30` (fixed)
- `minimum-idle=5-10` (adjust based on usage)

### 2. Connection Monitoring

Monitor connection usage regularly:
```bash
# Daily check
npm run diagnose:auth

# Or via health endpoint
curl http://localhost:3001/health
```

### 3. Connection Leak Detection

Watch for:
- `waitingCount` consistently > 0
- `totalCount` always at max
- Slow queries during peak times

### 4. Optimization

- Use connection pooling efficiently
- Release connections promptly
- Use transactions to reduce connection time
- Batch operations when possible

## Configuration Examples

### Development (Low Load)

```env
# Chat Server
DB_MAX_CONNECTIONS=10
DB_MIN_CONNECTIONS=2

# Java Backend
spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.minimum-idle=5
```

### Production (High Load)

```env
# Chat Server
DB_MAX_CONNECTIONS=10
DB_MIN_CONNECTIONS=3

# Java Backend
spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.minimum-idle=10
```

## Emergency: Temporarily Increase Limit

If you need more connections temporarily:

1. **Increase chat server (if Java backend uses < 25):**
   ```env
   DB_MAX_CONNECTIONS=15  # Temporary increase
   ```
   - **IMPORTANT:** Reduce Java backend to 25 to stay within 40 total

2. **Or increase Java backend (if chat server uses < 5):**
   ```properties
   spring.datasource.hikari.maximum-pool-size=35
   ```
   - **IMPORTANT:** Reduce chat server to 5 to stay within 40 total

3. **Monitor total connections:**
   ```sql
   SELECT COUNT(*) FROM pg_stat_activity;
   ```

## Summary

- **Total App Limit:** 40 connections
- **Chat Server:** 10 connections (fixed)
- **Java Backend:** 30 connections (fixed)
- **Monitor:** Use health endpoint and diagnostic script
- **Optimize:** Reduce connection time, batch operations

**Key Rule:** Chat server (10) + Java backend (30) = 40 total ✅

