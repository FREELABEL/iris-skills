---
name: playwright-tests
description: Build, run, debug, and maintain Playwright E2E tests for the Freelabel platform. Pass an action (create, run, debug, fix) and optional target as arguments.
allowed-tools:
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Bash
  - Task
---

# Playwright E2E Tests — Build, Run & Maintain

Create, run, debug, and fix Playwright end-to-end tests for the Freelabel Nuxt 2 frontend.

## Arguments

`$ARGUMENTS` — What to do. Examples:

- `/playwright-tests create signup` — Create a new test for the signup flow
- `/playwright-tests create "page builder drag and drop"` — Create a test from a description
- `/playwright-tests run signup` — Run a specific test file
- `/playwright-tests run all` — Run the full E2E suite
- `/playwright-tests debug signup` — Run headed with debug output
- `/playwright-tests fix signup` — Diagnose and fix failing tests
- `/playwright-tests list` — List all existing test files
- `/playwright-tests coverage` — Show what flows have/lack test coverage

## Project Configuration

### Key Paths

| File | Purpose |
|------|---------|
| `<project-root>/playwright.config.ts` | Global config (timeouts, projects, reporters) |
| `<project-root>/tests/e2e/` | All test spec files |
| `<project-root>/tests/e2e/helpers/` | Shared helpers (auth, page objects, providers) |
| `<project-root>/test-results/screenshots/` | Test screenshots |
| `<project-root>/playwright-report/` | HTML report output |

### Config Summary

```
testDir:         ./tests/e2e
timeout:         600s (10 min per test)
fullyParallel:   false (sequential)
actionTimeout:   15000ms
navigationTimeout: 30000ms
baseURL:         https://web.heyiris.io (override with BASE_URL env)
screenshot:      only-on-failure
projects:        chromium (full), local (safe/no-auth tests)
```

### Environment Variables

```bash
BASE_URL=http://localhost:9300        # Local dev (default)
BASE_URL=https://web.heyiris.io      # Production
HEYIRIS_TOKEN=ca54cd87...            # Auth token for logged-in tests
```

### Run Commands

```bash
# From project root (<project-root>)
npx playwright test tests/e2e/signup.spec.ts              # Run one test
npx playwright test tests/e2e/signup.spec.ts --headed     # With browser visible
npx playwright test tests/e2e/signup.spec.ts --debug      # Debug inspector
npx playwright test tests/e2e/ --reporter=list             # All tests, list output
npx playwright test --project=local --headed               # Safe local tests only
npx playwright show-report playwright-report               # View HTML report
```

## Test File Template

Every new test MUST follow this exact structure:

```typescript
import { test, expect, Page } from '@playwright/test'

const BASE_URL = process.env.BASE_URL || 'http://localhost:9300'

/** Longer timeout for Nuxt 2 SSR pages */
const NAV_OPTS = { timeout: 120000, waitUntil: 'domcontentloaded' as const }

test.use({ ignoreHTTPSErrors: true })

test.describe('Feature Name', () => {
  const consoleLogs: string[] = []

  test.beforeEach(async ({ page }) => {
    consoleLogs.length = 0
    page.on('console', (msg) => {
      const text = msg.text()
      consoleLogs.push(`[${msg.type()}] ${text}`)
      if (text.includes('ERROR') || text.includes('error')) {
        console.log(`  BROWSER ERROR: ${text.substring(0, 300)}`)
      }
    })
  })

  test('descriptive test name', async ({ page }) => {
    console.log('\n-- Step 1: Navigate --')
    await page.goto(`${BASE_URL}/path`, NAV_OPTS)
    await page.waitForTimeout(3000)

    // Assertions
    const element = page.locator('#my-element')
    await expect(element).toBeVisible({ timeout: 15000 })

    await page.screenshot({ path: 'test-results/screenshots/feature-01-step.png' })
  })
})
```

## Critical Patterns

### 1. NAV_OPTS — Always Use for Page Navigation

Nuxt 2 SSR is slow. NEVER use bare `page.goto()`:

```typescript
// BAD — will timeout on SSR pages
await page.goto(`${BASE_URL}/pricing`)

// GOOD — 120s timeout with domcontentloaded
const NAV_OPTS = { timeout: 120000, waitUntil: 'domcontentloaded' as const }
await page.goto(`${BASE_URL}/pricing`, NAV_OPTS)
```

### 2. Authentication — Inject via localStorage

For tests that need a logged-in user:

```typescript
const TOKEN = process.env.HEYIRIS_TOKEN || 'ca54cd87e7046098eee99de3b9c98cfd'

async function injectAuth(page: Page) {
  await page.goto(`${BASE_URL}/login`, { timeout: 120000, waitUntil: 'networkidle' })
  await page.waitForTimeout(3000)

  await page.evaluate((tkn) => {
    const userData = {
      id: 193,
      email: 'admin@example.com',
      name: null,
      user_name: 'admin',
      phone: '(817) 703-7623',
      account_type: '2',
      is_admin: 1,
      is_paid: 0,
      user_token: tkn,
      xp_points: 1007660,
      dashboard_type: 'artist',
      default_profile: null,
      platform_fee_percentage: 20
    }

    localStorage.setItem('user', JSON.stringify(userData))
    localStorage.setItem('user_token', tkn)
    localStorage.setItem('user_id', '193')
    localStorage.setItem('email', 'admin@example.com')
    localStorage.setItem('user_name', 'admin')
    localStorage.setItem('user_account_type', '2')
    localStorage.setItem('user_account_package', '2')
    localStorage.setItem('user_is_paid', '1')
    localStorage.setItem('user_session_key', 'e7ea9e64-ea8b-4c14-a6e3-687bf1888c40')
    localStorage.setItem('user_xp_points', '1007660')
  }, TOKEN)
}
```

Or use the helper class:

```typescript
import { AuthHelper } from './helpers/auth-helper'
await AuthHelper.loginWithToken(page, TOKEN)
```

### 3. Test Isolation — Clear Auth Between Tests

**CRITICAL**: localStorage persists between tests. If any test sets auth data, subsequent tests may redirect unexpectedly. Use `addInitScript` to clear BEFORE page JS runs:

```typescript
test.beforeEach(async ({ page }) => {
  await page.addInitScript(() => {
    const keys = ['user', 'user_token', 'user_id', 'email', 'user_name',
      'user_account_type', 'user_account_package', 'user_is_paid',
      'user_session_key', 'user_xp_points']
    try { keys.forEach(k => localStorage.removeItem(k)) } catch {}
  })
})
```

**Why `addInitScript` instead of `page.evaluate`?** Because `page.evaluate` runs AFTER page JS loads — the Nuxt `beforeCreate` hook may have already read localStorage and triggered a redirect before your cleanup runs.

### 4. API Response Mocking

Mock external APIs to prevent real account creation / payments:

```typescript
// Mock a successful response
await page.route('**/API/Auth/Register', async (route) => {
  const postData = route.request().postData() || ''
  const payload = Object.fromEntries(new URLSearchParams(postData))

  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify({
      data: { user: { id: 99999, email: payload.email, user_token: 'mock-token' } }
    })
  })
})

// Mock an error response
await page.route('**/API/Auth/Register', async (route) => {
  await route.fulfill({
    status: 422,
    contentType: 'application/json',
    body: JSON.stringify({
      error: [{ message: 'An account with this email already exists.' }]
    })
  })
})

// Intercept redirects to prevent navigation
await page.route('**/onboarding**', route => route.fulfill({
  status: 200,
  contentType: 'text/html',
  body: '<html><body><h1>Mocked</h1></body></html>'
}))
```

### 5. Request Payload Capture

Monitor outgoing API calls without blocking:

```typescript
let requestPayload: any = null
page.on('request', (request) => {
  if (request.url().includes('/api/v1/checkout')) {
    try {
      requestPayload = JSON.parse(request.postData() || '{}')
    } catch { /* ignore */ }
  }
})

// Later: verify the payload
expect(requestPayload).not.toBeNull()
expect(requestPayload.email).toBe('test@example.com')
```

### 6. API Error Detection

Capture 4xx/5xx responses during test execution:

```typescript
const apiErrors: Array<{ url: string; status: number; method: string }> = []

page.on('response', (response) => {
  const url = response.url()
  const status = response.status()
  if (url.includes('/api/') && status >= 400) {
    apiErrors.push({ url, status, method: response.request().method() })
    console.log(`  API ERROR: ${response.request().method()} ${url} -> ${status}`)
  }
})

// At end of test
expect(apiErrors.filter(e => e.status >= 500)).toHaveLength(0)
```

## Locator Strategies (Preferred Order)

Use the most stable selector available:

```typescript
// 1. By ID (best stability)
page.locator('#email')

// 2. By data-test attribute (if available)
page.locator('[data-test="submit-btn"]')

// 3. By semantic role
page.locator('button:has-text("Create Account")')

// 4. By class + text combination
page.locator('.create-account-button')

// 5. By form element attributes
page.locator('input[name="phone"]')
page.locator('input[placeholder*="email"]')

// 6. Composite selectors for Vue components
page.locator('.phone input, input[name="phone"]')

// 7. Multiple fallbacks with .or()
page.locator('button:has-text("Pay")').or(page.locator('[data-test="pay-btn"]'))

// 8. Filter chains
page.locator('button').filter({ hasText: 'Submit' })

// 9. Nth element
page.locator('table tbody tr').nth(0)
```

## Screenshot Conventions

**Naming**: `{feature}-{step-number}-{description}.png`

```typescript
await page.screenshot({ path: 'test-results/screenshots/signup-01-page-loaded.png' })
await page.screenshot({ path: 'test-results/screenshots/signup-02-email-error.png' })
await page.screenshot({ path: 'test-results/screenshots/signup-05-form-filled.png', fullPage: true })
```

Always `mkdir -p test-results/screenshots` before running tests.

## Known Gotchas

### Nuxt 2 SSR Load Times
- Pages take 3-8 seconds to hydrate after navigation
- Always add `await page.waitForTimeout(3000)` after `page.goto()`
- Use `waitUntil: 'domcontentloaded'` not `'networkidle'` (networkidle waits for WebSocket/Pusher)

### NuxtLink Same-Component Navigation Bug
Nuxt 2 intercepts same-origin `<a>` clicks and routes through Vue Router. **Vue Router silently fails** for same-component dynamic route param changes (e.g., `/login/register` -> `/login/signin` both map to `_method.vue`).

**Workaround**: Verify the `href` is correct, then use `page.goto()`:
```typescript
const href = await link.getAttribute('href')
expect(href).toContain('/login/signin')
await page.goto(`${BASE_URL}${href}`, NAV_OPTS)
```

### localStorage Domain Scoping
Must navigate to the app domain BEFORE setting localStorage:
```typescript
// BAD — no domain context
await page.evaluate(() => localStorage.setItem('user_token', 'x'))

// GOOD — navigate first
await page.goto(`${BASE_URL}/login`, NAV_OPTS)
await page.evaluate(() => localStorage.setItem('user_token', 'x'))
```

### Form Submit Without .prevent
If testing forms, verify `@submit.prevent` is present. Without it, native form POST fires alongside the Vue handler, causing page reload.

### Vue 2 Prop Defaults vs null
In Vue 2, passing `null` to a prop overrides its default. Use `undefined` to respect defaults:
```javascript
// BAD — overrides Register's title default with null
:title="$route.query.title || null"

// GOOD — undefined lets the default 'Create New Account' apply
:title="$route.query.title || undefined"
```

## Existing Helpers Reference

### AuthHelper (`tests/e2e/helpers/auth-helper.ts`)
```typescript
import { AuthHelper } from './helpers/auth-helper'
await AuthHelper.loginWithToken(page, TOKEN)
```

### LeadsPage (`tests/e2e/helpers/leads-page.ts`)
```typescript
import { LeadsPage } from './helpers/leads-page'
const leadsPage = new LeadsPage(page)
await leadsPage.goto(38)
await leadsPage.applyStrategyToLead('username', 'Strategy Name')
```

### TestHelpers (`tests/e2e/helpers/leads-page.ts`)
```typescript
import { TestHelpers } from './helpers/leads-page'
await TestHelpers.waitForNetworkIdle(page, 5000)
await TestHelpers.takeScreenshot(page, 'my-screenshot')
await TestHelpers.retryAction(() => page.click('button'), 3, 1000)
```

### ProgressManager (`tests/e2e/helpers/progress-manager.ts`)
Crash recovery for long-running scraper tests. Saves progress to disk, resumes if `RESUME=1`.

## Existing Test Coverage

| Area | Test File | Auth Required |
|------|-----------|---------------|
| Signup/Register | `signup.spec.ts` | No |
| Pricing Checkout | `pricing-checkout.spec.ts` | No |
| Profile Checkout | `profile-checkout.spec.ts` | No |
| Booking Wizard | `dent-society-booking.spec.ts` | No |
| Profile Features | `profile-features.spec.ts` | Yes |
| Page Builder | `page-builder.spec.ts` | Yes |
| Bloq Chat | `bloq-chat.spec.ts` | Yes |
| Command Bar | `command-bar.spec.ts` | Yes |
| Lead Outreach | `leads-outreach-workflow.spec.ts` | Yes |
| Batch Leads | `batch-process-5-leads.spec.ts` | Yes |
| Quick Start | `quick-start.spec.ts` | Yes |
| Social Sessions | `save-{platform}-session.spec.ts` | Yes |
| Scrapers | `{platform}-scraper.spec.ts` | Yes |
| Lead Gen | `leadgen-scrape.spec.ts` | Yes |

**Missing coverage** (no tests yet):
- Login/signin flow
- Logout
- Password reset
- Onboarding wizard
- Dashboard home
- Agent creation/configuration
- Workflow builder
- Settings pages
- Billing/subscription management
- Marketplace

## Steps

### 1. Parse Arguments

Determine the action from `$ARGUMENTS`:

- **create [name/description]**: Generate a new test file
- **run [file or "all"]**: Execute tests
- **debug [file]**: Run headed with verbose output
- **fix [file]**: Read failures, diagnose, and fix
- **list**: Show all test files with descriptions
- **coverage**: Audit what's tested vs not

### 2. For "create" — Build a New Test

1. **Identify the target flow**: Read the relevant Vue pages/components to understand:
   - Route URL and page component file
   - Form fields, validation rules, API endpoints
   - Success/error states and redirects
   - Required auth state (logged in or guest)

2. **Read existing tests** for similar patterns — reuse auth, locator, and assertion strategies

3. **Write the test file** at `tests/e2e/{name}.spec.ts` following the template above

4. **Include these test cases at minimum**:
   - Page loads with correct elements visible
   - Form validation (if applicable)
   - Successful happy-path flow (with API mocking if needed)
   - Error state handling
   - Edge cases (pre-filled params, redirects, etc.)

5. **Verify the test discovers correctly**:
   ```bash
   npx playwright test tests/e2e/{name}.spec.ts --list
   ```

6. **Run the test** and fix any failures:
   ```bash
   npx playwright test tests/e2e/{name}.spec.ts --reporter=list
   ```

7. **Check screenshots** in `test-results/screenshots/` for visual verification

### 3. For "run" — Execute Tests

```bash
# Single file
npx playwright test tests/e2e/{file}.spec.ts --reporter=list

# All tests
npx playwright test tests/e2e/ --reporter=list

# Specific test by name
npx playwright test tests/e2e/{file}.spec.ts -g "test name pattern"

# Local-safe tests only
npx playwright test --project=local
```

Report the pass/fail count and any error summaries.

### 4. For "debug" — Diagnose Failures

1. Run the test with `--headed` to see the browser
2. Check failure screenshots in `test-results/`
3. Read the error context files (`error-context.md`) generated by Playwright
4. Check browser console logs captured by the `consoleLogs` array
5. Verify API responses if routes are being intercepted

### 5. For "fix" — Repair Broken Tests

1. Run the test to reproduce the failure
2. Read the failure screenshot
3. Read the relevant Vue component to check if selectors changed
4. Common fixes:
   - **Element not found**: Check if class/id/text changed in the component
   - **Timeout**: Increase `waitForTimeout` or use `waitFor({ state: 'visible' })`
   - **Auth redirect**: Add `addInitScript` to clear stale localStorage
   - **API mock mismatch**: Check if the endpoint URL changed
   - **Flaky timing**: Add explicit waits after actions

## Important Notes

- **Always run from project root**: `<project-root>`
- **Never create real accounts** in tests — always mock the registration API
- **Never call real payment APIs** — mock Stripe checkout responses
- **Screenshots are cheap** — take them at every major step for debugging
- **Console logs are your friend** — use `console.log()` liberally with step markers
- **Test isolation matters** — clear localStorage in `beforeEach` if any test sets auth
- **ESLint Vue files** after editing: `npm run fix-file <path>` in `fl-docker-dev/fl-elon-web-ui/`
