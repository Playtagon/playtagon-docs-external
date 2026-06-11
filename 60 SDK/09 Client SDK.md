---
title: "Client SDK"
description: "ITransport, platform parameters and client-side SDK contract."
tags:
  - "sdk"
  - "client"
category: "sdk"
featured: false
---
front-sdk is a browser library that provides a transport abstraction between the game client and the server.

## ITransport

```typescript
interface ITransport<T extends IClientTypes> {
  init(): Promise<InitResponse<T>>;
  play(roundType: string, request: T["gameRequest"]): Promise<PlayResponse<T>>;

  onError(handler: TransportErrorHandler): void;
  onNetworkDelay(handler: TransportNetworkDelayHandler): void;

  getLang(): string;
  getMode(): ClientMode;
  getIsMobile(): boolean;
  getHomeUrl(): string | null;
  getDepositUrl(): string | null;
}
```

Parameterized with `IClientTypes` (not `IGameTypes`) - the client does not see server contexts.

## Transport

The transport class is imported from `@playtagon/runtime/transport`:

```typescript
import { Transport } from "@playtagon/runtime/transport";
import type { MyGameTypes } from "../model/types.js";

const transport = new Transport<MyGameTypes>();
```

Opening the client directly (bypassing the Debug Tool on `:3000`) is not supported - `Transport` is initialized from parameters that the Debug Tool passes via the iframe URL.

The `Transport` implementation depends on the environment (debug/prod), but the client always imports it from `@playtagon/runtime/transport` and does not change itself - see [[03 Architecture#Production vs Debug|Architecture]].

**The client imports `@playtagon/runtime` only through the `/transport` and `/vite` entries** - never the bare package, which is the server/dev runtime and would pull Node-only code into the browser bundle. `playtagon check` enforces this.

## InitResponse

Response from `transport.init()`:

```typescript
interface InitResponse<T extends IClientTypes> {
  currency: string;                       // "USD", "EUR"
  balance: number;                        // in smallest currency units (cents)
  denomination: number;                   // decimal places (USD=2)
  bets: Record<string, BetsConfig>;       // bets config per round type
  roundId: string | null;                 // current round ID (if a round is in progress)
  initData: T["initData"];                // data from the model
}
```

Currency, balance, denomination, and bets come from the RGS, **not from the model**. The model only returns `initData`.

### Bet Configuration

```typescript
interface BetsConfig {
  min: number;      // minimum round bet
  max: number;      // maximum round bet (sum of deductions across all sub-rounds)
  default: number;  // default bet
  step?: number;    // bet step (optional)
  list: number[];   // discrete values for UI buttons (may be empty)
}
```

`bets` is now an object keyed by `roundType`; the value is the config for that round type. Each round type has its own range because each can have a different max-win.

Before rendering bet UI the client picks the entry for the current round type:
```typescript
const initResult = await transport.init();
const normalBets = initResult.bets.normal;
// show buttons from normalBets.list, default normalBets.default
```

All values are in the smallest currency units (cents when denomination=2).

## play(roundType, request)

`transport.play()` takes two arguments:
- `roundType` - the round type (`"normal"` for a regular spin; a game can declare extras via [[05 Game Model#Round Types|`IModel.roundTypes`]]).
- `request` - the game-specific request (defined by the game via `IClientTypes.gameRequest`).

The RGS validates `roundType` before invoking the model. Within a single round (from start until `roundComplete: true`), every call must use the **same** `roundType` - attempting to change it mid-round returns `INVALID_REQUEST`.

Example:
```typescript
await transport.play("normal", { type: "spin", bet: 100 });
```

## PlayResponse

Response from `transport.play(roundType, request)`:

```typescript
interface PlayResponse<T extends IClientTypes> {
  balance: number;           // updated balance
  roundId: string | null;    // round ID
  playData: T["playData"];   // data from the model
}
```

## Platform Parameters

Parameters determined by the platform (not the game):

| Method | Type | Description |
|--------|------|-------------|
| `getLang()` | `string` | Player language ([BCP 47](#language-bcp-47-format)) |
| `getMode()` | `"gambling" \| "social"` | Mode: real money or social |
| `getIsMobile()` | `boolean` | Mobile/tablet layout - `isMobile` URL param, else auto-detected from UA |
| `getHomeUrl()` | `string \| null` | "Home" button URL (return to lobby) |
| `getDepositUrl()` | `string \| null` | Balance top-up URL |

In Debug RGS, these parameters are set via `platformParams` when creating the server and passed to the client through the iframe URL query string. Transport parses them from `window.location.search`.

`getIsMobile()` is the exception - it is not part of `platformParams`: it reads the `isMobile` query param when present (`1`/`true` or `0`/`false`), otherwise it auto-detects from the device user agent (phones and tablets count as mobile).

The client can use them for localization and showing/hiding buttons:

```typescript
const lang = transport.getLang();         // "en"
const mode = transport.getMode();         // "gambling"
const isMobile = transport.getIsMobile(); // true on phones/tablets
const homeUrl = transport.getHomeUrl();   // null or URL
```

### Language: BCP 47 format

The `lang` parameter is a language tag conforming to [BCP 47 / RFC 5646](https://www.rfc-editor.org/rfc/rfc5646). The value may contain just a language or a language + region:

- `"en"` - English
- `"en-US"` - English (United States)
- `"ru"` - Russian
- `"pt-BR"` - Portuguese (Brazil)
- `"zh-Hans"` - Chinese (Simplified)

The SDK does not validate the value - the operator/platform is responsible for correctness. The client decides which tags it supports and how to fall back to the closest available one.

## Error Callbacks

Two callbacks for handling errors and delays:

```typescript
transport.onError((error) => {
  // error: { code, message, recoverable }
  console.log("Error:", error.message);
});

transport.onNetworkDelay((isDelay) => {
  // isDelay: true - response has not arrived for more than 5 seconds
  // isDelay: false - response has finally been received
});
```

On error, the `init()` or `play()` promise **never resolves** (pending forever). The error is delivered only through the `onError` callback. This allows handling all errors in one place without cluttering the game logic with try/catch.

For more details, see [[16 Error Handling|error handling]].

## Balance Formatting

Balance and bets are stored in the smallest currency units. To display to the user:

```typescript
const divisor = Math.pow(10, denomination);
const displayBalance = (balance / divisor).toFixed(denomination);
// denomination=2, balance=10000 -> "100.00"
```

## Client Example

Minimal client (init + play + display):

```typescript
import { Transport } from "@playtagon/runtime/transport";
import type { MyGameTypes } from "../model/types.js";

const transport = new Transport<MyGameTypes>();

// Error handling - subscribe before the first init()/play()
transport.onError((error) => {
  showErrorOverlay(error.message, error.recoverable);
});

let balance = 0;
let denomination = 2;

// Initialization
const initResult = await transport.init();
balance = initResult.balance;
denomination = initResult.denomination;

// Player move
const playResult = await transport.play("normal", { bet: 100 });
balance = playResult.balance;
```

## What's Next

- [[10 Debug RGS|Debug RGS]] - local server, configuration, API
