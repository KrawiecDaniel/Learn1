# 01. Architektura i rozstrzygnięcia

## Rozstrzygnięcia otwartych pytań

### 1. Biblioteki i dystrybucja

| Rola | Wybór | Dlaczego |
|---|---|---|
| Serwer HTTP | FastAPI + uvicorn | Modele pydantic są jednocześnie walidacją i kontraktem; FastAPI sam generuje OpenAPI |
| Modele i konfiguracja | pydantic v2 + PyYAML | Walidacja konfiguracji i ciał zapytań jednym narzędziem |
| Walidacja odpowiedzi modelu | jsonschema (Draft 2020-12) | Schematy zadań i `response_format` z OpenAI to JSON Schema |
| CLI | Typer | Krótki kod, automatyczna pomoc `--help` |
| Klient HTTP w CLI | httpx | Timeouty, multipart, jasne wyjątki |
| Przeglądarka | Playwright (sync API) z `channel="msedge"` | Zainstalowany Edge, bez pobierania Chromium |
| Blokady plików | filelock | Działa na Windowsie; blokada znika sama, gdy proces umrze |
| Stan | sqlite3 (biblioteka standardowa) | Jeden plik, bez serwera |
| Załączniki | python-multipart | Wymagany przez FastAPI do formularzy z plikami |

**Dystrybucja:** katalog projektu + `.venv` + `pip install -e ".[dev]"`. Komenda `copilot-bridge` to skrypt konsolowy w `.venv\Scripts`. Dodaj ten katalog do zmiennej PATH użytkownika (bez administratora), żeby Excel, PAD i PowerShell widziały komendę. Pojedynczy `.exe` (PyInstaller) jest poza zakresem.

**Kontrakt API:** źródłem prawdy są modele pydantic w `models.py` i `openai_compat/models.py`, pisane przed endpointami (M2, M4). FastAPI generuje z nich OpenAPI, a `copilot-bridge openapi --out openapi.json` zapisuje specyfikację do repozytorium.

### 2. Model wątków

- **Wątek główny:** uvicorn + FastAPI (asyncio). Nie dotyka przeglądarki.
- **Wątek roboczy (`Worker`):** jedyny właściciel Playwrighta. Obsługuje priorytetową kolejkę zadań jedno po drugim. Playwright sync API nie jest bezpieczny wątkowo, więc przeglądarki dotyka wyłącznie ten wątek.
- **Wątek heartbeat (`WakeDetector`):** co kilka sekund zapisuje czas. Duża przerwa oznacza wybudzenie laptopa.
- Handlery HTTP wrzucają `WorkItem` do kolejki i czekają na `Future` przez `asyncio.wrap_future`, sprawdzając co sekundę, czy klient się nie rozłączył.

Dlaczego nie async Playwright w pętli uvicorna: na Windowsie uvicorn w niektórych trybach przełącza pętlę asyncio na `SelectorEventLoop`, a Playwright potrzebuje `ProactorEventLoop` do uruchamiania procesów. Osobny wątek z własną pętlą eliminuje ten problem i izoluje awarie przeglądarki od serwera.

### 3. Historia rozmowy w adapterze OpenAI

**Każde wywołanie `/v1/chat/completions` to nowa rozmowa w Copilocie**, a cała historia z `messages` jest spłaszczana do jednej wiadomości (format w `03-prompty-modelu.md`).

- Zalety: pełna izolacja wywołań, brak kruchego dopasowywania wątków, zgodność z bezstanowym modelem OpenAI.
- Koszt: przy długiej historii rośnie prompt. Limit `limits.max_input_chars` kończy się błędem `INPUT_TOO_LARGE`.
- Rozmowy stanowe są dostępne osobno przez `/v1/conversations` (M6).

### 4. Excel

- **Rekomendowane: Power Query.** `Json.Document(Web.Contents(url, [Headers=…, Content=…, Timeout=#duration(0,0,5,0)]), 65001)` sam parsuje JSON.
- **VBA:** `MSXML2.ServerXMLHTTP.6.0` z `setTimeouts` i parametrem `?format=text` (proste linie `klucz=wartość`, bez parsera JSON w VBA).
- **Funkcja `WEBSERVICE`: odradzana.** Obsługuje tylko GET i blokuje arkusz na czas odpowiedzi.
- Dodatkowy endpoint nie jest potrzebny. Zamiast niego jest parametr `format=json|flat|text` przy `/v1/ask` i `/v1/tasks/{name}`.

### 5. Pozostałe decyzje

| Temat | Decyzja |
|---|---|
| Okno przeglądarki | Domyślnie `offscreen` (widoczne okno przesunięte poza ekran), bo tryb headless bywa blokowany przez SSO. M1 może to zmienić. |
| Historia czatów w Copilocie | Domyślnie `keep`. Tryb `delete` (usuwanie rozmowy po zadaniu) włączany po M1, jeśli da się go niezawodnie zautomatyzować. |
| Wykrywanie wybudzenia | Wątek heartbeat porównujący czas ścienny; bez API zasilania Windows. |
| Stan | SQLite `state.db` w katalogu danych. |
| Strumieniowanie | Brak. `stream: true` w adapterze OpenAI zwraca błąd 400. |
| Uwierzytelnienie | Token Bearer w pliku `token` w katalogu danych. `%LOCALAPPDATA%` jest domyślnie dostępny tylko dla użytkownika, więc nie trzeba ustawiać ACL ręcznie. |
| CORS | Brak middleware CORS. Przeglądarki nie odczytają odpowiedzi z obcych stron. |
| Jedna karta | Zadania wykonywane pojedynczo. Pula kart jest poza zakresem. |

## Diagram

```text
 Klienci                            Daemon (jeden proces pythonw.exe)
┌───────────────┐  HTTP 127.0.0.1   ┌─────────────────────────────────────────────────────┐
│ CLI           │ ────────────────▶ │ Wątek główny: uvicorn + FastAPI                     │
│ skrypty Python│  Bearer token     │   middleware: Host, token, X-Request-Id             │
│ biblioteka    │                   │   /v1/ask  /v1/tasks  /v1/jobs  /v1/chat/completions│
│   openai      │ ◀──────────────── │        │ WorkItem + Future                          │
│ Excel / PAD   │  koperta JSON     │        ▼                                            │
└───────────────┘                   │ Wątek roboczy (Worker, kolejka priorytetowa)        │
                                    │   TaskRunner ─▶ Engine ─▶ Backend                   │
                                    │                             ├─ PlaywrightBackend ───┼──▶ Edge (osobny profil) ──▶ M365 Copilot
                                    │                             └─ MockBackend          │
                                    │ Wątek heartbeat: WakeDetector                       │
                                    │ %LOCALAPPDATA%\copilot-bridge: state.db, logi, token│
                                    └─────────────────────────────────────────────────────┘
```

## Przepływ zapytania o decyzję

1. Klient: `POST /v1/tasks/decide` z `{"input": {"question", "context", "options"}, "votes": 1}`.
2. Middleware: sprawdza nagłówek `Host`, token, nadaje `request_id`.
3. Handler tworzy `TaskJob`, pakuje go w `WorkItem(priority=0)` i czeka na wynik z timeoutem. Rozłączenie klienta anuluje zadanie.
4. Worker: `TaskRunner.run(job)`:
   1. `TaskRegistry.render()`: walidacja wejścia, instrukcja z opcjami, schemat wyjścia z `enum` z opcji,
   2. cache (jeśli włączony i brak załączników),
   3. `Engine.complete()` raz albo N razy przy głosowaniu.
5. Engine: `prompts.build_json_prompt()` → `backend.send()` → `json_extract.parse_reply()` → `validation.validate()` → ewentualnie `build_repair_prompt()` i ponowne `send()` w tej samej rozmowie → `CompletionResult`.
6. TaskRunner: próg pewności, decyzja zastępcza, większość głosów → dziennik decyzji, metryki, cache → `Envelope`.
7. API: koperta jako JSON (albo `format=flat|text`), status HTTP według kodu błędu.

## Wątki i procesy

| Element | Gdzie działa | Dotyka przeglądarki |
|---|---|---|
| uvicorn, FastAPI, middleware | wątek główny | nie |
| Worker, TaskRunner, Engine, Backend | wątek roboczy | tak (jedyny) |
| WakeDetector | wątek heartbeat | nie |
| Edge | osobne procesy uruchomione przez Playwright | – |
| CLI | osobny, krótki proces | nie |

## Struktura repozytorium

```text
copilot-bridge/
  pyproject.toml                              M1
  README.md                                   M7
  openapi.json                                M7 (eksport)
  tools/
    m1_probe.py                               M1
    m1_soak.py                                M1
  docs/
    m1-wyniki.md                              M1 (wypełniasz sam)
  src/copilot_bridge/
    __init__.py                               M1
    paths.py                                  M1
    errors.py                                 M2
    config.py                                 M2
    logging_setup.py                          M2
    models.py                                 M2
    schemas.py                                M2
    json_extract.py                           M2
    validation.py                             M2
    prompts.py                                M2
    tool_calls.py                             M2
    engine.py                                 M2
    storage.py                                M2
    metrics.py                                M2
    task_runner.py                            M2
    worker.py                                 M3
    batch.py                                  M6
    client.py                                 M3
    launcher.py                               M3
    cli.py                                    M3
    cli_admin.py                              M3
    doctor.py                                 M7
    evals.py                                  M7
    autostart.py                              M7
    tasks/
      __init__.py                             M2
      registry.py                             M2
      builtin/decide.yaml, ask.yaml           M2
      builtin/summarize.yaml … work_search.yaml M6
    backends/
      __init__.py (make_backend)              M2
      base.py                                 M2
      mock.py                                 M2
      playwright_backend.py                   M5
      recorder.py                             M5
    browser/
      __init__.py                             M5
      selectors.yaml                          M1
      selectors.py                            M5
      session.py                              M5
      chat.py                                 M5
      waiter.py                               M5
      extractor.py                            M5
      wake.py                                 M5
    api/
      __init__.py                             M3
      deps.py                                 M3
      app.py                                  M3
      routes_core.py                          M3
      routes_jobs.py                          M6
      uploads.py                              M6
    openai_compat/
      __init__.py                             M4
      models.py                               M4
      convert.py                              M4
      routes.py                               M4
    daemon/
      __init__.py                             M3
      __main__.py                             M3
      main.py                                 M3
    static/
      playground.html                         M7
  tests/
    conftest.py                               M3
    test_*.py                                 M2–M7
    fake_copilot/index.html                   M7
    fake_selectors.yaml                       M7
  examples/                                   M4, M7
  evals/decide_cases.yaml                     M7
```

Pliki `__init__.py` w podkatalogach są puste. Wyjątki: `copilot_bridge/__init__.py` (wersja) i `backends/__init__.py` (funkcja `make_backend`, krok M2.29).

## Katalog danych

`%LOCALAPPDATA%\copilot-bridge\` (albo katalog ze zmiennej `COPILOT_BRIDGE_HOME`, używanej w testach):

| Element | Zawartość |
|---|---|
| `config.yaml` | Konfiguracja (tworzona przez `copilot-bridge config init`) |
| `token` | Token Bearer |
| `daemon.json` | pid, port, wersja, czas startu działającego daemona |
| `daemon.lock`, `spawn.lock` | Blokady (filelock) |
| `state.db` | SQLite: zapytania, decyzje, joby, cache, rozmowy, idempotencja |
| `logs\` | `daemon.log` z rotacją, `daemon.stdout.log` |
| `profile\` | Profil Edge z sesją M365 (traktuj jak hasło) |
| `recordings\` | Nagrania zapytań i odpowiedzi (gdy włączone) |
| `debug\` | Zrzuty ekranu i HTML przy błędach |
| `uploads\` | Tymczasowe załączniki (sprzątane po zadaniu) |
| `tasks\` | Własne zadania użytkownika (nadpisują wbudowane) |

## Konfiguracja `config.yaml`

Wszystkie klucze mają wartości domyślne. Plik może zawierać tylko to, co zmieniasz.

```yaml
backend: playwright          # playwright | mock
mock_fixtures_dir: null      # katalog nagrań dla backendu mock

server:
  host: 127.0.0.1
  port: 8765

browser:
  channel: msedge
  url: https://m365.cloud.microsoft/chat
  window: offscreen          # normal | minimized | offscreen | headless
  selectors_file: null       # plik nadpisujący wbudowane selectors.yaml
  slow_mo_ms: 0
  refresh_every_min: 240     # okresowe przeładowanie karty w bezczynności

timeouts:
  response_s: 120            # maks. czas jednej odpowiedzi Copilota
  reply_start_s: 30          # maks. czas do pojawienia się odpowiedzi
  stable_ms: 1500            # tekst bez zmian przez tyle ms = koniec odpowiedzi
  page_load_s: 60
  login_wait_s: 600
  request_total_s: 300       # domyślny łączny limit zapytania (z kolejką i naprawami)

limits:
  min_interval_s: 3          # minimalny odstęp między wiadomościami do Copilota
  max_queue: 50
  max_input_chars: 30000
  max_file_mb: 20
  max_files: 5

repair:
  max_attempts: 2

cache:
  enabled: true
  ttl_s: 3600

decide:
  default_threshold: 0.6
  max_votes: 5

history:
  cleanup: keep              # keep | delete

wake:
  heartbeat_s: 5
  gap_s: 30
  resume_timeout_s: 300
  retry_interrupted: true

logging:
  level: INFO
  mask_content: true

recording:
  enabled: false
```

## Pułapki Windows (do pilnowania w każdym kroku)

1. **`os.kill(pid, 0)` zabija proces** na Windowsie. Do sprawdzania, czy daemon żyje, służy blokada `daemon.lock`: jeśli da się ją przejąć, daemona nie ma.
2. **`pythonw.exe` nie ma stdout ani stderr** (`sys.stdout is None`). Daemon na starcie przekierowuje je do `logs\daemon.stdout.log`, inaczej pierwszy zapis uvicorna go wywróci.
3. **Pętla asyncio:** daemon ustawia `asyncio.WindowsProactorEventLoopPolicy()` przed startem uvicorna. uvicorn uruchamiany bez `reload` i z `workers=1`.
4. **Uruchamianie w tle:** `pythonw.exe -m copilot_bridge.daemon` z flagami `DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP` i `CREATE_BREAKAWAY_FROM_JOB`. Jeśli breakaway zwróci `OSError` (zadanie nadrzędne nie pozwala), ponów bez tej flagi. Bez breakaway daemon może zginąć razem z terminalem VS Code albo przepływem PAD.
5. **Kodowanie znaków:**
   - odpowiedzi API mają `Content-Type: application/json; charset=utf-8`; bez `charset` PowerShell 5.1 psuje polskie znaki,
   - CLI na starcie robi `sys.stdout.reconfigure(encoding="utf-8")`,
   - w PowerShellu 5.1 ciało zapytania wysyłaj jako bajty UTF-8 (przykład w M7).
6. **Profil Edge w użyciu:** katalog `profile\` używa wyłącznie daemon. Nie otwieraj go ręcznie w Edge.
7. **OneDrive:** projekt i `.venv` poza synchronizowanymi folderami.
8. **Ścieżki ze spacjami i polskimi znakami:** zawsze `pathlib`, przy wywołaniach procesów lista argumentów, nigdy jeden string.
9. **Zamykanie Edge:** `taskkill /PID <pid> /T /F` zabija daemona razem z procesami Edge. Używać tylko awaryjnie, gdy `/v1/admin/shutdown` nie zadziała.
