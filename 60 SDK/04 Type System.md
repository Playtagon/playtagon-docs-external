---
title: "Type System"
description: "Core SDK type interfaces for game, client and replay data."
tags:
  - "sdk"
  - "types"
category: "sdk"
featured: false
---
All key SDK interfaces are parameterized through a generic `T`. This provides end-to-end type safety - from the model to the client, from scenarios to analytics.

## Type Hierarchy

```
IClientTypes          - visible to the client (browser)
+-- initData          - initialization data (config, paytable)
+-- playData          - play result (spin, card, dice)
+-- gameRequest       - player request (bet, choice)

IReplayTypes          - visible to the replay viewer
+-- replayData        - data for replaying one step

IGameTypes extends IClientTypes, IReplayTypes  - full set (server)
+-- initData          -+
+-- playData           | from IClientTypes
+-- gameRequest       -+
+-- replayData        -- from IReplayTypes
+-- roundContext       - round state (between sub-rounds)
+-- globalContext      - state between rounds
```

**IClientTypes** - the minimal set for the client. The client doesn't know about contexts or replays.

**IReplayTypes** - the minimal set for the replay viewer. It only needs replayData.

**IGameTypes** - the full set for the server model. Extends both interfaces and adds contexts.

## How to Define Types for Your Game

Create an interface extending `IGameTypes` and specify concrete types for each field:

```typescript
import type { IGameTypes } from "@playtagon/model-sdk";

// Initialization data - what the client receives at startup
interface MyInitData {
  reelCount: number;
  symbolCount: number;
}

// Play result - what the client receives after each play
interface MyPlayData {
  reels: number[][];
  winLines: number[];
  totalWin: number;
}

// Player request - what the client sends to the server
interface MyGameRequest {
  bet: number;
}

// Round state - stored on the server between sub-rounds
interface MyRoundContext {
  freeSpinsLeft: number;
  multiplier: number;
}

// Replay data - recorded at each step
interface MyReplayData {
  reels: number[][];
  winLines: number[];
}

// Putting it all together
export interface MyGameTypes extends IGameTypes {
  initData: MyInitData;
  playData: MyPlayData;
  gameRequest: MyGameRequest;
  roundContext: MyRoundContext | null;  // null = round not started
  globalContext: null;                  // if not needed
  replayData: MyReplayData;
}
```

A few rules:

- **roundContext** - typically `T | null`, where `null` means the round is not active (all sub-rounds completed).
- **globalContext** - `null` if global state is not needed. Otherwise - a type with data that persists between rounds.
- **replayData** - `null` if replays are not needed at this stage.

## Where Types Are Used

Once you define `MyGameTypes`, you use it as a parameter in all SDK interfaces:

| Interface | Parameter | Uses |
|-----------|-----------|------|
| `IModel<T>` | `T extends IGameTypes` | initData, playData, gameRequest, roundContext, globalContext, replayData. This is the [[06 Game Contract|game contract]] - default-exported from `server/model/index.ts`. |
| `ITransport<T>` | `T extends IClientTypes` | initData, playData, gameRequest |
| `IReplayTransport<T>` | `T extends IReplayTypes` | replayData |
| `IScenarioValidator<T>` | `T extends IGameTypes` | gameRequest, playData, roundContext, globalContext |
| `IBot<T>` | `T extends IGameTypes` | gameRequest, roundContext, globalContext, playData |
| `IAnalyticsTracker<T>` | `T extends IGameTypes` | playData, roundContext |
| `ISimulationConfig<T>` | `T extends IGameTypes` | trackers. See the [[06 Game Contract|game contract]]. |

The transport is parameterized through `IClientTypes` (not `IGameTypes`) - the client should not know about server-side contexts. Similarly, `IReplayTransport` only sees `IReplayTypes`.

## Example: test-game

For reference - the test game types:

```typescript
export interface TestGameTypes extends IGameTypes {
  initData: TestGameInitData;               // { sectors, history, pendingRound }
  playData: TestGamePlayData;               // { winningSector, accumulatedWin, roundComplete, history }
  gameRequest: TestGameRequest;             // { type: "guess"|"take"|"risk", sector?, bet? }
  roundContext: TestGameRoundContext | null; // { accumulatedWin, subRound }
  globalContext: { history: number[] } | null;
  replayData: TestGameReplayData;           // { type, chosenSector, winningSector, bet, accumulatedWin }
}
```

Detailed walkthrough - in [[19 test-game Walkthrough|test-game.md]].

## What's Next

- [[05 Game Model|Game Model]] - IModel, init/play, bet/win contract
