# DJ QR Request Box

A small DIY system for collecting song requests during an event.

The idea is simple: guests scan a QR code, send a request through Google Form, the request lands in Google Sheets, n8n looks up the track in Spotify, and the DJ decides what to do with it. No custom backend, no separate app, no fake complexity.

## Why this exists

This project came from a practical need: during an event, song requests tend to arrive as paper notes, shouted titles, Messenger messages, or random conversations near the booth. That gets messy fast.

This setup gives the DJ one shared queue, keeps a record of what came in, and still leaves the final decision in human hands.

## How it works

```text
Google Form
  -> Google Sheets
  -> n8n workflow 01
  -> OpenAI normalization
  -> Spotify search
  -> Google Sheets update
  -> Telegram notification to DJ
  -> manual decision in akcja_dj
  -> n8n workflow 02
  -> Spotify playlist
```

## Stack

- Google Form
- Google Sheets
- n8n
- OpenAI
- Spotify Web API
- Telegram Bot

## Project status

Current MVP works as two importable n8n workflows:

- `workflows/01-process-new-request.json`
- `workflows/02-add-approved-to-spotify.json`

There is no separate web app at this stage. Google Sheets is the DJ panel.

## Request flow

1. A guest sends a request through Google Form.
2. The response lands in Google Sheets.
3. Workflow 01 picks rows where `status` is empty.
4. OpenAI normalizes the request.
5. Spotify search tries to find a useful match.
6. The sheet is updated with `FOUND` or `NOT_FOUND`.
7. When the result is `FOUND`, the DJ gets a Telegram message.
8. The DJ sets `akcja_dj` in the sheet:
   - `APPROVED`
   - `SKIP`
9. Workflow 02 picks only rows where `status = FOUND` and `akcja_dj = APPROVED`.
10. The track is added to the Spotify playlist.
11. The sheet is updated to `ADDED` or `ERROR`.

## Workflow 01

`01-process-new-request.json`

This workflow:

- runs every minute,
- reads the Google Sheet,
- processes only rows with empty `status`,
- normalizes the request with OpenAI,
- searches Spotify,
- fills Spotify fields back into the sheet,
- sets `FOUND` or `NOT_FOUND`,
- sends a Telegram notification after a successful `FOUND` update.

## Workflow 02

`02-add-approved-to-spotify.json`

This workflow:

- runs every minute,
- reads the Google Sheet,
- processes only rows with `status = FOUND` and `akcja_dj = APPROVED`,
- adds `spotify_uri` to the playlist through Spotify API,
- sets `ADDED` or `ERROR`,
- clears `akcja_dj` after processing.

## Telegram

Telegram is only a notification channel for the DJ. It does not approve or reject anything.

The message is sent only after workflow 01 successfully writes `FOUND` to the sheet. The actual decision still happens in Google Sheets through the `akcja_dj` column.

## Setup

The main setup steps are documented here:

- [docs/setup-n8n.md](docs/setup-n8n.md)
- [docs/architecture.md](docs/architecture.md)
- [docs/operating-live-event.md](docs/operating-live-event.md)
- [docs/troubleshooting.md](docs/troubleshooting.md)

In short:

1. Create the Google Form.
2. Connect form responses to Google Sheets.
3. Add the system columns described in the docs.
4. Import both workflows into n8n.
5. Connect credentials.
6. Set the playlist ID.
7. Send a test request.

## What next

Ideas for later:

- better Spotify scoring,
- a separate DJ view instead of working directly in the sheet,
- duplicate protection for the playlist,
- event history,
- a simple filter for obvious junk requests.

## Practical notes

- `status` is a system column. The DJ should not edit it in normal use.
- `akcja_dj` is the manual DJ decision column. Use `APPROVED` or `SKIP`.
- `Sygnatura czasowa` is the row match key for Google Sheets updates.
- Do not put tokens or secrets into workflow JSON files.
- Before a live event, test everything on a separate sheet.
