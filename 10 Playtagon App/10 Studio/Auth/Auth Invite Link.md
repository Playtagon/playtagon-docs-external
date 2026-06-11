---
title: "Invite Link"
description: "Reusable studio invite link with default access settings."
tags:
  - "studio"
  - "auth"
  - "mvp"
category: "playtagon-app"
---
Invite Link lets a studio invite collaborators through a shared link. It is useful when the exact user list is not final yet, but the studio still wants a controlled default access level.

## MVP Behavior

- The link is created from [[Members Invite]].
- The studio defines default access before sharing it.
- A reset invalidates the previous link.
- The joining user still needs to authenticate through [[Auth Magic Link]] or [[Auth Google OAuth]].

For sensitive projects, prefer [[Auth Invite Email]] because it is bound to a known email address.
