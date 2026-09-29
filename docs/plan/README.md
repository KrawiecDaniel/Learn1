# Plan implementacji copilot-bridge

Plan budowy copilot-bridge krok po kroku, przygotowany pod pracę ze służbowym Copilotem, który pisze **jeden plik na raz** i nie widzi reszty projektu. Plan powstał na podstawie `docs/planning-prompt.md`.

Zasada główna: każdy krok to jeden plik kodu. Do każdego kroku wklejasz Copilotowi trzy rzeczy: **kartę projektu**, **bloki interfejsów** (z `02-kontrakty.md`) i **specyfikację pliku** (z pliku etapu). Dzięki blokom interfejsów pliki pisane w osobnych rozmowach pasują do siebie.

## Pliki planu

| Plik | Zawartość |
|---|---|
| `README.md` | Ten plik: sposób pracy, karta projektu, szablony promptów, etapy |
| `01-architektura.md` | Rozstrzygnięcia, architektura, wątki, struktura repozytorium, konfiguracja, pułapki Windows |
| `02-kontrakty.md` | Bloki interfejsów `I-…` dla każdego modułu, kody błędów, API, baza danych |
| `03-prompty-modelu.md` | Teksty promptów wysyłanych do M365 Copilota, schematy JSON, definicje zadań, przykłady |
| `M1-rozpoznanie.md` … `M7-wykonczenie.md` | Kroki etapów: jeden krok = jeden plik |
| `M8-przeplywy.md` | Lekkie przepływy YAML (Excel, Outlook, SharePoint, ERP + AI) z własnymi interfejsami |
| `99-ryzyka.md` | Ryzyka techniczne i rzeczy poza zakresem |
| `POSTEP.md` | Lista kontrolna wszystkich kroków do odhaczania, „Gdzie jestem”, odstępstwa od planu |

## Etapy

| Etap | Cel | Gotowe, gdy | Szacunek (1 osoba) |
|---|---|---|---|
| M1 Rozpoznanie | Sprawdzić na żywej stronie, jak zachowuje się M365 Copilot w Edge | Wypełnione `selectors.yaml` i `docs/m1-wyniki.md` | 1–2 dni |
| M2 Rdzeń | Prompty, parsowanie, walidacja, zadania, silnik, baza; bez przeglądarki i serwera | `pytest` zielony, decyzja na backendzie `mock` | 4–5 dni |
| M3 Daemon, API, CLI | Działający daemon z auto-startem, API i CLI na backendzie `mock` | `copilot-bridge task decide …` zwraca kopertę | 3–4 dni |
| M4 Adapter OpenAI | `/v1/chat/completions`, `/v1/models`, function calling | Biblioteka `openai` działa z daemonem, łącznie z pętlą narzędzi | 2–3 dni |
| M5 Przeglądarka | Prawdziwy backend Playwright + Edge, logowanie, uśpienie | Decyzja z prawdziwego Copilota, daemon przeżywa uśpienie | 4–6 dni |
| M6 Rozszerzenia | Joby, batch, załączniki, rozmowy, pozostałe zadania, metryki | Testy M6 zielone, batch 50 elementów działa | 3–4 dni |
| M7 Wykończenie | Playground, doctor, ewaluacja, autostart, przykłady, README | Wszystkie przykłady integracji działają | 3–4 dni |
| M8 Przepływy | Przepływy YAML z krokami ai, excel, file, outlook, browser; harmonogram | Trzy przykładowe przepływy działają, jeden z harmonogramu | 8–12 dni |

Etap M4 jest przed M5, bo adapter OpenAI ma być gotowy wcześnie. Ryzyko przeglądarki zdejmuje już M1, więc M2–M4 można budować na backendzie `mock` bez obaw.

## Przygotowanie środowiska (raz)

1. Zainstaluj Pythona 3.12 z python.org w wariancie „tylko dla bieżącego użytkownika” (nie wymaga administratora). Zaznacz „Add python.exe to PATH”.
2. Załóż katalog projektu **poza folderem synchronizowanym przez OneDrive**, np. `C:\dev\copilot-bridge`. OneDrive potrafi blokować pliki `.venv` i bazy SQLite.
3. W PowerShellu:
   ```powershell
   cd C:\dev\copilot-bridge
   py -3.12 -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```
   Jeśli PowerShell blokuje skrypty: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.
4. Po kroku M1.1 (`pyproject.toml`): `pip install -e ".[dev]"`. Za firmowym proxy ustaw wcześniej `$env:HTTPS_PROXY = "http://adres-proxy:port"`.
5. Nie uruchamiaj `playwright install`. Używamy zainstalowanego Edge (`channel="msedge"`), więc Playwright nie musi pobierać przeglądarek.
6. Jeśli masz gita, zrób `git init` i commituj po każdym działającym kroku. Jeśli nie, rób kopię katalogu po każdym etapie.

## Pętla pracy na każdy krok

1. Otwórz **nową rozmowę** w Copilocie. Stara rozmowa miesza kontekst.
2. Złóż prompt z **szablonu A** (albo **D** dla testów): karta projektu + bloki interfejsów wymienione w kroku („Wklej”) + specyfikacja kroku.
3. Zapisz odpowiedź jako plik we wskazanej ścieżce.
4. Sprawdź składnię: `python -m py_compile ścieżka\do\pliku.py`.
5. Porównaj publiczne nazwy i sygnatury z blokiem interfejsu. Jeśli Copilot coś zmienił, poproś o poprawkę (szablon B).
6. Przeczytaj „Założenia” pod kodem. Jeśli któreś jest sprzeczne z planem, poproś o poprawkę.
7. Uruchom sprawdzenie z kroku (zwykle test pytest).
8. Błąd → **szablon B** z pełnym komunikatem. Ucięta odpowiedź → **szablon C**.
9. Commit albo kopia.

Gdy test nie przechodzi, najpierw ustal, co odbiega od specyfikacji: test czy kod. Poprawiaj tę stronę, która odbiega od planu.

**Rozmiar plików:** cel to najwyżej 350 linii na plik, twardy limit 500. Plan jest pocięty tak, żeby żaden plik nie musiał być dłuższy. Krótsze pliki to mniej uciętych odpowiedzi i mniej pomyłek Copilota.

## Karta projektu

Wklejaj na początku **każdej** rozmowy:

```text
KARTA PROJEKTU copilot-bridge

Cel: lokalny daemon w Pythonie 3.12 na Windows 11 (firmowy laptop), który wystawia API HTTP na 127.0.0.1:8765. Pod spodem steruje przeglądarką Microsoft Edge przez Playwright (sync API, jeden wątek roboczy) z zalogowanym Microsoft 365 Copilot. Klient (CLI, skrypt Python, biblioteka `openai`, Excel, Power Automate Desktop) wysyła zadanie, najczęściej decyzję z listą dopuszczalnych opcji. Daemon buduje prompt wymuszający odpowiedź JSON, wysyła go do Copilota, wyciąga, naprawia i waliduje JSON i zwraca ustandaryzowaną kopertę JSON.

Pakiet: src/copilot_bridge/ (import: `from copilot_bridge.x import y`). Testy: tests/ (pytest).
Dozwolone zależności: fastapi, uvicorn, pydantic (v2), pyyaml, jsonschema, httpx, typer, playwright, filelock, python-multipart. W testach dodatkowo pytest i openai. Poza tym tylko biblioteka standardowa.

Zasady kodu:
1. `from __future__ import annotations` na początku, pełne adnotacje typów.
2. Docstring modułu (2–4 zdania) i krótkie docstringi funkcji publicznych.
3. Błędy zgłaszaj wyłącznie jako `BridgeError(ErrorCode.X, "komunikat po polsku")` z `copilot_bridge.errors`.
4. Logowanie: `logger = logging.getLogger(__name__)`. Treści pytań i odpowiedzi loguj tylko przez `mask()` z `copilot_bridge.logging_setup`.
5. Ścieżki tylko z `copilot_bridge.paths` (pathlib). Pliki tekstowe zawsze z `encoding="utf-8"`.
6. Żadnego `print()` poza cli.py i cli_admin.py. Żadnego globalnego stanu poza stałymi.
7. Kod działa na Windowsie: bez fork, bez sygnałów POSIX, nigdy `os.kill(pid, 0)` (na Windowsie zabija proces).
8. Nie zmieniaj nazw ani sygnatur z wklejonych bloków interfejsów. Możesz dodawać prywatne funkcje z prefiksem `_`.
```

## Szablon A: nowy plik

````text
<KARTA PROJEKTU>

INTERFEJSY, z których korzystasz albo które implementujesz:
<bloki I-… wymienione w kroku jako „Wklej”>

ZADANIE: napisz kompletny plik `<ścieżka z kroku>`.
<specyfikacja z kroku>

FORMA ODPOWIEDZI:
- Jeden blok kodu z całym plikiem. Bez skrótów typu "...", "# reszta bez zmian" i bez pomijania funkcji.
- Najwyżej <limit z kroku> linii.
- Pod blokiem kodu sekcja "Założenia:" z listą rzeczy, których nie było w specyfikacji, a musiałeś przyjąć.
````

## Szablon B: naprawa pliku

````text
<KARTA PROJEKTU>

INTERFEJSY pliku:
<te same bloki I-… co przy pisaniu pliku>

Poniżej plik `<ścieżka>` i błąd, który powoduje. Popraw plik zgodnie z interfejsami. Zwróć CAŁY poprawiony plik w jednym bloku kodu. Pod kodem jednym zdaniem napisz, co było przyczyną.

PLIK:
```python
<cała zawartość pliku>
```

BŁĄD (komenda i pełny wynik):
```text
<komenda i cały traceback albo wynik pytest>
```
````

Jeśli błąd dotyczy testu, dołącz też plik testu i napisz, który z nich ma być poprawiony.

## Szablon C: kontynuacja uciętej odpowiedzi

````text
Twoja odpowiedź została ucięta. Kontynuuj DOKŁADNIE od tej linii (nie powtarzaj jej ani niczego wcześniej):
<ostatnia kompletna linia z poprzedniej odpowiedzi>
Zwróć tylko dalszy ciąg w bloku kodu.
````

Po sklejeniu zawsze uruchom `python -m py_compile`.

## Szablon D: plik testów

````text
<KARTA PROJEKTU>

INTERFEJSY testowanego modułu:
<bloki I-… wymienione w kroku>

ZADANIE: napisz plik testów pytest `<ścieżka>` dla modułu `<moduł>`.
Testuj wyłącznie przez publiczne interfejsy z bloków powyżej.
Przypadki do pokrycia:
<lista z kroku>
Zasady: bez sieci i bez prawdziwej przeglądarki; ścieżki przez `monkeypatch.setenv("COPILOT_BRIDGE_HOME", str(tmp_path))`; każdy test niezależny.

FORMA ODPOWIEDZI: jeden blok kodu, najwyżej <limit> linii, pod kodem "Założenia:".
````

## Szablon E: zmiana istniejącego pliku

Używaj tylko wtedy, gdy krok wprost każe coś dopisać do istniejącego pliku.

````text
<KARTA PROJEKTU>
<bloki I-… pliku>

Poniżej obecny plik `<ścieżka>`. Wprowadź tylko tę zmianę: <opis z kroku>. Nie zmieniaj niczego innego. Zwróć CAŁY plik w jednym bloku kodu.

```python
<cała zawartość pliku>
```
````
