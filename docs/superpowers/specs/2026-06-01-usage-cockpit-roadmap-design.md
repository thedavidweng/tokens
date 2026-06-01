# Usage Cockpit Roadmap Design

## Goal

Build tokens into a lightweight, trustworthy AI usage cockpit: users should know whether collection is working, what was submitted, where spend is coming from, and what action to take next, without running a heavy always-on scanner.

The core product principle is: CLI and local cache are the source of truth; UI surfaces read from them. Background work should be scheduled, bounded, and explainable.

## Market Takeaways

Current usage tools tend to win on one of these axes:

- Menu bar visibility: always-visible usage, reset countdowns, and quick status checks.
- Early warning: alerts when users approach a quota, budget, or reset boundary.
- Low overhead: smart polling, on-demand refresh, or cached reads instead of heavy continuous scanning.
- Unified tracking: one view across Claude Code, Codex, Cursor, Gemini, Copilot, and similar clients.
- Trust and explainability: visible data sources, pricing assumptions, and confidence levels.
- Actionable diagnosis: not just usage totals, but next steps when services are broken or costs spike.

tokens should combine the useful parts while avoiding the common failure mode: a heavy resident app that burns power to show numbers the user does not fully trust.

## Non-Goals

- Do not build another high-frequency always-on scanner.
- Do not make a menu bar app compute usage independently from the CLI.
- Do not make mobile responsible for desktop usage collection.
- Do not duplicate existing device-name display work.
- Do not optimize for social leaderboard features before the local reliability loop is solid.

## Roadmap

### Phase 1: Release The Low-Power Service

Ship scheduled background submission as the default package behavior.

Requirements:

- Homebrew service runs `tokens --no-spinner submit` on a schedule.
- Linux systemd setup uses a user timer rather than a resident `tokens serve`.
- `tokens status` can distinguish scheduled submit, old keep-alive service, and missing service.
- Existing `tokens serve` remains available for manual or custom supervisors.

Success criteria:

- A fresh Homebrew install does not leave a resident `tokens serve` process idle between submits.
- A user can run `tokens status` and see whether their service needs migration.

### Phase 2: Status Doctor

Turn `tokens status` into the primary local diagnosis surface.

Requirements:

- Show service health as a stable machine-readable enum.
- Show current auth, device, config/cache paths, scheduler state, and helper process state.
- Show precise suggested fixes for old keep-alive services or missing services.
- Keep `--json` output stable enough for future UI integrations.

Later additions:

- Last submit result.
- Next scheduled run when available.
- Recent error summary.
- Log path.
- Data-source confidence summary.

Success criteria:

- A non-technical user can paste `tokens status` output and another person can tell what is broken.
- A menu bar app can read `tokens status --json` without reimplementing service detection.

### Phase 3: Submit History

Record a compact local history for recent background submissions.

Data model:

- `id`: stable UUID or timestamp-derived id.
- `startedAt`, `finishedAt`.
- `status`: `success`, `failed`, or `partial`.
- `clients`: submitted clients.
- `rowsSubmitted`.
- `tokensSubmitted`.
- `costSubmitted`.
- `deviceId`.
- `errorSummary`.
- `logPath`.
- `sourceVersion`: CLI version.

Storage:

- Append a JSONL file under the existing tokens config/cache directory.
- Keep only the most recent bounded window, for example 100 entries or 30 days.
- Never store credentials or raw chat content.

Consumers:

- `tokens status` reads the latest entry.
- `tokens status --json` exposes latest submit and recent error.
- Future menu bar reads the same JSON rather than scanning usage logs.

Success criteria:

- After a scheduled run, `tokens status` can show last submit time, outcome, and error if any.
- Failed submits are diagnosable without reading long logs first.

### Phase 4: Accuracy Layer

Make usage numbers explain where they came from.

Requirements:

- Attach source labels to aggregates: `local-scan`, `submitted-server`, `estimated-pricing`, `provider-official`, or `unknown`.
- Attach confidence levels: `high`, `medium`, `low`.
- Explain pricing assumptions when cost is estimated.
- Surface conflicts when local totals and submitted/server totals diverge.

Success criteria:

- When a user asks why tokens differs from ClaudeBar, Tokscale, or provider billing, the CLI can show source and confidence rather than only a final number.

### Phase 5: Cost Intelligence

Move from reporting totals to identifying useful action.

Features:

- Rank usage by client, model, workspace/repo, and time window.
- Detect unusual spikes by comparing against a recent baseline.
- Highlight expensive model/client combinations.
- Estimate pacing against a daily or weekly budget if the user configures one.

Success criteria:

- The user can answer: which project or tool is burning spend, and what should I check first?

### Phase 6: Lightweight Menu Bar

Build a macOS menu bar app only after the CLI has stable JSON surfaces.

Architecture:

- Menu bar reads `tokens status --json` and local summary cache.
- It may trigger `tokens --no-spinner submit` manually.
- It must not scan raw AI logs itself.
- Refresh cadence should be low by default, with user-triggered refresh for precision.

Core UI:

- Today usage.
- Last submit status.
- Service health indicator.
- Manual submit button.
- Open tokens.ci.

Success criteria:

- The menu bar solves "installed and forgot" without recreating ClaudeBar-style battery or memory issues.

### Phase 7: Web And Team Dashboard

Use submitted data for personal and team insights.

Features:

- Personal trend and device health.
- Team/repo cost ranking.
- Broken client/service detection across devices.
- Optional budget alerts.

Success criteria:

- Teams can identify expensive repos and broken submitters without asking every user to debug locally.

### Phase 8: Mobile Companion

Mobile should be a viewer and notification surface, not the collector.

Features:

- Today/weekly usage.
- Submit failure alerts.
- Budget or quota warning notifications.
- Device health overview.

Success criteria:

- Mobile improves awareness without adding collection complexity or background limitations.

## Recommended Next Build

Build Submit History next.

Reasoning:

- It supports Status Doctor immediately.
- It gives the future menu bar a stable data source.
- It improves debugging without needing Homebrew tap permission.
- It is smaller and safer than starting a UI app now.

Implementation outline:

1. Add a bounded local submit-history writer around `tokens submit`.
2. Record success and failure entries without storing secrets or raw content.
3. Extend `tokens status --json` with the latest submit result.
4. Extend text status with last submit time and recent error.
5. Add focused tests for success, failure, retention, and JSON output.

## Open Decisions

- Retention policy: 100 entries, 30 days, or both.
- Storage path: config directory vs cache directory.
- Whether failed pre-auth submits should be recorded.
- Whether submit history should include estimated cost when pricing cache is stale.

## Self-Review

- Scope is intentionally product and local-runtime focused; it does not include unrelated leaderboard or billing changes.
- The phases are ordered so each phase produces data or reliability needed by later phases.
- The menu bar and mobile ideas depend on CLI JSON surfaces, avoiding duplicated scanners.
- No requirement asks the CLI to store raw chat content or credentials.
