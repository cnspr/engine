# Web App Testing

## Overview

The browser client at https://cnspr.github.io is vanilla HTML/CSS/JS with no build step.
Tests use Playwright in headless mode against the live deployment.

---

## Prerequisites

1. **GitHub PAT** — a token with `repo` scope is required for all API calls.
   Store it in `~/game/t` (one line, no trailing whitespace). Tests read it from there at runtime.

2. **Playwright** — install once:
   ```bash
   npm install -D playwright
   npx playwright install chromium
   ```

---

## Running Tests

```bash
# All web app tests
npx playwright test tests/web/

# Single test file
npx playwright test tests/web/auth.spec.js

# Headed mode (watch the browser)
npx playwright test tests/web/ --headed

# Debug a specific test
npx playwright test tests/web/orders.spec.js --debug
```

---

## Test Structure

```
tests/web/
  auth.spec.js        # Login / PAT validation / AuthError redirect
  world-load.spec.js  # loadWorld(), loadEventLog(), loadStats()
  map.spec.js         # Canvas map renders, region click/hover
  orders.spec.js      # Order composition → PR submission flow
  setup.spec.js       # Onboarding: fork → isForkReady → initWorldBranch → PR
  stats.spec.js       # Stats panel renders snapshots in turn order
  events.spec.js      # Event viewer shows log lines newest-first
```

---

## Injecting the Token

Tests load the PAT at runtime so it never appears in source:

```js
// tests/web/helpers.js
import { readFileSync } from 'fs';
import { homedir } from 'os';
import path from 'path';

export function readToken() {
  return readFileSync(path.join(homedir(), 'game', 't'), 'utf8').trim();
}
```

Use it in a test fixture:

```js
import { test, expect } from '@playwright/test';
import { readToken } from '../helpers.js';

test.beforeEach(async ({ page }) => {
  await page.goto('https://cnspr.github.io');
  // Paste token into the PAT input and submit
  await page.fill('#pat-input', readToken());
  await page.click('#login-btn');
  await expect(page.locator('#main-panel')).toBeVisible();
});
```

---

## Key Scenarios

### Auth

| Scenario | Expected |
|---|---|
| Valid PAT submitted | Main panel visible; rate-limit shown in debug bar |
| Invalid PAT (bad chars) | `AuthError` thrown; login panel re-shown |
| Token revoked (401) | Redirected back to login without JS error in console |

### World Load

| Scenario | Expected |
|---|---|
| All seven world-state files present | `loadWorld()` returns complete object |
| Missing `heroes.json` (404) | Returns `heroes: []` — no throw |
| `loadEventLog()` called | Lines returned newest-first, no blanks |
| `loadStats()` called | Snapshots sorted by turn number ascending |

### Order Submission

| Scenario | Expected |
|---|---|
| Valid orders JSON composed | PR created; URL shown to player |
| Branch already exists (422) | Silently reuses branch; PR still opens |
| Repeat submission same turn | Existing branch updated; no duplicate PR error |

### Onboarding

| Scenario | Expected |
|---|---|
| `forkCanonical()` → poll `isForkReady()` | Fork appears; `initWorldBranch()` creates `join/{userid}` branch |
| `submitJoinPR()` | PR URL returned and displayed |

---

## Debugging

Open the in-app debug panel (bottom bar) to see:

- Per-call API timing
- Current `X-RateLimit-Remaining` / `X-RateLimit-Limit`
- Last error message

To capture console errors in Playwright:

```js
page.on('console', msg => {
  if (msg.type() === 'error') console.error('BROWSER:', msg.text());
});
```

---

## Rate Limit Budget

GitHub authenticated limit: **5 000 requests/hour**.

| Operation | Cost |
|---|---|
| `loadWorld()` | 7 (parallel) |
| `loadStats()` | 1 + N snapshots |
| `submitOrders()` | 4 |
| `forkCanonical()` + onboarding | 3–5 |

Run integration tests with a dedicated test account PAT to avoid exhausting the game-master token.
