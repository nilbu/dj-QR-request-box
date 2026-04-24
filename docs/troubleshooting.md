# Troubleshooting

Common problems and quick fixes.

## Spotify: insufficient scope

Symptom:

- workflow 02 sets `ERROR`,
- `dj_note` mentions permissions, scope, or authorization.

Fix:

- check custom OAuth2 credential for Spotify HTTP Request,
- use scopes:
  - `playlist-modify-public`
  - `playlist-modify-private`
- reconnect the credential after changing scopes.

## Wrong playlist ID

Symptom:

- Spotify API returns 404 or similar,
- workflow 02 cannot add tracks.

Fix:

- open the playlist in Spotify,
- copy the playlist ID from:
  `https://open.spotify.com/playlist/PLAYLIST_ID`
- paste that value into `__SPOTIFY_PLAYLIST_ID__`.

## Workflow 01 finds nothing

Symptom:

- many rows become `NOT_FOUND`.

Fix:

- check that `Tytuł utworu` is mapped correctly,
- check OpenAI output fields:
  - `AI_title`
  - `AI_artist`
  - `spotify_query`
- check Spotify credential,
- try a simpler title in the form.

## Request goes to NOT_FOUND

Possible reasons:

- typo in the song title,
- artist missing,
- title is too vague,
- Spotify result is a cover, karaoke version, or bad match,
- confidence is too low.

Fix:

- search Spotify manually,
- paste the correct `spotify_uri`,
- set `status = FOUND`,
- set `akcja_dj = APPROVED` if you want to add it.

## Telegram does not send messages

Symptom:

- `FOUND` rows appear, but no Telegram message arrives.

Fix:

- check `__TELEGRAM_CREDENTIAL_NAME__`,
- check `__TELEGRAM_CHAT_ID__`,
- make sure the bot can message the chat,
- run the Telegram node manually in n8n,
- check whether Telegram node failed but workflow continued.

The workflow sends Telegram after the Google Sheet row is updated to `FOUND`.

## Google Sheets credentials problem

Symptom:

- workflows cannot read rows,
- updates do not happen,
- n8n shows Google authorization errors.

Fix:

- reconnect Google Sheets credential,
- make sure the credential account has access to the spreadsheet,
- check `__GOOGLE_SPREADSHEET_ID__`,
- check `__REQUESTS_SHEET_NAME__`.

## Custom OAuth2 for Spotify does not work

Check:

- Authorization URL:
  `https://accounts.spotify.com/authorize`
- Access Token URL:
  `https://accounts.spotify.com/api/token`
- Scope:
  `playlist-modify-public playlist-modify-private`
- callback URL configured in Spotify developer dashboard,
- client ID and client secret are stored only in n8n credentials.

## Typos and phonetic requests

Guests will type songs badly. That is normal.

Examples:

- partial title,
- wrong artist,
- Polish spelling of an English title,
- song from TikTok described by chorus only.

OpenAI helps normalize this, but it will not catch everything.

Practical fix:

- keep `spotify_url` visible,
- use `NOT_FOUND` as a review bucket,
- manually paste `spotify_uri` for important requests.

## Workflow 02 does not add a track

Check the row:

- `status` must be `FOUND`,
- `akcja_dj` must be `APPROVED`,
- `spotify_uri` must not be empty.

If `akcja_dj = SKIP`, workflow 02 ignores it.

## Update hits the wrong row

The workflows update rows by:

`Sygnatura czasowa`

Check:

- the column exists,
- the header was not renamed,
- timestamps are unique,
- update nodes use `matchingColumns = Sygnatura czasowa`.
