# M8. Przepływy (lekki odpowiednik n8n)

**Cel:** proste automatyzacje z AI dla Excela, Outlooka, plików na SharePoincie i ERP w przeglądarce. Przepływ to jeden plik YAML: lista kroków z warunkami i pętlami. Nie ma edytora graficznego. Przepływy pisze i poprawia AI na podstawie **karty przepływów** (generowanej komendą `flow docs`). Każdy przepływ da się sprawdzić (`flow check`) i uruchomić na próbę (`flow run --dry-run`), zanim cokolwiek zmieni.

**Gotowe, gdy:** trzy przykładowe przepływy (M8.17) działają na prawdziwych danych, a jeden z nich uruchamia się sam według harmonogramu.

**Zależność:** przepływy wywołują AI przez API daemona (M3–M5). Resztę (Excel, Outlook, pliki, przeglądarka ERP) robią samodzielnie, w osobnym, krótko żyjącym procesie. Awaria przepływu nie wpływa na daemona.

## Założenia projektowe (lean)

- **Jeden format kroku:** `use` + `with`, opcjonalnie `id`, `if`, `on_error`, `retry`. Grupa kroków z pętlą: `foreach` + `as` + `steps`.
- **Wyrażenia w Jinja2**, bo AI zna tę składnię najlepiej. Wartość w całości w postaci `"{{ … }}"` daje obiekt (listę, liczbę), a nie tekst.
- **Krok = jeden mały plik Pythona** z listą `SPECS`. Parametry opisane modelem pydantic z opisami pól, z których powstaje karta przepływów. Nowy krok to nowy plik, bez zmian w silniku. Własne kroki można też wrzucać do `%LOCALAPPDATA%\copilot-bridge\steps\`.
- **Bez zdarzeń w czasie rzeczywistym.** Wyzwalaczem jest harmonogram Windows (co N minut albo codziennie). „Nowy mail” = odczyt nieprzeczytanych + `once_per`, który pamięta, co już przetworzono.
- **Bezpieczne domyślne ustawienia:**
  - kroki zmieniające dane są pomijane w trybie próbnym,
  - `outlook.send` działa tylko przy `allow_send: true` w przepływie,
  - dla decyzji AI zalecane są szkice zamiast wysyłki.
- **ERP w osobnym profilu Edge** (`profile-flows`), niezależnym od profilu daemona. Skrypty ERP nagrywa się `playwright codegen` i zamienia z pomocą AI na funkcję `run(page, params)`.

## M8.0 Sprawdzenie środowiska: `tools/m8_check.py` (≈180 linii)

**Wklej:** kartę projektu, I-paths i tę sekcję.

Skrypt (argparse, bez importów z reszty M8) wypisuje raport:
1. **Outlook klasyczny przez COM:** `win32com.client.Dispatch("Outlook.Application")`, `GetNamespace("MAPI").GetDefaultFolder(6)` (Skrzynka odbiorcza), liczba elementów, nazwy podfolderów. Błąd → „brak klasycznego Outlooka (nowy Outlook nie ma COM)”.
2. **Monit bezpieczeństwa Outlooka:** odczyt `SenderEmailAddress` i `Body` pierwszego maila. Jeśli Outlook pokaże okno „program próbuje uzyskać dostęp…”, zanotuj to ręcznie. To blokada dla kroków Outlooka.
3. **Plik na SharePoincie:** parametr `--sharepoint-dir` (zsynchronizowany folder biblioteki). Zapis, odczyt i usunięcie pliku testowego `.xlsx` przez `openpyxl`.
4. **Harmonogram zadań:** utworzenie i usunięcie zadania testowego `copilot-bridge\m8-test` (`schtasks /Create /XML … /TN …`, potem `/Delete /F`).
5. **Drugi profil Edge:** `launch_persistent_context(app_dir()/"profile-flows", channel="msedge")`, otwarcie `about:blank`, zamknięcie.

**Jak wynik zmienia plan:**

| Wynik | Zmiana |
|---|---|
| Klasyczny Outlook działa bez monitów | kroki `outlook.*` przez COM (M8.11) |
| Brak klasycznego Outlooka albo monity | kroki `outlook.*` przez Outlook w przeglądarce: zamiast M8.11 zrób wariant M8.11w (na końcu pliku) |
| Zapis na SharePoincie działa | `excel.*` i `file.*` na zsynchronizowanych folderach |
| `schtasks` zablokowany | uruchamianie ręczne albo z PAD; pomiń M8.14 |

## M8.1 Zależności i konfiguracja (szablon E)

1. `pyproject.toml`: dopisz `jinja2>=3.1`, `openpyxl>=3.1`, `docxtpl>=0.16`, `pywin32>=306; sys_platform == 'win32'`. Potem `pip install -e ".[dev]"`.
2. Karta projektu: od tego etapu dopisuj te cztery biblioteki do „Dozwolonych zależności”.
3. `config.py`: dodaj model i pole w `Settings`:
   ```python
   class FlowsCfg(BaseModel):
       dir: Path | None = None            # None → app_dir()/"flows"
       browser_window: Literal["normal", "minimized", "offscreen", "headless"] = "offscreen"
       body_max_chars: int = 20000        # limit treści maila przekazywanej dalej
       keep_work_dir: bool = False        # zostawić załączniki po przebiegu (diagnoza)
   flows: FlowsCfg = FlowsCfg()
   ```
4. `paths.py`: dodaj `flows_db_file()` (`app_dir()/"flows.db"`), `flows_profile_dir()` (`app_dir()/"profile-flows"`, tworzy), `scripts_dir()` (`app_dir()/"scripts"`, tworzy), `user_steps_dir()` (`app_dir()/"steps"`, tworzy), `flow_runs_dir()` (`app_dir()/"flow-runs"`, tworzy).

## Interfejsy M8

Wklejaj te bloki tak samo jak bloki z `02-kontrakty.md`.

### I-flow-model — `flows/model.py`

```python
class StepDef(BaseModel):          # model_config: extra="forbid", populate_by_name=True
    id: str | None = None          # [a-z0-9_]+, unikalne na swoim poziomie
    use: str | None = None         # nazwa kroku, np. "excel.read"
    with_: dict = Field(default_factory=dict, alias="with")
    if_: str | None = Field(None, alias="if")
    foreach: str | None = None     # wyrażenie dające listę
    as_: str = Field("item", alias="as")
    once_per: str | None = None    # klucz elementu; przetworzone elementy są pomijane w kolejnych przebiegach
    steps: list["StepDef"] | None = None
    on_error: Literal["stop", "continue", "skip_item"] = "stop"
    retry: int = Field(0, ge=0, le=5)
    # walidacja: dokładnie jedno z use / steps; steps wymaga foreach; once_per tylko z foreach;
    # with tylko przy use; skip_item tylko wewnątrz foreach (sprawdza check_flow)

class FlowDef(BaseModel):          # extra="forbid"
    name: str                      # [a-z0-9_-]+, zgodne z nazwą pliku
    description: str = ""
    allow_send: bool = False
    max_minutes: int = 60
    vars: dict = Field(default_factory=dict)
    steps: list[StepDef] = Field(min_length=1)
    source: Path | None = Field(None, exclude=True)

def flows_dir(settings: Settings) -> Path: ...
def find_flow(name: str, settings: Settings) -> Path: ...        # brak → BridgeError(NOT_FOUND)
def load_flow(path: Path) -> FlowDef: ...                         # błąd YAML/walidacji → BridgeError(INVALID_INPUT, details={"errors"})
def list_flows(settings: Settings) -> list[tuple[str, FlowDef | None, str | None]]: ...  # (nazwa, przepływ, błąd)
def check_flow(flow: FlowDef, registry: "StepRegistry") -> list[str]: ...
    # nieznane kroki; parametry bez wyrażeń walidowane modelem kroku; składnia wyrażeń (template.check_syntax);
    # zduplikowane id; skip_item poza foreach; outlook.send bez allow_send (ostrzeżenie jako błąd)
```

### I-flow-template — `flows/template.py`

```python
class TemplateError(Exception): ...
def make_env() -> SandboxedEnvironment: ...
    # StrictUndefined; filtry: tojson (ensure_ascii=False), date(fmt="%Y-%m-%d"), lower/upper/trim itd. z Jinja2
def render(value: Any, ctx: dict) -> Any: ...
    # rekurencyjnie po dict/list; str w całości "{{ wyr }}" → natywny wynik wyrażenia;
    # inny str z "{{" → tekst; inne typy bez zmian; błąd → TemplateError z wyrażeniem i komunikatem
def evaluate(expr: str, ctx: dict) -> Any: ...   # zdejmuje {{ }}, jeśli są; compile_expression
def check_syntax(value: Any) -> list[str]: ...
```

Kontekst wyrażeń: `vars`, `steps` (wyniki kroków po `id`; wewnątrz pętli wyniki bieżącej iteracji przykrywają zewnętrzne), `run` (`id`, `flow`, `dry_run`, `started_at`), `now` (datetime), zmienna pętli (`as`, domyślnie `item`), `loop` (`index` od 1, `first`, `last`, `length`).

### I-flow-steps — `flows/steps/base.py`

```python
class StepError(Exception):
    def __init__(self, message: str, *, retryable: bool = False) -> None: ...
class FlowStop(Exception): ...       # krok "stop": koniec przebiegu bez błędu
class SkipItem(Exception): ...       # krok "skip": następny element pętli

@dataclass
class StepContext:
    run_id: str
    flow: FlowDef
    settings: Settings
    dry_run: bool
    work_dir: Path                    # flow_runs_dir()/<run_id>, sprzątany po przebiegu
    vars: dict                        # zmienne przepływu (krok "set" je zmienia)
    log: Callable[[str], None]
    def client(self) -> BridgeClient: ...   # leniwie: ensure_daemon + client_from_settings

@dataclass
class StepSpec:
    name: str
    description: str
    params: type[BaseModel]           # pola z Field(description=…), extra="forbid"
    output: str                       # opis wyniku do karty
    side_effect: bool                 # True → pomijany w trybie próbnym
    run: Callable[[StepContext, BaseModel], dict]
    side_effect_param: str | None = None   # np. "writes": o skutkach decyduje parametr

class StepRegistry:
    def __init__(self) -> None: ...
    def register(self, spec: StepSpec) -> None: ...
    def load_builtin(self) -> None: ...     # pkgutil.iter_modules po copilot_bridge.flows.steps (poza base), lista SPECS
    def load_user(self, dir: Path) -> list[str]: ...   # pliki *.py z listą SPECS; zwraca błędy wczytania
    def get(self, name: str) -> StepSpec: ...          # brak → StepError("Nieznany krok …; dostępne: …")
    def names(self) -> list[str]: ...
    def docs_markdown(self) -> str: ...
        # dla każdego kroku: "### nazwa", opis, tabela parametrów (nazwa, typ, domyślna, opis), "Wynik: …",
        # "Zmienia dane: tak/nie"; kroki pogrupowane po prefiksie (ai, excel, file, outlook, browser, sterowanie)
```

### I-flow-store — `flows/store.py`

```python
class FlowStore:
    def __init__(self, db_path: Path) -> None: ...    # sqlite3, WAL, threading.Lock
    def init_schema(self) -> None: ...
    def run_start(self, run_id: str, flow: str, dry_run: bool) -> None: ...
    def run_finish(self, run_id: str, status: str, summary: dict) -> None: ...
    def step_log(self, run_id: str, path: str, item_key: str | None, status: str,
                 duration_ms: int, output: dict | None, error: str | None) -> None: ...
        # output jako JSON ucięty do 4000 znaków; przy logging.mask_content teksty > 60 znaków przez mask()
    def seen(self, flow: str, key: str) -> bool: ...
    def mark_seen(self, flow: str, key: str, run_id: str) -> None: ...
    def forget(self, flow: str, key: str | None = None) -> int: ...   # None → wszystkie klucze przepływu
    def runs(self, flow: str | None = None, limit: int = 20) -> list[dict]: ...
    def run_steps(self, run_id: str) -> list[dict]: ...
```

Tabele: `flow_runs(run_id PK, flow, dry_run, status, started_at, finished_at, summary_json)`, `flow_steps(id INTEGER PK, run_id, path, item_key, status, started_at, duration_ms, output_json, error)`, `flow_seen(flow, key, run_id, created_at, PRIMARY KEY(flow, key))`.

### I-flow-runner — `flows/runner.py`

```python
@dataclass
class RunResult:
    run_id: str
    status: Literal["ok", "failed", "stopped"]
    steps_ok: int
    steps_failed: int
    steps_skipped: int
    items_processed: int
    items_skipped: int
    error: str | None
    duration_ms: int

class FlowRunner:
    def __init__(self, settings: Settings, registry: StepRegistry, store: FlowStore,
                 echo: Callable[[str], None] | None = None) -> None: ...
    def run(self, flow: FlowDef, *, dry_run: bool = False, vars_override: dict | None = None) -> RunResult: ...
```

### I-flow-schedule — `flows/schedule.py`

```python
def task_name(flow: str) -> str: ...     # "\\copilot-bridge\\flow-<flow>"
def build_task_xml(flow: str, python_exe: Path, every_min: int | None, daily_at: str | None) -> str: ...
def schedule(flow: str, every_min: int | None = None, daily_at: str | None = None) -> None: ...
def unschedule(flow: str) -> bool: ...
def list_scheduled() -> list[dict]: ...  # [{"flow", "next_run", "status"}]
```

## Kroki etapu

### M8.2 `flows/model.py` (≈200 linii)

**Wklej:** kartę, I-errors, I-config (`FlowsCfg`), I-paths, I-flow-model, I-flow-template (`check_syntax`), I-flow-steps (`StepRegistry.get`).

- `StepDef.model_rebuild()` po definicji (model rekurencyjny).
- `load_flow`: `yaml.safe_load`, `FlowDef.model_validate`, `source = path`; nazwa w pliku musi być równa nazwie pliku.
- `check_flow`: rekurencyjnie po krokach ze ścieżką w komunikatach, np. `steps[2].steps[0] (id=decyzja): …`. Parametry kroku walidowane modelem tylko wtedy, gdy `with` nie zawiera wyrażeń `{{`; w przeciwnym razie tylko sprawdzenie, czy nie ma nieznanych kluczy.
- Utwórz ręcznie pusty `flows/__init__.py`.

### M8.3 `flows/template.py` (≈130 linii) i M8.3t `tests/test_flow_template.py` (≈120 linii)

**Wklej:** kartę, I-flow-template.

Test: tekst z wyrażeniem; całe wyrażenie zwraca listę i liczbę; zagnieżdżone słowniki i listy; nieznana zmienna → `TemplateError` z nazwą; filtr `tojson` z polskimi znakami; `evaluate` z i bez `{{ }}`; próba dostępu do `__class__` blokowana przez sandbox; `check_syntax` wykrywa niezamknięte `{{`.

### M8.4 `flows/steps/base.py` (≈230 linii)

**Wklej:** kartę, I-config, I-paths, I-client, I-launcher, I-flow-model, I-flow-steps.

- `docs_markdown`: typ pola z adnotacji (`str`, `int`, `bool`, `list[str]`, `dict`, `str | None` itd.), domyślna wartość (`wymagany`, gdy brak), opis z `Field(description=…)`.
- `load_user`: `importlib.util.spec_from_file_location` dla każdego pliku; błąd importu → wpis na liście błędów, bez przerywania.
- Utwórz ręcznie pusty `flows/steps/__init__.py`.

### M8.5 `flows/store.py` (≈170 linii) i M8.5t `tests/test_flow_store.py` (≈90 linii)

**Wklej:** kartę, I-logging (`mask`), I-flow-store.

Test: przebieg start/koniec; logi kroków w kolejności; `seen` przed i po `mark_seen`; `forget` jednego i wszystkich kluczy; ucięcie długiego wyniku.

### M8.6 `flows/runner.py` (≈330 linii)

**Wklej:** kartę, I-config, I-models (`now_iso`), I-flow-model, I-flow-template, I-flow-steps, I-flow-store, I-flow-runner.

`run(flow, dry_run, vars_override)`:
1. `run_id = "run_" + 12 znaków hex`, `work_dir`, `store.run_start`, kontekst: `vars` = `flow.vars` + `vars_override` (wartości też renderowane szablonem, raz, na starcie).
2. Wykonanie listy kroków funkcją rekurencyjną `_run_steps(steps, scope, path, item_key)`:
   - `if` → `evaluate`; fałsz → status `skipped`,
   - krok `use`: `spec = registry.get(use)`; `params = spec.params.model_validate(render(with_, ctx))`; tryb próbny i krok ze skutkami (`side_effect` albo `side_effect_param` ustawiony na prawdę) → nie wykonuje, wynik `{"dry_run": true, "would": params.model_dump()}`; inaczej `spec.run(step_ctx, params)` z ponowieniami `retry` (odstęp 10 s × numer próby, tylko przy `StepError(retryable=True)`),
   - wynik zapisywany w `scope["steps"][id]` (gdy `id`) i w `store.step_log`,
   - grupa `foreach`: lista z `evaluate`; dla każdego elementu nowy zakres (kopia `steps` z zewnątrz, zmienna `as`, `loop`); `once_per` → klucz z `render`; `store.seen` → pominięcie elementu; po udanym wykonaniu grupy i poza trybem próbnym `mark_seen`,
   - `on_error`: `stop` → przerwanie całego przebiegu ze statusem `failed`; `continue` → wynik `{"error": komunikat}` i dalej; `skip_item` → koniec bieżącego elementu (bez `mark_seen`),
   - `FlowStop` → status `stopped`; `SkipItem` → następny element,
   - przekroczenie `max_minutes` sprawdzane przed każdym krokiem → `failed`.
3. Każdy krok wypisywany przez `echo` w jednej linii: `[ok] steps[1].steps[0] ai.task (2.3 s)` albo `[BŁĄD] … komunikat`.
4. Na końcu `store.run_finish`, sprzątanie `work_dir` (chyba że `flows.keep_work_dir`), `RunResult`.
5. Wyjątki kroków inne niż `StepError`, `FlowStop`, `SkipItem`, `TemplateError`, `ValidationError`, `BridgeError` → traktowane jak `StepError` z nazwą wyjątku (logger.exception).

### M8.6t `tests/test_flow_runner.py` (≈230 linii)

**Wklej:** kartę, I-flow-model, I-flow-steps, I-flow-store, I-flow-runner, specyfikację M8.6.

W teście rejestr z krokami testowymi: `t.echo` (zwraca parametry), `t.write` (ze skutkami, zapisuje do listy), `t.fail` (rzuca `StepError`), `t.flaky` (za pierwszym razem błąd ponawialny). Przypadki: przekazywanie wyników między krokami; `if`; `foreach` z trzema elementami i dostępem do `loop.index`; `once_per`: drugi przebieg pomija elementy; `skip_item` nie zapisuje klucza; tryb próbny nie wywołuje `t.write`; `on_error: continue`; `retry` z `t.flaky`; krok `stop` kończy przebieg ze statusem `stopped`.

### M8.7 `flows/steps/control.py` (≈100 linii)

**Wklej:** kartę, I-flow-steps.

Kroki (wszystkie bez skutków): `set` (dowolne klucze z `with` trafiają do `ctx.vars`; model z `extra="allow"`), `log` (`message`; zapis przez `ctx.log`), `stop` (`reason` → `FlowStop`), `skip` (`reason` → `SkipItem`), `fail` (`message` → `StepError`).

### M8.8 `flows/steps/ai.py` (≈150 linii)

**Wklej:** kartę, I-errors, I-client, I-flow-steps, tabelę endpointów z 02, przykład koperty z 02.

- `ai.task`: parametry `task`, `input` (dict), `files` (lista ścieżek), `votes`, `threshold`, `fallback`, `mode`, `timeout_s`. Bez plików `POST /v1/tasks/{task}`; z plikami `/v1/tasks/{task}/upload`. Wynik: `{"ok", "data", "decision", "message", "needs_review", "confidence", "error", "request_id"}` (skróty z `data`).
- `ai.ask`: `question`, `context`, `schema` (nazwa albo dict), `files`, `mode`. Wynik jak wyżej.
- Koperta `ok=false` → `StepError(komunikat, retryable=error.retryable)`.
- Kroki bez skutków (wywołanie AI nie zmienia danych), więc działają też w trybie próbnym.

### M8.9 `flows/steps/excel.py` (≈260 linii) i M8.9t `tests/test_flow_excel.py` (≈140 linii)

**Wklej:** kartę, I-flow-steps.

- `excel.read`: `file`, `sheet` (domyślnie aktywny), `table` (nazwana tabela, opcjonalnie), `header_row` (1), `max_rows` (5000). Wynik `{"rows": [{nagłówek: wartość}], "count"}`; daty jako tekst ISO; puste wiersze pomijane.
- `excel.append` (skutki): `file`, `sheet`, `row` (dict) albo `rows` (lista), `create_if_missing` (False). Kolumny według nagłówków; nieznane klucze → `StepError`. Wynik `{"appended"}`.
- `excel.update` (skutki): `file`, `sheet`, `key_column`, `key`, `values` (dict). Wynik `{"updated"}`; brak wiersza → `StepError`, chyba że `upsert: true`.
- Zapis: `openpyxl.load_workbook`, zmiana, zapis do pliku tymczasowego obok i `os.replace`. `PermissionError` (plik otwarty w Excelu) → `StepError("Plik otwarty w Excelu albo zablokowany: …", retryable=True)`.
- Pliki `.xlsm` otwierane z `keep_vba=True`.

Test na plikach w `tmp_path`: odczyt z nagłówkami; dopisanie wiersza; aktualizacja po kluczu; `upsert`; nieznana kolumna → błąd; plik `.xlsx` z nazwaną tabelą.

### M8.10 `flows/steps/files.py` (≈230 linii) i M8.10t `tests/test_flow_files.py` (≈110 linii)

**Wklej:** kartę, I-flow-steps.

- `file.list`: `dir`, `pattern` (`*.pdf`), `newer_than_hours` (opcjonalnie), `recursive` (False). Wynik `{"files": [{"path", "name", "modified", "size"}]}`.
- `file.read`: `path`, `max_chars` (100000). Tekst z `.txt`, `.csv`, `.json`, `.md`. Wynik `{"text"}`.
- `file.write` (skutki): `path`, `content` (tekst) albo `json` (obiekt), `overwrite` (False), `encoding` (`utf-8`). Wynik `{"path"}`.
- `file.copy`, `file.move` (skutki): `src`, `dst`, `overwrite`. Wynik `{"path"}`.
- `word.render` (skutki): `template` (`.docx` ze znacznikami `{{ pole }}`), `output`, `data` (dict). `docxtpl.DocxTemplate`. Wynik `{"path"}`.
- Wszystkie ścieżki przez `Path(...).expanduser()`; foldery docelowe tworzone; zapisy przez plik tymczasowy i `os.replace`.
- Folder SharePointa to zwykła ścieżka do zsynchronizowanego folderu OneDrive, np. `C:/Users/jan/Firma/Zespół - Dokumenty/Raporty`.

Test: zapis i odczyt z polskimi znakami; `overwrite=False` na istniejącym pliku → błąd; kopiowanie do nowego folderu; `word.render` z prostego szablonu utworzonego w teście przez `python-docx`.

### M8.11 `flows/steps/outlook_com.py` (≈320 linii), wariant klasyczny

**Wklej:** kartę, I-flow-steps, wyniki M8.0 (nazwy folderów).

- COM w każdym kroku: `pythoncom.CoInitialize()`, `win32com.client.Dispatch("Outlook.Application")`, `GetNamespace("MAPI")`, na końcu `CoUninitialize()`.
- Ścieżka folderu: pierwszy człon to alias `inbox` (6), `sent` (5), `drafts` (16), `deleted` (3) albo nazwa skrzynki; kolejne człony to podfoldery, np. `inbox/Faktury`.
- `outlook.read`: `folder`, `unread_only` (True), `max` (20), `since_hours`, `subject_contains`, `from_contains`, `save_attachments` (True). Najnowsze najpierw (`Items.Sort("[ReceivedTime]", True)`), filtr `Restrict` dla dat i nieprzeczytanych. Załączniki zapisywane do `work_dir/<EntryID skrócony>/`. Treść: `Body` ucięty do `flows.body_max_chars`. Wynik `{"items": [{"id": EntryID, "from", "from_name", "to", "cc", "subject", "body", "received", "attachments": [ścieżki], "categories"}], "count"}`.
- `outlook.draft` (skutki): `to`, `cc`, `subject`, `body`, `html` (False), `attachments`, `reply_to_id` (opcjonalnie: `GetItemFromID(id).Reply()` albo `ReplyAll` przy `reply_all: true`, treść dopisana nad cytatem). Zapis (`Save()`), bez wysyłki. Wynik `{"id"}`.
- `outlook.send` (skutki): parametry jak `draft`; `ctx.flow.allow_send` fałsz → `StepError("Wysyłka wyłączona: ustaw allow_send: true w przepływie")`.
- `outlook.move` (skutki): `id`, `folder`. `outlook.mark` (skutki): `id`, `read` (bool, opcjonalnie), `category` (opcjonalnie, dopisywana do istniejących).
- Błędy COM (`pywintypes.com_error`) → `StepError` z opisem; przy zajętym Outlooku `retryable=True`.

Sprawdzenie ręczne (bez testów automatycznych): przepływ z `outlook.read` na folderze testowym w trybie próbnym, potem `outlook.draft` do siebie.

### M8.12 `flows/browser_profile.py` (≈120 linii)

**Wklej:** kartę, I-paths, I-config (`FlowsCfg`), I-flow-steps (`StepError`).

```python
@contextmanager
def flows_browser(settings: Settings, window: str | None = None) -> Iterator[BrowserContext]: ...
    # FileLock(app_dir()/"profile-flows.lock", timeout=120) → brak → StepError("Profil przeglądarki przepływów w użyciu")
    # sync_playwright, launch_persistent_context(flows_profile_dir(), channel="msedge", …), zamknięcie w finally
def login(settings: Settings, url: str, wait_s: float = 600) -> None: ...
    # widoczne okno na url; czeka na zamknięcie okna przez użytkownika albo wait_s
```

### M8.13 `flows/steps/browser.py` (≈170 linii)

**Wklej:** kartę, I-paths, I-flow-steps, specyfikację M8.12.

- `browser.script`: parametry `script` (nazwa `[a-z0-9_-]+`), `params` (dict), `url` (opcjonalny start), `timeout_s` (120), `writes` (True; `side_effect_param="writes"`).
- Skrypt: `scripts_dir()/<script>.py` z funkcją `run(page, params: dict) -> dict`, wczytywany przy każdym uruchomieniu (`importlib.util`), więc zmiany działają bez restartu.
- Wykonanie: `flows_browser()`, nowa karta, `page.set_default_timeout(timeout_s * 1000)`, `goto(url)` przy podanym `url`, `run(page, params)`; wynik musi być słownikiem dającym się zapisać jako JSON. Błąd → zrzut ekranu do `work_dir` i `StepError` ze ścieżką zrzutu.
- Wynik: słownik zwrócony przez skrypt.

### M8.14 `flows/schedule.py` (≈200 linii) i M8.14t `tests/test_flow_schedule.py` (≈80 linii)

**Wklej:** kartę, I-paths, I-flow-schedule.

- `build_task_xml`: XML Harmonogramu zadań (`<Task version="1.2" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">`) z:
  - wyzwalaczem `TimeTrigger` z `Repetition/Interval = PT{n}M` (co N minut) albo `CalendarTrigger` z `ScheduleByDay` (codziennie o `daily_at`),
  - `Settings`: `StartWhenAvailable=true` (nadrobienie po uśpieniu), `DisallowStartIfOnBatteries=false`, `StopIfGoingOnBatteries=false`, `MultipleInstancesPolicy=IgnoreNew`, `ExecutionTimeLimit=PT2H`,
  - `Principal` z `LogonType=InteractiveToken` (bieżący użytkownik, bez hasła i bez administratora),
  - akcją `Exec`: `Command` = `pythonw.exe` z `.venv`, `Arguments` = `-m copilot_bridge.flows run <flow>`, `WorkingDirectory` = `app_dir()`.
- `schedule`: zapis XML do pliku tymczasowego (UTF-16, bo tak oczekuje `schtasks`), `schtasks /Create /XML <plik> /TN <nazwa> /F`; błąd → `BridgeError(INVALID_INPUT, stderr)`.
- `unschedule`: `schtasks /Delete /TN <nazwa> /F`.
- `list_scheduled`: `schtasks /Query /FO CSV /V`, filtr po prefiksie nazwy.

Test (bez wywoływania `schtasks`): XML dla „co 10 minut” i „codziennie 08:00” jest poprawnym XML (parsowanie `xml.etree`) i zawiera właściwe `Interval`, `StartBoundary`, `DisallowStartIfOnBatteries=false`, ścieżkę `pythonw.exe`.

### M8.15 `flows/cli.py` (≈260 linii), `flows/__main__.py` (≈40 linii), zmiana `cli.py` (szablon E)

**Wklej:** kartę, I-errors, I-config, I-paths, I-logging, I-flow-model, I-flow-steps, I-flow-store, I-flow-runner, I-flow-schedule, specyfikację M8.12 (`login`).

`flow_app = typer.Typer()`:
- `list`: nazwa, opis, błąd wczytania, harmonogram.
- `check NAZWA | --all`: `load_flow` + `check_flow`; kod wyjścia 1 przy błędach.
- `run NAZWA [--dry-run] [--var klucz=wartość …]`: `FlowRunner.run`, podsumowanie na końcu, kod wyjścia 0/1.
- `history [NAZWA] [--limit 20]`, `show-run RUN_ID` (kroki z czasami i błędami).
- `forget NAZWA [--key]`: czyści pamięć `once_per`.
- `schedule NAZWA (--every MINUTY | --daily HH:MM)`, `unschedule NAZWA`, `scheduled`.
- `new NAZWA`: tworzy szkielet YAML z komentarzami (jeden krok `log`).
- `docs [--out PLIK]`: nagłówek karty (`flows/card_header.md` z pakietu, treść w M8.16) + `registry.docs_markdown()`; domyślnie zapis do `flows_dir()/KARTA-PRZEPLYWOW.md`.
- `browser-login URL`: logowanie w profilu przepływów (np. do ERP).

Rejestr w każdej komendzie: `load_builtin()` + `load_user(user_steps_dir())`.

`flows/__main__.py`: przy `sys.stdout is None` (uruchomienie przez `pythonw` z harmonogramu) przekierowanie do `logs_dir()/"flows.log"`; `setup_logging`; blokada `FileLock(app_dir()/f"flow-{nazwa}.lock", timeout=0)` (trwający przebieg → wyjście bez błędu); wywołanie `run`.

Zmiana `cli.py`: `app.add_typer(flow_app, name="flow")` z importem w `try` (`ModuleNotFoundError` → pominięcie).

### M8.16 `flows/card_header.md` (≈120 linii, napisz z Copilotem albo sam)

Nagłówek karty przepływów, który trafia do AI przy pisaniu i poprawianiu przepływów. Dodaj `flows/*.md` do danych pakietu w `pyproject.toml`. Treść:

````markdown
# Karta przepływów copilot-bridge

Przepływ to plik YAML w folderze przepływów. Uruchamianie: `copilot-bridge flow run NAZWA`, próba bez zmian w danych: `--dry-run`, sprawdzenie: `copilot-bridge flow check NAZWA`.

## Zasady dla AI piszącego przepływ
1. Używaj wyłącznie kroków opisanych w tej karcie i wyłącznie ich parametrów.
2. Zwracaj cały plik YAML w jednym bloku kodu.
3. Maile twórz przez `outlook.draft`. `outlook.send` tylko, gdy użytkownik wprost o to prosi (wymaga `allow_send: true`).
4. Decyzje AI z `needs_review: true` kieruj do człowieka (szkic, kategoria, wiersz „do sprawdzenia”), nie wykonuj ich automatycznie.
5. Każdy krok, którego wynik jest potem używany, musi mieć `id`.
6. Pętle nad mailami i wierszami, które mają być przetworzone raz, muszą mieć `once_per`.

## Format
```yaml
name: nazwa-przeplywu          # taka jak nazwa pliku, małe litery, cyfry, - i _
description: Co robi przepływ
allow_send: false              # true pozwala na outlook.send
max_minutes: 60
vars:                          # stałe przepływu, dostępne jako vars.nazwa
  rejestr: "C:/Users/jan/Firma/Zespół - Dokumenty/rejestr.xlsx"
steps:
  - id: maile                  # id: nazwa wyniku (steps.maile)
    use: outlook.read          # nazwa kroku z karty
    with: {folder: inbox/Faktury, unread_only: true}
  - foreach: "{{ steps.maile.items }}"   # pętla: lista z wyrażenia
    as: mail                   # nazwa elementu (domyślnie item)
    once_per: "{{ mail.id }}"  # element przetwarzany tylko raz
    steps:                     # kroki dla każdego elementu
      - id: ocena
        use: ai.task
        with: {task: decide, input: {question: "…", context: "{{ mail.body }}", options: [tak, nie]}}
      - use: outlook.mark
        if: "{{ steps.ocena.decision == 'tak' }}"
        with: {id: "{{ mail.id }}", category: "AI: tak"}
        on_error: continue     # stop (domyślnie) | continue | skip_item
        retry: 1
```

## Wyrażenia (Jinja2)
- `"{{ wyrażenie }}"` jako cała wartość daje obiekt (listę, liczbę, słownik); w środku tekstu wstawia tekst.
- Dostępne: `vars`, `steps.<id>` (wynik kroku), zmienna pętli (`as`), `loop.index`, `run.id`, `run.dry_run`, `now`.
- Filtry: `tojson`, `date('%Y-%m-%d')`, `default('…')`, `lower`, `upper`, `trim`, `length`, `join(', ')`.
- Warunek `if` to wyrażenie Jinja2, np. `steps.ocena.confidence < 0.6 or steps.ocena.needs_review`.

## Przykład
(pełny przykład `faktury-z-maila` z M8.17)
````

### M8.17 Przykładowe przepływy i skrypt ERP

Napisz z Copilotem, wklejając kartę przepływów wygenerowaną przez `copilot-bridge flow docs` i opis przepływu. To jednocześnie test, czy karta wystarcza AI.

1. **`faktury-z-maila.yaml`**: `outlook.read` (folder z fakturami) → pętla z `once_per` → `ai.task extract` z załącznikiem (numer, kwota, NIP, data) → `ai.task decide` („czy dane są kompletne i poprawne”, opcje `kompletna`/`do_sprawdzenia`) → `excel.append` do rejestru na SharePoincie → przy `do_sprawdzenia` albo `needs_review` `outlook.draft` do siebie z `message` → `outlook.mark` z kategorią.
2. **`raport-dzienny.yaml`**: `excel.read` (dzisiejsze wpisy) → `ai.task summarize` → `word.render` z szablonu → `file.copy` do folderu SharePointa → `outlook.draft` z raportem w załączniku. Harmonogram: `flow schedule raport-dzienny --daily 16:30`.
3. **`erp-statusy.yaml`**: `excel.read` (lista numerów zamówień) → pętla → `browser.script erp_status` (`writes: false`) → `ai.task decide` (priorytet `pilne`/`normalne`) → `excel.update` (status i priorytet).
4. **`scripts/erp_status.py`**: najpierw nagranie `playwright codegen --channel msedge --user-data-dir "%LOCALAPPDATA%\copilot-bridge\profile-flows" <adres ERP>` (wcześniej `copilot-bridge flow browser-login <adres ERP>`), potem prompt do Copilota: „zamień nagrany kod na funkcję `run(page, params) -> dict`, numer zamówienia z `params['numer']`, zwróć `{"status", "data_dostawy"}`, bez `browser.close()` i bez logowania”.

### M8.18 Sprawdzenie końcowe

1. `python -m pytest` zielony (testy M8).
2. `copilot-bridge flow check --all` bez błędów.
3. Każdy przykład najpierw `--dry-run`, potem normalnie.
4. `faktury-z-maila` drugi raz: maile już przetworzone są pomijane.
5. `copilot-bridge flow schedule faktury-z-maila --every 15`, uśpienie laptopa, po wybudzeniu `flow history` pokazuje nadrobiony przebieg.
6. Poproś Copilota o nowy przepływ, wklejając tylko kartę i opis słowny. Przepływ przechodzi `check` i `--dry-run` najwyżej po jednej poprawce.

## Wariant M8.11w: Outlook w przeglądarce

Tylko gdy M8.0 wykazał brak klasycznego Outlooka albo monity bezpieczeństwa. Kroki `outlook.*` mają te same nazwy i parametry, ale działają przez `browser.script` na `https://outlook.office.com` w profilu przepływów:
- najpierw rozpoznanie jak w M1 (nagranie `codegen`, selektory listy maili, otwierania wiadomości, pobierania załączników, tworzenia szkicu),
- plik `flows/steps/outlook_web.py` (≈350 linii) z tymi samymi `SPECS` co M8.11,
- identyfikator maila: z adresu URL wiadomości,
- ten wariant jest wyraźnie bardziej kruchy i wolniejszy niż COM, więc zacznij od `outlook.read` i `outlook.draft`.
