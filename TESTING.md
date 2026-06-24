# Backend API Automation Testing Guide

## Overview

This document describes the backend API automation test suite built with **Postman/Newman** for the Orta Shift Manager API. The suite covers authentication, authorization, and shift management workflows.

**Test Framework**: Postman + Newman (JavaScript execution via npm)  
**Collection Format**: JSON (Collection v2.1)  
**Environment**: YAML (Postman v12 native format; JSON via older versions if needed)  
**Test Location**: `postman/` folder (separate from source code)  
**CI/CD**: GitHub Actions with htmlextra & JSON reporting  
**Reports**: https://dhirensinghem.github.io/automation-engineer-test-be/

---

## Quick Start

### Prerequisites
- Node.js 18+
- npm
- Newman installed: `npm install -D newman newman-reporter-htmlextra`

### Installation
```bash
npm install
npm install -D newman newman-reporter-htmlextra
```

### Run Tests
```bash
# Run full collection (all 38 requests)
npm run test:api

# Run with Postman Desktop (interactive)
# Import collection: postman/collections/orta-api-collection.json
# Import environment: postman/environments/dev.postman_environment.json
# Click "Run" folder by folder
```

### View Reports
```bash
# After running npm run test:api
open postman/reports/report.html
```

### Environment Setup
Create `.env` file at root:
```
# Newman uses these via the workflow; local testing reads from postman environment
NEWMAN_BASE_URL=https://orta-full-stack-dev-test-be-63gp.onrender.com/api
ADMIN_EMAIL=john@example.com
ADMIN_PASSWORD=StrongPass123!
WORKER_EMAIL=testUser1@example.com
WORKER_PASSWORD=StrongPass123!
```

---

## Project Structure

```
automation-engineer-test-be/
├── .env                              # Environment variables
├── .env.example                      # Template
├── package.json                      # Dependencies & scripts
├── .github/
│   └── workflows/
│       └── newman-tests.yml          # GitHub Actions CI/CD
└── postman/
    ├── collections/
    │   └── orta-api-collection.json  # All 38 requests (Auth, Authz, Shifts)
    ├── environments/
    │   └── dev.postman_environment.json  # Variables & secrets
    └── reports/                      # Generated reports (gitignored)
        ├── report.html               # Interactive htmlextra report
        └── report.json               # Machine-readable JSON report
```

- Automation related scripts written in main package.json file
- Similarly, main .gitignore & .env files used to accomodate automation related artifacts

---

### Test Execution Order

**Critical**: Requests execute sequentially; order matters for generating artifacts to be used in subsequent requests:

1. **Auth Folder** (first)
   - Registers new user → captures `newUserId` + `newUserToken`
   - Logs in admin → captures `adminToken`
   - Logs in worker → captures `workerToken`
   - Tests error cases (invalid, duplicate, weak password)

2. **Authorization Folder** (second)
   - Uses tokens from Auth folder
   - Tests role-based access control
   - Promotes new user to admin (requires `newUserId` from Auth)

3. **Shifts Folder** (third)
   - Uses `adminToken`, `newUserId`, `newUserToken` from previous folders
   - Creates, reads, updates, deletes shifts
   - Tests state transitions (clock-in/out, cancel)
   - Tests pagination, batch operations

**Running Folders Out of Order**: Will fail with missing variables.

### Request Structure

Each request includes:

**Pre-request Script**:
- Generate unique data (emails, dates)
- Set up request-specific variables
- Example:
  ```javascript
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  const isoDate = tomorrow.toISOString();
  pm.environment.set('shiftDate', isoDate);
  ```

**Test Script** (assertions):
- Validate status code
- Capture response data for chaining
- Assert on response structure
- Example:
  ```javascript
  pm.test('Status code is 200', () => {
    pm.response.to.have.status(200);
  });
  pm.test('Response contains token', () => {
    const body = pm.response.json();
    pm.expect(body).to.have.property('token');
    pm.environment.set('adminToken', body.token);
  });
  ```

### Variable Management

**Environment Variables** (shared across requests):
- `baseUrl` — API base URL
- `adminToken` — Captured from login
- `workerToken` — Captured from login
- `newUserId` — Captured from registration
- `newUserToken` — Captured from registration
- `shiftId` — Captured from shift creation

**Pre-request Variables** (request-specific):
- `shiftDate` — Generated per request
- `newUserEmail` — Generated per request

---

### What's Tested

**Happy Paths**:
- Registration with unique email
- Login with valid credentials
- Admin operations (create, promote, delete shifts)
- Worker operations (view own shifts, clock in/out)

**Error Handling**:
- Validation errors (missing fields, weak password, past date)
- Authorization errors (non-admin trying admin actions)
- Authentication errors (missing token, malformed token, invalid scheme)
- State errors (clock-in twice, clock-out without clock-in)

**Business Logic**:
- User role defaults to "worker"
- Promotion to admin works correctly
- Shift state transitions (pending → in-progress → completed)
- Date validation (rejects past dates)
- Pagination parameters respected

### Design Decisions (Trade-offs Documented)

#### 1. **Smoke/Regression Tagging**
- **Status**: Not implemented as separate folders
- **Reason**: Folder-based filtering adds complexity; all the requests are essential for E2E validation
- **Future**: If suite grows to 100+ requests, implement folder-based categorization:
  ```yaml
  # Create separate folders in Postman
  - E2E Smoke Flow (critical path only)
  - Full Regression Suite (all tests)
  ```
  Then run via Newman CLI:
  ```bash
  newman run collection.json --folder "E2E Smoke Flow"
  ```

#### 2. **Multi-Environment Support**
- **Status**: Single `dev.environment.json` for dev
- **Reason**: Test backend is single Render deployment; multi-env config adds overhead
- **Future**: Create environment files per deployment:
  ```
  dev.environment.json   (current)
  staging.environment.json
  prod.environment.json
  ```
  Then run: `newman run collection.json -e staging.postman_environment.json`

#### 3. **Collection Format (JSON vs YAML)**
- **Current**: JSON format (Postman Collection v2.1 standard)
- **Postman Version**: v12+ stores native YAML format internally
- **Why JSON Here**: Universal compatibility; any Newman version can consume JSON
- **If Needed**: Downgrade Postman to v10 or earlier for native JSON export/sync
- **Trade-off**: YAML is readable in Postman UI; JSON is tool-agnostic

#### 4. **Request Isolation vs Chaining**
- **Status**: Requests are chained (dependent on execution order)
- **Reason**: Real-world E2E flow requires state (tokens, IDs)
- **Benefit**: Tests actual user journeys, not isolated endpoints
- **Limitation**: Cannot run requests out of order; folder run order matters

#### 5. **Error Code Validation**
- **Status**: Tests assert on `errorCode` field + status code
- **Reason**: API returns structured errors (`{ statusCode, errorCode, message }`)
- **Example**:
  ```javascript
  pm.expect(body.errorCode).to.eql('ADMIN_ROLE_REQUIRED');
  pm.expect(body.statusCode).to.eql(403);
  ```
- **Benefit**: Catches API contract changes; generic "403" is insufficient

---

## Running Tests

### Local Execution

**Run full collection (all folders, all requests)**:
```bash
npm run test:api
```

**Run specific folder** (e.g., Auth only):
```bash
npx newman run postman/collections/orta-api-collection.json \
  -e postman/environments/dev.postman_environment.json \
  --folder "Auth"
```

**Run with verbose logging**:
```bash
npx newman run postman/collections/orta-api-collection.json \
  -e postman/environments/dev.postman_environment.json \
  -r cli
```

**Run and generate HTML report only** (no JSON):
```bash
npx newman run postman/collections/orta-api-collection.json \
  -e postman/environments/dev.postman_environment.json \
  -r htmlextra --reporter-htmlextra-export report.html
```

### CI/CD Execution

**GitHub Actions** runs automatically on:
- Push to `main` or `develop`
- PR to `main` or `develop`
- Daily at 3 AM UTC
- Manual trigger via Actions tab

**Secrets required** (Settings → Secrets):
- `ADMIN_EMAIL`
- `ADMIN_PASSWORD`
- `WORKER_EMAIL`
- `WORKER_PASSWORD`
- `NEWMAN_BASE_URL` (optional, defaults to https://orta-full-stack-dev-test-be-63gp.onrender.com/api)

**View Results**:
- GitHub Actions tab → latest run
- View logs in real-time
- Download artifacts: `newman-html-report-{run}` and `newman-json-report-{run}`
- View on GitHub Pages: https://dhirensinghem.github.io/automation-engineer-test-be/

---

## Test Reports

### Local HTML Report
After running `npm run test:api`:
```bash
open postman/reports/report.html
```

**Report shows**:
- Total requests, passes, failures
- Per-request status code & response time
- Test assertion results
- Request/response body (if failures)
- Execution timeline

### JSON Report
Machine-readable output at `postman/reports/report.json`:
- Used for CI/CD integrations
- Can be parsed for metrics dashboards
- Contains full request/response details

### CI/CD Reports
- **Artifacts**: Downloaded from GitHub Actions run
- **GitHub Pages**: https://dhirensinghem.github.io/automation-engineer-test-be/ (auto-deployed on main branch)
- **Retention**: 30 days (configurable)

---

## Best Practices

1. **Run Folders in Order** — Auth → Authorization → Shifts (don't skip)
2. **Use Environment Variables** — Never hardcode credentials in requests
3. **Generate Unique Data** — Use pre-request scripts for unique emails/dates
4. **Capture Response Data** — Chain requests via `pm.environment.set()`
5. **Assert on Multiple Levels** — Status code + errorCode + message
6. **Validate Response Structure** — Not just status code
7. **Use Descriptive Request Names** — Include expected status (e.g., "Create shift - missing fields returns 400")
8. **Document Trade-offs** — Keep this file updated as suite evolves

---

## Extending the Suite

### Add New Request to Existing Folder
1. In Postman UI: Right-click folder → "Add request"
2. Name it descriptively: "GET /shifts/:id - returns 404 for invalid ID"
3. Fill in:
   - Method: GET
   - URL: `{{baseUrl}}/shifts/invalid-id`
   - Headers: `Authorization: Bearer {{adminToken}}`
4. Add test script:
   ```javascript
   pm.test('Status code is 404', () => {
     pm.response.to.have.status(404);
   });
   ```
5. Export collection and commit

### Add New Environment Variable
1. In `postman/environments/dev.postman_environment.json`, add to `values` array:
   ```json
   {
     "key": "myNewVar",
     "value": "someValue",
     "type": "default",
     "enabled": true
   }
   ```
2. Reference in requests: `{{myNewVar}}`

### Create New Folder (e.g., Reports)
1. Right-click collection → "Add folder"
2. Name it (e.g., "Reports")
3. Add requests to it
4. Export collection

---

## Known Limitations & Future Work

### Current Limitations
1. **Single Deployment**: Only tests https://orta-full-stack-dev-test-be-63gp.onrender.com/api
2. **Chained Execution**: Cannot run individual requests standalone
3. **No Smoke Filter**: All 38 requests treated equally (no fast subset)
4. **Seed Data Dependent**: Relies on seeded admin/worker accounts
5. **No Performance Metrics**: Reports don't include response time analysis
6. **No API Mock**: Tests hit live backend (no offline testing)

### Planned Enhancements
1. **Multi-Environment** — staging.postman_environment.json, prod variant
2. **Smoke Folder** — Subset of critical requests (register → login → create shift)
3. **Performance Baselines** — Track response times across runs
4. **Data Factory Pattern** — Generate complex payloads dynamically
5. **Scheduled Monitoring** — Daily runs with Slack notifications on failure
6. **API Versioning** — Support multiple API versions in same collection
7. **Contract Testing** — Validate API response schema against OpenAPI spec
8. **Load Testing** — Extend to k6 or Artillery for stress testing

---

## Debugging & Development

### Enable Verbose Output
```bash
npx newman run collection.json -e environment.json --verbose
```

### Inspect Request Before Sending
Add to pre-request script:
```javascript
console.log('Request URL:', pm.request.url);
console.log('Headers:', pm.request.headers);
console.log('Body:', pm.request.body);
```

### Inspect Response
Add to test script:
```javascript
console.log('Status:', pm.response.code);
console.log('Response:', pm.response.text());
```

### Run with Postman Desktop
1. Import collection: `postman/collections/orta-api-collection.json`
2. Import environment: `postman/environments/dev.postman_environment.json`
3. Select environment dropdown (top-left)
4. Open "Runner" (CMD/CTRL + ALT + R)
5. Select collection and folder
6. Click "Run"
7. Interactive feedback on each request

---

## CI/CD Integration

### GitHub Actions Workflow
File: `.github/workflows/newman-tests.yml`

**Triggers**:
- Push to main/develop
- PR to main/develop
- Daily at 3 AM UTC
- Manual via Actions tab

**Features**:
- Automatic PR comment with results
- Artifact uploads (HTML + JSON reports)
- GitHub Pages deployment
- Execution summary

**Fail on Error**: ✅ Enabled (PR merge blocked if tests fail)

---

## Related Documentation

- **Collection JSON**: `postman/collections/orta-api-collection.json`
- **Environment**: `postman/environments/dev.environment.json`
- **CI/CD Setup**: `.github/workflows/newman-tests.yml`

---
