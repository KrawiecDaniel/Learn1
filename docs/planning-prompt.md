# Prompt planistyczny: Copilot Bridge (uniwersalne lokalne API AI → M365 Copilot → JSON)

Jesteś doświadczonym architektem oprogramowania. Przygotuj szczegółowy plan implementacji narzędzia **copilot-bridge**. Nie pisz jeszcze kodu produkcyjnego. Twoim zadaniem jest plan, który da się potem wykonać krok po kroku. Odpowiedz po polsku.

## Cel

**Uniwersalne, lokalne API AI** działające na firmowym laptopie. Pod spodem działa Microsoft 365 Copilot w przeglądarce Edge sterowanej przez Playwright, ale dla klientów wygląda to jak zwykłe API zwracające ustrukturyzowany JSON. Dowolne narzędzie albo skrypt może z niego skorzystać do różnych zadań: odpowiadania na pytania, streszczania, klasyfikacji, wyciągania danych z tekstu i dokumentów, generowania treści.

Całość:
1. przyjmuje zadanie od klienta (API HTTP albo CLI): pytanie wprost albo nazwane zadanie z danymi wejściowymi (tekst i opcjonalnie pliki),
2. opakowuje je w prompt wymuszający odpowiedź w ustalonym schemacie JSON,
3. przekazuje je do stałego daemona wystawiającego lokalne API HTTP (CLI uruchamia daemona, jeśli nie działa), który przez Playwright steruje Edge z zalogowaną sesją Microsoft 365 Copilot, wysyła prompt i czeka na pełną odpowiedź,
4. wyciąga odpowiedź ze strony, parsuje ją, waliduje względem schematu i w razie potrzeby naprawia,
5. zwraca poprawny JSON, który da się przetwarzać maszynowo: przez API HTTP dla narzędzi i przez stdout CLI dla skryptów.

Przykład użycia (PowerShell):
```powershell
copilot-bridge ask "Jakie są zalety Rusta?" --schema default --timeout 120
copilot-bridge task summarize --input .\notatka.txt
copilot-bridge task classify --input-file .\zgloszenia.json
copilot-bridge task extract --file .\faktura.pdf --schema .\schemas\invoice.json

$token = Get-Content "$env:LOCALAPPDATA\copilot-bridge\token"
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8765/v1/tasks/extract-invoice `
  -Headers @{ Authorization = "Bearer $token" } -ContentType "application/json" `
  -Body '{"input": {"text": "..."}}'
```

## Decyzje już podjęte

- **Samodzielne, działające narzędzie.** Budujemy kompletne rozwiązanie do codziennego użytku, nie prototyp. Droga przez Playwright i interfejs M365 Copilota jest przesądzona, nie analizuj alternatyw. Jeden użytkownik, jego laptop i jego konto M365.
- **Platforma: firmowy laptop z Windowsem, przeglądarka Edge.** Laptop łączy się przez VPN. Plan projektuj pod Windows (procesy w tle, blokady, uprawnienia do plików, autostart, ścieżki w `%LOCALAPPDATA%`), bez rozwiązań linuksowych.
- **Działanie po uśpieniu jest wymaganiem.** Daemon ma przetrwać uśpienie i hibernację laptopa i po wybudzeniu sam wrócić do pracy, bez ręcznej interwencji (o ile sesja M365 nie wygasła).
- **Uniwersalność ponad jeden przypadek użycia.** Rozwiązanie nie jest zbudowane pod jedno zadanie. Nowy przypadek użycia dodaje się konfiguracją (szablon zadania + schemat), bez zmiany kodu.
- **Docelowy serwis: Microsoft 365 Copilot** (konto firmowe, logowanie przez Entra ID, czat pod adresem w rodzaju `m365.cloud.microsoft/chat`). Nie masz dostępu do tej strony, więc aktualny URL, selektory i zachowanie interfejsu plan ma przewidzieć do sprawdzenia w M1.
- **Tryb pracy: stały daemon.** Jeden długo działający proces trzyma otwartą przeglądarkę z zalogowaną sesją. CLI jest cienkim klientem. Jeśli daemon nie działa, CLI **sam go uruchamia** w tle i dopiero potem wysyła zapytanie.
- **Daemon to wierne API.** Komunikacja przez HTTP na `127.0.0.1`, kontrakt zdefiniowany najpierw (contract-first, OpenAPI), wersjonowany (`/v1`). Narzędzia wołają API bezpośrednio, CLI to tylko jeden z klientów. Z punktu widzenia klienta nie może być widać, że pod spodem jest przeglądarka.

## Pytania, które trzeba rozstrzygnąć na początku

Zanim zaproponujesz architekturę, wypisz te pytania i podaj rekomendowaną odpowiedź na każde:
- **Stos technologiczny:** TypeScript/Node czy Python? Uzasadnij wybór (Playwright, serwer HTTP, walidacja schematu, daemon na Windowsie, dystrybucja CLI, np. jako pojedynczy plik wykonywalny, instalacja najlepiej bez uprawnień administratora).
- **Kształt API:** własny REST (zadania, joby, rozmowy) plus **adapter zgodny z formatem OpenAI** (`/v1/chat/completions` z `response_format: json_schema`). Adapter sprawia, że gotowe SDK, biblioteki i narzędzia działają od razu. Oceń, jak daleko warto iść w zgodności.

## Obszary, które plan musi pokryć

### 1. Architektura i przepływ danych
Diagram albo opis warstw: Klienci (CLI, własne narzędzia, SDK zgodne z OpenAI) → HTTP API (`/v1`) → Daemon [Auth → Kolejka/Joby → TaskRegistry → Cache → PromptBuilder → **Backend** → JsonParser/Validator] → odpowiedź.

Warstwa `Backend` to interfejs z wymiennymi implementacjami:
- `playwright`: BrowserDriver + ResponseExtractor (właściwe działanie),
- `mock`: deterministyczne odpowiedzi z plików, do pracy nad klientami bez przeglądarki i do testów.

Lokalny magazyn stanu (np. SQLite): historia zapytań, joby, mapowanie `conversation_id` na wątki w Copilocie, cache i nagrania odpowiedzi.

Określ odpowiedzialność każdej warstwy i interfejsy między nimi.

### 2. Kontrakt lokalnego API (kluczowe)
Zaprojektuj API tak, jakby było prawdziwym, publicznym serwisem:
- **Endpointy** (propozycja do dopracowania):
  - `GET /v1/health`: stan daemona i sesji (`starting`, `ready`, `auth_required`, `busy`, `resuming`, `degraded`), wersja, backend,
  - `POST /v1/ask`: dowolne pytanie z opcjonalnym schematem, synchronicznie z timeoutem (skrót do zadania `ask`),
  - `GET /v1/tasks`, `POST /v1/tasks/{name}`: lista i wywołanie nazwanych zadań (sekcja 3),
  - `POST /v1/tasks/{name}/batch`: wiele elementów w jednym wywołaniu (sekcja 12),
  - `POST /v1/jobs` + `GET /v1/jobs/{id}` + `DELETE /v1/jobs/{id}`: tryb asynchroniczny, bo odpowiedzi trwają od kilku do kilkudziesięciu sekund,
  - opcjonalnie strumieniowanie przez SSE,
  - `POST /v1/conversations` i `POST /v1/conversations/{id}/messages`: rozmowy wieloturowe mapowane na wątek w Copilocie,
  - `GET /v1/schemas`, `GET /v1/schemas/{name}`: dostępne schematy odpowiedzi,
  - `POST /v1/chat/completions`: adapter zgodny z formatem OpenAI,
  - `GET /v1/metrics`: statystyki działania (sekcja 16).
- **Specyfikacja OpenAPI** jako artefakt w repozytorium, z której da się wygenerować klienta i dokumentację.
- **Uwierzytelnienie:** token Bearer generowany przy pierwszym starcie, w pliku dostępnym tylko dla użytkownika.
- **Zachowanie jak prawdziwe API:** kody HTTP zmapowane na kody błędów (`401`, `404` dla nieznanego zadania, `409`, `413` przy za dużym wejściu, `422`, `429` z `Retry-After`, `503` przy `auth_required` i wznawianiu po uśpieniu, `504` przy timeoucie), nagłówek `X-Request-Id`, `Idempotency-Key` dla bezpiecznych ponowień, nagłówki limitów, jednolity format błędu.
- **Parametry zapytania:** `question` albo `input`, `files`, `schema` (nazwa albo inline JSON Schema), `mode` (`work|web`), `timeout_ms`, `conversation_id`, `include_raw`.
- **Załączniki:** pliki (PDF, DOCX, XLSX, obrazy) jako `multipart/form-data` w API i `--file` w CLI. Limity rozmiaru i typów zgodne z tym, co przyjmuje M365 Copilot.
- **Semantyka adaptera zgodnego z OpenAI:** jak role `system`/`user`/`assistant` są spłaszczane do jednej wiadomości w Copilocie, co z parametrami bez odpowiednika (`temperature`, `n`, `tools`, `max_tokens`: ignorowane z ostrzeżeniem czy błąd `400`), skąd pole `usage` (szacunek) i jak `response_format: json_schema` przechodzi na kopertę promptu.
- **Stabilność kontraktu:** zasady wersjonowania i co jest zmianą łamiącą.

### 3. Rejestr zadań (uniwersalność)
- **Zadanie** to plik konfiguracyjny (YAML/JSON): nazwa, opis, szablon promptu z miejscami na dane, schemat wejścia, schemat wyjścia, przykłady (few-shot), domyślny `mode` i timeout.
- Dodanie nowego przypadku użycia = dodanie pliku, bez zmiany kodu i bez restartu (albo z przeładowaniem na żądanie).
- **Zadania wbudowane** (propozycja): `ask` (pytanie ogólne), `summarize` (streszczenie z kluczowymi punktami), `classify` (klasyfikacja do podanych etykiet z uzasadnieniem), `extract` (wyciąganie pól według schematu podanego przez klienta, także z załączonych plików), `rewrite` (zmiana tonu, tłumaczenie, korekta), `draft` (szkic maila lub dokumentu), `work-search` (pytanie o dane firmowe w trybie Work).
- **Schematy dynamiczne:** schemat wyjścia może zależeć od wejścia, np. `enum` budowany z etykiet przekazanych do `classify`, żeby walidator odrzucał etykiety spoza listy.
- Walidacja wejścia przed wysłaniem do Copilota, limit rozmiaru danych wejściowych, dzielenie długich tekstów, jeśli potrzebne.
- Wersjonowanie zadań i koperty promptu (wersje zapisywane w `meta`), żeby zmiana szablonu nie psuła klientów po cichu, a wyniki dało się porównać.

### 4. Cykl życia daemona (kluczowe)
- **Auto-start z CLI:** CLI próbuje się połączyć. Jeśli nie ma daemona, uruchamia go jako odłączony proces w tle, bez okna konsoli, czeka na gotowość (`/v1/health` z timeoutem) i dopiero wtedy wysyła zapytanie.
- **Wyścigi przy starcie:** dwa równoległe wywołania CLI nie mogą uruchomić dwóch daemonów. Zaprojektuj blokadę działającą na Windowsie (np. nazwany mutex albo plik otwarty na wyłączność) i plik PID.
- **Wykrywanie martwego daemona:** nieaktualny plik PID, zajęty port po awarii, proces wiszący bez odpowiedzi. Jak CLI to rozpoznaje i sprząta.
- **Gotowość vs. żywotność:** daemon działa, ale sesja M365 wygasła albo trwa wznawianie po uśpieniu. Health check ma zwracać stan (`starting`, `ready`, `auth_required`, `busy`, `resuming`, `degraded`).
- **Komendy zarządzania:** `copilot-bridge daemon start|stop|restart|status|logs`, plus `copilot-bridge login` do ręcznego logowania w widocznym oknie.
- **Zgodność wersji:** co się dzieje, gdy CLI zostało zaktualizowane, a stary daemon wciąż działa (sprawdzenie wersji API przez `/v1/health`, automatyczny restart).
- **Zasoby:** limit pamięci przeglądarki, okresowe odświeżanie karty lub kontekstu.
- **Start razem z systemem:** Harmonogram zadań albo folder Autostart (bez uprawnień administratora), jako uzupełnienie auto-startu z CLI.
- Logi daemona do pliku z rotacją w `%LOCALAPPDATA%`.

### 5. Sesja i logowanie (Entra ID)
- Pierwsze logowanie w widocznym oknie (użytkownik loguje się ręcznie, łącznie z MFA), potem trwały profil (`launchPersistentContext`). Daemon po zalogowaniu może pracować headless albo w zminimalizowanym oknie. Oceń, czy M365 działa poprawnie w trybie headless.
- Specyfika firmowa: SSO, Conditional Access, zgodność urządzenia, wygasanie sesji po określonym czasie. Jak daemon wykrywa przekierowanie na stronę logowania i przechodzi w stan `auth_required` zamiast się wieszać.
- Ścieżka odzyskania: API zwraca `503` z kodem `AUTH_REQUIRED`, a CLI dodaje instrukcję uruchomienia `copilot-bridge login`, które otwiera widoczne okno w tym samym profilu.
- Gdzie i jak bezpiecznie przechowywać profil i ciasteczka (katalog dostępny tylko dla użytkownika, poza repozytorium).

### 6. Środowisko uruchomieniowe (Windows, Edge, uśpienie)
- **Przeglądarka:** zainstalowany Edge (`channel: "msedge"`), bez pobierania osobnego Chromium. Zawsze osobny katalog profilu, nie codzienny profil użytkownika (blokada profilu, mieszanie danych). Sprawdź, czy tak uruchomiony Edge przechodzi SSO i Conditional Access, oraz czy funkcje oszczędzania Edge (usypianie kart, tryb wydajności) nie usypiają karty daemona.
- **Uśpienie i wybudzenie (wymaganie):**
  - wykrycie wybudzenia (np. zdarzenia zasilania Windows albo skok czasu między kolejnymi sygnałami życia),
  - odczekanie na sieć, w tym ponowne zestawienie VPN,
  - sprawdzenie, czy karta i połączenie strony z serwerem żyją; w razie potrzeby przeładowanie karty albo restart przeglądarki,
  - stan `resuming` w `/v1/health` na czas wznawiania,
  - zapytania przerwane uśpieniem: automatyczne ponowienie albo błąd `INTERRUPTED`, który klient może bezpiecznie ponowić,
  - zapytania przychodzące w trakcie wznawiania czekają w kolejce zamiast od razu zwracać błąd.
- **Blokada ekranu i zminimalizowane okno:** sprawdź, czy nie spowalniają strumieniowania odpowiedzi ani wykrywania jej końca.
- **Instalacja:** wszystko w profilu użytkownika, najlepiej bez uprawnień administratora. Uwzględnij firmowe proxy przy instalacji zależności.
- **Specyfika Windows:** uprawnienia do plików przez ACL, proces w tle bez okna konsoli, ścieżki konfiguracji i logów w `%LOCALAPPDATA%`.

### 7. Skutki uboczne na koncie M365
- **Historia czatów:** każde zapytanie może tworzyć rozmowę widoczną w historii Copilota na wszystkich urządzeniach (przeglądarka, Teams, Outlook). Porównaj strategie: usuwanie rozmów po zakończeniu, jedna rozmowa robocza z rotacją albo pozostawienie historii.
- **Pamięć i personalizacja:** jeśli w tenancie działa pamięć Copilota albo personalizacja, automatyczne zapytania mogą ją zaśmiecać, a zapamiętane preferencje mogą zmieniać wyniki zadań. Jak to sprawdzić i ograniczyć.
- **Izolacja zapytań:** kontekst jednego zadania nie może przeciekać do następnego (nowa rozmowa dla każdego zadania kontra dodatkowy czas).
- **Niedeterminizm:** brak kontroli nad temperaturą i wersją modelu, więc to samo wejście może dać różne wyniki. Jeśli interfejs pozwala wybrać tryb lub model (np. szybka odpowiedź kontra dłuższe rozumowanie), udostępnij to jako parametr.
- **Limity usługi:** ewentualne limity liczby tur w rozmowie, dzienne limity i dławienie. Jak wpływają na strategię rozmów i pętlę naprawczą.

### 8. Sterowanie stroną
- Strategia selektorów: role/aria/teksty zamiast kruchych klas CSS, wszystkie selektory w jednym pliku konfiguracyjnym, żeby łatwo je aktualizować.
- Nowa rozmowa dla każdego zapytania albo reużycie wątku (wpływ na kontekst i izolację; powiązanie z `conversation_id` w API).
- Wklejanie długich promptów (fill albo schowek zamiast wpisywania znak po znaku) i limit długości wiadomości w M365 Copilot.
- **Załączniki:** wgrywanie plików do czatu (np. `setInputFiles`) i czekanie, aż Copilot je przetworzy. Opcjonalnie odwołania do plików z OneDrive lub SharePointa.
- Elementy specyficzne dla M365: przełącznik Work/Web (grounding na danych firmowych vs. internet), wybór agenta, ewentualne dialogi zgód i komunikaty tenanta. Mają być konfigurowalne parametrem `mode` w API i flagą `--mode work|web` w CLI.
- **Wykrywanie końca odpowiedzi strumieniowanej:** zniknięcie przycisku „Stop”, stabilność tekstu przez N ms, pojawienie się przycisków akcji. Zaproponuj metodę główną i zapasową.
- Obsługa limitów, komunikatów o błędach i zmian w UI: jak je wykryć i jaki kod błędu zwrócić.

### 9. Ekstrakcja odpowiedzi
- Skąd brać tekst: `innerText` wyrenderowanego markdownu, blok kodu, przycisk „Kopiuj” czy przechwytywanie ruchu sieciowego (WebSocket/SSE). Oceń ryzyko zniekształcenia JSON-a przez renderowanie markdownu przy każdej opcji.
- Jak jednoznacznie zidentyfikować właściwą odpowiedź, np. przez unikalny `request_id` (nonce) w prompcie, który model musi odesłać.
- **Cytowania w trybie Work:** Copilot dodaje do tekstu znaczniki przypisów i listę źródeł. Mogą trafić do środka JSON-a albo go zepsuć. Zaplanuj ich usuwanie przed parsowaniem, a prawdziwe źródła (linki do plików i maili) wyciągaj z interfejsu, zamiast ufać adresom wygenerowanym przez model.
- **Ucięte odpowiedzi:** jak wykryć, że odpowiedź została przerwana (limit długości, zerwany strumień), i co wtedy zrobić: poprosić o kontynuację, podzielić zadanie albo zwrócić błąd.

### 10. Prompt wymuszający schemat (kluczowe)
Zaprojektuj szablon „koperty” promptu, wspólny dla wszystkich zadań, który:
- jasno instruuje model, żeby zwrócił **tylko** jeden blok ```json bez tekstu przed nim i po nim,
- zawiera schemat JSON (JSON Schema) albo przykład wypełnionej odpowiedzi,
- zawiera `request_id` do odesłania,
- oddziela instrukcję zadania od danych klienta delimiterami, żeby ograniczyć prompt injection z treści danych,
- określa zachowanie przy niepewności lub odmowie (pole `confidence`, `status: "refused" | "uncertain"`),
- określa język wartości tekstowych (klucze JSON zawsze bez zmian).

Zaproponuj **domyślny schemat odpowiedzi modelu** (dla `ask`), np.:
```json
{
  "request_id": "string",
  "status": "ok | uncertain | refused",
  "answer": "string",
  "key_points": ["string"],
  "confidence": 0.0,
  "sources": [{"title": "string", "url": "string"}]
}
```
Umożliw też podanie własnego schematu (parametr `schema` w API, `--schema` w CLI) oraz schematów per zadanie z rejestru.

Opisz **naprawę odpowiedzi** w dwóch krokach. Najpierw lokalnie, bez modelu: usunięcie znaczników cytowań, przecinków końcowych i tekstu wokół bloku JSON. Dopiero gdy to nie wystarczy, prompt „repair” w tej samej rozmowie z listą błędów walidacji. Ogranicz liczbę prób (np. 2) i zapisuj w logach, co się stało.

### 11. Koperta odpowiedzi (API i CLI)
Zdefiniuj stałą „kopertę” odpowiedzi niezależną od schematu modelu. Ta sama struktura wraca z API i jest wypisywana przez CLI:
```json
{
  "ok": true,
  "request_id": "uuid",
  "task": "summarize",
  "data": { },
  "raw": "surowy tekst odpowiedzi (opcjonalnie, include_raw / --include-raw)",
  "error": null,
  "meta": { "duration_ms": 0, "queue_ms": 0, "attempts": 1, "cached": false, "backend": "m365-playwright", "api_version": "v1", "schema": "summarize@1", "prompt_version": "envelope@1" }
}
```
- Kody błędów (`AUTH_REQUIRED`, `TIMEOUT`, `INTERRUPTED`, `UI_CHANGED`, `RATE_LIMITED`, `INVALID_INPUT`, `INPUT_TOO_LARGE`, `ATTACHMENT_REJECTED`, `UNKNOWN_TASK`, `INVALID_JSON`, `SCHEMA_MISMATCH`, `TRUNCATED`, `DAEMON_UNAVAILABLE`, `DAEMON_START_FAILED`, `VERSION_MISMATCH`) oraz ich mapowanie na statusy HTTP i kody wyjścia procesu CLI.
- Zasada dla CLI: stdout zawiera tylko JSON, a logi idą na stderr lub do pliku (`--verbose`, `--debug` ze zrzutami ekranu i trace'ami Playwrighta).

### 12. Wydajność, batch i cache
- **Realne oczekiwania:** jedna sesja przeglądarki obsłuży kilka zapytań na minutę, a nie setki na sekundę. Plan ma to jasno powiedzieć, a M1 ma to zmierzyć.
- **Batch:** wiele elementów (np. 50 zgłoszeń do klasyfikacji) pakowanych po kilka lub kilkanaście w jeden prompt. Wynik to tablica z identyfikatorem każdego elementu, z obsługą częściowych błędów i ponawianiem tylko nieudanych elementów. Dobierz wielkość paczki z uwagi na jakość i limit długości wiadomości.
- **Cache:** identyczne zapytanie (zadanie, wejście, schemat, tryb, wersja promptu) zwraca zapisany wynik z określonym czasem ważności. Da się go wyłączyć parametrem lub flagą, a odpowiedź ma `meta.cached`. Przyspiesza powtarzalne zapytania i oszczędza limity.
- **Priorytety w kolejce:** zapytania interaktywne przed długimi batchami.

### 13. Współbieżność i niezawodność
- Kolejka zapytań w daemonie (jedna karta albo pula kart), limit długości kolejki, timeouty globalne i per etap, anulowanie zapytania, gdy klient się rozłączy (Ctrl+C) albo wywoła `DELETE /v1/jobs/{id}`.
- Czas oczekiwania w kolejce wliczony do timeoutu klienta i raportowany w `meta`.
- Retry z backoffem tylko dla błędów przejściowych.
- Odtwarzanie po awarii przeglądarki lub karty wewnątrz działającego daemona, bez utraty sesji.
- **Trwałość stanu:** mapowanie `conversation_id` na wątki w Copilocie i joby asynchroniczne zapisywane lokalnie, żeby restart daemona albo uśpienie laptopa ich nie gubiły.

### 14. Testowanie
- Testy jednostkowe PromptBuildera, rejestru zadań, parsera JSON (ekstrakcja z fence'ów, lokalna naprawa, usuwanie znaczników cytowań) i walidatora.
- Testy kontraktowe API (w tym adaptera zgodnego z OpenAI) względem specyfikacji OpenAPI, uruchamiane na backendzie `mock`.
- Testy integracyjne cyklu życia daemona na Windowsie: auto-start, dwa równoległe starty, martwy plik PID, niezgodność wersji, symulowane uśpienie i wybudzenie.
- Testy BrowserDrivera na lokalnej atrapie strony (fixture HTML symulujący streaming), żeby nie odpytywać prawdziwego Copilota w testach automatycznych.
- **Nagrywanie i odtwarzanie:** tryb, w którym backend `playwright` zapisuje pary zapytanie → odpowiedź, a backend `mock` je odtwarza. Z nagrań powstają testy i zestaw ewaluacyjny.
- Zestaw ewaluacyjny dla zadań wbudowanych: kilka przykładów wejście → oczekiwane cechy wyniku, uruchamiany ręcznie na prawdziwym serwisie. Mierzy odsetek poprawnego JSON-a za pierwszym razem i jakość odpowiedzi.
- Wykrywanie regresji po zmianie UI Copilota (np. osobna komenda `copilot-bridge doctor`).

### 15. Bezpieczeństwo
- Limit częstotliwości zapytań, żeby usługa nie zaczęła dławić konta. Logowanie z MFA zawsze ręczne przez `copilot-bridge login`.
- Profil przeglądarki z sesją traktowany jak hasło: katalog dostępny tylko dla użytkownika, poza repozytorium, łatwy do usunięcia (`copilot-bridge logout` czyści profil).
- Co trafia do logów, maskowanie treści pytań i odpowiedzi, uprawnienia do katalogu profilu, logów i pliku z tokenem.
- Daemon słucha wyłącznie na `127.0.0.1` i wymaga tokenu, więc inne procesy i użytkownicy na maszynie nie skorzystają z sesji.
- **Ochrona lokalnego API przed stronami w przeglądarce:** walidacja nagłówka `Host` (ochrona przed DNS rebinding), żadnego luźnego CORS, a playground nie może ujawnić tokenu innym stronom.
- **Odpowiedź to niezaufane dane:** w trybie Work Copilot czyta maile i dokumenty, które mogą zawierać wstrzyknięte instrukcje. Klienci nie mogą automatycznie wykonywać akcji na podstawie odpowiedzi bez walidacji.
- **Nagrania, cache i historia zawierają dane firmowe:** gdzie leżą, jak długo są trzymane i jak je czyścić.

### 16. Interfejs użytkownika i integracje
- **Prosty playground:** strona HTML serwowana przez daemona (wybór zadania, wejście, załączniki, sformatowany JSON na wyjściu, stan daemona i sesji).
- **Przykłady integracji:** PowerShell, krótki skrypt w Pythonie/Node, gotowe SDK OpenAI wskazujące na lokalny adres.
- **Przykłady zastosowań:** 3–5 gotowych przykładów z danymi wejściowymi i oczekiwanym wynikiem (np. klasyfikacja zgłoszeń, dane z faktury do JSON-a, streszczenie wątku mailowego, szkic odpowiedzi, pytanie o wiedzę firmową w trybie Work). Służą jako dokumentacja i część zestawu ewaluacyjnego.
- **Metryki:** `/v1/metrics` z liczbą zapytań, odsetkiem poprawnego JSON-a za pierwszym razem i po naprawie, średnim i p95 czasu odpowiedzi, podziałem na zadania.

### 17. Struktura repozytorium i konfiguracja
Proponowane drzewo katalogów (w tym `openapi.yaml`, `tasks/` z definicjami zadań, `examples/` z przykładami, `fixtures/` z nagraniami, `evals/` z zestawem ewaluacyjnym), plik konfiguracyjny (URL, port, backend, ścieżka profilu Edge, selektory, timeouty, domyślny schemat, cache), zmienne środowiskowe i sposób instalacji na Windowsie.

## Oczekiwany format planu

1. Rozstrzygnięcia otwartych pytań (z rekomendacją i uzasadnieniem).
2. Architektura z diagramem (ASCII lub mermaid).
3. Pełny szablon promptu-koperty i promptu naprawczego (gotowe teksty) oraz przykładowa definicja zadania z rejestru.
4. Domyślny JSON Schema odpowiedzi, koperta odpowiedzi i szkic specyfikacji OpenAPI lokalnego API.
5. Przykłady zastosowań z danymi wejściowymi i oczekiwanym wynikiem.
6. Lista rzeczy do sprawdzenia na żywej stronie w M1 (czy sesja przeżywa dzień pracy i uśpienie laptopa, czy działa tryb headless, czy Copilot stabilnie oddaje blok JSON, ile trwa odpowiedź, jakie są limity, jak zachowują się cytowania, historia i pamięć) i jak plan się dostosuje, jeśli wynik będzie inny niż zakładany.
7. Kamienie milowe z kryteriami „done” i szacunkiem pracochłonności dla jednej osoby:
   - **M1:** `copilot-bridge login` + skrypt wysyłający jedno pytanie do M365 Copilota przez Edge → surowy tekst odpowiedzi; sprawdzenie rzeczy z punktu 6,
   - **M2:** daemon z lokalnym API (`/v1/health`, `/v1/ask`, token, walidacja `Host`) + backend `mock` + auto-start z CLI + `daemon status/stop`,
   - **M3:** prompt-koperta + parsowanie, walidacja i lokalna naprawa JSON + pętla naprawcza + rejestr zadań z 3 zadaniami wbudowanymi + nagrywanie odpowiedzi,
   - **M4:** kody błędów i statusy HTTP, stan `auth_required`, wznawianie po uśpieniu, logi, debug (trace/screenshot), kolejka, joby asynchroniczne, anulowanie, trwałość stanu,
   - **M5:** adapter zgodny z OpenAI, załączniki, batch, cache, pozostałe zadania wbudowane, rozmowy wieloturowe, metryki,
   - **M6:** playground, przykłady integracji i zastosowań, komenda `doctor`, start razem z systemem, testy na atrapie, dokumentacja, instalator lub paczka dla Windowsa.
8. Rozbicie każdego kamienia milowego na małe zadania, które można po kolei przekazać agentowi AI do implementacji: cel, pliki, kryteria akceptacji, sposób sprawdzenia.
9. Lista ryzyk technicznych z prawdopodobieństwem, wpływem i mitygacją.
10. Lista rzeczy świadomie poza zakresem.

Pisz konkretnie. Tam, gdzie decyzja zależy od informacji, której nie masz (np. aktualny DOM strony Copilota), zaznacz to wprost i zaproponuj, jak to zweryfikować.
