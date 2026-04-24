# Setup n8n dla DJ Request Box

Ten dokument opisuje import i test workflow:

`workflows/01-process-new-request.json`

oraz:

`workflows/02-add-approved-to-spotify.json`

Workflow dziala w trybie polling:

1. Co 1 minute uruchamia go `Schedule Trigger`.
2. Workflow pobiera wiersze z Google Sheet.
3. Przetwarza tylko wiersze, w ktorych kolumna `status` jest pusta.
4. Dla kazdego takiego wiersza normalizuje request przez natywny node OpenAI.
5. Wyszukuje utwor przez natywny node Spotify.
6. Aktualizuje ten sam wiersz po kolumnie `Sygnatura czasowa`.

Workflow 01 nie dodaje utworow do playlisty Spotify. Konczy prace na statusie `FOUND` albo `NOT_FOUND`.

Workflow 02 dodaje do playlisty tylko wiersze, w ktorych DJ recznie ustawi `akcja_dj = APPROVED`, a techniczny `status` nadal wynosi `FOUND`.

## 1. Import workflow

1. Otworz n8n.
2. Wejdz w `Workflows`.
3. Wybierz `Import from File`.
4. Wskaz plik:
   `workflows/01-process-new-request.json`
5. Powtorz import dla:
   `workflows/02-add-approved-to-spotify.json`
6. Po imporcie zostaw oba workflow jako `Inactive`.
7. Otworz kazdy workflow i sprawdz sticky notes po lewej stronie.
8. Podmien placeholdery i podepnij credentials przed pierwszym uruchomieniem.

## 2. Placeholdery do podmiany

W workflow pozostaja tylko placeholdery konfiguracyjne. Nie ma prawdziwych sekretow ani realnych ID credentials.

Podmien:

- `__GOOGLE_SPREADSHEET_ID__`
- `__REQUESTS_SHEET_NAME__`
- `__GOOGLE_SHEETS_CREDENTIAL_NAME__`
- `__OPENAI_CREDENTIAL_NAME__`
- `__SPOTIFY_CREDENTIAL_NAME__`
- `__SPOTIFY_HTTP_OAUTH_CREDENTIAL_NAME__`
- `__SPOTIFY_PLAYLIST_ID__`

`__GOOGLE_SPREADSHEET_ID__` to ID arkusza z URL:

`https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`

`__REQUESTS_SHEET_NAME__` to nazwa zakladki z odpowiedziami, np. `Requests`.

`__SPOTIFY_PLAYLIST_ID__` to ID albo URI playlisty Spotify uzywanej przez workflow 02.

Przyklad URL playlisty:

`https://open.spotify.com/playlist/PLAYLIST_ID`

W takim przypadku `PLAYLIST_ID` to wartosc do wpisania w node:

`HTTP Request - add track to Spotify playlist`

Node'y Google Sheets do sprawdzenia:

- `Google Sheets - read request rows`
- `Google Sheets - update timestamp row FOUND`
- `Google Sheets - update timestamp row NOT_FOUND`
- `Google Sheets - update timestamp row missing title`
- `Google Sheets - update timestamp row ADDED`
- `Google Sheets - update timestamp row ERROR`

## 3. Credentials do wyboru

### Google Sheets OAuth2

Utworz albo wybierz credential:

`Google Sheets - DJ Request Box`

Podepnij go do wszystkich node'ow `Google Sheets - ...`.

Credential musi miec dostep do spreadsheetu z odpowiedziami Google Form.

### OpenAI

Utworz albo wybierz credential:

`OpenAI - DJ Request Box`

Podepnij go do:

- `OpenAI - normalize request to JSON`

Workflow uzywa natywnego node'a:

`@n8n/n8n-nodes-langchain.openAi`

Node jest ustawiony na model `gpt-4.1-mini`, ma wlaczone `jsonOutput: true` i prosi o odpowiedz JSON z polami:

- `AI_title`
- `AI_artist`
- `spotify_query`

Jesli w Twojej instancji n8n model nie jest dostepny, wybierz dostepny model w UI node'a OpenAI i zostaw ten sam prompt oraz `jsonOutput`.

### Spotify OAuth2 dla workflow 01

Utworz albo wybierz credential:

`Spotify - DJ Request Box`

Podepnij go do:

- `Spotify - search tracks by spotify_query`
- `Spotify - fallback search by AI_title`

Workflow uzywa natywnego node'a:

`n8n-nodes-base.spotify`

Konfiguracja node'a Spotify:

- Resource: `track`
- Operation: `search`
- Query pierwszego node'a: `={{ $json.spotify_query }}`
- Query fallback node'a: `={{ $json.AI_title }}`
- Limit: `5`

Workflow 01 wykonuje dwa wyszukiwania Spotify:

1. po `spotify_query` wygenerowanym przez OpenAI,
2. po `AI_title` jako fallback.

Node `Evaluate Spotify results and confidence` laczy wyniki z obu wyszukiwan, usuwa duplikaty po `uri/id`, ocenia wszystkie tracki jednym scoringiem i wybiera najlepszy wynik.

Ten workflow tylko wyszukuje utwory. Scope do modyfikacji playlisty bedzie potrzebny dopiero w workflow `02-add-approved-to-spotify.json`.

### Spotify HTTP OAuth2 dla workflow 02

Workflow 02 nie uzywa natywnego node'a Spotify do dodawania utworu do playlisty. Dodawanie jest wykonywane przez node:

`HTTP Request - add track to Spotify playlist`

Ten node uzywa OAuth2 credential pod placeholderem:

`__SPOTIFY_HTTP_OAUTH_CREDENTIAL_NAME__`

Utworz w n8n credential typu OAuth2 / HTTP OAuth2 zgodny z HTTP Request node. Skonfiguruj go dla Spotify Web API:

- Access Token URL: `https://accounts.spotify.com/api/token`
- Authorization URL: `https://accounts.spotify.com/authorize`
- Scope: `playlist-modify-public playlist-modify-private`

W samym node `HTTP Request - add track to Spotify playlist` ustawienia sa:

- Method: `POST`
- URL: `https://api.spotify.com/v1/playlists/__SPOTIFY_PLAYLIST_ID__/items`
- Authentication: OAuth2 / predefined credential type zgodny z HTTP Request node
- Body Content Type: JSON
- Body:

```json
{
  "uris": ["{{ $json.spotify_uri }}"]
}
```

Credential musi miec dostep do konta Spotify, ktore moze modyfikowac wybrana playliste.

Dla workflow 02 credential HTTP OAuth2 musi pozwalac na modyfikacje playlisty:

- `playlist-modify-public` dla playlisty publicznej,
- `playlist-modify-private` dla playlisty prywatnej.

## 4. Wymagane kolumny Google Sheet

Kolumny wejscia z Google Form:

- `Sygnatura czasowa`
- `Tytuł utworu`
- `Wykonawca`
- `Imię / stolik`
- `Komentarz`
- `akcja_dj`
- `status`

Kolumna `status` powinna byc pusta po wyslaniu formularza. To jest poprawny stan poczatkowy. Workflow traktuje pusty `status` jako sygnal: ten wiersz jest nowy i ma zostac przetworzony.

Kolumny zapisywane przez workflow:

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

Workflow 02 dodatkowo wymaga, zeby po workflow 01 wiersz mial:

- `akcja_dj`
- `status`
- `spotify_uri`

DJ recznie ustawia `akcja_dj = APPROVED`. Workflow 02 przetwarza tylko wiersze, gdzie:

- `akcja_dj = APPROVED`
- `status = FOUND`
- `spotify_uri` nie jest puste

## 5. Jak workflow wybiera wiersze

Workflow nie uzywa Google Sheets Trigger.

Zamiast tego:

1. `Schedule Trigger - every 1 minute` uruchamia workflow co minute.
2. `Google Sheets - read request rows` pobiera wiersze z arkusza.
3. `Filter - only empty status rows` zostawia tylko wiersze, gdzie `status` jest pusty.
4. `Loop - process one request row` przetwarza wiersze pojedynczo.

Workflow nie ustawia `NEW`. Pusty `status` jest stanem roboczym przed przetworzeniem. Po przetworzeniu workflow ustawia `FOUND` albo `NOT_FOUND`.

## 6. Jak dziala aktualizacja tego samego wiersza

Workflow aktualizuje wiersz po fizycznej kolumnie:

`Sygnatura czasowa`

To jest kolumna tworzona przez Google Form. Musi byc obecna w arkuszu i powinna byc unikalna dla odpowiedzi formularza.

W praktyce:

- node `Prepare request - map form fields` kopiuje `Sygnatura czasowa` do pola `timestamp_signature`,
- kazdy update Google Sheets przekazuje `Sygnatura czasowa = {{ $json.timestamp_signature }}`,
- `matchingColumns` w node'ach update jest ustawione na `Sygnatura czasowa`,
- workflow nie uzywa `row_number` jako match key.

## 7. Jak przetestowac workflow na jednym formularzu

1. Utworz testowy spreadsheet albo skopiuj produkcyjny.
2. Upewnij sie, ze workflow wskazuje testowy `__GOOGLE_SPREADSHEET_ID__`.
3. Upewnij sie, ze zakladka ma nazwe zgodna z `__REQUESTS_SHEET_NAME__`.
4. Dodaj wymagane kolumny wynikowe:
   `AI_title`, `AI_artist`, `spotify_query`, `spotify_track_name`, `spotify_artist`, `spotify_url`, `spotify_uri`, `confidence`, `status`, `dj_note`.
5. Podepnij formularz Google Form do arkusza.
6. Wyslij jedna testowa odpowiedz przez formularz:

| Sygnatura czasowa | Tytuł utworu | Wykonawca | Imię / stolik | Komentarz | status |
| --- | --- | --- | --- | --- | --- |
| automatycznie z formularza | September | Earth, Wind & Fire | stolik 4 | test | |

7. W n8n kliknij `Execute Workflow` albo aktywuj workflow i poczekaj maksymalnie 1 minute.
8. Otworz execution i sprawdz kolejne node'y:
   - `Google Sheets - read request rows` powinien pobrac wiersz testowy.
   - `Filter - only empty status rows` powinien przepuscic wiersz, bo `status` jest pusty.
   - `Prepare request - map form fields` powinien pokazac `form_title = September` i `timestamp_signature` z kolumny `Sygnatura czasowa`.
   - `OpenAI - normalize request to JSON` powinien zwrocic `AI_title`, `AI_artist`, `spotify_query`.
   - `Spotify - search tracks by spotify_query` powinien zwrocic wyniki Spotify dla pelnego query.
   - `Spotify - fallback search by AI_title` powinien zwrocic wyniki Spotify dla samego tytulu.
   - `Evaluate Spotify results and confidence` powinien polaczyc oba zestawy wynikow, usunac duplikaty i ustawic `FOUND` albo `NOT_FOUND`.
9. W Google Sheet sprawdz, czy uzupelniony zostal wiersz z ta sama `Sygnatura czasowa`.

Oczekiwany wynik dla poprawnego requestu:

- `status = FOUND`
- `AI_title` wypelnione
- `AI_artist` wypelnione albo puste, jesli OpenAI nie jest pewne wykonawcy
- `spotify_query` wypelnione
- `spotify_track_name` wypelnione
- `spotify_artist` wypelnione
- `spotify_url` wypelnione
- `spotify_uri` wypelnione
- `confidence >= 0.65`
- `dj_note` informuje, ze DJ musi recznie zaakceptowac request

## 8. Jak sprawdzic, czy update trafia do wlasciwego wiersza

W execution n8n sprawdz:

1. Node `Prepare request - map form fields`.
2. Pole `timestamp_signature`.
3. Node koncowy Google Sheets, np. `Google Sheets - update timestamp row FOUND`.
4. Wartosc przekazywana w polu `Sygnatura czasowa`.

Te wartosci powinny byc takie same:

- `timestamp_signature`
- update `Sygnatura czasowa`
- wartosc w arkuszu w kolumnie `Sygnatura czasowa`

Najprostszy test: wyslij formularz tylko raz, zapamietaj jego timestamp i sprawdz, czy workflow uzupelnil dokladnie ten wiersz.

## 9. Test braku tytulu

Wyslij formularz bez tytulu albo recznie dodaj testowy wiersz z pustym polem `Tytuł utworu` i pustym `status`.

Oczekiwany wynik:

- `status = NOT_FOUND`
- `dj_note = Missing song title in the form response...`
- workflow nie uruchamia OpenAI ani Spotify dla tego wiersza

## 10. Test braku sensownego wyniku Spotify

Wyslij formularz z losowym tytulem i pustym `status`:

| Sygnatura czasowa | Tytuł utworu | Wykonawca | Imię / stolik | Komentarz | status |
| --- | --- | --- | --- | --- | --- |
| automatycznie z formularza | asdfgh impossible song xyz | | stolik test | test | |

Oczekiwany wynik:

- `status = NOT_FOUND`
- `spotify_uri` puste
- `confidence` niskie albo `0`
- `dj_note` zawiera informacje, ze nie znaleziono sensownego wyniku

## 11. Zasady bezpieczenstwa

- Nie wpisuj tokenow ani API keys do workflow JSON.
- Nie aktywuj workflow na produkcyjnym arkuszu przed testem na kopii.
- Nie ustawiaj automatycznie `APPROVED` w tym workflow.
- Nie dodawaj w tym workflow nic do playlisty Spotify.
- Dodawanie do playlisty moze zrobic dopiero osobny workflow po recznej akceptacji DJ-a.

## 12. Workflow 02: dodawanie zaakceptowanych utworow

Workflow:

`workflows/02-add-approved-to-spotify.json`

Dziala tak:

1. `Schedule Trigger - every 1 minute` uruchamia workflow co minute.
2. `Google Sheets - read request rows` pobiera wiersze z arkusza.
3. `Filter - only DJ approved FOUND rows` zostawia tylko wiersze, gdzie `akcja_dj = APPROVED` oraz `status = FOUND`.
4. `Prepare approved row` pobiera `Sygnatura czasowa` i `spotify_uri`.
5. `IF - spotify_uri exists` rozdziela poprawne i bledne wiersze.
6. `HTTP Request - add track to Spotify playlist` wysyla `POST` do Spotify Web API:
   `https://api.spotify.com/v1/playlists/__SPOTIFY_PLAYLIST_ID__/items`
   z body:

```json
{
  "uris": ["{{ $json.spotify_uri }}"]
}
```
7. Po sukcesie `Google Sheets - update timestamp row ADDED` ustawia:
   - `status = ADDED`
   - `dj_note = Added to Spotify playlist`
8. Gdy `spotify_uri` jest puste albo dodanie do Spotify sie nie uda, `Google Sheets - update timestamp row ERROR` ustawia:
   - `status = ERROR`
   - `dj_note` z komunikatem bledu

Workflow 02 aktualizuje wiersze po kolumnie:

`Sygnatura czasowa`

Nie uzywa `row_number` jako match key.

## 13. Jak ustawic playlist ID

1. Otworz playlist Spotify.
2. Skopiuj link do playlisty.
3. Z linku:

`https://open.spotify.com/playlist/PLAYLIST_ID`

wez fragment `PLAYLIST_ID`.

4. W n8n otworz node:

`HTTP Request - add track to Spotify playlist`

5. W polu playlisty podmien:

`__SPOTIFY_PLAYLIST_ID__`

na swoje `PLAYLIST_ID`.

Nie wpisuj tokenow OAuth, client secret ani innych sekretow do workflow JSON. Do autoryzacji uzyj credential:

`__SPOTIFY_HTTP_OAUTH_CREDENTIAL_NAME__`

## 14. Reczny test FOUND + akcja_dj APPROVED -> ADDED

1. Uruchom workflow 01 na testowym formularzu.
2. Poczekaj, az w Google Sheet wiersz dostanie:
   - `status = FOUND`
   - `spotify_uri` wypelnione
3. DJ albo tester recznie ustawia w tym samym wierszu:
   - `akcja_dj = APPROVED`
   - `status` zostaje `FOUND`
4. Uruchom workflow 02 recznie przez `Execute Workflow` albo aktywuj go i poczekaj maksymalnie 1 minute.
5. Sprawdz node:
   - `Filter - only DJ approved FOUND rows`
   - powinien przepuscic testowy wiersz
6. Sprawdz node:
   - `HTTP Request - add track to Spotify playlist`
   - powinien wyslac `spotify_uri` do endpointu Spotify playlist items
7. Sprawdz Google Sheet:
   - `status = ADDED`
   - `dj_note = Added to Spotify playlist`
8. Sprawdz playlist Spotify i potwierdz, ze utwor zostal dodany.

Test bledu:

1. Ustaw w testowym wierszu:
   - `akcja_dj = APPROVED`
   - `status = FOUND`
   - `spotify_uri` puste
2. Uruchom workflow 02.
3. Oczekiwany wynik:
   - `status = ERROR`
   - `dj_note = Cannot add to Spotify playlist: spotify_uri is empty for an APPROVED row.`

## 15. Typowe problemy

### Workflow nie widzi nowych formularzy

Sprawdz:

- czy `Schedule Trigger - every 1 minute` jest aktywny,
- czy `Google Sheets - read request rows` wskazuje poprawny spreadsheet i sheet,
- czy kolumna `status` w nowym wierszu jest naprawde pusta.

### Update trafia w zly wiersz albo nie dziala

Sprawdz:

- czy arkusz ma kolumne `Sygnatura czasowa`,
- czy wartosci w `Sygnatura czasowa` sa unikalne,
- czy node update ma `matchingColumns = Sygnatura czasowa`,
- czy `Prepare request - map form fields` ustawia `timestamp_signature`.

### OpenAI node zwraca blad

Sprawdz:

- czy credential OpenAI jest podpiety,
- czy model jest dostepny w Twojej instancji,
- czy `jsonOutput` jest wlaczone,
- czy odpowiedz zawiera `AI_title`, `AI_artist`, `spotify_query`.

### Spotify nie zwraca wynikow

Sprawdz:

- czy credential Spotify jest podpiety,
- czy node ma `Resource = track`,
- czy node ma `Operation = search`,
- czy `spotify_query` nie jest puste,
- czy `AI_title` nie jest puste,
- czy oba node'y Spotify zwracaja wyniki albo czy fallback po `AI_title` poprawia wynik.

### Workflow 02 nie dodaje do playlisty

Sprawdz:

- czy wiersz ma dokladnie `akcja_dj = APPROVED`,
- czy wiersz ma dokladnie `status = FOUND`,
- czy `spotify_uri` nie jest puste,
- czy `__SPOTIFY_PLAYLIST_ID__` zostal podmieniony,
- czy `__SPOTIFY_HTTP_OAUTH_CREDENTIAL_NAME__` jest podpiety w HTTP Request node,
- czy HTTP OAuth2 credential ma scope do modyfikacji playlisty,
- czy konto Spotify z credential jest wlascicielem albo ma dostep do playlisty.

### Co oznacza status ERROR w workflow 02

`ERROR` oznacza, ze DJ zaakceptowal wiersz, ale workflow 02 nie mogl dodac utworu do playlisty.

Najczestsze powody:

- `spotify_uri` jest puste,
- `spotify_uri` ma zly format,
- playlist ID jest niepoprawne,
- credential HTTP OAuth2 nie ma wymaganych scope,
- konto Spotify nie ma dostepu do playlisty,
- Spotify API zwrocilo blad.

Szczegol bledu jest zapisywany w `dj_note`.
