# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **data-only fixture repo** — no source code, no build, no test suite, no lint. It is the synthetic working environment for the "Always-On Ops Agent" (a scheduled Claude Code routine pointed at this repo). All content is fictional; the `bts-synthetic.example` email domain and the company names (Acme, Globex, Sirius) map to nothing real.

The agent is the program; these files are its input. Two tracks run against it:

- **Incident triage** — read `issues/`, correlate against `runbooks/` and `deploys/recent.json`, then set severity/labels/assignee and propose a fix.
- **Compliance drift** ("Card C") — scan `contracts/` against `compliance-policy.md` and open an issue per violation.

## Commands

There is nothing to build or test. The useful commands are data queries and repo setup.

Enable GitHub issues before running the compliance routine — forks inherit issues **disabled**, and the routine's whole output is issue creation:

```bash
gh repo edit JonHaz/always-on-agent --enable-issues
```

Inspect the fixtures:

```bash
jq -s 'map({id, title, severity, labels, opened_at})' issues/*.json
```

```bash
jq '.deploys | sort_by(.deployed_at) | .[] | "\(.deployed_at)  \(.service) \(.version)  \(.summary)"' -r deploys/recent.json
```

Validate JSON after any edit (the fixtures are hand-written and unguarded by CI):

```bash
for f in issues/*.json deploys/recent.json; do jq -e . "$f" >/dev/null || echo "INVALID: $f"; done
```

Sync from the template this was forked from (`origin` = `JonHaz/always-on-agent`, `upstream` = `rosscrooke/always-on-agent`):

```bash
git fetch upstream && git merge upstream/main
```

## Architecture: the cross-file joins

The value of this repo is not any single file — it is the deliberate wiring between `issues/`, `runbooks/`, and `deploys/recent.json`. Every incident is designed to be *unsolvable from the issue text alone* and solvable only by joining the three. Preserve these joins when editing.

**Schemas.** Issues carry `severity: null`, `labels: []`, `assignee: null`, `comments: []` — those nulls are the triage agent's output slots, not missing data. Don't pre-fill them in the fixtures. Deploy entries carry `rollback_available` and `last_known_good`, which exist so a remediation proposal can name a concrete rollback target.

**The five issues each exercise a different path:**

| Issue | Join | What it tests |
|---|---|---|
| `PROD-4521` | `payment-service-degraded.md` §4 names `PaymentService.java:142` verbatim; `payment-service` v4.8.2 ("Add guest checkout support") deployed 14 minutes before it opened | The clean case — runbook + deploy both point at the same root cause, and `rollback_available: true` |
| `PROD-4487` | Same guest-checkout root cause one layer up: `tenant-config-service` v3.2.1 enabled the `guest-checkout` flag for cohort B, Acme included, 13 minutes before onset | Linking two issues to one cause across a day, and applying the runbook's severity guide (one tenant → P1) |
| `PROD-4498` | `auth-502-windows.md` only — the issue predates the `auth-service` deploy, and that deploy has `rollback_available: false` | Runbook-only diagnosis when no deploy correlates; the runbook's explicit "don't scale pods / don't raise LB timeout" anti-advice |
| `PROD-4519` | `cdn-upload-latency.md` gates the signing-service TTL diagnosis on "started within 6 hours of a deploy" — the timeline does **not** satisfy that window | Whether the agent checks the runbook's precondition or pattern-matches on the service name |
| `PROD-4506` | None — it is a feature request, not an incident | Negative control; triage should not manufacture a severity for it |

**Deploy correlation is time-ordered, not name-matched.** `deploys/recent.json` is stored unsorted and spans 2026-05-14 to 2026-05-20; the correlation that matters is the gap between `deployed_at` and the issue's `opened_at`. Two entries are decoys (`frontend` copy change, `auth-service` Redis migration).

## Architecture: the compliance track

`compliance-policy.md` defines seven rule areas, each with an explicit violation test. All three contracts in `contracts/` share the same eight-section skeleton, and sections 2–8 line up with those rule areas — so the scan is a section-by-section comparison, not free-text search.

The three contracts are graded fixtures:

- **`globex-messaging.md`** — clean control. Passes every rule.
- **`acme-data-platform.md`** — a handful of genuine violations (data residency, subprocessor notice given *after* the change rather than before, 96-hour breach notice), plus deliberate near-misses: 60-day audit notice and 90-day termination notice both sit *inside* the policy thresholds. Flagging those is a false positive. Its California governing law is a judgment call the policy does not settle.
- **`sirius-storage.md`** — violates nearly every section. The saturated case.

## Conventions when adding fixtures

- Keep everything fictional and on `bts-synthetic.example`; never introduce a real vendor, person, or system.
- Issue IDs are `PROD-4xxx`; timestamps are ISO-8601 UTC clustered in May 2026. A new issue needs a matching runbook and/or deploy entry, or it teaches the agent nothing.
- Runbooks state a root cause, ordered checks, a named config key and file to change, and — where it matters — an explicit "Don't" section. That anti-advice is load-bearing; agents reach for the wrong fix without it.
