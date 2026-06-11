---
title: "Auth"
description: "Entry points for approved users, invited users and waitlist applicants."
tags:
  - "studio"
  - "auth"
  - "mvp"
category: "playtagon-app"
---
Auth is the entry layer for Playtagon. In MVP, a user either joins through an invitation, signs in with a magic link or Google OAuth, or applies through the waitlist.

## Supported Paths

| Path | Use case |
| --- | --- |
| [[Auth Invite Email]] | A studio owner invites a specific email address. |
| [[Auth Invite Link]] | A studio shares a reusable invite link. |
| [[Auth Magic Link]] | The user signs in by email without a password. |
| [[Auth Google OAuth]] | The user signs in with a Google account. |
| [[Auth Waitlist]] | A new user requests access before approval. |
| [[Auth Multi Studio]] | A user with several studios chooses the active workspace. |

Registration and sign-in are intentionally part of one flow: Playtagon decides the next screen from the user's access state.
