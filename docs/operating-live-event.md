# Operating a live event

This is the practical runbook for using the system during an event.

## Before the event

- Create or copy the Google Form.
- Check that responses go to the correct Google Sheet.
- Check that the sheet has all required columns.
- Set data validation for `akcja_dj`:
  - `APPROVED`
  - `SKIP`
- Import both n8n workflows.
- Connect credentials:
  - Google Sheets
  - OpenAI
  - Spotify search
  - Spotify HTTP OAuth2
  - Telegram
- Set `__SPOTIFY_PLAYLIST_ID__`.
- Set `__TELEGRAM_CHAT_ID__`.
- Send one test request.
- Confirm Telegram notification arrives after `FOUND`.
- Approve one test row and confirm it becomes `ADDED`.

## During the event

Use the Google Sheet as the working panel.

Suggested filters:

- `status = FOUND`
- `status = NOT_FOUND`
- `status = ERROR`
- `akcja_dj` empty
- `akcja_dj = SKIP`

The normal DJ flow:

1. Watch new `FOUND` rows.
2. Open `spotify_url` if unsure.
3. If the track fits, set `akcja_dj = APPROVED`.
4. If it does not fit, set `akcja_dj = SKIP`.
5. Let workflow 02 add approved tracks.

Do not manually change `status` unless you are debugging.

## After the event

- Stop or disable both workflows.
- Export or copy the final sheet if you want history.
- Clear credentials from any temporary n8n instance.
- Archive the Google Form if the QR code should stop working.
- Review `ERROR` rows if anything looked strange.

## If a request is wrong

If the guest typed nonsense or the request is incomplete:

- leave `NOT_FOUND` as-is,
- optionally write a note in `dj_note`,
- do not set `akcja_dj = APPROVED`.

## If Spotify finds nothing

For `NOT_FOUND`:

- search manually in Spotify,
- if you find the right track, paste its URI into `spotify_uri`,
- set `status = FOUND`,
- then set `akcja_dj = APPROVED` if you want workflow 02 to add it.

## If the request does not fit the party

Use:

`akcja_dj = SKIP`

That keeps the request visible but prevents workflow 02 from adding it.

No need to change `status`.

## If something breaks

Use `docs/troubleshooting.md`.

Most live-event issues are one of these:

- bad Spotify OAuth scope,
- wrong playlist ID,
- Google Sheet column renamed,
- Telegram credential or chat ID wrong,
- `status` edited manually by mistake.
