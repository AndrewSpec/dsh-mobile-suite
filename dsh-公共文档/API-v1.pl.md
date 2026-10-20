# DSH Remote API v1

[English](API-v1.md) | [Polski](API-v1.pl.md)

Dokument angielski [API-v1.md](API-v1.md) pozostaje źródłem nadrzędnym; to tłumaczenie referencyjne.

Bazowy URL: `https://<relay-host>` w środowiskach chmurowych i `http://127.0.0.1:<port>` lokalnie. Żądania i odpowiedzi JSON używają `application/json`, chyba że wskazano inaczej.

## Uwierzytelnianie

Tokeny dostępu są tokenami typu bearer ważnymi przez siedem dni. Tokeny odświeżania są nieprzezroczystymi, jednorazowymi danymi uwierzytelniającymi ważnymi przez 30 dni. Klient mobilny przechowuje oba w bezpiecznym magazynie. Relay przechowuje wyłącznie skrót tokena odświeżania.

Błędy REST API mają postać `{"error":"<code>"}`. Błędy zwracane przez proxy WebView mają postać `{"reason":"<code>"}`, ponieważ klient mobilny przedstawia je jako widoczne dla użytkownika stany połączenia.

Domyślny limit każdego ciała JSON API wynosi 64 KiB. Zbyt duże ciała zwracają `413 {"error":"request_too_large","limit":65536}`. Wywołania API objęte ograniczeniem częstotliwości zwracają `429 {"error":"rate_limited","retryAfter":<seconds>}` z takim samym opóźnieniem w nagłówku `Retry-After`. Oprócz ogólnego limitu API trasy uwierzytelniania i parowania mają bardziej rygorystyczne limity na klienta.

## Polityka wersji aplikacji mobilnej

### `GET /app/version?platform=android|ios` (publiczne)

Zwraca politykę wydań dla jednej platformy mobilnej:

```json
{
  "platform": "android",
  "latestVersion": "0.2.0",
  "minimumVersion": "0.1.3",
  "downloadUrl": "https://play.google.com/store/apps/details?id=io.github.apriljk.dshremote",
  "releaseNotes": "Improved remote session stability."
}
```

`latestVersion` wyświetla możliwy do zamknięcia komunikat o aktualizacji, gdy jest nowsza niż zainstalowana wersja. `minimumVersion` wyświetla komunikat, którego nie można zamknąć, gdy jest nowsza niż zainstalowana wersja. Wersje są porządkowane numerycznie według segmentów rozdzielonych kropkami, np. `1.4.2`; wersje przedpremierowe są umieszczane poniżej odpowiadającej im wersji stabilnej. `downloadUrl` i `releaseNotes` mogą mieć wartość `null`, gdy nie opublikowano aktualizacji. Nieznane platformy zwracają `400 {"error":"unsupported_platform"}`.

Klient mobilny sprawdza wersję po uruchomieniu, przy każdym powrocie na pierwszy plan oraz po ręcznym sprawdzeniu przez użytkownika w Ustawieniach. Relay może przechowywać tę publiczną odpowiedź w pamięci podręcznej przez najwyżej pięć minut. Operatorzy muszą opublikować docelową stronę sklepu/pobierania przed zwiększeniem którejkolwiek skonfigurowanej wersji.

### `POST /auth/register`

Żądanie:

```json
{"email":"person@example.com","password":"at-least-8-characters"}
```

Zwraca `201` z takim samym ciałem jak logowanie. Powtórzone adresy e-mail należy odrzucać kodem `409 {"error":"email_exists"}`. W MVP hasła muszą mieć co najmniej 8 znaków.

### `POST /auth/login`

Żądanie: te same pola co przy rejestracji. Zwraca `200`:

```json
{"accessToken":"...","refreshToken":"..."}
```

Nieprawidłowe dane uwierzytelniające należy odrzucać kodem `401 {"error":"invalid_credentials"}`. Nie wolno ujawniać, czy adres e-mail istnieje.

### `POST /auth/refresh`

Żądanie: `{"refreshToken":"..."}`. Zwraca nową parę tokenów po rotacji. Użyty, wygasły lub nieprawidłowy token odświeżania zwraca `401 {"error":"invalid_refresh_token"}`.

Wszystkie poniższe trasy oznaczone **User** wymagają `Authorization: Bearer <accessToken>`.

Klient przeglądarkowy używa `/web-auth/register`, `/web-auth/login`, `/web-auth/session` i `/web-auth/logout`. Pomyślna rejestracja lub logowanie ustawia podpisane ciasteczko `dsh_web_auth` z `HttpOnly; SameSite=Strict; Path=/`; JavaScript przeglądarki nigdy nie otrzymuje tokenów bearer ani tokenów odświeżania. Operacje zmieniające stan uwierzytelnione ciasteczkiem wymagają żądania przeglądarki z tego samego originu. Zachowanie natywnego klienta z tokenem bearer pozostaje bez zmian.

## Parowanie

Sześciocyfrowy kod i sekret urządzenia są celowo rozdzielone. Sekret należy do Companion; po przejęciu kodu klient mobilny otrzymuje jedynie tożsamość urządzenia.

### `POST /pair/session` (Companion)

Z ograniczeniem częstotliwości, bez uwierzytelnienia użytkownika. Zwraca:

```json
{
  "deviceId":"dev_xxx",
  "deviceSecret":"hex-32-bytes",
  "code":"482913",
  "expiresAt":1786860000000
}
```

`expiresAt` to czas uniksowy w milisekundach. Companion generuje również lokalny 32-bajtowy klucz główny E2EE i prezentuje `https://<relay-host>/app/#/pair?code=482913&key=<base64url>`. Fragment URL nigdy nie jest wysyłany do Relay. Mobile akceptuje ten link oraz starszy format JSON QR w wersji 2; ręczne parowanie sześciocyfrowym kodem nie może ustanowić podstawy zaufania E2EE.

### `POST /pair/claim` (User)

Żądanie: `{"code":"482913"}`. Zwraca `200 {"deviceId":"dev_xxx"}`. Nieprawidłowy, wygasły, użyty lub przejęty przez innego użytkownika kod zwraca `409 {"error":"invalid_or_expired_code"}`.

### `POST /pair/confirm` (Companion)

Żądanie:

```json
{"deviceId":"dev_xxx","deviceSecret":"...","deviceName":"Watson's MacBook Air"}
```

Przed przejęciem kodu zwraca `202 {"status":"pending"}`. Po sukcesie zwraca `200 {"deviceToken":"..."}`. Companion trwale zapisuje ten token i używa go do uwierzytelnienia połączenia WSS. Relay przechowuje wyłącznie jego skrót.

## Urządzenia

### `GET /devices` (User)

Zwraca `200`:

```json
{
  "devices":[{
    "id":"dev_xxx",
    "name":"Watson's MacBook Air",
    "online":true,
    "dshStatus":"online",
    "lastSeenAt":1786860000000
  }]
}
```

`online` odzwierciedla połączenie WSS urządzenia. `dshStatus` to ostatnia obserwacja lokalnego stanu DSH dokonana przez Companion. `lastSeenAt` to czas uniksowy w milisekundach lub `null`.

### `PATCH /devices/:id` (User)

Żądanie: `{"name":"New name"}`. Zwraca `200 {"ok":true}`. Wywołujący musi być właścicielem urządzenia. Po sukcesie Mobile odświeża `GET /devices`.

### `DELETE /devices/:id` (User)

Zwraca `200 {"ok":true}`. Relay natychmiast unieważnia token urządzenia i zamyka aktywne połączenie Companion. Kolejna próba połączenia Companion musi otrzymać `auth_fail` i wrócić do parowania.

### `POST /device-management/:id/unbind` (Companion)

Wymaga `Authorization: Device <deviceToken>`. Zwraca `200 {"ok":true}` po atomowym usunięciu przypisania do konta i unieważnieniu tokena urządzenia. Relay zamyka aktywne połączenie Companion i kończy aktywne sesje dostępu WebView z powodem `unbound`.

Ten punkt końcowy jest używany przez lokalną stronę ustawień DSH **Remote Access** oraz polecenie odzyskiwania `dsh-mobile unpair`. Nieprawidłowe lub już unieważnione dane uwierzytelniające urządzenia zwracają `401 {"error":"invalid_device_token"}`. Companion usuwa lokalne `deviceId`, `deviceSecret` i `deviceToken` dopiero po potwierdzeniu odłączenia przez Relay, a następnie wraca do stanu niesparowanego. Zdalne żądania WebView nie mogą wywoływać tej operacji.

## Autoryzacja WebView i proxy

### `POST /web-ticket` (User)

Żądanie: `{"deviceId":"dev_xxx"}`. Dla podłączonego Companion z `sealed-tunnel-v1` zwraca `200 {"ticket":"...","expiresIn":60,"accessSessionId":"access_xxx","tunnelUrl":"wss://<relay-host>/client-tunnel","e2eeRequired":true}`. Bilety są jednorazowe i przypisane do konta, urządzenia oraz sesji dostępu. Brak bezpiecznej funkcjonalności zwraca `426 {"reason":"e2ee_required"}`.

Klient mobilny łączy się z biletem w nagłówku `Authorization: WebTicket`, a następnie ładuje swoją prywatną bramę loopback:

```
http://127.0.0.1:<ephemeral>/?bootstrap=<local-capability>
```

Brama loopback zużywa lokalne uprawnienie bootstrap, ustawia ciasteczko `HttpOnly; SameSite=Strict; Path=/` i udostępnia odszyfrowane odpowiedzi. Bezwzględne adresy DSH `/assets`, `/plugins`, `/api` oraz główne adresy WebSocket pozostają w originie loopback. Żaden token bearer Relay, dane uwierzytelniające urządzenia ani klucz E2EE nie trafiają do JavaScript WebView.

Relay `/client-tunnel` przekazuje wyłącznie nieprzezroczyste ramki E2EE. Starsze proxy `/s/:deviceId` jest domyślnie wyłączone i NIE WOLNO go używać jako automatycznego mechanizmu awaryjnego.

Klienci przeglądarkowi kodują ten sam jednorazowy bilet w podprotokole WebSocket `dsh-ticket.<base64url(utf8(ticket))>` i wymagają, by Relay wybrał `dsh-e2ee-v1`. Service Worker o zasięgu katalogu głównego mapuje ruch DSH HTTP, SSE i WebSocket z tego samego originu na szyfrowane koperty wewnętrzne. Klucz pozostaje w IndexedDB i pamięci Service Workera; nigdy nie trafia do żądania HTTP, Cookie, bazy danych Relay ani odpowiedzi panelu administracyjnego.

## Administracja Relay

`/admin/` jest wyłączone, chyba że zarówno `ADMIN_USERNAME`, jak i `ADMIN_PASSWORD` skonfigurowano jako sekrety wdrożenia. `POST /admin/api/login` tworzy odizolowaną, ośmiogodzinną sesję `HttpOnly; SameSite=Strict; Path=/admin`. `GET /admin/api/stats` zwraca zagregowane metadane kont, urządzeń, stanów online i sesji dostępu oraz najwyżej 50 ostatnich sesji dostępu.

Odpowiedzi administracyjne NIE MOGĄ zawierać skrótów haseł, tokenów, danych uwierzytelniających urządzeń, kluczy E2EE, adresów IP, ścieżek URL DSH, ciał HTTP, ramek WebSocket, tekstu zadań ani wyników modelu. Sesja administracyjna nie jest akceptowana przez API User ani Companion.

## Błędy proxy widoczne dla użytkownika

| HTTP | Ciało | Znaczenie w Mobile |
| --- | --- | --- |
| `401` | `{"reason":"invalid_local_session"}` | Utwórz ponownie lokalne proxy i poproś o nowy bilet |
| `403` | `{"reason":"invalid_local_origin"}` | Odrzuć żądanie loopback z nieoczekiwanym Host |
| `403` | `{"error":"forbidden"}` | Konto nie jest właścicielem urządzenia lub nie może poprosić o bilet |
| `413` | `{"reason":"request_too_large"}` | Przekazywane żądanie przekracza wewnętrzny limit Mobile |
| `429` | `{"reason":"too_many_tunnels"}` | Tymczasowo wyczerpano pojemność kanałów wewnętrznych Mobile lub Companion |
| `429` | `{"error":"rate_limited","retryAfter":<seconds>}` | Klient przekroczył limit częstotliwości API Relay lub bezpiecznych ramek |
| `502` | `{"reason":"secure_tunnel_failed"}` | Uwierzytelniony tunel uległ awarii lub odrzucił odpowiedź wewnętrzną |
| `503` | `{"reason":"device_offline"}` | Komputer/Companion jest offline |
| `503` | `{"reason":"dsh_offline"}` | Komputer jest online, ale lokalny DSH jest niedostępny |
| `504` | `{"reason":"tunnel_timeout"}` | Błąd łączności umożliwiający ponowienie próby |

Domyślne granice wewnętrzne Mobile i Companion to 2 MiB na żądanie, 32 MiB na odpowiedź, 32 równoczesne kanały HTTP i 16 równoczesnych kanałów WebSocket na bezpieczną sesję. Relay osobno ogranicza liczbę bezpiecznych sesji, częstotliwość ramek, liczbę bajtów szyfrogramu, czas uzgadniania i czas bezczynności. Wdrożenia mogą obniżać te limity. Klienci muszą traktować odpowiedzi pojemnościowe `429` i `503` jako umożliwiające ponowienie próby z narastającym opóźnieniem.

Klient mobilny musi zezwalać na nawigację WebView wyłącznie w obrębie aktywnego originu loopback. Zewnętrzne originy HTTPS otwierają się w przeglądarce systemu operacyjnego; wszystkie inne originy są blokowane.
