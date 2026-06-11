---
title: "SDK"
description: "Playtagon SDK for developers: .playtagon archives, rounds, custom scenarios and AI-friendly docs."
tags:
  - "sdk"
  - "mvp"
category: "sdk"
featured: false
---
## Purpose

Playtagon SDK is the developer-facing integration path for studios that build game logic in code and connect it to Playtagon. In the MVP, this is the only active creator type.

The SDK path is designed for teams that build locally, package a game as a `.playtagon` archive, upload that archive to [[Resources]], test it in [[Launch Game]] and submit it through [[Publishing]].

## AI-Friendly SDK

The SDK documentation is written so both developers and AI assistants can use it reliably. [[AI Wizard]] can answer SDK questions directly inside Playtagon.

- Every integration step should be documented with endpoints, parameters and expected formats.
- API methods should state the permissions and context required to call them.
- Data structures should be described in a machine-readable way where possible.
- SDK documentation should be usable from the product, especially during archive upload, launch testing and publishing.

## Terminology: Rounds, Not Spins

The SDK uses neutral terminology because Playtagon can support more than slot-style games.

> **Round** is the base game unit across SDK, documentation, launch testing and publishing checks.

Use `round` in external docs, API examples and support answers unless a specific game mechanic requires a narrower term.

## Onboarding Quiz

During first setup, a developer can answer a short onboarding quiz:

- Runtime requirements and supported integration options.
- Game mechanics: base game, bonuses and supported modes.
- Target platforms and client expectations.

The answers can generate a starter configuration and point the developer to the relevant SDK sections.

## Integration Flow

1. Create or join a studio in Playtagon.
2. Create a game with the [[Types Play SDK|Play SDK creator type]].
3. Build and test game logic locally.
4. Package the game as a `.playtagon` archive.
5. Upload the archive in [[Resources]].
6. Wait for archive build and validation status.
7. Test the game in [[Launch Game]].
8. Prepare [[Promo Assets]].
9. Submit the selected archive version through [[Publishing]].

Standalone math simulation is not part of the external MVP app flow. Required checks are handled through archive validation and publishing.

## `.playtagon` Archive

Playtagon uses a branded archive format named `.playtagon`. This is the expected upload artifact for the Play SDK path.

```text
game.playtagon
├── manifest.json     # metadata, versions and runtime configuration
├── math/             # game math, rounds, RTP and mechanics
├── assets/           # static assets when needed
└── config/           # permissions and mechanics configuration
```

`manifest.json` is required. Without it, the archive cannot be accepted.

After upload, the archive appears in [[Resources]], where Playtagon builds it and returns status.

## CLI

```bash
# Sign in
playtagon login

# Upload a game archive
playtagon upload ./game.playtagon --studio my-studio --game my-game

# Check upload or validation status
playtagon status --game my-game
```

The CLI is useful for CI/CD pipelines when a team wants archive upload to be part of an automated build.

## Custom Scenarios

The SDK can expose custom scenarios for testing. A scenario is a predefined game event sequence, such as forcing a bonus trigger or a specific round outcome.

1. The developer defines scenarios in SDK code.
2. The uploaded archive makes those scenarios available to the launch runtime.
3. [[Launch Game]] lets the developer choose a scenario when supported.
4. The launch runtime executes the scenario and renders the result in the game iframe.

Scenario selection belongs to the launch runtime. It should be treated as a testing feature, not live game behavior.

## Runtime Statistics

When an archive is available, Playtagon can show runtime information for the game workspace, such as:

- historical round data;
- payout data;
- technical metrics;
- build and validation state.

The external MVP docs should describe only the information visible to studios, not internal runtime operations.

## Related Pages

- [[Resources]]: `.playtagon` archive upload.
- [[Launch Game]]: test stand and custom scenarios.
- [[Publishing]]: selected archive version goes through review.
- [[AI Wizard]]: SDK support inside Playtagon.
