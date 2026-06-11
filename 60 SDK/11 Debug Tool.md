---
title: "Debug Tool"
description: "Local debug toolbar, session controls, scenarios and replay browser links."
tags:
  - "sdk"
  - "debug-tool"
category: "sdk"
featured: false
---
# Debug Tool

Debug Tool is a web page at `http://localhost:3000` that combines a control toolbar and an iframe with the game client.

## Structure

```
+--------------------------------------+
|  Debug Tool (toolbar)                |
|  Balance: $100.00  [Reset] [Replays] |
|  Scenario: [None v]                  |
+--------------------------------------+
|                                      |
|  iframe: http://localhost:5173       |
|  (game client)                       |
|                                      |
+--------------------------------------+
```

The toolbar and the client **are not directly connected**. The toolbar communicates with the server through its own requests, and the client through its own. They do not exchange any messages with each other.

## Display

- **Balance** - current balance, formatted according to denomination and currency
- **Round status** - idle or in progress
- **Active scenario** - name of the selected scenario or None

Data is updated automatically in real time.

## Controls

### Reset Session

Resets everything to the initial state:
- Balance -> initial value
- roundContext and globalContext -> `null`
- Replays -> cleared
- Active scenario -> None
- Scenario search -> stopped

### Replays

Link to the [[15 Replays|Replay Browser]] (`/replays`) - a page with a list of saved replays.

### Scenario Selection

Dropdown with a list of available [[12 Test Scenarios|scenarios]]:

- **None** - regular play with random RNG, test logic is off
- **Scenario name** - enables brute-force search for the selected scenario

Game scenarios update dynamically - scenarios for which `isApplicable()` returns `false` are hidden from the dropdown. If the active validator becomes inapplicable, it is automatically reset to None.

The selected scenario stays active and applies to every play until the tester changes it or selects None.

### Search Indicator

- `Searching: N iterations...` - search in progress
- `Found in N iterations` - desired result has been found

## What's Next

- [[12 Test Scenarios|Test Scenarios]] - IScenarioValidator, brute-force search
