---
name: pi-minimax-setup
description: Reference for pi-mono fork infrastructure - Minimax M2.5-highspeed provider setup, 1Password service token auth chain, pi binary builds, and dual-tool wiring (Claude Code Max + Minimax). Use when configuring providers, debugging auth, building pi, or onboarding to this fork.
---

# Pi Minimax Setup

Reference skill for the pi-mono fork's provider infrastructure and development tooling.

## Fork Context

This is a fork of [badlogic/pi-mono](https://github.com/badlogic/pi-mono). Fork-specific configuration lives in locations upstream does not track to avoid merge conflicts on sync.

| What                     | Location                         | In upstream?         |
| ------------------------ | -------------------------------- | -------------------- |
| Claude Code instructions | `CLAUDE.md`                      | No                   |
| Project skills           | `.claude/skills/`                | No                   |
| Local settings           | `.claude/settings.local.json`    | No (auto-gitignored) |
| API credentials          | `~/.pi/agent/auth.json`          | N/A (user-global)    |
| Upstream config          | `AGENTS.md`, `.pi/`, `packages/` | Yes                  |

Sync with upstream: `git fetch upstream && git rebase upstream/main`

---

## Dual-Tool Architecture

Two independent tools, two independent auth paths, no conflicts.

```
Developer workstation
├── Claude Code CLI (Max $200/mo subscription)
│    ├── Auth: claude.ai OAuth (subscriptionType: max)
│    ├── Models: Opus, Sonnet, Haiku (latest family)
│    └── Purpose: developing pi-mono source code
│
└── Pi Coding Agent (compiled binary)
     ├── Auth: 1Password service token → op read → Minimax API key
     ├── Model: MiniMax-M2.5-highspeed
     └── Purpose: running the pi coding agent with Minimax provider
```

**Key rule**: Do NOT set `ANTHROPIC_API_KEY` in your environment. If set, Claude Code will use API billing instead of the Max subscription.

---

## Auth Chain

```
~/.config/op/service-token          (0600, raw 1Password service token)
  ↓ sourced by
~/.zshrc                            (exports OP_SERVICE_ACCOUNT_TOKEN)
  ↓ used by
~/.pi/agent/auth.json               ("!op read 'op://Claude Automation/...'")
  ↓ resolves to
sk-cp-***                           (Minimax API key, never on disk)
  ↓ sent to
https://api.minimax.io/anthropic    (Anthropic-compatible endpoint)
```

### auth.json Format

Located at `~/.pi/agent/auth.json` (0600 permissions):

```json
{
  "minimax": {
    "type": "api_key",
    "key": "!op read 'op://Claude Automation/MiniMax API - High-Speed Plan/password'"
  }
}
```

The `!` prefix tells pi to execute the command and use stdout as the key value. With `OP_SERVICE_ACCOUNT_TOKEN` set, `op read` runs silently (no biometric prompt).

### 1Password References

| Item                  | Vault             | ID                           |
| --------------------- | ----------------- | ---------------------------- |
| Minimax API key       | Claude Automation | `e54cb3ujopexslaq7loywpuycm` |
| Service account token | Employee          | `xtzirdfnngcgbir7wy4ohfu7i4` |

### Credential Resolution Order (pi)

1. CLI `--api-key` flag
2. `auth.json` entry (API key or OAuth token)
3. Environment variable (`MINIMAX_API_KEY`)
4. Custom provider keys from `models.json`

---

## MiniMax-M2.5-highspeed Specifications

| Property       | Value                                       |
| -------------- | ------------------------------------------- |
| Model ID       | `MiniMax-M2.5-highspeed`                    |
| Provider       | `minimax`                                   |
| API type       | `anthropic-messages` (Anthropic-compatible) |
| Base URL       | `https://api.minimax.io/anthropic`          |
| Context window | 204,800 tokens                              |
| Max output     | 131,072 tokens                              |
| Input cost     | $0.60/M tokens                              |
| Output cost    | $2.40/M tokens                              |
| Cache read     | $0.06/M tokens                              |
| Cache write    | $0.375/M tokens                             |
| Reasoning      | Yes (extended thinking)                     |
| Tool calling   | Yes                                         |
| Throughput     | ~100 TPS sustained, up to 3x burst          |
| Rate limit     | 300 prompts / 5 hours                       |
| Plan           | Plus - High-Speed (subscription)            |

### Cost Comparison with Standard M2.5

Highspeed is exactly **2x** cost for input/output/cache-read. Cache-write is identical. Same model, same context, same capabilities - flat 2x premium for lower latency.

---

## Building Pi

### From Source (development)

```bash
cd /Users/terryli/fork-tools/pi-mono
npm install
npm run build    # ~5s with tsgo
```

Run from source: `./pi-test.sh --provider minimax --model MiniMax-M2.5-highspeed`

### Standalone Binary (production, lowest resource usage)

```bash
cd packages/coding-agent
npm run build:binary    # bun build --compile → dist/pi
```

Produces an arm64 Mach-O binary at `packages/coding-agent/dist/pi`. Bundles the bun runtime - no Node.js needed at execution time.

Run binary: `./packages/coding-agent/dist/pi --provider minimax --model MiniMax-M2.5-highspeed`

### Optional: Symlink to PATH

```bash
ln -sf /Users/terryli/fork-tools/pi-mono/packages/coding-agent/dist/pi /usr/local/bin/pi
```

---

## Validation Commands

### Test Auth Chain (no biometric)

```bash
export OP_SERVICE_ACCOUNT_TOKEN="$(cat ~/.config/op/service-token)"
op read 'op://Claude Automation/MiniMax API - High-Speed Plan/password'
```

### Test Minimax API Directly

```bash
curl -s -X POST "https://api.minimax.io/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: $(op read 'op://Claude Automation/MiniMax API - High-Speed Plan/password')" \
  -d '{"model":"MiniMax-M2.5-highspeed","max_tokens":50,"messages":[{"role":"user","content":"Say: PROBE OK"}]}'
```

### Test Pi with Minimax

```bash
./pi-test.sh --provider minimax --model MiniMax-M2.5-highspeed --no-session --print "Say: PROBE OK"
```

### Verify Claude Code Subscription

```bash
claude auth status
# Should show: subscriptionType: max, authMethod: claude.ai
```

---

## TodoWrite Task Templates

### Template A: Rebuild Pi Binary After Upstream Sync

```
1. [Preflight] Verify upstream remote: git remote -v
2. [Execute] Sync: git fetch upstream && git rebase upstream/main
3. [Execute] Rebuild: npm install && npm run build
4. [Execute] Rebuild binary: cd packages/coding-agent && npm run build:binary
5. [Verify] Test: ./packages/coding-agent/dist/pi --provider minimax --model MiniMax-M2.5-highspeed --no-session --print "PROBE OK"
```

### Template B: Rotate Minimax API Key

```
1. [Preflight] Generate new key on MiniMax platform dashboard
2. [Execute] Update 1Password item "MiniMax API - High-Speed Plan" with new key
3. [Verify] Test auth chain: op read 'op://Claude Automation/MiniMax API - High-Speed Plan/password'
4. [Verify] Test API: curl probe against api.minimax.io/anthropic
```

### Template C: Troubleshoot Auth Failure

```
1. [Preflight] Check service token: cat ~/.config/op/service-token | head -c 4 (should show "ops_")
2. [Preflight] Check OP_SERVICE_ACCOUNT_TOKEN is exported: echo $OP_SERVICE_ACCOUNT_TOKEN | head -c 4
3. [Preflight] Check auth.json exists: cat ~/.pi/agent/auth.json
4. [Execute] Test op read: op read 'op://Claude Automation/MiniMax API - High-Speed Plan/password'
5. [Execute] Test API directly with curl
6. [Verify] If op read fails: re-source ~/.zshrc or check service token file permissions
```

### Template D: Add Another Provider to auth.json

```
1. [Preflight] Check supported providers: packages/coding-agent/docs/providers.md
2. [Preflight] Check env var name: packages/ai/src/env-api-keys.ts (envMap)
3. [Execute] Add entry to ~/.pi/agent/auth.json (literal key or !op read)
4. [Verify] Test: ./pi-test.sh --provider <name> --model <model> --no-session --print "PROBE OK"
```

---

## Post-Change Checklist

After modifying this skill:

1. [ ] Auth chain diagram matches actual file locations
2. [ ] Model specs match `packages/ai/src/models.generated.ts`
3. [ ] 1Password item IDs are current
4. [ ] Validation commands still work
5. [ ] Append changes to [evolution-log.md](./references/evolution-log.md)
