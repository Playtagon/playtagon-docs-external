---
title: "Google OAuth"
description: "Google-based sign-in for invited and existing users."
tags:
  - "studio"
  - "auth"
  - "mvp"
category: "playtagon-app"
---
Google OAuth is the social sign-in option for Playtagon users.

## MVP Rules

- The Google account email must match the expected user identity for invite-based entry.
- Existing users can use Google OAuth as a returning sign-in method.
- If the user belongs to several studios, Playtagon shows [[Auth Multi Studio]].
- If the user has no access yet, Playtagon routes them to [[Auth Waitlist]].
