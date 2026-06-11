---
title: "Game Model"
description: "IModel, init/play, bet and win contract, contexts and round types."
tags:
  - "sdk"
  - "model"
category: "sdk"
featured: false
---
The model is the core of the game. It implements the `IModel<T>` interface and contains all game logic: what happens during initialization and on each player action.

## IModel

```typescript
interface IModel<T extends IGameTypes> {
  roundTypes?: string[];
  init(args: InitArgs<T>): InitResult<T>;
  play(args: PlayArgs<T>): PlayResult<T>;
}
```

Two methods - `init()` and `play()`. The model is a pure synchronous function: it takes data and returns a result. No side effects, no state, no async.

### roundTypes

Optional list of round types the model accepts (e.g., `["normal", "bonusBuy"]`). See [Round Types](#round-types) below.

## init()

Called once when the client loads. Returns initial data for rendering the UI.

```typescript
interface InitArgs<T extends IGameTypes> {
  roundContext: T["roundContext"] | null;
  globalContext: T["globalContext"] | null;
}

interface InitResult<T extends IGameTypes> {
  initData: T["initData"];
}
```

**Arguments:**
- `roundContext` - current round state. Not `null` if the previous round was not completed (e.g., after a page reload in the middle of free spins). The model can use it to restore the UI.
- `globalContext` - global state (history, jackpot progress, etc.).

**Result:**
- `initData` - arbitrary data for the client (game config, paytable, state for UI restoration).

init() does not receive RNG - random numbers are not needed during initialization.

## play()

Called on each player action (spin, card pick, take/risk, etc.).

```typescript
interface PlayArgs<T extends IGameTypes> {
  roundType: string;
  gameRequest: T["gameRequest"];
  rng: IRng;
  roundContext: T["roundContext"] | null;
  globalContext: T["globalContext"] | null;
}

interface PlayResult<T extends IGameTypes> {
  bet: number;
  win: number;
  roundComplete: boolean;
  roundContext: T["roundContext"];
  globalContext: T["globalContext"];
  playData: T["playData"];
  replayData: T["replayData"];
}
```

**Arguments:**
- `roundType` - round type (`"normal"` by default). RGS validates the value before calling the model. See [Round Types](#round-types).
- `gameRequest` - request from the client (action type, bet parameters, player's choice).
- `rng` - random number generator ([[07 RNG|IRng]]). The only source of randomness in the model.
- `roundContext` - round state (or `null` at the start of a new round).
- `globalContext` - global state.

**Result:**
- `bet` - amount to deduct from the balance for this turn (in abstract units).
- `win` - amount to credit to the balance (in abstract units).
- `roundComplete` - `true` if the round is complete, `false` if a next turn (sub-round) is expected.
- `roundContext` - updated round state.
- `globalContext` - updated global state.
- `playData` - data for the client (turn result).
- `replayData` - data for the [[15 Replays|replay]] (recorded at each step).

## bet and win

The model operates in **abstract units**, not real money. RGS converts them to real currency via denomination:

```
real_amount = abstract_units x 10^denomination
```

For example, with denomination=2 (USD): bet=1 means $0.01, bet=100 means $1.00.

RGS applies `normalizePlayResult()` after each model call - it rounds bet and win to integers (`Math.round`). This is a safeguard against floating-point errors.

The balance is updated using the formula:

```
balance = balance - bet + win
```

If the balance goes negative after deduction, RGS returns an `INSUFFICIENT_BALANCE` error (see [[16 Error Handling|error handling]] for details).

### Round bet

**"Round bet"** is the sum of `PlayResult.bet` across all sub-rounds within a single round. The allowed range is defined per-roundType on the SDK / Debug RGS side (see [[10 Debug RGS#Bet validation|Debug RGS]]). The model must satisfy:

- **First sub-round of a round** - `result.bet >= min` (a round must start with a real bet, not zero).
- **Round cumulative** - the sum of all `result.bet` for the round must not exceed `max`.
- For other sub-rounds, `result.bet` may be `0` (e.g. `take`, or free spins inside a bonus round) or any positive value as long as the cumulative stays within `max`.

On violation the SDK returns `INVALID_REQUEST` (Debug RGS) or stops the simulation with an error. This is a safety net against model bugs so that RTP measurements and balance behaviour stay valid.

## roundComplete

The `roundComplete` flag controls the round lifecycle:

- `roundComplete: false` - the round continues. RGS saves roundContext for the next play() call.
- `roundComplete: true` - the round is complete. RGS resets roundContext to `null`. Winnings are credited to the balance.

Typical example: the player guesses correctly - `roundComplete: false`, risk/take is offered. Chooses take - `roundComplete: true`, winnings are credited. Chooses risk and loses - `roundComplete: true`, win=0.

For more details on contexts, see [[08 Contexts|round-context.md]].

## Model Example

Minimal model:

```typescript
import type { IModel } from "@playtagon/model-sdk";
import type { MyGameTypes } from "./types.js";

export const model: IModel<MyGameTypes> = {
  init() {
    return {
      initData: { message: "Welcome!" },
    };
  },

  play({ gameRequest }) {
    return {
      bet: gameRequest.bet,
      win: 0,
      roundComplete: true,
      roundContext: null,
      globalContext: null,
      playData: { message: "You played!" },
      replayData: null,
    };
  },
};
```

A more complex example with sub-rounds, risk/take, and globalContext is available in the [[19 test-game Walkthrough|test-game walkthrough]].

## Round Types

A model can declare multiple round types - for example, a regular spin and a bonus-buy round:

```typescript
export const model: IModel<MyGameTypes> = {
  roundTypes: ["normal", "bonusBuy"],
  // ...
};
```

Rules:
- `"normal"` is always present in the list. RGS adds it automatically if the developer didn't.
- If `roundTypes` is omitted entirely, the default is `["normal"]`.
- RGS validates the `roundType` coming from the client before calling `model.play()`:
  - At the start of a round - the value must be in the declared list.
  - Inside a sub-round - the value must match the one the round started with.
- An invalid `roundType` -> `INVALID_REQUEST` (see [[16 Error Handling|error handling]]).
- Inside `play()`, the model can branch on `args.roundType` (a `switch`).

## What's Next

- [[06 Game Contract|Game Contract]] - the three-directory convention, how the SDK finds the model
- [[07 RNG|RNG]] - IRng, determinism, constraints
