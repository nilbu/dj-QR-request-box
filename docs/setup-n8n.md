# Setup n8n dla DJ Request Box

Ten dokument opisuje import i test pierwszego workflow:

`workflows/01-process-new-request.json`

Workflow przetwarza nowy wiersz z Google Sheet, normalizuje request przez OpenAI, wyszukuje utwor w Spotify i aktualizuje ten sam wiersz statusem `FOUND` albo `NOT_FOUND`.

Workflow nie dodaje utworow do playlisty Spotify. Dodawanie do playlisty bedzie obslugiwane przez osobny workflow po recznym ustawieniu statusu `APPROVED`.

## 1. Import workflow

1. Otworz n8n.
2. Wejdz w `Workflows`.
3. Wybierz `Import from File`.
4. Wskaz plik:
   `workflows/01-process-new-request.json`
5. Po imporcie workflow zostaw jako `Inactive` do czasu ukonczenia konfiguracji.
6. Otworz workflow i sprawdz sticky notes po lewej stronie. Zawieraja liste placeholderow i opis logiki.

## 2. Placeholdery do podmiany

Po imporcie podmien placeholdery w node'ach Google Sheets oraz credentials.

### Google Sheets

Podmien:

- `__GOOGLE_SPREADSHEET_ID__`
- `__REQUESTS_SHEET_NAME__`

Gdzie:

- `__GOOGLE_SPREADSHEET_ID__` to ID arkusza z URL Google Sheet.
- `__REQUESTS_SHEET_NAME__` to nazwa zakladki, np. `Requests`.

ID arkusza jest fragmentem URL:

`https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`

Node'y do sprawdzenia:

- `Trigger - new Google Sheet row`
- `Google Sheets - set status NEW`
- `Google Sheets - update row FOUND`
- `Google Sheets - update row NOT_FOUND`
- `Google Sheets - reject missing title`

### Credentials placeholders

W JSON sa tylko nazwy placeholderow. Po imporcie n8n moze pokazac brakujace credentials. To jest oczekiwane.

Podmien:

- `__GOOGLE_SHEETS_CREDENTIAL_NAME__`
- `__OPENAI_CREDENTIAL_NAME__`
- `__SPOTIFY_CREDENTIAL_NAME__`

Placeholder `__SPOTIFY_PLAYLIST_ID__` jest obecny tylko jako przypomnienie dla drugiego workflow. Pierwszy workflow go nie uzywa.

## 3. Wymagane credentials

### Google Sheets OAuth2

Utworz lub wybierz credential:

`Google Sheets - DJ Request Box`

Uzyj go w node'ach:

- `Trigger - new Google Sheet row`
- wszystkie node'y zaczynajace sie od `Google Sheets - ...`

Credential musi miec dostep do arkusza z odpowiedziami Google Form.

### OpenAI

Utworz lub wybierz credential:

`OpenAI - DJ Request Box`

Uzyj go w node:

- `OpenAI - normalize title artist and query`

Ten node wywoluje endpoint OpenAI Chat Completions i oczekuje JSON z polami:

- `AI_title`
- `AI_artist`
- `spotify_query`

### Spotify OAuth2

Utworz lub wybierz credential:

`Spotify - DJ Request Box`

Uzyj go w node:

- `Spotify - search tracks`

Do samego wyszukiwania wystarczy credential pozwalajacy odpytywac Spotify Web API. Scope do modyfikacji playlisty bedzie potrzebny dopiero w workflow `02-add-approved-to-spotify.json`.

## 4. Wymagane kolumny w Google Sheet

Workflow zaklada, ze arkusz ma kolumny wejsciowe:

- `song_title_raw`
- `artist_raw`
- `requester_name`
- `dedication`
- `status`

Workflow umie tez odczytac kilka alternatywnych nazw z formularza, np. `Song title`, `Artist`, `Your name`, `Dedication`, ale rekomendowane sa nazwy techniczne powyzej.

Workflow zapisuje kolumny:

- `AI_title`
- `AI_artist`
- `spotify_query`
- `spotify_track_name`
- `spotify_artist`
- `spotify_url`
- `spotify_uri`
- `confidence`
- `status`
- `dj_note`

Wazne:

- Kolumna `status` moze byc pusta dla nowych odpowiedzi.
- Jesli `status` jest pusty, workflow ustawi `NEW`.
- Jesli utwor zostanie znaleziony z sensowna pewnoscia, workflow ustawi `FOUND`.
- Jesli wynik nie bedzie sensowny, workflow ustawi `NOT_FOUND` i wpisze powod w `dj_note`.

## 5. Jak przetestowac na jednym wierszu

1. Utworz testowy arkusz albo skopiuj produkcyjny arkusz.
2. Upewnij sie, ze workflow wskazuje testowy `__GOOGLE_SPREADSHEET_ID__`.
3. W zakladce `Requests` dodaj jeden testowy wiersz.
4. Wpisz minimalne dane:

| song_title_raw | artist_raw | requester_name | dedication | status |
| --- | --- | --- | --- | --- |
| September | Earth, Wind & Fire | Test table | Test request | |

5. W n8n otworz workflow.
6. Kliknij `Execute Workflow` albo wlacz workflow i poczekaj na polling triggera.
7. Sprawdz wykonanie krok po kroku:
   - `Prepare request data and default status NEW` powinien pokazac `status: NEW`.
   - `OpenAI - normalize title artist and query` powinien zwrocic JSON z `AI_title`, `AI_artist`, `spotify_query`.
   - `Spotify - search tracks` powinien zwrocic liste wynikow.
   - `Evaluate Spotify result and confidence` powinien ustawic `status` na `FOUND` albo `NOT_FOUND`.
8. W Google Sheet sprawdz, czy ten sam wiersz ma uzupelnione pola Spotify.

Oczekiwany wynik dla poprawnego testu:

- `AI_title` nie jest puste.
- `spotify_query` nie jest puste.
- `spotify_uri` jest wypelnione.
- `confidence` jest wieksze lub rowne `0.65`.
- `status = FOUND`.
- `dj_note` przypomina, ze DJ musi recznie zaakceptowac request przed dodaniem do playlisty.

## 6. Test negatywny

Dodaj drugi wiersz z losowym albo niepelny tytulem:

| song_title_raw | artist_raw | requester_name | dedication | status |
| --- | --- | --- | --- | --- |
| asdfgh impossible song | | Test table | Test request | |

Oczekiwany wynik:

- `status = NOT_FOUND`
- `spotify_uri` puste
- `dj_note` zawiera informacje, ze nie znaleziono sensownego wyniku

## 7. Zasady bezpieczenstwa

- Nie wpisuj tokenow ani API keys bezposrednio w workflow JSON.
- Nie aktywuj workflow na produkcyjnym arkuszu przed testem na kopii.
- Nie zmieniaj statusu na `APPROVED` automatycznie w tym workflow.
- Ten workflow konczy prace na `FOUND` albo `NOT_FOUND`.
- Dodawanie do Spotify playlisty moze zrobic dopiero workflow `02-add-approved-to-spotify.json`.

## 8. Typowe problemy

### n8n nie widzi arkusza

Sprawdz:

- czy credential Google ma dostep do pliku,
- czy spreadsheet ID jest poprawny,
- czy nazwa zakladki zgadza sie dokladnie z `__REQUESTS_SHEET_NAME__`.

### Update row nie dziala

Sprawdz:

- czy trigger zwraca `row_number`,
- czy arkusz ma naglowki w pierwszym wierszu,
- czy nazwy kolumn docelowych istnieja w arkuszu.

### OpenAI node zwraca blad

Sprawdz:

- czy credential OpenAI jest podpiety,
- czy konto ma dostep do modelu ustawionego w node,
- czy endpoint Chat Completions jest dostepny w Twojej konfiguracji n8n.

### Spotify nie zwraca wynikow

Sprawdz:

- czy credential Spotify jest podpiety,
- czy OAuth token jest aktywny,
- czy `spotify_query` nie jest puste,
- czy rynek `PL` w node `Spotify - search tracks` jest odpowiedni dla wydarzenia.
