# Prompt planistyczny: Copilot Bridge (uniwersalne lokalne API AI → M365 Copilot → JSON)

Jesteś doświadczonym architektem oprogramowania. Przygotuj szczegółowy plan implementacji narzędzia **copilot-bridge**. Nie pisz jeszcze kodu produkcyjnego. Twoim zadaniem jest plan, który da się potem wykonać krok po kroku. Odpowiedz po polsku.

## Cel

**Uniwersalne, lokalne API AI do podejmowania decyzji**, działające na firmowym laptopie. Pod spodem działa Microsoft 365 Copilot w przeglądarce Edge sterowanej przez Playwright, ale dla klientów wygląda to jak zwykłe API zwracające ustrukturyzowany JSON. Główne zastosowanie: skrypt, arkusz albo przepływ automatyzacji przekazuje dane i dopuszczalne opcje, a dostaje z powrotem **decyzję i wiadomość od AI** (uzasadnienie), które da się od razu przetworzyć maszynowo. Poza decyzjami API obsługuje też inne zadania: odpowiadanie na pytania, streszczanie, klasyfikację, wyciąganie danych z tekstu i dokumentów, generowanie treści.

Całość:
1. przyjmuje zadanie od klienta (API HTTP albo CLI): pytanie wprost albo nazwane zadanie z danymi wejściowymi (tekst i opcjonalnie pliki),
2. opakowuje je w prompt wymuszający odpowiedź w ustalonym schemacie JSON,
3. przekazuje je do stałego daemona wystawiającego lokalne API HTTP (CLI uruchamia daemona, jeśli nie działa), który przez Playwright steruje Edge z zalogowaną sesją Microsoft 365 Copilot, wysyła prompt i czeka na pełną odpowiedź,
4. wyciąga odpowiedź ze strony, parsuje ją, waliduje względem schematu i w razie potrzeby naprawia,
5. zwraca poprawny JSON, który da się przetwarzać maszynowo: przez API HTTP dla narzędzi i przez stdout CLI dla skryptów.

Przykład użycia (PowerShell):
```powershell
copilot-bridge task decide --input-file .\wniosek.json
copilot-bridge ask "Czy ta umowa wymaga akceptacji prawnika?" --file .\umowa.pdf
copilot-bridge task extract --file .\faktura.pdf --schema .\schemas\invoice.json

$token = Get-Content "$env:LOCALAPPDATA\copilot-bridge\token"
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8765/v1/tasks/decide `
  -Headers @{ Authorization = "Bearer $token" } -ContentType "application/json" `
  -Body '{"input": {"context": "...", "question": "Zatwierdzić zwrot?", "options": ["approve", "reject", "escalate"]}}'
```

Przykład użycia (Python, biblioteka `openai` wskazująca na lokalny daemon):
```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8765/v1", api_key=token)
resp = client.chat.completions.create(
    model="copilot-work",
    messages=[{"role": "user", "content": "Czy zatwierdzić ten wniosek? ..."}],
    response_format={"type": "json_schema", "json_schema": {...}},
)
```

## Decyzje już podjęte

- **Samodzielne, działające narzędzie.** Budujemy kompletne rozwiązanie do codziennego użytku, nie prototyp. Droga przez Playwright i interfejs M365 Copilota jest przesądzona, nie analizuj alternatyw. Jeden użytkownik, jego laptop i jego konto M365.
- **Język: Python.** Daemon, CLI i testy w Pythonie (Playwright dla Pythona).
- **Główne zastosowanie: decyzje.** Odpowiedź ma przede wszystkim nieść decyzję (jedną z dopuszczalnych opcji) i wiadomość od AI z uzasadnieniem. Liczy się niezawodność i przewidywalność formatu, nie wygląd odpowiedzi.
- **Bez strumieniowania.** Klient zawsze dostaje całą odpowiedź naraz. Strumieniowanie nie jest potrzebne ani w API, ani w adapterze OpenAI.
- **Klienci:** własne skrypty w Pythonie (bezpośrednio albo przez bibliotekę `openai`), Excel (Power Query, VBA) i Power Automate Desktop.
- **Adapter zgodny z OpenAI z function calling udawanym promptem**, gotowy wcześnie, zaraz po podstawowym daemonie (szczegóły w sekcji 3).
- **Platforma: firmowy laptop z Windowsem, przeglądarka Edge.** Laptop łączy się przez VPN. Plan projektuj pod Windows (procesy w tle, blokady, uprawnienia do plików, autostart, ścieżki w `%LOCALAPPDATA%`), bez rozwiązań linuksowych.
- **Działanie po uśpieniu jest wymaganiem.** Daemon ma przetrwać uśpienie i hibernację laptopa i po wybudzeniu sam wrócić do pracy, bez ręcznej interwencji (o ile sesja M365 nie wygasła).
- **Uniwersalność ponad jeden przypadek użycia.** Nowy przypadek użycia dodaje się konfiguracją (szablon zadania + schemat), bez zmiany kodu.
- **Docelowy serwis: Microsoft 365 Copilot** (konto firmowe, logowanie przez Entra ID, czat pod adresem w rodzaju `m365.cloud.microsoft/chat`). Nie masz dostępu do tej strony, więc aktualny URL, selektory i zachowanie interfejsu plan ma przewidzieć do sprawdzenia w M1.
- **Tryb pracy: stały daemon.** Jeden długo działający proces trzyma otwartą przeglądarkę z zalogowaną sesją. CLI jest cienkim klientem. Jeśli daemon nie działa, CLI **sam go uruchamia** w tle i dopiero potem wysyła zapytanie.
- **Daemon to wierne API.** Komunikacja przez HTTP na `127.0.0.1`, kontrakt zdefiniowany najpierw (contract-first, OpenAPI), wersjonowany (`/v1`). Narzędzia wołają API bezpośrednio, CLI to tylko jeden z klientów. Z punktu widzenia klienta nie może być widać, że pod spodem jest przeglądarka.

## Pytania, które trzeba rozstrzygnąć na początku

Zanim zaproponujesz architekturę, wypisz te pytania i podaj rekomendowaną odpowiedź na każde:
- **Biblioteki i dystrybucja w Pythonie:** serwer HTTP (np. FastAPI + uvicorn), walidacja (pydantic, jsonschema), CLI (np. Typer), magazyn stanu (SQLite). Jak dostarczyć narzędzie na Windowsa: `uv`/`pipx` czy pojedynczy plik wykonywalny (np. PyInstaller), najlepiej bez uprawnień administratora. Jak uruchamiać daemona w tle bez okna konsoli (`pythonw`, flagi procesu).
- **Historia rozmowy w adapterze OpenAI:** format OpenAI jest bezstanowy (klient za każdym razem wysyła całą historię), a Copilot pamięta wątek. Czy każde wywołanie to nowa rozmowa w Copilocie z całą historią spłaszczoną do jednej wiadomości, czy daemon dopasowuje historię do istniejącego wątku (np. po skrócie prefiksu wiadomości)? Porównaj niezawodność, czas i izolację.
- **Excel:** jaki sposób wywołania rekomendujesz (Power Query, VBA, funkcja `WEBSERVICE`, która obsługuje tylko GET i blokuje arkusz na czas odpowiedzi)? Czy potrzebny jest dodatkowy, uproszczony endpoint pod Excela?

## Obszary, które plan musi pokryć

### 1. Architektura i przepływ danych
Diagram albo opis warstw: Klienci (CLI, skrypty Python, biblioteka `openai`, Excel, Power Automate Desktop) → HTTP API (`/v1`, w tym adapter OpenAI) → Daemon [Auth → Kolejka/Joby → TaskRegistry → Cache → PromptBuilder → **Backend** → JsonParser/Validator] → odpowiedź.

Warstwa `Backend` to interfejs z wymiennymi implementacjami:
- `playwright`: BrowserDriver + ResponseExtractor (właściwe działanie),
- `mock`: deterministyczne odpowiedzi z plików, do pracy nad klientami bez przeglądarki i do testów.

Lokalny magazyn stanu (np. SQLite): historia zapytań i dziennik decyzji, joby, mapowanie `conversation_id` na wątki w Copilocie, cache i nagrania odpowiedzi.

Określ odpowiedzialność każdej warstwy i interfejsy między nimi.

### 2. Kontrakt lokalnego API (kluczowe)
Zaprojektuj API tak, jakby było prawdziwym, publicznym serwisem:
- **Endpointy** (propozycja do dopracowania):
  - `GET /v1/health`: stan daemona i sesji (`starting`, `ready`, `auth_required`, `busy`, `resuming`, `degraded`), wersja, backend,
  - `POST /v1/ask`: dowolne pytanie z opcjonalnym schematem, synchronicznie z timeoutem (skrót do zadania `ask`),
  - `GET /v1/tasks`, `POST /v1/tasks/{name}`: lista i wywołanie nazwanych zadań (sekcja 4),
  - `POST /v1/tasks/{name}/batch`: wiele elementów w jednym wywołaniu (sekcja 13),
  - `POST /v1/jobs` + `GET /v1/jobs/{id}` + `DELETE /v1/jobs/{id}`: tryb asynchroniczny dla klientów, które nie mogą długo czekać na odpowiedź,
  - `POST /v1/conversations` i `POST /v1/conversations/{id}/messages`: rozmowy wieloturowe mapowane na wątek w Copilocie,
  - `GET /v1/schemas`, `GET /v1/schemas/{name}`: dostępne schematy odpowiedzi,
  - `POST /v1/chat/completions` i `GET /v1/models`: adapter zgodny z OpenAI (sekcja 3),
  - `GET /v1/metrics`: statystyki działania (sekcja 17).
- **Specyfikacja OpenAPI** jako artefakt w repozytorium, z której da się wygenerować klienta i dokumentację.
- **Uwierzytelnienie:** token Bearer generowany przy pierwszym starcie, w pliku dostępnym tylko dla użytkownika. Działa też jako `api_key` w bibliotece `openai`.
- **Zachowanie jak prawdziwe API:** kody HTTP zmapowane na kody błędów (`401`, `404` dla nieznanego zadania, `409`, `413` przy za dużym wejściu, `422`, `429` z `Retry-After`, `503` przy `auth_required` i wznawianiu po uśpieniu, `504` przy timeoucie), nagłówek `X-Request-Id`, `Idempotency-Key` dla bezpiecznych ponowień, nagłówki limitów, jednolity format błędu.
- **Parametry zapytania:** `question` albo `input`, `files`, `schema` (nazwa albo inline JSON Schema), `mode` (`work|web`), `timeout_ms`, `conversation_id`, `include_raw`.
- **Załączniki:** pliki (PDF, DOCX, XLSX, obrazy) jako `multipart/form-data` w API i `--file` w CLI. Limity rozmiaru i typów zgodne z tym, co przyjmuje M365 Copilot.
- **Długie czasy odpowiedzi a klienci:** odpowiedź trwa od kilku do kilkudziesięciu sekund. Podaj zalecane ustawienia timeoutów dla Pythona, Power Query, VBA i Power Automate Desktop oraz kiedy użyć jobów z odpytywaniem zamiast wywołania synchronicznego.
- **Stabilność kontraktu:** zasady wersjonowania i co jest zmianą łamiącą.

### 3. Adapter zgodny z OpenAI i function calling (kluczowe)
Cel: skrypty w Pythonie mogą używać oficjalnej biblioteki `openai` (i innych narzędzi mówiących tym formatem), wskazując tylko `base_url` na daemona.
- **Endpointy:** `POST /v1/chat/completions` oraz `GET /v1/models`. Nazwa modelu wybiera tryb, np. `copilot-work` i `copilot-web`.
- **Wiadomości:** jak role `system`/`user`/`assistant`/`tool` są spłaszczane do jednej wiadomości w Copilocie i jak jest obsługiwana historia (pytanie z początku planu).
- **`response_format`:** `json_object` i `json_schema` przechodzą na kopertę promptu i walidację (sekcje 11–12). Bez `response_format` odpowiedź jest zwykłym tekstem.
- **Function calling udawany promptem:**
  - definicje narzędzi (`tools`: nazwa, opis, JSON Schema parametrów) trafiają do promptu,
  - model odpowiada w ustalonym formacie JSON: albo wywołanie narzędzia (nazwa + argumenty), albo odpowiedź końcowa,
  - daemon sprawdza, czy narzędzie istnieje i czy argumenty są zgodne ze schematem; w razie błędu stosuje naprawę (sekcja 11),
  - wynik zamienia na `tool_calls` z wygenerowanymi identyfikatorami i `finish_reason: "tool_calls"`,
  - wyniki narzędzi (`role: "tool"`) z kolejnego wywołania trafiają do promptu jako kontekst,
  - obsługa `tool_choice` (`auto`, `none`, `required`, konkretna funkcja) i `parallel_tool_calls`,
  - pętlę wywołań prowadzi klient, jak w oryginalnym API; daemon pilnuje limitu długości historii.
- **Parametry bez odpowiednika:** `temperature`, `top_p`, `n`, `max_tokens`, `seed` są ignorowane z ostrzeżeniem w nagłówku albo logu. `stream: true` zwraca czytelny błąd `400`, bo strumieniowanie nie jest obsługiwane.
- **Pole `usage`:** szacunek liczby tokenów, oznaczony jako szacunek.
- **Błędy** w formacie OpenAI (`{"error": {"message", "type", "code"}}`), żeby biblioteka `openai` rzucała właściwe wyjątki.
- **Test zgodności:** zestaw testów, w którym klientem jest oficjalna biblioteka `openai`: zwykłe pytanie, `json_schema`, pełna pętla z narzędziami.

### 4. Rejestr zadań i decyzje
- **Zadanie** to plik konfiguracyjny (YAML/JSON): nazwa, opis, szablon promptu z miejscami na dane, schemat wejścia, schemat wyjścia, przykłady (few-shot), domyślny `mode` i timeout.
- Dodanie nowego przypadku użycia = dodanie pliku, bez zmiany kodu i bez restartu (albo z przeładowaniem na żądanie).
- **Zadanie `decide` (najważniejsze):**
  - wejście: kontekst, pytanie decyzyjne, lista dopuszczalnych opcji (z opcjonalnym opisem każdej), opcjonalne kryteria i reguły,
  - wyjście: `decision` (wyłącznie jedna z podanych opcji), `message` (wiadomość AI z uzasadnieniem), `reasons`, `confidence`, `needs_review`,
  - próg pewności w zapytaniu: poniżej progu `needs_review: true` albo decyzja zastępcza wskazana przez klienta,
  - opcjonalne głosowanie (`votes: N`): N niezależnych zapytań i decyzja większościowa ze zgodnością w `meta`, bo model nie jest deterministyczny; kosztuje N razy więcej czasu,
  - dziennik decyzji: wejście, decyzja, wiadomość, wersje promptu i zadania, żeby każdą decyzję dało się później odtworzyć i sprawdzić.
- **Pozostałe zadania wbudowane** (propozycja): `ask` (pytanie ogólne), `summarize`, `classify` (klasyfikacja do podanych etykiet), `extract` (wyciąganie pól według schematu, także z załączonych plików), `rewrite`, `draft`, `work-search` (pytanie o dane firmowe w trybie Work).
- **Schematy dynamiczne:** schemat wyjścia może zależeć od wejścia, np. `enum` budowany z opcji przekazanych do `decide` albo etykiet do `classify`, żeby walidator odrzucał wartości spoza listy.
- Walidacja wejścia przed wysłaniem do Copilota, limit rozmiaru danych wejściowych, dzielenie długich tekstów, jeśli potrzebne.
- Wersjonowanie zadań i koperty promptu (wersje zapisywane w `meta`), żeby zmiana szablonu nie psuła klientów po cichu, a wyniki dało się porównać.

### 5. Cykl życia daemona (kluczowe)
- **Auto-start z CLI:** CLI próbuje się połączyć. Jeśli nie ma daemona, uruchamia go jako odłączony proces w tle, bez okna konsoli, czeka na gotowość (`/v1/health` z timeoutem) i dopiero wtedy wysyła zapytanie.
- **Wyścigi przy starcie:** dwa równoległe wywołania CLI nie mogą uruchomić dwóch daemonów. Zaprojektuj blokadę działającą na Windowsie (np. nazwany mutex albo plik otwarty na wyłączność) i plik PID.
- **Wykrywanie martwego daemona:** nieaktualny plik PID, zajęty port po awarii, proces wiszący bez odpowiedzi. Jak CLI to rozpoznaje i sprząta.
- **Gotowość vs. żywotność:** daemon działa, ale sesja M365 wygasła albo trwa wznawianie po uśpieniu. Health check ma zwracać stan (`starting`, `ready`, `auth_required`, `busy`, `resuming`, `degraded`).
- **Komendy zarządzania:** `copilot-bridge daemon start|stop|restart|status|logs`, plus `copilot-bridge login` do ręcznego logowania w widocznym oknie.
- **Zgodność wersji:** co się dzieje, gdy CLI zostało zaktualizowane, a stary daemon wciąż działa (sprawdzenie wersji API przez `/v1/health`, automatyczny restart).
- **Zasoby:** limit pamięci przeglądarki, okresowe odświeżanie karty lub kontekstu.
- **Start razem z systemem:** Harmonogram zadań albo folder Autostart (bez uprawnień administratora), jako uzupełnienie auto-startu z CLI.
- Logi daemona do pliku z rotacją w `%LOCALAPPDATA%`.

### 6. Sesja i logowanie (Entra ID)
- Pierwsze logowanie w widocznym oknie (użytkownik loguje się ręcznie, łącznie z MFA), potem trwały profil (`launch_persistent_context`). Daemon po zalogowaniu może pracować headless albo w zminimalizowanym oknie. Oceń, czy M365 działa poprawnie w trybie headless.
- Specyfika firmowa: SSO, Conditional Access, zgodność urządzenia, wygasanie sesji po określonym czasie. Jak daemon wykrywa przekierowanie na stronę logowania i przechodzi w stan `auth_required` zamiast się wieszać.
- Ścieżka odzyskania: API zwraca `503` z kodem `AUTH_REQUIRED`, a CLI dodaje instrukcję uruchomienia `copilot-bridge login`, które otwiera widoczne okno w tym samym profilu.
- Gdzie i jak bezpiecznie przechowywać profil i ciasteczka (katalog dostępny tylko dla użytkownika, poza repozytorium).

### 7. Środowisko uruchomieniowe (Windows, Edge, uśpienie)
- **Przeglądarka:** zainstalowany Edge (`channel="msedge"`), bez pobierania osobnego Chromium. Zawsze osobny katalog profilu, nie codzienny profil użytkownika (blokada profilu, mieszanie danych). Sprawdź, czy tak uruchomiony Edge przechodzi SSO i Conditional Access, oraz czy funkcje oszczędzania Edge (usypianie kart, tryb wydajności) nie usypiają karty daemona.
- **Uśpienie i wybudzenie (wymaganie):**
  - wykrycie wybudzenia (np. zdarzenia zasilania Windows albo skok czasu między kolejnymi sygnałami życia),
  - odczekanie na sieć, w tym ponowne zestawienie VPN,
  - sprawdzenie, czy karta i połączenie strony z serwerem żyją; w razie potrzeby przeładowanie karty albo restart przeglądarki,
  - stan `resuming` w `/v1/health` na czas wznawiania,
  - zapytania przerwane uśpieniem: automatyczne ponowienie albo błąd `INTERRUPTED`, który klient może bezpiecznie ponowić,
  - zapytania przychodzące w trakcie wznawiania czekają w kolejce zamiast od razu zwracać błąd.
- **Blokada ekranu i zminimalizowane okno:** sprawdź, czy nie spowalniają strumieniowania odpowiedzi w interfejsie Copilota ani wykrywania jej końca.
- **Instalacja:** wszystko w profilu użytkownika, najlepiej bez uprawnień administratora. Uwzględnij firmowe proxy przy instalacji zależności.
- **Specyfika Windows:** uprawnienia do plików przez ACL, proces w tle bez okna konsoli, ścieżki konfiguracji i logów w `%LOCALAPPDATA%`.

### 8. Skutki uboczne na koncie M365
- **Historia czatów:** każde zapytanie może tworzyć rozmowę widoczną w historii Copilota na wszystkich urządzeniach (przeglądarka, Teams, Outlook). Porównaj strategie: usuwanie rozmów po zakończeniu, jedna rozmowa robocza z rotacją albo pozostawienie historii.
- **Pamięć i personalizacja:** jeśli w tenancie działa pamięć Copilota albo personalizacja, automatyczne zapytania mogą ją zaśmiecać, a zapamiętane preferencje mogą zmieniać wyniki zadań i decyzji. Jak to sprawdzić i ograniczyć.
- **Izolacja zapytań:** kontekst jednego zadania nie może przeciekać do następnego (nowa rozmowa dla każdego zadania kontra dodatkowy czas). Szczególnie ważne przy decyzjach.
- **Niedeterminizm:** brak kontroli nad temperaturą i wersją modelu, więc to samo wejście może dać różne decyzje. Łagodzą to ścisłe schematy, głosowanie w `decide` i próg pewności. Jeśli interfejs pozwala wybrać tryb lub model (np. szybka odpowiedź kontra dłuższe rozumowanie), udostępnij to jako parametr.
- **Limity usługi:** ewentualne limity liczby tur w rozmowie, dzienne limity i dławienie. Jak wpływają na strategię rozmów, pętlę naprawczą i głosowanie.

### 9. Sterowanie stroną
- Strategia selektorów: role/aria/teksty zamiast kruchych klas CSS, wszystkie selektory w jednym pliku konfiguracyjnym, żeby łatwo je aktualizować.
- Nowa rozmowa dla każdego zapytania albo reużycie wątku (wpływ na kontekst i izolację; powiązanie z `conversation_id` w API i historią w adapterze OpenAI).
- Wklejanie długich promptów (fill albo schowek zamiast wpisywania znak po znaku) i limit długości wiadomości w M365 Copilot.
- **Załączniki:** wgrywanie plików do czatu (np. `set_input_files`) i czekanie, aż Copilot je przetworzy. Opcjonalnie odwołania do plików z OneDrive lub SharePointa.
- Elementy specyficzne dla M365: przełącznik Work/Web (grounding na danych firmowych vs. internet), wybór agenta, ewentualne dialogi zgód i komunikaty tenanta. Mają być konfigurowalne parametrem `mode` w API, nazwą modelu w adapterze OpenAI i flagą `--mode work|web` w CLI.
- **Wykrywanie końca odpowiedzi w interfejsie Copilota** (Copilot pisze odpowiedź stopniowo): zniknięcie przycisku „Stop”, stabilność tekstu przez N ms, pojawienie się przycisków akcji. Zaproponuj metodę główną i zapasową.
- Obsługa limitów, komunikatów o błędach i zmian w UI: jak je wykryć i jaki kod błędu zwrócić.

### 10. Ekstrakcja odpowiedzi
- Skąd brać tekst: `inner_text` wyrenderowanego markdownu, blok kodu, przycisk „Kopiuj” czy przechwytywanie ruchu sieciowego strony (WebSocket/SSE). Oceń ryzyko zniekształcenia JSON-a przez renderowanie markdownu przy każdej opcji.
- Jak jednoznacznie zidentyfikować właściwą odpowiedź, np. przez unikalny `request_id` (nonce) w prompcie, który model musi odesłać.
- **Cytowania w trybie Work:** Copilot dodaje do tekstu znaczniki przypisów i listę źródeł. Mogą trafić do środka JSON-a albo go zepsuć. Zaplanuj ich usuwanie przed parsowaniem, a prawdziwe źródła (linki do plików i maili) wyciągaj z interfejsu, zamiast ufać adresom wygenerowanym przez model.
- **Ucięte odpowiedzi:** jak wykryć, że odpowiedź została przerwana (limit długości, zerwane połączenie), i co wtedy zrobić: poprosić o kontynuację, podzielić zadanie albo zwrócić błąd.

### 11. Prompt wymuszający schemat (kluczowe)
Zaprojektuj szablon „koperty” promptu, wspólny dla wszystkich zadań i adaptera OpenAI, który:
- jasno instruuje model, żeby zwrócił **tylko** jeden blok ```json bez tekstu przed nim i po nim,
- zawiera schemat JSON (JSON Schema) albo przykład wypełnionej odpowiedzi,
- zawiera `request_id` do odesłania,
- oddziela instrukcję zadania od danych klienta delimiterami, żeby ograniczyć prompt injection z treści danych,
- określa zachowanie przy niepewności lub odmowie (pole `confidence`, `status: "refused" | "uncertain"`, `needs_review`),
- przy decyzjach wymusza wybór wyłącznie spośród podanych opcji,
- określa język wartości tekstowych (klucze JSON zawsze bez zmian),
- ma wariant z definicjami narzędzi dla function calling (sekcja 3).

Zaproponuj **domyślny schemat odpowiedzi modelu** (decyzja + wiadomość), np.:
```json
{
  "request_id": "string",
  "status": "ok | uncertain | refused",
  "decision": "jedna z opcji podanych w zapytaniu albo null, gdy zapytanie nie dotyczy decyzji",
  "message": "wiadomość AI z odpowiedzią lub uzasadnieniem",
  "reasons": ["string"],
  "confidence": 0.0,
  "needs_review": false,
  "sources": [{"title": "string", "url": "string"}]
}
```
Umożliw też podanie własnego schematu (parametr `schema` w API, `--schema` w CLI, `response_format` w adapterze OpenAI) oraz schematów per zadanie z rejestru.

Opisz **naprawę odpowiedzi** w dwóch krokach. Najpierw lokalnie, bez modelu: usunięcie znaczników cytowań, przecinków końcowych i tekstu wokół bloku JSON. Dopiero gdy to nie wystarczy, prompt „repair” w tej samej rozmowie z listą błędów walidacji. Ogranicz liczbę prób (np. 2) i zapisuj w logach, co się stało.

### 12. Koperta odpowiedzi (API i CLI)
Zdefiniuj stałą „kopertę” odpowiedzi niezależną od schematu modelu. Ta sama struktura wraca z API i jest wypisywana przez CLI (adapter OpenAI używa własnego formatu odpowiedzi, sekcja 3):
```json
{
  "ok": true,
  "request_id": "uuid",
  "task": "decide",
  "data": { },
  "raw": "surowy tekst odpowiedzi (opcjonalnie, include_raw / --include-raw)",
  "error": null,
  "meta": { "duration_ms": 0, "queue_ms": 0, "attempts": 1, "votes": 1, "agreement": 1.0, "cached": false, "backend": "m365-playwright", "api_version": "v1", "schema": "decide@1", "prompt_version": "envelope@1" }
}
```
- Kody błędów (`AUTH_REQUIRED`, `TIMEOUT`, `INTERRUPTED`, `UI_CHANGED`, `RATE_LIMITED`, `INVALID_INPUT`, `INPUT_TOO_LARGE`, `ATTACHMENT_REJECTED`, `UNKNOWN_TASK`, `INVALID_JSON`, `SCHEMA_MISMATCH`, `INVALID_TOOL_CALL`, `TRUNCATED`, `DAEMON_UNAVAILABLE`, `DAEMON_START_FAILED`, `VERSION_MISMATCH`) oraz ich mapowanie na statusy HTTP, błędy w formacie OpenAI i kody wyjścia procesu CLI.
- Zasada dla CLI: stdout zawiera tylko JSON, a logi idą na stderr lub do pliku (`--verbose`, `--debug` ze zrzutami ekranu i trace'ami Playwrighta).

### 13. Wydajność, batch i cache
- **Realne oczekiwania:** jedna sesja przeglądarki obsłuży kilka zapytań na minutę, a nie setki na sekundę. Plan ma to jasno powiedzieć, a M1 ma to zmierzyć. Głosowanie w `decide` mnoży czas.
- **Batch:** wiele elementów (np. 50 wniosków do decyzji) pakowanych po kilka lub kilkanaście w jeden prompt. Wynik to tablica z identyfikatorem każdego elementu, z obsługą częściowych błędów i ponawianiem tylko nieudanych elementów. Dobierz wielkość paczki z uwagi na jakość decyzji i limit długości wiadomości.
- **Cache:** identyczne zapytanie (zadanie, wejście, schemat, tryb, wersja promptu) zwraca zapisany wynik z określonym czasem ważności. Da się go wyłączyć parametrem lub flagą, a odpowiedź ma `meta.cached`. Przyspiesza powtarzalne zapytania i oszczędza limity.
- **Priorytety w kolejce:** zapytania interaktywne przed długimi batchami.

### 14. Współbieżność i niezawodność
- Kolejka zapytań w daemonie (jedna karta albo pula kart), limit długości kolejki, timeouty globalne i per etap, anulowanie zapytania, gdy klient się rozłączy (Ctrl+C) albo wywoła `DELETE /v1/jobs/{id}`.
- Czas oczekiwania w kolejce wliczony do timeoutu klienta i raportowany w `meta`.
- Retry z backoffem tylko dla błędów przejściowych.
- Odtwarzanie po awarii przeglądarki lub karty wewnątrz działającego daemona, bez utraty sesji.
- **Trwałość stanu:** mapowanie `conversation_id` na wątki w Copilocie, joby asynchroniczne i dziennik decyzji zapisywane lokalnie, żeby restart daemona albo uśpienie laptopa ich nie gubiły.

### 15. Testowanie
- Testy jednostkowe (pytest) PromptBuildera, rejestru zadań, parsera JSON (ekstrakcja z fence'ów, lokalna naprawa, usuwanie znaczników cytowań), walidatora i parsera wywołań narzędzi.
- Testy kontraktowe API względem specyfikacji OpenAPI, uruchamiane na backendzie `mock`.
- Testy zgodności adaptera OpenAI z oficjalną biblioteką `openai` jako klientem (sekcja 3).
- Testy integracyjne cyklu życia daemona na Windowsie: auto-start, dwa równoległe starty, martwy plik PID, niezgodność wersji, symulowane uśpienie i wybudzenie.
- Testy BrowserDrivera na lokalnej atrapie strony (fixture HTML symulujący stopniowe pisanie odpowiedzi), żeby nie odpytywać prawdziwego Copilota w testach automatycznych.
- **Nagrywanie i odtwarzanie:** tryb, w którym backend `playwright` zapisuje pary zapytanie → odpowiedź, a backend `mock` je odtwarza. Z nagrań powstają testy i zestaw ewaluacyjny.
- **Zestaw ewaluacyjny** uruchamiany ręcznie na prawdziwym serwisie: odsetek poprawnego JSON-a za pierwszym razem, trafność decyzji na przykładach z oczekiwanym wynikiem, zgodność decyzji przy wielokrotnym uruchomieniu tego samego wejścia, poprawność wywołań narzędzi.
- Wykrywanie regresji po zmianie UI Copilota (np. osobna komenda `copilot-bridge doctor`).

### 16. Bezpieczeństwo
- Limit częstotliwości zapytań, żeby usługa nie zaczęła dławić konta. Logowanie z MFA zawsze ręczne przez `copilot-bridge login`.
- Profil przeglądarki z sesją traktowany jak hasło: katalog dostępny tylko dla użytkownika, poza repozytorium, łatwy do usunięcia (`copilot-bridge logout` czyści profil).
- Co trafia do logów, maskowanie treści pytań i odpowiedzi, uprawnienia do katalogu profilu, logów i pliku z tokenem.
- Daemon słucha wyłącznie na `127.0.0.1` i wymaga tokenu, więc inne procesy i użytkownicy na maszynie nie skorzystają z sesji.
- **Ochrona lokalnego API przed stronami w przeglądarce:** walidacja nagłówka `Host` (ochrona przed DNS rebinding), żadnego luźnego CORS, a playground nie może ujawnić tokenu innym stronom.
- **Odpowiedź to niezaufane dane:** w trybie Work Copilot czyta maile i dokumenty, które mogą zawierać wstrzyknięte instrukcje. Decyzje z niską pewnością albo `needs_review` powinny trafiać do człowieka, a klienci muszą walidować wywołania narzędzi proponowane przez model, zanim je wykonają.
- **Dziennik decyzji, nagrania, cache i historia zawierają dane firmowe:** gdzie leżą, jak długo są trzymane i jak je czyścić.

### 17. Interfejs użytkownika i integracje
- **Prosty playground:** strona HTML serwowana przez daemona (wybór zadania, wejście, załączniki, sformatowany JSON na wyjściu, stan daemona i sesji, podgląd dziennika decyzji).
- **Gotowe przykłady integracji** z zalecanymi timeoutami i obsługą błędów:
  - Python: `requests`/`httpx` oraz biblioteka `openai`, w tym pełna pętla function calling,
  - Excel: Power Query i VBA (plus ewentualny uproszczony endpoint, jeśli plan go rekomenduje),
  - Power Automate Desktop: akcja wywołania usługi sieciowej i zamiana JSON-a na obiekt, rozgałęzienie przepływu po polu `decision`,
  - PowerShell.
- **Przykłady zastosowań:** 3–5 gotowych przykładów z danymi wejściowymi i oczekiwanym wynikiem, głównie decyzyjnych (np. akceptacja wniosku, kierowanie zgłoszenia, ocena zgodności dokumentu z regułami), plus wyciąganie danych z faktury i streszczenie. Służą jako dokumentacja i część zestawu ewaluacyjnego.
- **Metryki:** `/v1/metrics` z liczbą zapytań, odsetkiem poprawnego JSON-a za pierwszym razem i po naprawie, średnim i p95 czasu odpowiedzi, rozkładem decyzji, odsetkiem `needs_review`, podziałem na zadania.

### 18. Struktura repozytorium i konfiguracja
Proponowane drzewo katalogów projektu Pythona (`pyproject.toml`, `src/copilot_bridge/`, `openapi.yaml`, `tasks/` z definicjami zadań, `examples/` z przykładami dla Pythona, Excela i Power Automate Desktop, `fixtures/` z nagraniami, `evals/` z zestawem ewaluacyjnym, `tests/`), plik konfiguracyjny (URL, port, backend, ścieżka profilu Edge, selektory, timeouty, domyślny schemat, cache, progi pewności), zmienne środowiskowe i sposób instalacji na Windowsie.

## Oczekiwany format planu

1. Rozstrzygnięcia otwartych pytań (z rekomendacją i uzasadnieniem).
2. Architektura z diagramem (ASCII lub mermaid).
3. Pełny szablon promptu-koperty, promptu naprawczego i wariantu z narzędziami (gotowe teksty) oraz przykładowa definicja zadania `decide`.
4. Domyślny JSON Schema odpowiedzi, koperta odpowiedzi, szkic specyfikacji OpenAPI lokalnego API i przykład wymiany z function calling w formacie OpenAI.
5. Przykłady zastosowań z danymi wejściowymi i oczekiwanym wynikiem.
6. Lista rzeczy do sprawdzenia na żywej stronie w M1 (czy sesja przeżywa dzień pracy i uśpienie laptopa, czy działa tryb headless, czy Copilot stabilnie oddaje blok JSON, ile trwa odpowiedź, jakie są limity, jak zachowują się cytowania, historia i pamięć) i jak plan się dostosuje, jeśli wynik będzie inny niż zakładany.
7. Kamienie milowe z kryteriami „done” i szacunkiem pracochłonności dla jednej osoby:
   - **M1:** `copilot-bridge login` + skrypt w Pythonie wysyłający jedno pytanie do M365 Copilota przez Edge → surowy tekst odpowiedzi; sprawdzenie rzeczy z punktu 6,
   - **M2:** daemon z lokalnym API (`/v1/health`, `/v1/ask`, token, walidacja `Host`) + backend `mock` + auto-start z CLI + `daemon status/stop`,
   - **M3:** prompt-koperta + parsowanie, walidacja i lokalna naprawa JSON + pętla naprawcza + zadania `decide` i `ask` + **podstawowy adapter OpenAI** (`/v1/chat/completions` bez narzędzi, `/v1/models`, `response_format`) z testami na bibliotece `openai` + nagrywanie odpowiedzi,
   - **M4:** function calling udawany promptem (`tools`, `tool_choice`, `tool_calls`, wiadomości `tool`) z testami pełnej pętli,
   - **M5:** kody błędów i statusy HTTP, stan `auth_required`, wznawianie po uśpieniu, logi, debug (trace/screenshot), kolejka, joby asynchroniczne, anulowanie, trwałość stanu, dziennik decyzji,
   - **M6:** załączniki, batch, cache, głosowanie i progi pewności w `decide`, pozostałe zadania wbudowane, rozmowy wieloturowe, metryki,
   - **M7:** playground, przykłady integracji (Python, Excel, Power Automate Desktop, PowerShell) i zastosowań, komenda `doctor`, start razem z systemem, testy na atrapie, dokumentacja, paczka dla Windowsa.
8. Rozbicie każdego kamienia milowego na małe zadania, które można po kolei przekazać agentowi AI do implementacji: cel, pliki, kryteria akceptacji, sposób sprawdzenia.
9. Lista ryzyk technicznych z prawdopodobieństwem, wpływem i mitygacją.
10. Lista rzeczy świadomie poza zakresem.

Pisz konkretnie. Tam, gdzie decyzja zależy od informacji, której nie masz (np. aktualny DOM strony Copilota), zaznacz to wprost i zaproponuj, jak to zweryfikować.
