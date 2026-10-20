# Zdalny dostęp DSH w przeglądarce i administracja Relay v1

[English](Browser-Remote-and-Relay-Admin-v1.md) | [Polski](Browser-Remote-and-Relay-Admin-v1.pl.md)

Dokument angielski [Browser-Remote-and-Relay-Admin-v1.md](Browser-Remote-and-Relay-Admin-v1.md) pozostaje źródłem nadrzędnym; to tłumaczenie referencyjne.

## Zakres

Relay udostępnia dwa oddzielne interfejsy WWW:

- `/app/` to publiczny klient dla iPhone'a i przeglądarek komputerowych.
- `/admin/` to konsola statystyk przeznaczona wyłącznie dla operatorów, wyłączona do czasu skonfigurowania danych uwierzytelniających.

Publiczny klient przeglądarkowy jest nowym punktem końcowym istniejącego profilu `sealed-tunnel-v1`. Nie jest proxy Relay przesyłającym tekst jawny i nie kopiuje ani nie implementuje na nowo interfejsu produktu DSH. Ładuje oryginalny interfejs WWW lokalnego DSH przez szyfrowaną bramę po stronie przeglądarki.

## Inicjalizacja zaufania w przeglądarce

Companion tworzy sześciocyfrowy kod przejęcia oraz losowy 32-bajtowy klucz główny, a następnie prezentuje:

```text
https://<relay-origin>/app/#/pair?code=<six-digits>&key=<base64url-key>
```

Aparat iPhone'a rozpoznaje QR HTTPS i otwiera Safari. Do serwera wysyłane jest wyłącznie `/app/`. Fragment pozostaje wewnątrz przeglądarki. Po zalogowaniu przeglądarka przejmuje kod, przechowuje klucz pod zwróconym `deviceId` w IndexedDB, usuwa fragment z historii i odświeża listę urządzeń.

Linki parowania są danymi uwierzytelniającymi. Użytkownicy nie mogą wklejać ich do czatu, analityki, raportów awarii ani zgłoszeń pomocy technicznej. Ponowne parowanie powoduje rotację klucza. Usunięcie urządzenia usuwa klucz przeglądarki i unieważnia przypisanie urządzenia w Relay.

## Dodawanie kolejnych przeglądarek

Początkowe parowanie Companion jest celowo jednorazowe. Gdy urządzenie jest już przypisane, Companion lub klient Mobile może wygenerować wielokrotnie używalny link dodania przeglądarki:

```text
https://<relay-origin>/app/#/web-pair?device=<device-id>&key=<base64url-key>
```

Link ponownie wykorzystuje istniejący klucz główny urządzenia i nie wywołuje `/pair/claim`. Przeglądarka musi być już zalogowana na konto będące właścicielem `device`. Pobiera `/devices`, sprawdza, czy urządzenie znajduje się na liście tego konta, a następnie zapisuje klucz z fragmentu w IndexedDB ograniczonym do originu. Klucz pozostaje we fragmencie URL i nigdy nie jest wysyłany do Relay. Każde dodanie przeglądarki dotyczy tylko tej przeglądarki; wygenerowanie kolejnego linku nie unieważnia Mobile ani innych przeglądarek.

Lokalne API zarządzania Companion udostępnia `GET /dsh-mobile/api/browser-access` wyłącznie lokalnym żądaniom zarządzania. Odpowiednikiem w CLI Companion jest `dsh-mobile web`. Zdalne/tunelowane żądania zarządzania otrzymują `local_management_required`. Odłączenie urządzenia lub usunięcie lokalnej autoryzacji przeglądarki unieważnia dostęp; poza tym zwykłe sesje Mobile i Browser mogą równocześnie korzystać z oddzielnych sesji dostępu.

## Brama sesji przeglądarkowej

1. Przeglądarka żąda jednorazowego `/web-ticket` przy użyciu swojej sesji użytkownika HttpOnly.
2. Przeglądarka uruchamia Service Workera DSH o zasięgu katalogu głównego i przekazuje bilet oraz klucz urządzenia przez `postMessage`.
3. Service Worker otwiera `/client-tunnel` z `dsh-e2ee-v1` i podprotokołem zakodowanego biletu.
4. Service Worker i Companion kończą uwierzytelnione uzgadnianie E2EE.
5. Nawigacja do `/remote/:deviceId/` jest przechwytywana lokalnie. HTTP i SSE stają się zaszyfrowanymi kopertami `http_req` i `http_res`.
6. Kod inicjalizacyjny wstawiony do odszyfrowanego HTML DSH mapuje obiekty WebSocket przeglądarki na zaszyfrowane koperty `ws_open`, `ws_frame` i `ws_close`.

Service Worker przepisuje przekierowania loopback na kontrolowany origin przeglądarki, przechowuje ciasteczka serwera docelowego w lokalnym magazynie ważnym tylko przez sesję i usuwa CSP serwera docelowego przed wstawieniem kodu inicjalizacyjnego WebSocket. Do kontrolowanej strony DSH przekazuje wyłącznie odszyfrowaną treść DSH. Jeśli Safari zwolni workera, bezpieczna trasa kończy się, a użytkownik wraca do `/app/`, aby uzyskać nowy bilet.

Pakiet przeglądarkowy hostowany przez Relay jest częścią zaufanego punktu końcowego. Ten projekt chroni tekst jawny przed zwykłym routowaniem Relay, trwałym zapisem, logami, dostępem do bazy danych i pasywną obserwacją infrastruktury. Nie chroni użytkownika przed złośliwym lub przejętym wdrożeniem Relay, które udostępnia zmodyfikowany JavaScript, ponieważ ten kod wykonuje się, zanim przeglądarka ustanowi E2EE. Użytkownicy wymagający ochrony przed samym hostem WWW muszą korzystać z oddzielnie podpisanego klienta natywnego lub rozszerzenia z przypiętym kodem.

## Widoczne metadane

Relay i Admin mogą obserwować:

- powiązanie konta i urządzenia;
- stan online urządzenia i stan DSH;
- początek dostępu, ostatnią aktywność, wygaśnięcie i stan zakończenia;
- etykiety platformy i systemu operacyjnego o ograniczonej długości;
- rozmiary szyfrogramów, rozkład w czasie i liczbę połączeń.

Nie mogą obserwować ścieżek DSH, nagłówków żądań, ciasteczek, ciał, tekstu zadań, wyników modelu, danych WebSocket ani powodów zamknięcia wewnątrz szyfrowanych ramek.

## Granica administracji

Operatorzy ustawiają `ADMIN_USERNAME` i `ADMIN_PASSWORD` w menedżerze sekretów wdrożenia. Puste wartości wyłączają logowanie administracyjne. Uwierzytelnianie administracyjne używa oddzielnego podpisanego Cookie i podlega ograniczeniu częstotliwości wraz z uwierzytelnianiem przeglądarkowym.

Pierwsze wydanie udostępnia wyłącznie statystyki tylko do odczytu:

- łączną liczbę użytkowników i liczbę nowo zarejestrowanych;
- aktywnych użytkowników z 30 dni;
- urządzenia ogółem, sparowane, online oraz z DSH online;
- aktywne i ostatnie sesje dostępu;
- dzienne szeregi nowych użytkowników i sesji dostępu z 30 dni;
- najwyżej 50 wierszy metadanych ostatnich sesji.

Admin v1 nie umożliwia modyfikowania użytkowników, resetowania haseł, sterowania urządzeniami, podszywania się pod użytkowników, inspekcji tunelu ani wyszukiwania treści.

## Warunki wdrożenia

- HTTPS/WSS jest obowiązkowe dla publicznego dostępu przeglądarkowego.
- `PUBLIC_RELAY_URL` musi być kanonicznym originem HTTPS.
- `ALLOW_LEGACY_WEB_PROXY` musi pozostać równe `0`.
- `JWT_SECRET` musi spełniać istniejące produkcyjne kontrole siły sekretu.
- Dane uwierzytelniające administratora muszą istnieć wyłącznie w sekretach wdrożenia.
- Relay pozostaje pojedynczą instancją, dopóki korzysta z lokalnego SQLite i routowania połączeń w pamięci.
