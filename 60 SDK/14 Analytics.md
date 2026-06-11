---
title: "Analytics"
description: "Simulation trackers, built-in metrics and custom analytics."
tags:
  - "sdk"
  - "analytics"
category: "sdk"
featured: false
---
# Analytics

Trackers collect statistics from [[13 Simulation|simulation]] results. BuiltinTracker is added automatically and calculates core metrics. Custom trackers allow collecting game-specific statistics.

## IAnalyticsTracker

```typescript
interface IAnalyticsTracker<T extends IGameTypes> {
  name: string;
  pushRoundResult(result: IRoundResult<T>): void;
  getResult(): unknown;
}
```

### name

Unique tracker name. Becomes a key in the final JSON report. Names must be unique - duplicates cause an error at simulation start.

### pushRoundResult()

Called after each completed round:

```typescript
interface IRoundResult<T extends IGameTypes> {
  totalBet: number;                // total bet for the round
  totalWin: number;                // total win for the round
  stages: IStageResult<T>[];       // array of sub-rounds
  roundContext: T["roundContext"];  // final round context
}

interface IStageResult<T extends IGameTypes> {
  win: number;                     // sub-round win
  playData: T["playData"];         // sub-round data
}
```

The tracker accumulates data in memory. `stages` contains all sub-rounds - you can analyze each step (via `playData`) or just the totals (via `totalBet`/`totalWin`).

### getResult()

Returns the final result. The type is `unknown` - the tracker defines its own format. The result must be JSON-serializable.

## BuiltinTracker

Added automatically to every simulation (name: `"builtin"`). Calculates:

| Metric | Field | Description |
|--------|-------|-------------|
| RTP | `rtp` | Return To Player - average return ratio (win/bet). 0.4 = 40% |
| STD | `std` | Standard deviation of the win coefficient |
| Volatility | `volatility` | STD x 1.96 (95% confidence interval) |
| Hit Frequency | `hitFrequency` | Proportion of rounds with a win > 0 |
| Avg Win | `avgWin` | Average win coefficient among winning rounds |
| Max Win | `maxWin` | Maximum win coefficient |
| Q1 Win | `q1Win` | First quartile of the win coefficient |
| Median Win | `medianWin` | Median of the win coefficient |
| Q3 Win | `q3Win` | Third quartile of the win coefficient |
| Avg Stages | `avgStagesPerRound` | Average number of sub-rounds per round |
| Win Distribution | `winDistribution` | Proportion of rounds with a win >=1x, >=7x, >=10x, >=50x, >=100x |
| RTP Volatilities | `rtpVolatilities` | RTP stability across different sample sizes |

All coefficients are calculated as `totalWin / totalBet` (win coefficient). For example, a bet of 100 with a win of 500 yields a coefficient of 5.0 (5x).

### BuiltinTracker output example

```json
{
  "rtp": 0.4012,
  "std": 0.8721,
  "volatility": 1.7093,
  "hitFrequency": 0.2004,
  "avgWin": 2.0021,
  "maxWin": 32,
  "q1Win": 0,
  "medianWin": 0,
  "q3Win": 0,
  "avgStagesPerRound": 1.2004,
  "winDistribution": {
    "1x": 0.2004,
    "7x": 0.0012,
    "10x": 0.0003,
    "50x": 0,
    "100x": 0
  },
  "rtpVolatilities": [...]
}
```

## Custom trackers

A custom tracker collects game-specific statistics. It implements the same `IAnalyticsTracker` interface:

```typescript
import type { IAnalyticsTracker, IRoundResult } from "@playtagon/model-sdk";
import type { MyGameTypes } from "../model/types.js";

export function createMyTracker(): IAnalyticsTracker<MyGameTypes> {
  let bonusCount = 0;
  let roundCount = 0;

  return {
    name: "myTracker",

    pushRoundResult(result: IRoundResult<MyGameTypes>) {
      roundCount++;
      // Analyze playData of each sub-round
      for (const stage of result.stages) {
        if (stage.playData.isBonus) {
          bonusCount++;
        }
      }
    },

    getResult() {
      return {
        bonusFrequency: roundCount > 0 ? bonusCount / roundCount : 0,
      };
    },
  };
}
```

Passed to `runSimulation()`:

```typescript
const report = await runSimulation({
  workerPath: new URL("./simulations/default.worker.ts", import.meta.url),
  trackers: [createMyTracker()],
});
```

## Result format

All trackers (including `builtin`) live under the `trackers` key - tracker names cannot collide with run metadata.

**Single run** (`runSimulation`):
```json
{
  "rounds": 1000000,
  "trackers": {
    "builtin": { ... },
    "myTracker": { ... }
  }
}
```

**Global run** (`runGlobalSimulation`) - the config is run against every `model.roundTypes`, and each becomes an entry in `runs`:
```json
{
  "rounds": 1000000,
  "runs": {
    "normal": {
      "trackers": {
        "builtin": { ... },
        "myTracker": { ... }
      }
    },
    "bonus_buy": {
      "trackers": { ... }
    }
  }
}
```

`"builtin"` is always present. Custom trackers add their own keys under `trackers`. See [[13 Simulation#GlobalReport format|Simulation]] for details on both formats.

## What's Next

- [[15 Replays|Replays]] - replayData, viewer
