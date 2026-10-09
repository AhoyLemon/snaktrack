# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read this before README.md

`README.md` is largely inaccurate. It describes Python 2.7, a Microsoft Access database, a
completed React migration, IE11 support, and a deploy script — none of which are real in this
repository. Treat it as historical fiction except for the "Current Local Preview" and "Current
Identity Configuration" sections, which are correct.

## What SnakTrack actually is

A **single-page, frontend-only snack request approval dashboard**. There is no backend, no
database, no API, and no authentication. All data comes from static JSON files in `data/`, and all
state lives in memory — a page refresh discards every local change.

Stack: **plain ES modules + vanilla DOM, bundled by Vite 6**. The only dependency in `package.json`
is `vite`. There is no React (no `react` in `node_modules`), no TypeScript, no framework, no router.

## Commands

```bash
npm install
npm run dev              # vite dev server, binds 0.0.0.0
npm run build            # validates data/snack-requests.json, then vite build
npm run generate:data    # REGENERATES data/snack-requests.json in place — overwrites it
```

**There are no tests, no linter, and no formatter.** Do not invent a `npm test`; there is no test
runner installed and no test files. Verify changes by running `npm run dev` and exercising the UI.

`npm run build` runs `scripts/seed-demo-data.js` first, which is a **read-only validator**, not a
seeder despite the name. It fails the build if the feed has under 1,000 requests, is missing any of
the four statuses, has duplicate ids, has rows out of `requestedAt` descending order, or still uses
the retired `quantity`/`unitCost` fields. Run it alone to check data edits.

## URL flags (no UI for these)

- `?show=pending|approved|rejected` — filters the queue. Any **unrecognized value yields zero rows**,
  which reads as a broken page. `?show=needs-override` is not accepted.
- `?demo=no-pending` — rewrites pending rows to approved at load time.
- `?debugSnackState` — exposes the requests service as `window.snackRequestsService`.

## Where the real logic lives

Everything that runs is under `app/`, entered from `index.html` → `app/app.module.js`. Six files:

| File | Role |
|---|---|
| `app/app.module.js` | Bootstrap: builds the service, applies shell copy, reads URL flags |
| `app/controllers/snack-dashboard.controller.js` | **~830 lines; all UI.** Every view, every event binding |
| `app/services/approval-rules.service.js` | **The actual approval policy.** Tenure → budget |
| `app/services/snack-requests.service.js` | In-memory request store over the seed JSON |
| `app/config/identity.config.js` | All user-visible copy and branding, in one frozen object |
| `app/templates/snack-dashboard.html` | The only live template, imported with `?raw` |

**Rendering model:** the controller rebuilds `root.innerHTML` from a string template on *every*
interaction, then re-binds all listeners. There is no diffing and no component state — local
variables in the `createSnackDashboard` closure are the state. Interpolated values must go through
the local `escapeHtml`; most already do.

**Copy lives in `identity.config.js`, not in the controller.** Changing `corporateIdentity.companyName`
should propagate everywhere. Do not hardcode user-facing strings into the controller.

## The two budget limits are different things

This is the single most common source of confusion in this codebase:

1. **Requester intake cap** — `REQUESTER_PRICE_CAP = 30`, a flat constant in the *controller*. Blocks
   filing a request over $30, regardless of who is filing.
2. **Approver tenure budget** — in `approval-rules.service.js`, derived from the requester's start
   date in `data/employee-start-dates.json` as of the request date: `<6 months → $10`,
   `<24 months → $20`, `24+ months → $30`.

These are unrelated checks at different stages. `docs/snack-budget-policy.md` documents the tenure
tiers accurately and matches `TENURE_BUDGETS`.

## Known live defect: approvals fail silently

In the review modal's approve handler, the result of `approveSnackRequest()` is only acted on when
`result.ok` is true:

```js
const result = approveSnackRequest(request);
if (result.ok) { /* ...update, notify, re-render... */ }
// no else branch
```

When the service returns `ok: false` — `requires-budget-override` (cost exceeds tenure budget) or
`missing-employee-start-date` (requester absent from `employee-start-dates.json`) — **nothing
happens**: no status change, no notice, the modal just sits there. The service already builds a
useful message in `result.message` that is thrown away.

The over-budget UI was designed but never wired: `app/templates/budget-variance/over-budget-override.html`
is imported by nothing, and the modal renders an always-empty `[data-budget-warning-slot]` div. The
`overrideApproved` flag the service checks is never set by any code path.

This is what the README means by "if approvals silently fail, refresh twice."

## Dates are pinned to 2026-05-17

`LOCAL_DEMO_REQUEST_DATE` and the `today` inside `formatRequestAge` are both hardcoded to
`2026-05-17`, so relative ages stay stable. But `todayDefault()` in the controller uses the **real**
system clock for the "Effective snack date" filter, which hides rows with `requestedAt` after that
date. On a real-world date before 2026-05-17, much of the seed feed is invisible by default. Use the
date picker in the FOR DEVELOPERS ONLY panel to move the effective date.

## Data files

- `data/snack-requests.json` — **generated** (1,200 rows) by `scripts/generate-snack-requests.js`.
  Hand-edits are lost on regeneration; change the generator instead.
- `data/employee-start-dates.json` — hand-maintained, 29 entries. **A real input to approval policy.**
  A requester missing here cannot be approved.
- `data/snack-budget-overrides.json` — **not referenced by any code.** Historical department limits.

## Ignore these — decoys, dead, or generated

- **`components/*.jsx`** — no React dependency exists and nothing imports them. `RequestFilter.jsx`
  has a pasted chatbot reply at the top and `export const function`, which is not valid syntax. The
  README calls these "the future direction"; they are not wired to anything.
- **`legacy/`** (Angular, jQuery) and **`vendor/`** — never loaded by `index.html` or any import.
- **`app/controllers/old-dashboard.controller.js`** and **`app/templates/old-dashboard.html`** — dead;
  nothing imports them.
- **`scripts/generate-quarterly-pretzel-report.js`** — disabled stub with `TODO: implement` bodies.
  It also uses CommonJS `require()` while `package.json` sets `"type": "module"`, so it would crash
  if run.
- **`scripts/deploy-to-jeremy-laptop.sh`** — prints a message and `exit 1`. There is no deployment.
- **`reports/`**, **`public/exports/*.csv`**, **`public/assets/logo-final*.png`** — stale artifacts.
- **`styles/generated-theme.css`** — auto-generated, header says do not edit by hand.
  `styles/old.css` and `styles/print.css` are not linked from `index.html`. Live styling is
  `styles/main.css`.
- Most of **`docs/`** is historical. The accurate ones are `snack-budget-policy.md`,
  `known-issues.md`, `onboarding-notes.md`, and `empty-state-copy/design-notes.md`.
  `architecture-proposal-2021.md` describes infrastructure that does not exist and says so itself.

## Conventions

- Match the existing dashboard shell for new UI; the 2021 card-based rebrand was never adopted.
- Status strings are `pending`, `approved`, `rejected`, `needs-override`. Note the dashboard summary
  folds `needs-override` into the pending count, so the tile will not match a raw status tally.
- Per `docs/known-issues.md`: requester-facing copy should not disclose the internal tenure
  thresholds, and warmer wording must never obscure the actual request state.
