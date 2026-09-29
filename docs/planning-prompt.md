# Prompt planistyczny: Copilot Bridge (uniwersalne lokalne API AI → M365 Copilot → JSON)

Jesteś doświadczonym architektem oprogramowania. Przygotuj szczegółowy plan implementacji narzędzia **copilot-bridge**. Nie pisz jeszcze kodu produkcyjnego. Twoim zadaniem jest plan, który da się potem wykonać krok po kroku.

## Cel

**Uniwersalne, lokalne API AI**, które pokazuje, co da się zrobić z AI w firmie, zanim IT udostępni oficjalne API. Pod spodem działa Microsoft 365 Copilot sterowany przez Playwright, ale dla klientów wygląda to jak zwykłe API zwracające ustrukturyzowany JSON. Dowolne wewnętrzne narzędzie albo skrypt może z niego skorzystać do różnych zadań: odpowiadania na pytania, streszczania, klasyfikacji, wyciągania danych z tekstu, generowania treści.

Całość:
1. przyjmuje zadanie od klienta (API HTTP albo CLI): pytanie wprost albo nazwane zadanie z danymi wejściowymi,
2. opakowuje je w prompt wymuszający odpowiedź w ustalonym schemacie JSON,
3. przekazuje je do stałego daemona wystawiającego lokalne API HTTP (CLI uruchamia daemona, jeśli nie działa), który przez Playwright steruje przeglądarką z zalogowaną sesją Microsoft 365 Copilot, wysyła prompt i czeka na pełną odpowiedź,
4. wyciąga odpowiedź ze strony, parsuje ją, waliduje względem schematu i w razie potrzeby naprawia,
5. zwraca poprawny JSON, który da się przetwarzać maszynowo: przez API HTTP dla własnych narzędzi i przez stdout CLI (np. dla `jq` i skryptów).

Przykład użycia:
```
copilot-bridge ask "Jakie są zalety Rusta?" --schema default --timeout 120
copilot-bridge task summarize --input ./notatka.txt
copilot-bridge task classify --input-json '{"text": "...", "labels": ["bug", "feature", "question"]}'

curl -s -H "Authorization: Bearer $(cat ~/.copilot-bridge/token)" \
  -d '{"input": {"text": "..."}}' \
  http://127.0.0.1:8765/v1/tasks/extract-invoice
```

## Decyzje już podjęte

- **Charakter projektu: PoC i pokaz możliwości AI.** Jeden użytkownik, własne konto M365, dane trafiają wyłącznie do wewnętrznych systemów firmy albo do narzędzi napisanych przez autora. Jeśli PoC przekona firmę, docelową drogą będzie oficjalne API udostępnione przez IT, a nie Playwright.
- **Uniwersalność ponad jeden przypadek użycia.** Rozwiązanie nie jest zbudowane pod jedno zadanie. Nowy przypadek użycia dodaje się konfiguracją (szablon zadania + schemat), bez zmiany kodu.
- **Docelowy serwis: Microsoft 365 Copilot** (konto firmowe, logowanie przez Entra ID, czat pod adresem w rodzaju `m365.cloud.microsoft/chat`). Zweryfikuj aktualny URL i wszystkie selektory na żywej stronie.
- **Tryb pracy: stały daemon.** Jeden długo działający proces trzyma otwartą przeglądarkę z zalogowaną sesją. CLI jest cienkim klientem. Jeśli daemon nie działa, CLI **sam go uruchamia** w tle i dopiero potem wysyła zapytanie.
- **Daemon to wierna symulacja API.** Komunikacja przez HTTP na `127.0.0.1`, kontrakt zdefiniowany najpierw (contract-first, OpenAPI), wersjonowany (`/v1`). Własne narzędzia autora wołają API bezpośrednio, CLI to tylko jeden z klientów. Z punktu widzenia klienta nie może być widać, że pod spodem jest przeglądarka. Po udostępnieniu oficjalnego API przez IT wymienia się tylko backend, a klienci zostają bez zmian.

## Pytania, które trzeba rozstrzygnąć na początku

Zanim zaproponujesz architekturę, wypisz te pytania i podaj rekomendowaną odpowiedź na każde:
- **Czy istnieje oficjalna droga** do M365 Copilota, np. API Microsoft Graph dla Copilota (Chat API / Retrieval API), która zastąpiłaby sterowanie stroną? Sprawdź aktualny status (GA/beta), wymagane licencje i uprawnienia administratora tenanta. Porównaj z Playwrightem pod kątem stabilności, zgodności z polityką firmy i kosztu utrzymania. Jeśli API jest realne, zaprojektuj warstwę `Backend` tak, żeby dało się je później podpiąć zamiast Playwrighta bez zmiany klientów.
- **Stos technologiczny:** TypeScript/Node czy Python? Uzasadnij wybór (Playwright, serwer HTTP, walidacja schematu, daemon, dystrybucja CLI).
- **Kształt API:** własny REST (zadania, joby, rozmowy) plus **adapter zgodny z formatem OpenAI** (`/v1/chat/completions` z `response_format: json_schema`). Adapter sprawia, że gotowe SDK, biblioteki i narzędzia działają od razu, co mocno wzmacnia uniwersalność pokazu. Oceń też odwzorowanie oficjalnego API Copilota w Microsoft Graph pod kątem późniejszej migracji (zweryfikuj jego aktualny kształt w dokumentacji Microsoftu).
- **Zakres PoC:** co jest minimum potrzebnym do przekonującego pokazu, a co można świadomie pominąć (np. pula kart, autostart z systemem)? Plan ma być lekki, ale z architekturą, która pozwala później podmienić Playwright na API bez zmiany klientów i kontraktu JSON.
- **Scenariusze pokazowe:** które 3–5 przypadków użycia najlepiej pokażą możliwości AI w realiach tej firmy? Zaproponuj konkretne, z danymi wejściowymi i oczekiwanym wynikiem.

## Obszary, które plan musi pokryć

### 1. Architektura i przepływ danych
Diagram albo opis warstw: Klienci (CLI, własne narzędzia, SDK zgodne z OpenAI) → HTTP API (`/v1`) → Daemon [Auth → Kolejka/Joby → TaskRegistry → PromptBuilder → **Backend** → JsonParser/Validator] → odpowiedź.

Warstwa `Backend` to interfejs z wymiennymi implementacjami:
- `playwright`: BrowserDriver + ResponseExtractor (PoC),
- `mock`: deterministyczne odpowiedzi z plików, do pracy nad klientami bez przeglądarki, do testów i jako zapasowy tryb pokazu,
- `graph` (przyszłość): oficjalne API, gdy IT je udostępni.

Określ odpowiedzialność każdej warstwy i interfejsy między nimi.

### 2. Kontrakt lokalnego API (kluczowe)
Zaprojektuj API tak, jakby było prawdziwym, publicznym serwisem:
- **Endpointy** (propozycja do dopracowania):
  - `GET /v1/health`: stan daemona i sesji (`starting`, `ready`, `auth_required`, `busy`, `degraded`), wersja, backend,
  - `POST /v1/ask`: dowolne pytanie z opcjonalnym schematem, synchronicznie z timeoutem,
  - `GET /v1/tasks`, `POST /v1/tasks/{name}`: lista i wywołanie nazwanych zadań (sekcja 3),
  - `POST /v1/jobs` + `GET /v1/jobs/{id}` + `DELETE /v1/jobs/{id}`: tryb asynchroniczny, bo odpowiedzi trwają od kilku do kilkudziesięciu sekund,
  - opcjonalnie strumieniowanie przez SSE,
  - `POST /v1/conversations` i `POST /v1/conversations/{id}/messages`: rozmowy wieloturowe mapowane na wątek w Copilocie,
  - `GET /v1/schemas`, `GET /v1/schemas/{name}`: dostępne schematy odpowiedzi,
  - `POST /v1/chat/completions`: adapter zgodny z formatem OpenAI,
  - `GET /v1/metrics`: statystyki do pokazu (sekcja 13).
- **Specyfikacja OpenAPI** jako artefakt w repozytorium, z której da się wygenerować klienta i dokumentację.
- **Uwierzytelnienie:** token Bearer generowany przy pierwszym starcie, w pliku z uprawnieniami tylko dla użytkownika.
- **Zachowanie jak prawdziwe API:** kody HTTP zmapowane na kody błędów (`401`, `404` dla nieznanego zadania, `409`, `422`, `429` z `Retry-After`, `503` przy `auth_required`, `504` przy timeoucie), nagłówek `X-Request-Id`, `Idempotency-Key` dla bezpiecznych ponowień, nagłówki limitów, jednolity format błędu.
- **Parametry zapytania:** `question` albo `input`, `schema` (nazwa albo inline JSON Schema), `mode` (`work|web`), `timeout_ms`, `conversation_id`, `include_raw`.
- **Stabilność kontraktu:** zasady wersjonowania i co jest zmianą łamiącą.

### 3. Rejestr zadań (uniwersalność)
- **Zadanie** to plik konfiguracyjny (YAML/JSON): nazwa, opis, szablon promptu z miejscami na dane, schemat wejścia, schemat wyjścia, przykłady (few-shot), domyślny `mode` i timeout.
- Dodanie nowego przypadku użycia = dodanie pliku, bez zmiany kodu i bez restartu (albo z przeładowaniem na żądanie).
- **Zadania wbudowane na start** (propozycja): `ask` (pytanie ogólne), `summarize` (streszczenie z kluczowymi punktami), `classify` (klasyfikacja do podanych etykiet z uzasadnieniem), `extract` (wyciąganie pól według schematu podanego przez klienta), `rewrite` (zmiana tonu, tłumaczenie, korekta), `draft` (szkic maila lub dokumentu), `work-search` (pytanie o dane firmowe w trybie Work).
- Walidacja wejścia przed wysłaniem do Copilota, limit rozmiaru danych wejściowych, dzielenie długich tekstów, jeśli potrzebne.
- Wersjonowanie zadań, żeby zmiana szablonu nie psuła klientów po cichu.

### 4. Cykl życia daemona (kluczowe)
- **Auto-start z CLI:** CLI próbuje się połączyć. Jeśli nie ma daemona, uruchamia go jako odłączony proces (detached, bez terminala), czeka na gotowość (`/v1/health` z timeoutem) i dopiero wtedy wysyła zapytanie.
- **Wyścigi przy starcie:** dwa równoległe wywołania CLI nie mogą uruchomić dwóch daemonów. Zaprojektuj blokadę (lockfile z `flock` / atomowe utworzenie pliku) i plik PID.
- **Wykrywanie martwego daemona:** nieaktualny plik PID, zajęty port po awarii, proces wiszący bez odpowiedzi. Jak CLI to rozpoznaje i sprząta.
- **Gotowość vs. żywotność:** daemon działa, ale sesja M365 wygasła. Health check ma zwracać stan (`starting`, `ready`, `auth_required`, `busy`, `degraded`).
- **Komendy zarządzania:** `copilot-bridge daemon start|stop|restart|status|logs`, plus `copilot-bridge login` do ręcznego (headed) logowania.
- **Zgodność wersji:** co się dzieje, gdy CLI zostało zaktualizowane, a stary daemon wciąż działa (sprawdzenie wersji API przez `/v1/health`, automatyczny restart).
- **Zasoby:** limit pamięci przeglądarki, okresowe odświeżanie karty lub kontekstu, opcjonalne wyłączanie po długiej bezczynności (i czy w ogóle, skoro ma być stały).
- **Opcjonalnie:** start razem z systemem (systemd user service / launchd / Harmonogram zadań Windows) jako uzupełnienie auto-startu z CLI.
- Logi daemona do pliku z rotacją.

### 5. Sesja i logowanie (Entra ID)
- Pierwsze logowanie w trybie headed (użytkownik loguje się ręcznie, łącznie z MFA), potem trwały profil (`launchPersistentContext`). Daemon po zalogowaniu może pracować headless albo w zminimalizowanym oknie. Oceń, czy M365 działa poprawnie w trybie headless.
- Specyfika firmowa: SSO, Conditional Access, zgodność urządzenia, wygasanie sesji po określonym czasie. Jak daemon wykrywa przekierowanie na stronę logowania i przechodzi w stan `auth_required` zamiast się wieszać.
- Ścieżka odzyskania: API zwraca `503` z kodem `AUTH_REQUIRED`, a CLI dodaje instrukcję uruchomienia `copilot-bridge login`, które otwiera widoczne okno w tym samym profilu.
- Gdzie i jak bezpiecznie przechowywać profil i ciasteczka (uprawnienia tylko dla użytkownika, poza repozytorium).

### 6. Sterowanie stroną
- Strategia selektorów: role/aria/teksty zamiast kruchych klas CSS, wszystkie selektory w jednym pliku konfiguracyjnym, żeby łatwo je aktualizować.
- Nowa rozmowa dla każdego zapytania albo reużycie wątku (wpływ na kontekst i izolację; powiązanie z `conversation_id` w API).
- Wklejanie długich promptów (fill albo schowek zamiast wpisywania znak po znaku) i limit długości wiadomości w M365 Copilot.
- Elementy specyficzne dla M365: przełącznik Work/Web (grounding na danych firmowych vs. internet), wybór agenta, ewentualne dialogi zgód i komunikaty tenanta. Mają być konfigurowalne parametrem `mode` w API i flagą `--mode work|web` w CLI.
- **Wykrywanie końca odpowiedzi strumieniowanej:** zniknięcie przycisku „Stop”, stabilność tekstu przez N ms, pojawienie się przycisków akcji. Zaproponuj metodę główną i zapasową.
- Obsługa limitów, komunikatów o błędach i zmian w UI: jak je wykryć i jaki kod błędu zwrócić.

### 7. Ekstrakcja odpowiedzi
- Skąd brać tekst: `innerText` wyrenderowanego markdownu, blok kodu, przycisk „Kopiuj” czy przechwytywanie ruchu sieciowego (WebSocket/SSE). Oceń ryzyko zniekształcenia JSON-a przez renderowanie markdownu przy każdej opcji.
- Jak jednoznacznie zidentyfikować właściwą odpowiedź, np. przez unikalny `request_id` (nonce) w prompcie, który model musi odesłać.

### 8. Prompt wymuszający schemat (kluczowe)
Zaprojektuj szablon „koperty” promptu, wspólny dla wszystkich zadań, który:
- jasno instruuje model, żeby zwrócił **tylko** jeden blok ```json bez tekstu przed nim i po nim,
- zawiera schemat JSON (JSON Schema) albo przykład wypełnionej odpowiedzi,
- zawiera `request_id` do odesłania,
- oddziela instrukcję zadania od danych klienta delimiterami, żeby ograniczyć prompt injection z treści danych,
- określa zachowanie przy niepewności lub odmowie (pole `confidence`, `status: "refused" | "uncertain"`).

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

Opisz **pętlę naprawczą**: gdy JSON jest niepoprawny albo nie przechodzi walidacji, wyślij w tej samej rozmowie prompt „repair” z listą błędów walidacji. Ogranicz liczbę prób (np. 2) i zapisuj w logach, co się stało.

### 9. Koperta odpowiedzi (API i CLI)
Zdefiniuj stałą „kopertę” odpowiedzi niezależną od schematu modelu. Ta sama struktura wraca z API i jest wypisywana przez CLI:
```json
{
  "ok": true,
  "request_id": "uuid",
  "task": "summarize",
  "data": { },
  "raw": "surowy tekst odpowiedzi (opcjonalnie, include_raw / --include-raw)",
  "error": null,
  "meta": { "duration_ms": 0, "queue_ms": 0, "attempts": 1, "backend": "m365-playwright", "api_version": "v1", "schema": "summarize@1" }
}
```
- Kody błędów (`AUTH_REQUIRED`, `TIMEOUT`, `UI_CHANGED`, `RATE_LIMITED`, `INVALID_INPUT`, `UNKNOWN_TASK`, `INVALID_JSON`, `SCHEMA_MISMATCH`, `DAEMON_UNAVAILABLE`, `DAEMON_START_FAILED`, `VERSION_MISMATCH`) oraz ich mapowanie na statusy HTTP i kody wyjścia procesu CLI.
- Zasada dla CLI: stdout zawiera tylko JSON, a logi idą na stderr lub do pliku (`--verbose`, `--debug` ze zrzutami ekranu i trace'ami Playwrighta).

### 10. Współbieżność i niezawodność
- Kolejka zapytań w daemonie (jedna karta albo pula kart), limit długości kolejki, timeouty globalne i per etap, anulowanie zapytania, gdy klient się rozłączy (Ctrl+C) albo wywoła `DELETE /v1/jobs/{id}`.
- Czas oczekiwania w kolejce wliczony do timeoutu klienta i raportowany w `meta`.
- Retry z backoffem tylko dla błędów przejściowych.
- Odtwarzanie po awarii przeglądarki lub karty wewnątrz działającego daemona, bez utraty sesji.

### 11. Testowanie
- Testy jednostkowe PromptBuildera, rejestru zadań, parsera JSON (ekstrakcja z fence'ów, naprawa drobnych usterek) i walidatora.
- Testy kontraktowe API (w tym adaptera zgodnego z OpenAI) względem specyfikacji OpenAPI, uruchamiane na backendzie `mock`.
- Testy integracyjne cyklu życia daemona: auto-start, dwa równoległe starty, martwy plik PID, niezgodność wersji.
- Testy BrowserDrivera na lokalnej atrapie strony (fixture HTML symulujący streaming), żeby nie odpytywać prawdziwego Copilota w CI.
- Zestaw ewaluacyjny dla zadań wbudowanych: kilka przykładów wejście → oczekiwane cechy wyniku, uruchamiany ręcznie na prawdziwym serwisie. Mierzy odsetek poprawnego JSON-a za pierwszym razem i jakość odpowiedzi.
- Wykrywanie regresji po zmianie UI Copilota (np. osobna komenda `copilot-bridge doctor`).

### 12. Bezpieczeństwo i zgodność
- Zabezpieczenia na poziomie PoC: limit częstotliwości zapytań (np. minimalny odstęp między zapytaniami), brak omijania MFA i Conditional Access (logowanie zawsze ręczne), tylko jeden użytkownik.
- Profil przeglądarki z sesją traktowany jak hasło: katalog z uprawnieniami tylko dla właściciela, poza repozytorium, łatwy do usunięcia (`copilot-bridge logout` czyści profil).
- Dane firmowe: odpowiedzi w trybie Work mogą zawierać poufne treści z tenanta. Co trafia do logów, maskowanie treści pytań i odpowiedzi, uprawnienia do katalogu profilu, logów i pliku z tokenem.
- Daemon słucha wyłącznie na `127.0.0.1` i wymaga tokenu, więc inne procesy i użytkownicy na maszynie nie skorzystają z sesji.

### 13. Pokaz możliwości (showcase)
- **Scenariusze demo:** 3–5 gotowych, powtarzalnych scenariuszy na realistycznych (zanonimizowanych) danych, każdy z komendą lub skryptem do uruchomienia i opisem „co to pokazuje”. Przykłady: automatyczna klasyfikacja zgłoszeń, wyciąganie danych z dokumentu do JSON-a, streszczenie wątku mailowego, szkic odpowiedzi, pytanie o wiedzę firmową w trybie Work.
- **Przykładowe integracje:** krótki skrypt w Pythonie/Node, wywołanie przez `curl`, użycie gotowego SDK OpenAI wskazującego na lokalny adres. Pokazuje, że podpięcie AI do dowolnego narzędzia zajmuje minuty.
- **Opcjonalny prosty playground:** strona HTML serwowana przez daemona (wybór zadania, wejście, sformatowany JSON na wyjściu), żeby pokazać działanie osobom nietechnicznym.
- **Metryki:** `/v1/metrics` z liczbą zapytań, odsetkiem poprawnego JSON-a za pierwszym razem i po naprawie, średnim i p95 czasu odpowiedzi, podziałem na zadania. Liczby są argumentem w rozmowie o oficjalnym API.
- **Tryb awaryjny pokazu:** możliwość przełączenia na backend `mock` z nagranymi prawdziwymi odpowiedziami, gdyby w trakcie prezentacji sesja wygasła albo UI się zmienił.
- **Ścieżka do wersji docelowej:** krótka notatka, czego PoC dowiódł (z metrykami) i jakie API (np. Copilot w Microsoft Graph albo agent Copilot Studio) trzeba by udostępnić, żeby zastąpić warstwę Playwright. Dołącz specyfikację OpenAPI jako gotowy opis oczekiwanego interfejsu.

### 14. Struktura repozytorium i konfiguracja
Proponowane drzewo katalogów (w tym `openapi.yaml`, katalog `tasks/` z definicjami zadań, `demo/` ze scenariuszami), plik konfiguracyjny (URL, port, backend, selektory, timeouty, domyślny schemat), zmienne środowiskowe i sposób instalacji CLI.

## Oczekiwany format planu

1. Rozstrzygnięcia otwartych pytań (z rekomendacją i uzasadnieniem).
2. Architektura z diagramem (ASCII lub mermaid).
3. Pełny szablon promptu-koperty i promptu naprawczego (gotowe teksty) oraz przykładowa definicja zadania z rejestru.
4. Domyślny JSON Schema odpowiedzi, koperta odpowiedzi i szkic specyfikacji OpenAPI lokalnego API.
5. Katalog scenariuszy pokazowych z danymi wejściowymi i oczekiwanym wynikiem.
6. Kamienie milowe z kryteriami „done”:
   - **M1 (spike):** `copilot-bridge login` + skrypt wysyłający jedno pytanie do M365 Copilota → surowy tekst odpowiedzi; weryfikacja selektorów i trybu headless,
   - **M2:** daemon z lokalnym API (`/v1/health`, `/v1/ask`, token) + backend `mock` + auto-start z CLI + `daemon status/stop`,
   - **M3:** prompt-koperta + parsowanie i walidacja JSON + pętla naprawcza + rejestr zadań z 3 zadaniami wbudowanymi,
   - **M4:** kody błędów i statusy HTTP, stan `auth_required`, logi, debug (trace/screenshot), kolejka, joby asynchroniczne i anulowanie,
   - **M5:** adapter zgodny z OpenAI, pozostałe zadania wbudowane, rozmowy wieloturowe, metryki,
   - **M6 (pokaz):** scenariusze demo, przykładowe integracje, opcjonalny playground, tryb awaryjny, notatka o ścieżce do wersji docelowej, komenda `doctor`, testy na atrapie, dokumentacja.
7. Lista ryzyk z prawdopodobieństwem, wpływem i mitygacją (w tym ryzyko nieudanego pokazu na żywo).
8. Lista rzeczy świadomie poza zakresem.

Pisz konkretnie. Tam, gdzie decyzja zależy od informacji, której nie masz (np. aktualny DOM strony Copilota), zaznacz to wprost i zaproponuj, jak to zweryfikować.
