# M4. Adapter zgodny z OpenAI i function calling

**Cel:** skrypty w Pythonie mogą używać oficjalnej biblioteki `openai`, wskazując `base_url` na daemona: zwykłe pytania, odpowiedzi według `json_schema` i pełna pętla function calling. Silnik obsługuje narzędzia od M2, więc ten etap to tłumaczenie formatów i endpointy.

**Gotowe, gdy:** testy M4.5 są zielone na backendzie `mock`, a przykład M4.6 przechodzi pełną pętlę narzędzi.

## M4.1 `openai_compat/models.py` (≈170 linii)

**Wklej:** kartę, I-config (`Mode`), I-openai (część `models.py`).

- Modele pydantic v2 według I-openai; `ChatMessage`, `ResponseFormat`, `ChatCompletionRequest` z `model_config = ConfigDict(extra="allow")` (nieznane pola od bibliotek klienckich nie mogą powodować błędu).
- `openai_error_body`, `openai_error_type` według I-openai.
- Utwórz ręcznie pusty `openai_compat/__init__.py`.

## M4.2 `openai_compat/convert.py` (≈260 linii)

**Wklej:** kartę, I-errors, I-config, I-schemas, I-engine, I-openai, sekcję „Spłaszczanie rozmowy OpenAI” z 03.

- `flatten_messages`: dokładnie według 03.
- `to_completion_request(req, request_id, settings)`:
  - `mode = mode_for_model(req.model)`,
  - `instructions, data = flatten_messages(req.messages)`,
  - `response_format`: brak albo `text` → `schema=None`; `json_object` → `{"type": "object"}`; `json_schema` → `req.response_format.json_schema["schema"]` (brak klucza → `INVALID_INPUT`),
  - `tools` → lista słowników `model_dump()`; `tool_choice` domyślnie `"auto"`; `parallel_tool_calls` domyślnie `True`,
  - przy narzędziach i `json_schema` → `final_schema` = ten schemat,
  - `hints = {"has_tool_results": any(m.role == "tool" for m in req.messages)}`,
  - `timeout_s = settings.timeouts.response_s`.
- `to_chat_completion(result, req, completion_id)`:
  - `tool_calls` → `ToolCall(id=new_call_id(), function=ToolCallFunction(name, arguments=json.dumps(args, ensure_ascii=False)))`, `content=None`, `finish_reason="tool_calls"`,
  - `json` → `content = json.dumps(data bez "request_id", ensure_ascii=False)`, chyba że schemat klienta sam zawierał `request_id`,
  - `text` → `content = result.text`,
  - `usage`: `prompt_tokens = estimate_tokens` z `result.prompt_chars` (przez `"x" * n` albo bezpośrednio `n // 4 + 1`), `completion_tokens = estimate_tokens(result.raw)`,
  - `created = int(time.time())`, `model = req.model`.

## M4.3 `tests/test_openai_convert.py` (≈170 linii)

**Wklej:** kartę, I-openai, I-engine, sekcję „Spłaszczanie rozmowy OpenAI” z 03.

Przypadki: `system` + `user` → instrukcje i transkrypt; brak `system` → domyślna instrukcja; wiadomość `tool` dostaje nazwę narzędzia z wcześniejszego `tool_calls`; `content` jako lista części tekstowych; część obrazkowa → `INVALID_INPUT`; nieznany model → `INVALID_INPUT` z listą modeli; `json_schema` trafia do `schema`; narzędzia + `json_schema` → `final_schema`; `to_chat_completion` dla trzech rodzajów wyniku; `arguments` jest tekstem JSON; `ignored_params` wykrywa `temperature`.

## M4.4 `openai_compat/routes.py` (≈200 linii)

**Wklej:** kartę, I-errors, I-models (`new_request_id`), I-engine, I-worker, I-api, I-openai.

`router = APIRouter(prefix="/v1")`:
- `GET /models` → `{"object": "list", "data": [{"id": nazwa, "object": "model", "created": 0, "owned_by": "copilot-bridge"} …]}` dla `MODELS`.
- `POST /chat/completions`:
  1. `stream is True` → 400 w formacie OpenAI: „Strumieniowanie nie jest obsługiwane. Ustaw stream=false.”,
  2. `n` większe niż 1 → 400,
  3. `to_completion_request`, `WorkItem(kind="completion", priority=0)`, `submit_and_wait` z limitem `resolve_timeout(ctx, None)`,
  4. `to_chat_completion`, odpowiedź `JSONResponse` (`charset=utf-8`),
  5. nagłówek `X-Copilot-Bridge-Ignored` z listą `ignored_params` (gdy niepusta),
  6. `BridgeError` → `JSONResponse(status_code=err.http_status, content=openai_error_body(message, openai_error_type(status), code=err.code.value))`; przy `retry_after_s` nagłówek `Retry-After`,
  7. metryki: `ctx.metrics.record(task="chat.completions", …)`.

Router jest rejestrowany automatycznie przez `app.py` (M3.4), bez zmian w innych plikach.

## M4.5 `tests/test_openai_compat.py` (≈220 linii)

**Wklej:** kartę, I-openai, fixture `live_server` z M3.6, specyfikację M2.18, przykład z sekcji „Przykład function calling” z 03.

Klient: `openai.OpenAI(base_url=f"{base_url}/v1", api_key=token, max_retries=0)`.

Przypadki:
- `client.models.list()` zawiera `copilot-work`,
- zwykłe pytanie → `choices[0].message.content` niepuste, `finish_reason == "stop"`,
- `response_format` typu `json_schema` → `json.loads(content)` przechodzi walidację tego schematu,
- narzędzia: pierwsze wywołanie → `tool_calls`, `arguments` da się sparsować; drugie wywołanie z wiadomością `tool` → odpowiedź końcowa,
- `stream=True` → `openai.BadRequestError`,
- zły token → `openai.AuthenticationError`,
- nieznany model → `openai.BadRequestError` albo `UnprocessableEntityError` (422),
- `tool_choice` wymuszające konkretne narzędzie → wywołanie tego narzędzia.

## M4.6 `examples/python/openai_tools_loop.py` (≈120 linii)

**Wklej:** kartę, przykład function calling z 03.

Przykład dla użytkownika: token z `copilot_bridge.paths.ensure_token()`, klient `OpenAI(base_url="http://127.0.0.1:8765/v1", api_key=token, timeout=330, max_retries=1)`, dwa narzędzia lokalne (`get_order(id)`, `get_customer(id)` zwracające dane z fikcyjnego słownika), pętla: wywołanie → wykonanie narzędzi z `tool_calls` (walidacja nazwy i argumentów przed wykonaniem) → dopisanie wyników → ponowne wywołanie, maksymalnie 5 rund. Na końcu wypisanie odpowiedzi.

**Sprawdzenie:** przy daemonie na `mock`: `python examples\python\openai_tools_loop.py` kończy się odpowiedzią końcową po jednej rundzie narzędzi.
