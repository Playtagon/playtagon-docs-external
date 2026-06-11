---
title: "Replays"
description: "Replay data, storage, browser and viewer integration."
tags:
  - "sdk"
  - "replays"
category: "sdk"
featured: false
---
# Replays

Replays allow viewing the history of played rounds - each sub-round with its data. The model returns `replayData` at each step, Debug RGS accumulates and stores it, and the developer decides how to display them.

## replayData in the model

On each `play()` call, the model returns `replayData` - data for reproducing that step. The format is defined by the game:

```typescript
interface MyReplayData {
  reels: number[][];
  winLines: number[];
  totalWin: number;
}

// In play():
return {
  bet: 100,
  win: 500,
  roundComplete: true,
  playData: { ... },
  replayData: { reels: [[1,2,3], [4,5,6]], winLines: [0, 2], totalWin: 500 },
  // ...
};
```

If replays are not needed at this stage, return `null`:

```typescript
replayData: null,
```

What data to include in `replayData` is up to the developer. It depends on the chosen approach to displaying replays.

## Display approaches

The SDK does not dictate the replay display format. The developer chooses an approach depending on the task:

### Report (layout)

A separate page that displays round data as a table, list, or other static layout. It is sufficient to include key numbers and results in `replayData` - the viewer will render them.

Suitable for: QA, analytics, quick review. Implementation is simpler - plain HTML/CSS based on the data.

Example: test-game uses exactly this approach - a table with sub-rounds, sectors, and wins.

### In-game playback

The game client can operate in replay mode - it receives `replayData` and plays back the round with animations, as if the player were playing. `replayData` needs to include everything the client requires for playback (positions, states, event sequences).

Suitable for: demonstrations, visual correctness verification, player disputes. Implementation is more complex - the client must support playback mode.

### Choosing an approach

Both approaches use the same SDK infrastructure: the model returns `replayData`, Debug RGS stores it, `IReplayTransport` loads it. The only difference is what the viewer displays and what data the model puts into `replayData`.

## Storage

Debug RGS stores replays in memory:

- Maximum 100 rounds
- When exceeded - the oldest is removed (FIFO)
- Reset Session clears all replays

Each round receives a unique `roundId` (UUID), which is returned to the client in `init()` and `play()` responses.

## Data structure

### ReplayRound

```typescript
interface ReplayRound<T extends IReplayTypes> {
  id: string;               // roundId (UUID)
  currency: string;          // currency
  denomination: number;      // decimal places
  balanceAtStart: number;    // balance at round start
  totalBet: number;          // total bet for the round
  totalWin: number;          // total win for the round
  stages: ReplayStage<T>[];  // array of sub-rounds
}
```

### ReplayStage

```typescript
interface ReplayStage<T extends IReplayTypes> {
  replayData: T["replayData"];  // data returned by the model
}
```

## Replay Browser

A page with a list of saved replays, available at `http://localhost:3000/replays`. There is also a link from the [[11 Debug Tool|Debug Tool]].

Shows all saved rounds with links to the Replay Viewer.

## Replay Viewer

A second HTML entry inside the game client - `replay.html` + `replay.ts`. It lives next to `index.html` in the same `client/` directory, so it automatically shares code, assets, and build configuration with the game.

### How it works

1. Replay Browser generates a link: `http://localhost:5173/replay.html?roundId=<uuid>`
2. Replay Viewer loads and calls `replayTransport.loadReplay()`
3. `ReplayTransport` takes `roundId` from the URL query and requests `GET /api/replay/:id`
4. Viewer displays the round data using the chosen method

The Debug RGS Vite dev server serves both HTML files (`index.html` and `replay.html`). In a production build, `playtagonPlugin()` auto-detects `replay.html` and adds it to `rollupOptions.input`, producing two entry bundles with shared chunks.

### IReplayTransport

```typescript
interface IReplayTransport<T extends IReplayTypes> {
  loadReplay(): Promise<ReplayRound<T>>;
}
```

One method - `loadReplay()`. No parameters needed - `roundId` is taken from the URL.

Factory:

```typescript
import { ReplayTransport } from "@playtagon/runtime/transport";
import type { MyGameTypes } from "../model/types.js";

const replayTransport = new ReplayTransport<MyGameTypes>();
const replay = await replayTransport.loadReplay();
```

### How to write a Replay Viewer

Create two files next to `index.html`/`main.ts`:

```
client/
+-- index.html       - game
+-- main.ts          - game entry
+-- replay.html      - replay viewer
+-- replay.ts        - replay viewer entry
```

Minimal example (report approach):

```typescript
// client/replay.ts
import { ReplayTransport } from "@playtagon/runtime/transport";
import type { MyGameTypes } from "../model/types.js";

const transport = new ReplayTransport<MyGameTypes>();
const replay = await transport.loadReplay();

document.getElementById("app")!.innerHTML = `
  <h1>Round ${replay.id}</h1>
  <p>Bet: ${replay.totalBet}, Win: ${replay.totalWin}</p>
  <p>Stages: ${replay.stages.length}</p>
  ${replay.stages.map((s, i) => `
    <div>Stage ${i + 1}: ${JSON.stringify(s.replayData)}</div>
  `).join("")}
`;
```

No extra option in `dev.ts` is required - if `replay.html` is present in `clientDir`, Vite serves it and Debug RGS generates links to it on `/replays`. If `replay.html` is absent, replays are still stored and available via the API, but the Replay Browser has nothing to link to.

## What's Next

- [[16 Error Handling|Error Handling]] - TransportError, codes, callbacks
