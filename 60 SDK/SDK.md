---
title: "SDK"
description: "Playtagon SDK documentation for developers building, testing and packaging games."
tags:
  - "sdk"
  - "mvp"
category: "sdk"
featured: false
---
Playtagon SDK is the developer-facing integration path for teams that build game logic in code and connect it to Playtagon. It includes the server-side game model contract, the client transport contract, local debug tools, scenarios, simulation, analytics, replays and CLI packaging into a `.playtagon` archive.

In the Playtagon app workflow, developers build locally, package the game with the CLI, upload the archive in [[Resources]], test it in [[Launch Game]] and submit the selected version through [[Publishing]].

## Contents

### Getting Started

- [[01 SDK Overview|SDK Overview]] - what the SDK is, its components and target audience.
- [[02 Quick Start|Quick Start]] - installation, local launch and the first play flow.

### Architecture

- [[03 Architecture|Architecture]] - package relationships, data flow, debug servers and runtime boundaries.
- [[04 Type System|Type System]] - `IGameTypes`, `IClientTypes`, `IReplayTypes` and related generics.

### Server Model

- [[05 Game Model|Game Model]] - `IModel`, `init`, `play`, bet/win behavior, contexts and round types.
- [[06 Game Contract|Game Contract]] - `model/`, `scenarios/` and `simulations/` directory convention.
- [[07 RNG|RNG]] - `IRng`, determinism requirements and debug RNG behavior.
- [[08 Contexts|Contexts]] - `roundContext` and `globalContext` lifecycle.

### Client

- [[09 Client SDK|Client SDK]] - `ITransport`, platform parameters and client-side contract.

### Development Tools

- [[10 Debug RGS|Debug RGS]] - local server, configuration, API and validation behavior.
- [[11 Debug Tool|Debug Tool]] - toolbar, balance/session controls, scenarios and replays.
- [[12 Test Scenarios|Test Scenarios]] - `IScenarioValidator`, brute-force search and examples.

### Testing And Analytics

- [[13 Simulation|Simulation]] - bots, Worker Threads, simulation runs and report formats.
- [[14 Analytics|Analytics]] - built-in and custom trackers, RTP, volatility and metrics.
- [[15 Replays|Replays]] - `replayData`, replay browser and replay viewer integration.

### Advanced

- [[16 Error Handling|Error Handling]] - transport errors, codes and callbacks.
- [[17 Creating a New Game|Creating a New Game]] - step-by-step game setup guide.
- [[18 CLI|CLI]] - `check` and `pack` commands for validation and `.playtagon` archives.
- [[19 test-game Walkthrough|test-game Walkthrough]] - walkthrough of the SDK example game.
- [[20 Glossary|Glossary]] - SDK terminology and definitions.

## Related Playtagon App Pages

- [[Types Play SDK|Play SDK creator type]] - create a Play SDK game in Playtagon.
- [[Resources]] - upload and validate `.playtagon` archives.
- [[Launch Game]] - test a built game version and supported scenarios.
- [[Publishing]] - submit an approved archive version for review and release.
- [[AI Wizard]] - ask SDK and publishing questions inside Playtagon.
