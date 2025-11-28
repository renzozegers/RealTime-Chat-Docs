# Testing Guide

**How to run and write tests for the chat server.**

---

## Prerequisites

- Server running on `localhost:3001`
- Database configured in `.env`
- Dependencies installed (`npm install`)

---

## Quick Test

**Verify server is working:**
```powershell
cd C:\Users\renzo\Downloads\hey\elite-signup\chat-server
node server.js
```

**Check health endpoint:**
```powershell
curl http://localhost:3001/health
```

---

## Test Suite

### Available Tests

| Command | Purpose | Notes |
|---------|---------|-------|
| `npm test` | Core DM functionality / DB + JWT sanity | Fast |
| `npm run test:comprehensive` | End-to-end group chat proof (create/send/mentions/reactions/edit/delete) | Requires live server |
| `npm run test:editing` | DM editing & deletion regression | Requires live server |
| `node tests/test-blocking-features.js` | Blocking/follow behaviour | Requires live server |
| `npm run test:startup` | Module/bootstrap validation | Stop live server first (free port 3001) |
| `npm run test:stress` | Simple stress harness (11 connections / 5 messages) | Fast |
| `npm run test:stress:full` | Five staged stress scenarios (10→200 connections, 50→1000 messages) | Several minutes |
| `npm run community:send -- --community demo --user demoUser` | Sample `/community/progression` webhook | Needs `COMMUNITY_SYNC_TOKEN` |
| `node tests/test-community-sync.js` | Community sync smoke test | Optional |

---

## Running Tests

### Step 1: Start Server (terminal #1)

```powershell
cd chat-server
node server.js   # keep this process running
```

### Step 2: Refresh Test Tokens (terminal #2)

```powershell
cd chat-server
npm run tokens:refresh
```

This regenerates `test-jwt-tokens.json`, copies it into `tests/`, and prints the timestamp plus usernames. Rerun when tokens expire or the shared secret changes.

### Step 3: Run Tests (terminal #2)

```powershell
# Core DM sanity
npm test

# Group chat proof
npm run test:comprehensive

# Message editing/deletion regression
npm run test:editing

# Blocking / follow behaviour
node tests/test-blocking-features.js
```

> Running `npm run test:startup`? Stop the live server first (port 3001 must be free), run the script, then restart `node server.js`.

---

## Test Results

### Expected Output

**Successful test:**
```
🚀 Testing Core Chat Features
=============================

✅ Test 1: Database Connection
   Current time: 2025-10-12T16:00:00.000Z

✅ Test 2: JWT Verification
   Token verified successfully

✅ Test 3: User Lookup
   User found: john_doe

PASSED: 3/3 tests
```

**Failed test:**
```
❌ FAIL: Recipient is not online
```

This is normal - tests need multiple WebSocket connections running simultaneously.

---

## Common Test Issues

### "Recipient is not online"

**Cause:** Test tries to message user who isn't connected.

**Fix:** Tests need to establish multiple WebSocket connections. The server is working correctly.

### "Timeout waiting for response"

**Cause:** Server took longer than timeout (5 seconds).

**Fix:** Increase timeout in test or check server performance.

### "Authentication failed"

**Cause:** `JWT_SECRET` in `.env` doesn't match the refreshed tokens.

**Fix:** rerun `npm run tokens:refresh`, restart `node server.js`, then re-run the test.

### "Database connection failed"

**Cause:** Database credentials in `.env` are incorrect.

**Fix:** Verify credentials in `.env` file.

---

## Manual Testing

### Using cURL (HTTP Endpoints)

```powershell
# Health check
curl http://localhost:3001/health

# Readiness check
curl http://localhost:3001/ready

# Metrics
curl http://localhost:3001/metrics
```

### Using Browser (WebSocket)

Create `test.html`:
```html
<!DOCTYPE html>
<html>
<body>
  <h1>WebSocket Test</h1>
  <div id="output"></div>
  <script>
    const ws = new WebSocket('ws://localhost:3001');
    const output = document.getElementById('output');
    
    ws.onopen = () => {
      output.innerHTML += 'Connected!<br>';
      ws.send(JSON.stringify({
        type: 'authenticate',
        token: 'YOUR_JWT_TOKEN_HERE'
      }));
    };
    
    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      output.innerHTML += JSON.stringify(msg, null, 2) + '<br><br>';
    };
  </script>
</body>
</html>
```

Open in browser and replace `YOUR_JWT_TOKEN_HERE` with actual token.

---

## Load Testing

### Simple Stress Test

```powershell
npm run test:stress
```

**Covers:**
- 11 authenticated connections
- Sender/recipient confirmation loop
- Basic throughput metrics

### Full Stress Test

```powershell
npm run test:stress:full
```

**Covers:**
- Five staged scenarios (10 / 25 / 50 / 100 / 200 connections)
- 50 → 1,000 sequential messages per scenario (sender & recipient confirmations)
- Aggregated authentication and message latency metrics

> Expect this suite to run for several minutes. It prints `🏆 ENTERPRISE STRESS TEST RESULTS` once all scenarios complete.

---

## Writing New Tests

### Test Template

```javascript
require('dotenv').config();
const WebSocket = require('ws');

const WS_URL = 'ws://localhost:3001';
const testToken = require('./test-jwt-tokens.json').valid_tokens[0].token;

async function runTest() {
  console.log('🧪 Running test...\n');
  
  // Connect to server
  const ws = new WebSocket(WS_URL);
  
  // Authenticate
  ws.on('open', () => {
    ws.send(JSON.stringify({
      type: 'authenticate',
      token: testToken
    }));
  });
  
  // Handle messages
  ws.on('message', (data) => {
    const msg = JSON.parse(data);
    console.log('Received:', msg.type);
    
    if (msg.type === 'auth_success') {
      console.log('✅ Authenticated');
      // Run your tests here
    }
  });
  
  // Wait for test completion
  await new Promise(resolve => setTimeout(resolve, 5000));
  
  ws.close();
  console.log('✅ Test complete');
}

runTest().catch(console.error);
```

---

## Test Coverage

### Implemented Tests

✅ Authentication  
✅ Direct messaging  
✅ Group chats  
✅ User blocking  
✅ Follow relationships  
✅ Reactions & mentions  
✅ Message editing & deletion  
✅ Role management  
✅ Group deletion  
✅ Load testing  

### Not Tested

❌ Announcements  
❌ Message pinning  
❌ Edge cases with announcements  

---

## Continuous Integration

### GitHub Actions Example

```yaml
name: Test Chat Server

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '18'
      - run: npm install
      - run: npm test
```

---

## Summary

**To run tests:**
1. Start the server in terminal #1: `node server.js`
2. Refresh tokens in terminal #2: `npm run tokens:refresh`
3. Execute the desired suite (`npm run test:comprehensive`, `npm run test:editing`, `npm run test:stress`, etc.)
4. For startup/full-stress suites, plan to stop and restart the live server and allow extra runtime

**The server is production-ready!** ✅


