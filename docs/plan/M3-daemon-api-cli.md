# M3. Daemon, API i CLI

**Cel:** działający daemon z kolejką, API HTTP, auto-startem z CLI i komendami zarządzania. Wszystko na backendzie `mock`, więc bez przeglądarki.

**Gotowe, gdy:** przy `backend: mock` w `config.yaml` komenda `copilot-bridge task decide …` sama uruchamia daemona i zwraca kopertę, `copilot-bridge daemon status` pokazuje stan, a testy API są zielone.

## M3.1 `worker.py` (≈300 linii)

**Wklej:** kartę, I-errors, I-config, I-models (`HealthStatus`), I-backend, I-engine, I-tasks, I-storage, I-metrics, I-runner, I-worker.

- Kolejka `queue.PriorityQueue` z krotkami `(priority, seq, item)`; `seq` z `itertools.count()` zapewnia kolejność FIFO w obrębie priorytetu.
- `submit`: kolejka dłuższa niż `limits.max_queue` → `QUEUE_FULL`; `persist=True` → `storage.job_create`; słownik `job_id → WorkItem` do anulowania.
- Pętla wątku `worker`:
  1. `backend = backend_factory()`, `backend.start()` (błąd startu → logger.exception, stan `degraded`, pętla dalej działa i zwraca błędy `INTERRUPTED`), potem `Engine` i `TaskRunner`,
  2. `get(timeout=settings.wake.heartbeat_s)`; pusta kolejka → `backend.tick()` (wyjątki logowane, nie zatrzymują pętli),
  3. element anulowany przed startem → `future.set_exception(BridgeError(CANCELLED, …))`,
  4. `payload.cancel_event = item.cancel_event` (dla `TaskJob` i `CompletionRequest`),
  5. przy stanie backendu `auth_required` zadania `task`, `completion`, `batch` kończą się od razu błędem `AUTH_REQUIRED`,
  6. wykonanie według `kind`: `task` → `runner.run(payload)` (koperta); `completion` → `engine.complete(payload)`; `batch` → `from copilot_bridge.batch import BatchRunner` (import w funkcji; plik z M6); `admin` → `login`/`logout`/`reload_tasks`,
  7. ponowienie po przerwaniu: gdy `settings.wake.retry_interrupted` i wynik to błąd `INTERRUPTED` (wyjątek albo koperta z tym kodem), `backend.tick()` i jedno ponowienie,
  8. wynik do `future` (`set_result` albo `set_exception`); `persist=True` → `storage.job_update` (`done`, `failed`, `cancelled`) z kopertą albo błędem.
- `cancel(job_id)`: ustawia `cancel_event`; zwraca `True`, gdy element istnieje i nie jest zakończony.
- `stop`: flaga zakończenia, element-strażnik w kolejce, `join`, `backend.stop()` w wątku roboczym przed wyjściem z pętli.
- `state()`: przed startem backendu `starting`; w trakcie elementu `busy`; inaczej `backend.state()`.

## M3.2 `tests/test_worker.py` (≈170 linii)

**Wklej:** kartę, I-worker, I-runner, I-engine, I-backend, specyfikację M2.18 (`script()`).

Przypadki (MockBackend): zadanie `task` zwraca kopertę; `completion` zwraca `CompletionResult`; priorytet 0 wyprzedza wcześniej dodane z priorytetem 20 (backend spowolniony przez `script()` z opóźnieniem albo podmieniony `send` z `time.sleep`); pełna kolejka → `QUEUE_FULL`; anulowanie przed startem → `CANCELLED`; `persist=True` zapisuje stan w bazie; `stop()` kończy wątek.

## M3.3 `api/deps.py` (≈220 linii)

**Wklej:** kartę, I-errors, I-models, I-worker, I-storage, I-api, sekcję „Formaty wyjścia” z 02.

- `submit_and_wait`: `fut = ctx.worker.submit(item)`; pętla `asyncio.wait_for(asyncio.shield(asyncio.wrap_future(fut)), 1.0)`; po każdej sekundzie `await request.is_disconnected()`; limit łączny `total_timeout_s`.
- `envelope_response`: `format=json` → `JSONResponse` z `media_type=JSON_UTF8` (`model_dump(mode="json")`); `flat` i `text` według 02; nagłówki `X-Request-Id` i `Retry-After` (z `error.details["retry_after_s"]`, jeśli jest).
- `run_idempotent`: hash `sha256` z `json.dumps(body, sort_keys=True)`.
- Utwórz ręcznie pusty `api/__init__.py`.

## M3.4 `api/app.py` (≈200 linii)

**Wklej:** kartę, I-errors, I-models, I-worker, I-api.

- `create_app(ctx)`: `FastAPI(title="copilot-bridge", version=ctx.version)`, `app.state.ctx = ctx`.
- Middleware (jeden, w tej kolejności):
  1. `request_id` z nagłówka `X-Request-Id` klienta albo `new_request_id()`, zapisany w `request.state.request_id`,
  2. nagłówek `Host` spoza `ctx.allowed_hosts` → 403 `FORBIDDEN_HOST`,
  3. ścieżki inne niż `GET /v1/health` i `GET /` bez poprawnego `Authorization: Bearer` (porównanie `secrets.compare_digest`) → 401 `UNAUTHORIZED`,
  4. odpowiedź dostaje `X-Request-Id`.
  Błędy z middleware w formacie OpenAI dla ścieżek z `is_openai_path`, inaczej jako koperta.
- Handlery wyjątków: `BridgeError`, `RequestValidationError` (→ `INVALID_INPUT` z listą błędów w `details`) i `Exception` (→ `INTERNAL`, logger.exception); format zależny od ścieżki jak wyżej.
- Rejestracja routerów: `routes_core.router` zawsze; opcjonalnie moduły `copilot_bridge.openai_compat.routes`, `copilot_bridge.api.routes_jobs`, `copilot_bridge.api.uploads`: import w `try`, a `ModuleNotFoundError` z `e.name` równym nazwie modułu → pominięcie (log debug). Inne błędy importu przechodzą dalej.
- `GET /`: gdy istnieje `paths.static_dir() / "playground.html"`, zwraca go jako `HTMLResponse`; inaczej krótki tekst „playground w M7”.
- Brak middleware CORS.

## M3.5 `api/routes_core.py` (≈260 linii)

**Wklej:** kartę, I-errors, I-models, I-schemas, I-tasks, I-runner, I-worker, I-api.

`router = APIRouter(prefix="/v1")`:
- `GET /health` → `HealthResponse` (`status = ctx.worker.state()`, `pid = os.getpid()`, `uptime_s`).
- `POST /ask` (`AskRequest`, `format` z query): `TaskJob(task="ask", input={"question", "context"})`, `output_schema` przez `resolve_schema`, `WorkItem(kind="task", priority=0)`, `submit_and_wait`, `envelope_response`, całość przez `run_idempotent`.
- `GET /tasks`, `GET /tasks/{name}` → `TaskInfo`.
- `POST /tasks/{name}` (`TaskRunRequest`, `format`): jak `ask`.
- `timeout_s` z zapytania (w `ask` i zadaniach) to łączny limit czekania: `resolve_timeout(ctx, timeout_s)`. Limit jednej odpowiedzi Copilota w `TaskJob.timeout_s` to `min(timeout_s, settings.timeouts.response_s)` albo `None`, gdy klient go nie podał.
- `GET /schemas`, `GET /schemas/{name}`.
- `POST /admin/shutdown`: odpowiedź `{"ok": true}`, potem (przez `BackgroundTasks`) `ctx.server.should_exit = True`.
- `POST /admin/reload-tasks`: `WorkItem(kind="admin", payload={"cmd": "reload_tasks"})`.
- `POST /admin/login` (`{"wait_s"}`), `POST /admin/logout`: element `admin` z najwyższym priorytetem (0); limit czekania `wait_s + 60`.

Nazwa zadania w ścieżce jest sprawdzana przez `registry.get` przed wstawieniem do kolejki (szybki 404).

## M3.6 `tests/conftest.py` (≈150 linii)

**Wklej:** kartę, I-config, I-worker, I-api, I-storage, I-tasks, I-metrics, specyfikację M2.18.

Fixture:
- `home` (autouse): `monkeypatch.setenv("COPILOT_BRIDGE_HOME", str(tmp_path))`.
- `settings`: `Settings(backend="mock")` z `limits.min_interval_s=0`.
- `ctx`: rejestr (wbudowane zadania), `Storage` w `tmp_path`, `Metrics`, `Worker` z `MockBackend` (uruchomiony, zatrzymany po teście), token `"test-token"`, `allowed_hosts` z `"testserver"` i `"127.0.0.1:<port>"`.
- `client`: `TestClient(create_app(ctx))` z nagłówkiem `Authorization: Bearer test-token`.
- `live_server`: uvicorn w wątku na wolnym porcie (`socket` bind na port 0), `uvicorn.Server(config)`, czekanie aż `/v1/health` odpowie; zwraca `(base_url, token)`; na końcu `should_exit = True`.

## M3.7 `tests/test_api_core.py` (≈220 linii)

**Wklej:** kartę, I-models, I-api, tabelę endpointów i formatów z 02, fixture z M3.6.

Przypadki: `health` bez tokenu → 200; `ask` bez tokenu → 401 z kopertą; zły `Host` → 403; `POST /v1/tasks/decide` → 200, `decision` z listy; brak `options` → 422 `INVALID_INPUT`; nieznane zadanie → 404 `UNKNOWN_TASK`; `format=flat` → płaski JSON z `decision`; `format=text` → linie `klucz=wartość`; `Content-Type` z `charset=utf-8`; polskie znaki w odpowiedzi poprawne; `Idempotency-Key`: powtórka zwraca to samo z `Idempotent-Replay: true`, inne ciało → 409; `GET /v1/tasks` zawiera `decide` i `ask`.

## M3.8 `daemon/main.py` (≈200 linii)

**Wklej:** kartę, I-paths, I-config, I-logging, I-storage, I-tasks, I-metrics, I-worker, I-api, sekcję „Pułapki Windows” z 01, funkcję `make_backend` z M2.29.

`def run_daemon(foreground: bool = False) -> int:`
1. `sys.stdout is None` albo `sys.stderr is None` → przekierowanie do `logs_dir() / "daemon.stdout.log"` (tryb dopisywania, utf-8, `line_buffering=True`).
2. Na Windowsie `asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())`.
3. `load_settings()`, `setup_logging(settings, to_console=foreground)`.
4. `FileLock(daemon_lock_file())`, `acquire(timeout=0)`; zajęta → log „daemon już działa”, zwraca 0.
5. `ensure_token()`, `Storage` + `init_schema()` + `jobs_fail_unfinished(ErrorBody INTERRUPTED)`, `TaskRegistry([builtin_tasks_dir(), user_tasks_dir()])` + `load()`, `Metrics()`.
6. `Worker(…, backend_factory=lambda: make_backend(settings))`, `start()`.
7. `AppContext` z `allowed_hosts = {f"127.0.0.1:{port}", f"localhost:{port}", "127.0.0.1", "localhost"}`, `create_app`.
8. `uvicorn.Config(app, host, port, log_config=None, access_log=False, loop="asyncio", lifespan="on")`, `Server`, `ctx.server = server`.
9. Zapis `daemon.json`: `{"pid", "port", "version", "started_at"}`.
10. `server.run()`. Zajęty port → log błędu i kod 1.
11. `finally`: `worker.stop()`, `storage.close()`, usunięcie `daemon.json`, zwolnienie blokady.

Ręcznie utwórz `daemon/__init__.py` (pusty) i `daemon/__main__.py`:
`from copilot_bridge.daemon.main import run_daemon` oraz `raise SystemExit(run_daemon())`.

**Sprawdzenie:** `python -m copilot_bridge.daemon` w konsoli (przy `backend: mock`), potem w drugiej konsoli `Invoke-RestMethod http://127.0.0.1:8765/v1/health`.

## M3.9 `client.py` (≈150 linii)

**Wklej:** kartę, I-errors, I-config, I-paths, I-client.

`httpx.Client` z `timeout=httpx.Timeout(timeout_s, connect=5)`, nagłówek `Authorization`; `httpx.ConnectError`, `ConnectTimeout` → `DAEMON_UNAVAILABLE`; odpowiedź JSON → `dict`, inna → tekst; multipart: pole `payload` (tekst JSON) i pola `files` (otwarte pliki, zamykane po wysłaniu).

## M3.10 `launcher.py` (≈210 linii)

**Wklej:** kartę, I-errors, I-paths, I-config, I-logging (`tail_log`), I-client, I-launcher, punkty 1, 4 i 9 z „Pułapek Windows” w 01.

- `spawn_daemon`: `pythonw = Path(sys.executable).with_name("pythonw.exe")` (brak → `sys.executable`); `subprocess.Popen([str(pythonw), "-m", "copilot_bridge.daemon"], creationflags=DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP | CREATE_BREAKAWAY_FROM_JOB, stdin/stdout/stderr=DEVNULL, close_fds=True, cwd=app_dir())`; `OSError` → ponowienie bez `CREATE_BREAKAWAY_FROM_JOB`. Stałe flag przez `getattr(subprocess, …, wartość_liczbowa)`, żeby moduł importował się też poza Windowsem.
- `ensure_daemon`: health → gdy jest i `version == copilot_bridge.__version__`, zwróć; inna wersja → `stop_daemon` i start; brak → pod `FileLock(spawn_lock_file(), timeout=60)` ponowne sprawdzenie, `spawn_daemon()`, odpytywanie health co 0,5 s do `wait_s`. Każda odpowiedź health (dowolny `status`) oznacza sukces.
- `stop_daemon`: shutdown przez API; czekanie aż `daemon_process_alive()` da `False`; po `wait_s` `taskkill /PID <pid> /T /F` (pid z `daemon.json`).

## M3.11 `cli.py` (≈300 linii)

**Wklej:** kartę, I-errors, I-config, I-client, I-launcher, tabelę kodów błędów z 02.

`app = typer.Typer(no_args_is_help=True)`. Na starcie `sys.stdout.reconfigure(encoding="utf-8")`. Rejestracja `cli_admin` przez `app.add_typer(...)` i `app.command(...)` (import z `copilot_bridge.cli_admin`).

Komendy (każda: `load_settings`, `ensure_daemon`, `client_from_settings`, wypisanie JSON z `ensure_ascii=False, indent=2`, kod wyjścia 0 albo `info(error.code).exit_code`):
- `ask PYTANIE [--context-file] [--schema NAZWA|PLIK.json] [--mode] [--file …] [--timeout] [--include-raw] [--no-cache] [--format json|flat|text]`. Plik `.json` w `--schema` czytany przez CLI i wysyłany jako słownik. `--file` → `post_multipart` na `/v1/ask/upload` (działa od M6).
- `task NAZWA [--input-json] [--input-file] [--text] [--text-file] [--set klucz=wartość …] [--list klucz=a,b,c …] [--file …] [--votes] [--threshold] [--fallback] [--mode] [--timeout] [--include-raw] [--no-cache] [--async] [--format]`. `--set`: wartość parsowana jako JSON, a gdy się nie da, jako tekst. `--async` → `POST /v1/jobs` (od M6).
- `tasks`: lista zadań (nazwa, wersja, opis).
- `job get ID`, `job cancel ID`, `job wait ID [--timeout]` (od M6; `wait` odpytuje co 2 s).
- Błędy po stronie CLI (`DAEMON_UNAVAILABLE`, `DAEMON_START_FAILED`) wypisywane jako koperta błędu w tym samym formacie.

## M3.12 `cli_admin.py` (≈260 linii)

**Wklej:** kartę, I-errors, I-paths, I-config, I-logging, I-client, I-launcher.

- `daemon_app = typer.Typer()` z komendami: `start` (`ensure_daemon`), `stop`, `restart`, `status` (health + `daemon.json` + ścieżka logów), `logs [--tail 50]`, `run` (`run_daemon(foreground=True)` w bieżącej konsoli, do diagnozy).
- Funkcje rejestrowane jako komendy główne: `login [--wait 600]` (`POST /v1/admin/login` z długim timeoutem, komunikat „Zaloguj się w oknie Edge”), `logout` (z potwierdzeniem `typer.confirm`), `config init|path|show`, `openapi [--out openapi.json]` (pobiera `/openapi.json` z daemona i zapisuje z wcięciami).
- `doctor [--quick]`, `eval PLIK [--repeat 1]`, `autostart enable|disable|status`: import modułów z M7 wewnątrz funkcji; `ModuleNotFoundError` → komunikat „dostępne od M7”.

## M3.13 Sprawdzenie ręczne

```powershell
copilot-bridge config init
notepad (copilot-bridge config path)      # ustaw: backend: mock
copilot-bridge task decide --set "question=Czy zatwierdzić?" --list "options=approve,reject,escalate"
copilot-bridge daemon status
copilot-bridge task decide --set "question=Test" --list "options=a,b" --format text
copilot-bridge daemon stop
copilot-bridge tasks                       # sam uruchamia daemona ponownie
```

Sprawdź też, że po zamknięciu okna PowerShell daemon dalej działa (`daemon status` w nowym oknie).

W PowerShellu argumenty z przecinkami zawsze bierz w cudzysłów (`"options=a,b"`), bo bez niego PowerShell zamienia je na kilka osobnych argumentów.
