# n8n setup

This project uses two n8n workflows:

- `workflows/01-process-new-request.json`
- `workflows/02-add-approved-to-spotify.json`

Both run on a 1 minute schedule. There is no separate backend. Google Sheets is the source of truth.

## Import

1. Open n8n.
2. Go to `Workflows`.
3. Import `workflows/01-process-new-request.json`.
4. Import `workflows/02-add-approved-to-spotify.json`.
5. Keep both workflows `Inactive` until credentials and placeholders are connected.

## Placeholders to replace

After import, replace:

- `__GOOGLE_SPREADSHEET_ID__`
- `__REQUESTS_SHEET_NAME__`
- `__GOOGLE_SHEETS_CREDENTIAL_NAME__`
- `__OPENAI_CREDENTIAL_NAME__`
- `__SPOTIFY_CREDENTIAL_NAME__`
- `__SPOTIFY_HTTP_OAUTH_CREDENTIAL_NAME__`
- `__SPOTIFY_PLAYLIST_ID__`
- `__TELEGRAM_CREDENTIAL_NAME__`
- `__TELEGRAM_CHAT_ID__`

`__GOOGLE_SPREADSHEET_ID__` comes from the sheet URL:

`https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`

`__REQUESTS_SHEET_NAME__` is the tab name, for example `Requests`.

`__SPOTIFY_PLAYLIST_ID__` comes from the playlist URL:

`https://open.spotify.com/playlist/PLAYLIST_ID`

## Required credentials

### Google Sheets

Credential placeholder:

`__GOOGLE_SHEETS_CREDENTIAL_NAME__`

Used in both workflows for:

- reading rows,
- updating rows by the `Sygnatura czasowa` column.

The Google account must have access to the response sheet.

### OpenAI

Credential placeholder:

`__OPENAI_CREDENTIAL_NAME__`

Used in workflow 01 by:

`OpenAI - normalize request to JSON`

This node returns:

- `AI_title`
- `AI_artist`
- `spotify_query`

### Spotify search

Credential placeholder:

`__SPOTIFY_CREDENTIAL_NAME__`

Used in workflow 01 by the native Spotify node:

`Spotify - search tracks by query`

This credential is only for Spotify search in workflow 01.

### Spotify add-to-playlist over HTTP Request

Credential placeholder:

`__SPOTIFY_HTTP_OAUTH_CREDENTIAL_NAME__`

Used in workflow 02 by:

`HTTP Request - add track to Spotify playlist`

Configure it as a custom OAuth2 credential for Spotify with:

- Authorization URL: `https://accounts.spotify.com/authorize`
- Access Token URL: `https://accounts.spotify.com/api/token`
- Scope: `playlist-modify-public playlist-modify-private`

The HTTP Request node sends:

- Method: `POST`
- URL: `https://api.spotify.com/v1/playlists/__SPOTIFY_PLAYLIST_ID__/items`
- Body:

```json
{
  "uris": ["{{ $json.spotify_uri }}"]
}
```

Workflow 02 uses HTTP Request on purpose instead of the native Spotify playlist node.

### Telegram

Credential placeholder:

`__TELEGRAM_CREDENTIAL_NAME__`

Chat placeholder:

`__TELEGRAM_CHAT_ID__`

Workflow 01 sends Telegram only after Google Sheets has already been updated with `status = FOUND`.

Node name:

`Telegram - notify DJ after FOUND`

The message includes the matched track, artist, Spotify link, confidence, and a reminder to set `akcja_dj` to `APPROVED` or `SKIP` in the sheet.

## Google Sheet columns

Form columns:

- `Sygnatura czasowa`
- `Tytuł utworu`
- `Wykonawca`
- `Imię / stolik`
- `Komentarz`

System columns:

- `status`
- `AI_title`
- `AI_artist`
- `spotify_query`
- `spotify_track_name`
- `spotify_artist`
- `spotify_url`
- `spotify_uri`
- `confidence`
- `dj_note`

Manual DJ column:

- `akcja_dj`

Recommended allowed values for `akcja_dj`:

- `APPROVED`
- `SKIP`

`status` is a system column. The DJ should not change it manually during normal use.

## Workflow 01

`01-process-new-request.json`

What it does:

1. Reads the sheet every minute.
2. Processes only rows where `status` is empty.
3. Maps form fields into internal workflow fields.
4. If title is missing, sets `status = NOT_FOUND`.
5. If title exists, normalizes the request through OpenAI.
6. Searches Spotify.
7. Scores the best result.
8. Sets `FOUND` or `NOT_FOUND`.
9. After `FOUND`, sends a Telegram notification to the DJ.

Row update is matched by:

`Sygnatura czasowa`

## Workflow 02

`02-add-approved-to-spotify.json`

What it does:

1. Reads the sheet every minute.
2. Processes only rows where:
   - `status = FOUND`
   - `akcja_dj = APPROVED`
3. Checks `spotify_uri`.
4. Adds the track to the playlist through Spotify API using HTTP Request.
5. On success it sets:
   - `status = ADDED`
   - `dj_note = Added to Spotify playlist`
   - clears `akcja_dj`
6. On error it sets:
   - `status = ERROR`
   - `dj_note` to a readable error message
   - clears `akcja_dj`

Workflow 02 does not react to `SKIP`. `SKIP` is simply the DJ deciding not to send that request to the playlist.

## How the system is used

1. A guest sends a request through Google Form.
2. Workflow 01 checks the sheet and sets `FOUND` or `NOT_FOUND`.
3. If the result is `FOUND`, the DJ gets a Telegram message.
4. The DJ reviews the row and sets `akcja_dj` to `APPROVED` or `SKIP`.
5. Workflow 02 picks approved rows and adds the track to the playlist.
6. The row ends up as `ADDED` or `ERROR`.

## Manual test

1. Send a test request through Google Form.
2. Wait for workflow 01 to run.
3. In the sheet you should see:
   - `status = FOUND`
   - `spotify_uri`
   - Spotify fields filled in
4. The DJ should receive a Telegram message.
5. In the sheet set:
   - `akcja_dj = APPROVED`
6. Run workflow 02 manually or wait a minute.
7. On success you should see:
   - `status = ADDED`
   - `akcja_dj` cleared
   - `dj_note = Added to Spotify playlist`

Skip test:

1. After `FOUND`, set `akcja_dj = SKIP`.
2. Workflow 02 should not add anything to the playlist.
3. `status` should stay `FOUND`.

## How to verify row updates hit the right row

Both workflows update Google Sheets by matching on:

`Sygnatura czasowa`

If an update lands on the wrong row, check:

- that `Sygnatura czasowa` exists in the sheet,
- that the values are unique enough for your form responses,
- that the Google Sheets update nodes are still configured to match this exact column.

## Important

- Do not put tokens into workflow JSON files.
- Do not commit real client secrets to the repo.
- Do not manually change `status` unless you are debugging.
- Use only `akcja_dj` for the DJ decision.
- If row updates hit the wrong row, start by checking `Sygnatura czasowa`.
