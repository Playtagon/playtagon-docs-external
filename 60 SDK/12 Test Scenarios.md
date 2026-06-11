---
title: "Test Scenarios"
description: "IScenarioValidator, scenario search, validation and examples."
tags:
  - "sdk"
  - "scenarios"
category: "sdk"
featured: false
---
Scenarios allow you to find specific game situations via brute-force in debug mode: for example, force a win, trigger a bonus, or get a particular combination.

The name `hold` is reserved by the SDK. If `scenarios/index.ts` contains a validator with that name, Debug RGS fails at startup.

## IScenarioValidator

```typescript
interface IScenarioValidator<T extends IGameTypes> {
  name: string;
  isApplicable(context: Contexts<T>): boolean;
  validate(
    gameRequest: T["gameRequest"],
    before: Contexts<T>,
    after: Contexts<T>,
    playData: T["playData"],
  ): boolean;
}
```

### name

Unique scenario name. Displayed in the [[11 Debug Tool|Debug Tool]] dropdown.

### isApplicable()

Determines whether the scenario is visible in the current game state.

```typescript
interface Contexts<T extends IGameTypes> {
  roundContext: T["roundContext"] | null;
  globalContext: T["globalContext"] | null;
}
```

- Returns `true` - the scenario is shown in the dropdown
- Returns `false` - the scenario is hidden

Called after every state change. If the active scenario becomes inapplicable, it is automatically reset to None.

For example, a scenario may only be applicable when a round is in progress:

```typescript
isApplicable(context) {
  return context.roundContext !== null;
}
```

Or always:

```typescript
isApplicable() {
  return true;
}
```

### validate()

Checks whether the result of a `model.play()` call matches the scenario.

**Arguments:**
- `gameRequest` - the player's request (the same one the client sent)
- `before` - contexts before the model call
- `after` - contexts after the model call
- `playData` - the result that will be sent to the client

**Returns:**
- `true` - the result matches, send it to the client
- `false` - does not match, keep searching

## Brute-force search

When a scenario is active and the client calls `play()`, the Debug RGS performs a search:

1. Saves the current contexts (`before`)
2. In a loop:
   - Clones the contexts (`structuredClone`) - protection against mutation
   - Creates a new RNG
   - Calls `model.play()` with the same gameRequest
   - Calls `validator.validate()`
   - If `true` - commits the result, responds to the client
   - If `false` - retries
3. Batches of 1000 iterations with pauses between them (the event loop is not blocked)

The client knows nothing about the search - it simply waits for a response. From its perspective, it is a regular `play()`, just slightly slower.

At any given moment, **only one `/api/play` is handled**. If the client sends a second request while the first is still in flight (searching or a regular play) - the server responds with `409 INVALID_REQUEST` "Another play in progress". A well-behaved client sends the next play only after receiving the response to the previous one.

The selected scenario **stays active** and applies to every subsequent `play()` (including sub-rounds) until the tester changes it or selects None.

## Switching scenarios on the fly

- Selecting a different scenario during a search - the current search is aborted, a new one starts
- Selecting None during a search - the search is aborted; the current `play()` executes once with a random RNG (test mode off)
- Selecting the same scenario - ignored, the search continues
- Reset Session during a search - the client receives `409 SERVER` "Aborted", state is reset to defaults, active scenario becomes None

## Where scenarios live

Scenarios live in the game's `server/scenarios/` directory with its own `index.ts`. The file must contain a **default export** of an `IScenarioValidator<T>[]` array:

```typescript
// games/my-game/server/scenarios/index.ts
import type { IScenarioValidator } from "@playtagon/model-sdk";
import type { MyGameTypes } from "../model/types.js";

const scenarios: IScenarioValidator<MyGameTypes>[] = [
  // ... validators
];

export default scenarios;
```

`createDebugRgs` picks them up automatically at start - there is no need to pass them through options. The `scenarios/` folder is required for a published game, but at dev time the SDK is graceful: if the folder is absent, Debug Tool simply shows an empty scenario list. See [[06 Game Contract|game contract]].

## Examples from test-game

Below are scenario examples from [[19 test-game Walkthrough|test-game]]. In test-game, the player picks one of 5 sectors, and the model randomly determines the winning sector. If the player guesses correctly, they can either collect the winnings (take) or gamble (risk). The take action does not depend on RNG - the player simply collects what has been accumulated.

### Simple: target sector

Forces a specific sector to land. If the action is take (not RNG-dependent), it passes without searching:

```typescript
function sectorValidator(sector: number): IScenarioValidator<TestGameTypes> {
  return {
    name: `Sector ${sector}`,
    isApplicable() {
      return true;
    },
    validate(gameRequest, _before, _after, playData) {
      if (gameRequest.type === "take") return true;
      return playData.winningSector === sector;
    },
  };
}
```

### Conditional: Risk & Win

Only visible when a round is in progress (take/risk choice is available). Guarantees that on risk, the chosen sector lands:

```typescript
{
  name: "Risk & Win",
  isApplicable(context) {
    return context.roundContext !== null;
  },
  validate(gameRequest, _before, _after, playData) {
    if (gameRequest.type === "take") return true;
    if (gameRequest.type === "risk") {
      return playData.winningSector === gameRequest.sector;
    }
    return true;
  },
}
```

### Impossible: testing infinite search

A scenario that will never be found (sector 6 with only 5 sectors). Useful for testing scenario switching and search termination:

```typescript
{
  name: "Sector 6 (impossible)",
  isApplicable() {
    return true;
  },
  validate(gameRequest, _before, _after, playData) {
    if (gameRequest.type === "take") return true;
    return playData.winningSector === 6;
  },
}
```

## What's Next

- [[13 Simulation|Simulation]] - IBot, Worker Threads, running
