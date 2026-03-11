# dependabot-alert-action-to-youtrack

> Automatically turn Dependabot security alerts into YouTrack issues — no manual triage, no alerts lost in the noise.

## The Problem

GitHub Dependabot surfaces high and critical vulnerabilities in your dependencies, but your engineering team lives in YouTrack. The gap between "alert fired" and "issue tracked" is manual, error-prone, and easy to deprioritize. Critical vulnerabilities slip through. Teams forget to check the Security tab. Nothing gets fixed.

## The Solution

This GitHub Action bridges that gap. Running on a cron schedule, it:

1. Fetches all **open** high/critical Dependabot alerts from your repository via the GitHub API.
2. Filters out already-processed alerts using a lightweight JSON state file committed to the repo.
3. Creates a **YouTrack issue** for every new alert — with severity, CVE ID, package name, version range, and fix availability pre-filled.
4. Commits the updated state back to the repo, giving you a persistent and auditable dedup record.

No duplicate cards. No missed alerts. Fully automated.

## Key Features

- **Zero noise** — only open, high/critical alerts are processed; fixed or dismissed ones are ignored.
- **Configurable YouTrack fields** — map any YouTrack field (Type, Subsystem, Priority, etc.) to a value via a simple YAML config file.
- **Ignore list** — team members can exclude specific alerts (e.g. false positives) by adding their numbers to the state file via a PR. The action never touches this list.
- **Auditable state** — the dedup state file lives in your repo, versioned and reviewable like any other file.
- **Composite TypeScript action** — reusable across repos via `uses:`, fully type-safe, no external services required.

## How It Works

```
Cron trigger
    │
    ▼
Fetch open high/critical Dependabot alerts (GitHub API)
    │
    ▼
Load state file (processedAlerts + ignoredAlerts)
    │
    ▼
Filter: skip already-processed and ignored alerts
    │
    ▼
For each new alert → create YouTrack issue
    │
    ▼
Commit updated state file back to repo
```

## YouTrack Issue Format

Each created issue contains:

- **Summary**: `[{severity}] {CVE ID} — {package name}`
- **Description**: severity level, vulnerable package + version range, whether a fix is available, dependency scope (runtime/development)
- **Custom fields**: fully configurable via `.github/youtrack-fields.yml`, including a severity → Priority mapping

## Configuration

```yaml
# .github/youtrack-fields.yml
fields:
  Type: "Bug"
  Subsystem: "Security"

priorityMapping:
  critical: "Critical"
  high: "Major"
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `youtrack-project-id` | Yes | — | YouTrack project ID |
| `youtrack-token` | Yes | — | YouTrack API token (secret) |
| `youtrack-base-url` | Yes | — | YouTrack instance base URL (secret) |
| `github-token` | Yes | `${{ github.token }}` | GitHub token with `contents: write` |
| `config-file` | No | `.github/youtrack-fields.yml` | Path to field config |
| `state-file` | No | `.github/dependabot-youtrack-state.json` | Path to dedup state file |

## Quick Start

```yaml
# .github/workflows/dependabot-youtrack.yml
name: Dependabot Alerts → YouTrack

on:
  schedule:
    - cron: '0 9 * * 1-5'
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      security-events: read
    steps:
      - uses: dinopascale-prima/dependabot-alert-action-to-youtrack@trunk
        with:
          youtrack-project-id: MY_PROJECT
          youtrack-token: ${{ secrets.YOUTRACK_TOKEN }}
          youtrack-base-url: ${{ secrets.YOUTRACK_BASE_URL }}
```

