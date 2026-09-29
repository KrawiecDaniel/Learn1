# M7. Wykończenie: playground, doctor, ewaluacja, autostart, przykłady

**Cel:** narzędzie gotowe do codziennej pracy: diagnostyka, pomiar jakości decyzji, start razem z Windowsem, testy przeglądarki na atrapie strony, przykłady integracji z Pythonem, Excelem, Power Automate Desktop i PowerShellem oraz dokumentacja.

**Gotowe, gdy:** `copilot-bridge doctor` przechodzi wszystkie sprawdzenia, ewaluacja daje raport, a każdy przykład integracji działa na Twoim laptopie.

## M7.1 `static/playground.html` (≈300 linii)

**Wklej:** kartę, tabelę endpointów i formatów z 02, przykład koperty z 02.

Jedna strona HTML z wbudowanym CSS i JS, bez zewnętrznych bibliotek i bez CDN:
- pole tokenu (zapamiętywany w `sessionStorage`, nigdy wpisany w HTML przez serwer),
- stan daemona (`GET /v1/health` co 10 s),
- wybór zadania (`GET /v1/tasks`), edytor JSON wejścia wypełniany przykładem z `input_schema`, pola `votes`, `threshold`, `mode`, opcjonalne pliki (`/upload`),
- przycisk „Wyślij”, czas oczekiwania, wynik: decyzja i wiadomość wyróżnione na górze, pełna koperta sformatowana poniżej,
- zakładka „Dziennik decyzji” (`GET /v1/decisions`) i „Metryki” (`GET /v1/metrics`),
- wszystkie zapytania z nagłówkiem `Authorization`; teksty z odpowiedzi wstawiane przez `textContent` (bez `innerHTML`).

`app.py` (M3.4) serwuje ten plik pod `/` bez zmian w kodzie.

## M7.2 `doctor.py` (≈250 linii)

**Wklej:** kartę, I-errors, I-paths, I-config, I-client, I-launcher, I-browser (części `selectors.py`, `session.py`).

```python
def run_doctor(settings: Settings) -> list[dict]: ...   # [{"check", "ok", "detail"}]
```

Sprawdzenia po kolei (każde niezależne, wyjątki zamieniane na `ok=False` z opisem):
1. konfiguracja wczytuje się bez błędów,
2. katalog danych zapisywalny, token istnieje,
3. Edge zainstalowany (typowe ścieżki `Program Files (x86)\Microsoft\Edge\Application\msedge.exe` i `Program Files\...`),
4. daemon odpowiada (`ensure_daemon`), wersja zgodna,
5. stan sesji: `ready` (inny stan → wskazówka, np. „uruchom copilot-bridge login”),
6. próbne zapytanie `ask` z krótkim pytaniem („Odpowiedz słowem OK”) → `ok: true` i czas odpowiedzi,
7. próbna decyzja z dwiema opcjami → decyzja z listy,
8. adapter OpenAI: `GET /v1/models`.

`cli_admin.py` wypisuje wynik jako tabelę (✔/✘) i kończy się kodem 1, gdy któreś sprawdzenie nie przeszło. Sprawdzenia 6–7 wysyłają prawdziwe zapytania, więc mają flagę `--quick`, która je pomija.

## M7.3 `evals.py` (≈250 linii)

**Wklej:** kartę, I-errors, I-config, I-client, I-launcher, format pliku z M7.4.

```python
def run_eval(settings: Settings, cases_file: Path, repeat: int = 1) -> dict: ...
```

- Czyta przypadki z YAML, każdy wysyła `repeat` razy przez API (`use_cache=false`).
- Raport: dla każdego przypadku odsetek trafień `expected_decision` (albo zgodności z `expected_any`), zgodność między powtórzeniami, średnia `confidence`, odsetek `needs_review`, czasy; globalnie: trafność, odsetek poprawnego JSON-a za pierwszym razem (`meta.first_try_valid`), p95 czasu.
- Zapis raportu do `app_dir() / "evals" / <czas>.json` i skrócona tabela na ekran (przez `cli_admin`).

## M7.4 `evals/decide_cases.yaml` (napisz sam, wzór poniżej)

```yaml
- id: zwrot-w-terminie
  task: decide
  input:
    question: Czy zaakceptować zwrot?
    context: "Kwota 349 zł, 12 dni od zakupu, klient bez wcześniejszych zwrotów."
    options: [approve, reject, escalate]
    criteria: "Zwroty do 30 dni. Kwoty powyżej 1000 zł zawsze escalate."
  expected_decision: approve
- id: zwrot-duza-kwota
  task: decide
  input:
    question: Czy zaakceptować zwrot?
    context: "Kwota 2400 zł, 5 dni od zakupu."
    options: [approve, reject, escalate]
    criteria: "Zwroty do 30 dni. Kwoty powyżej 1000 zł zawsze escalate."
  expected_decision: escalate
```

Przygotuj 15–30 przypadków z Twoich prawdziwych, zanonimizowanych sytuacji decyzyjnych, także trudnych i granicznych. Plik zostaje tylko na firmowym laptopie.

## M7.5 `autostart.py` (≈110 linii)

**Wklej:** kartę, I-paths.

```python
def startup_shortcut() -> Path: ...     # %APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\copilot-bridge.lnk
def enable() -> Path: ...
def disable() -> bool: ...
def status() -> dict: ...               # {"enabled": bool, "path": str, "target": str | None}
```

`enable()` tworzy skrót przez PowerShell (`subprocess.run(["powershell", "-NoProfile", "-Command", skrypt])`): `WScript.Shell.CreateShortcut`, `TargetPath` = `pythonw.exe` z `.venv`, `Arguments` = `-m copilot_bridge.daemon`, `WorkingDirectory` = `app_dir()`, `WindowStyle` = 7 (zminimalizowane). Ścieżki przekazywane jako osobne argumenty skryptu, nie sklejane w tekst polecenia. Nie wymaga uprawnień administratora.

## M7.6 `tests/fake_copilot/index.html` (≈200 linii)

**Wklej:** kartę, tabelę kluczy selektorów z M1.4, specyfikację `wait_for_reply` (M5.4) i `extract_reply` (M5.5).

Atrapa czatu do testów przeglądarki: pole tekstowe, przycisk „Wyślij”, przycisk „Stop” widoczny w trakcie odpowiedzi, lista wiadomości. Po wysłaniu skrypt JS:
1. wyciąga `req_[0-9a-f]{20}` z treści,
2. przez ~2 s dopisuje fragmenty odpowiedzi (symulacja pisania),
3. kończy blokiem `<pre><code>` z JSON-em zgodnym z `DEFAULT_DECISION_SCHEMA` (pierwsza opcja z listy `"options"` znalezionej w treści, albo `"approve"`) oraz znacznikiem przypisu `<sup class="cite">1</sup>` poza blokiem kodu i linkiem źródła,
4. chowa przycisk „Stop”.

Jeśli treść zawiera słowo `ZEPSUJ`, pierwsza odpowiedź ma niepoprawny JSON (test pętli naprawczej).

## M7.7 `tests/fake_selectors.yaml` (≈40 linii)

Selektory dopasowane do atrapy z M7.6 (te same klucze co M1.4). Napisz go sam albo razem z M7.6 w tej samej rozmowie.

## M7.8 `tests/test_browser_fake.py` (≈180 linii)

**Wklej:** kartę, I-config, I-backend, I-browser, specyfikacje M7.6 i M5.8.

- Oznacz moduł `pytest.mark.browser`; pomijany, gdy brak Edge (`shutil.which` albo typowe ścieżki).
- `Settings` z `browser.url = Path("tests/fake_copilot/index.html").resolve().as_uri()`, `window = "headless"`, `selectors_file = tests/fake_selectors.yaml`, `limits.min_interval_s = 0`; katalog danych w `tmp_path`.

Przypadki: `PlaywrightBackend.send` zwraca tekst z JSON-em i poprawnym `request_id`; `sources` zawiera link; znacznik przypisu usunięty z `full_text`; `Engine.complete` z `ZEPSUJ` naprawia odpowiedź w drugiej próbie; anulowanie w trakcie pisania → `CANCELLED`.

## M7.9 Przykłady integracji (każdy osobnym krokiem, szablon A)

Wklej za każdym razem kartę, tabelę endpointów i formatów z 02, przykład koperty z 02 i punkt 5 z „Pułapek Windows” w 01.

| Plik | Zawartość |
|---|---|
| `examples/python/decide_requests.py` (≈80 linii) | `httpx` → `POST /v1/tasks/decide`, obsługa `ok=false`, ponowienie przy `retryable`, timeout 330 s |
| `examples/powershell/decide.ps1` (≈60 linii) | `Invoke-RestMethod` z ciałem jako bajty UTF-8 (`[Text.Encoding]::UTF8.GetBytes`), `-TimeoutSec 330`, rozgałęzienie po `data.decision` |
| `examples/excel/decide_powerquery.m` (≈50 linii) | Power Query: `Web.Contents` z `Content` (JSON jako `Text.ToBinary(..., TextEncoding.Utf8)`), nagłówki, `Timeout=#duration(0,0,5,0)`, `Json.Document(…, 65001)`, rozwinięcie pól `decision`, `message`, `confidence`, `needs_review` do tabeli |
| `examples/excel/decide_vba.bas` (≈120 linii) | VBA: funkcja `CopilotDecide(pytanie, kontekst, opcje)` przez `MSXML2.ServerXMLHTTP.6.0`, `setTimeouts 5000, 5000, 30000, 330000`, `?format=text`, parsowanie linii `klucz=wartość`, zwraca decyzję; osobna funkcja na wiadomość; token z pliku w `%LOCALAPPDATA%` |
| `examples/pad/README.md` (≈80 linii) | Power Automate Desktop krok po kroku: „Odczytaj tekst z pliku” (token), „Wywołaj usługę sieci Web” (POST, nagłówki, treść JSON, limit czasu żądania co najmniej 330 s), „Konwertuj JSON na obiekt niestandardowy”, „Jeśli” po `decision`, obsługa `ok = false` |

Przykłady mają używać fikcyjnych danych i zawierać komentarze po polsku.

## M7.10 `README.md` projektu (≈250 linii)

**Wklej:** kartę, sekcje „Przygotowanie środowiska” z README planu, „Konfiguracja” z 01, tabelę endpointów i kodów błędów z 02, listę komend z M3.11–M3.12.

Rozdziały: czym jest narzędzie; instalacja; pierwsze uruchomienie (`config init`, `login`, `doctor`); przykłady CLI; API (tabela endpointów, formaty, koperta, kody błędów); adapter OpenAI z przykładem; zadania wbudowane i dodawanie własnych (`%LOCALAPPDATA%\copilot-bridge\tasks`); konfiguracja; rozwiązywanie problemów (`AUTH_REQUIRED` → `login`, `UI_CHANGED` → `check-selectors` z M1 i aktualizacja `selectors.yaml`, daemon nie startuje → `daemon run`, logi); autostart.

## M7.11 Lista końcowa

1. `python -m pytest` (także `-m browser`) zielony.
2. `copilot-bridge doctor` bez `--quick`: wszystko ✔.
3. `copilot-bridge eval evals\decide_cases.yaml --repeat 3`: trafność i zgodność zapisane jako punkt odniesienia na przyszłość.
4. `copilot-bridge autostart enable`, restart Windowsa, `copilot-bridge daemon status` bez wcześniejszego uruchamiania ręcznego.
5. Każdy przykład z M7.9 uruchomiony na prawdziwym Copilocie.
6. `copilot-bridge openapi --out openapi.json` zapisane w repozytorium.
