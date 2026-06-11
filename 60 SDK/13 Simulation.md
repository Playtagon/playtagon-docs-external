---
title: "Simulation"
description: "Automated multi-round simulation, bots, workers and reports."
tags:
  - "sdk"
  - "simulation"
category: "sdk"
featured: false
---
Simulation is an automatic run of millions of rounds to compute [[14 Analytics|metrics]] (RTP, volatility, etc.). It uses Worker Threads for maximum throughput.

## Two launch forms

The SDK provides two API levels:

- **`runGlobalSimulation()`** - the primary form. Auto-loads the model from `<CWD>/model/index.ts` and the simulation config from `<CWD>/simulations/index.ts`, runs the config against every declared [[05 Game Model#Round Types|round type]], returns a single unified report. Use for regression and CI.
- **`runSimulation()`** - the lower-level form. Runs **one** specific worker file with the trackers you supply. Convenient for local debugging of a single strategy.

Mirrors `dev.ts`/`createDebugRgs`: the SDK pulls in the right parts of the game by [[06 Game Contract|directory convention]]; the developer writes only a thin entry file.

## IBot

A bot defines the autoplay strategy - which request to send on each turn:

```typescript
interface IBot<T extends IGameTypes> {
  getGameRequest(ctx: IBotContext<T>): T["gameRequest"];
}

interface IBotContext<T extends IGameTypes> {
  roundType: string;                     // type of the current simulated round
  bets: BetsConfig;                      // bet range for this round type
  rng: IRng;
  roundContext: T["roundContext"] | null;
  globalContext: T["globalContext"] | null;
  playData: T["playData"] | null;        // null on the first turn of a round
}
```

The bot receives the current game state and decides what to do. On the first turn of a round, `roundContext` and `playData` are `null`. Any randomness in the bot comes only from `ctx.rng` (`getInt(min, max)`), never `Math.random()` - a separate RNG stream from the model's.

### Bets in simulation

`ctx.bets` comes **from the SDK**, not from the game config. The SDK hardcodes a wide standard range (`min: 100_000`, `max: 100_000_000`, `default: 100_000`) - the same for all round types and all simulations. This guarantees reproducible RTP measurements independent of dev settings.

The bot picks any value **within** the range (e.g. `bets.default`, or a random value from `bets.list` via `ctx.rng` if non-empty) and places it into its `gameRequest`. The model then uses that value in its calculations.

**The worker validates** `result.bet` the same way as Debug RGS: first sub-round >= min, cumulative <= max. On violation the simulation stops with an error (`below min bet` / `max bet exceeded`) - a safety net against model/bot bugs.

### How to write a bot

```typescript
import type { IBot } from "@playtagon/model-sdk";
import type { MyGameTypes } from "../model/types.js";

export function createMyBot(): IBot<MyGameTypes> {
  return {
    getGameRequest({ bets, roundContext }) {
      if (roundContext === null) {
        // Start of a round - use the SDK-provided default
        return { type: "spin", bet: bets.default };
      }
      // Sub-round - take, no bet
      return { type: "take" };
    },
  };
}
```

A bot can take **no parameters** (as above), or only strategy parameters (e.g. `maxRisks`). **The bet size is not passed to the constructor** - it's read from the context provided by the SDK.

## Worker files

Each simulation run executes inside Worker Threads - each CPU core processes its own portion of rounds. The run logic lives in a worker file:

```typescript
// simulations/take.worker.ts
import { workerSimulation } from "@playtagon/runtime";
import { model } from "../model/index.js";
import { createMyBot } from "./bots.js";

workerSimulation({ model, bot: createMyBot() });
```

`workerSimulation()` accepts either ready-made options, or a factory `(workerData) => options` - if you need to parameterise bots (e.g. `maxRisks`):

```typescript
workerSimulation((data: { maxRisks: number }) => ({
  model,
  bot: createAlwaysRiskBot(data.maxRisks),
}));
```

`workerData` comes from the simulation config (see `ISimulationConfig.workerData` below).

## ISimulationConfig

`ISimulationConfig` is the default export of `simulations/index.ts`. One config per game; the SDK runs it against every `model.roundTypes`:

```typescript
interface ISimulationConfig<T extends IGameTypes> {
  workerPath: string | URL;
  workerData?: JsonValue;
  trackers?: () => IAnalyticsTracker<T>[];
}
```

`workerPath` - one worker file per game (glues model and bot together). Bots, trackers, and the worker file typically live alongside `simulations/index.ts`; only this config object is exposed.

`workerData` - arbitrary JSON forwarded to the `workerSimulation` factory inside the worker. Useful when the bot is parameterised.

`trackers` is a **factory**, not an array of instances. The SDK calls it fresh per `roundType` so tracker state doesn't accumulate across runs. `builtin` is added automatically.

## runGlobalSimulation

```typescript
async function runGlobalSimulation(options?: {
  rounds?: number;       // default 1_000_000
}): Promise<GlobalReport>;
```

What it does:
1. Auto-loads the model (`<CWD>/model/index.ts`) and the config (`<CWD>/simulations/index.ts`).
2. Takes `model.roundTypes` (with `"normal"` guaranteed) and sequentially calls `runSimulation` with the config for each type.
3. Before each call it invokes `config.trackers()` to get fresh tracker instances.
4. Returns a combined report keyed by `roundType`.

Worker threads are recreated between rounds - bots, models, and trackers are fresh per `roundType`.

Usage (entry file):

```typescript
// games/my-game/simulate.ts
import { runGlobalSimulation } from "@playtagon/runtime";

const report = await runGlobalSimulation();
console.log(JSON.stringify(report, null, 2));
```

`package.json`:
```json
{
  "scripts": {
    "simulate": "tsx simulate.ts"
  }
}
```

```bash
npm run simulate -w @playtagon/my-game-server
```

### GlobalReport format

```typescript
interface GlobalReport {
  rounds: number;                         // rounds per roundType
  runs: Record<string, RunReport>;        // key = roundType
}

interface RunReport {
  trackers: Record<string, unknown>;      // key = tracker.name, value = tracker.getResult()
}
```

Example (model with two round types):

```json
{
  "rounds": 1000000,
  "runs": {
    "normal": {
      "trackers": {
        "builtin": { "rtp": 0.96, "std": 0.87, ... },
        "roulette": { "sectorHits": {...}, "avgSubRounds": 1.2 }
      }
    },
    "bonus_buy": {
      "trackers": {
        "builtin": { ... },
        "roulette": { ... }
      }
    }
  }
}
```

All trackers (including `builtin`) live under `trackers` - tracker names cannot collide with run metadata.

## runSimulation

Lower-level API. Runs one worker file - useful in local entry files like `simulate-take.ts` when you want to quickly check a single strategy.

```typescript
interface SimulationOptions<T extends IGameTypes> {
  workerPath: string | URL;             // path to the worker file
  workerData?: JsonValue;               // data for the worker
  trackers?: IAnalyticsTracker<T>[];    // custom trackers
  rounds?: number;                      // number of rounds (default 1 000 000)
  roundType?: string;                   // round type (default "normal")
}
```

`runGlobalSimulation` calls `runSimulation` under the hood - one call per `roundType` declared by the model.

Direct usage:

```typescript
// games/my-game/simulate-take.ts
import { runSimulation } from "@playtagon/runtime";
import { createMyTracker } from "./simulations/tracker.js";

const report = await runSimulation({
  workerPath: new URL("./simulations/take.worker.ts", import.meta.url),
  trackers: [createMyTracker()],
  rounds: 1_000_000,
});

console.log(JSON.stringify(report, null, 2));
```

### SingleReport format

```typescript
interface SingleReport {
  rounds: number;
  trackers: Record<string, unknown>;    // builtin + custom
}
```

```json
{
  "rounds": 1000000,
  "trackers": {
    "builtin": { "rtp": 0.96, ... },
    "roulette": { ... }
  }
}
```

Same as one `runs.<name>` entry in `GlobalReport`, without `roundType` (which is provided via options).

## Progress

Progress is printed to stderr every 100 000 rounds:

```
[simulation] === roundType="normal" ===
[simulation] Starting 8 workers, 1000000 rounds total
[100K/1000K] RTP so far: 95.85%
[200K/1000K] RTP so far: 96.12%
...
```

Progress goes to stderr, the report goes to stdout. This allows output redirection: `npm run simulate > result.json`.

## Safety mechanisms

- **maxSubRounds** (default 500) - if a round does not complete within 500 sub-rounds, the worker stops with an error. Protection against infinite loops in the model.
- **structuredClone** - contexts are cloned before being passed to the bot and the model. Protects against mutation.
- **Bet validation** - the worker stops on a range violation (see [Bets in simulation](#bets-in-simulation)).
- **Errors** - if the bot, the model, or a tracker throws, the worker emits an error with the round number and the stack to stderr.

## What's Next

- [[14 Analytics|Analytics]] - IAnalyticsTracker, BuiltinTracker, metrics
