---
title: "Quick Start"
description: "Install the SDK, run a local game and understand the first play flow."
tags:
  - "sdk"
  - "quick-start"
category: "sdk"
featured: false
---
## Requirements

- Node.js 20+
- npm 10+

## Installation

```bash
git clone <repo-url> playtagon-sdk
cd playtagon-sdk
npm install
```

The project uses npm workspaces. Running `npm install` at the root will install dependencies for all packages (`sdk/*`, `games/*`).

## Running empty-game

```bash
npm run dev -w @playtagon/empty-game-server
```

The console will display URLs. Open `http://localhost:3000` in a browser - this is the Debug Tool (control toolbar + iframe with the game client). You will see a toolbar with balance and an iframe with the empty-game client.

## First Play

Click the **Play** button in the client. Here's what happens:

1. Client calls `transport.play("normal", { action: "spin", bet })`, where `bet` is the default bet from `init().bets.normal.default`
2. `Transport` sends a POST request to `http://localhost:3000/api/play`
3. Debug RGS calls `model.play()` with the request, RNG, and contexts
4. Model returns a result, RGS updates the balance
5. Client receives the response and updates the UI

The balance in the Debug Tool toolbar updates automatically.

## empty-game Structure

```
games/empty-game/
+-- server/
|   +-- dev.ts              - Debug RGS entry point
|   +-- simulate.ts         - simulation entry point
|   +-- model/
|   |   +-- index.ts        - IModel (init/play), default export
|   |   +-- types.ts        - game types (EmptyGameTypes)
|   +-- scenarios/
|   |   +-- index.ts        - validators array (empty)
|   +-- simulations/
|       +-- index.ts        - ISimulationConfig, default export
|       +-- bots.ts         - auto-play bot
|       +-- spin.worker.ts  - simulation worker
+-- client/
    +-- index.html          - HTML page
    +-- main.ts             - client (transport.init / transport.play)
    +-- vite.config.ts      - Vite config
```

The client runs through [Vite](https://vite.dev/) - a dev server with hot module replacement. Vite is installed automatically as a game dependency. `vite.config.ts` applies `playtagonPlugin()` from `@playtagon/runtime/vite` - the plugin configures resolution of SDK packages.

The entry point `server/dev.ts` creates a Debug RGS. The model is auto-loaded from the [[06 Game Contract|game contract]] - the default export of `<game>/server/model/index.ts`. The developer only passes debug parameters:

```typescript
import { createDebugRgs } from "@playtagon/runtime";

createDebugRgs({
  clientDir: "../client",
  currency: "USD",
  balance: 100_00,       // $100.00 in cents
  denomination: 2,       // 2 decimal places (USD)
  bets: {
    normal: { min: 100, max: 100, default: 100, list: [100] },
  },
  platformParams: { lang: "en", mode: "gambling" },
}).start();
```

## empty-game as a Starting Point

empty-game is the simplest possible project with a complete contract: a minimal model, an empty scenarios list, and a basic simulation. It's convenient to use as a template for new games - copy the directory, rename the package, and start writing your own logic.

If you're interested in a more advanced example using all SDK features (scenarios, bots, simulation, replays, error handling) - see the [[19 test-game Walkthrough|test-game walkthrough]].

## What's Next

- [[03 Architecture|Architecture]] - how packages are connected, data flow, ports
