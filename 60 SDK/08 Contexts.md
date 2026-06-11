---
title: "Contexts"
description: "roundContext and globalContext lifecycle in model execution."
tags:
  - "sdk"
  - "contexts"
category: "sdk"
featured: false
---
Contexts are a mechanism for persisting state between model calls. There are two kinds: roundContext (within a round) and globalContext (between rounds).

## roundContext

The state of the current round. Used for multi-step rounds (sub-rounds): free spins, risk/take, bonus games.

- `null` - no round is active (start of a new game or previous round is complete)
- not `null` - a round is in progress, the next step is expected

The model returns roundContext in every `play()`. The RGS saves it and passes it back in the next `play()` call (and in `init()` if the page was reloaded).

### Lifecycle

```
1. init({ roundContext: null, globalContext })
   -> New game, round not started

2. play({ roundContext: null, ... })
   -> First step. Model returns roundComplete: false
   -> RGS saves roundContext

3. play({ roundContext: {...}, ... })
   -> Sub-round. Model returns roundComplete: false
   -> RGS updates roundContext

4. play({ roundContext: {...}, ... })
   -> Last step. Model returns roundComplete: true
   -> RGS resets roundContext to null

5. play({ roundContext: null, ... })
   -> New round
```

When `roundComplete: true`, the RGS resets roundContext to `null`, regardless of what the model returned in the `roundContext` field.

### Example: accumulated win

```typescript
interface MyRoundContext {
  accumulatedWin: number;
  freeSpinsLeft: number;
}
```

The player won - `roundContext: { accumulatedWin: 200, freeSpinsLeft: 3 }`, `roundComplete: false`. Another spin - `accumulatedWin` grows, `freeSpinsLeft` decreases. When `freeSpinsLeft === 0` - `roundComplete: true`, `win = accumulatedWin`.

### UI Recovery

If the client reloads the page in the middle of a round, `init()` will receive the saved roundContext. The model can return information in initData to restore the UI:

```typescript
init({ roundContext, globalContext }) {
  return {
    initData: {
      pendingRound: roundContext
        ? { accumulatedWin: roundContext.accumulatedWin }
        : null,
    },
  };
}
```

## globalContext

State that persists between rounds. Used for data that needs to live longer than a single round: result history, jackpot progress, statistics.

- `null` - global state is not used (or has not been initialized yet)
- not `null` - data that is updated every round

Unlike roundContext, globalContext **is not reset** when `roundComplete: true`. It is updated every `play()` and passed into every subsequent call.

### Example: result history

```typescript
globalContext: { history: number[] } | null
```

In every `play()`, the model appends the result to the history:

```typescript
play({ gameRequest, rng, roundContext, globalContext }) {
  const history = globalContext?.history ?? [];
  const result = rng.getInt(1, 5);

  // ... game logic ...

  const newHistory = [...history, result].slice(-10); // last 10

  return {
    // ...
    globalContext: { history: newHistory },
  };
}
```

In `init()`, the history is passed to the client for display:

```typescript
init({ roundContext, globalContext }) {
  return {
    initData: {
      history: globalContext?.history ?? [],
    },
  };
}
```

### Reset Session

In the Debug RGS, the **Reset Session** button resets both contexts to `null` and restores the balance to its initial value.

## What's Next

- [[09 Client SDK|Client SDK]] - ITransport, platform parameters
