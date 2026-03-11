# Plan: Dependabot Alerts to YouTrack GitHub Action

Build a **composite GitHub Action** (TypeScript/Node.js) that runs on a cron schedule, fetches high/critical **open** Dependabot security alerts from the current repo via the GitHub API, and creates YouTrack issues for each new alert. Dedup is handled by a JSON state file committed back to the repo. YouTrack card fields are configurable via a YAML config file.

---

## Phase 1: Project Scaffolding (steps 1–4)

1. **Initialize Node.js project** — `package.json` with deps: `@actions/core`, `@actions/github`, `js-yaml`. Dev deps: `typescript`, `@vercel/ncc`, `@types/node`.
2. **TypeScript config** — `tsconfig.json` targeting ES2023/Node22.
3. **Action metadata** — `action.yml` with inputs:
   - `youtrack-project-id` (required)
   - `youtrack-token` (required, from secrets)
   - `youtrack-base-url` (required, from secrets)
   - `config-file` (optional, default: `.github/youtrack-fields.yml`)
   - `state-file` (optional, default: `.github/dependabot-youtrack-state.json`)
   - `github-token` (required, defaults to `${{ github.token }}`)
   - Runs: `node22`, `dist/index.js`
4. **Build** — `ncc build src/index.ts` → bundle into `dist/index.js` (committed, as required by JS actions).

## Phase 2: Core Logic — TypeScript modules (steps 5–11)

5. **`src/index.ts`** — Entry point: parse inputs, orchestrate fetch → filter → ignore-list → dedup → create → commit-state flow.
6. **`src/github-alerts.ts`** — Fetch Dependabot alerts via `GET /repos/{owner}/{repo}/dependabot/alerts`, filter `state: open` + `severity: high | critical`. Returns typed alert array.
7. **`src/youtrack-client.ts`** — YouTrack REST client: `createIssue(project, summary, description, customFields)` via `POST /api/issues` with `Bearer` auth.
8. **`src/config.ts`** — Load + validate the YAML config file. The config maps YouTrack field names to values, plus a `priorityMapping` that maps Dependabot severity levels to YouTrack Priority enum values:
   ```yaml
   # .github/youtrack-fields.yml
   fields:
     Type: "Bug"
     Subsystem: "Security"

   priorityMapping:
     critical: "Critical"
     high: "Major"
   ```
   At issue creation time, the `priorityMapping` value for the alert's severity is merged into the fields as `Priority`.
9. **`src/state.ts`** — Read/write JSON state file. Schema:
   ```json
   {
     "processedAlerts": [101, 102],
     "ignoredAlerts": [
       { "number": 99, "reason": "False positive — not reachable in our codebase" }
     ]
   }
   ```
   - Diff fetched alerts against `processedAlerts` to find new ones.
   - Filter out any alert whose `number` appears in `ignoredAlerts`.
   - The `reason` field is optional but encouraged for auditability.
   - Team members add entries to `ignoredAlerts` manually via PR; the action never writes to this array.
10. **`src/card-builder.ts`** — Build YouTrack issue from alert data:
    - **Summary**: `[{severity}] {CVE ID} — {package name}`
    - **Description**: severity level, vulnerable package + version range, fix available (yes/no), dependency scope/tags (e.g. `development`, `runtime`) from alert's `dependency.scope` field
11. **`src/commit-state.ts`** — After processing, commit updated state file via GitHub Contents API (`PUT /repos/{owner}/{repo}/contents/{path}`). Requires `contents: write` on the token. *(depends on 6, 9)*

## Phase 3: Example Workflow & Config (steps 12–13, parallel with Phase 4)

12. **`.github/workflows/dependabot-youtrack.yml`** — Example caller workflow with configurable cron, `workflow_dispatch` trigger, and all inputs wired to secrets.
13. **`.github/youtrack-fields.yml`** — Example YAML config with common YouTrack fields.

## Phase 4: Build & Quality (steps 14–16, parallel with Phase 3)

14. **npm scripts** — `build` (`ncc build`), `typecheck` (`tsc --noEmit`). *(parallel with 15)*
15. **ESLint** — TypeScript plugin, basic config. *(parallel with 14)*
16. **Unit tests** — Vitest tests for `card-builder`, `config`, `state` modules with mocked API responses. *(depends on Phase 2)*

---

## Relevant Files

| File | Purpose |
|---|---|
| `action.yml` | Action metadata (inputs, entry point) |
| `src/index.ts` | Orchestrator entry point |
| `src/github-alerts.ts` | Dependabot alert fetching & filtering |
| `src/youtrack-client.ts` | YouTrack API client |
| `src/config.ts` | YAML config loader/validator |
| `src/state.ts` | Dedup state file read/write + ignore list filtering |
| `src/card-builder.ts` | Build card summary/description |
| `src/commit-state.ts` | Commit state file back to repo via GitHub API |
| `.github/workflows/dependabot-youtrack.yml` | Example caller workflow |
| `.github/youtrack-fields.yml` | Example YouTrack field config |

## Verification

1. `npm run typecheck` — no TypeScript errors
2. `npm run build` — `dist/index.js` generated
3. `npm test` — unit tests pass for config parsing, state dedup, ignore-list filtering, card building
4. Manual: trigger via `workflow_dispatch` → verify YouTrack card created with correct fields
5. Run again → verify no duplicate card (state file updated + committed)
6. Confirm state file commit appears in repo history

## Decisions

- **TypeScript composite action** — reusable via `uses:`, type-safe
- **State file committed to repo** — persistent, auditable dedup; needs `contents: write` permission
- **YouTrack base URL as secret** (per user preference)
- **Open alerts only** — fixed/dismissed excluded
- **Card content**: CVE + package name, severity, version range, fix available. Advisory link and full description excluded.
- **Current repo only** — no cross-repo scanning
- **Ignore list in state file** — team can manually add alert numbers to `ignoredAlerts` in the state file (via PR); the action skips those alerts. The action only writes to `processedAlerts`, never to `ignoredAlerts`.
