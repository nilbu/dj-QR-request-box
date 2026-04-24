# Google Sheet setup

The sheet is the main working panel.

Required columns:

- `Sygnatura czasowa`
- `Tytuł utworu`
- `Wykonawca`
- `Imię / stolik`
- `Komentarz`
- `status`
- `akcja_dj`
- `AI_title`
- `AI_artist`
- `spotify_query`
- `spotify_track_name`
- `spotify_artist`
- `spotify_url`
- `spotify_uri`
- `confidence`
- `dj_note`

Recommended setup:

- freeze the header row,
- add filters,
- add data validation for `akcja_dj`:
  - `APPROVED`
  - `SKIP`
- avoid editing `status` manually.

`status` is for workflows. `akcja_dj` is for the DJ.
