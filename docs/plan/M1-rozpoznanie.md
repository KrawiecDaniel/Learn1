# M1. Rozpoznanie strony M365 Copilot

**Cel:** zanim powstanie właściwy kod, sprawdzić na żywej stronie, jak zachowuje się M365 Copilot w Edge sterowanym przez Playwright. Wynik to działające selektory i odpowiedzi na listę pytań. Od nich zależą ustawienia w M5.

**Gotowe, gdy:** `selectors.yaml` zawiera działające selektory, a `docs/m1-wyniki.md` odpowiada na wszystkie pytania z listy kontrolnej.

## M1.1 `pyproject.toml` (≈60 linii)

**Wklej:** kartę projektu.

**Specyfikacja:**
- Nazwa `copilot-bridge`, wersja `0.1.0`, `requires-python = ">=3.12"`, backend budowania setuptools, układ `src/`.
- Zależności: `fastapi>=0.110`, `uvicorn>=0.29`, `pydantic>=2.6`, `pyyaml>=6`, `jsonschema>=4.21`, `httpx>=0.27`, `typer>=0.12`, `playwright>=1.49`, `filelock>=3.13`, `python-multipart>=0.0.9` (formularze z plikami w FastAPI).
- Dodatki `dev`: `pytest>=8`, `openai>=1.40`.
- Skrypt konsolowy: `copilot-bridge = "copilot_bridge.cli:app"`.
- Dane pakietu: `tasks/builtin/*.yaml`, `browser/*.yaml`, `static/*.html`.
- `[tool.pytest.ini_options]`: `testpaths = ["tests"]`, `addopts = "-q"`, `markers = ["browser: testy wymagające zainstalowanego Edge"]`.

**Sprawdzenie:** po utworzeniu `src/copilot_bridge/__init__.py` (M1.2): `pip install -e ".[dev]"` kończy się bez błędów.

## M1.2 `src/copilot_bridge/__init__.py` (wpisz ręcznie)

Jedna linia: `__version__ = "0.1.0"`. Utwórz też puste pliki `__init__.py` w `src/copilot_bridge/browser/` i `tools/` nie jest pakietem (bez `__init__.py`).

## M1.3 `src/copilot_bridge/paths.py` (≈90 linii)

**Wklej:** kartę projektu, I-paths.

**Specyfikacja:**
- Implementuj dokładnie I-paths.
- `app_dir()`: zmienna `COPILOT_BRIDGE_HOME`, jeśli ustawiona; inaczej `%LOCALAPPDATA%\copilot-bridge`; gdy `LOCALAPPDATA` nie istnieje, `Path.home() / "AppData" / "Local" / "copilot-bridge"`. Tworzy katalog (`mkdir(parents=True, exist_ok=True)`).
- Funkcje katalogów tworzą je przy każdym wywołaniu; funkcje plików tworzą tylko katalog nadrzędny.
- `ensure_token()`: czyta i zwraca token bez białych znaków; gdy plik nie istnieje albo jest pusty, zapisuje `secrets.token_urlsafe(32)` i zwraca go.

**Sprawdzenie:** `python -c "from copilot_bridge import paths; print(paths.app_dir(), paths.ensure_token())"`.

## M1.4 `src/copilot_bridge/browser/selectors.yaml` (≈80 linii)

**Wklej:** kartę projektu i tę sekcję.

**Specyfikacja:** plik YAML: klucz → lista kandydatów (selektory Playwright: CSS, `role=…[name=/…/i]`, `text=/…/i`). Na start wpisz najbardziej prawdopodobnych kandydatów dla interfejsu czatu M365 Copilot (polskiego i angielskiego) z komentarzem `# do weryfikacji w M1`. Klucze:

| Klucz | Znaczenie |
|---|---|
| `input_box` | Pole wpisywania wiadomości |
| `send_button` | Przycisk wysłania |
| `stop_button` | Przycisk zatrzymania, widoczny tylko w trakcie pisania odpowiedzi |
| `new_chat_button` | Nowa rozmowa |
| `assistant_message` | Kontener jednej odpowiedzi Copilota (ostatni = najnowszy) |
| `code_block` | Blok kodu wewnątrz odpowiedzi |
| `citation_marker` | Znacznik przypisu w tekście odpowiedzi |
| `sources_link` | Link do źródła w odpowiedzi |
| `attach_button` | Przycisk dodania pliku (otwiera okno wyboru) |
| `file_input` | `input[type=file]` |
| `attachment_ready` | Znak, że plik jest wgrany |
| `mode_work`, `mode_web` | Przełączniki trybu Work/Web |
| `error_banner` | Komunikat błędu |
| `rate_limit_banner` | Komunikat o limicie |
| `login_marker` | Element strony logowania (np. `input[name="loginfmt"]`) |
| `truncated_marker` | Znak uciętej odpowiedzi (np. przycisk „Kontynuuj”) |
| `chat_menu_button`, `delete_chat_button`, `confirm_delete_button` | Usuwanie bieżącej rozmowy |

Klucz bez znanego kandydata ma pustą listę `[]`.

## M1.5 `tools/m1_probe.py` (≈230 linii)

**Wklej:** kartę projektu, I-paths i tę sekcję.

**Specyfikacja:** samodzielny skrypt (argparse), `playwright.sync_api`, `yaml`, `copilot_bridge.paths`. Publiczne funkcje pomocnicze (importowane przez M1.6 i M1.7):

```python
DEFAULT_URL = "https://m365.cloud.microsoft/chat"
def launch(window: str = "normal") -> tuple[Playwright, BrowserContext, Page]: ...
    # launch_persistent_context(str(paths.profile_dir()), channel="msedge", headless=(window == "headless"),
    #   no_viewport=True, args: offscreen → ["--window-position=-32000,-32000"], minimized → ["--start-minimized"])
def close(pw: Playwright, ctx: BrowserContext) -> None: ...
def load_selectors() -> dict[str, list[str]]: ...     # paths.builtin_selectors_file()
def is_login_page(page: Page) -> bool: ...            # "login.microsoftonline.com" lub "login.live.com" w URL
def find_first(page: Page, candidates: list[str], timeout_ms: int = 5000) -> Locator | None: ...
```

Podkomendy:
- `login [--url]`: widoczne okno, przejście na URL, komunikat „Zaloguj się, otwórz nowy czat i naciśnij Enter tutaj”, `input()`, wypisanie URL i wyniku `is_login_page`, zamknięcie.
- `dump [--url] [--window]`: otwarcie, czekanie na Enter użytkownika (strona gotowa), zapis do `paths.debug_dir()`: `m1-aria.yaml` (`page.locator("body").aria_snapshot()`), `m1-page.html` (`page.content()`), `m1-page.png` (zrzut całej strony). Wypisanie ścieżek.
- `check-selectors [--window]`: dla każdego klucza i kandydata `locator.count()` i widoczność pierwszego trafienia; tabela na ekran i do `m1-selectors.txt`.

Wszystkie pliki z `encoding="utf-8"`. Błędy Playwrighta łapane i wypisywane czytelnie.

**Sprawdzenie:** `python tools\m1_probe.py login` otwiera Edge i pozwala się zalogować.

## M1.6 `tools/m1_ask.py` (≈280 linii)

**Wklej:** kartę projektu, I-paths, publiczne funkcje z M1.5 i szablon JSON z `03-prompty-modelu.md` razem z `DEFAULT_DECISION_SCHEMA`.

**Specyfikacja:** skrypt (argparse) importujący funkcje z `m1_probe` (ten sam katalog). Podkomendy:
- `ask TEKST [--window] [--repeat N]`:
  1. nowa rozmowa (`new_chat_button`, a gdy go brak, `page.goto(DEFAULT_URL)`),
  2. `fill` w `input_box`, klik `send_button` (albo Enter),
  3. pomiar `t_start` (pojawia się nowa `assistant_message` albo `stop_button`) i `t_done` (brak `stop_button` i tekst ostatniej odpowiedzi bez zmian przez 1,5 s; limit 180 s),
  4. zapis ostatniej odpowiedzi: `m1-reply-<n>.html` (outer HTML), `.txt` (`inner_text`), `.code.txt` (`inner_text` ostatniego `code_block`, jeśli jest),
  5. wypisanie czasów.
- `json-test [--window] [--repeat 10]`: wysyła prompt według szablonu JSON z 03 (losowy `request_id`, instrukcja: „Zdecyduj, czy zatwierdzić zwrot 349 zł złożony 12 dni po zakupie; opcje approve/reject/escalate”, schemat `DEFAULT_DECISION_SCHEMA`) i sprawdza: czy jest blok kodu, czy `json.loads` działa, czy `request_id` się zgadza, czy są wymagane pola. Na końcu podsumowanie: liczba sukcesów, średni i maksymalny czas.
- `length-test [--window]`: wysyła wiadomości o długości 5 000, 10 000, 20 000 i 40 000 znaków (tekst wypełniający + prośba o odpowiedź „OK”) i raportuje, które zostały przyjęte.
- `upload PLIK [--window]`: dodaje plik (przez `attach_button` i `page.expect_file_chooser()`, a gdy brak, przez `file_input.set_input_files`), zrzut ekranu po 5 s, pytanie „Podsumuj załączony plik w jednym zdaniu” i zapis odpowiedzi.

**Sprawdzenie:** `python tools\m1_ask.py json-test --repeat 3` kończy się podsumowaniem.

## M1.7 `tools/m1_soak.py` (≈150 linii)

**Wklej:** kartę projektu, publiczne funkcje z M1.5 i opis podkomendy `json-test` z M1.6.

**Specyfikacja:** test długotrwały. Jedna przeglądarka otwarta przez cały czas. Co `--interval-min` minut (domyślnie 10) przez `--hours` godzin (domyślnie 8) wysyła jeden test JSON i dopisuje wiersz do `debug/m1-soak.csv`: czas, przerwa od poprzedniej iteracji w sekundach (duża przerwa = laptop spał), wynik, czas odpowiedzi, błąd, czy strona logowania. Przy błędzie próbuje `page.reload()` i ponawia raz, zapisując, czy pomogło. Każdą iterację wypisuje w jednej linii.

**Sprawdzenie:** uruchom na noc albo na dzień pracy z kilkoma uśpieniami laptopa (zamknięcie klapy na 10–30 minut).

## M1.8 Procedura i wyniki

1. `python tools\m1_probe.py login`: zaloguj się.
2. `python tools\m1_probe.py dump`: zapisz strukturę strony.
3. Ustal selektory. Możesz wkleić Copilotowi fragment `m1-aria.yaml` i poprosić o selektory Playwright dla kluczy z M1.4. **Przed zrzutem otwórz nową, pustą rozmowę**, żeby w pliku nie było treści rozmów.
4. `python tools\m1_probe.py check-selectors`: popraw `selectors.yaml`, aż kluczowe selektory działają.
5. `python tools\m1_ask.py ask "Napisz jedno zdanie o pogodzie"`, a potem `json-test --repeat 10` w trybach okna `offscreen` i `headless`.
6. `length-test` i `upload` z przykładowym PDF bez danych wrażliwych.
7. Sprawdź ręcznie w Edge (profil daemona, uruchomiony przez `login`): historię rozmów, usuwanie rozmowy, ustawienia pamięci i personalizacji.
8. `m1_soak.py` na dzień pracy z uśpieniami.
9. Wypełnij `docs/m1-wyniki.md` według listy poniżej.

## Lista kontrolna `docs/m1-wyniki.md`

1. Czy Edge z osobnym profilem przechodzi SSO, MFA i Conditional Access?
2. Czy sesja przetrwała zamknięcie i ponowne uruchomienie przeglądarki? Po jakim czasie wymaga ponownego logowania?
3. Które tryby okna działają: `headless`, `offscreen`, `minimized`?
4. Tabela: klucz selektora → działający selektor.
5. Czasy odpowiedzi z `json-test`: do początku odpowiedzi i do końca (min, średnio, max).
6. Czy przycisk stop znika po zakończeniu odpowiedzi? Czy tekst się stabilizuje?
7. JSON: ile z 10 prób dało poprawny blok? Czy `code_block.inner_text()` to czysty JSON (bez numerów linii, napisów „Kopiuj”)? Czy `request_id` się zgadzał?
8. Tryb Work: jak wyglądają znaczniki cytowań? Czy trafiają do bloku kodu?
9. Czy jest przełącznik Work/Web i jakie ma selektory?
10. Czy wgrywanie pliku działa? Jakie typy i rozmiary?
11. Od jakiej długości wiadomość jest odrzucana albo ucinana?
12. Czy rozmowy pojawiają się w historii (także w Teams i Outlooku)? Jak usunąć rozmowę (kroki i selektory)?
13. Czy jest pamięć albo personalizacja Copilota? Czy została wyłączona?
14. Wyniki `m1_soak`: czy po uśpieniu strona działała od razu, po przeładowaniu, po restarcie przeglądarki, czy wymagała logowania? Czy VPN przeszkadzał?
15. Czy seria zapytań co 3 s wywołała komunikat o limicie?
16. Jak wygląda ucięta odpowiedź (np. przycisk „Kontynuuj”)?
17. Czy odpowiedzi przychodzą przy zablokowanym ekranie?

## Jak wyniki M1 zmieniają dalsze etapy

| Wynik | Zmiana |
|---|---|
| `headless` działa | `browser.window: headless` |
| `headless` nie, `offscreen` tak | zostaje `offscreen` (domyślne) |
| `offscreen` nie działa | `minimized` albo `normal` |
| `code_block` daje czysty JSON | ekstrakcja z bloku kodu (domyślna w M5) |
| Bloku kodu brak albo jest zniekształcony | ekstrakcja z pełnego tekstu (`parse_reply` znajduje `{…}`); w M5.5 dodaj wariant przez przycisk „Kopiuj” |
| Mniej niż 8 z 10 poprawnych JSON-ów | dopisz do szablonu JSON przykład wypełnionej odpowiedzi; `repair.max_attempts: 3` |
| Brak przełącznika Work/Web | `mode` ignorowany z ostrzeżeniem w logu; `copilot-web` działa jak `copilot-work` |
| Wgrywanie plików nie działa | pomiń M6.4 (`uploads.py`); załączniki poza zakresem |
| Sesja krótsza niż dzień pracy | częstsze `copilot-bridge login`; stan `auth_required` w `/v1/health` |
| Po uśpieniu wystarcza przeładowanie | domyślne `recover_after_wake` |
| Po uśpieniu potrzebny restart przeglądarki | w M5.6 `recover_after_wake` od razu robi `session.restart()` |
| Usuwanie rozmów da się zautomatyzować | można włączyć `history.cleanup: delete` |
| Limit długości N znaków | `limits.max_input_chars` = 80% N |
| Komunikaty o limicie przy serii | `limits.min_interval_s: 10` |
| Typowa odpowiedź dłuższa niż 60 s | `timeouts.response_s: 180` |
