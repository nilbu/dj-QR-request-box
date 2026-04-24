# Plan implementacji n8n MVP dla DJ QR Request Box

## Cel MVP

MVP ma obsluzyc zbieranie requestow muzycznych od gosci przez Google Form, zapis do Google Sheet, automatyczna normalizacje i wyszukanie utworu w Spotify przez n8n, a nastepnie reczna akceptacje DJ-a przed dodaniem utworu do dedykowanej playlisty Spotify.

Zakres MVP:
- Google Form jako publiczny formularz dostepny z kodu QR.
- Google Sheet jako jedyne zrodlo danych operacyjnych i panel akceptacji.
- n8n jako warstwa automatyzacji.
- AI tylko do normalizacji danych z formularza.
- Spotify Search do znalezienia najlepszego dopasowania.
- Reczna decyzja DJ-a przez zmiane statusu na `APPROVED`.
- Dodawanie do playlisty Spotify dopiero po statusie `APPROVED`.

Poza zakresem na tym etapie:
- Wlasny backend.
- Aplikacja Next.js.
- Automatyczne dodawanie requestow bez akceptacji DJ-a.
- Wpisywanie tokenow, sekretow lub credentials do eksportowanych plikow JSON n8n.

## 1. Struktura Google Form

Formularz powinien byc prosty, szybki do wypelnienia na telefonie i odporny na niepelne dane.

### Nazwa formularza

`DJ Request Box`

### Opis formularza

Krotka informacja dla gosci:
- popros o podanie utworu,
- zaznacz, ze DJ decyduje o kolejnosci i wyborze,
- dodaj informacje, ze request nie gwarantuje odtworzenia.

Przykladowy opis:

`Wpisz utwor, ktory chcesz uslyszec. DJ wybiera requesty pasujace do momentu imprezy.`

### Pola formularza

1. `Song title`
   - Typ: Short answer.
   - Wymagane: tak.
   - Cel: tytul utworu podany przez goscia.
   - Przyklad: `September`.

2. `Artist`
   - Typ: Short answer.
   - Wymagane: nie.
   - Cel: wykonawca, jezeli gosc go zna.
   - Przyklad: `Earth, Wind & Fire`.

3. `Your name`
   - Typ: Short answer.
   - Wymagane: nie.
   - Cel: imie osoby skladajacej request.
   - Przyklad: `Ania`.

4. `Dedication`
   - Typ: Paragraph.
   - Wymagane: nie.
   - Cel: dedykacja, komentarz lub kontekst dla DJ-a.
   - Przyklad: `Dla pary mlodej`.

5. `Consent / note`
   - Typ: Checkbox.
   - Wymagane: tak.
   - Wartosc: `I understand this request may be moderated by the DJ.`
   - Cel: ograniczenie oczekiwania, ze kazdy request zostanie odtworzony.

### Ustawienia formularza

- Nie wymagac logowania do konta Google.
- Nie zbierac adresow e-mail.
- Ograniczyc formularz do jednej strony.
- Wlaczyc zapis odpowiedzi do Google Sheet.
- Po wyslaniu pokazac komunikat:
  `Thanks! Your request was sent to the DJ.`

## 2. Struktura Google Sheet

Google Sheet pelni trzy role:
- baza requestow,
- widok roboczy dla DJ-a,
- stan przetwarzania workflow n8n.

### Arkusz glowny

Nazwa arkusza:

`Requests`

Ten arkusz jest podlaczony do Google Form i aktualizowany przez n8n.

### Arkusz opcjonalny dla konfiguracji

Nazwa arkusza:

`Config`

Moze przechowywac niesekretne ustawienia operacyjne, np. nazwe playlisty, identyfikator wydarzenia, limit requestow na okno czasowe. Sekrety i tokeny nie moga byc przechowywane w arkuszu.

### Widoki / filtry dla DJ-a

W arkuszu `Requests` warto utworzyc filtry:
- `New / Found`: wiersze ze statusem `FOUND`.
- `Approved`: wiersze ze statusem `APPROVED`.
- `Added`: wiersze ze statusem `ADDED`.
- `Needs review`: wiersze ze statusem `NEEDS_REVIEW` albo niska wartoscia `confidence`.
- `Rejected`: wiersze ze statusem `REJECTED`.

### Zamrozenie i ochrona

- Zamrozic pierwszy wiersz z naglowkami.
- Zabezpieczyc kolumny techniczne przed przypadkowa edycja.
- Pozostawic DJ-owi do recznej edycji przede wszystkim kolumny:
  - `status`,
  - `dj_notes`,
  - ewentualnie `manual_spotify_uri`.

## 3. Wymagane kolumny Google Sheet

Kolumny powinny byc stabilne, poniewaz workflow n8n bedzie mapowal dane po nazwach naglowkow.

### Kolumny z Google Form

1. `timestamp`
   - Zrodlo: Google Form.
   - Typ: data/czas.
   - Opis: moment wyslania formularza.

2. `song_title_raw`
   - Zrodlo: Google Form.
   - Typ: tekst.
   - Opis: tytul wpisany przez goscia.

3. `artist_raw`
   - Zrodlo: Google Form.
   - Typ: tekst.
   - Opis: wykonawca wpisany przez goscia.

4. `requester_name`
   - Zrodlo: Google Form.
   - Typ: tekst.
   - Opis: imie osoby skladajacej request.

5. `dedication`
   - Zrodlo: Google Form.
   - Typ: tekst.
   - Opis: dedykacja lub komentarz.

6. `consent`
   - Zrodlo: Google Form.
   - Typ: tekst/checkbox.
   - Opis: potwierdzenie zasad moderacji requestow.

### Kolumny techniczne n8n

7. `request_id`
   - Zrodlo: n8n.
   - Typ: tekst.
   - Opis: stabilny identyfikator requestu, np. hash z timestampu i tresci albo UUID.

8. `normalized_title`
   - Zrodlo: AI normalizacja.
   - Typ: tekst.
   - Opis: oczyszczony tytul utworu.

9. `normalized_artist`
   - Zrodlo: AI normalizacja.
   - Typ: tekst.
   - Opis: oczyszczony wykonawca.

10. `search_query`
    - Zrodlo: n8n / AI.
    - Typ: tekst.
    - Opis: zapytanie uzyte do Spotify Search.

11. `spotify_track_name`
    - Zrodlo: Spotify Search.
    - Typ: tekst.
    - Opis: nazwa znalezionego utworu.

12. `spotify_artist_name`
    - Zrodlo: Spotify Search.
    - Typ: tekst.
    - Opis: glowny wykonawca znalezionego utworu.

13. `spotify_album_name`
    - Zrodlo: Spotify Search.
    - Typ: tekst.
    - Opis: album znalezionego utworu.

14. `spotify_url`
    - Zrodlo: Spotify Search.
    - Typ: URL.
    - Opis: link do utworu w Spotify dla DJ-a.

15. `spotify_uri`
    - Zrodlo: Spotify Search.
    - Typ: tekst.
    - Opis: URI wymagane do dodania utworu do playlisty, np. `spotify:track:...`.

16. `manual_spotify_uri`
    - Zrodlo: DJ.
    - Typ: tekst.
    - Opis: reczny override, jezeli automatyczne dopasowanie jest bledne.

17. `confidence`
    - Zrodlo: n8n.
    - Typ: liczba od `0` do `1`.
    - Opis: pewnosc dopasowania.

18. `status`
    - Zrodlo: n8n / DJ.
    - Typ: enum tekstowy.
    - Opis: aktualny stan requestu.

19. `status_reason`
    - Zrodlo: n8n.
    - Typ: tekst.
    - Opis: krotkie wyjasnienie statusu, np. `No Spotify match` albo `Low confidence`.

20. `processed_at`
    - Zrodlo: n8n.
    - Typ: data/czas.
    - Opis: kiedy workflow `01-process-new-request` przetworzyl wiersz.

21. `approved_at`
    - Zrodlo: DJ / n8n.
    - Typ: data/czas.
    - Opis: kiedy request zostal zaakceptowany. Moze byc wypelniany przez drugi workflow przy wykryciu `APPROVED`, jesli pole jest puste.

22. `added_at`
    - Zrodlo: n8n.
    - Typ: data/czas.
    - Opis: kiedy utwor zostal dodany do playlisty.

23. `playlist_id`
    - Zrodlo: n8n.
    - Typ: tekst.
    - Opis: identyfikator playlisty Spotify, do ktorej dodano utwor.

24. `n8n_execution_id`
    - Zrodlo: n8n.
    - Typ: tekst.
    - Opis: identyfikator wykonania workflow przydatny do debugowania.

25. `error_message`
    - Zrodlo: n8n.
    - Typ: tekst.
    - Opis: ostatni blad dotyczacy wiersza, jezeli wystapil.

26. `retry_count`
    - Zrodlo: n8n.
    - Typ: liczba.
    - Opis: licznik ponowien dla bledow przejsciowych.

27. `dj_notes`
    - Zrodlo: DJ.
    - Typ: tekst.
    - Opis: robocze notatki DJ-a.

### Statusy

Wymagane statusy:

- `NEW`: nowa odpowiedz z formularza, jeszcze nieprzetworzona.
- `PROCESSING`: wiersz aktualnie obslugiwany przez workflow.
- `FOUND`: utwor znaleziony, czeka na decyzje DJ-a.
- `NEEDS_REVIEW`: brak pewnego dopasowania, wymaga sprawdzenia.
- `APPROVED`: DJ zaakceptowal request do dodania.
- `ADDED`: utwor zostal dodany do playlisty Spotify.
- `REJECTED`: DJ odrzucil request.
- `ERROR`: wystapil blad techniczny.

Minimalna regula MVP:
- tylko status `APPROVED` moze uruchomic dodanie do playlisty.
- workflow nie moze dodawac utworu ze statusem `FOUND`, `NEEDS_REVIEW`, `NEW` lub `ERROR`.

## 4. Wymagane credentials w n8n

Credentials musza byc skonfigurowane w n8n i podpinane do node'ow przez referencje n8n. Nie wolno wpisywac tokenow, client secretow ani kluczy API do eksportowanego JSON workflow.

### Google Sheets OAuth2

Nazwa credential w n8n:

`Google Sheets - DJ Request Box`

Zakres:
- odczyt arkusza `Requests`,
- aktualizacja wierszy,
- ewentualnie odczyt arkusza `Config`.

Wymagane uprawnienia:
- Google Sheets API.
- Dostep do konkretnego spreadsheetu uzywanego przez wydarzenie.

Zastosowanie:
- workflow `01-process-new-request.json`,
- workflow `02-add-approved-to-spotify.json`.

### Spotify OAuth2

Nazwa credential w n8n:

`Spotify - DJ Playlist`

Zakres:
- wyszukiwanie utworow,
- dodawanie utworow do playlisty.

Wymagane scope Spotify:
- `playlist-modify-public` jezeli playlista jest publiczna,
- `playlist-modify-private` jezeli playlista jest prywatna.

Zastosowanie:
- Spotify Search w workflow `01-process-new-request.json`,
- dodawanie trackow do playlisty w workflow `02-add-approved-to-spotify.json`.

### AI provider credential

Nazwa credential w n8n:

`AI - Request Normalization`

Provider:
- OpenAI, albo inny dostawca obslugiwany przez n8n.

Zastosowanie:
- normalizacja `song_title_raw` i `artist_raw`,
- przygotowanie `normalized_title`, `normalized_artist`, `search_query`,
- ewentualna pomoc w ocenie dopasowania tekstowego.

Zasada:
- prompt nie moze wymagac danych wrazliwych,
- do AI wysylac tylko pola potrzebne do normalizacji requestu.

### Opcjonalne credentials / konfiguracja

1. `n8n environment variables`
   - `SPOTIFY_PLAYLIST_ID`
   - `REQUESTS_SPREADSHEET_ID`
   - `REQUESTS_SHEET_NAME`
   - `MIN_CONFIDENCE_FOUND`

2. `Error notification credential`
   - Slack, Discord, e-mail albo inny kanal alertow.
   - Poza obowiazkowym MVP, ale zalecane przy wydarzeniu live.

## 5. Opis dwoch workflow

Workflow maja byc docelowo eksportowalne jako JSON do importu w n8n:

- `workflows/01-process-new-request.json`
- `workflows/02-add-approved-to-spotify.json`

Na tym etapie nie tworzymy tych plikow.

### Workflow 1: `01-process-new-request.json`

Cel:

Wykryc nowy wiersz w Google Sheet, znormalizowac dane requestu, wyszukac utwor w Spotify i zaktualizowac ten sam wiersz wynikiem oraz statusem.

#### Trigger

Rekomendowany trigger MVP:
- Schedule Trigger co 15-30 sekund albo co 1 minute.

Powod:
- Google Sheets w n8n jest prostsze i stabilniejsze w pollingu niz poleganie na niestandardowych webhookach z Google Form.

#### Warunek wyboru wierszy

Workflow pobiera wiersze z arkusza `Requests`, dla ktorych:
- `status` jest puste albo `NEW`,
- `song_title_raw` nie jest puste,
- `spotify_uri` jest puste,
- `processed_at` jest puste.

#### Kroki

1. Pobierz wiersze z Google Sheet.
2. Odfiltruj tylko nieprzetworzone requesty.
3. Dla kazdego wiersza ustaw tymczasowo:
   - `status = PROCESSING`,
   - `n8n_execution_id = current execution id`.
4. Przygotuj payload do AI:
   - `song_title_raw`,
   - `artist_raw`,
   - opcjonalnie `dedication` tylko jezeli pomaga rozpoznac utwor.
5. AI zwraca strukture:
   - `normalized_title`,
   - `normalized_artist`,
   - `search_query`,
   - `normalization_notes`.
6. Wykonaj Spotify Search:
   - najpierw query z wykonawca, jezeli jest dostepny,
   - fallback: sam tytul,
   - limit wynikow: 3-5.
7. Oblicz `confidence`:
   - porownanie tytulu,
   - porownanie wykonawcy,
   - bonus za dokladne dopasowanie,
   - kara za remix/live/karaoke/cover, jezeli nie bylo ich w requestcie.
8. Wybierz najlepszy wynik.
9. Zaktualizuj wiersz:
   - gdy dopasowanie jest dobre: `status = FOUND`,
   - gdy brak wyniku albo niska pewnosc: `status = NEEDS_REVIEW`,
   - wypelnij pola Spotify, confidence i status reason.
10. W przypadku bledu:
    - `status = ERROR`,
    - `error_message = krotki opis`,
    - `retry_count = retry_count + 1`.

#### Minimalne progi

- `confidence >= 0.75`: `FOUND`.
- `confidence < 0.75`: `NEEDS_REVIEW`.
- brak `spotify_uri`: `NEEDS_REVIEW`.

Progi powinny byc latwe do zmiany przez zmienna env albo stala w workflow.

#### Output

Workflow nie dodaje nic do playlisty Spotify. Jedynym efektem jest aktualizacja wiersza Google Sheet.

### Workflow 2: `02-add-approved-to-spotify.json`

Cel:

Cyklicznie sprawdzac wiersze zaakceptowane przez DJ-a i dodawac ich `spotify_uri` do dedykowanej playlisty Spotify.

#### Trigger

Rekomendowany trigger MVP:
- Schedule Trigger co 15-30 sekund albo co 1 minute.

#### Warunek wyboru wierszy

Workflow pobiera wiersze z arkusza `Requests`, dla ktorych:
- `status = APPROVED`,
- `added_at` jest puste,
- `spotify_uri` albo `manual_spotify_uri` nie jest puste.

#### Kroki

1. Pobierz wiersze z Google Sheet.
2. Odfiltruj tylko `status = APPROVED`.
3. Dla kazdego wiersza wybierz URI:
   - najpierw `manual_spotify_uri`, jezeli wypelnione,
   - w przeciwnym razie `spotify_uri`.
4. Sprawdz, czy URI ma format `spotify:track:...`.
5. Opcjonalnie sprawdz, czy utwor nie zostal juz dodany:
   - w tym samym arkuszu przez status `ADDED`,
   - albo na playliscie Spotify przez pobranie ostatnich utworow.
6. Dodaj utwor do playlisty Spotify.
7. Zaktualizuj wiersz:
   - `status = ADDED`,
   - `added_at = now`,
   - `playlist_id = configured playlist id`,
   - `error_message = empty`.
8. W przypadku bledu:
   - nie zmieniaj na `ADDED`,
   - ustaw `status = ERROR` albo zostaw `APPROVED` i wypelnij `error_message`, zaleznie od typu bledu,
   - zwieksz `retry_count`.

#### Zasada bezpieczenstwa

Ten workflow musi miec twardy warunek:

`status == APPROVED`

Nie wolno dodawac do playlisty wierszy ze statusem `FOUND`, nawet przy wysokim `confidence`.

## 6. Ryzyka i zabezpieczenia

### Ryzyko: przypadkowe dodanie niezaakceptowanego utworu

Zabezpieczenia:
- drugi workflow filtruje wylacznie `status = APPROVED`,
- pierwszy workflow nigdy nie wywoluje akcji dodania do playlisty,
- test importu workflow powinien obejmowac przypadki `FOUND`, `NEEDS_REVIEW`, `ERROR`, ktore nie moga dodac utworu.

### Ryzyko: bledne dopasowanie Spotify

Zabezpieczenia:
- uzywac `confidence`,
- status `NEEDS_REVIEW` dla niskiej pewnosci,
- pokazywac DJ-owi `spotify_url`, `spotify_track_name`, `spotify_artist_name`,
- dodac `manual_spotify_uri` jako reczny override.

### Ryzyko: duplikaty na playliscie

Zabezpieczenia:
- przed dodaniem sprawdzac, czy dany wiersz nie ma `added_at`,
- opcjonalnie sprawdzac, czy `spotify_uri` juz istnieje w arkuszu ze statusem `ADDED`,
- opcjonalnie pobierac ostatnie pozycje playlisty i porownywac URI.

### Ryzyko: rownolegle wykonania workflow

Zabezpieczenia:
- pierwszy workflow ustawia `PROCESSING` przed dalszym przetwarzaniem,
- drugi workflow po dodaniu natychmiast ustawia `ADDED`,
- ograniczyc concurrency workflow w n8n do 1 dla MVP,
- uzywac `n8n_execution_id` do debugowania.

### Ryzyko: limity API Google, AI albo Spotify

Zabezpieczenia:
- polling co 30-60 sekund zamiast agresywnego odpytywania,
- batch processing z limitem wierszy na jedno wykonanie,
- retry z limitem `retry_count`,
- oznaczanie trwalych bledow statusem `ERROR`.

### Ryzyko: wyciek sekretow

Zabezpieczenia:
- credentials tylko w n8n Credentials,
- playlist ID i spreadsheet ID najlepiej jako env vars albo niesekretna konfiguracja,
- eksport JSON workflow bez tokenow i sekretow,
- nie wpisywac API key w node Function, Code ani HTTP Request.

### Ryzyko: niepozadane tresci w dedykacjach

Zabezpieczenia:
- dedykacja nie jest publikowana automatycznie,
- DJ widzi dedykacje przed ewentualnym uzyciem,
- mozna dodac filtr slow w przyszlej wersji,
- do AI wysylac minimalny potrzebny zakres danych.

### Ryzyko: manipulacja formularzem przez osoby spoza wydarzenia

Zabezpieczenia:
- publikowac link tylko przez QR na miejscu,
- opcjonalnie dodac proste haslo/event code w formularzu,
- monitorowac nagly wzrost requestow,
- dodac limit przetwarzanych requestow na jedno wykonanie.

### Ryzyko: przypadkowa edycja arkusza przez DJ-a

Zabezpieczenia:
- zabezpieczyc kolumny techniczne,
- udostepnic DJ-owi filtrowany widok,
- uzywac walidacji danych dla kolumny `status`,
- dopuszczalne wartosci statusu trzymac jako liste.

### Ryzyko: bledne formaty danych

Zabezpieczenia:
- walidowac `spotify_uri` przed dodaniem do playlisty,
- traktowac puste `song_title_raw` jako blad danych,
- wymagac `song_title_raw` w Google Form,
- zapisywac `error_message` przy kazdym bledzie walidacji.

## Kolejnosc dalszej implementacji

1. Przygotowac `docs/setup-google-form.md`.
2. Przygotowac `docs/setup-google-sheet.md`.
3. Przygotowac `docs/setup-spotify.md`.
4. Przygotowac `docs/setup-n8n.md`.
5. Przygotowac `docs/operating-live-event.md`.
6. Dopiero potem utworzyc eksportowalne workflow:
   - `workflows/01-process-new-request.json`,
   - `workflows/02-add-approved-to-spotify.json`.
7. Zaimportowac workflow do n8n testowo i sprawdzic na kopii arkusza.
8. Przetestowac scenariusze:
   - poprawny request,
   - brak wykonawcy,
   - brak dopasowania Spotify,
   - niska pewnosc,
   - reczny override przez `manual_spotify_uri`,
   - status `APPROVED`,
   - duplikat,
   - blad API.
