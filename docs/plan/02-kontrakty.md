# 02. Kontrakty: interfejsy modułów, błędy, API

Każdy blok `I-…` wklejasz Copilotowi, gdy krok go wymienia. Bloki opisują **publiczne** nazwy i sygnatury. Implementacja może mieć dodatkowe prywatne funkcje (`_nazwa`), ale nie może zmieniać tego, co jest w blokach.

## Kody błędów i ich mapowanie

| Kod | HTTP | Kod wyjścia CLI | Ponawialny | Kiedy |
|---|---|---|---|---|
| `INVALID_INPUT` | 422 | 2 | nie | Złe dane wejściowe, zły schemat, zła konfiguracja |
| `INPUT_TOO_LARGE` | 413 | 2 | nie | Dane albo historia przekraczają `max_input_chars` |
| `ATTACHMENT_REJECTED` | 422 | 2 | nie | Zły typ, rozmiar albo liczba plików |
| `UNKNOWN_TASK` | 404 | 2 | nie | Brak zadania o tej nazwie |
| `NOT_FOUND` | 404 | 2 | nie | Brak joba, rozmowy, schematu |
| `UNAUTHORIZED` | 401 | 2 | nie | Brak lub zły token |
| `FORBIDDEN_HOST` | 403 | 2 | nie | Nagłówek `Host` spoza listy dozwolonych |
| `IDEMPOTENCY_CONFLICT` | 409 | 2 | nie | Ten sam `Idempotency-Key` z innym ciałem |
| `CANCELLED` | 409 | 4 | nie | Zadanie anulowane |
| `AUTH_REQUIRED` | 503 | 3 | nie | Sesja M365 wygasła; trzeba `copilot-bridge login` |
| `INTERRUPTED` | 503 | 4 | tak | Przerwane przez uśpienie, awarię karty albo błąd usługi |
| `TIMEOUT` | 504 | 4 | tak | Przekroczony czas |
| `QUEUE_FULL` | 429 | 6 | tak | Kolejka pełna |
| `RATE_LIMITED` | 429 | 6 | tak | Copilot zgłosił limit |
| `UI_CHANGED` | 502 | 5 | nie | Nie znaleziono elementu strony (selektory do aktualizacji) |
| `INVALID_JSON` | 502 | 7 | tak | Odpowiedź bez poprawnego JSON-a po naprawach |
| `SCHEMA_MISMATCH` | 502 | 7 | tak | JSON niezgodny ze schematem po naprawach |
| `INVALID_TOOL_CALL` | 502 | 7 | tak | Błędne wywołanie narzędzia po naprawach |
| `TRUNCATED` | 502 | 7 | tak | Odpowiedź ucięta |
| `DAEMON_UNAVAILABLE` | – | 8 | tak | Tylko CLI: daemon nie odpowiada |
| `DAEMON_START_FAILED` | – | 8 | nie | Tylko CLI: nie udało się uruchomić daemona |
| `VERSION_MISMATCH` | – | 8 | nie | Tylko CLI: inna wersja daemona |
| `INTERNAL` | 500 | 1 | nie | Nieoczekiwany wyjątek |

Dla kodów oznaczonych „–” jako `http_status` przyjmij 500.

## I-paths — `copilot_bridge/paths.py`

```python
APP_NAME = "copilot-bridge"

def app_dir() -> Path: ...            # %LOCALAPPDATA%\copilot-bridge albo COPILOT_BRIDGE_HOME; tworzy katalog
def config_file() -> Path: ...        # app_dir() / "config.yaml"
def token_file() -> Path: ...         # app_dir() / "token"
def daemon_info_file() -> Path: ...   # app_dir() / "daemon.json"
def daemon_lock_file() -> Path: ...   # app_dir() / "daemon.lock"
def spawn_lock_file() -> Path: ...    # app_dir() / "spawn.lock"
def db_file() -> Path: ...            # app_dir() / "state.db"
def logs_dir() -> Path: ...           # app_dir() / "logs" (tworzy)
def profile_dir() -> Path: ...        # app_dir() / "profile" (tworzy)
def recordings_dir() -> Path: ...     # app_dir() / "recordings" (tworzy)
def debug_dir() -> Path: ...          # app_dir() / "debug" (tworzy)
def uploads_dir() -> Path: ...        # app_dir() / "uploads" (tworzy)
def user_tasks_dir() -> Path: ...     # app_dir() / "tasks" (tworzy)
def package_dir() -> Path: ...        # katalog pakietu copilot_bridge (Path(__file__).parent)
def builtin_tasks_dir() -> Path: ...  # package_dir() / "tasks" / "builtin"
def builtin_selectors_file() -> Path: ...  # package_dir() / "browser" / "selectors.yaml"
def static_dir() -> Path: ...         # package_dir() / "static"
def ensure_token() -> str: ...        # czyta token z token_file(); gdy brak, tworzy secrets.token_urlsafe(32)
```

## I-errors — `copilot_bridge/errors.py`

```python
class ErrorCode(str, Enum):
    INVALID_INPUT = "INVALID_INPUT"; INPUT_TOO_LARGE = "INPUT_TOO_LARGE"
    ATTACHMENT_REJECTED = "ATTACHMENT_REJECTED"; UNKNOWN_TASK = "UNKNOWN_TASK"
    NOT_FOUND = "NOT_FOUND"; UNAUTHORIZED = "UNAUTHORIZED"; FORBIDDEN_HOST = "FORBIDDEN_HOST"
    IDEMPOTENCY_CONFLICT = "IDEMPOTENCY_CONFLICT"; CANCELLED = "CANCELLED"
    AUTH_REQUIRED = "AUTH_REQUIRED"; INTERRUPTED = "INTERRUPTED"; TIMEOUT = "TIMEOUT"
    QUEUE_FULL = "QUEUE_FULL"; RATE_LIMITED = "RATE_LIMITED"; UI_CHANGED = "UI_CHANGED"
    INVALID_JSON = "INVALID_JSON"; SCHEMA_MISMATCH = "SCHEMA_MISMATCH"
    INVALID_TOOL_CALL = "INVALID_TOOL_CALL"; TRUNCATED = "TRUNCATED"
    DAEMON_UNAVAILABLE = "DAEMON_UNAVAILABLE"; DAEMON_START_FAILED = "DAEMON_START_FAILED"
    VERSION_MISMATCH = "VERSION_MISMATCH"; INTERNAL = "INTERNAL"

@dataclass(frozen=True)
class ErrorInfo:
    http_status: int
    exit_code: int
    retryable: bool

ERROR_TABLE: dict[ErrorCode, ErrorInfo]   # wartości z tabeli „Kody błędów” (kody CLI-only: http 500)

class BridgeError(Exception):
    def __init__(self, code: ErrorCode, message: str, *, details: dict | None = None,
                 retry_after_s: int | None = None) -> None: ...
    code: ErrorCode
    message: str
    details: dict                 # {} gdy None
    retry_after_s: int | None
    @property
    def http_status(self) -> int: ...
    @property
    def exit_code(self) -> int: ...
    @property
    def retryable(self) -> bool: ...
    def to_dict(self) -> dict: ...   # {"code": str, "message": str, "retryable": bool, "details": dict}

def info(code: ErrorCode | str) -> ErrorInfo: ...   # przyjmuje też nazwę kodu jako str; nieznany → INTERNAL
```

## I-config — `copilot_bridge/config.py`

```python
Mode = Literal["work", "web"]            # zdefiniowany tutaj, importowany przez inne moduły

class ServerCfg(BaseModel):   host: str = "127.0.0.1"; port: int = 8765
class BrowserCfg(BaseModel):
    channel: str = "msedge"
    url: str = "https://m365.cloud.microsoft/chat"
    window: Literal["normal", "minimized", "offscreen", "headless"] = "offscreen"
    selectors_file: Path | None = None
    slow_mo_ms: int = 0
    refresh_every_min: int = 240
class TimeoutsCfg(BaseModel):
    response_s: float = 120; reply_start_s: float = 30; stable_ms: int = 1500
    page_load_s: float = 60; login_wait_s: float = 600; request_total_s: float = 300
class LimitsCfg(BaseModel):
    min_interval_s: float = 3; max_queue: int = 50; max_input_chars: int = 30000
    max_file_mb: int = 20; max_files: int = 5
class RepairCfg(BaseModel):    max_attempts: int = 2
class CacheCfg(BaseModel):     enabled: bool = True; ttl_s: int = 3600
class DecideCfg(BaseModel):    default_threshold: float = 0.6; max_votes: int = 5
class HistoryCfg(BaseModel):   cleanup: Literal["keep", "delete"] = "keep"
class WakeCfg(BaseModel):
    heartbeat_s: float = 5; gap_s: float = 30; resume_timeout_s: float = 300; retry_interrupted: bool = True
class LoggingCfg(BaseModel):   level: str = "INFO"; mask_content: bool = True
class RecordingCfg(BaseModel): enabled: bool = False

class Settings(BaseModel):
    backend: Literal["playwright", "mock"] = "playwright"
    mock_fixtures_dir: Path | None = None
    server: ServerCfg = ServerCfg(); browser: BrowserCfg = BrowserCfg()
    timeouts: TimeoutsCfg = TimeoutsCfg(); limits: LimitsCfg = LimitsCfg()
    repair: RepairCfg = RepairCfg(); cache: CacheCfg = CacheCfg(); decide: DecideCfg = DecideCfg()
    history: HistoryCfg = HistoryCfg(); wake: WakeCfg = WakeCfg()
    logging: LoggingCfg = LoggingCfg(); recording: RecordingCfg = RecordingCfg()

def load_settings(path: Path | None = None) -> Settings: ...
    # path None → paths.config_file(); brak pliku → Settings(); błędny YAML albo wartości → BridgeError(INVALID_INPUT)
def write_default_config(path: Path | None = None) -> Path: ...
    # zapisuje pełny YAML z wartościami domyślnymi (nie nadpisuje istniejącego pliku); zwraca ścieżkę
```

Wszystkie modele z `model_config = ConfigDict(extra="forbid")`, żeby literówki w YAML były błędem.

## I-logging — `copilot_bridge/logging_setup.py`

```python
def setup_logging(settings: Settings, *, to_file: bool = True, to_console: bool = False) -> None: ...
    # RotatingFileHandler logs_dir()/"daemon.log", 5 MB x 5 plików, utf-8
    # format: "%(asctime)s %(levelname)s %(name)s [%(threadName)s] %(message)s"
    # ustawia flagę maskowania z settings.logging.mask_content
def set_masking(enabled: bool) -> None: ...
def mask(text: str | None) -> str: ...
    # maskowanie włączone: pierwsze 60 znaków + "…(len=N)"; wyłączone: cały tekst; None → "<none>"
def tail_log(lines: int = 50) -> list[str]: ...   # ostatnie linie daemon.log (pusta lista, gdy brak)
```

## I-models — `copilot_bridge/models.py`

```python
# Mode importowany z copilot_bridge.config
API_VERSION = "v1"
HealthStatus = Literal["starting", "ready", "auth_required", "busy", "resuming", "degraded"]
OutputFormat = Literal["json", "flat", "text"]

def new_request_id() -> str: ...     # "req_" + 20 znaków hex
def new_job_id() -> str: ...         # "job_" + 20 znaków hex
def new_conversation_id() -> str: ...# "conv_" + 20 znaków hex
def now_iso() -> str: ...            # UTC ISO 8601 z "Z"

class ErrorBody(BaseModel):
    code: str; message: str; retryable: bool = False; details: dict = {}

class Meta(BaseModel):
    duration_ms: int = 0; queue_ms: int = 0; attempts: int = 0; first_try_valid: bool | None = None
    votes: int = 1; agreement: float | None = None; cached: bool = False; fallback_applied: bool = False
    backend: str = ""; api_version: str = API_VERSION; schema_name: str | None = None
    task_version: str | None = None; prompt_version: str = ""; sources: list[dict] = []

class Envelope(BaseModel):
    ok: bool; request_id: str; task: str | None = None
    data: dict | None = None; raw: str | None = None
    error: ErrorBody | None = None; meta: Meta = Meta()

class AskRequest(BaseModel):
    question: str = Field(min_length=1); context: str | None = None
    output_schema: str | dict | None = None       # nazwa schematu albo pełny JSON Schema
    mode: Mode | None = None; timeout_s: float | None = None
    include_raw: bool = False; use_cache: bool = True

class TaskRunRequest(BaseModel):
    input: dict; mode: Mode | None = None; timeout_s: float | None = None
    include_raw: bool = False; use_cache: bool = True
    votes: int = Field(1, ge=1, le=9); threshold: float | None = Field(None, ge=0, le=1)
    fallback_decision: str | None = None; output_schema: str | dict | None = None

class BatchItem(BaseModel):     id: str; input: dict
class BatchRequest(BaseModel):
    items: list[BatchItem] = Field(min_length=1, max_length=500)
    chunk_size: int = Field(10, ge=1, le=50); mode: Mode | None = None; timeout_s: float | None = None

class JobCreate(BaseModel):     task: str; request: TaskRunRequest; priority: int = 10
class JobInfo(BaseModel):
    job_id: str; kind: str; status: Literal["queued", "running", "done", "failed", "cancelled"]
    created_at: str; updated_at: str; result: dict | None = None

class ConversationCreate(BaseModel): mode: Mode | None = None
class ConversationInfo(BaseModel):   conversation_id: str; mode: Mode; created_at: str; turns: int = 0

class HealthResponse(BaseModel):
    status: HealthStatus; version: str; api_version: str = API_VERSION; backend: str
    queue_length: int = 0; uptime_s: int = 0; pid: int

class TaskInfo(BaseModel):
    name: str; version: int; description: str; kind: str
    input_schema: dict; output_schema: dict; default_mode: str
```

W modelach używaj `Field(default_factory=…)` dla list i słowników. Pole nazwane `schema` jest zabronione (koliduje z pydantic), dlatego `output_schema` i `schema_name`.

## I-schemas — `copilot_bridge/schemas.py`

```python
DEFAULT_DECISION_SCHEMA: dict   # treść w 03-prompty-modelu.md
TEXT_CONTENT_SCHEMA: dict       # {"request_id", "content"}; treść w 03
def ensure_request_id(schema: dict) -> dict: ...
    # zwraca głęboką kopię z properties.request_id = {"type": "string"} i "request_id" w required;
    # korzeń musi mieć "type": "object", inaczej BridgeError(INVALID_INPUT)
def tool_output_schema(tool_names: list[str], final_schema: dict | None) -> dict: ...   # treść w 03
def batch_output_schema(item_schema: dict) -> dict: ...                              # treść w 03
def named_schema(name: str) -> dict: ...     # "default" | "text"; inna nazwa → BridgeError(NOT_FOUND)
def list_named() -> dict[str, dict]: ...
def resolve_schema(ref: str | dict | None) -> dict | None: ...
    # None → None; str → named_schema(ref); dict → validation.check_schema(ref) i zwraca ref
```

## I-json — `copilot_bridge/json_extract.py`

```python
@dataclass
class ParseResult:
    obj: dict | None            # sparsowany obiekt (korzeń musi być obiektem)
    errors: list[str]           # opisy problemów, gdy obj is None
    candidate: str | None       # tekst, który próbowano sparsować
    repaired: bool              # czy użyto lokalnej naprawy
    looks_truncated: bool       # niedomknięty blok ``` albo niezbalansowane nawiasy

def strip_citations(text: str) -> str: ...
    # usuwa wyłącznie wzorce, które nie są poprawnym JSON: [^1^], 【…】, cyfry w indeksie górnym ¹²³…
def find_json_candidates(text: str) -> list[str]: ...
    # kolejność: bloki ```json, potem inne bloki ```, potem zbalansowane {…} (od najdłuższego); bez duplikatów
def local_repair(s: str) -> str: ...
    # BOM, znaki zerowej szerokości, NBSP poza stringami, przecinki końcowe przed } i ],
    # surowe znaki nowej linii wewnątrz stringów → \n; NIE zamienia cudzysłowów „” ani ' na "
def parse_reply(text: str) -> ParseResult: ...
    # strip_citations → kandydaci → json.loads; przy błędzie local_repair i ponownie json.loads
```

## I-validation — `copilot_bridge/validation.py`

```python
def validate(obj: dict, schema: dict, *, max_errors: int = 20) -> list[str]: ...
    # Draft202012Validator; komunikaty "ścieżka: opis" (ścieżka "/" dla korzenia); [] gdy poprawne
def check_schema(schema: dict) -> None: ...                  # błędny schemat → BridgeError(INVALID_INPUT)
def with_enum(schema: dict, pointer: str, values: list[str], *, nullable: bool = False) -> dict: ...
    # głęboka kopia; pod JSON Pointer (np. "/properties/decision") ustawia {"type": "string", "enum": values}
    # (z null, gdy nullable); brak ścieżki → BridgeError(INVALID_INPUT)
def get_pointer(obj: dict, pointer: str) -> object: ...      # odczyt wartości spod JSON Pointer; brak → KeyError
```

## I-prompts — `copilot_bridge/prompts.py`

```python
ENVELOPE_VERSION = "envelope@1"
def build_json_prompt(request_id: str, instructions: str, data: str, schema: dict, *, language: str = "polskim") -> str: ...
def build_tools_prompt(request_id: str, instructions: str, data: str, tools: list[dict],
                       tool_choice: str | dict, allow_parallel: bool, final_schema: dict | None) -> str: ...
def build_batch_prompt(request_id: str, instructions: str, items: list[dict], item_schema: dict) -> str: ...
def build_repair_prompt(request_id: str, errors: list[str], schema: dict, *, truncated: bool = False) -> str: ...
def format_tools(tools: list[dict]) -> str: ...   # lista narzędzi w formacie z 03
```

Treści szablonów są w `03-prompty-modelu.md` i trafiają do pliku dosłownie jako stałe.

## I-tasks — `copilot_bridge/tasks/registry.py`

```python
@dataclass
class TaskDef:
    name: str
    version: int
    description: str
    kind: Literal["generic", "decide"]
    instructions: str                 # szablon string.Template: $question, $options, $criteria, …
    input_schema: dict
    output_schema: dict               # po rozwinięciu "default"/"text" na pełny schemat
    enum_from: dict[str, str]         # JSON Pointer w output_schema → nazwa pola wejścia z listą stringów
    default_mode: Mode = "work"
    timeout_s: float = 120
    source: Path | None = None
    @property
    def ref(self) -> str: ...         # f"{name}@{version}"

class TaskRegistry:
    def __init__(self, dirs: list[Path]) -> None: ...   # późniejsze katalogi nadpisują wcześniejsze
    def load(self) -> None: ...        # *.yaml; błędny plik → logger.warning i pominięcie
    def get(self, name: str) -> TaskDef: ...            # brak → BridgeError(UNKNOWN_TASK)
    def list(self) -> list[TaskDef]: ...
    def render(self, task: TaskDef, input: dict) -> tuple[str, str, dict]: ...
        # waliduje input względem input_schema (BridgeError(INVALID_INPUT) z listą błędów),
        # zwraca (instructions, data, output_schema):
        #  - instructions: szablon z podstawionymi polami (str bez zmian, inne json.dumps(ensure_ascii=False),
        #    brakujące pola → "brak"),
        #  - data: json.dumps(input, ensure_ascii=False, indent=2),
        #  - output_schema: z enum_from zastosowanym przez validation.with_enum
```

## I-backend — `copilot_bridge/backends/base.py`

```python
BackendState = Literal["starting", "ready", "auth_required", "resuming", "degraded"]

@dataclass
class BackendRequest:
    request_id: str
    prompt: str
    mode: Mode = "work"
    files: list[Path] = field(default_factory=list)
    conversation_url: str | None = None          # None = nowa rozmowa
    timeout_s: float = 120
    cancel_event: threading.Event | None = None
    hints: dict = field(default_factory=dict)    # tylko dla mock: schema, tools, tool_choice, final_schema, has_tool_results, batch_ids

@dataclass
class BackendReply:
    text: str                  # najlepszy kandydat: zawartość ostatniego bloku kodu albo cały tekst
    full_text: str             # cały tekst odpowiedzi
    sources: list[dict]        # [{"title": str, "url": str}] z interfejsu Copilota
    conversation_url: str | None
    truncated: bool = False
    duration_ms: int = 0

class Backend(Protocol):
    name: str
    def start(self) -> None: ...                          # tylko w wątku roboczym
    def stop(self) -> None: ...
    def state(self) -> BackendState: ...                  # bezpieczne z każdego wątku
    def send(self, req: BackendRequest) -> BackendReply: ...   # blokujące; błędy jako BridgeError
    def release(self, conversation_url: str | None) -> None: ...   # koniec pracy z rozmową (np. usunięcie)
    def tick(self) -> None: ...                           # konserwacja w bezczynności (wybudzenie, odświeżanie)
    def login(self, wait_s: float) -> BackendState: ...   # widoczne okno do ręcznego logowania
    def logout(self) -> None: ...                         # zamknięcie przeglądarki i usunięcie profilu
```

## I-tools — `copilot_bridge/tool_calls.py`

Narzędzia w formacie OpenAI: `{"type": "function", "function": {"name", "description", "parameters"}}`.

```python
def tool_names(tools: list[dict]) -> list[str]: ...
def forced_tool(tool_choice: str | dict) -> str | None: ...   # nazwa z {"type":"function","function":{"name":…}}
def validate_tool_output(obj: dict, tools: list[dict], tool_choice: str | dict,
                         allow_parallel: bool, final_schema: dict | None) -> list[str]: ...
    # schemat tool_output_schema + reguły: nieznane narzędzie, argumenty niezgodne z "parameters",
    # "required" bez wywołania, wymuszone narzędzie, >1 wywołanie przy allow_parallel=False,
    # "final" bez "content" (albo bez "data" zgodnego z final_schema)
def split_output(obj: dict) -> tuple[str, dict | None, str | None, list[dict]]: ...
    # ("tool_calls" | "json" | "text", data, text, tool_calls=[{"name", "arguments"}])
```

## I-engine — `copilot_bridge/engine.py`

```python
@dataclass
class CompletionRequest:
    request_id: str
    instructions: str
    data: str
    schema: dict | None = None             # None i brak tools → tryb tekstowy (TEXT_CONTENT_SCHEMA)
    tools: list[dict] | None = None
    tool_choice: str | dict = "auto"
    parallel_tool_calls: bool = True
    final_schema: dict | None = None       # schemat odpowiedzi końcowej przy tools
    mode: Mode = "work"
    files: list[Path] = field(default_factory=list)
    timeout_s: float = 120                 # limit jednej odpowiedzi Copilota
    cancel_event: threading.Event | None = None
    conversation_url: str | None = None
    keep_conversation: bool = False        # True → nie wołaj backend.release()
    hints: dict = field(default_factory=dict)

@dataclass
class CompletionResult:
    kind: Literal["json", "text", "tool_calls"]
    data: dict | None
    text: str | None
    tool_calls: list[dict]                 # [{"name": str, "arguments": dict}]
    raw: str
    attempts: int                          # liczba wiadomości wysłanych do Copilota
    first_try_valid: bool                  # poprawne bez promptu naprawczego
    duration_ms: int
    sources: list[dict]
    conversation_url: str | None
    prompt_chars: int

class Engine:
    def __init__(self, settings: Settings, backend: Backend) -> None: ...
    def complete(self, req: CompletionRequest) -> CompletionResult: ...
```

## I-storage — `copilot_bridge/storage.py`

```python
class Storage:
    def __init__(self, db_path: Path) -> None: ...     # sqlite3, check_same_thread=False, własny threading.Lock
    def init_schema(self) -> None: ...                 # CREATE TABLE IF NOT EXISTS …, PRAGMA journal_mode=WAL
    def close(self) -> None: ...
    # zapytania i decyzje
    def log_request(self, rec: dict) -> None: ...      # kolumny tabeli requests
    def log_decision(self, rec: dict) -> None: ...     # kolumny tabeli decisions
    def list_decisions(self, limit: int = 50, task: str | None = None) -> list[dict]: ...
    # cache
    def cache_get(self, key: str) -> dict | None: ...  # tylko nieprzeterminowane
    def cache_put(self, key: str, value: dict, ttl_s: int) -> None: ...
    def cache_purge(self) -> int: ...
    # joby
    def job_create(self, job_id: str, kind: str, request: dict, priority: int) -> None: ...
    def job_update(self, job_id: str, status: str, result: dict | None = None) -> None: ...
    def job_get(self, job_id: str) -> dict | None: ...   # pola JobInfo + "request"
    def jobs_fail_unfinished(self, error: dict) -> int: ...   # queued/running → failed z podanym błędem
    # rozmowy
    def conv_create(self, conv_id: str, mode: str) -> None: ...
    def conv_get(self, conv_id: str) -> dict | None: ...
    def conv_update(self, conv_id: str, chat_url: str | None) -> None: ...   # zapisuje url, turns += 1
    # idempotencja
    def idem_get(self, key: str) -> dict | None: ...    # {"body_hash", "status", "response"}
    def idem_put(self, key: str, body_hash: str, status: int, response: dict) -> None: ...
```

Tabele (wszystkie znaczniki czasu jako tekst ISO z `models.now_iso()`, JSON jako tekst):

```sql
CREATE TABLE requests (request_id TEXT PRIMARY KEY, created_at TEXT, task TEXT, ok INTEGER,
  error_code TEXT, duration_ms INTEGER, attempts INTEGER, first_try_valid INTEGER,
  task_version TEXT, prompt_version TEXT, input_hash TEXT);
CREATE TABLE decisions (request_id TEXT PRIMARY KEY, created_at TEXT, task TEXT, decision TEXT,
  confidence REAL, needs_review INTEGER, agreement REAL, votes INTEGER, message TEXT,
  input_json TEXT, task_version TEXT, prompt_version TEXT);
CREATE TABLE cache (key TEXT PRIMARY KEY, value_json TEXT, created_at TEXT, expires_at REAL);
CREATE TABLE jobs (job_id TEXT PRIMARY KEY, kind TEXT, status TEXT, priority INTEGER,
  request_json TEXT, result_json TEXT, created_at TEXT, updated_at TEXT);
CREATE TABLE conversations (conversation_id TEXT PRIMARY KEY, mode TEXT, chat_url TEXT,
  created_at TEXT, last_used_at TEXT, turns INTEGER DEFAULT 0);
CREATE TABLE idempotency (key TEXT PRIMARY KEY, body_hash TEXT, status INTEGER,
  response_json TEXT, created_at TEXT);
```

## I-metrics — `copilot_bridge/metrics.py`

```python
class Metrics:
    def __init__(self, window: int = 1000) -> None: ...   # threading.Lock, ostatnie `window` czasów
    def record(self, *, task: str, ok: bool, duration_ms: int, attempts: int,
               first_try_valid: bool | None, error_code: str | None,
               decision: str | None = None, needs_review: bool | None = None, cached: bool = False) -> None: ...
    def snapshot(self) -> dict: ...
        # {"requests_total", "ok_total", "errors_by_code": {}, "first_try_valid_rate", "repaired_rate",
        #  "cached_total", "avg_ms", "p95_ms", "by_task": {task: {"total", "ok", "avg_ms"}},
        #  "decisions": {task: {decision: n}}, "needs_review_rate"}
```

## I-runner — `copilot_bridge/task_runner.py`

```python
@dataclass
class TaskJob:
    request_id: str
    task: str                               # "ask" dla /v1/ask
    input: dict
    mode: Mode | None = None
    files: list[Path] = field(default_factory=list)
    timeout_s: float | None = None          # limit jednej odpowiedzi; None → z zadania
    include_raw: bool = False
    use_cache: bool = True
    votes: int = 1
    threshold: float | None = None
    fallback_decision: str | None = None
    output_schema: dict | None = None       # już rozwiązany; nadpisuje schemat zadania
    conversation_id: str | None = None
    cancel_event: threading.Event | None = None

class TaskRunner:
    def __init__(self, settings: Settings, engine: Engine, registry: TaskRegistry,
                 storage: Storage, metrics: Metrics, backend_name: str) -> None: ...
    def run(self, job: TaskJob) -> Envelope: ...
        # nigdy nie rzuca: BridgeError → Envelope(ok=False, error=…); inny wyjątek → INTERNAL (logger.exception)
```

## I-worker — `copilot_bridge/worker.py`

```python
@dataclass
class WorkItem:
    job_id: str
    kind: Literal["task", "completion", "batch", "admin"]
    payload: Any            # TaskJob | CompletionRequest | BatchJob | {"cmd": "login"|"logout"|"reload_tasks", ...}
    priority: int = 10      # 0 interaktywne, 10 zwykłe, 20 batch
    persist: bool = False   # True → stan w tabeli jobs
    created_at: float = field(default_factory=time.time)
    future: Future = field(default_factory=Future)
    cancel_event: threading.Event = field(default_factory=threading.Event)
    started_at: float | None = None

class Worker:
    def __init__(self, settings: Settings, storage: Storage, metrics: Metrics,
                 registry: TaskRegistry, backend_factory: Callable[[], Backend]) -> None: ...
    def start(self) -> None: ...                  # wątek "worker"; w nim backend_factory(), start(), Engine, TaskRunner
    def stop(self, timeout_s: float = 15) -> None: ...
    def submit(self, item: WorkItem) -> Future: ...   # pełna kolejka → BridgeError(QUEUE_FULL)
    def cancel(self, job_id: str) -> bool: ...
    def state(self) -> HealthStatus: ...          # "starting" przed startem backendu, "busy" w trakcie zadania, inaczej backend.state()
    def queue_length(self) -> int: ...
    @property
    def backend_name(self) -> str: ...
```

## I-api — `copilot_bridge/api/app.py` i `api/deps.py`

```python
# app.py
@dataclass
class AppContext:
    settings: Settings
    worker: Worker
    storage: Storage
    registry: TaskRegistry
    metrics: Metrics
    token: str
    version: str
    started_at: float
    allowed_hosts: set[str]
    server: Any | None = None          # uvicorn.Server, do zamknięcia przez /v1/admin/shutdown

def create_app(ctx: AppContext) -> FastAPI: ...   # app.state.ctx = ctx

# deps.py
JSON_UTF8 = "application/json; charset=utf-8"
def get_ctx(request: Request) -> AppContext: ...
def is_openai_path(path: str) -> bool: ...        # /v1/chat/completions, /v1/models
async def submit_and_wait(ctx: AppContext, item: WorkItem, total_timeout_s: float, request: Request) -> Any: ...
    # submit; czeka na future; co 1 s sprawdza request.is_disconnected() (rozłączenie → cancel, CANCELLED);
    # przekroczenie → cancel i BridgeError(TIMEOUT); wyjątek z future przechodzi dalej
def envelope_response(env: Envelope, fmt: OutputFormat = "json", *, request_id: str | None = None) -> Response: ...
    # status: 200 albo http_status kodu błędu; nagłówki X-Request-Id i Retry-After; formaty w sekcji „Formaty”
def error_envelope(request_id: str, err: BridgeError, task: str | None = None) -> Envelope: ...
async def run_idempotent(ctx: AppContext, request: Request, body: dict, handler: Callable[[], Awaitable[Response]]) -> Response: ...
    # nagłówek Idempotency-Key: brak → handler(); jest i ten sam hash ciała → zapisana odpowiedź
    # z nagłówkiem Idempotent-Replay: true; inny hash → IDEMPOTENCY_CONFLICT; zapisuje tylko odpowiedzi 200
def resolve_timeout(ctx: AppContext, requested: float | None) -> float: ...   # requested albo timeouts.request_total_s
```

## I-client — `copilot_bridge/client.py`

```python
class BridgeClient:
    def __init__(self, base_url: str, token: str, timeout_s: float = 330) -> None: ...
    def health(self) -> dict | None: ...    # GET /v1/health z timeoutem 2 s; brak połączenia → None
    def get(self, path: str, params: dict | None = None) -> tuple[int, dict | str]: ...
    def post_json(self, path: str, body: dict, params: dict | None = None,
                  headers: dict | None = None) -> tuple[int, dict | str]: ...
    def post_multipart(self, path: str, payload: dict, files: list[Path],
                       params: dict | None = None) -> tuple[int, dict | str]: ...
    def delete(self, path: str) -> tuple[int, dict | str]: ...
    # błąd połączenia → BridgeError(DAEMON_UNAVAILABLE); odpowiedź nie-JSON → str
def client_from_settings(settings: Settings) -> BridgeClient: ...   # base_url z server.host/port, token z ensure_token()
```

## I-launcher — `copilot_bridge/launcher.py`

```python
def daemon_health(settings: Settings) -> dict | None: ...
def daemon_process_alive() -> bool: ...     # próba przejęcia daemon.lock z timeout=0; udało się → False (i zwolnienie)
def spawn_daemon() -> int: ...              # pythonw.exe -m copilot_bridge.daemon w tle; zwraca pid
def ensure_daemon(settings: Settings, wait_s: float = 45) -> dict: ...
    # zwraca health; uruchamia daemona w razie potrzeby (pod blokadą spawn.lock);
    # inna wersja → restart; brak odpowiedzi po wait_s → BridgeError(DAEMON_START_FAILED, details={"log_tail": [...]})
def stop_daemon(settings: Settings, wait_s: float = 20) -> bool: ...
    # POST /v1/admin/shutdown; czeka na zwolnienie daemon.lock; awaryjnie taskkill /PID <pid z daemon.json> /T /F
def read_daemon_info() -> dict | None: ...  # daemon.json
```

## I-browser — `copilot_bridge/browser/*`

```python
# selectors.py
class Selectors:
    def __init__(self, data: dict[str, list[str]]) -> None: ...
    @classmethod
    def load(cls, override: Path | None = None) -> "Selectors": ...   # builtin selectors.yaml + nadpisania kluczy
    def get(self, name: str) -> list[str]: ...          # lista kandydatów (może być pusta)
    def has(self, name: str) -> bool: ...
    def first(self, root: Page | Locator, name: str, *, timeout_ms: int = 10000,
              state: str = "visible") -> Locator: ...   # pierwszy kandydat w danym stanie; brak → BridgeError(UI_CHANGED)
    def any_visible(self, root: Page | Locator, name: str) -> bool: ...   # bez czekania
    def all(self, root: Page | Locator, name: str) -> Locator: ...       # kandydaci połączeni przez Locator.or_(); brak kandydatów → BridgeError(UI_CHANGED)

# session.py
class EdgeSession:
    def __init__(self, settings: Settings, selectors: Selectors) -> None: ...
    def start(self, window: str | None = None) -> None: ...
    def stop(self) -> None: ...
    def restart(self, window: str | None = None) -> None: ...
    @property
    def page(self) -> Page: ...
    def open_chat_home(self) -> None: ...          # goto(url) i czekanie na input_box albo stronę logowania
    def is_login_page(self) -> bool: ...
    def is_ready(self, timeout_ms: int = 0) -> bool: ...
    def wait_for_login(self, wait_s: float) -> bool: ...
    def debug_dump(self, name: str) -> Path | None: ...   # zrzut ekranu + HTML do debug_dir()

# chat.py
class ChatPage:
    def __init__(self, session: EdgeSession, selectors: Selectors, settings: Settings) -> None: ...
    def new_chat(self) -> None: ...
    def open_conversation(self, url: str) -> None: ...
    def set_mode(self, mode: Mode) -> None: ...
    def attach(self, files: list[Path]) -> None: ...
    def assistant_messages(self) -> Locator: ...
    def count_assistant_messages(self) -> int: ...
    def send(self, text: str) -> None: ...
    def stop_generation(self) -> None: ...
    def current_url(self) -> str: ...
    def delete_current_chat(self) -> bool: ...

# waiter.py
def wait_for_reply(chat: ChatPage, selectors: Selectors, prev_count: int, timeouts: TimeoutsCfg,
                   cancel_event: threading.Event | None = None) -> Locator: ...

# extractor.py
@dataclass
class Extracted:
    code_text: str | None; full_text: str; sources: list[dict]; truncated: bool
def extract_reply(message: Locator, selectors: Selectors) -> Extracted: ...

# wake.py
class WakeDetector:
    def __init__(self, heartbeat_s: float, gap_s: float, *, clock: Callable[[], float] = time.time) -> None: ...
    def start(self) -> None: ...          # wątek "heartbeat" (daemon=True)
    def stop(self) -> None: ...
    def check_once(self) -> bool: ...     # jeden pomiar (używany przez wątek i testy); True = wykryto wybudzenie
    def woke_since(self, t: float) -> bool: ...
    def consume(self) -> bool: ...        # True raz po każdym wybudzeniu
def recover_after_wake(session: EdgeSession, timeout_s: float) -> BackendState: ...
```

## I-openai — `copilot_bridge/openai_compat/*`

```python
# models.py
MODELS: dict[str, Mode] = {"copilot-work": "work", "copilot-web": "web", "copilot": "work"}
class FunctionDef(BaseModel):     name: str; description: str | None = None; parameters: dict = {}
class ToolDef(BaseModel):         type: Literal["function"] = "function"; function: FunctionDef
class ToolCallFunction(BaseModel):name: str; arguments: str          # JSON jako tekst
class ToolCall(BaseModel):        id: str; type: Literal["function"] = "function"; function: ToolCallFunction
class ChatMessage(BaseModel):     # extra="allow"
    role: Literal["system", "developer", "user", "assistant", "tool"]
    content: str | list[dict] | None = None; name: str | None = None
    tool_calls: list[ToolCall] | None = None; tool_call_id: str | None = None
class ResponseFormat(BaseModel):  # extra="allow"
    type: Literal["text", "json_object", "json_schema"]; json_schema: dict | None = None
class ChatCompletionRequest(BaseModel):   # extra="allow"
    model: str; messages: list[ChatMessage] = Field(min_length=1)
    response_format: ResponseFormat | None = None; tools: list[ToolDef] | None = None
    tool_choice: str | dict | None = None; parallel_tool_calls: bool | None = None
    stream: bool | None = False; temperature: float | None = None; top_p: float | None = None
    n: int | None = None; max_tokens: int | None = None; max_completion_tokens: int | None = None
    seed: int | None = None; user: str | None = None
class ResponseMessage(BaseModel): role: Literal["assistant"] = "assistant"; content: str | None = None; tool_calls: list[ToolCall] | None = None
class Choice(BaseModel):          index: int = 0; message: ResponseMessage; finish_reason: Literal["stop", "tool_calls", "length"]; logprobs: None = None
class Usage(BaseModel):           prompt_tokens: int; completion_tokens: int; total_tokens: int
class ChatCompletion(BaseModel):
    id: str; object: Literal["chat.completion"] = "chat.completion"; created: int; model: str
    choices: list[Choice]; usage: Usage; system_fingerprint: str | None = None
def openai_error_body(message: str, type_: str, code: str | None, param: str | None = None) -> dict: ...
    # {"error": {"message", "type", "param", "code"}}
def openai_error_type(http_status: int) -> str: ...
    # 400/404/409/413/422 → "invalid_request_error", 401 → "authentication_error",
    # 403 → "permission_error", 429 → "rate_limit_error", inne → "api_error"

# convert.py
def mode_for_model(model: str) -> Mode: ...            # nieznany → BridgeError(INVALID_INPUT, lista modeli)
def message_text(msg: ChatMessage) -> str: ...          # content str albo połączone części {"type":"text"}; obraz → INVALID_INPUT
def flatten_messages(messages: list[ChatMessage]) -> tuple[str, str]: ...   # (instructions, transcript) — format w 03
def to_completion_request(req: ChatCompletionRequest, request_id: str, settings: Settings) -> CompletionRequest: ...
def to_chat_completion(result: CompletionResult, req: ChatCompletionRequest, completion_id: str) -> ChatCompletion: ...
def estimate_tokens(text: str) -> int: ...              # len(text) // 4 + 1
def ignored_params(req: ChatCompletionRequest) -> list[str]: ...   # ustawione: temperature, top_p, n (≠1), max_tokens, max_completion_tokens, seed
def new_completion_id() -> str: ...                     # "chatcmpl-" + 24 znaki hex
def new_call_id() -> str: ...                           # "call_" + 24 znaki hex
```

## Endpointy API

Wszystkie ścieżki poza `GET /v1/health` i `GET /` wymagają `Authorization: Bearer <token>`. Odpowiedzi JSON mają `Content-Type: application/json; charset=utf-8` i nagłówek `X-Request-Id`.

| Metoda | Ścieżka | Wejście | Wyjście | Etap |
|---|---|---|---|---|
| GET | `/v1/health` | – | `HealthResponse` | M3 |
| POST | `/v1/ask` | `AskRequest`, `?format=` | `Envelope` | M3 |
| GET | `/v1/tasks` | – | `list[TaskInfo]` | M3 |
| GET | `/v1/tasks/{name}` | – | `TaskInfo` | M3 |
| POST | `/v1/tasks/{name}` | `TaskRunRequest`, `?format=` | `Envelope` | M3 |
| GET | `/v1/schemas`, `/v1/schemas/{name}` | – | schematy | M3 |
| POST | `/v1/admin/shutdown` | – | `{"ok": true}` | M3 |
| POST | `/v1/admin/reload-tasks` | – | `{"ok": true, "tasks": n}` | M3 |
| POST | `/v1/admin/login` | `{"wait_s": 600}` | `HealthResponse` | M3 (mock), M5 (Edge) |
| POST | `/v1/admin/logout` | – | `HealthResponse` | M3 (mock), M5 (Edge) |
| POST | `/v1/chat/completions` | `ChatCompletionRequest` | `ChatCompletion` | M4 |
| GET | `/v1/models` | – | `{"object": "list", "data": [...]}` | M4 |
| POST | `/v1/jobs` | `JobCreate` | 202 `JobInfo` | M6 |
| GET | `/v1/jobs/{id}` | – | `JobInfo` | M6 |
| DELETE | `/v1/jobs/{id}` | – | `JobInfo` | M6 |
| POST | `/v1/tasks/{name}/batch` | `BatchRequest` | 202 `JobInfo` | M6 |
| POST | `/v1/ask/upload` | multipart: `payload` (JSON `AskRequest`), `files` | `Envelope` | M6 |
| POST | `/v1/tasks/{name}/upload` | multipart: `payload` (JSON `TaskRunRequest`), `files` | `Envelope` | M6 |
| POST | `/v1/conversations` | `ConversationCreate` | `ConversationInfo` | M6 |
| POST | `/v1/conversations/{id}/messages` | `AskRequest` | `Envelope` | M6 |
| GET | `/v1/decisions?limit=&task=` | – | lista wpisów dziennika | M6 |
| GET | `/v1/metrics` | – | `Metrics.snapshot()` | M6 |
| GET | `/` | – | playground HTML | M7 |

## Formaty wyjścia `?format=`

- `json` (domyślny): pełna koperta `Envelope`.
- `flat`: jeden poziom JSON:
  `{"ok", "request_id", "task", <wszystkie pola z data o wartościach skalarnych>, <listy stringów złączone "; ">, "error_code", "error_message", "duration_ms"}`.
- `text`: te same pola co `flat`, jako linie `klucz=wartość` (znaki nowej linii w wartości zamienione na `\n`), `Content-Type: text/plain; charset=utf-8`.

## Przykład koperty (decyzja)

```json
{
  "ok": true,
  "request_id": "req_4f1c2a9b8e7d6c5b4a39",
  "task": "decide",
  "data": {
    "request_id": "req_4f1c2a9b8e7d6c5b4a39",
    "status": "ok",
    "decision": "approve",
    "message": "Zwrot mieści się w limicie 30 dni, a klient ma czystą historię. Kwota jest poniżej progu wymagającego akceptacji kierownika.",
    "reasons": ["zgłoszenie w terminie", "kwota poniżej 500 zł", "brak wcześniejszych reklamacji"],
    "confidence": 0.86,
    "needs_review": false
  },
  "raw": null,
  "error": null,
  "meta": {
    "duration_ms": 18450, "queue_ms": 12, "attempts": 1, "first_try_valid": true,
    "votes": 1, "agreement": null, "cached": false, "fallback_applied": false,
    "backend": "m365-playwright", "api_version": "v1", "schema_name": "decide",
    "task_version": "decide@1", "prompt_version": "envelope@1", "sources": []
  }
}
```

Błąd: `"ok": false`, `"data": null`, `"error": {"code": "AUTH_REQUIRED", "message": "Sesja M365 wygasła. Uruchom: copilot-bridge login", "retryable": false, "details": {}}`.
