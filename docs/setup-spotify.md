# Spotify setup

Spotify is used in two places:

1. workflow 01 searches for tracks,
2. workflow 02 adds approved tracks to a playlist.

## Playlist

Create a playlist for the event and copy its ID from:

`https://open.spotify.com/playlist/PLAYLIST_ID`

Use that value as:

`__SPOTIFY_PLAYLIST_ID__`

## Search credential

Workflow 01 uses:

`__SPOTIFY_CREDENTIAL_NAME__`

This can be the native Spotify credential in n8n.

## Add-to-playlist credential

Workflow 02 uses HTTP Request with custom OAuth2:

`__SPOTIFY_HTTP_OAUTH_CREDENTIAL_NAME__`

OAuth2 settings:

- Authorization URL: `https://accounts.spotify.com/authorize`
- Access Token URL: `https://accounts.spotify.com/api/token`
- Scope: `playlist-modify-public playlist-modify-private`

Store client ID and client secret only in n8n credentials.
