# Plan integracji i wydania

[English](Integration-Plan.md) | [Polski](Integration-Plan.pl.md)

Dokument angielski [Integration-Plan.md](Integration-Plan.md) pozostaje źródłem nadrzędnym; to tłumaczenie referencyjne.

## Odpowiedzialność

| Kamień milowy | Mobile | Plugin | Relay |
| --- | --- | --- | --- |
| Cykl życia konta i tokenów | Integracja | - | Implementacja |
| Parowanie QR/przejęcie kodu | Integracja skanera | Tworzenie/potwierdzanie | Implementacja |
| Lista urządzeń i statusy | Integracja UI | Raportowanie statusu | Implementacja |
| Bilet WebView i ciasteczko | Integracja kontenera | - | Implementacja |
| Zdalne HTTP/WS DSH | Weryfikacja UI | Przekazywanie lokalnego DSH | Kierowanie tunelu |
| Wdrożenie Railway | Weryfikacja URL w chmurze | Ustawienie URL Relay | Wdrożenie i obsługa |

## Lokalna sekwencja end-to-end

1. Zbuduj `dsh-plugin` 0.1.4, a następnie zainstaluj go przez `dsh plugin --profile web add "/absolute/path/to/dsh-plugin"`.
2. Uruchom Relay z deweloperską bazą danych i jawnie ustawionym lokalnym sekretem JWT.
3. Uruchom `dsh web`; zainstalowany pakiet uruchamia Companion i przypisuje go do rzeczywistego portu WWW DSH. Otwórz **Settings > Remote Access** i jawnie wygeneruj dane QR/kod w stanie niesparowanym.
4. Zarejestruj/zaloguj się w aplikacji mobilnej, przejmij kod i poczekaj na potwierdzenie Companion.
5. Sprawdź, czy `/devices` raportuje `online` oraz `dshStatus: online`.
6. Poproś o bilet, ustanów `/client-tunnel`, załaduj origin loopback Mobile i wyślij nowe zadanie DSH.
7. Sprawdź, czy ruch HTTP, SSE i WebSocket jest szyfrowany end-to-end, a obserwacje Relay nie zawierają kontrolnego tekstu jawnego.
8. Zatrzymaj lokalny DSH i potwierdź `503 dsh_offline`; zatrzymaj Companion i potwierdź `503 device_offline`.
9. Odłącz urządzenie w aplikacji mobilnej i sprawdź, czy Companion nie może połączyć się ponownie przy użyciu unieważnionego tokena.

Weryfikacja chmurowa MVP z 2026-08-16 zrealizowała kroki 1–7 wobec `https://dsh-relay-production.up.railway.app`: zdalna przeglądarka załadowała zasoby DSH i wtyczki klienckie, wybrała katalog hosta przez `host.listDirectory`, utworzyła obszar roboczy/sesję i dostarczyła `session.prompt`. Wykonanie modelu zatrzymało się następnie na oczekiwanej granicy `MISSING_CREDENTIAL`, ponieważ odizolowany profil DSH celowo nie miał klucza API DeepSeek.

## Warunki wydania Railway

- Ustaw `JWT_SECRET` na wygenerowany sekret produkcyjny; nigdy nie używaj domyślnej wartości deweloperskiej.
- Skonfiguruj trwały magazyn dla SQLite lub zastąp implementację Store przed skalowaniem powyżej jednej instancji.
- Railway musi zapewniać HTTPS przed dopuszczeniem mobilnego WebView; skonfiguruj publiczny origin Relay w Companion i kompilacji mobilnej.
- Wykonaj całą lokalną sekwencję dla URL Railway z rzeczywistym Companion po stronie komputera.
- Potwierdź, że logi aplikacji zawierają wyłącznie metadane żądań, bez nagłówka autoryzacji, hasła, tokena, ciała HTTP ani treści ramek WebSocket.
- Odrzucaj QR v1, parowanie wyłącznie sześciocyfrowym kodem, brak kluczy, brak funkcjonalności, powtórzenia, przestawione ramki i wszystkie próby awaryjnego przejścia na tekst jawny.
- Sprawdź, czy zmiana szyfrogramu/znacznika/AAD zamyka bezpieczną sesję, zanim treść dotrze do DSH.
- Sprawdź, czy zbyt duże dane wejściowe API, tunelu HTTP i WebSocket są odrzucane bez restartowania Relay.
- Sprawdź, czy limity częstotliwości na klienta i limity równoczesnych tuneli na urządzenie zwracają udokumentowane błędy `413`/`429`.
