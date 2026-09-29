# M6. Rozszerzenia: joby, batch, załączniki, rozmowy, zadania

**Cel:** wszystko, czego potrzebują dłuższe i bardziej złożone zastosowania: zadania asynchroniczne, przetwarzanie zbiorcze, załączniki, rozmowy wieloturowe, pozostałe zadania wbudowane, dziennik decyzji i metryki przez API.

**Gotowe, gdy:** testy M6 są zielone, a batch 50 elementów `classify` kończy się na prawdziwym Copilocie wynikami dla wszystkich elementów.

## M6.1 `batch.py` (≈260 linii)

**Wklej:** kartę, I-errors, I-config, I-models, I-schemas, I-prompts, I-json, I-validation, I-tasks, I-engine, I-storage, I-metrics, I-backend, szablon zbiorczy z 03.

```python
@dataclass
class BatchJob:
    request_id: str
    task: str
    items: list[dict]            # [{"id": str, "input": dict}]
    chunk_size: int = 10
    mode: Mode | None = None
    timeout_s: float | None = None
    cancel_event: threading.Event | None = None

class BatchRunner:
    def __init__(self, settings: Settings, backend: Backend, engine: Engine, registry: TaskRegistry,
                 storage: Storage, metrics: Metrics) -> None: ...
    def run(self, job: BatchJob) -> Envelope: ...    # nigdy nie rzuca
```

- Każdy element walidowany przez `registry.render` (błąd elementu zapisany jako wynik elementu, bez przerywania całości). Zadanie w batchu musi mieć instrukcję niezależną od elementu, więc `instructions` bierz z pierwszego poprawnego elementu, a dane elementów jako `{"id", "input"}`.
- Paczki po `chunk_size`: `build_batch_prompt`, `backend.send` z `hints={"batch_ids": [...], "schema": item_schema}`, `parse_reply`, walidacja `batch_output_schema(item_schema)`, a potem każdego `result` osobno (z `enum` z `enum_from` dla danego elementu).
- Elementy brakujące albo niepoprawne w odpowiedzi paczki: jedno ponowienie paczki z samymi nieudanymi elementami; potem ponowienie pojedynczo przez `engine.complete` (zwykły szablon JSON).
- Anulowanie sprawdzane między paczkami.
- Wynik: `Envelope(data={"items": [{"id", "ok", "data" | "error"}], "summary": {"total", "ok", "failed"}})`, w `meta` łączny czas i liczba prób.
- Dla zadań `decide` każdy element dostaje `needs_review` według progu, a do dziennika decyzji trafia każdy element osobno.

**Uwaga:** `Worker` (M3.1) tworzy `BatchRunner` przy pierwszym elemencie `batch` i przekazuje mu swój `backend` i `engine`.

## M6.2 `tests/test_batch.py` (≈160 linii)

**Wklej:** kartę, specyfikację M6.1, specyfikację M2.18 (`script()` i batch w syntezie).

Przypadki: 25 elementów przy `chunk_size=10` → 3 paczki, wszystkie wyniki; paczka z brakującym elementem → ponowienie tylko tego elementu; niepoprawne wejście jednego elementu → błąd tylko w tym elemencie; anulowanie po pierwszej paczce → koperta `CANCELLED` z częściowymi wynikami w `details`.

## M6.3 `api/routes_jobs.py` (≈240 linii)

**Wklej:** kartę, I-errors, I-models, I-schemas, I-runner, I-worker, I-storage, I-metrics, I-api, specyfikację `BatchJob` z M6.1, tabelę endpointów z 02.

`router = APIRouter(prefix="/v1")`:
- `POST /jobs` (`JobCreate`) → `TaskJob`, `WorkItem(kind="task", priority=request.priority, persist=True)`, **bez czekania**; odpowiedź 202 `JobInfo` ze statusem `queued`.
- `GET /jobs/{id}` → `storage.job_get`; brak → 404 `NOT_FOUND`.
- `DELETE /jobs/{id}` → `worker.cancel`; w bazie status `cancelled`, jeśli zadanie nie zdążyło się zakończyć.
- `POST /tasks/{name}/batch` (`BatchRequest`) → `BatchJob`, `WorkItem(kind="batch", priority=20, persist=True)`, 202 `JobInfo`.
- `POST /conversations` (`ConversationCreate`) → `storage.conv_create`, `ConversationInfo`.
- `POST /conversations/{id}/messages` (`AskRequest`) → `TaskJob(task="ask", conversation_id=id)`, czekanie jak w `/v1/ask`.
- `GET /decisions?limit=50&task=` → `storage.list_decisions`.
- `GET /metrics` → `metrics.snapshot()` plus `queue_length` i stan.

Router rejestrowany automatycznie przez `app.py`.

## M6.4 `api/uploads.py` (≈170 linii)

Pomiń ten krok, jeśli M1 wykazał, że wgrywanie plików nie działa.

**Wklej:** kartę, I-errors, I-paths, I-config, I-models, I-schemas, I-runner, I-worker, I-api.

`router = APIRouter(prefix="/v1")`:
- `POST /ask/upload` i `POST /tasks/{name}/upload`: `multipart/form-data` z polem `payload` (tekst JSON `AskRequest` albo `TaskRunRequest`) i polami `files` (`list[UploadFile]`).
- Kontrola: liczba plików ≤ `limits.max_files`, rozmiar ≤ `max_file_mb`, rozszerzenia z listy `.pdf .docx .xlsx .pptx .txt .csv .png .jpg .jpeg` → inaczej `ATTACHMENT_REJECTED`.
- Zapis do `uploads_dir() / request_id /` z oryginalną nazwą oczyszczoną z niedozwolonych znaków; `TaskJob.files`; po zakończeniu zadania katalog usuwany (`finally`).
- Cache wyłączony dla zapytań z plikami (robi to już `TaskRunner`).
- Formularze w FastAPI wymagają pakietu `python-multipart` (jest w zależnościach od M1.1).

## M6.5 Zadania `classify.yaml`, `summarize.yaml`, `extract.yaml`

**Wklej:** kartę, sekcje „Format pliku zadania”, `decide.yaml` (jako wzór) i wiersze tych zadań z tabeli „Pozostałe zadania” w 03.

Poproś o trzy pliki YAML w jednej odpowiedzi (każdy w osobnym bloku, łącznie ≤ 250 linii). Instrukcje w stylu `decide.yaml`: krótkie zdania, wyraźne nazwy pól, zachowanie przy braku danych.

## M6.6 Zadania `rewrite.yaml`, `draft.yaml`, `work_search.yaml`

Jak M6.5, dla pozostałych trzech wierszy tabeli.

**Sprawdzenie M6.5–M6.6:** `copilot-bridge daemon restart`, potem `copilot-bridge tasks` pokazuje osiem zadań; test M6.7 je wczytuje.

## M6.7 `tests/test_jobs_api.py` (≈220 linii)

**Wklej:** kartę, I-models, I-api, fixture z M3.6, tabelę endpointów z 02.

Przypadki: `POST /v1/jobs` → 202, potem `GET` aż do `done` (limit 10 s), wynik z decyzją; `DELETE` joba w kolejce → `cancelled`; nieznany job → 404; batch 12 elementów → `done` z 12 wynikami; rozmowa: utworzenie i dwie wiadomości → `turns == 2`; `GET /v1/decisions` zwraca wpisy po zadaniach `decide`; `GET /v1/metrics` ma `requests_total > 0`; `upload` z plikiem `.txt` → 200 (pomiń, jeśli M6.4 pominięty); plik `.exe` → 422 `ATTACHMENT_REJECTED`; wszystkie zadania wbudowane (osiem) wczytane.

## M6.8 Sprawdzenie ręczne

1. Batch na prawdziwym Copilocie: plik `zgloszenia.json` z 50 elementami `{"id", "input": {"text", "labels"}}`, wywołanie `POST /v1/tasks/classify/batch`, odpytywanie `GET /v1/jobs/{id}`. Zmierz czas i porównaj z 50 pojedynczymi zapytaniami.
2. Załącznik: `copilot-bridge task extract --file .\faktura.pdf --set "fields={\"numer\": \"numer faktury\", \"kwota\": \"kwota brutto\"}"`.
3. Rozmowa: dwie wiadomości w jednej rozmowie; druga odwołuje się do pierwszej.
4. Jeśli M1 potwierdził automatyczne usuwanie rozmów: `history.cleanup: delete` i sprawdzenie, że historia Copilota się nie zapełnia.
