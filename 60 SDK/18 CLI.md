---
title: "CLI"
description: "Playtagon CLI commands for checking and packing game archives."
tags:
  - "sdk"
  - "cli"
category: "sdk"
featured: false
---
# CLI

`@playtagon/cli` is a utility for validating a game and packaging it into a `.playtagon` archive for subsequent upload to an RGS.

## Running

Inside the monorepo, the commands are available through npm workspaces:

```
npx playtagon check <game-dir>
npx playtagon pack <game-dir> [--client | --model | --both] [--output <path>] [--no-check]
```

## pack

Archives the game directory into a file with the `.playtagon` extension (a zip container). By default `pack` runs [`check`](#check) first and aborts if it fails.

**Flags:**

- `--both` (default) - archive includes both `server/` and `client/`
- `--client` - only `client/`
- `--model` - only `server/`
- `--output <path>` / `-o <path>` - output file path. Defaults to `./<game-name>.playtagon`
- `--no-check` - skip the pre-pack `check`

**Game name** is taken from the directory name (`basename`).

**Example:**

```
npx playtagon pack games/test-game
Created /path/to/test-game.playtagon (14028 bytes)
```

## check

Validates that a game is self-consistent and buildable - without needing the production runtime installed.

```
npx playtagon check <game-dir>
```

Steps:

1. **structure** - required files exist (`server/`, `client/`, `server/model/index.ts`)
2. **contract** - the other two contract folders are present: `server/scenarios/index.ts` and `server/simulations/index.ts` (the model is checked in structure). A missing one is a fail. See [[06 Game Contract|game contract]]
3. **tsc** - type-checks `server/` and `client/` (each with its own `tsconfig`)
4. **determinism (source)** - `model/`, `scenarios/`, `simulations/` use no forbidden non-deterministic APIs (`Math.random`, `Date.now`, `performance.now`, `crypto`); all randomness must come through [[07 RNG#Model determinism|IRng]]. Reports `file:line` on a violation
5. **model bundle (prod)** - bundles `server/model/index.ts` with esbuild to confirm the model packs into a single dependency-free module (what production needs)
6. **model bundle (test)** - same, including `server/scenarios/index.ts`
7. **determinism (bundle)** - the same ban over the bundled model graph: catches APIs pulled in through helpers/dependencies that the directory-scoped scan can't see
8. **client imports** - the client imports the runtime only as `@playtagon/runtime/transport` or `/vite`, never the bare `@playtagon/runtime` (which drags Node code into the browser bundle)
9. **client build** - runs `vite build`

The production model bundle is verified locally with the CLI's own esbuild, so the production runtime is not required. A step that cannot run in the current setup is skipped (not failed). The exit code is non-zero if any step fails.

`check` leaves nothing on disk: esbuild bundles in memory, `tsc` runs with `--noEmit`, and the Vite output goes to a temporary directory that is removed afterwards.

## Manifest

`manifest.json` at the archive root:

```json
{
  "name": "test-game",
  "includes": { "model": true, "client": true },
  "sdkVersion": "0.0.1",
  "builtAt": "2026-05-18T09:43:05.025Z"
}
```

**Fields:**

- `name` - game directory name
- `includes` - what RGS should install from the archive (driven by the `--client`/`--model`/`--both` flags)
- `sdkVersion` - version of `@playtagon/model-sdk` the game was built against. Read from `node_modules/@playtagon/model-sdk/package.json`
- `builtAt` - moment of packaging in UTC ISO 8601

## Notes

- The archive contains **raw sources** - `pack` does not transpile or bundle what goes inside; the model and client are built on/before deploy.
- `check` validates structure, types, and buildability locally; the RGS still runs its own validation on upload.
