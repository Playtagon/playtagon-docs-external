---
title: "Invite Email"
description: "Email-bound invitation flow for joining a studio."
tags:
  - "studio"
  - "auth"
  - "mvp"
category: "playtagon-app"
---
Invite Email is used when a studio invites a specific person. The invitation is bound to the recipient email and opens a token-based entry flow.

## Flow

1. An owner or permitted member sends an invitation from [[Members Invite]].
2. The recipient opens the email link.
3. Playtagon verifies the token and email.
4. The user signs in or confirms identity.
5. The user lands in the invited studio with the assigned access.

Invite Email is the safest MVP path for named collaborators because permissions are prepared before the user joins.
