# n8n MVP implementation plan

> Note: this file keeps the original MVP planning view. The current implementation status is documented in `README.md`, `docs/architecture.md`, and `docs/setup-n8n.md`.

## MVP goal

The MVP should handle music requests from guests through Google Form, store them in Google Sheets, normalize and search them through n8n, and then leave the final approval to the DJ before anything is added to a Spotify playlist.

Planned scope:

- Google Form as the public QR entry point
- Google Sheets as the operational queue and DJ panel
- n8n as the automation layer
- AI used only for request normalization
- Spotify search used to find the best match
- Manual DJ decision before adding anything to the playlist

Out of scope for the MVP:

- a custom backend,
- a standalone web app,
- fully automatic playlist adding with no DJ approval,
- storing secrets directly in exported n8n JSON files.

## 1. Google Form structure

The form should be short, mobile-friendly, and tolerant of incomplete guest input.

### Form name

`DJ Request Box`

### Form description

Keep it short and clear:

- ask for the song,
- explain that the DJ decides what fits the moment,
- make it clear that a request does not guarantee playback.

Example:

`Send a track you want to hear. The DJ decides what fits the moment.`

### Suggested form fields

1. `Song title`
   - Type: Short answer
   - Required: yes
   - Purpose: raw track title from the guest

2. `Artist`
   - Type: Short answer
   - Required: no
   - Purpose: raw artist name if the guest knows it

3. `Your name`
   - Type: Short answer
   - Required: no
   - Purpose: guest name or identifier

4. `Dedication`
   - Type: Paragraph
   - Required: no
   - Purpose: short comment or dedication for the DJ

5. `Consent / note`
   - Type: Checkbox
   - Required: yes
   - Value: `I understand this request may be moderated by the DJ.`

### Form settings

- Do not require Google login.
- Do not collect email addresses.
- Keep it to one page.
- Send responses into Google Sheets.
- Show a simple confirmation after submit:
  `Thanks! Your request was sent to the DJ.`

## 2. Google Sheet structure

Google Sheets plays three roles here:

- request database,
- DJ work queue,
- current workflow state.

### Main sheet

Suggested sheet name:

`Requests`

### Optional config sheet

Suggested sheet name:

`Config`

This can hold non-secret operational values such as playlist label, event name, or soft limits. Tokens and secrets should not live in the sheet.

### Useful DJ filters

Helpful filtered views:

- `Found`: rows with `status = FOUND`
- `Approved`: rows approved by the DJ
- `Added`: rows with `status = ADDED`
- `Not found`: rows with `status = NOT_FOUND`
- `Errors`: rows with `status = ERROR`

### Sheet protection

- Freeze the header row.
- Protect system columns from accidental edits.
- Leave only the intended manual columns editable for the DJ.

## 3. Required Google Sheet columns

Column names should stay stable because n8n maps by header names.

### Form columns

1. `Sygnatura czasowa`
   - Source: Google Form
   - Meaning: submission timestamp and row match key

2. `Tytuł utworu`
   - Source: Google Form
   - Meaning: raw song title from the guest

3. `Wykonawca`
   - Source: Google Form
   - Meaning: raw artist name from the guest

4. `Imię / stolik`
   - Source: Google Form
   - Meaning: guest name, table number, or other short identifier

5. `Komentarz`
   - Source: Google Form
   - Meaning: optional comment or dedication

### System columns

6. `status`
   - System state of the request

7. `akcja_dj`
   - Manual DJ decision

8. `AI_title`
   - Title normalized by OpenAI

9. `AI_artist`
   - Artist normalized by OpenAI

10. `spotify_query`
    - Search query produced from normalized data

11. `spotify_track_name`
    - Best Spotify track name found

12. `spotify_artist`
    - Best Spotify artist found

13. `spotify_url`
    - Public Spotify URL

14. `spotify_uri`
    - Spotify URI used when adding to playlist

15. `confidence`
    - Practical confidence score from workflow 01

16. `dj_note`
    - Human-readable status note or error note

### Minimum status set

- empty: new row waiting for processing
- `FOUND`: useful Spotify match found
- `NOT_FOUND`: no useful match found
- `ADDED`: track added to playlist
- `ERROR`: technical problem during processing

### Manual DJ actions

- `APPROVED`
- `SKIP`

## 4. Required credentials in n8n

Credentials should be created in n8n and referenced by workflow nodes. Do not hardcode secrets into exported workflow JSON.

### Google Sheets

Used for:

- reading rows from `Requests`,
- updating rows by `Sygnatura czasowa`.

### OpenAI

Used for:

- cleaning the raw request,
- generating `AI_title`,
- generating `AI_artist`,
- generating `spotify_query`.

### Spotify search credential

Used in workflow 01 for native Spotify search nodes.

### Spotify HTTP OAuth2 credential

Used in workflow 02 for adding tracks to the playlist through HTTP Request.

Required Spotify scopes:

- `playlist-modify-public`
- `playlist-modify-private`

### Telegram

Used in workflow 01 for the DJ notification after a successful `FOUND` update.

## 5. Workflow outline

Exported workflow files:

- `workflows/01-process-new-request.json`
- `workflows/02-add-approved-to-spotify.json`

### Workflow 01: process new requests

Goal:

- read new form rows from Google Sheets,
- normalize them,
- search Spotify,
- write results back,
- notify the DJ when a match is found.

Core logic:

1. Run on a schedule.
2. Read rows from the sheet.
3. Filter only rows where `status` is empty.
4. Map form fields into internal workflow fields.
5. If title is missing, write `NOT_FOUND` and a note.
6. Otherwise normalize with OpenAI.
7. Search Spotify.
8. Score the result.
9. If good enough, write `FOUND` and Spotify fields.
10. If not good enough, write `NOT_FOUND`.
11. After a successful `FOUND` sheet update, send Telegram to the DJ.

### Workflow 02: add approved tracks to Spotify

Goal:

- find rows approved by the DJ,
- add their Spotify URI to the target playlist,
- write the result back to the sheet.

Core logic:

1. Run on a schedule.
2. Read rows from the sheet.
3. Filter only rows where:
   - `status = FOUND`
   - `akcja_dj = APPROVED`
4. Check `spotify_uri`.
5. If the URI is missing, write `ERROR`.
6. If the URI exists, call Spotify API through HTTP Request.
7. On success, write `ADDED`, set a note, and clear `akcja_dj`.
8. On failure, write `ERROR`, save a readable note, and clear `akcja_dj`.

## 6. Risks and safeguards

### Risk: wrong Spotify match

Safeguards:

- use a confidence score,
- show Spotify fields in the sheet,
- let the DJ decide through `akcja_dj`,
- keep `dj_note` readable.

### Risk: adding a track without DJ approval

Safeguards:

- workflow 02 only reacts to `akcja_dj = APPROVED`,
- workflow 01 never adds anything to the playlist,
- `status` stays system-controlled.

### Risk: duplicate playlist entries

Safeguards:

- use sheet history to spot repeats,
- optionally add duplicate checks later,
- keep the first version simple and observable.

### Risk: bad guest input

Safeguards:

- require the song title in the form,
- normalize the request with OpenAI,
- allow `NOT_FOUND` instead of forcing a bad match,
- let the DJ skip weak requests.

### Risk: row update hitting the wrong row

Safeguards:

- update by `Sygnatura czasowa`,
- keep that column stable,
- do not rely on `row_number`.

### Risk: leaking secrets

Safeguards:

- keep credentials only in n8n credentials storage,
- use placeholders in exported JSON,
- never commit tokens or client secrets to the repo.

## Suggested implementation order

1. Prepare the Google Form.
2. Prepare the Google Sheet with the required columns.
3. Configure Spotify credentials.
4. Configure n8n.
5. Import both workflows.
6. Test on a copy of the sheet.
7. Run through the main cases:
   - valid request,
   - missing artist,
   - no Spotify match,
   - DJ approval,
   - DJ skip,
   - playlist add success,
   - playlist add error.
