---
title: "SDK Overview"
description: "Playtagon SDK components, target audience and scope."
tags:
  - "sdk"
  - "overview"
category: "sdk"
featured: false
---
# SDK Overview

Playtagon SDK is a framework for developing gambling games with a server-side model, client transport, and a full set of tools for debugging, testing, and analytics.

## Three Components

### model-sdk

A package of types and interfaces defining the game model contract. The model is a pure deterministic function: it takes a player request and RNG, and returns a result. The model knows nothing about currency, balance, or transport.

Key interfaces: [[05 Game Model|IModel]], [[07 RNG|IRng]], [[12 Test Scenarios|IScenarioValidator]], [[14 Analytics|IAnalyticsTracker]], [[13 Simulation|IBot]].

### front-sdk

A contract package: interfaces ([[09 Client SDK|ITransport]]), types, and constants. Contains no implementations - only describes contracts that `@playtagon/runtime` implements.

The game client is written once and works on any transport without changes.

### runtime

The execution environment (`@playtagon/runtime`) - the transport implementation (`Transport` class from `@playtagon/runtime/transport`) and a local development server. Transport makes HTTP requests to the local server.

The local server launches two processes:

- **Express** (:3000) - API server and [[11 Debug Tool|Debug Tool]] (toolbar with balance, scenarios, link to replays)
- **Vite** (:5173) - game client with HMR. Serves `index.html` (game) and `replay.html` ([[15 Replays|replay viewer]], if present).

Also includes infrastructure for [[12 Test Scenarios|test scenarios]] (brute-force search for desired spins), [[13 Simulation|simulation]] (millions of rounds on Worker Threads), and [[14 Analytics|analytics]] (RTP, volatility, win distribution).

## Target Audience

- **Game model developers** - write server logic (init/play), define types, contexts, replay data
- **Client developers** - write game UI, use ITransport for server communication
- **QA and analysts** - write scenarios for manual testing, bots and trackers for automated simulation

## What the SDK Handles

- Transport layer between client and server
- Balance, bets, and currency management (denomination)
- Debug infrastructure: local server, toolbar, hot reload
- Brute-force test scenario search
- Multi-threaded simulation with analytics
- Replay storage and viewing
- Error handling and typing
- [[18 CLI|Packaging a game]] for upload to RGS (CLI)

## What the SDK Does Not Do

- Does not define game logic - that is the model's responsibility
- Does not define the UI - the client is written by the developer from scratch
- Is not a production server - the local runtime server is for development only
- Does not manage authentication or sessions - that is the production RGS's responsibility

## What's Next

- [[02 Quick Start|Quick Start]] - installation, launching, first play
