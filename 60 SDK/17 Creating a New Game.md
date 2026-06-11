---
title: "Creating a New Game"
description: "Step-by-step SDK game setup guide."
tags:
  - "sdk"
  - "new-game"
category: "sdk"
featured: false
---
# Creating a New Game

A step-by-step guide to creating a game based on empty-game. Each step covers the bare minimum needed to get everything working.

## 1. Copy empty-game

```bash
cp -r games/empty-game games/my-game
```

A game consists of two npm packages: `server/` and `client/`. Update the names in both `package.json` files.

`games/my-game/server/package.json`:

```json
{
  "name": "@playtagon/my-game-server",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx dev.ts",
    "build:model": "playtagon-build-model ./model/index.ts"
  },
  "dependencies": {
    "@playtagon/runtime": "*",
    "@playtagon/model-sdk": "*"
  },
  "devDependencies": {
    "tsx": "^4.0.0"
  }
}
```

`games/my-game/client/package.json`:

```json
{
  "name": "@playtagon/my-game-client",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "vite build"
  },
  "dependencies": {
    "@playtagon/runtime": "*",
    "@playtagon/front-sdk": "*"
  },
  "devDependencies": {
    "vite": "^6.0.0"
  }
}
```

The server side (model, scenarios, simulations, dev/simulate entries) lives in `server/`. The client side (HTML, UI code, vite config) lives in `client/`. The split via separate `package.json` guarantees Node-only deps never leak into the browser bundle and vice versa.

Install dependencies (from the repo root):

```bash
npm install
```

## 2. Define the Types

`server/model/types.ts` - describe the game data structures (details in [[04 Type System|type system]]):

```typescript
import type { IGameTypes } from "@playtagon/model-sdk";

export interface MyInitData {
  // data sent to the client on init
}

export interface MyPlayData {
  // play result for the client
}

export interface MyGameRequest {
  // player request
  bet: number;
}

export interface MyGameTypes extends IGameTypes {
  initData: MyInitData;
  playData: MyPlayData;
  gameRequest: MyGameRequest;
  roundContext: null;            // null if no sub-rounds
  globalContext: null;           // null if not used
  replayData: null;              // null at the initial stage
}
```

## 3. Write the Model

`server/model/index.ts` - implement [[05 Game Model|IModel]]. The SDK expects a **default export** of the model from this file:

```typescript
import type { IModel } from "@playtagon/model-sdk";
import type { MyGameTypes } from "./types.js";

export const model: IModel<MyGameTypes> = {
  init({ roundContext, globalContext }) {
    return {
      initData: { /* ... */ },
    };
  },

  play({ gameRequest, rng, roundContext, globalContext }) {
    // game logic
    const result = rng.getInt(0, 9);

    return {
      bet: gameRequest.bet,
      win: result >= 7 ? gameRequest.bet * 3 : 0,
      roundComplete: true,
      roundContext: null,
      globalContext: null,
      playData: { /* ... */ },
      replayData: null,
    };
  },
};

export default model;
```

The named `export const model` is for static imports from sibling folders (workers in `simulations/`, for example). The default export is for the SDK loader. See [[06 Game Contract|Game Contract]].

## 4. Write the Client

`client/main.ts` - use [[09 Client SDK|ITransport]]. Types are pulled from the neighbouring `server/` package via a relative path - this is a type-only import, nothing from server ends up in the client's runtime bundle:

```typescript
import { Transport } from "@playtagon/runtime/transport";
import type { MyGameTypes } from "../server/model/types.js";

const transport = new Transport<MyGameTypes>();

// Error handling - subscribe before the first init()/play()
transport.onError((error) => {
  alert(`Error: ${error.message}`);
});

const initResult = await transport.init();
let balance = initResult.balance;
const denomination = initResult.denomination;

// UI and play handling
document.getElementById("btn-play")!.addEventListener("click", async () => {
  const result = await transport.play("normal", { bet: 100 });
  balance = result.balance;
  // Update UI from result.playData
});
```

`client/vite.config.ts` - applies `playtagonPlugin()` from `@playtagon/runtime/vite` (copied from empty-game, no changes needed).

## 5. Set Up dev.ts

`server/dev.ts`:

```typescript
import { createDebugRgs } from "@playtagon/runtime";

createDebugRgs({
  clientDir: "../client",
  currency: "USD",
  balance: 100_00,
  denomination: 2,
  bets: {
    normal: { min: 100, max: 1000, default: 100, step: 50, list: [100, 500, 1000] },
  },
  platformParams: { lang: "en", mode: "gambling" },
}).start();
```

The model from `server/model/index.ts` is loaded automatically - there is no need to pass it. `clientDir: "../client"` points to the sibling client package (CWD at runtime is `server/`).

Verify it works:

```bash
npm run dev -w @playtagon/my-game-server
```

At this point the game is functional: the client sends requests, the model processes them, and the balance updates.

## 6. Add Scenarios

`server/scenarios/index.ts` - write [[12 Test Scenarios|validators]]. The SDK expects a **default export** of the array:

```typescript
import type { IScenarioValidator } from "@playtagon/model-sdk";
import type { MyGameTypes } from "../model/types.js";

const scenarios: IScenarioValidator<MyGameTypes>[] = [
  {
    name: "Always Win",
    isApplicable() { return true; },
    validate(_gameRequest, _before, _after, playData) {
      return playData.win > 0;
    },
  },
];

export default scenarios;
```

`createDebugRgs` picks them up automatically - `dev.ts` does not change. The `scenarios/` folder is required for a published game, but at intermediate development steps you can skip it: the SDK will gracefully handle its absence.

## 7. Add Bots and Simulation

Simulation lives in its own `server/simulations/` folder - alongside bots, trackers, and worker files. This gives isolation: the production model build never looks into this folder.

**Bot** (`server/simulations/bots.ts`) - defines the autoplay strategy. See [[13 Simulation|Simulation]] for details:

```typescript
import type { IBot } from "@playtagon/model-sdk";
import type { MyGameTypes } from "../model/types.js";

export function createMyBot(): IBot<MyGameTypes> {
  return {
    getGameRequest({ bets }) {
      return { bet: bets.default };
    },
  };
}
```

The bot does not take a `bet` parameter - it reads the range from `ctx.bets`, which the SDK sets itself.

**Worker file** (`server/simulations/default.worker.ts`) - worker entry point, creates the model and the bot:

```typescript
import { workerSimulation } from "@playtagon/runtime";
import { model } from "../model/index.js";
import { createMyBot } from "./bots.js";

workerSimulation({ model, bot: createMyBot() });
```

**Register the config** in `server/simulations/index.ts` - default-export of an `ISimulationConfig<T>` object:

```typescript
import type { ISimulationConfig } from "@playtagon/model-sdk";
import type { MyGameTypes } from "../model/types.js";
import { createMyTracker } from "./tracker.js";

const simulations: ISimulationConfig<MyGameTypes> = {
  workerPath: new URL("./default.worker.ts", import.meta.url),
  trackers: () => [createMyTracker()],
};

export default simulations;
```

One config per game; `runGlobalSimulation` runs it against every `model.roundTypes`. `trackers` is a factory so instances are recreated fresh per roundType.

**Entry file** (`server/simulate.ts`) - one-liner that delegates to the SDK:

```typescript
import { runGlobalSimulation } from "@playtagon/runtime";

const report = await runGlobalSimulation();
console.log(JSON.stringify(report, null, 2));
```

Add the script to `server/package.json`:

```json
{
  "scripts": {
    "dev": "tsx dev.ts",
    "simulate": "tsx simulate.ts"
  }
}
```

```bash
npm run simulate -w @playtagon/my-game-server
```

For local debugging of a single strategy (bypassing the global run), create a separate entry file and call `runSimulation` directly - see [[13 Simulation#runSimulation|Simulation]].

## 8. Add Replays

In the model, return `replayData` instead of `null` (details in [[15 Replays|replays]]).

Add `replay.html` and `replay.ts` alongside `index.html` in the same `client/` directory - they'll share code and assets with the game. `dev.ts` does not need to change: the Vite plugin auto-detects `replay.html` and wires it into the build.

## Final Structure

```
games/my-game/
+-- server/                          - npm package @playtagon/my-game-server
|   +-- package.json
|   +-- tsconfig.json
|   +-- dev.ts
|   +-- simulate.ts
|   +-- model/
|   |   +-- index.ts                 - default-export IModel<T>
|   |   +-- types.ts
|   +-- scenarios/
|   |   +-- index.ts                 - default-export IScenarioValidator<T>[]
|   +-- simulations/
|       +-- index.ts                 - default-export ISimulationConfig<T>
|       +-- bots.ts
|       +-- default.worker.ts
+-- client/                          - npm package @playtagon/my-game-client
    +-- package.json
    +-- tsconfig.json
    +-- index.html
    +-- main.ts
    +-- replay.html                  - replay viewer (optional)
    +-- replay.ts
    +-- vite.config.ts
```

## What's Next

- [[19 test-game Walkthrough|test-game Walkthrough]] - walkthrough of a real game
