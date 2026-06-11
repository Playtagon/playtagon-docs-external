---
title: "Architecture"
description: "SDK package relationships, data flow, ports and debug/runtime boundaries."
tags:
  - "sdk"
  - "architecture"
category: "sdk"
featured: false
---
## Repository Structure

```
playtagon-sdk/
+-- sdk/
|   +-- model-sdk/       - types and interfaces (model contract)
|   +-- front-sdk/       - contract (interfaces, types)
|   +-- runtime/         - execution environment (transport, server, simulation)
|   +-- cli/             - packaging and checking a game (playtagon)
+-- games/
|   +-- empty-game/      - minimal template
|   +-- test-game/       - full-featured example
+-- docs/
    +-- en/              - documentation
```

The project is organized as a monorepo with npm workspaces. Packages in `sdk/` are the SDK itself, packages in `games/` are specific games using the SDK.

## Three Packages and Dependencies

```
model-sdk  <-  front-sdk  <-  runtime
  (types)     (contract)     (implementation)
```

- **model-sdk** - has no dependencies. Defines interfaces used by all other packages.
- **front-sdk** - depends on model-sdk (uses IClientTypes, IReplayTypes types). Contains only interfaces, types, and constants.
- **runtime** - depends on model-sdk and front-sdk. Contains implementations: Transport, server, simulation, analytics.

A game depends on all three: the model implements interfaces from model-sdk, the client uses runtime/transport, the entry point (`dev.ts`) uses runtime.

## Data Flow

```
Browser (game client)
    |
    |  transport.init() / transport.play()
    v
Transport (runtime)
    |
    |  HTTP POST /api/init, /api/play
    v
Express server (runtime, :3000)
    |
    |  model.init() / model.play()
    v
Game model (model-sdk)
    |
    |  result: { bet, win, roundComplete, playData, ... }
    v
Express server
    |
    |  updates balance, saves contexts
    v
Transport
    |
    |  JSON response
    v
Client updates UI
```

The client does not call the model directly. All communication goes through the transport and server.

## Debug RGS: Two Servers

When running `npm run dev`, two processes are started:

| Port | Server | Purpose |
|------|--------|---------|
| :3000 | Express | API (`/api/init`, `/api/play`, ...), [[11 Debug Tool|Debug Tool]] (toolbar + iframe), [[15 Replays|Replay Browser]] |
| :5173 | Vite | Game client with HMR. Serves `index.html` (game) and `replay.html` ([[15 Replays|replay viewer]], if present). |

Debug Tool at `:3000` loads the game client in an iframe from `:5173`. The toolbar and client communicate with the server independently of each other.

## Separation of Concerns

### The Model Knows

- Game logic (init/play)
- Its own types (initData, playData, gameRequest, contexts)
- How much to deduct (bet) and how much to award (win)

### The Model Does Not Know

- Currency, denomination, absolute balance
- Available bet options
- How data is delivered to the client
- About scenarios, simulation, replays (except returning replayData)

### The RGS (Debug RGS / production) Knows

- Balance, currency, denomination
- Bet configuration (bets)
- Platform parameters (lang, mode, homeUrl, depositUrl)
- How to store and update contexts

### The Client Knows

- How to display initData and playData
- How to format the balance (`amount / 10^denomination`)
- How to build gameRequest from player actions

## Production vs Debug

In production, the `@playtagon/runtime` package is swapped entirely - Transport implements the protocol of the specific RGS. The model and client **do not change** - they work with abstract interfaces:

- The model implements `IModel<T>` - any RGS can call it
- The client uses `ITransport<T>` - it doesn't care who responds to requests

This is the core principle of the SDK: **a game is written once and works on any transport**.

## What's Next

- [[04 Type System|Type System]] - IGameTypes, IClientTypes, IReplayTypes
