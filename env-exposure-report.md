# Security Report: Container Secret Exposure via `data/env`

**Date:** 2026-03-21
**Severity:** High — all `.env` secrets (API keys, bot tokens) readable by container agents
**Status:** Mitigated locally; upstream skill files still reference the vulnerable pattern

---

## Summary

The main-group container agent had read access to a full copy of `.env` secrets at `/workspace/project/data/env/env`. This bypassed the existing `.env` shadow mount, exposing credentials that should never reach the container (Anthropic API keys, Telegram bot tokens, and any other values in `.env`).

## Root Cause

Two independent issues combined to create the exposure:

### 1. Legacy `data/env/env` created by setup skills

Multiple skill files (add-telegram, add-slack, add-whatsapp, add-discord, add-voice-transcription, add-telegram-swarm) instruct the user (or Claude Code) to run:

```bash
mkdir -p data/env && cp .env data/env/env
```

This was the original mechanism for passing credentials into containers — the directory was mounted at `/workspace/env-dir/` and sourced by the entrypoint. That approach was later replaced by a stdin-based JSON secrets mechanism (`input.secrets` passed via stdin, merged into `sdkEnv` in the agent runner). However, the skill files were never updated to remove the copy step, so running any channel setup skill still creates the file.

### 2. Shadow mount only covers `.env`, not `data/env`

The main-group container gets the entire project root mounted read-only at `/workspace/project`. The code correctly shadows `/workspace/project/.env` by bind-mounting `/dev/null` over it. But `data/env/` sits inside the project root at `data/env/`, so it was visible at `/workspace/project/data/env/env` with no shadow.

### Secondary issue: stale per-group `agent-runner-src`

The new stdin secrets mechanism (`readSecrets()` in `container-runner.ts` → `containerInput.secrets` → `sdkEnv` merge in the agent runner) was also not functioning. The agent runner source is copied once per group to `data/sessions/{group}/agent-runner-src/` and never updated afterward. Existing groups retained old code with the comment `// No real secrets exist in the container environment` and no secrets-merging loop, silently dropping any secrets passed via stdin.

## What Was Exposed

Everything in `.env` was readable by the main-group container agent at `/workspace/project/data/env/env`. In this instance:

- Anthropic API key / OAuth token
- Telegram bot token
- Any other keys present in `.env` (e.g., GITLAB_TOKEN, OPENAI_API_KEY)

## Remediation Steps Taken (Local)

1. **Deleted `data/env/`** — removed the exposed secrets copy
2. **Deleted stale `agent-runner-src/`** — forces fresh copy with working secrets-merging code on next container start
3. **Added shadow mount** for `data/env` in `buildVolumeMounts()` (`container-runner.ts`) — if the directory is ever recreated, the container sees an empty directory instead
4. **Rebuilt and restarted** the service

## Recommended Upstream Fixes

1. **Update all skill files** to remove `mkdir -p data/env && cp .env data/env/env` instructions. Affected skills:
   - `add-telegram`, `add-slack`, `add-discord`, `add-whatsapp`
   - `add-voice-transcription`, `add-telegram-swarm`
   - `debug` (architecture diagram, manual testing commands, diagnostic script)

2. **Merge the shadow mount** for `data/env` into `container-runner.ts` (defensive, in case old skill instructions are followed)

3. **Fix the stale agent-runner-src problem** — the one-time copy logic (`if (!fs.existsSync(dir))`) means code updates to `container/agent-runner/src/` never propagate to existing groups. Consider either:
   - Always overwriting the copy (simple, but breaks per-group customizations)
   - Comparing timestamps or a version marker before copying
   - Documenting that `data/sessions/*/agent-runner-src/` must be manually deleted after upstream updates

4. **Update `docs/SPEC.md` and `docs/docker-sandboxes.md`** — both still describe the old `data/env/env` mount approach as the current architecture

## Key Rotation

If the exposed keys were used in a context where the container agent could have exfiltrated them (e.g., via outbound network access), rotate:

- Anthropic API key (revoke at console.anthropic.com, generate new)
- Telegram bot token (`/revoke` via @BotFather)
- Any other keys that were present in `.env`

After updating `.env`, restart the service. No other config files embed the keys.
