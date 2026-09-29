# Prompt planistyczny: Copilot Bridge (CLI → daemon → Playwright → M365 Copilot → JSON)

Jesteś doświadczonym architektem oprogramowania. Przygotuj szczegółowy plan implementacji narzędzia **copilot-bridge**. Nie pisz jeszcze kodu produkcyjnego. Twoim zadaniem jest plan, który da się potem wykonać krok po kroku.

## Cel

Narzędzie CLI, które:
1. przyjmuje pytanie od użytkownika (argument, stdin albo plik),
2. opakowuje je w prompt wymuszający odpowiedź w ustalonym schemacie JSON,
3. przekazuje zapytanie do stałego daemona (uruchamiając go, jeśli nie działa), który przez Playwright steruje przeglądarką z zalogowaną sesją Microsoft 365 Copilot, wysyła prompt i czeka na pełną odpowiedź,
4. wyciąga odpowiedź ze strony, parsuje ją, waliduje względem schematu i w razie potrzeby naprawia,
5. zwraca na stdout **wyłącznie** poprawny JSON, który da się przetwarzać maszynowo (np. `jq` albo inny program).

Przykład użycia:
```
copilot-bridge ask "Jakie są zalety Rusta?" --schema default --timeout 120
echo "pytanie" | copilot-bridge ask --stdin --schema ./schemas/custom.json
```

## Decyzje już podjęte

- **Docelowy serwis: Microsoft 365 Copilot** (konto firmowe, logowanie przez Entra ID, czat pod adresem w rodzaju `m365.cloud.microsoft/chat`). Zweryfikuj aktualny URL i wszystkie selektory na żywej stronie.
- **Tryb pracy: stały daemon.** Jeden długo działający proces trzyma otwartą przeglądarkę z zalogowaną sesją. CLI jest cienkim klientem. Jeśli daemon nie działa, CLI **sam go uruchamia** w tle i dopiero potem wysyła zapytanie.

## Pytania, które trzeba rozstrzygnąć na początku

Zanim zaproponujesz architekturę, wypisz te pytania i podaj rekomendowaną odpowiedź na każde:
- **Czy istnieje oficjalna droga** do M365 Copilota, np. API Microsoft Graph dla Copilota (Chat API / Retrieval API), która zastąpiłaby sterowanie stroną? Sprawdź aktualny status (GA/beta), wymagane licencje i uprawnienia administratora tenanta. Porównaj z Playwrightem pod kątem stabilności, zgodności z polityką firmy i kosztu utrzymania. Jeśli API jest realne, zaprojektuj warstwę `Backend` tak, żeby dało się je później podpiąć zamiast Playwrighta bez zmiany CLI.
- **Stos technologiczny:** TypeScript/Node czy Python? Uzasadnij wybór (Playwright, walidacja schematu, daemon, dystrybucja CLI).
- **Kanał komunikacji CLI ↔ daemon:** Unix socket / named pipe (Windows) czy HTTP na `127.0.0.1`? Uwzględnij, na jakich systemach narzędzie ma działać, oraz uwierzytelnienie klienta (np. token w pliku z uprawnieniami tylko dla użytkownika).
- **Polityka firmy:** czy automatyzacja UI M365 Copilota jest dozwolona przez politykę IT i DLP tenanta? Plan ma to wymienić jako warunek wstępny do potwierdzenia.

## Obszary, które plan musi pokryć

### 1. Architektura i przepływ danych
Diagram albo opis warstw: CLI (klient) → IPC → Daemon [Kolejka → PromptBuilder → BrowserDriver (Playwright) → ResponseExtractor → JsonParser/Validator] → odpowiedź do CLI → Output. Określ odpowiedzialność każdej warstwy i interfejsy między nimi, w tym protokół wiadomości IPC (żądanie, odpowiedź, błąd, wersja protokołu).

### 2. Cykl życia daemona (kluczowe)
- **Auto-start z CLI:** CLI próbuje się połączyć. Jeśli nie ma daemona, uruchamia go jako odłączony proces (detached, bez terminala), czeka na gotowość (health check z timeoutem) i dopiero wtedy wysyła zapytanie.
- **Wyścigi przy starcie:** dwa równoległe wywołania CLI nie mogą uruchomić dwóch daemonów. Zaprojektuj blokadę (lockfile z `flock` / atomowe utworzenie pliku) i plik PID.
- **Wykrywanie martwego daemona:** nieaktualny plik PID albo socket po awarii, proces wiszący bez odpowiedzi. Jak CLI to rozpoznaje i sprząta.
- **Gotowość vs. żywotność:** daemon działa, ale sesja M365 wygasła. Health check ma zwracać stan (`starting`, `ready`, `auth_required`, `busy`, `degraded`).
- **Komendy zarządzania:** `copilot-bridge daemon start|stop|restart|status|logs`, plus `copilot-bridge login` do ręcznego (headed) logowania.
- **Zgodność wersji:** co się dzieje, gdy CLI zostało zaktualizowane, a stary daemon wciąż działa (sprawdzenie wersji protokołu, automatyczny restart).
- **Zasoby:** limit pamięci przeglądarki, okresowe odświeżanie karty lub kontekstu, opcjonalne wyłączanie po długiej bezczynności (i czy w ogóle, skoro ma być stały).
- **Opcjonalnie:** start razem z systemem (systemd user service / launchd / Harmonogram zadań Windows) jako uzupełnienie auto-startu z CLI.
- Logi daemona do pliku z rotacją.

### 3. Sesja i logowanie (Entra ID)
- Pierwsze logowanie w trybie headed (użytkownik loguje się ręcznie, łącznie z MFA), potem trwały profil (`launchPersistentContext`). Daemon po zalogowaniu może pracować headless albo w zminimalizowanym oknie. Oceń, czy M365 działa poprawnie w trybie headless.
- Specyfika firmowa: SSO, Conditional Access, zgodność urządzenia, wygasanie sesji po określonym czasie. Jak daemon wykrywa przekierowanie na stronę logowania i przechodzi w stan `auth_required` zamiast się wieszać.
- Ścieżka odzyskania: CLI zwraca `AUTH_REQUIRED` z instrukcją uruchomienia `copilot-bridge login`, które otwiera widoczne okno w tym samym profilu.
- Gdzie i jak bezpiecznie przechowywać profil i ciasteczka (uprawnienia tylko dla użytkownika, poza repozytorium).

### 4. Sterowanie stroną
- Strategia selektorów: role/aria/teksty zamiast kruchych klas CSS, wszystkie selektory w jednym pliku konfiguracyjnym, żeby łatwo je aktualizować.
- Nowa rozmowa dla każdego zapytania albo reużycie wątku (wpływ na kontekst i izolację).
- Wklejanie długich promptów (fill albo schowek zamiast wpisywania znak po znaku) i limit długości wiadomości w M365 Copilot.
- Elementy specyficzne dla M365: przełącznik Work/Web (grounding na danych firmowych vs. internet), wybór agenta, ewentualne dialogi zgód i komunikaty tenanta. Mają być konfigurowalne flagą (np. `--mode work|web`).
- **Wykrywanie końca odpowiedzi strumieniowanej:** zniknięcie przycisku „Stop”, stabilność tekstu przez N ms, pojawienie się przycisków akcji. Zaproponuj metodę główną i zapasową.
- Obsługa limitów, komunikatów o błędach i zmian w UI: jak je wykryć i jaki kod błędu zwrócić.

### 5. Ekstrakcja odpowiedzi
- Skąd brać tekst: `innerText` wyrenderowanego markdownu, blok kodu, przycisk „Kopiuj” czy przechwytywanie ruchu sieciowego (WebSocket/SSE). Oceń ryzyko zniekształcenia JSON-a przez renderowanie markdownu przy każdej opcji.
- Jak jednoznacznie zidentyfikować właściwą odpowiedź, np. przez unikalny `request_id` (nonce) w prompcie, który model musi odesłać.

### 6. Prompt wymuszający schemat (kluczowe)
Zaprojektuj szablon „koperty” promptu, który:
- jasno instruuje model, żeby zwrócił **tylko** jeden blok ```json bez tekstu przed nim i po nim,
- zawiera schemat JSON (JSON Schema) albo przykład wypełnionej odpowiedzi,
- zawiera `request_id` do odesłania,
- oddziela pytanie użytkownika delimiterami, żeby ograniczyć prompt injection z treści pytania,
- określa zachowanie przy niepewności lub odmowie (pole `confidence`, `status: "refused" | "uncertain"`).

Zaproponuj **domyślny schemat odpowiedzi modelu**, np.:
```json
{
  "request_id": "string",
  "status": "ok | uncertain | refused",
  "answer": "string",
  "key_points": ["string"],
  "confidence": 0.0,
  "sources": [{"title": "string", "url": "string"}]
}
```
Umożliw też podanie własnego schematu przez `--schema`.

Opisz **pętlę naprawczą**: gdy JSON jest niepoprawny albo nie przechodzi walidacji, wyślij w tej samej rozmowie prompt „repair” z listą błędów walidacji. Ogranicz liczbę prób (np. 2) i zapisuj w logach, co się stało.

### 7. Kontrakt wyjścia CLI
Zdefiniuj stałą „kopertę” wyjściową niezależną od schematu modelu:
```json
{
  "ok": true,
  "request_id": "uuid",
  "data": { },
  "raw": "surowy tekst odpowiedzi (opcjonalnie, flaga --include-raw)",
  "error": null,
  "meta": { "duration_ms": 0, "attempts": 1, "backend": "m365-playwright", "daemon_version": "...", "schema": "default" }
}
```
- Kody błędów (`AUTH_REQUIRED`, `TIMEOUT`, `UI_CHANGED`, `RATE_LIMITED`, `INVALID_JSON`, `SCHEMA_MISMATCH`, `DAEMON_UNAVAILABLE`, `DAEMON_START_FAILED`, `PROTOCOL_MISMATCH`) i odpowiadające im kody wyjścia procesu.
- Zasada: stdout zawiera tylko JSON, a logi idą na stderr lub do pliku (`--verbose`, `--debug` ze zrzutami ekranu i trace'ami Playwrighta).

### 8. Współbieżność i niezawodność
- Kolejka zapytań w daemonie (jedna karta albo pula kart), limit długości kolejki, timeouty globalne i per etap, anulowanie zapytania, gdy CLI się rozłączy (Ctrl+C).
- Czas oczekiwania w kolejce wliczony do timeoutu CLI i raportowany w `meta`.
- Retry z backoffem tylko dla błędów przejściowych.
- Odtwarzanie po awarii przeglądarki lub karty wewnątrz działającego daemona, bez utraty sesji.

### 9. Testowanie
- Testy jednostkowe PromptBuildera, parsera JSON (ekstrakcja z fence'ów, naprawa drobnych usterek) i walidatora.
- Testy integracyjne cyklu życia daemona: auto-start, dwa równoległe starty, martwy plik PID, niezgodność wersji.
- Testy BrowserDrivera na lokalnej atrapie strony (fixture HTML symulujący streaming), żeby nie odpytywać prawdziwego Copilota w CI.
- Ręczny smoke test na prawdziwym serwisie z checklistą.
- Wykrywanie regresji po zmianie UI Copilota (np. osobna komenda `copilot-bridge doctor`).

### 10. Bezpieczeństwo i zgodność
- Ryzyka regulaminowe (ToS) i polityki firmy przy automatyzacji UI M365 oraz rekomendacja, jak je ograniczyć (użytek osobisty, limity częstotliwości, brak omijania zabezpieczeń, Conditional Access i MFA).
- Dane firmowe: odpowiedzi w trybie Work mogą zawierać poufne treści z tenanta. Co trafia do logów, maskowanie treści pytań i odpowiedzi, uprawnienia do katalogu profilu, logów i socketu.
- Daemon słucha wyłącznie lokalnie i przyjmuje połączenia tylko od bieżącego użytkownika.

### 11. Struktura repozytorium i konfiguracja
Proponowane drzewo katalogów, plik konfiguracyjny (URL, selektory, timeouty, domyślny schemat), zmienne środowiskowe i sposób instalacji CLI.

## Oczekiwany format planu

1. Rozstrzygnięcia otwartych pytań (z rekomendacją i uzasadnieniem).
2. Architektura z diagramem (ASCII lub mermaid).
3. Pełny szablon promptu-koperty i promptu naprawczego (gotowe teksty).
4. Domyślny JSON Schema odpowiedzi i kontrakt wyjścia CLI.
5. Kamienie milowe z kryteriami „done”:
   - **M1 (spike):** `copilot-bridge login` + skrypt wysyłający jedno pytanie do M365 Copilota → surowy tekst odpowiedzi; weryfikacja selektorów i trybu headless,
   - **M2:** daemon z trwałą sesją + IPC + auto-start z CLI + `daemon status/stop`,
   - **M3:** prompt-koperta + parsowanie i walidacja JSON + pętla naprawcza,
   - **M4:** kody błędów, stan `auth_required`, logi, debug (trace/screenshot), kolejka i anulowanie,
   - **M5:** własne schematy, komenda `doctor`, testy na atrapie, dokumentacja, pakowanie (opcjonalnie autostart z systemem).
6. Lista ryzyk z prawdopodobieństwem, wpływem i mitygacją.
7. Lista rzeczy świadomie poza zakresem.

Pisz konkretnie. Tam, gdzie decyzja zależy od informacji, której nie masz (np. aktualny DOM strony Copilota), zaznacz to wprost i zaproponuj, jak to zweryfikować.
