# M2. Rdzeń

**Cel:** cała logika przetwarzania bez przeglądarki i bez serwera: konfiguracja, błędy, prompty, parsowanie, walidacja, zadania, silnik z naprawą, baza, metryki, obsługa decyzji. Wszystko sprawdzane testami na backendzie `mock`.

**Gotowe, gdy:** `python -m pytest` jest zielony, a skrypt z M2.30 zwraca kopertę decyzji z backendu `mock`.

Kroki wykonuj po kolei. Kroki testowe używają szablonu D.

## M2.1 `errors.py` (≈130 linii)

**Wklej:** kartę, I-errors, tabelę „Kody błędów” z 02.

- `ErrorCode`, `ErrorInfo`, `ERROR_TABLE` dokładnie według tabeli; kody tylko dla CLI mają `http_status=500`.
- `BridgeError.__str__` zwraca `"<KOD>: <komunikat>"`.
- `info()` dla nieznanego kodu zwraca informacje `INTERNAL`.

## M2.2 `tests/test_errors.py` (≈60 linii)

**Wklej:** kartę, I-errors, tabelę kodów.

Przypadki: każdy kod ma wpis w `ERROR_TABLE`; `AUTH_REQUIRED` → 503, wyjście 3, nieponawialny; `TIMEOUT` ponawialny; `to_dict()` ma cztery klucze; `details` domyślnie `{}`; `info("NIEZNANY")` zwraca dane `INTERNAL`.

## M2.3 `config.py` (≈170 linii)

**Wklej:** kartę, I-errors, I-paths, I-config, sekcję „Konfiguracja `config.yaml`” z 01.

- Modele z `extra="forbid"`; `load_settings` czyta YAML (`yaml.safe_load`); pusty plik → domyślne.
- Błąd YAML albo walidacji pydantic → `BridgeError(INVALID_INPUT, "Błędna konfiguracja: …", details={"errors": [...]})`.
- `write_default_config` zapisuje YAML z komentarzami z sekcji 01 (tekst wpisany w kod jako stała) i nie nadpisuje istniejącego pliku.

## M2.4 `tests/test_config.py` (≈80 linii)

**Wklej:** kartę, I-errors, I-paths, I-config.

Przypadki: brak pliku → domyślne (`server.port == 8765`); częściowy YAML nadpisuje tylko podane klucze; nieznany klucz → `BridgeError` `INVALID_INPUT`; zły typ → błąd; `write_default_config` tworzy plik, który `load_settings` wczytuje bez błędu; drugi zapis nie nadpisuje.

## M2.5 `logging_setup.py` (≈90 linii)

**Wklej:** kartę, I-paths, I-config, I-logging.

- Handler plikowy z rotacją, opcjonalnie konsola (stderr).
- Ponowne wywołanie `setup_logging` nie dubluje handlerów (usuń poprzednie handlery dodane przez ten moduł).
- Wyciszenie gadatliwych loggerów: `uvicorn.access` na WARNING, `httpx` na WARNING.

## M2.6 `models.py` (≈200 linii)

**Wklej:** kartę, I-config (tylko `Mode`), I-models.

- Modele dokładnie według I-models, pydantic v2, `Field(default_factory=…)` dla list i słowników.
- `Envelope` i `Meta` z `model_config = ConfigDict(extra="forbid")`; modele zapytań z `extra="forbid"`, żeby literówki klientów były błędem 422.

## M2.7 `schemas.py` (≈150 linii)

**Wklej:** kartę, I-errors, I-validation, I-schemas, sekcję „Schematy” z 03.

- Schematy jako stałe słownikowe (dosłownie z 03).
- `resolve_schema` importuje `check_schema` z `copilot_bridge.validation` (import wewnątrz funkcji, żeby uniknąć cykli).
- `tool_output_schema` i `batch_output_schema` usuwają `request_id` z kopii schematu wyniku (z `properties` i `required`).

## M2.8 `json_extract.py` (≈220 linii)

**Wklej:** kartę, I-json.

- `find_json_candidates`: wyrażenie dla bloków ```` ```json … ``` ```` (bez rozróżniania wielkości liter), potem pozostałe bloki ```` ``` ````, potem skaner nawiasów klamrowych uwzględniający stringi i znaki ucieczki.
- `local_repair`: prosty skaner znak po znaku, który wie, czy jest wewnątrz stringa (żeby nie ruszać treści).
- `looks_truncated`: otwarty ```` ``` ```` bez zamknięcia albo dodatnia głębokość nawiasów na końcu tekstu.
- Korzeń inny niż obiekt → błąd „Oczekiwano obiektu JSON”.

## M2.9 `tests/test_json_extract.py` (≈200 linii)

**Wklej:** kartę, I-json.

Przypadki: czysty blok ```` ```json ````; tekst przed i po bloku; blok bez oznaczenia języka; JSON bez bloku w środku zdania; dwa bloki (wygrywa ```` ```json ````); przecinek końcowy; BOM; znak nowej linii w stringu; cudzysłowy „” wewnątrz wartości zostają nietknięte; `[1]` jako wartość tablicy nie jest usuwane; `[^1^]` i `¹` są usuwane; ucięty JSON → `looks_truncated=True`; tablica jako korzeń → błąd; pusty tekst → `obj is None`.

## M2.10 `validation.py` (≈120 linii)

**Wklej:** kartę, I-errors, I-validation.

- `validate`: `Draft202012Validator(schema).iter_errors(obj)`, sortowanie po ścieżce, komunikat `"/properties/…: opis"` (ścieżka z `absolute_path`, `/` dla korzenia).
- `with_enum`: głęboka kopia, nawigacja po JSON Pointer (obsługa `~0`, `~1`).

## M2.11 `tests/test_validation.py` (≈100 linii)

**Wklej:** kartę, I-errors, I-validation.

Przypadki: poprawny obiekt → `[]`; brak wymaganego pola; zły typ w zagnieżdżeniu (ścieżka w komunikacie); `max_errors`; `check_schema` przy błędnym schemacie → `INVALID_INPUT`; `with_enum` nie zmienia oryginału; `nullable=True` dopuszcza `null`; zły pointer → `INVALID_INPUT`; `get_pointer`.

## M2.12 `prompts.py` (≈260 linii)

**Wklej:** kartę, I-prompts, sekcje „Zasady podstawiania”, „Szablon JSON”, „Szablon z narzędziami”, „Szablon zbiorczy”, „Szablon naprawczy” z 03.

- Szablony jako stałe tekstowe dosłownie z 03, podstawianie wyłącznie przez `str.replace`.
- `build_json_prompt`: `@@STATUS_RULE@@` zgodnie z warunkiem z 03; końcowe puste linie usunięte.
- `build_tools_prompt`: reguły według tabeli z 03; `tool_choice` jako słownik → wymuszona funkcja.
- `build_repair_prompt`: najwyżej 10 błędów; wariant `truncated`; warunek długości schematu 4000 znaków.

## M2.13 `tests/test_prompts.py` (≈120 linii)

**Wklej:** kartę, I-prompts.

Przypadki: `request_id` występuje w prompcie i w obu delimiterach; klamry z danych nie psują szablonu (dane z `{}` i `@@` w treści); reguła statusu jest tylko przy schemacie z `status`, `confidence`, `needs_review`; każdy wariant `tool_choice` daje właściwe zdanie; `format_tools` bez opisu → „(brak opisu)”; naprawa ucina listę do 10 błędów i przy długim schemacie pisze „Schemat jak w poprzedniej wiadomości.”.

## M2.14 `tasks/registry.py` (≈230 linii)

**Wklej:** kartę, I-errors, I-config (`Mode`), I-schemas, I-validation, I-tasks, sekcję „Format pliku zadania” z 03.

- `load()`: dla każdego katalogu z listy pliki `*.yaml` w kolejności alfabetycznej; `output_schema: default|text` rozwijane przez `schemas.named_schema`; `check_schema` dla obu schematów; nazwa z pliku musi zgadzać się z nazwą pliku (bez `.yaml`); błędny plik → `logger.warning` i pominięcie.
- `render()`: walidacja wejścia; `string.Template(instructions).safe_substitute(mapping)`, gdzie mapping zawiera wszystkie pola wejścia (str bez zmian, inne `json.dumps(ensure_ascii=False)`) i wartość „brak” dla nazw występujących w szablonie, a nieobecnych w wejściu; `enum_from` przez `with_enum` z `nullable=False`, a wskazane pole trafia też do `required`.
- Utwórz pusty `tasks/__init__.py` ręcznie.

## M2.15 `tasks/builtin/decide.yaml` i `ask.yaml` (skopiuj z 03)

Pliki są w całości w `03-prompty-modelu.md`. Skopiuj je bez zmian.

## M2.16 `tests/test_registry.py` (≈150 linii)

**Wklej:** kartę, I-errors, I-tasks, treść `decide.yaml` z 03.

Przypadki (pliki zadań tworzone w `tmp_path`): wczytanie wbudowanych `decide` i `ask` (katalog z `paths.builtin_tasks_dir()`); nieznane zadanie → `UNKNOWN_TASK`; `render` dla `decide` daje `enum` z opcji w `/properties/decision` i opcje w instrukcji; brak `options` → `INVALID_INPUT`; zadanie z katalogu użytkownika nadpisuje wbudowane; błędny YAML jest pomijany bez wyjątku; `ref == "decide@1"`.

## M2.17 `backends/base.py` (≈90 linii)

**Wklej:** kartę, I-config (`Mode`), I-backend.

Dataclasses i `Protocol` dokładnie według I-backend. Utwórz ręcznie pusty `backends/__init__.py` (uzupełniony w M2.29).

## M2.18 `backends/mock.py` (≈230 linii)

**Wklej:** kartę, I-config, I-paths, I-backend, I-schemas.

- `class MockBackend` implementuje `Backend`, `name = "mock"`, stan od razu `ready` po `start()`.
- `synthesize(schema: dict, request_id: str) -> dict` (publiczna funkcja modułu): przykładowy obiekt ze schematu: `enum` → pierwsza wartość; `string` → „przykład”; `number` → 0.9; `integer` → 1; `boolean` → False; `array` → jeden element; `object` → wymagane pola; typ `[x, "null"]` → x; pole `request_id` → podany identyfikator; pole `message` → „Odpowiedź testowa (mock).”.
- `send()`:
  1. `request_id` wyciągnięty z promptu wyrażeniem `req_[0-9a-f]{20}` (brak → z `req.request_id`),
  2. nagranie: `sha256` promptu z identyfikatorem zamienionym na `<RID>`; jeśli w `settings.mock_fixtures_dir` jest `<hash>.json` z polem `text`, zwróć ten tekst z `<RID>` zamienionym na bieżący identyfikator,
  3. inaczej synteza: przy `hints["tools"]` i braku `hints["has_tool_results"]` i `tool_choice != "none"` → wywołanie narzędzia (wymuszone albo pierwsze) z argumentami z `synthesize(parameters)`; przy `hints["batch_ids"]` → `{"request_id", "items": [{"id", "result"}]}`; inaczej `synthesize(hints["schema"])`,
  4. wynik opakowany w blok ```` ```json ````; `conversation_url = "mock://conv/<n>"`.
- `release`, `tick`, `logout` nic nie robią; `login` zwraca `"ready"`.
- Opcjonalna kolejka odpowiedzi dla testów: `mock.script(replies: list[str])`: kolejne `send()` zwracają te teksty zamiast syntezy.

## M2.19 `tests/test_mock_backend.py` (≈120 linii)

**Wklej:** kartę, I-backend, specyfikację M2.18.

Przypadki: synteza decyzji z `enum` daje pierwszą opcję i poprawny `request_id`; narzędzia → wywołanie pierwszego narzędzia; wymuszone narzędzie; `has_tool_results` → odpowiedź końcowa; batch zwraca wszystkie `id`; nagranie z katalogu ma pierwszeństwo; `script()` zwraca teksty po kolei.

## M2.20 `tool_calls.py` (≈170 linii)

**Wklej:** kartę, I-validation, I-schemas, I-tools.

- `validate_tool_output`: najpierw `validate(obj, tool_output_schema(...))`, potem reguły z I-tools; argumenty każdego wywołania walidowane schematem `parameters` narzędzia (pusty schemat = dowolny obiekt).
- Komunikaty po polsku, np. `tool_calls[0]: nieznane narzędzie "x"`.

## M2.21 `tests/test_tool_calls.py` (≈130 linii)

**Wklej:** kartę, I-tools.

Przypadki: poprawne wywołanie; nieznane narzędzie; złe argumenty; `required` z odpowiedzią końcową → błąd; wymuszone narzędzie inne niż wywołane → błąd; dwa wywołania przy `allow_parallel=False` → błąd; `final` bez `content` → błąd; `final` z `data` zgodnym z `final_schema`; `split_output` dla trzech rodzajów.

## M2.22 `engine.py` (≈280 linii)

**Wklej:** kartę, I-errors, I-config, I-logging, I-backend, I-json, I-validation, I-schemas, I-prompts, I-tools, I-engine.

`Engine.complete(req)`:
1. Dobór trybu: `tools` i `tool_choice != "none"` → narzędzia; inaczej `schema` albo `TEXT_CONTENT_SCHEMA`. Schemat przez `ensure_request_id`.
2. `len(req.data) > settings.limits.max_input_chars` → `INPUT_TOO_LARGE`.
3. Prompt: `build_tools_prompt` albo `build_json_prompt`.
4. `backend.send(BackendRequest(…, hints={**req.hints, "schema", "tools", "tool_choice", "final_schema"}))`.
5. `parse_reply(reply.text)`; gdy `obj is None`, druga próba na `reply.full_text`.
6. Błędy: walidacja schematu albo `validate_tool_output`; dodatkowo `request_id` różny od oczekiwanego → błąd „request_id niezgodny”.
7. Naprawa: do `settings.repair.max_attempts` razy `build_repair_prompt(…, truncated=reply.truncated or parse.looks_truncated)` wysłane z `conversation_url` poprzedniej odpowiedzi.
8. Sukces: `split_output` (tryb narzędzi) albo `kind="json"`; w trybie tekstowym `kind="text"`, `text=data["content"]`, `data=None`.
9. Porażka: `TRUNCATED` (gdy ucięte), `INVALID_JSON` (gdy nic nie sparsowano), `INVALID_TOOL_CALL` (tryb narzędzi) albo `SCHEMA_MISMATCH`; `details={"errors": [...], "raw": ostatni tekst}`.
10. Zawsze na końcu (także po błędzie): `backend.release(conversation_url)`, chyba że `keep_conversation`.
11. Logi: `mask()` dla promptu i odpowiedzi; czas i liczba prób.

## M2.23 `tests/test_engine.py` (≈220 linii)

**Wklej:** kartę, I-engine, I-backend, specyfikację M2.18 (`script()`), specyfikację M2.22.

Przypadki (MockBackend ze `script()`): poprawna odpowiedź za pierwszym razem (`attempts=1`, `first_try_valid=True`); zły JSON, potem poprawny (`attempts=2`); dwa złe → `INVALID_JSON`; niezgodność ze schematem → `SCHEMA_MISMATCH`; niezgodny `request_id` → naprawa; tryb tekstowy zwraca `text`; narzędzia → `kind="tool_calls"`; za długie dane → `INPUT_TOO_LARGE`; `release` wywołane także po błędzie (podmieniona metoda licząca wywołania).

## M2.24 `storage.py` (≈300 linii)

**Wklej:** kartę, I-storage (z SQL).

- Jedno połączenie `sqlite3.connect(str(path), check_same_thread=False)`, `row_factory = sqlite3.Row`, każda operacja pod `threading.Lock`, `commit()` po zapisie.
- JSON w kolumnach przez `json.dumps(ensure_ascii=False)`.
- `cache_get` usuwa przeterminowany wpis i zwraca `None`.
- `job_get` zwraca słownik z polami `JobInfo` i `request`, `result` jako obiekty.

## M2.25 `tests/test_storage.py` (≈150 linii)

**Wklej:** kartę, I-storage.

Przypadki: `init_schema` dwa razy bez błędu; zapis i odczyt decyzji (kolejność od najnowszych, filtr `task`); cache z TTL 0 wygasa; cykl joba `queued → running → done`; `jobs_fail_unfinished` zmienia tylko niedokończone; rozmowa: `conv_update` zwiększa `turns`; idempotencja.

## M2.26 `metrics.py` (≈120 linii)

**Wklej:** kartę, I-metrics.

p95 z posortowanej listy ostatnich czasów; stopy jako ułamki 0–1 zaokrąglone do 3 miejsc; `snapshot()` dla zera zapytań zwraca zera, bez dzielenia przez zero.

## M2.27 `task_runner.py` (≈300 linii)

**Wklej:** kartę, I-errors, I-config, I-models, I-schemas, I-tasks, I-engine, I-storage, I-metrics, I-runner.

`TaskRunner.run(job)`:
1. `task = registry.get(job.task)`; `instructions, data, schema = registry.render(task, job.input)`; `job.output_schema` nadpisuje schemat.
2. Dla `kind="decide"`: `fallback_decision` spoza listy opcji (pole z `enum_from`) → `INVALID_INPUT`.
3. Cache: gdy `settings.cache.enabled`, `job.use_cache`, brak plików i brak `conversation_id`: klucz `sha256` z `json.dumps({"task": task.ref, "input": job.input, "schema": schema, "mode": tryb, "prompt": ENVELOPE_VERSION, "votes": głosy}, sort_keys=True)`; trafienie → koperta z `meta.cached=True` i nowym `request_id`.
4. Rozmowa: przy `conversation_id` pobierz `storage.conv_get` (brak → `NOT_FOUND`), użyj `chat_url` i `keep_conversation=True`; po sukcesie `conv_update`.
5. Wywołanie `engine.complete` `votes` razy (limit `settings.decide.max_votes`), każde jako nowa rozmowa (poza trybem rozmowy, gdzie głosowanie jest wyłączone).
6. Decyzje (`kind="decide"`), po zebraniu wyników:
   - większość według `decision`; remis → opcja z wyższą średnią `confidence`,
   - `agreement` = liczba głosów zwycięskiej opcji / liczba głosów (`None` przy 1 głosie),
   - dane z najpewniejszej odpowiedzi zwycięskiej opcji, `data["votes"] = {opcja: liczba}` przy głosowaniu,
   - `needs_review = True`, gdy `status != "ok"`, `confidence < próg` (`job.threshold` albo `settings.decide.default_threshold`) albo `agreement < 2/3`,
   - gdy `needs_review` i jest `fallback_decision`: `data["original_decision"] = decision`, `decision = fallback`, `meta.fallback_applied = True`.
7. `Meta`: czasy, próby (suma), `first_try_valid` (wszystkie głosy), `schema_name` (nazwa zadania albo „custom”), `task_version`, `prompt_version`, `sources`, `backend`.
8. `include_raw` → `raw` z ostatniego wyniku (przy błędzie z `details["raw"]`).
9. Zapis: `storage.log_request`, przy `decide` `storage.log_decision`, `metrics.record`, cache.
10. Wyjątki: `BridgeError` → koperta błędu; inne → `INTERNAL` z `logger.exception`. Nigdy nie rzuca.

## M2.28 `tests/test_task_runner.py` (≈220 linii)

**Wklej:** kartę, I-runner, I-models, I-backend, specyfikację M2.18 i M2.27.

Przypadki (prawdziwe `TaskRegistry`, `Storage` w `tmp_path`, `MockBackend`): `decide` zwraca decyzję z listy opcji i wpis w dzienniku; niska `confidence` (odpowiedź ze `script()`) → `needs_review` i decyzja zastępcza; `fallback_decision` spoza listy → `INVALID_INPUT` w kopercie; głosowanie 3 razy z odpowiedziami A, A, B → A i `agreement≈0.667`; drugie identyczne zapytanie → `cached=True`; nieznane zadanie → koperta z `UNKNOWN_TASK`; `ask` zwraca `message`.

## M2.29 `backends/__init__.py` (≈35 linii)

**Wklej:** kartę, I-config, I-paths, I-backend.

```python
def make_backend(settings: Settings) -> Backend: ...
```

`backend == "mock"` → `MockBackend(settings)`. Inaczej import wewnątrz funkcji: `from .playwright_backend import PlaywrightBackend` (plik powstanie w M5); przy `settings.recording.enabled` owinięcie w `RecordingBackend(backend, paths.recordings_dir())` z `.recorder`.

## M2.30 Sprawdzenie końcowe etapu

1. `python -m pytest` zielony.
2. Poproś Copilota o krótki skrypt `tools/m2_demo.py` (≈40 linii, szablon A, wklej I-runner, I-tasks, I-storage, I-metrics, I-engine, I-backend), który buduje `Settings(backend="mock")`, rejestr, bazę w katalogu tymczasowym, `Engine`, `TaskRunner` i wypisuje kopertę dla `decide` z trzema opcjami.
3. Koperta ma `ok: true`, `decision` z listy opcji i wypełnione `meta`.
