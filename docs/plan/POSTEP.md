# Postęp copilot-bridge

Skopiuj ten plik do katalogu projektu na służbowym laptopie (`C:\dev\copilot-bridge\POSTEP.md`) i aktualizuj tam.

## Zasady

- Krok odhaczasz (`[x]`) dopiero, gdy przeszło jego sprawdzenie: `py_compile` i test z kroku.
- Po każdym odhaczonym kroku commit z numerem kroku w opisie, np. `M2.8 json_extract.py`. Bez gita: kopia katalogu po każdym etapie.
- Przed przerwą uzupełnij sekcję „Gdzie jestem”. Po przerwie zacznij od niej.
- Odstępstwa od planu (inna nazwa, dodatkowe pole, pominięty krok) zapisuj w „Odstępstwach”. Przy kolejnych krokach dopisuj je do bloków interfejsów, które wklejasz Copilotowi.

## Gdzie jestem

- **Następny krok:** M1.1
- **W trakcie / problem:** –
- **Ostatnia sesja:** –

## Odstępstwa od planu

| Krok | Co jest inaczej | Wpływ na późniejsze kroki |
|---|---|---|
| – | – | – |

## M1. Rozpoznanie

- [ ] M1.1 `pyproject.toml` + `pip install -e ".[dev]"`
- [ ] M1.2 `__init__.py` (ręcznie)
- [ ] M1.3 `paths.py`
- [ ] M1.4 `browser/selectors.yaml` (kandydaci)
- [ ] M1.5 `tools/m1_probe.py`
- [ ] M1.6 `tools/m1_ask.py`
- [ ] M1.7 `tools/m1_soak.py`
- [ ] M1.8a logowanie i `dump`
- [ ] M1.8b selektory działają (`check-selectors`)
- [ ] M1.8c `json-test --repeat 10` w trybach okna
- [ ] M1.8d `length-test` i `upload`
- [ ] M1.8e historia, usuwanie rozmów, pamięć (ręcznie)
- [ ] M1.8f `m1_soak` przez dzień z uśpieniami
- [ ] M1.8g `docs/m1-wyniki.md` wypełniony, decyzje z tabeli przeniesione do „Odstępstw”

## M2. Rdzeń

- [ ] M2.1 `errors.py` · [ ] M2.2 test
- [ ] M2.3 `config.py` · [ ] M2.4 test
- [ ] M2.5 `logging_setup.py`
- [ ] M2.6 `models.py`
- [ ] M2.7 `schemas.py`
- [ ] M2.8 `json_extract.py` · [ ] M2.9 test
- [ ] M2.10 `validation.py` · [ ] M2.11 test
- [ ] M2.12 `prompts.py` · [ ] M2.13 test
- [ ] M2.14 `tasks/registry.py` · [ ] M2.15 `decide.yaml`, `ask.yaml` · [ ] M2.16 test
- [ ] M2.17 `backends/base.py`
- [ ] M2.18 `backends/mock.py` · [ ] M2.19 test
- [ ] M2.20 `tool_calls.py` · [ ] M2.21 test
- [ ] M2.22 `engine.py` · [ ] M2.23 test
- [ ] M2.24 `storage.py` · [ ] M2.25 test
- [ ] M2.26 `metrics.py`
- [ ] M2.27 `task_runner.py` · [ ] M2.28 test
- [ ] M2.29 `backends/__init__.py`
- [ ] M2.30 `pytest` zielony + `tools/m2_demo.py`

## M3. Daemon, API, CLI

- [ ] M3.1 `worker.py` · [ ] M3.2 test
- [ ] M3.3 `api/deps.py`
- [ ] M3.4 `api/app.py`
- [ ] M3.5 `api/routes_core.py`
- [ ] M3.6 `tests/conftest.py` · [ ] M3.7 `tests/test_api_core.py`
- [ ] M3.8 `daemon/main.py` (+ `__init__.py`, `__main__.py`)
- [ ] M3.9 `client.py`
- [ ] M3.10 `launcher.py`
- [ ] M3.11 `cli.py`
- [ ] M3.12 `cli_admin.py`
- [ ] M3.13 sprawdzenie ręczne (auto-start, status, zamknięcie terminala)

## M4. Adapter OpenAI

- [ ] M4.1 `openai_compat/models.py`
- [ ] M4.2 `openai_compat/convert.py` · [ ] M4.3 test
- [ ] M4.4 `openai_compat/routes.py`
- [ ] M4.5 `tests/test_openai_compat.py`
- [ ] M4.6 `examples/python/openai_tools_loop.py` działa na `mock`

## M5. Przeglądarka

- [ ] M5.1 `browser/selectors.py`
- [ ] M5.2 `browser/session.py`
- [ ] M5.3 `browser/chat.py`
- [ ] M5.4 `browser/waiter.py`
- [ ] M5.5 `browser/extractor.py`
- [ ] M5.6 `browser/wake.py` · [ ] M5.7 test
- [ ] M5.8 `backends/playwright_backend.py`
- [ ] M5.9 `backends/recorder.py`
- [ ] M5.10a decyzja z prawdziwego Copilota
- [ ] M5.10b uśpienie 15 min i dalsza praca
- [ ] M5.10c timeout i anulowanie
- [ ] M5.10d pętla narzędzi na prawdziwym Copilocie
- [ ] M5.10e nagrania skopiowane do `tests\fixtures\recordings\`

## M6. Rozszerzenia

- [ ] M6.1 `batch.py` · [ ] M6.2 test
- [ ] M6.3 `api/routes_jobs.py`
- [ ] M6.4 `api/uploads.py` (albo pominięty po M1: wpisz w „Odstępstwa”)
- [ ] M6.5 `classify.yaml`, `summarize.yaml`, `extract.yaml`
- [ ] M6.6 `rewrite.yaml`, `draft.yaml`, `work_search.yaml`
- [ ] M6.7 `tests/test_jobs_api.py`
- [ ] M6.8 batch 50 elementów, załącznik, rozmowa, ewentualnie `history.cleanup: delete`

## M7. Wykończenie

- [ ] M7.1 `static/playground.html`
- [ ] M7.2 `doctor.py`
- [ ] M7.3 `evals.py` · [ ] M7.4 `evals/decide_cases.yaml` (15–30 przypadków)
- [ ] M7.5 `autostart.py`
- [ ] M7.6 `tests/fake_copilot/index.html` · [ ] M7.7 `tests/fake_selectors.yaml` · [ ] M7.8 test
- [ ] M7.9 przykłady: [ ] Python · [ ] PowerShell · [ ] Power Query · [ ] VBA · [ ] PAD
- [ ] M7.10 `README.md`
- [ ] M7.11 lista końcowa (pytest, doctor, eval, autostart po restarcie, openapi.json)

## M8. Przepływy

- [ ] M8.0 `tools/m8_check.py` + wynik (Outlook COM / web, SharePoint, schtasks, profil Edge)
- [ ] M8.1 zależności, `FlowsCfg`, nowe ścieżki w `paths.py`
- [ ] M8.2 `flows/model.py`
- [ ] M8.3 `flows/template.py` · [ ] test
- [ ] M8.4 `flows/steps/base.py`
- [ ] M8.5 `flows/store.py` · [ ] test
- [ ] M8.6 `flows/runner.py` · [ ] test
- [ ] M8.7 `steps/control.py`
- [ ] M8.8 `steps/ai.py`
- [ ] M8.9 `steps/excel.py` · [ ] test
- [ ] M8.10 `steps/files.py` · [ ] test
- [ ] M8.11 `steps/outlook_com.py` (albo M8.11w `outlook_web.py`)
- [ ] M8.12 `flows/browser_profile.py`
- [ ] M8.13 `steps/browser.py`
- [ ] M8.14 `flows/schedule.py` · [ ] test
- [ ] M8.15 `flows/cli.py`, `flows/__main__.py`, zmiana `cli.py`
- [ ] M8.16 `flows/card_header.md` + `flow docs`
- [ ] M8.17 przykłady: [ ] faktury-z-maila · [ ] raport-dzienny · [ ] erp-statusy + `erp_status.py`
- [ ] M8.18 sprawdzenie końcowe (dry-run, once_per, harmonogram po uśpieniu, przepływ od AI)

## Dziennik sesji

| Data | Kroki | Uwagi |
|---|---|---|
| – | – | – |
