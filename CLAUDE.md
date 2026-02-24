# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install                  # Install all dependencies
npm run build                # Build all packages (sequential: tui → ai → agent → coding-agent → mom → web-ui → pods)
npm run check                # Biome lint/format + tsgo type check (requires build first)
./test.sh                    # Run all tests with API keys unset
./pi-test.sh                 # Run coding-agent from source (must run from repo root)
```

**Run a single test** (from the package directory, not repo root):

```bash
cd packages/ai
npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

The `tui` package uses Node's native test runner instead of vitest:

```bash
cd packages/tui && node --test --import tsx test/*.test.ts
```

**Never run**: `npm run dev`, `npm run build`, `npm test` (use the specific commands above).

**Build standalone binary** (fastest runtime, lowest resource usage):

```bash
cd packages/coding-agent && npm run build:binary   # produces dist/pi (arm64 Mach-O)
```

## Architecture

This is an npm workspaces monorepo. All packages use **lockstep versioning** (same version number, released together).

### Package Dependency Graph

```
Layer 0 (no internal deps):  pi-tui, pi-ai
Layer 1:                      pi-agent-core  → pi-ai
Layer 2:                      pi-coding-agent → pi-ai, pi-agent-core, pi-tui
                              pi-mom          → pi-ai, pi-agent-core, pi-coding-agent
                              pi-web-ui       → pi-ai, pi-tui
                              pi-pods         → pi-agent-core
```

| Package                 | npm name                        | Purpose                                                   |
| ----------------------- | ------------------------------- | --------------------------------------------------------- |
| `packages/ai`           | `@mariozechner/pi-ai`           | Unified multi-provider LLM streaming API (15+ providers)  |
| `packages/tui`          | `@mariozechner/pi-tui`          | Terminal UI library with differential rendering           |
| `packages/agent`        | `@mariozechner/pi-agent-core`   | Agent runtime with tool calling loop and state management |
| `packages/coding-agent` | `@mariozechner/pi-coding-agent` | Interactive coding agent CLI (the main product)           |
| `packages/mom`          | `@mariozechner/pi-mom`          | Slack bot delegating to the coding agent                  |
| `packages/web-ui`       | `@mariozechner/pi-web-ui`       | Web components for AI chat interfaces                     |
| `packages/pods`         | `@mariozechner/pi-pods`         | CLI for managing vLLM deployments on GPU pods             |

### Key Architecture Concepts

**LLM Provider System** (`packages/ai`): Each provider in `src/providers/` implements a stream function returning `AssistantMessageEventStream`. Providers are registered in `ApiRegistry`. Model metadata is auto-generated in `models.generated.ts`. API keys are auto-discovered from environment variables via `env-api-keys.ts`.

**Agent Runtime** (`packages/agent`): The `Agent` class in `agent.ts` drives a tool-calling loop (`agent-loop.ts`). It takes a model, tools (defined with TypeBox schemas), and state, then streams responses. No transport abstraction at this layer.

**Coding Agent** (`packages/coding-agent`): Built on `AgentSession` (the core abstraction shared across all run modes). Three modes: interactive (full TUI), print (streaming stdout), and RPC (WebSocket/SSE server). Six core tools: `read`, `write`, `edit`, `bash`, `grep`, `find`. Context window management via automatic compaction with branch summaries.

**Extension System** (`packages/coding-agent/src/core/extensions/`): Event-driven hooks for `BeforeAgentStart`, `ToolCall`, `ToolResult`, `TurnStart/End`, `SessionStart/Shutdown`, and `CustomToolCall`. Extensions loaded from `.pi/extensions/` or npm packages.

## Build System

All packages compile with `tsgo` (TypeScript Go compiler) via `tsgo -p tsconfig.build.json`. The root `tsconfig.json` defines path aliases so cross-package imports resolve to source during type checking. Biome handles linting and formatting (tabs, indent width 3, line width 120).

## Code Rules (from AGENTS.md)

- No `any` types unless absolutely necessary
- **Never use inline imports** (`await import("./foo.js")`, `import("pkg").Type`). Always use top-level imports.
- Never remove/downgrade code to fix type errors; upgrade the dependency instead
- All keybindings must be configurable via `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS`
- Never commit unless asked. Never use `git add -A` or `git add .`
- Do not edit `CHANGELOG.md` released sections. New entries go under `## [Unreleased]`

## Adding a New LLM Provider

Requires changes across: `packages/ai/src/types.ts` (types + `Api` union) → `packages/ai/src/providers/` (implementation) → `packages/ai/src/stream.ts` (registration) → `packages/ai/scripts/generate-models.ts` → tests in `packages/ai/test/` (11+ test files) → `packages/coding-agent/src/core/model-resolver.ts` + `src/cli/args.ts`. See AGENTS.md for the full checklist.

## Fork Configuration

This is a fork of [badlogic/pi-mono](https://github.com/badlogic/pi-mono). Fork-specific configuration lives in locations that upstream does not track, to avoid merge conflicts on sync:

| What                 | Location                         | Tracked by upstream? |
| -------------------- | -------------------------------- | -------------------- |
| Agent instructions   | `CLAUDE.md` (this file)          | No                   |
| Project skills       | `.claude/skills/`                | No                   |
| Fork-local settings  | `.claude/settings.local.json`    | No (auto-gitignored) |
| API credentials      | `~/.pi/agent/auth.json`          | N/A (user-global)    |
| Upstream code/config | `AGENTS.md`, `.pi/`, `packages/` | Yes                  |

**Sync with upstream**: `git fetch upstream && git rebase upstream/main` — no conflicts expected since fork-specific files don't exist upstream.

## Project Skills

| Skill              | Purpose                                                                                                                     |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `pi-minimax-setup` | Minimax M2.5-highspeed provider setup, 1Password auth chain, pi binary builds, dual-tool wiring (Claude Code Max + Minimax) |
| `pi-build-runtime` | Build pipeline (tsgo + bun build --compile), standalone binary compilation, cross-platform builds, runtime comparison       |
