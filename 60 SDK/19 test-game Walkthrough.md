---
title: "test-game Walkthrough"
description: "Walkthrough of the SDK test-game example."
tags:
  - "sdk"
  - "example"
category: "sdk"
featured: false
---
# test-game Walkthrough

test-game is a full-featured example demonstrating all SDK capabilities: sub-rounds, contexts, scenarios, bots, a custom tracker, replays, and error handling.

## Game Rules

The player picks one of 5 sectors and places a bet. The model randomly determines the winning sector:

- **Miss** - the bet is deducted, the round is over
- **Hit** - the bet is deducted, accumulated win = bet x 2. The player is offered a choice:
  - **Take** - collect the accumulated win, the round is over
  - **Risk** - pick a sector and place a bet again. Hit - the accumulated win grows, take/risk choice again. Miss - all accumulated winnings and the current bet are lost

## Project Structure

```
games/test-game/
+-- server/                          - @playtagon/test-game-server
|   +-- package.json
|   +-- dev.ts                       - Debug RGS launcher
|   +-- simulate.ts                  - global run of all simulations
|   +-- simulate-take.ts             - local debug of a single strategy: take
|   +-- simulate-risk.ts             - local debug of a single strategy: risk
|   +-- model/
|   |   +-- index.ts                 - model (init/play), default-export
|   |   +-- types.ts                 - game types
|   +-- scenarios/
|   |   +-- index.ts                 - scenario validators, default-export
|   +-- simulations/
|       +-- index.ts                 - default-export ISimulationConfig
|       +-- bots.ts                  - AlwaysTakeBot, AlwaysRiskBot
|       +-- tracker.ts               - RouletteTracker
|       +-- take.worker.ts           - worker for take
|       +-- risk.worker.ts           - worker for risk
+-- client/                          - @playtagon/test-game-client
    +-- package.json
    +-- index.html                   - game
    +-- main.ts                      - game UI
    +-- replay.html                  - replay viewer
    +-- replay.ts                    - Replay Viewer (subround table)
    +-- vite.config.ts
```

## Types

```typescript
// gameRequest - three kinds of actions
type TestGameRequest =
  | { type: "guess"; sector: number; bet: number }
  | { type: "take" }
  | { type: "risk"; sector: number; bet: number };

// playData - move result
interface TestGamePlayData {
  winningSector: number;      // which sector landed (0 on take)
  accumulatedWin: number;     // current accumulated win
  roundComplete: boolean;     // is the round complete?
  history: number[];          // last 10 sectors
}

// roundContext - state between sub-rounds
interface TestGameRoundContext {
  accumulatedWin: number;
  subRound: number;
}

// globalContext - sector history between rounds
globalContext: { history: number[] } | null;

// replayData - data for replays
interface TestGameReplayData {
  type: "guess" | "risk" | "take";
  chosenSector: number | null;
  winningSector: number;
  bet: number;
  accumulatedWin: number;
}
```

## Model

### init()

Returns the configuration (5 sectors), history from globalContext, and data for restoring the UI from roundContext:

```typescript
init({ roundContext, globalContext }) {
  return {
    initData: {
      sectors: 5,
      history: globalContext?.history ?? [],
      pendingRound: roundContext
        ? { accumulatedWin: roundContext.accumulatedWin, subRound: roundContext.subRound }
        : null,
    },
  };
}
```

### play()

Three branches by gameRequest type:

| Action | bet | win | roundComplete | roundContext |
|----------|-----|-----|:-------------:|--------------|
| guess, hit | bet | 0 | false | `{ accumulatedWin: betx2, subRound: 1 }` |
| guess, miss | bet | 0 | true | null |
| take | 0 | accumulatedWin | true | null |
| risk, hit | bet | 0 | false | `{ accumulatedWin += betx2, subRound++ }` |
| risk, miss | bet | 0 | true | null |

Win is credited only on take - the entire accumulated win is paid out at once.

Each play() call also updates globalContext.history (last 10 landed sectors, FIFO).

## Client

The client uses `new Transport<TestGameTypes>()` from `@playtagon/runtime/transport` and implements the UI:

- On initialization - restoration from `pendingRound` (if the page was reloaded mid-round)
- After guess/risk - display of the winning sector, accumulated win, Take/Risk buttons
- After take or loss - return to sector selection
- Sector history - colored indicators of the last 10 results
- Error handling via `transport.onError()` - overlay with a message and a "Try Again" button

## Scenarios

Seven validators:

### Sector 1-5

Forces a specific sector to land. Always applicable (`isApplicable: true`). On take - skips:

```typescript
validate(gameRequest, _before, _after, playData) {
  if (gameRequest.type === "take") return true;
  return playData.winningSector === sector;
}
```

### Sector 6 (impossible)

An impossible scenario - sector 6 with only 5 sectors. The search spins indefinitely. Useful for testing scenario switching.

### Risk & Win

Visible only when a round is in progress (`isApplicable: roundContext !== null`). Guarantees that on risk the player's chosen sector lands.

## Bots

Two bots for [[13 Simulation|simulation]]:

### AlwaysTakeBot

Always collects the win at the first opportunity:

```typescript
getGameRequest({ bets, roundContext, rng }) {
  if (roundContext === null) {
    return { type: "guess", sector: randomSector(rng), bet: bets.default };
  }
  return { type: "take" };
}
```

### AlwaysRiskBot

Risks up to `maxRisks` times, then collects:

```typescript
getGameRequest({ bets, roundContext, rng }) {
  if (roundContext === null) {
    riskCount = 0;
    return { type: "guess", sector: randomSector(rng), bet: bets.default };
  }
  if (riskCount >= maxRisks) {
    riskCount = 0;
    return { type: "take" };
  }
  riskCount++;
  return { type: "risk", sector: randomSector(rng), bet: bets.default };
}
```

### Registering Runs

In `simulations/index.ts` the global run uses `AlwaysTakeBot` via `take.worker.ts`:

```typescript
import type { ISimulationConfig } from "@playtagon/model-sdk";
import type { TestGameTypes } from "../model/types.js";
import { createRouletteTracker } from "./tracker.js";

const simulations: ISimulationConfig<TestGameTypes> = {
  workerPath: new URL("./take.worker.ts", import.meta.url),
  trackers: () => [createRouletteTracker()],
};

export default simulations;
```

`AlwaysRiskBot` (via `risk.worker.ts`) is used only by the local debugging entry `simulate-risk.ts` - the global run does not exercise it.

### Running the Simulation

```bash
npm run simulate -w @playtagon/test-game-server        # global: both runs at once
npm run simulate:take -w @playtagon/test-game-server   # local: only take
npm run simulate:risk -w @playtagon/test-game-server   # local: only risk
```

The global run (`simulate.ts`) is a one-liner that calls [[13 Simulation#runGlobalSimulation|`runGlobalSimulation`]]. The local entries (`simulate-take.ts` / `simulate-risk.ts`) are for debugging a single strategy and call [[13 Simulation#runSimulation|`runSimulation`]] directly.

## Custom Tracker

RouletteTracker (name: `"roulette"`) collects game-specific statistics:

- **sectorHits** - hit percentage for each sector (should be ~20% each)
- **avgSubRounds** - average number of sub-rounds per round
- **maxSubRounds** - maximum number of sub-rounds

Wired in via the `trackers` field of `ISimulationConfig` (global run) or via the `runSimulation` options (local run).

## Replays

The model returns `TestGameReplayData` on each sub-round: action type, chosen sector, landed sector, bet, accumulated win.

Replay Viewer (report approach) - a sub-round table with Prev/Next navigation:

| # | Type | Chosen | Landed | Result | Bet | Accumulated |
|---|------|--------|--------|--------|-----|-------------|
| 1 | guess | 3 | 3 | HIT | $1.00 | $2.00 |
| 2 | risk | 2 | 4 | MISS | $1.00 | $0.00 |

## Running

```bash
npm run dev -w @playtagon/test-game-server
```

- `http://localhost:3000` - Debug Tool with the game
- `http://localhost:3000/replays` - Replay Browser
- `http://localhost:5173/replay.html?roundId=...` - Replay Viewer

## What's Next

- [[20 Glossary|Glossary]] - terms and definitions
