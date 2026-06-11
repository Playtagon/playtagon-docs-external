---
title: "Game Contract"
description: "Required model, scenarios and simulations directory contract."
tags:
  - "sdk"
  - "contract"
category: "sdk"
featured: false
---
# Game Contract

A game consists of two npm packages: `games/<my-game>/server/` (server-side model + scenarios + simulations) and `games/<my-game>/client/` (HTML/UI). The SDK contract concerns the server side - inside `server/` it expects three subdirectories, each with its own `index.ts` and **default export**. All paths below are relative to `server/`; CWD at `npm run dev -w @playtagon/my-game-server` is `server/`.

## Three directories

| Folder | Default export | Who loads it |
|--------|----------------|--------------|
| `model/` | `IModel<T>` | `createDebugRgs`, `runGlobalSimulation` |
| `scenarios/` | `IScenarioValidator<T>[]` | `createDebugRgs` |
| `simulations/` | `ISimulationConfig<T>` | `runGlobalSimulation` |

**All three folders are required for a published game.** Without scenarios there is nothing to feed Debug Tool; without simulations there is nothing to measure RTP against. Presence of all three is checked by [[18 CLI|`playtagon check`]] (it fails if something is missing); the platform additionally validates at release build time.

**During development the SDK is graceful:** if a folder is missing, the corresponding list is empty - no error. This lets you build a game iteratively (first model, then scenarios, then simulations) and still run `npm run dev` / `npm run simulate` at every step.

## model/

```typescript
// model/index.ts
import type { IModel } from "@playtagon/model-sdk";
import type { MyGameTypes } from "./types.js";

export const model: IModel<MyGameTypes> = {
  init(/* ... */) { /* ... */ },
  play(/* ... */) { /* ... */ },
};

export default model;
```

The default export is for the SDK loader. The named `export const model` is for static imports from sibling folders (workers, for example). If the file is missing, has no default export, or is not an object - `createDebugRgs` / `runGlobalSimulation` fail with a clear error.

**The model and `scenarios/` may depend only on `@playtagon/model-sdk`** - never import `@playtagon/runtime`. The runtime *calls* the model; the model needs nothing from it (RNG and contexts arrive as arguments). Pulling `@playtagon/runtime` into the model graph - even transitively through a shared file, even if unused - bundles dev/simulation-only code into the production model, where that API does not exist, and the container build breaks. `playtagon check` enforces this rule.

## scenarios/ and simulations/

Same rules: `index.ts` with a default export of the required type.

- **scenarios/** - `IScenarioValidator<T>[]`. See [[12 Test Scenarios|Test Scenarios]].
- **simulations/** - `ISimulationConfig<T>`, plus bots, trackers, and the worker file alongside. See [[13 Simulation|Simulation]].

## Auto-loading

`createDebugRgs` loads `model/` + `scenarios/`:

```typescript
// games/my-game/server/dev.ts
import { createDebugRgs } from "@playtagon/runtime";

createDebugRgs({
  clientDir: "../client",
  currency: "USD",
  balance: 100_00,
  denomination: 2,
  bets: { /* ... */ },
  platformParams: { lang: "en", mode: "gambling" },
}).start();
```

`runGlobalSimulation` loads `model/` (for `roundTypes` and coverage check) + `simulations/`:

```typescript
// games/my-game/server/simulate.ts
import { runGlobalSimulation } from "@playtagon/runtime";
const report = await runGlobalSimulation();
console.log(JSON.stringify(report, null, 2));
```

## Overrides

The `createDebugRgs` options support targeted overrides - useful for debug presets. When an override is provided, the corresponding file is not loaded at all:

| Field | What it overrides |
|-------|-------------------|
| `model?: IModel<T>` | Replaces the default export from `model/index.ts`. |
| `testScenarios?: IScenarioValidator<T>[]` | Replaces the default export from `scenarios/index.ts`. |

## What's Next

- [[07 RNG|RNG]] - IRng, determinism, constraints
- [[10 Debug RGS|Debug RGS]] - `createDebugRgs`, options, API
