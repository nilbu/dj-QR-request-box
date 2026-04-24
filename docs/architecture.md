# Architecture

The system is built from a few boring pieces that work well together:

```text
Guest
  -> Google Form
  -> Google Sheets
  -> n8n workflow 01
  -> OpenAI
  -> Spotify Search
  -> Google Sheets
  -> Telegram
  -> DJ sets akcja_dj
  -> n8n workflow 02
  -> Spotify Playlist
```

## Google Sheet as the main state

The sheet is both database and DJ panel.

Rows are matched by:

`Sygnatura czasowa`

This column comes from Google Form and is used by n8n updates. The workflows do not use `row_number` as the match key.

## Columns

### `Sygnatura czasowa`

Timestamp from Google Form. Used as the row identifier for updates.

### `Tytuł utworu`

Raw song title from the guest.

### `Wykonawca`

Raw artist name from the guest. Optional, but useful.

### `Imię / stolik`

Guest name, table number, or any short identifier.

### `Komentarz`

Optional comment or dedication.

### `status`

System column. The DJ should not edit it during normal use.

Typical values:

- empty: new row, not processed yet
- `FOUND`: Spotify match found
- `NOT_FOUND`: no useful match
- `ADDED`: added to playlist
- `ERROR`: something failed

### `akcja_dj`

Manual DJ decision.

Allowed values:

- `APPROVED`
- `SKIP`

Workflow 02 only reacts to `akcja_dj = APPROVED` and `status = FOUND`.

### `AI_title`

Title normalized by OpenAI.

### `AI_artist`

Artist normalized by OpenAI.

### `spotify_query`

Search query generated from normalized data.

### `spotify_track_name`

Track name returned from Spotify.

### `spotify_artist`

Artist returned from Spotify.

### `spotify_url`

Public Spotify URL for quick DJ preview.

### `spotify_uri`

Spotify track URI used by workflow 02 when adding to the playlist.

### `confidence`

Simple score from workflow 01. It is not magic, just a practical confidence number.

### `dj_note`

Human-readable note from the workflows. Useful for errors and quick context.

## Workflow 01

`workflows/01-process-new-request.json`

Main job:

- find new rows,
- normalize request,
- search Spotify,
- update the sheet,
- notify DJ on Telegram when a track is found.

It processes only rows where `status` is empty.

On success:

- writes Spotify fields,
- sets `status = FOUND`,
- sends Telegram notification.

On failure:

- sets `status = NOT_FOUND`,
- writes a note to `dj_note`.

## Workflow 02

`workflows/02-add-approved-to-spotify.json`

Main job:

- find rows approved by the DJ,
- add the Spotify URI to a playlist,
- update the sheet.

It processes only rows where:

- `status = FOUND`
- `akcja_dj = APPROVED`
- `spotify_uri` is not empty

Spotify playlist adding is done with HTTP Request and custom OAuth2 credentials.

On success:

- `status = ADDED`
- `dj_note = Added to Spotify playlist`
- `akcja_dj` is cleared

On failure:

- `status = ERROR`
- `dj_note` gets the error message
- `akcja_dj` is cleared
