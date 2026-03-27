# NanoClaw

Personal Claude assistant. See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions.

## Quick Context

Single Node.js process with skill-based channel system. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) are skills that self-register at startup. Messages route to Claude Agent SDK running in containers (Linux VMs). Each group has isolated filesystem and memory.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/ipc.ts` | IPC watcher and task processing |
| `src/router.ts` | Message formatting and outbound routing |
| `src/config.ts` | Trigger pattern, paths, intervals |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks |
| `src/db.ts` | SQLite operations |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/skills/` | Skills loaded inside agent containers (browser, status, formatting) |

## Skills

Four types of skills exist in NanoClaw. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full taxonomy and guidelines.

- **Feature skills** — merge a `skill/*` branch to add capabilities (e.g. `/add-telegram`, `/add-slack`)
- **Utility skills** — ship code files alongside SKILL.md (e.g. `/claw`)
- **Operational skills** — instruction-only workflows, always on `main` (e.g. `/setup`, `/debug`)
- **Container skills** — loaded inside agent containers at runtime (`container/skills/`)

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/uninstall` | Remove NanoClaw from the host (service, config, optionally data) |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update-nanoclaw` | Bring upstream NanoClaw updates into a customized install |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |

## Security

Never read, grep, cat, or otherwise access `.env` or any file likely to contain secrets (e.g. `*.pem`, `*credentials*`, `*secret*`). If you need to know whether a key exists, ask the user.

## Secrets & Credentials

Secrets in this NanoClaw instance are handled non-standardly — do NOT assume env vars.

### Storage
All secrets live in the host `.env` file (project root). **The agent must never read,
write, or touch the `.env` file in any way.** If a new secret needs to be added, the
agent must ask the user to add it manually and wait for the user to confirm it is done
before proceeding.

### How they reach containers
1. `readSecrets()` in `src/container-runner.ts` calls `readEnvFile()` with an explicit allowlist of keys
2. The result is added to the container input and passed **via stdin as JSON** — never via `-e` flags or `process.env`
3. The `.env` file is shadowed to `/dev/null` inside containers so agents cannot read it directly

### Adding a new secret (agent workflow)
1. **Tell the user** which key needs to be added to `.env` and what value to use
2. **Wait for the user to confirm** they have added it — trust that it is done correctly
3. Add the key name to the `readEnvFile([...])` call inside `readSecrets()` in `src/container-runner.ts`
4. If needed for an MCP server, reference it in `.mcp.json` under `env: { "KEY": "${KEY}" }`

### Credential proxy
Anthropic API credentials are handled separately by `src/credential-proxy.ts`, which intercepts
container API calls and injects real credentials. Containers only ever see placeholder tokens.

### Relevant files
- `src/env.ts` — `readEnvFile()` implementation
- `src/container-runner.ts` — `readSecrets()` and stdin injection
- `src/credential-proxy.ts` — Anthropic credential proxy
- `.mcp.json` — MCP server registrations (currently empty)
- `src/mount-security.ts` — blocked mount patterns

## Contributing

Before creating a PR, adding a skill, or preparing any contribution, you MUST read [CONTRIBUTING.md](CONTRIBUTING.md). It covers accepted change types, the four skill types and their guidelines, SKILL.md format rules, PR requirements, and the pre-submission checklist (searching for existing PRs/issues, testing, description format).

## Development

Run commands directly—don't tell the user to run them.

```bash
npm run dev          # Run with hot reload
npm run build        # Compile TypeScript
./container/build.sh # Rebuild agent container
```

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

## Troubleshooting

**WhatsApp not connecting after upgrade:** WhatsApp is now a separate skill, not bundled in core. Run `/add-whatsapp` (or `npx tsx scripts/apply-skill.ts .claude/skills/add-whatsapp && npm run build`) to install it. Existing auth credentials and groups are preserved.

## Container Build Cache

The container buildkit caches the build context aggressively. `--no-cache` alone does NOT invalidate COPY steps — the builder's volume retains stale files. To force a truly clean rebuild, prune the builder then re-run `./container/build.sh`.
