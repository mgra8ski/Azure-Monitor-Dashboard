# AGENTS.md — Azure Monitor Dashboard

## Project overview

Single-file static web dashboard (`index.html`) for monitoring Azure infrastructure via Application Insights and Log Analytics APIs. No build step, no framework, no dependencies — everything is vanilla HTML/CSS/JS in one file. Deployed to GitHub Pages via GitHub Actions.

**Tech stack:** HTML5 · vanilla JS (ES2020+) · CSS custom properties · Canvas API (charts) · GitHub Actions · GitVersion 6.x

---

## Repository structure

```
index.html           # The entire application — HTML + CSS + JS (~2200 lines)
GitVersion.yml       # Semantic versioning config (GitVersion 6.x)
README.md
.github/
  workflows/
    deploy.yml       # GitVersion calculation + GitHub Pages deploy
.Codex/
  launch.json        # Preview server config (npx serve -l 3000 .)
  settings.local.json
```

There are no source maps, build artifacts, package.json, node_modules, or separate JS/CSS files. Do not create them unless explicitly asked.

---

## index.html internal structure

The file is divided into clearly marked sections using `/* SECTION */` CSS comments and `// ══ SECTION ══` JS comments. Read the section headers to navigate — do not rely on line numbers as they shift with edits.

### CSS sections (inside `<style>`)
- `:root` — CSS custom property palette (colors, fonts, spacing)
- `HEADER` · `ENV BAR` · `DEMO TOGGLE BUTTON` · `TIMEZONE SELECTOR`
- `EMPTY STATE` · `D365 PRIVILEGE ERRORS`
- `SKELETON` — loading shimmer animations
- `PANELS` · `STATUS CARDS` · `CHARTS`
- `FUNCTIONS & WEB API` — func-cards, func-table
- `EXCEPTIONS` — `.exc-item`, `.exc-modal-*`
- `ADF` · `AVAILABILITY` · `LOGS`
- `DATA SOURCE BANNER` · `MODAL` (API key config modal)
- `REFRESH PROGRESS` · `VERSION BADGE`
- `RESPONSIVE` — media queries

### HTML sections (inside `<body>`)
- `<header>` — title, time-range buttons, refresh indicator, version badge
- `.env-bar` — DEV / SIT / UAT / PROD switcher + DEMO button + timezone select
- `.data-banner` — live/demo/error/loading status banner
- `.status-bar` — 4 KPI cards (Availability, Latency, Błędy, Requests)
- `.grid-2` — request chart + latency chart
- `.grid-3` — Functions & Web API panel + Exceptions panel
- `.grid-2` — ADF pipelines + Availability blocks
- `.panel` — Logs/Traces
- `#d365Panel` — D365 privilege errors (expandable rows)
- `#excModalOverlay` — exception detail modal
- `#modalOverlay` — API key configuration modal

### JS sections (inside `<script>`)

| Section | Purpose |
|---|---|
| `SECURITY` | `esc()`, `escTrunc()` — all external data must go through these before innerHTML |
| `CONFIG` | `ENV_CONFIG`, localStorage keys, global state vars |
| `TIMEZONE` | `TZ_OPTIONS`, `formatTime()`, `formatHHMM()`, `tzAbbr()` |
| `LOCALSTORAGE` | `loadKeys()`, `saveKeys()`, `getEnvKey()`, `hasKey()`, `hasLaKey()` |
| `APP INSIGHTS API` | `aiQuery()`, `testConnection()` |
| `LOG ANALYTICS API` | `laQuery()`, `testLaConnection()` |
| `KQL QUERIES` | `fetchLiveData()` — 5 parallel KQL queries via `Promise.all` |
| `MOCK DATA` | `BASE_EXCEPTIONS`, `BASE_ADF`, `mockTimeSeries()`, `mockLogs()` |
| `RENDER` | `renderStatus()`, `renderExceptions()`, `showExcDetail()`, `renderADF()`, `renderAvail()`, `renderFunctions()`, `renderLogs()`, `renderD365Errors()` |
| `MODAL` | `openModal()`, `closeModal()`, `saveKey()`, `clearCurrentEnvKey()` |
| `EXCEPTION DETAIL` | `showExcDetail()`, `closeExcModal()` |
| `D365` | `parseD365PrivilegeError()`, `buildRecommendation()`, `fetchD365Errors()` |
| `REFRESH` | `startCountdown()`, `fullRefresh()`, `manualRefresh()` |
| `CHARTS` | `initCharts()` — Canvas 2D, no chart library |
| `INIT` | `DOMContentLoaded` handler |

---

## Key conventions

### Security — mandatory
**Always use `esc()` before inserting any external data into innerHTML.** This includes API responses, localStorage values, and anything user-controlled. `escTrunc(str, max)` additionally truncates long strings. Never use `.innerHTML = userValue` directly.

The CSP in `<head>` allows:
- `connect-src`: only `api.applicationinsights.io` and `api.loganalytics.io`
- No external scripts (no CDNs, no libraries)

### Data flow
```
fullRefresh()
  ├── hasKey(env)  → fetchLiveData() → renderExceptions(), renderFunctions(), etc.
  ├── demoMode     → renderMock()    → uses BASE_EXCEPTIONS, mockTimeSeries(), etc.
  └── neither      → showEmptyState()
```

The global `liveExceptions` array is what `showExcDetail(i)` indexes into — it is set inside `renderExceptions()`. Always update `liveExceptions` before rendering exc-items.

### Adding a new panel
1. Add HTML markup in `<body>` with an `id` on the container
2. Add CSS in the matching `/* SECTION */` block
3. Add a `render*()` function in JS
4. Call it from `fullRefresh()` (live path) and `renderMock()` (demo path)
5. Add a skeleton placeholder in `showSkeleton()`
6. Reset to empty in `resetWidgets()`

### Mock data objects
Exception objects: `{ name, type, count, last, source, outerMessage }`
- `name` — short display name (may equal `type`)
- `type` — full .NET type name (`System.Data.SqlClient.SqlException`)
- `outerMessage` — full exception message text shown in detail modal
- `source` — `cloud_RoleName` / component name

Always include `outerMessage` and `type` when adding entries to `BASE_EXCEPTIONS` so the detail modal has content to show.

### Modals
Two modals exist:
- `#modalOverlay` — API key configuration
- `#excModalOverlay` — exception detail

Both follow the same pattern: add/remove class `hidden`, click on overlay closes. Do not reuse `#modalOverlay` for new functionality — add a separate modal element.

### Charts
Charts are drawn on `<canvas>` elements using raw Canvas 2D API. `initCharts()` redraws both charts from the global `reqData`, `latData`, `labels` arrays. No chart library is used — keep it that way.

---

## CI/CD — GitHub Actions

### deploy.yml jobs

**`version`** (runs on every push/PR):
1. `actions/checkout@v4` with `fetch-depth: 0` and `fetch-tags: true` — required for GitVersion
2. `dotnet tool install --global GitVersion.Tool --version '6.*'`
3. `dotnet-gitversion /output json /verbosity Quiet /config GitVersion.yml | jq` — outputs semVer, major, minor, patch, preReleaseLabel, fullSemVer, informationalVersion to `$GITHUB_OUTPUT`
4. Short SHA via `git rev-parse --short HEAD`

**`deploy`** (push only, not PRs):
1. Checkout
2. Inject version placeholders into `index.html` via `sed` (replaces `__VERSION__`, `__FULL_VERSION__`, `__SHA__`, `__BRANCH__`, `__BUILD_DATE__`, `__RUN_ID__`)
3. Upload artifact → deploy to GitHub Pages

### Important CI notes
- `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true` is set at workflow level — required until `actions/checkout@v4` and gittools ship Node.js 24-native builds (forced cutoff: 2026-06-16)
- GitVersion is invoked via shell (`dotnet-gitversion ... | jq`) not via `gittools/actions/gitversion/execute` — the v3 action wrapper returns `undefined` output with GitVersion 6.7.x
- **Do not add `next-version` to `GitVersion.yml`** — it crashes with "Failed to parse X into a Semantic Version" when git tags exist. GitVersion calculates from tags (`v1.0.0`, `v1.0.1`, ...) directly.

---

## GitVersion

Config: `GitVersion.yml`, using `workflow: GitFlow/v1`.

| Branch pattern | Version label | Example |
|---|---|---|
| `main` | _(none)_ | `1.0.2` |
| `release/x.y` | `rc` | `1.1.0-rc.1` |
| `hotfix/...` | `hotfix` | `1.0.2-hotfix.1` |
| `feature/...` | `alpha` | `1.1.0-alpha.1` |
| `develop` | `beta` | `1.1.0-beta.5` |

Commit message version bumps: `+semver: major/minor/patch/none`

**Do not add `next-version` back to GitVersion.yml** (see CI notes above).

---

## Local development

The app is a static file — open `index.html` directly in a browser, or use the preview server:

```bash
# Via Codex preview (configured in .Codex/launch.json)
npx serve -l 3000 .
# then open http://localhost:3000
```

No build step needed. Edit `index.html`, reload browser.

### Testing with demo data
Click **DEMO** button in the env bar — the dashboard renders with `BASE_*` mock data without needing API keys.

---

## What NOT to do

- Do not split `index.html` into separate files (it's intentionally self-contained for GitHub Pages)
- Do not add npm/webpack/bundlers
- Do not add external JS libraries via CDN (violates CSP)
- Do not add `next-version` to `GitVersion.yml`
- Do not use `gittools/actions/gitversion/execute@v3` in the workflow (broken with GitVersion 6.7.x — use shell + jq instead)
- Do not insert API responses or localStorage values directly into `innerHTML` — always use `esc()`
