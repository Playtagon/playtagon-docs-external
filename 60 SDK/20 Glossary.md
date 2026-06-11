---
title: "Glossary"
description: "Playtagon SDK terminology and definitions."
tags:
  - "sdk"
  - "glossary"
category: "sdk"
featured: false
---
# Glossary

## SDK Terms

### denomination

The number of decimal places in a currency. Determines how to convert internal units into displayed amounts.

- USD: denomination = 2 (100 cents = $1.00)
- Balances and bets are stored in minimal units (cents)
- Display formula: `amount / 10^denomination`

### roundContext

The state of the current round, stored on the server between sub-rounds. `null` when no round is active. Reset to `null` when `roundComplete: true`. See [[08 Contexts|contexts]] for details.

### globalContext

State preserved between rounds. `null` if not used. Not reset when a round ends. See [[08 Contexts|contexts]] for details.

### roundComplete

A flag in the `play()` result. `true` means the round is complete, `false` means the next sub-round is expected. See [[05 Game Model|game model]] for details.

### roundType

A round type - a string label the client passes to `transport.play(roundType, request)`. Used to separate regular spins from purchased modes (bonus buy, etc.). The model declares the allowed list via [[05 Game Model#Round Types|`IModel.roundTypes`]]; `"normal"` is always present. The type is fixed within a single round (from start until `roundComplete: true`). An invalid or mismatched value -> `INVALID_REQUEST`.

### round bet

The sum of `PlayResult.bet` across all sub-rounds in a round. The model must stay within the `[bets[roundType].min, bets[roundType].max]` range: the first sub-round must give `result.bet >= min`, and the cumulative for the round must not exceed `max`. The range config is set via `DebugRgsOptions.bets` in dev mode and is hardcoded in the SDK for simulation. See [[10 Debug RGS#Bet validation|Debug RGS]] for details.

### sub-round

A single `play()` call within a round. A round can consist of one or more sub-rounds (e.g., spin -> risk -> risk -> take = 4 sub-rounds).

### gameRequest

A player request sent by the client to the server. The format is defined by the game. See [[04 Type System|type system]] for details.

### initData / playData

Data from the model for the client. `initData` is sent on initialization, `playData` is sent on each move. The format is defined by the game.

### replayData

Data for replaying a single sub-round. Written by the model, stored in Debug RGS, displayed by Replay Viewer. See [[15 Replays|replays]] for details.

### simulation run

A single simulation pass for a specific round type. The config for runs is described by `ISimulationConfig` (`{workerPath, workerData?, trackers?}`) and exported as the default of `simulations/index.ts`. `runGlobalSimulation` runs this single config against every `model.roundTypes`, producing one report entry per type. See [[13 Simulation#ISimulationConfig|Simulation]] for details.

### game contract

The convention of three directories inside `games/<my-game>/server/`: `model/`, `scenarios/`, `simulations/`. All three are required for a published game; during development the SDK gracefully handles the absence of any of them (empty list). Each has its own `index.ts` with a default export of the required type. SDK tools load what they need: `createDebugRgs` - `model/` + `scenarios/`; `runGlobalSimulation` - `model/` + `simulations/`. See [[06 Game Contract|game contract]] for details.

## Analytics Metrics

### RTP (Return To Player)

The average return ratio - the total winnings divided by the total bet. Expressed as a fraction (0.4 = 40%). See [[14 Analytics|analytics]] for details.

### STD (Standard Deviation)

The standard deviation of the win coefficient (win/bet). Shows the spread of results around the mean (RTP).

### Volatility

A volatility measure: STD x 1.96 (95% confidence interval). The higher it is, the greater the spread of winnings.

### Hit Frequency

The fraction of rounds with a win > 0. For example, 0.2 = 20% of rounds are winning.

### Win Coefficient

The win coefficient per round: `totalWin / totalBet`. Bet 100, win 500 -> coefficient 5.0 (5x). Used to calculate RTP, STD, quartiles, and win distribution.

### Win Distribution

The distribution of winnings by thresholds - the fraction of rounds with a coefficient >=1x, >=7x, >=10x, >=50x, >=100x.

### Quartiles (Q1, Median, Q3)

Win coefficient quartiles:
- **Q1** - 25% of rounds have a coefficient below this value
- **Median** - 50% (usually 0, since most rounds are losing)
- **Q3** - 75%

### RTP Volatilities

RTP stability across different sample sizes. Shows how much RTP fluctuates with small (100 rounds) and large (100K rounds) samples.

## What's Next

- [[SDK|Contents]] - back to table of contents
