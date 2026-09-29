# Prompt planistyczny: Copilot Bridge (CLI → Playwright → JSON)

Jesteś doświadczonym architektem oprogramowania. Przygotuj szczegółowy plan implementacji narzędzia **copilot-bridge**. Nie pisz jeszcze kodu produkcyjnego. Twoim zadaniem jest plan, który da się potem wykonać krok po kroku.

## Cel

Narzędzie CLI, które:
1. przyjmuje pytanie od użytkownika (argument, stdin albo plik),
2. opakowuje je w prompt wymuszający odpowiedź w ustalonym schemacie JSON,
3. przez Playwright steruje przeglądarką z zalogowaną sesją Copilota, wysyła prompt i czeka na pełną odpowiedź,
4. wyciąga odpowiedź ze strony, parsuje ją, waliduje względem schematu i w razie potrzeby naprawia,
5. zwraca na stdout **wyłącznie** poprawny JSON, który da się przetwarzać maszynowo (np. `jq` albo inny program).

Przykład użycia:
```
copilot-bridge ask "Jakie są zalety Rusta?" --schema default --timeout 120
echo "pytanie" | copilot-bridge ask --stdin --schema ./schemas/custom.json
```

## Pytania, które trzeba rozstrzygnąć na początku

Zanim zaproponujesz architekturę, wypisz te pytania i podaj rekomendowaną odpowiedź na każde:
- **Który Copilot?** Microsoft Copilot (copilot.microsoft.com), M365 Copilot (konto firmowe) czy GitHub Copilot Chat (github.com/copilot)? Od tego zależą logowanie, selektory i ograniczenia.
- **Czy istnieje oficjalna droga**, czyli API, CLI albo SDK dla wybranego Copilota, która zastąpiłaby scraping? Porównaj ją z Playwrightem pod kątem stabilności, zgodności z regulaminem (ToS) i kosztu utrzymania.
- **Stos technologiczny:** TypeScript/Node czy Python? Uzasadnij wybór (Playwright, walidacja schematu, dystrybucja CLI).
- **Tryb pracy:** każde wywołanie uruchamia własną przeglądarkę albo działa daemon z trwałą sesją, do którego CLI łączy się przez socket lub HTTP na localhost. Porównaj opóźnienia i złożoność.

## Obszary, które plan musi pokryć

### 1. Architektura i przepływ danych
Diagram albo opis warstw: CLI → (opcjonalnie daemon) → PromptBuilder → BrowserDriver (Playwright) → ResponseExtractor → JsonParser/Validator → Output. Określ odpowiedzialność każdej warstwy i interfejsy między nimi.

### 2. Sesja i logowanie
- Pierwsze logowanie w trybie headed (użytkownik loguje się ręcznie, łącznie z MFA), potem trwały profil (`launchPersistentContext` / `storageState`).
- Wykrywanie wygasłej sesji i czytelny błąd `AUTH_REQUIRED` zamiast zawieszenia.
- Gdzie i jak bezpiecznie przechowywać profil i ciasteczka.

### 3. Sterowanie stroną
- Strategia selektorów: role/aria/teksty zamiast kruchych klas CSS, wszystkie selektory w jednym pliku konfiguracyjnym, żeby łatwo je aktualizować.
- Nowa rozmowa dla każdego zapytania albo reużycie wątku (wpływ na kontekst i izolację).
- Wklejanie długich promptów (fill albo schowek zamiast wpisywania znak po znaku).
- **Wykrywanie końca odpowiedzi strumieniowanej:** zniknięcie przycisku „Stop”, stabilność tekstu przez N ms, pojawienie się przycisków akcji. Zaproponuj metodę główną i zapasową.
- Obsługa CAPTCHA, limitów, komunikatów o błędach i zmian w UI: jak je wykryć i jaki kod błędu zwrócić.

### 4. Ekstrakcja odpowiedzi
- Skąd brać tekst: `innerText` wyrenderowanego markdownu, blok kodu, przycisk „Kopiuj” czy przechwytywanie ruchu sieciowego (WebSocket/SSE). Oceń ryzyko zniekształcenia JSON-a przez renderowanie markdownu przy każdej opcji.
- Jak jednoznacznie zidentyfikować właściwą odpowiedź, np. przez unikalny `request_id` (nonce) w prompcie, który model musi odesłać.

### 5. Prompt wymuszający schemat (kluczowe)
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

### 6. Kontrakt wyjścia CLI
Zdefiniuj stałą „kopertę” wyjściową niezależną od schematu modelu:
```json
{
  "ok": true,
  "request_id": "uuid",
  "data": { },
  "raw": "surowy tekst odpowiedzi (opcjonalnie, flaga --include-raw)",
  "error": null,
  "meta": { "duration_ms": 0, "attempts": 1, "copilot_variant": "...", "schema": "default" }
}
```
- Kody błędów (`AUTH_REQUIRED`, `TIMEOUT`, `UI_CHANGED`, `RATE_LIMITED`, `INVALID_JSON`, `SCHEMA_MISMATCH`, `CAPTCHA`) i odpowiadające im kody wyjścia procesu.
- Zasada: stdout zawiera tylko JSON, a logi idą na stderr lub do pliku (`--verbose`, `--debug` ze zrzutami ekranu i trace'ami Playwrighta).

### 7. Współbieżność i niezawodność
- Kolejka zapytań (jedna karta albo pula kart), blokady, timeouty globalne i per etap.
- Retry z backoffem tylko dla błędów przejściowych.
- Odtwarzanie po awarii przeglądarki.

### 8. Testowanie
- Testy jednostkowe PromptBuildera, parsera JSON (ekstrakcja z fence'ów, naprawa drobnych usterek) i walidatora.
- Testy BrowserDrivera na lokalnej atrapie strony (fixture HTML symulujący streaming), żeby nie odpytywać prawdziwego Copilota w CI.
- Ręczny smoke test na prawdziwym serwisie z checklistą.
- Wykrywanie regresji po zmianie UI Copilota (np. osobna komenda `copilot-bridge doctor`).

### 9. Bezpieczeństwo i zgodność
- Ryzyka regulaminowe (ToS) automatyzacji UI i rekomendacja, jak je ograniczyć (użytek osobisty, limity częstotliwości, brak omijania zabezpieczeń).
- Ochrona danych: co trafia do logów, maskowanie treści pytań, uprawnienia do katalogu profilu.

### 10. Struktura repozytorium i konfiguracja
Proponowane drzewo katalogów, plik konfiguracyjny (URL, selektory, timeouty, domyślny schemat), zmienne środowiskowe i sposób instalacji CLI.

## Oczekiwany format planu

1. Rozstrzygnięcia otwartych pytań (z rekomendacją i uzasadnieniem).
2. Architektura z diagramem (ASCII lub mermaid).
3. Pełny szablon promptu-koperty i promptu naprawczego (gotowe teksty).
4. Domyślny JSON Schema odpowiedzi i kontrakt wyjścia CLI.
5. Kamienie milowe z kryteriami „done”:
   - **M1 (MVP):** logowanie + jedno pytanie → surowy tekst odpowiedzi,
   - **M2:** prompt-koperta + parsowanie i walidacja JSON + pętla naprawcza,
   - **M3:** kody błędów, logowanie, debug (trace/screenshot),
   - **M4:** daemon/kolejka, własne schematy, komenda `doctor`,
   - **M5:** testy na atrapie, dokumentacja, pakowanie.
6. Lista ryzyk z prawdopodobieństwem, wpływem i mitygacją.
7. Lista rzeczy świadomie poza zakresem.

Pisz konkretnie. Tam, gdzie decyzja zależy od informacji, której nie masz (np. aktualny DOM strony Copilota), zaznacz to wprost i zaproponuj, jak to zweryfikować.
