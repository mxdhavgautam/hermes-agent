# VPS Backports Branch Notes

This file documents the purpose and contents of the `vps-backports-20260510` branch in Madhav's fork of `hermes-agent`.

## Why this branch exists

This branch preserves a known-good Hermes deployment state for the native VPS install at:

- code root: `/usr/local/lib/hermes-agent`
- branch: `vps-backports-20260510`
- fork remote: `madhav-fork`

It was created instead of blindly upgrading to upstream `origin/main`, because upstream was far ahead of the deployed release and a full jump was unnecessarily risky for the live VPS.

## Base and key commits

Release base:
- `498bfc7bc` (`v2026.5.7`)

Important local commits on top of that base:
- `8a083ac9a` `fix(aux): evict stale async codex wrappers after timeout`
- `1b97c2ea4` `fix(gateway): adopt unit's HERMES_HOME for --system CLI ops`
- `85b343c40` `feat(gateway): stream Telegram edits safely`
- `94bdf83ff` `fix(gateway): finalize final stream edit on done`
- `8b2f9f78e` `fix(kanban): gate claim + unblock on parent completion`
- `33b7867c8` `fix(context): handle JSON decode errors in compression`
- `5259e3faa` `fix(session): route OR-combined short CJK tokens to LIKE fallback`
- `9fdc6ac63` `fix(gateway): degrade gracefully when all platform adapters are missing`
- `77482212d` `fix(gateway): pass max_total_size_mb and max_file_size_mb to CheckpointManager`
- `bea4f16c5` `fix(gateway): detect gateway process via /proc in Docker without procps`
- `631d18277` `backport: harden telegram notify, kanban init, and recall init`
- `98c958719` `fix(cron): normalize safe script paths at tool boundary`

## Major fixes included

### 1. Auxiliary client / compaction stability

A real production issue was observed where session summarization could fail with errors like:
- `Cannot send a request, as the client has been closed.`

The branch includes a fix for stale async Codex auxiliary wrappers so poisoned cached wrappers are evicted and rebuilt correctly instead of being reused after the inner sync client is closed.

This was verified with targeted repros plus the full auxiliary-client test file.

### 2. Safe upstream backports without a full upgrade

Selected low-risk upstream fixes were brought in on top of the release base instead of jumping to `origin/main` wholesale.

These include:
- gateway home/profile handling
- safer Telegram streaming/edit behavior
- parent-gated kanban unblocking
- context compression JSON salvage
- better session-search fallback for short OR/CJK cases
- more graceful gateway startup behavior
- proc detection fallback where `procps` is missing

### 3. Manual backports for locally valuable behavior

Some useful upstream behavior did not cherry-pick cleanly, so it was backported manually into this branch instead.

That includes:
- Telegram notification-mode behavior
- recall / session DB initialization hardening
- kanban/init cleanup on the local runtime path

### 4. Cron script path normalization

The scheduler already safely validated absolute and `~` paths as long as they resolved inside `~/.hermes/scripts/`, but the `cronjob` tool used to reject those paths earlier at the API boundary.

This branch fixes that mismatch.

Now safe inputs like:
- `send_theo_vercel_reminder.sh`
- `/home/hermes/.hermes/scripts/send_theo_vercel_reminder.sh`
- `~/.hermes/scripts/send_theo_vercel_reminder.sh`

are normalized and stored canonically as relative script paths under the Hermes scripts directory.

Unsafe paths outside that directory are still blocked.

## Verification that was done

Representative checks completed during rollout:
- `tests/agent/test_auxiliary_client.py`: `143 passed`
- focused backport regression suite: `416 passed`
- `tests/cron/test_cron_script.py`: `40 passed`
- live Hermes smoke returned `OK`
- live `session_search` smoke returned `OK`
- dashboard listener verified on `100.88.21.120:9119`
- live cron tool verification confirmed absolute script paths inside the scripts dir normalize correctly

## Operational notes

This branch is intended as a practical VPS-preservation branch, not as a claim that it should be merged upstream unchanged.

It is useful for:
- disaster recovery
- recreating a known-good VPS state
- comparing future upstream changes against the exact local fixes that kept this deployment stable
- opening selective PRs later if any of these changes are worth upstreaming cleanly

## GitHub location

Fork:
- `https://github.com/mxdhavgautam/hermes-agent`

Branch:
- `vps-backports-20260510`

## Quick diff commands

From the repo root:

```bash
git checkout vps-backports-20260510
git log --oneline 498bfc7bc..HEAD
git diff --stat 498bfc7bc..HEAD
git diff 498bfc7bc..HEAD -- tools/cronjob_tools.py tests/cron/test_cron_script.py
```

## Bottom line

This branch is the tracked, reproducible record of the Hermes fixes and backports that were applied to stabilize the native VPS deployment without doing a reckless full upgrade.
