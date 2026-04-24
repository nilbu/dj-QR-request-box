<<<<<<< Updated upstream
# DJ Request Box — n8n MVP

Budujemy system DJ requestów bez własnego backendu.

Architektura:
Google Form → Google Sheets → n8n → AI normalizacja → Spotify Search → Google Sheets update → ręczna akceptacja DJ-a → Spotify playlist.

Nie twórz aplikacji Next.js na tym etapie.
Nie dodawaj automatycznie piosenek bez statusu APPROVED.
Workflow ma być eksportowalny jako JSON do importu w n8n.
Sekrety, tokeny i credentials nie mogą być wpisane w JSON.

Wymagane workflow:
1. `01-process-new-request.json`
- wykrywa nowy wiersz w Google Sheet
- czyta tytuł, wykonawcę, imię, dedykację
- normalizuje dane przez AI
- wyszukuje utwór w Spotify
- aktualizuje wiersz o spotify_url, spotify_uri, confidence, status=FOUND

2. `02-add-approved-to-spotify.json`
- cyklicznie sprawdza wiersze ze statusem APPROVED
- dodaje spotify_uri do dedykowanej playlisty Spotify
- ustawia status=ADDED

Wymagane dokumenty:
- docs/setup-google-form.md
- docs/setup-google-sheet.md
- docs/setup-n8n.md
- docs/setup-spotify.md
- docs/operating-live-event.md
=======
Budujemy system DJ requestów bez własnego backendu.

Architektura: Google Form → Google Sheets → n8n → AI normalizacja → Spotify Search → Google Sheets update → ręczna akceptacja DJ-a → Spotify playlist.

Nie twórz aplikacji Next.js na tym etapie. Nie dodawaj automatycznie piosenek bez statusu APPROVED. Workflow ma być eksportowalny jako JSON do importu w n8n. Sekrety, tokeny i credentials nie mogą być wpisane w JSON.

Wymagane workflow:

01-process-new-request.json
wykrywa nowy wiersz w Google Sheet
czyta tytuł, wykonawcę, imię, dedykację
normalizuje dane przez AI
wyszukuje utwór w Spotify
aktualizuje wiersz o spotify_url, spotify_uri, confidence, status=FOUND
02-add-approved-to-spotify.json
cyklicznie sprawdza wiersze ze statusem APPROVED
dodaje spotify_uri do dedykowanej playlisty Spotify
ustawia status=ADDED
Wymagane dokumenty:

docs/setup-google-form.md
docs/setup-google-sheet.md
docs/setup-n8n.md
docs/setup-spotify.md
docs/operating-live-event.md
>>>>>>> Stashed changes
