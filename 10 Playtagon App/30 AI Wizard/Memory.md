---
title: "Wizard Memory"
description: "Context used by the AI Wizard to answer consistently."
tags:
  - "ai-wizard"
  - "mvp"
category: "playtagon-app"
---
Wizard Memory is the context the assistant can use to stay helpful.

## MVP Context

- Active studio and game.
- Relevant archive or submission state.
- Studio-level assistant preferences from [[Settings AI Wizard]].
- Recent conversation context.

Memory should be scoped so the Wizard gives useful answers without exposing private or unrelated information.
