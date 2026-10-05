# CLAUDE.md

Guidance for Claude Code in this repository.

## What this is

Financial Freedom: a personal-finance PWA served from GitHub Pages (branch
`main`). The whole app is **one file, `index.html` (~11,000 lines, ~580 KB)**:
CSS, markup and one classic `<script>`. No build step, no package manager, no
tests, no service worker. Other files: `manifest.json`, `apple-touch-icon.png`.

Its backend brain, JARVIS, lives in `LawrenceL212/jarvis` and reads this app's
data read-only. "Ask My Finances" and the voice call JARVIS at `FF_WORKER_URL`.

## Saving tokens in index.html

**Never read the whole file.** Grep for the function or banner you need, then
read a window around it. Line numbers drift; search by name.

- Sections are marked with `// ═══` banners in CAPS and `// ── name ──` subheads:
  `grep -n "^// ═" -A1 index.html` lists the big ones.
- Screens are top-level functions named `*Screen()` (`overviewScreen`,
  `accountsScreen`, `transactionsScreen`, `hoursScreen`, `planForecastScreen`,
  `calendarScreen`, `goalsScreen`, `debtScreen`, `settingsScreen`, …):
  `grep -nE "^function [a-zA-Z]+Screen" index.html`.
- `mainScreen()` builds the 5-tab nav (Home, Money, Plan, Work, More) plus the
  Payday button; Money and Plan have sub-tabs.
- `jarvis*Panel()` functions are the HUD at the top of each tab.

Rough order of the file: CSS design tokens and components → banking/worker
helpers → Firebase init, `el()` helper, JARVIS voice → per-person settings
(`SETTINGS_DEFAULT`, `PRESET_LAWRENCE`) → `render()`/`save()` → screens and
payday flow → overview → Ask → statement/transaction import → recurring
detection, analytics, **FINANCIAL MODEL** → hours → plan intelligence and
forecast → JARVIS panels → monthly budget, calendar, accounts, goals, debt →
`mainScreen()` → JARVIS bubble → auth bootstrap.

## How it works

- UI is built with `el(tag, props, ...kids)` and swapped in by `render(node)`;
  screens rebuild wholesale, no framework.
- Data: Firebase **Realtime Database** (compat SDK 10.13 from gstatic), all
  state under `users/{uid}` as one `appData` object, transactions at
  `users/{uid}/transactions/{id}`. Auth is Firebase email/password.
- Bank sync: TrueLayer via the Cloudflare worker at `FF_BANKING_URL`.

## Rules

- **`save()` uses `update()`, never `set()`**, and strips the keys in
  `SYNCED_ELSEWHERE` (transactions, statements, bank tokens/connections, …),
  which are written directly elsewhere. Writing the whole account back wipes
  synced data and breaks bank tokens. Keep both guards.
- **Session guards:** async work checks `sessionId` / `expectedUid` so one
  user's data can never land in another's account. Preserve them in new async
  writes; reset per-user caches (`resetUserDerived`) on auth change.
- **The FINANCIAL MODEL section is the single source of truth** for money
  maths (ΔNetWorth = grossIncome − trueSpending; transfers and card payments
  are not spending). Derive figures from it; don't recompute ad hoc on a screen.
- Don't rename `appData` keys or database paths: JARVIS
  (`pipeline/financial_freedom.py` in the jarvis repo) reads them, and live
  user data depends on them.
- No secrets in this file. The Anthropic key lives only in the worker.
- Styling: use the CSS custom properties (surfaces, semantic colours, spacing,
  radii) defined at the top; colour carries meaning, not decoration.
- Mobile-first: must hold up on a ~390 px portrait phone.

## Running

Serve the folder over HTTP (`python -m http.server 8000`) and open
`http://localhost:8000`. Verification is manual in a browser; it signs in to
the live Firebase project, so test with care.
