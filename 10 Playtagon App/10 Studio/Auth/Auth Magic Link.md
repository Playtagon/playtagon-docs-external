---
title: "Magic Link"
description: "Passwordless email sign-in for Playtagon users."
tags:
  - "studio"
  - "auth"
  - "mvp"
category: "playtagon-app"
---
Magic Link is the email-based sign-in method for users who do not use Google OAuth.

## Flow

1. The user enters an email address.
2. Playtagon sends a time-limited sign-in link.
3. The user opens the link in the same browser or device.
4. Playtagon resolves the user's access state and sends them to the correct next screen.

Magic Link can complete an invite flow or return an existing user to their studio.
