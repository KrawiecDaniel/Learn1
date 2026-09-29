# M5. Backend przeglądarki (Edge + Playwright)

**Cel:** prawdziwy backend: Edge z osobnym profilem, logowanie, wysyłanie promptów, czekanie na odpowiedź, ekstrakcja, wykrywanie uśpienia i powrót do pracy. Wszystko według wyników M1.

**Gotowe, gdy:** przy `backend: playwright` `copilot-bridge task decide …` zwraca decyzję z prawdziwego Copilota, a po uśpieniu laptopa na 15 minut kolejne zapytanie działa bez ręcznej interwencji.

Przed startem: `selectors.yaml` z M1 zawiera działające selektory, a `docs/m1-wyniki.md` jest wypełniony. Do każdego kroku z tego etapu dołącz Copilotowi odpowiednie fragmenty wyników M1 (np. czy `code_block` daje czysty JSON).

## M5.1 `browser/selectors.py` (≈150 linii)

**Wklej:** kartę, I-errors, I-paths, I-browser (część `selectors.py`).

- `load`: YAML z `paths.builtin_selectors_file()`, potem nadpisanie kluczy z pliku `override` (jeśli podany i istnieje); wartości zawsze listy stringów.
- `first`: dla każdego kandydata `root.locator(k).first.wait_for(state=state, timeout=…)` z podziałem czasu: pierwszy kandydat dostaje połowę limitu, reszta równo resztę. Brak kandydatów albo wszystkie zawiodły → `BridgeError(UI_CHANGED, f"Nie znaleziono elementu '{name}'", details={"candidates": [...]})`.
- `any_visible`: `root.locator(k).first.is_visible()` dla kandydatów, wyjątki traktowane jako `False`.
- `all`: `root.locator(k1).or_(root.locator(k2))…`.

## M5.2 `browser/session.py` (≈260 linii)

**Wklej:** kartę, I-errors, I-paths, I-config, I-browser (części `selectors.py` i `session.py`), punkt 6 z „Pułapek Windows” w 01, wyniki M1 dotyczące trybów okna i logowania.

- `start(window)`: `sync_playwright().start()`; `launch_persistent_context(str(profile_dir()), channel=browser.channel, headless=window == "headless", no_viewport=True, slow_mo=browser.slow_mo_ms, args=…)`; argumenty okna jak w M1.5; strona `ctx.pages[0]` albo `new_page()`; `page.set_default_timeout(15000)`. Błąd uruchomienia (np. profil w użyciu) → `BridgeError(INTERNAL, "Nie udało się uruchomić Edge: …")`.
- `window=None` → `settings.browser.window`.
- `open_chat_home`: `goto(url, wait_until="domcontentloaded", timeout=page_load_s)`, potem do `page_load_s` pętla co 0,5 s: `is_login_page()` → koniec; `input_box` widoczny → koniec.
- `is_login_page`: domeny `login.microsoftonline.com`, `login.live.com` w URL albo widoczny `login_marker`.
- `is_ready(timeout_ms)`: `input_box` widoczny (bez czekania przy 0).
- `wait_for_login(wait_s)`: co 2 s sprawdza `is_ready`; `True`, gdy gotowe przed końcem czasu.
- `debug_dump(name)`: zrzut ekranu i `page.content()` do `debug_dir()` z nazwą `<czas>-<name>`; błędy połykane (zwraca `None`).
- `stop`: zamknięcie kontekstu i Playwrighta, wyjątki logowane.

## M5.3 `browser/chat.py` (≈280 linii)

**Wklej:** kartę, I-errors, I-config, I-browser (części `selectors.py`, `session.py`, `chat.py`), wyniki M1 (selektory, przełącznik trybu, załączniki, usuwanie rozmów).

- `new_chat`: klik `new_chat_button`, gdy jest; inaczej `goto(url)`; potem czekanie na `input_box`.
- `open_conversation(url)`: `goto(url)` tylko, gdy bieżący URL jest inny; czekanie na `input_box`.
- `set_mode(mode)`: klik `mode_work` albo `mode_web`, gdy selektor istnieje; brak → jednorazowe ostrzeżenie w logu (pamiętane w atrybucie obiektu).
- `attach(files)`:
  - sprawdzenie liczby i rozmiaru (`limits.max_files`, `max_file_mb`) → `ATTACHMENT_REJECTED`,
  - z `attach_button`: `with page.expect_file_chooser() as fc: przycisk.click()`, `fc.value.set_files(...)`; bez niego: `selectors.first(page, "file_input", state="attached").set_input_files(...)`,
  - czekanie na `attachment_ready` (gdy zdefiniowany) do 60 s, inaczej 3 s.
- `send(text)`: `box = first(input_box)`, `box.click()`, `box.fill(text)`; klik `send_button`, gdy jest i jest aktywny, inaczej `box.press("Enter")`.
- `assistant_messages`: `selectors.all(page, "assistant_message")`; `count_assistant_messages`: `count()`.
- `stop_generation`: klik `stop_button`, jeśli widoczny.
- `delete_current_chat`: `chat_menu_button` → `delete_chat_button` → `confirm_delete_button`; brak któregokolwiek selektora → `False` bez wyjątku.

## M5.4 `browser/waiter.py` (≈200 linii)

**Wklej:** kartę, I-errors, I-config, I-browser (części `selectors.py`, `chat.py`, `waiter.py`), wyniki M1 (pytania 5, 6, 15, 16).

`wait_for_reply(chat, selectors, prev_count, timeouts, cancel_event)`:
1. Faza startu (do `reply_start_s`): co 0,25 s sprawdza: anulowanie → `stop_generation()` i `CANCELLED`; strona logowania → `AUTH_REQUIRED`; `rate_limit_banner` → `RATE_LIMITED` z `retry_after_s=60`; nowa wiadomość (`count > prev_count`) albo widoczny `stop_button` → faza 2. Koniec czasu → `TIMEOUT` („Copilot nie zaczął odpowiadać”).
2. Faza pisania (do `response_s` od wysłania): `message = chat.assistant_messages().last`; tekst przez `inner_text()` (wyjątek → pusty); zmiana tekstu zeruje licznik stabilności; koniec, gdy `stop_button` niewidoczny, tekst niepusty i bez zmian przez `stable_ms`. W każdej iteracji sprawdzenie anulowania, logowania i `rate_limit_banner`. Widoczny `error_banner` przy niewidocznym `stop_button` → `INTERRUPTED` z treścią komunikatu w `details`.
3. Koniec czasu → `stop_generation()` i `TIMEOUT`.
4. Zwraca locator ostatniej wiadomości.

## M5.5 `browser/extractor.py` (≈180 linii)

**Wklej:** kartę, I-browser (części `selectors.py`, `extractor.py`), wyniki M1 (pytania 7, 8, 16).

`extract_reply(message, selectors)`:
1. `sources`: dla `sources_link` w `message`: `evaluate_all` zwracające `[{title: textContent.trim(), url: href}]`, bez duplikatów URL, tylko `http(s)`.
2. Usunięcie znaczników cytowań z DOM tej wiadomości: `message.locator(sel).evaluate_all("els => els.forEach(e => e.remove())")` dla każdego kandydata `citation_marker`.
3. `code_text`: `inner_text()` ostatniego `code_block` w wiadomości (brak → `None`).
4. `full_text`: `message.inner_text()`.
5. `truncated`: widoczny `truncated_marker` w wiadomości.

Jeśli M1 wykazał, że blok kodu jest zniekształcony, a przycisk „Kopiuj” daje czysty tekst, dodaj wariant: klik przycisku kopiowania i odczyt schowka przez `page.evaluate("navigator.clipboard.readText()")` po nadaniu uprawnień `context.grant_permissions(["clipboard-read"])`. Opisz to Copilotowi jako dodatkowy punkt specyfikacji.

## M5.6 `browser/wake.py` (≈160 linii)

**Wklej:** kartę, I-config, I-backend (`BackendState`), I-browser (części `session.py`, `wake.py`), wyniki M1 (pytanie 14).

- `WakeDetector`: `check_once()` porównuje `clock()` z poprzednim pomiarem; przerwa większa niż `gap_s` → zapis `last_wake_at = teraz`, flaga do `consume()`, log INFO „wykryto wybudzenie po N s”. Wątek `heartbeat` wywołuje `check_once()` co `heartbeat_s` (przez `threading.Event.wait`, żeby `stop()` działał od razu). Dostęp do pól pod `threading.Lock`.
- `woke_since(t)`: `last_wake_at is not None and last_wake_at >= t`.
- `recover_after_wake(session, timeout_s)`: pętla do `timeout_s`: `page.reload(wait_until="domcontentloaded", timeout=30000)`; strona logowania → `"auth_required"`; `is_ready(20000)` → `"ready"`; wyjątek „Target closed” albo podobny → `session.restart()`, potem `open_chat_home()`; odstępy 5, 10, 20, 30, 30… s. Koniec czasu → `"degraded"`. Jeśli M1 wykazał, że samo przeładowanie nie wystarcza, pierwsza próba to od razu `restart()`.

## M5.7 `tests/test_wake.py` (≈90 linii)

**Wklej:** kartę, I-browser (część `wake.py`).

Przypadki z podstawionym zegarem (lista czasów): brak przerwy → `False`; przerwa 100 s przy `gap_s=30` → `True`, `consume()` zwraca `True` raz, potem `False`; `woke_since` przed i po wybudzeniu. Bez uruchamiania wątku i bez przeglądarki.

## M5.8 `backends/playwright_backend.py` (≈320 linii)

**Wklej:** kartę, I-errors, I-config, I-paths, I-backend, I-browser (całość), `make_backend` z M2.29.

`class PlaywrightBackend`, `name = "m365-playwright"`:
- `__init__`: zapisuje ustawienia, stan `"starting"` (pole chronione `threading.Lock`), bez uruchamiania przeglądarki.
- `start()`: `Selectors.load(browser.selectors_file)`, `EdgeSession`, `start()`, `open_chat_home()`, stan `ready` albo `auth_required`, `ChatPage`, `WakeDetector(...).start()`.
- `send(req)`:
  1. stan `auth_required` → `AUTH_REQUIRED`; `resuming` albo `degraded` → najpierw `tick()`,
  2. odstęp od poprzedniego wysłania co najmniej `limits.min_interval_s` (`time.sleep` różnicy),
  3. `open_conversation(req.conversation_url)` albo `new_chat()`, `set_mode`, `attach` (gdy pliki), `prev = count_assistant_messages()`, `send(req.prompt)`,
  4. `wait_for_reply` z `timeouts` (limit odpowiedzi `min(req.timeout_s, timeouts.response_s)`), `extract_reply`,
  5. `BackendReply(text=code_text or full_text, full_text, sources, conversation_url=current_url(), truncated, duration_ms)`,
  6. obsługa błędów:
     - wybudzenie w trakcie (`wake.woke_since(start)`) → `INTERRUPTED`,
     - strona logowania → stan `auth_required`, `AUTH_REQUIRED`,
     - `UI_CHANGED` → `debug_dump(request_id)`; trzy takie błędy z rzędu → stan `degraded`,
     - wyjątki Playwrighta (`playwright.sync_api.Error`, `TimeoutError`) → zamknięta karta albo przeglądarka: `session.restart()` i `INTERRUPTED`; pozostałe: `debug_dump` i `UI_CHANGED`.
- `release(url)`: przy `history.cleanup == "delete"` i gdy bieżąca rozmowa to `url`: `delete_current_chat()` (błędy tylko logowane).
- `tick()`: `wake.consume()` → stan `resuming`, `recover_after_wake(session, settings.wake.resume_timeout_s)`, nowy stan; poza tym co `browser.refresh_every_min` minut bezczynności `open_chat_home()`; stan `degraded` → próba `recover_after_wake`.
- `login(wait_s)`: `session.restart(window="normal")`, `open_chat_home()`, `wait_for_login(wait_s)`, `session.restart()` w trybie z konfiguracji, `open_chat_home()`, stan według `is_ready`/`is_login_page`.
- `logout()`: `session.stop()`, usunięcie katalogu `profile_dir()` (`shutil.rmtree`, ponowienie po 1 s przy `PermissionError`), `session.start()`, `open_chat_home()`, stan `auth_required`.
- `stop()`: zatrzymanie `WakeDetector` i sesji.

## M5.9 `backends/recorder.py` (≈100 linii)

**Wklej:** kartę, I-paths, I-backend, specyfikację nagrań z M2.18 (hash promptu z `<RID>`).

`class RecordingBackend`: przekazuje wszystkie metody do backendu wewnętrznego (`name` jak wewnętrzny). Po udanym `send()` zapisuje `recordings_dir() / <data> / <hash>.json` z polami `{"prompt", "text", "full_text", "sources", "duration_ms", "created_at"}`, gdzie `request_id` w `prompt` i `text` zamieniony na `<RID>`. Format zgodny z tym, co czyta `MockBackend` (pole `text`). Błąd zapisu tylko w logu.

## M5.10 Sprawdzenie ręczne

1. `config.yaml`: `backend: playwright`, `browser.window` według M1.
2. `copilot-bridge daemon restart`, potem `copilot-bridge login` i logowanie w oknie Edge.
3. `copilot-bridge task decide --set "question=Czy zatwierdzić zwrot 349 zł po 12 dniach?" --list "options=approve,reject,escalate" --set "criteria=Zwroty do 30 dni"`: koperta z decyzją.
4. `copilot-bridge ask "Podaj trzy zalety pracy zdalnej"`: `message` z odpowiedzią.
5. Test uśpienia: zamknij klapę na 15 minut, otwórz, od razu `copilot-bridge daemon status` (może pokazać `resuming`), potem ponów punkt 3.
6. Test anulowania: zadanie z `--timeout 5` → `TIMEOUT`, kolejne zadanie działa.
7. Adapter: `python examples\python\openai_tools_loop.py` na prawdziwym Copilocie.
8. Włącz `recording.enabled: true` na kilka zapytań: w `recordings\` pojawiają się pliki. Skopiuj kilka do `tests\fixtures\recordings\` na potrzeby M7.
