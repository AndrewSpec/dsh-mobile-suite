# Profil E2EE szyfrowanego tunelu DSH v1

[English](Tunnel-E2EE-v1.md) | [Polski](Tunnel-E2EE-v1.pl.md)

Dokument angielski [Tunnel-E2EE-v1.md](Tunnel-E2EE-v1.md) pozostaje źródłem nadrzędnym; to tłumaczenie referencyjne.

Ten profil definiuje szyfrowanie warstwy aplikacyjnej używane przez DSH Remote 0.1.4. Relay uwierzytelnia konta i urządzenia oraz kieruje ramki, ale nigdy nie otrzymuje klucza szyfrowania treści ani jawnego ruchu DSH.

## Granica bezpieczeństwa

Profil chroni metody, ścieżki, nagłówki i ciała HTTP, status odpowiedzi, nagłówki odpowiedzi, fragmenty SSE, dane WebSocket oraz powody zamknięcia przed ujawnieniem lub modyfikacją przez płaszczyznę danych Relay. Nie ukrywa powiązań kont/urządzeń, stanu online, czasu połączenia, długości szyfrogramu ani rozkładu ruchu w czasie. Przejęte punkty końcowe Mobile, Browser lub Companion nie są objęte zakresem.

W przypadku Web JavaScript i Service Worker dostarczane przez skonfigurowany origin Relay są częścią zaufanego punktu końcowego Browser. E2EE przeglądarki chroni ruch przed routowaniem Relay, magazynem danych, logami i pasywną inspekcją infrastruktury, ale nie może chronić przed złośliwym wdrożeniem Relay, które zastępuje kod klienta przed wykonaniem. Obrona w takim przypadku wymaga oddzielnie podpisanego klienta z przypiętym kodem, np. aplikacji natywnej lub rozszerzenia przeglądarki.

Wersja 1 używa wstępnie współdzielonego klucza dostarczanego przez QR i nie zapewnia utajnienia z wyprzedzeniem (forward secrecy). Późniejszy profil może dodać uwierzytelniony efemeryczny X25519 bez zmiany wewnętrznej koperty HTTP/WS.

## Bezpieczne parowanie

Companion lokalnie generuje nowy 32-bajtowy `e2eeMasterKey` przy każdej próbie parowania. Nigdy nie jest on wysyłany do Relay. Nowe wersje Companion kodują link przeglądarkowy HTTPS:

```text
https://relay.example/app/#/pair?code=482913&key=<unpadded-base64url-32-bytes>
```

Kod i klucz znajdują się we fragmencie URL, dlatego nie są wysyłane w żądaniu HTTP, Referer, logu dostępu Relay ani docelowej ścieżce żądania reverse proxy. Przeglądarka usuwa fragment po pomyślnym przejęciu kodu. Starsze wydania Mobile mogą nadal akceptować format JSON w wersji 2:

```json
{"v":2,"relay":"https://relay.example","code":"482913","e2eeKey":"<unpadded-base64url-32-bytes>"}
```

Klient Mobile akceptuje wyłącznie QR w wersji 2, sprawdza, czy `e2eeKey` dekoduje się do dokładnie 32 bajtów, przejmuje `code`, a następnie przechowuje klucz w bezpiecznym magazynie platformy pod uwierzytelnionym kontem i zwróconym `deviceId`. Companion zapisuje ten sam klucz dopiero po sukcesie `/pair/confirm`, używając istniejącego pliku konfiguracyjnego z trybem 0600.

Sam sześciocyfrowy kod nie jest bezpieczną inicjalizacją E2EE i NIE MOŻE kończyć parowania produkcyjnego. Odłączenie urządzenia, wylogowanie z konta i ponowne parowanie usuwają powiązany klucz. Istniejące urządzenia MUSZĄ zostać ponownie sparowane przez zeskanowanie QR w wersji 2.

## Autoryzacja Relay

Companion deklaruje `sealed-tunnel-v1` w danych `auth` dla `/device`. `POST /web-ticket` kończy się sukcesem wyłącznie wtedy, gdy wybrane urządzenie jest połączone z tą funkcjonalnością. Rozszerzona odpowiedź ma postać:

```json
{
  "ticket":"<one-time-token>",
  "expiresIn":60,
  "accessSessionId":"access_uuid",
  "tunnelUrl":"wss://relay.example/client-tunnel",
  "e2eeRequired":true
}
```

Mobile otwiera `tunnelUrl` z `Authorization: WebTicket <ticket>`. API WebSocket przeglądarki nie pozwala ustawić nagłówka Authorization, więc Web używa `Sec-WebSocket-Protocol: dsh-e2ee-v1, dsh-ticket.<base64url(utf8(ticket))>`, a Relay wybiera wyłącznie `dsh-e2ee-v1`. Biletów NIE WOLNO umieszczać w URL. Relay zużywa bilet jednokrotnie, przypisuje gniazdo do jego konta, urządzenia i sesji dostępu oraz kieruje wyłącznie poniższe typy komunikatów.

Przeglądarka przechowuje klucz główny według `deviceId` w IndexedDB ograniczonym do originu. Wysyła klucz wyłącznie do Service Workera z tego samego originu, który trzyma materiał aktywnej sesji w pamięci, realizuje opisane uzgadnianie przez Web Crypto i pośredniczy w ruchu DSH HTTP, SSE i WebSocket za pomocą szyfrowanych kopert wewnętrznych. Skrypty stron Relay, skrypty administracyjne i API serwera nigdy nie otrzymują klucza głównego.

## Kodowanie kanoniczne

- Wartości binarne używają base64url bez dopełnienia.
- HMAC i HKDF używają SHA-256.
- AEAD to AES-256-GCM z 16-bajtowym znacznikiem dołączonym do szyfrogramu przed kodowaniem base64url.
- JSON używany przez HMAC/AAD ma postać UTF-8 bez białych znaków. Uwierzytelniane struktury są tablicami, aby kolejność pól była jednoznaczna.
- Wartości sekwencji w transmisji są dziesiętnymi ciągami znaków reprezentującymi 64-bitowe liczby bez znaku.

## Uzgadnianie

Mobile tworzy 32 losowe bajty `clientRandom` i wysyła `client_hello`:

```json
{"v":1,"type":"client_hello","id":"uuid","ts":0,"payload":{"accessSessionId":"access_uuid","clientRandomB64":"...","clientProofB64":"..."}}
```

`clientProof` ma postać:

```text
HMAC(masterKey, utf8(JSON(["dsh-e2ee-client",1,accessSessionId,clientRandomB64])))
```

Po weryfikacji dowodu Companion tworzy 32 losowe bajty `serverRandom` i zwraca `server_hello`. `serverProof` ma postać:

```text
HMAC(masterKey, utf8(JSON(["dsh-e2ee-server",1,accessSessionId,clientRandomB64,serverRandomB64])))
```

Obie strony obliczają:

```text
salt = SHA256(utf8(JSON(["dsh-e2ee-salt",1,accessSessionId,clientRandomB64,serverRandomB64])))
c2dKey       = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:c2d:key"), 32)
d2cKey       = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:d2c:key"), 32)
c2dNonceBase = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:c2d:nonce"), 4)
d2cNonceBase = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:d2c:nonce"), 4)
```

Niepowodzenie dowodu, przekroczenie czasu, brak kluczy, błędny base64url lub nieoczekiwane przejście stanu uzgadniania zamyka bezpieczną sesję. Relay nie może zażądać awaryjnego przejścia na tekst jawny.

## Szyfrowane ramki

Zewnętrzna koperta zachowuje wersję tunelu 1:

```json
{"v":1,"type":"sealed","id":"uuid","ts":0,"payload":{"accessSessionId":"access_uuid","seq":"0","ciphertextB64":"..."}}
```

12-bajtowy nonce GCM to `nonceBase32 || seq64be`. AAD ma postać:

```text
utf8(JSON(["dsh-e2ee",1,accessSessionId,"c2d|d2c",seq]))
```

Tekst jawny stanowi jedną kompletną istniejącą kopertę v1 `http_req`, `http_res`, `http_close`, `ws_open`, `ws_open_ok`, `ws_frame` lub `ws_close`. Relay NIE MOŻE otrzymywać `channel`, pól HTTP/WS ani danych poza szyfrogramem.

Każdy kierunek zaczyna od sekwencji 0 i zwiększa ją dokładnie o jeden. Ramki powtórzone, pominięte, przestawione, o błędnym kierunku, błędnej sesji lub nieuwierzytelnione zamykają bezpieczną sesję, zanim zostanie przekazany jakikolwiek tekst jawny.

## Wspólny wektor o znanym wyniku

Implementacje MUSZĄ odtworzyć ten wektor przed rozpoczęciem współpracy:

```text
masterKeyB64      = AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8
accessSessionId   = access_test_vector
clientRandomB64   = ICEiIyQlJicoKSorLC0uLzAxMjM0NTY3ODk6Ozw9Pj8
clientProofB64    = F3mAmAuR30RnLXq7TMUeMNWZquo8GPHVyCxWycTDW80
serverRandomB64   = QEFCQ0RFRkdISUpLTE1OT1BRUlNUVVZXWFlaW1xdXl8
serverProofB64    = IXvvgrKVbAjjW-M2rLOmf-blsUwmbgLa6y79lEj1vNA
```

Dla pierwszego tekstu jawnego c2d poniżej `seq` wynosi `0`, a `ciphertextB64` ma następującą wartość:

```json
{"v":1,"type":"http_req","channel":"ch_test","id":"request_1","ts":0,"payload":{"method":"POST","path":"/canary","bodyB64":"c2VjcmV0"}}
```

```text
voqHzLsUCWC__C-NBr-s0t1AshPpTEwNwSoOrhx_ihApnwkPlwKxr7EX28kmqOjoVQus161QnjXyzjxDil_WXgnkvu0pgQfiGV27QIgL97KPe2X0nv9vlzLUmwMll0ipeUo2IZKgM-Rt_WRa_-TyyL9SEkozijz1Z6HBxk1hfLiwQEb602fzHVGQP4Wh3Q2B41mrL_-AGQ
```

## Cykl życia i limity

`client_close` i `device_close` kończą sesję dostępu. Rozłączenie Relay/urządzenia/Mobile zamyka wszystkie wewnętrzne kanały HTTP i WebSocket. Relay egzekwuje rozmiar ramek szyfrogramu, przepływność bajtów, limit bezczynności, limit czasu uzgadniania oraz liczbę bezpiecznych sesji na urządzenie i globalnie. Mobile i Companion egzekwują wewnętrzne limity żądań, odpowiedzi, kanałów i tekstu jawnego.

Starsze proxy tekstu jawnego `/s/:deviceId` jest domyślnie wyłączone. Środowisko produkcyjne zwraca `426 {"reason":"e2ee_required"}` dla klientów lub urządzeń bez tego profilu; nigdy nie obniża poziomu bezpieczeństwa po cichu.

## Powody błędów

| Powód | Znaczenie |
| --- | --- |
| `e2ee_required` | Klientowi lub Companion brakuje wymaganej funkcjonalności/klucza |
| `invalid_web_ticket` | Bilet jest nieprawidłowy, wygasły, użyty ponownie lub przypisany gdzie indziej |
| `e2ee_handshake_failed` | Błąd dowodu, stanu, kodowania lub limitu czasu |
| `e2ee_auth_failed` | Niepowodzenie uwierzytelnienia AEAD, sekwencji, kierunku lub przypisania sesji |
| `secure_tunnel_capacity` | Osiągnięto granicę zasobów bezpiecznych sesji Relay |

## Weryfikacja wydania

Testy MUSZĄ obejmować jeden wspólny wektor o znanym wyniku, szyfrowanie/odszyfrowywanie między językami, odrzucanie zmiany bitu, odrzucanie powtórzeń/przestawienia kolejności, odrzucanie błędnej sesji, cykl życia QR/klucza, odrzucanie nieprawidłowego uprawnienia loopback oraz rzeczywiste przesyłanie HTTP/SSE/WebSocket. Logi Relay, ramki i SQLite są skanowane w poszukiwaniu kontrolnego tekstu jawnego.
