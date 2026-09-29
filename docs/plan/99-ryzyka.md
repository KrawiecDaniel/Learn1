# 99. Ryzyka techniczne i poza zakresem

## Ryzyka

| Ryzyko | Prawdop. | Wpływ | Mitygacja |
|---|---|---|---|
| Zmiana interfejsu M365 Copilota psuje selektory | wysokie | wysoki | Wszystkie selektory w `selectors.yaml` z kandydatami zapasowymi; błąd `UI_CHANGED` ze zrzutem ekranu w `debug\`; `m1_probe.py check-selectors` do szybkiej naprawy; atrapa strony w testach (M7) |
| Wygaśnięcie sesji M365 w środku dnia | średnie | wysoki | Stan `auth_required` w `/v1/health`, błąd `AUTH_REQUIRED` z instrukcją, `copilot-bridge login` bez restartu daemona |
| Conditional Access blokuje Edge uruchomiony przez Playwright | niskie–średnie | wysoki | Sprawdzone jako pierwsze w M1; zainstalowany Edge zamiast Chromium; osobny, trwały profil |
| Tryb headless blokowany przez SSO | średnie | niski | Domyślnie `offscreen`; M1 wybiera tryb |
| Copilot nie zwraca stabilnie poprawnego JSON-a | średnie | średni | Ścisła koperta promptu, ekstrakcja z bloku kodu, lokalna naprawa, prompt naprawczy, `request_id` jako kontrola |
| Znaczniki cytowań psują JSON w trybie Work | średnie | niski | Usuwanie elementów cytowań z DOM przed odczytem, `strip_citations`, źródła z interfejsu zamiast z modelu |
| Niedeterministyczne decyzje | wysokie | średni | Enum z opcji, próg pewności, `needs_review`, głosowanie, dziennik decyzji, ewaluacja (M7) |
| Dławienie albo limity usługi | średnie | średni | `min_interval_s`, batch zamiast serii pojedynczych zapytań, cache, `RATE_LIMITED` z `Retry-After` |
| Uśpienie i VPN zrywają połączenie strony | średnie | średni | `WakeDetector`, stan `resuming`, przeładowanie albo restart przeglądarki, automatyczne ponowienie przerwanych zadań |
| Pamięć albo personalizacja Copilota zmienia wyniki | niskie–średnie | średni | Sprawdzenie w M1 i wyłączenie w ustawieniach; każde zadanie w nowej rozmowie |
| Historia czatów zapełnia się rozmowami daemona | wysokie | niski | Opcja `history.cleanup: delete`, gdy M1 potwierdzi automatyzację usuwania |
| Wstrzyknięte polecenia w danych (maile, dokumenty w trybie Work) | niskie | wysoki | Delimitery danych w promptach, walidacja schematem, `needs_review`, klient waliduje wywołania narzędzi przed wykonaniem |
| Aktualizacja Edge rozjeżdża się z Playwrightem | niskie | średni | Aktualizacja `playwright` w `.venv`; `doctor` wykrywa problem |
| Copilot piszący kod gubi spójność między plikami | wysokie | średni | Bloki interfejsów, małe pliki, test po każdym kroku, szablon naprawczy |
| Ucięte odpowiedzi Copilota piszącego kod | średnie | niski | Limit ≈350 linii na plik, szablon C, `py_compile` po sklejeniu |
| Instalacja pakietów blokowana przez firmowe proxy | średnie | wysoki | `HTTPS_PROXY` przy `pip`; w ostateczności paczki `.whl` pobrane innym zatwierdzonym kanałem |
| Długie odpowiedzi przekraczają timeouty klientów (Excel, PAD) | średnie | niski | Zalecane timeouty w przykładach, joby asynchroniczne z odpytywaniem |
| Daemon ginie razem z terminalem albo przepływem PAD | średnie | średni | `DETACHED_PROCESS` + `CREATE_BREAKAWAY_FROM_JOB`, autostart z Windowsem |
| Plik Excel otwarty albo współedytowany na SharePoincie blokuje zapis | wysokie | średni | Zapis przez plik tymczasowy, błąd ponawialny, `retry` w kroku; rejestry zapisywane przez przepływy nie powinny być równocześnie edytowane ręcznie |
| Monity bezpieczeństwa Outlooka przy dostępie przez COM | niskie–średnie | wysoki | Sprawdzenie w M8.0; w razie monitów wariant M8.11w (Outlook w przeglądarce) |
| Nowy Outlook bez COM | średnie | średni | Wariant M8.11w |
| Zmiany ekranów ERP psują nagrane skrypty | wysokie | średni | Skrypty jako osobne pliki poprawiane bez restartu; zrzut ekranu przy błędzie; na start tylko odczyt z ERP |
| AI pisze przepływ z błędem | wysokie | niski | `flow check`, `--dry-run`, zasady w karcie przepływów, szkice zamiast wysyłki |
| Harmonogram pomija przebiegi przy uśpieniu albo na baterii | średnie | niski | `StartWhenAvailable=true`, wyłączone ograniczenia baterii w XML zadania |

## Poza zakresem

- Wielu użytkowników i dostęp z innych komputerów (daemon słucha tylko na `127.0.0.1`).
- Strumieniowanie odpowiedzi.
- Równoległa praca wielu kart albo przeglądarek.
- Generowanie obrazów, plików i stron Copilot Pages.
- Automatyczne logowanie i obchodzenie MFA (logowanie zawsze ręczne).
- HTTPS na lokalnym porcie.
- Systemy inne niż Windows.
- Pojedynczy plik `.exe` (PyInstaller) i instalator MSI.
- Interfejs graficzny poza prostym playgroundem (także edytor przepływów).
- Wyzwalacze przepływów w czasie rzeczywistym (webhooki, zdarzenia); tylko harmonogram i uruchomienie ręczne.
