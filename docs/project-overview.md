# Project overview

DJ QR Request Box is a small workflow-based setup for collecting song requests during an event.

The goal is simple: guests scan a QR code, send a song request, and the DJ gets a clean list with Spotify matches. The DJ still makes the final call. Nothing is added to the playlist just because a guest typed it.

## Assumptions

- The event does not need a custom app.
- Google Form is enough for guests.
- Google Sheet is enough for the DJ panel.
- n8n is enough for automation.
- The DJ should approve tracks manually.
- Secrets stay in n8n credentials, not in exported JSON files.

## Why Google Form and Google Sheets

For this kind of project, building a web app first would be overkill.

Google Form gives:

- an easy QR target,
- mobile-friendly input,
- no login requirement if configured that way,
- automatic rows in Google Sheets.

Google Sheets gives:

- a simple working queue,
- manual control for the DJ,
- easy filtering,
- a visible audit trail of what happened.

n8n sits in the middle and does the repetitive work:

- reads new rows,
- normalizes messy titles,
- searches Spotify,
- writes results back,
- sends Telegram notifications,
- adds approved tracks to a playlist.

## What this project is not

It is not a full booking system, CRM, event platform, or polished SaaS product.

It is a practical event helper. The point is to work reliably enough during a live event without needing a backend server.

## Current limitations

- Google Sheets is the main UI, so it can get messy if many people edit it.
- Spotify matching is useful, but not perfect.
- OpenAI normalization helps, but bad input can still produce bad matches.
- Telegram is a notification, not an approval interface.
- `akcja_dj` is currently handled manually in the sheet.

That is fine for MVP. The project is intentionally simple.
