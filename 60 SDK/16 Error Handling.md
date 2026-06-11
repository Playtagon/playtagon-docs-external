---
title: "Error Handling"
description: "TransportError codes, callbacks and debug error behavior."
tags:
  - "sdk"
  - "errors"
category: "sdk"
featured: false
---
The SDK provides a typed error system. The client receives errors through the `onError` callback rather than via promise rejection, allowing all errors to be handled in one place.

## TransportError

```typescript
interface TransportError {
  code: number;         // error code
  serverCode?: number;  // raw code from the server (when present)
  message: string;      // text description
  recoverable: boolean; // whether recovery is possible (show "Try Again")
}
```

### code

Numeric error code. The client handles known codes; unknown ones go through a fallback.

### serverCode

The original error code from the server, as-is (present only when the error came from a server response). For display and logs - branch on `code`.

### message

Text description for logging or display.

### recoverable

- `true` - the error is recoverable; a "Try Again" button can be shown
- `false` - the game is "dead" and cannot continue (e.g., the session has expired)

## TransportErrorCodes

```typescript
const TransportErrorCodes = {
  UNKNOWN: 0,
  NETWORK: 1,
  SERVER: 2,
  INSUFFICIENT_BALANCE: 3,
  SESSION_EXPIRED: 4,
  GEO_BLOCKED: 5,
  INVALID_REQUEST: 6,
  CLIENT_VERSION: 7,
} as const;
```

| Code | Name | When it occurs |
|------|------|----------------|
| 0 | UNKNOWN | Unknown error |
| 1 | NETWORK | Network error (fetch failed) |
| 2 | SERVER | Server error (model.play() threw, scenario error, etc.) |
| 3 | INSUFFICIENT_BALANCE | Insufficient funds for the bet |
| 4 | SESSION_EXPIRED | Session expired (production) |
| 5 | GEO_BLOCKED | Geo-blocking (production) |
| 6 | INVALID_REQUEST | Invalid request from the client (e.g., unknown `roundType` or mismatch in a sub-round) |
| 7 | CLIENT_VERSION | Client version rejected (production) |

## onError

> **Important:** subscribe to `onError` **before** the first `init()` call. If `init()` returns an error (e.g., `model.init()` throws), the `init()` promise stays pending forever - without a subscriber the error is silently lost and the client hangs on loading.

```typescript
transport.onError((error: TransportError) => {
  console.log(error.code, error.message, error.recoverable);
});
```

When an error occurs, the `init()` or `play()` promise **does not resolve** - it remains in a pending state forever. The error is delivered only through the callback. This means:

- Game logic does not need try/catch
- All errors are handled in one place
- The client decides what to show (overlay, message, retry button)

Example of handling in the client:

```typescript
transport.onError((error) => {
  const overlay = document.createElement("div");
  overlay.className = "error-overlay";
  overlay.innerHTML = `
    <p>${error.message}</p>
    ${error.recoverable ? '<button onclick="location.reload()">Try Again</button>' : ""}
  `;
  document.body.appendChild(overlay);
});
```

## onNetworkDelay

```typescript
transport.onNetworkDelay((isDelay: boolean) => {
  if (isDelay) {
    showSpinner();   // no response received for more than 5 seconds
  } else {
    hideSpinner();   // response received
  }
});
```

This is **not an error** - it is simply an indicator that the response is delayed. The timer is 5 seconds:

- `isDelay: true` - 5 seconds have passed with no response. A spinner can be shown.
- `isDelay: false` - the response has finally been received (or an error occurred, which will go to onError).

## Debug RGS Errors

Debug RGS returns errors in the following format:

```
HTTP 4xx/5xx
{ "error": "message", "code": number, "recoverable": boolean }
```

Transport parses this JSON and calls `onError` with the received fields. All Debug RGS errors have `recoverable: true` - there are no fatal errors in debug mode.

| Situation | HTTP | Code | Message |
|-----------|------|------|---------|
| `model.init()` throws an exception | 500 | SERVER | model.init() threw |
| `model.play()` throws an exception | 500 | SERVER | model.play() threw |
| Scenario error | 500 | SERVER | Error description |
| Balance < bet | 400 | INSUFFICIENT_BALANCE | Insufficient balance |
| `play()` before `init()` | 400 | SERVER | play() called before init() |
| Missing or invalid `roundType` | 400 | INVALID_REQUEST | Missing or invalid roundType |
| Unknown `roundType` | 400 | INVALID_REQUEST | Unknown roundType "..." |
| `roundType` does not match the active round | 400 | INVALID_REQUEST | roundType mismatch: ... |
| `result.bet` on first sub-round < `bets[roundType].min` | 400 | INVALID_REQUEST | below min bet: result.bet=... < min=... |
| Sum of `result.bet` for the round > `bets[roundType].max` | 400 | INVALID_REQUEST | max bet exceeded: round sum ... > max=... |

The error stack trace is printed to the server's stderr - check the terminal for debugging.

## What's Next

- [[17 Creating a New Game|Creating a New Game]] - step-by-step guide
