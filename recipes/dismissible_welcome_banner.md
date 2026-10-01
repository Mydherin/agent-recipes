---
name: Dismissible welcome banner
description: Show a friendly welcome banner on the main screen that stays hidden once the user dismisses it.
---

# Dismissible welcome banner

## Goal

Greet first-time users with a short message and a close action, without bothering them again.

## Requirements

### Functional

- A banner at the top of the main screen with a title, one sentence and a close button.
- Closing hides it immediately and permanently on that device.
- Copy: title "Welcome!", text "Glad to have you here. Take a look around."

### Non-functional

- Follows the project's existing design system: colors, spacing, typography and icons.
- Accessible: `role="status"`, close button with an accessible label, keyboard reachable.
- Responsive: readable on mobile and desktop, never causes horizontal scroll.
- Dismissal persisted in client storage under the key `welcome-banner-dismissed`; storage failures fail open (banner shown) without errors.

## Approach

One small self-contained UI component plus a tiny persistence helper. Render it in the main screen layout above the content.

## Steps

1. Create the banner component using existing UI primitives.
2. Read the dismissal flag on mount; render nothing when set.
3. On close, set the flag and hide the banner.
4. Mount it at the top of the main screen.

## Acceptance criteria

- The banner shows on first visit.
- After closing it and reloading, it stays hidden.
- Clearing site data shows it again.
