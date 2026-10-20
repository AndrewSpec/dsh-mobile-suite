# DSH Remote MVP — interfejsy międzyplatformowe

[中文](DSH-远程MVP-跨端接口.md) | [English](DSH-Remote-MVP-Interfaces.en.md) | [Polski](DSH-Remote-MVP-Interfaces.pl.md)

To tłumaczenie referencyjne chińskiego podsumowania.

Ten plik jest wspólnym kontraktem dla klienta mobilnego, Companion po stronie komputera i Relay. Payloady biznesowe nie są zapisywane w logach Relay.

## Konwencje adresów

- Lokalny Relay deweloperski: `http://127.0.0.1:8787`
- Relay produkcyjny: domena HTTPS Railway, na przykład `https://dsh-relay.example`
- Połączenie urządzenia z Companion: zamień protokół adresu Relay na `ws` / `wss`, ścieżka `/device`
- Wejście WebView: lokalny adres Mobile `http://127.0.0.1:<ephemeral>/{path}`; Relay udostępnia wyłącznie szyfrogram WSS `/client-tunnel`

## Uwierzytelnianie

`POST /auth/register` `{ email, password }`, hasło co najmniej 8 znaków, zwraca `{ accessToken, refreshToken }`.

`POST /auth/login` — jak wyżej. `POST /auth/refresh` `{ refreshToken }` rotuje refresh token. Access token używa `Authorization: Bearer <token>` i jest ważny 7 dni; refresh token jest ważny 30 dni.

## Parowanie

1. Companion wywołuje `POST /pair/session` i otrzymuje `{ code, deviceId, deviceSecret, expiresAt }`. deviceSecret jest przechowywany wyłącznie lokalnie na komputerze.
2. Companion lokalnie generuje 32-bajtowy master key i koduje `{"v":2,"relay":"...","code":"...","e2eeKey":"<base64url>"}` jako kod QR; master key nie jest wysyłany do Relay, a parowanie produkcyjne nie akceptuje samego sześciocyfrowego code.
3. Telefon wywołuje `POST /pair/claim` `{ code }` z JWT użytkownika.
4. Companion wywołuje `POST /pair/confirm` `{ deviceId, deviceSecret, deviceName }` co 2 sekundy. Przed przejęciem kodu zwraca `202 { status: "pending" }`, a po przejęciu zwraca `{ deviceToken }`.
5. Następnie Companion używa wyłącznie `deviceId + deviceToken` do ustanowienia połączenia WSS `/device`.

Kody parowania są ważne 5 minut i jednorazowe; deviceToken staje się nieważny natychmiast po odłączeniu urządzenia.

## Zarządzanie urządzeniami

- `GET /devices`: zwraca `{ devices: [{ id, name, online, dshStatus, lastSeenAt }] }`
- `PATCH /devices/:id` `{ name }`
- `DELETE /devices/:id`
- `POST /device-management/:id/unbind`: Companion używa `Authorization: Device <deviceToken>` lokalnie na komputerze, aby usunąć parowanie całego komputera; Relay unieważnia token urządzenia, kończy sesje dostępu i rozłącza bieżące połączenie
- `POST /web-ticket` `{ deviceId }`: zwraca jednorazowy ticket ważny 60 sekund

Mobile używa jednorazowego ticketu w `Authorization: WebTicket`, aby połączyć się z `/client-tunnel`, a następnie uruchamia lokalną bramę HTTP/WS powiązaną wyłącznie z loopback. WebView uzyskuje dostęp wyłącznie do lokalnej bramy; HTTP, SSE i WebSocket są przekazywane jako szyfrowane koperty wewnętrzne. Ticket można użyć tylko raz i nie może on występować w URL.

## Koperta tunelu

Zewnętrzne ramki Companion, Mobile i Relay pozostają w formacie JSON v1; koperty biznesowe są szyfrowane zgodnie z `Tunnel-E2EE-v1.md`:

```json
{
  "v": 1,
  "type": "auth | auth_ok | http_req | http_res | http_close | ws_open | ws_open_ok | ws_frame | ws_close | status | ping | pong | event",
  "channel": "ch_uuid",
  "id": "message_uuid",
  "ts": 1755500000000,
  "payload": {}
}
```

Pierwsza ramka po ustanowieniu połączenia urządzenia musi mieć postać:

```json
{"v":1,"type":"auth","id":"...","ts":0,"payload":{"deviceId":"dev_x","deviceToken":"..."}}
```

Odszyfrowany payload żądania HTTP to `{ method, path, headers, bodyB64 }`; payload odpowiedzi to `{ status?, headers?, bodyB64, seq, final }`. Odpowiedzi są dzielone na fragmenty z rosnącym `seq`, co obsługuje SSE i długie odpowiedzi. `Host`, `Origin` i `Referer` są przepisywane przez Companion na lokalny adres DSH. Gdy lokalna przeglądarka Mobile przerwie żądanie, wysyła szyfrowany `http_close`, a Companion anuluje lokalne żądanie.

WebSocket używa tego samego szyfrowanego wewnętrznego `channel`: lokalna brama Mobile wysyła `ws_open`, obie strony wymieniają `ws_frame`, a strona zamykająca wysyła `ws_close`. Relay widzi wyłącznie sealed frame.

Ramka statusu: `status { dsh: "online" | "offline" }`. Gdy Relay nie ma połączenia z urządzeniem, zwraca `503 {"reason":"device_offline"}`; gdy urządzenie jest online, ale próba DSH kończy się niepowodzeniem, zwraca `503 {"reason":"dsh_offline"}`.

## Znane ograniczenia MVP

- Treść biznesowa jest szyfrowana end-to-end od Mobile do Companion; Relay nadal widzi konta, urządzenia, stan online, długość szyfrogramu i rozkład w czasie.
- Relay jest pojedynczą instancją; SQLite wymaga trwałego wolumenu Railway.
- Ciała żądań HTTP są buforowane przez bramę loopback Mobile przed szyfrowaniem; odpowiedzi HTTP i SSE są przesyłane strumieniowo fragmentami sealed frame, a Relay nie widzi tekstu jawnego.
- Tabela zdarzeń Relay przechowuje najwyżej 50 najnowszych wpisów metadanych `kind`, a `payload_json` jest zawsze pustym obiektem; treść zdarzeń biznesowych musi przechodzić przez sealed data plane.
