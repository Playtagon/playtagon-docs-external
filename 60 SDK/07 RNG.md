---
title: "RNG"
description: "IRng usage, determinism requirements and debug RNG behavior."
tags:
  - "sdk"
  - "rng"
category: "sdk"
featured: false
---
The model receives a random number generator as an argument of `play()`. This is the only allowed source of randomness.

## IRng

```typescript
interface IRng {
  getInt(min: number, max: number): number;
}
```

A single method - `getInt(min, max)`. Returns an integer in the range `[min, max]` inclusive on both ends.

```typescript
rng.getInt(1, 5);  // -> 1, 2, 3, 4 or 5
rng.getInt(0, 1);  // -> 0 or 1
rng.getInt(0, 99); // -> 0 to 99
```

## Model determinism

The model must be **strictly deterministic**: given the same inputs (gameRequest, contexts) and the same sequence of numbers from RNG, the result is always the same.

This means:

- You must not use `Math.random()` in the model
- You must not depend on `Date.now()`, timers, or external sources
- All randomness must come exclusively through `rng.getInt()`

The main violations are caught by [[18 CLI|`playtagon check`]]: system randomness and clock access (`Math.random`, `Date.now`, `performance.now`, `crypto`) in `model/`/`scenarios/`/`simulations/` fail the check.

Determinism is needed for:

- **Test scenarios** - [[12 Test Scenarios|brute-force search]] iterates over RNG until the desired result is found
- **Simulation** - [[13 Simulation|millions of rounds]] run in parallel across workers: the model is a pure function with no state of its own
- **Production** - the real RGS provides its own numbers, and the model works the same way

## Implementation in Debug RGS

The Debug RGS uses a wrapper around `Math.random()`:

```typescript
function createMathRandomRng(): IRng {
  return {
    getInt(min, max) {
      return min + Math.floor(Math.random() * (max - min + 1));
    },
  };
}
```

## What's Next

- [[08 Contexts|Contexts]] - roundContext and globalContext
