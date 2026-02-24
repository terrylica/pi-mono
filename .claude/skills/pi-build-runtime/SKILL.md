---
name: pi-build-runtime
description: Reference for pi-mono build pipeline and runtime compilation methods. Covers tsgo compilation, bun build --compile for standalone binaries, cross-platform builds, build order, and performance characteristics. Use when building pi, optimizing startup, choosing between source vs binary execution, or troubleshooting build issues.
---

# Pi Build & Runtime

Reference for the pi-mono build pipeline - from TypeScript source to standalone native binary.

## Build Pipeline Overview

```
TypeScript source (.ts)
  ↓ tsgo (Go-based TS compiler, ~5s for entire monorepo)
JavaScript output (.js in dist/)
  ↓ bun build --compile (bundles 1585 modules into single executable)
Native binary (Mach-O arm64, ~143MB)
```

Three stages, two compilers, one binary. Each stage serves a distinct purpose.

---

## Stage 1: tsgo Compilation

**Tool**: `tsgo` (`@typescript/native-preview` - the Go-based TypeScript compiler)

Every package compiles with:

```bash
tsgo -p tsconfig.build.json
```

**Why tsgo over tsc**: tsgo is a native Go binary that compiles TypeScript significantly faster than the standard `tsc`. The entire 7-package monorepo builds in ~5 seconds.

**Build order** (sequential, respects dependency graph):

```
tui → ai → agent → coding-agent → mom → web-ui → pods
```

This is hardcoded in root `package.json`'s `build` script. Each package must complete before its dependents start because they consume the compiled `.d.ts` and `.js` output.

**Per-package build commands**:

| Package        | Build command                                                                                            | Notes                                      |
| -------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| `tui`          | `tsgo -p tsconfig.build.json`                                                                            | Foundation, no deps                        |
| `ai`           | `npm run generate-models && tsgo -p tsconfig.build.json`                                                 | Auto-generates `models.generated.ts` first |
| `agent`        | `tsgo -p tsconfig.build.json`                                                                            | Depends on ai                              |
| `coding-agent` | `tsgo -p tsconfig.build.json && shx chmod +x dist/cli.js && npm run copy-assets`                         | Copies theme JSON, HTML templates          |
| `mom`          | `tsgo -p tsconfig.build.json && chmod +x dist/main.js`                                                   | Slack bot                                  |
| `web-ui`       | `tsgo -p tsconfig.build.json && tailwindcss -i ./src/app.css -o ./dist/app.css --minify`                 | Also compiles CSS                          |
| `pods`         | `tsgo -p tsconfig.build.json && chmod +x dist/cli.js && cp src/models.json dist/ && cp -r scripts dist/` | Copies runtime assets                      |

**Output**: `packages/*/dist/` directories containing `.js`, `.d.ts`, `.d.ts.map`, `.js.map` files.

---

## Stage 2: bun build --compile (Standalone Binary)

**Tool**: `bun build --compile` (Bun's ahead-of-time compiler)

```bash
cd packages/coding-agent
npm run build:binary
```

This runs:

```
tsgo -p tsconfig.build.json       # Stage 1: compile TS → JS
bun build --compile ./dist/cli.js --outfile dist/pi   # Stage 2: bundle + compile
npm run copy-binary-assets        # Copy themes, templates, WASM, docs
```

**What bun build --compile does**:

1. Bundles all 1585 JS modules (dist/ + node_modules) into a single file
2. Embeds the Bun runtime (JavaScriptCore engine)
3. Produces a native Mach-O executable - no Node.js or npm needed to run it

**Binary assets copied alongside** (`copy-binary-assets`):

- `package.json`, `README.md`, `CHANGELOG.md`
- Theme files (`src/modes/interactive/theme/*.json`)
- HTML export templates (`src/core/export-html/`)
- Documentation and examples
- `photon_rs_bg.wasm` (image processing)

---

## Stage 3: Cross-Platform Builds

**Script**: `scripts/build-binaries.sh`

Builds for all five platforms from a single machine:

| Platform    | Target flag        | Output               |
| ----------- | ------------------ | -------------------- |
| macOS ARM   | `bun-darwin-arm64` | `pi` (Mach-O arm64)  |
| macOS Intel | `bun-darwin-x64`   | `pi` (Mach-O x86_64) |
| Linux x64   | `bun-linux-x64`    | `pi` (ELF x86_64)    |
| Linux ARM   | `bun-linux-arm64`  | `pi` (ELF aarch64)   |
| Windows     | `bun-windows-x64`  | `pi.exe` (PE x86_64) |

```bash
# Build for current platform only
./scripts/build-binaries.sh --platform darwin-arm64

# Build all platforms
./scripts/build-binaries.sh

# Skip cross-platform native deps (faster for local-only builds)
./scripts/build-binaries.sh --skip-deps --platform darwin-arm64
```

**koffi handling**: The `koffi` native module (Windows VT input) is externalized from the bundle (`--external koffi`) to avoid embedding all 18 platform `.node` files (~74MB). Windows builds copy the appropriate `.node` file alongside the binary.

**CI**: `.github/workflows/build-binaries.yml` runs on tag push (`v*`), uses `ubuntu-latest` with Bun 1.2.20 and Node 22.

---

## Runtime Comparison

| Method        | Command                                  | Startup          | Memory                                | Requires              |
| ------------- | ---------------------------------------- | ---------------- | ------------------------------------- | --------------------- |
| Source (tsx)  | `./pi-test.sh`                           | ~1-2s            | Higher (Node JIT + node_modules)      | Node.js + npm         |
| Source (node) | `node packages/coding-agent/dist/cli.js` | ~0.5-1s          | Medium (Node + pre-compiled JS)       | Node.js + npm install |
| **Binary**    | `packages/coding-agent/dist/pi`          | **Near-instant** | **Lowest** (Bun runtime, pre-bundled) | **Nothing**           |

**Recommendation**: Use the binary (`dist/pi`) for day-to-day usage. Use source mode (`pi-test.sh`) only when actively developing pi itself.

---

## Quick Reference

```bash
# Full monorepo build from clean state
npm install && npm run build                          # ~8s total

# Build binary (includes Stage 1)
cd packages/coding-agent && npm run build:binary      # ~7s

# Rebuild after code changes (skip npm install)
npm run build                                          # ~5s

# Rebuild binary only (after npm run build already done)
cd packages/coding-agent && bun build --compile ./dist/cli.js --outfile dist/pi  # ~0.2s

# Type check without emitting (fast feedback)
npm run check                                          # biome + tsgo --noEmit

# Run from binary
./packages/coding-agent/dist/pi --provider minimax --model MiniMax-M2.5-highspeed
```

---

## TodoWrite Task Templates

### Template A: Rebuild After Code Changes

```
1. [Execute] npm run build (from repo root)
2. [Execute] cd packages/coding-agent && npm run build:binary
3. [Verify] ./packages/coding-agent/dist/pi --version
```

### Template B: Clean Rebuild from Scratch

```
1. [Execute] npm run clean (removes all dist/)
2. [Execute] npm install
3. [Execute] npm run build
4. [Execute] cd packages/coding-agent && npm run build:binary
5. [Verify] ./packages/coding-agent/dist/pi --version
```

### Template C: Troubleshoot Build Failure

```
1. [Preflight] Check Node version: node --version (must be >=20)
2. [Preflight] Check bun version: bun --version (needed for binary step)
3. [Execute] npm run clean && npm install
4. [Execute] npm run build 2>&1 | head -50 (check first error)
5. [Verify] If tsgo fails: check tsconfig.build.json paths
6. [Verify] If bun build fails: check dist/cli.js exists from Stage 1
```

### Template D: Build for Specific Platform

```
1. [Execute] npm install && npm run build
2. [Execute] ./scripts/build-binaries.sh --platform darwin-arm64
3. [Verify] file packages/coding-agent/binaries/darwin-arm64/pi
4. [Verify] ls -lh packages/coding-agent/binaries/darwin-arm64/pi
```

---

## Post-Change Checklist

After modifying this skill:

1. [ ] Build commands match current package.json scripts
2. [ ] Platform targets match scripts/build-binaries.sh
3. [ ] Runtime comparison reflects actual measured performance
4. [ ] Append changes to [evolution-log.md](./references/evolution-log.md)
