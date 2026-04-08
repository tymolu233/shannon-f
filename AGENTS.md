# AGENTS.md

This file is the working guide for agentic coding assistants in this repository.
It complements the checked-in `CLAUDE.md` and is intentionally focused on how to make safe, repo-aligned changes.

## Project Overview

- Shannon is an AI-powered white-box pentesting framework for web apps and APIs.
- This repo is a pnpm monorepo with two apps:
  - `apps/cli` — published CLI package `@keygraph/shannon`
  - `apps/worker` — private Temporal worker and pipeline logic
- Root tooling is managed with pnpm workspaces, Turborepo, TypeScript, and Biome.

## Instruction Sources

- Primary repository guidance: `CLAUDE.md`
- This `AGENTS.md` is a condensed agent-facing version of that guidance.
- No `.cursorrules` file was found.
- No `.cursor/rules/` directory was found.
- No `.github/copilot-instructions.md` file was found.

## Important Working Assumptions

- Follow existing architecture and naming patterns instead of inventing new ones.
- Prefer minimal, surgical changes over broad refactors.
- This is a defensive security tool. Do not normalize unsafe use in code or examples.
- Do not weaken validation, retries, or safety checks just to make a change “work.”

## Monorepo Layout

### Root

- `package.json` — root scripts for build, typecheck, Biome, clean
- `pnpm-workspace.yaml` — workspace definition (`apps/*`)
- `turbo.json` — Turborepo task graph
- `biome.json` — formatting and linting rules
- `tsconfig.base.json` — shared strict TypeScript settings
- `docker-compose.yml` — Temporal + router infra
- `CLAUDE.md` — detailed repo instructions and architecture notes

### CLI App

- `apps/cli/src/index.ts` — CLI dispatch entry point
- `apps/cli/src/commands/` — command handlers
- `apps/cli/src/docker.ts` — Docker orchestration
- `apps/cli/src/env.ts` / `mode.ts` — env/config resolution and mode detection

### Worker App

- `apps/worker/src/temporal/worker.ts` — worker/client entry
- `apps/worker/src/temporal/workflows.ts` — main workflow orchestration
- `apps/worker/src/temporal/activities.ts` — thin Temporal activity wrappers
- `apps/worker/src/services/` — core business logic boundary
- `apps/worker/src/session-manager.ts` / `config-parser.ts` — agent registry and YAML config validation
- `apps/worker/prompts/` — prompt templates
- `apps/worker/configs/` — YAML configs and schema-backed settings

## Build, Lint, and Typecheck Commands

Run from the repo root unless you intentionally need a package-local command.

### Root Commands

```bash
pnpm run build      # turbo run build
pnpm run check      # turbo run check
pnpm biome          # biome check .
pnpm biome:fix      # biome check --write .
pnpm run clean      # turbo run clean
```

### Package-Local Commands

```bash
pnpm --filter @keygraph/shannon run build
pnpm --filter @keygraph/shannon run check

pnpm --filter @shannon/worker run build
pnpm --filter @shannon/worker run check
```

### CLI / Runtime Commands

```bash
./shannon build
./shannon start -u <url> -r <repo>
./shannon logs <workspace>
./shannon status
./shannon stop
./shannon stop --clean
```

For npx mode:

```bash
npx @keygraph/shannon setup
npx @keygraph/shannon start -u <url> -r <repo>
npx @keygraph/shannon uninstall
```

## Test Guidance

- No `test` script is currently defined in the root `package.json`.
- No `test` script is currently defined in `apps/cli/package.json` or `apps/worker/package.json`.
- No checked-in `*.test.*` or `*.spec.*` files were found during repo scan.
- Current verification appears to rely on `pnpm run check`, `pnpm run build`, and `pnpm biome` / `pnpm biome:fix`.

### Running a Single Test

- There is no documented single-test command because there is no configured test runner in the current repository state.
- Do not invent Jest/Vitest/Playwright test commands in patches or docs unless you also add the tooling.
- If future test infrastructure is introduced, document both the suite command and exact single-test invocation here.

## Code Style and Conventions

### Formatting

- Biome is the formatter and linter.
- Use 2-space indentation, single quotes, semicolons, and trailing commas.
- Keep line width within 120 when practical.

### Imports

- Use ESM imports.
- Use explicit `.js` extensions for local TypeScript module imports in source.
- Keep Node built-in imports using the `node:` prefix when applicable.
- Use `import type` for type-only imports.
- Do not leave unused imports; Biome treats them as errors.

### TypeScript

- The repo is strict TypeScript. Respect `strict`, `noUncheckedIndexedAccess`, and `exactOptionalPropertyTypes`.
- Add explicit return types on exported and top-level functions.
- Prefer `function` declarations for top-level functions instead of arrow functions.
- Prefer `readonly` data shapes when mutation is not required.
- For optional properties, prefer conditional object spread over assigning `undefined` directly.
- Do not use `any` unless there is a very strong, local justification; Biome warns on explicit `any`.

### Naming and Structure

- Use descriptive names over terse abbreviations.
- Keep functions focused on one responsibility.
- Prefer early returns and guard clauses over nested branches.
- Do not use nested ternary operators.
- Extract complex conditions into well-named booleans.
- Avoid abstractions for one-off operations.
- Avoid backwards-compatibility shims or stale wrappers when removing old code.

### Error Handling

- This codebase favors explicit error handling over silent failure.
- Follow existing patterns such as `Result<T, E>` utilities in `apps/worker/src/types/result.ts` where callers need structured handling.
- Use typed domain errors like `PentestError` when working in the worker error-classification path.
- Preserve retryability and error classification behavior; do not flatten everything into generic errors.
- Do not swallow errors with empty catches.

### Comments

- Keep comments timeless.
- Use JSDoc for file headers and exported APIs where the codebase already does so.
- In longer functions, use numbered step comments like `// 1. ...` when there are multiple distinct phases.
- Use section dividers like `// === Section ===` in long files.
- Use `// NOTE:`, `// WARNING:`, and `// IMPORTANT:` for constraints or gotchas.
- Do not add obvious comments or conversation/history comments.

## Architecture Rules to Preserve

- Keep Temporal activities thin; business logic belongs in `apps/worker/src/services/`.
- Avoid importing Temporal concerns into service-layer code.
- Reuse existing config parsing, prompt loading, audit logging, and git-checkpoint flows rather than reimplementing them.
- Respect the configuration-driven design and shared types in `apps/worker/src/types/`.
- Match existing boundaries between CLI orchestration and worker logic.

## Validation Expectations for Changes

Run the smallest relevant package checks first, then root checks if the change crosses boundaries:

```bash
pnpm --filter @keygraph/shannon run check
pnpm --filter @shannon/worker run check
pnpm run build
pnpm biome
```

Use judgment:

- CLI-only change → start with `@keygraph/shannon`
- Worker-only change → start with `@shannon/worker`
- Shared/root config change → run root-level validation

## Operational Notes

- Local mode uses `./shannon`; npx mode uses `npx @keygraph/shannon`.
- Local mode reads credentials from `./.env`; npx mode uses env vars or `~/.shannon/config.toml`.
- Local prompts are live-editable under `apps/worker/prompts/`.
- `--pipeline-testing` exists for faster prompt iteration and degraded-tool scenarios.
- Use `host.docker.internal` instead of `localhost` when containers need to reach host services.

## When Unsure

- Read `CLAUDE.md` first.
- Prefer existing nearby file patterns over generic framework habits.
- If a command or test path is not defined in the repo, say so explicitly rather than guessing.
