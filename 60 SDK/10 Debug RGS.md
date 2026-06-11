---
title: "Debug RGS"
description: "Local RGS server, configuration, validation, API and debug runtime behavior."
tags:
  - "sdk"
  - "debug"
category: "sdk"
featured: false
---
# Debug RGS

Debug RGS is a local server for game development and testing. It replaces the production RGS by providing an API, balance management, [[12 Test Scenarios|test scenarios]], [[15 Replays|replays]], and [[11 Debug Tool|Debug Tool]].

## createDebugRgs()

Factory for creating and starting the server. The model and test scenarios are loaded automatically from the [[06 Game Contract|game contract]] - the default export of `<game>/server/model/index.ts`. The developer only passes debug parameters:

```typescript
import { createDebugRgs } from "@playtagon/runtime";

createDebugRgs({
  clientDir: "../client",
  currency: "USD",
  balance: 100_00,
  denomination: 2,
  bets: {
    normal: { min: 1_00, max: 50_00, default: 1_00, step: 50, list: [1_00, 5_00, 10_00, 50_00] },
  },
  platformParams: { lang: "en", mode: "gambling" },
}).start();
```

### DebugRgsOptions

| Parameter | Type | Required | Description |
|-----------|------|:--------:|-------------|
| `clientDir` | `string` | yes | Path to the client directory (served by Vite). Replay viewer lives here as replay.html + replay.ts. |
| `currency` | `string` | yes | Currency code (`"USD"`, `"EUR"`) |
| `balance` | `number` | yes | Starting balance in smallest currency units |
| `denomination` | `number` | yes | Decimal places in currency (USD=2) |
| `bets` | `Record<string, BetsConfig>` | yes | Bets configuration per round type. See [Bet validation](#bet-validation). |
| `platformParams` | `PlatformParams` | yes | Platform parameters (lang, mode, homeUrl, depositUrl) |
| `port` | `number` | no | Express port (default 3000) |
| `vitePort` | `number` | no | Vite client port (default 5173) |
| `model` | `IModel<T>` | no | Override: replaces the auto-loaded `server/model/index.ts`. Useful for debug presets. See [[06 Game Contract#Overrides|game contract]]. |
| `testScenarios` | `IScenarioValidator<T>[]` | no | Override: replaces the auto-loaded `server/scenarios/index.ts`. |

### Bet validation

`bets` is an object whose key is `roundType` and whose value is the `BetsConfig` for that round type:

```typescript
interface BetsConfig {
  min: number;      // minimum round bet
  max: number;      // maximum round bet (sum of deductions across all sub-rounds)
  default: number;  // default bet
  step?: number;    // step (optional)
  list: number[];   // discrete values for UI (may be empty)
}
```

**Startup validation** (at `createDebugRgs.start()`):

- Every `roundType` declared by `IModel.roundTypes` (with `"normal"` always guaranteed) must have a `bets` entry. Otherwise - `Error: bets missing entry for roundType "X"`.
- For every entry: `min > 0`, `max >= min`, `default in [min, max]`, all `list` values in `[min, max]`. Otherwise - the corresponding error.

**Server validation per round** (after each `model.play()`):

- **First sub-round** of the round - `result.bet >= bets[roundType].min`. Otherwise -> `INVALID_REQUEST` (`below min bet`).
- **Any sub-round** - `currentRoundBet + result.bet <= bets[roundType].max` (cumulative for the round). Otherwise -> `INVALID_REQUEST` (`max bet exceeded`).
- On `roundComplete: true` the `currentRoundBet` accumulator is reset.

These rules apply uniformly to plain play and to scenario brute-force - the final result is always checked. See [[16 Error Handling|error handling]] for details.

## dev.ts

Each game has a `server/dev.ts` file serving as the entry point for launching Debug RGS. It is started via:

```bash
npm run dev -w @playtagon/my-game-server
```

In `server/package.json`:

```json
{
  "scripts": {
    "dev": "tsx dev.ts"
  }
}
```

## Two Servers

When `start()` is called, two processes are launched:

| Port | Server | Purpose |
|------|--------|---------|
| :3000 | Express | API, [[11 Debug Tool|Debug Tool]], [[15 Replays|Replay Browser]] |
| :5173 | Vite | Game client with HMR. Serves `index.html` (game) and `replay.html` (replay viewer, if present). |

Ports can be changed via `port`, `vitePort`.

## API endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Debug Tool HTML (toolbar + iframe with client) |
| `POST` | `/api/init` | Initialization: calls `model.init()`, returns balance, currency, bets, initData |
| `POST` | `/api/play` | Play: accepts `{ roundType, gameRequest }`, calls `model.play()`, updates balance |
| `GET` | `/api/state` | Current state: balance, currency, round status, active scenario |
| `POST` | `/api/reset` | Session reset: balance, contexts, replays, scenario |
| `POST` | `/api/scenario` | Select scenario: `{ name }` or `{ name: null }` to clear |
| `GET` | `/api/replays` | List of saved replays |
| `GET` | `/api/replay/:id` | Replay data by roundId |
| `GET` | `/replays` | Replay Browser HTML (page with replay list) |

### Call sequence

1. Client calls `POST /api/init` to receive configuration and initData
2. Client calls `POST /api/play` for each player action
3. If needed, `POST /api/reset` to reset the session

Calling `POST /api/play` before `POST /api/init` will return an error.

### roundType validation

`POST /api/play` accepts a `roundType` field in the body and validates it:
- At the start of a new round - the value must be one of [[05 Game Model#Round Types|`IModel.roundTypes`]] (`"normal"` is always allowed).
- Inside a sub-round - the value must match the one the round started with.

A violation returns `INVALID_REQUEST` (see [[16 Error Handling|error handling]]).

## Balance management

The RGS stores the balance in memory. On each `play()`:

```
balance = balance - bet + win
```

If `balance < bet` after the model call, the RGS returns an `INSUFFICIENT_BALANCE` error and the state remains unchanged.

On `POST /api/reset`, the balance is restored to the initial value from `DebugRgsOptions`.

## What's Next

- [[11 Debug Tool|Debug Tool]] - toolbar, session management
