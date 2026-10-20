# Protokół tunelu DSH Relay v1

[English](Tunnel-Protocol-v1.md) | [Polski](Tunnel-Protocol-v1.pl.md)

Dokument angielski [Tunnel-Protocol-v1.md](Tunnel-Protocol-v1.md) pozostaje źródłem nadrzędnym; to tłumaczenie referencyjne.

Companion utrzymuje jedno wychodzące połączenie WebSocket do `wss://<relay-host>/device`. Relay kieruje żądania przeglądarki adresowane do `/s/:deviceId/*` przez to połączenie. Dane biznesowe są przesyłane wyłącznie w pamięci i nigdy nie są zapisywane w logach ani bazie danych.

## Koperta

Każda ramka to JSON w UTF-8:

```json
{
  "v": 1,
  "type": "http_req",
  "channel": "ch_123",
  "id": "uuid",
  "ts": 1755500000000,
  "payload": {}
}
```

`channel` jest wymagany w komunikatach tunelu i nie występuje w komunikatach sterujących. `id` jest unikatowy dla każdego nadawcy. Nieznane pola rozszerzające muszą być ignorowane.

DSH Remote 0.1.4 używa tej koperty jako zaszyfrowanego wewnętrznego formatu płaszczyzny danych zdefiniowanego w [Tunnel-E2EE-v1.md](Tunnel-E2EE-v1.md). Relay widzi jedynie zewnętrzne typy komunikatów `client_hello`, `server_hello`, `sealed`, `client_close` i `device_close`. NIE MOŻE otrzymywać wewnętrznego kanału ani danych HTTP/WS w postaci jawnej. Starsze bezpośrednie przekazywanie z przeglądarki jest wyłączone w środowisku produkcyjnym.

## Cykl życia połączenia

1. Companion otwiera WSS i natychmiast wysyła `auth`.
2. Relay weryfikuje `deviceId` i `deviceToken`, odpowiada `auth_ok` i oznacza urządzenie jako online.
3. Companion wysyła `ping` co 25 sekund; Relay zwraca `pong`.
4. Relay, który nie otrzyma żadnej ramki przez 60 sekund, zamyka gniazdo.
5. Companion ponawia połączenie z opóźnieniami `1s, 2s, 5s, 10s, 30s`, ponownie się uwierzytelnia i wysyła aktualny stan DSH.

Ramka uwierzytelniania:

```json
{"v":1,"type":"auth","id":"uuid","ts":1755500000000,"payload":{"deviceId":"dev_xxx","deviceToken":"..."}}
```

Odpowiedź przy niepowodzeniu: `{"v":1,"type":"auth_fail","id":"uuid","ts":0,"payload":{"reason":"invalid_device_token"}}`. Odpowiedź po sukcesie używa `auth_ok`.

## Status i zdarzenia

Companion wysyła `status` przy każdej zmianie lokalnego stanu DSH:

```json
{"v":1,"type":"status","id":"uuid","ts":1755500000000,"payload":{"dsh":"online"}}
```

Relay trwale zapisuje wyłącznie metadane statusu. `event` jest zarezerwowany do przyszłego zbierania zdarzeń DSH i zapisuje najwyżej 50 rekordów metadanych zdarzeń na urządzenie. Nie wpływa na dostępność zdalnego interfejsu v1.

## Przekazywanie HTTP

Od Relay do Companion:

```json
{
  "v":1,
  "type":"http_req",
  "channel":"ch_a1b2",
  "id":"uuid",
  "ts":1755500000000,
  "payload":{"method":"POST","path":"/api/tasks","headers":{"content-type":"application/json"},"bodyB64":"eyJwcm9tcHQiOiJoZWxsbyJ9"}
}
```

Ścieżka to sufiks po `/s/:deviceId` dla żądania bootstrap albo pierwotna bezwzględna ścieżka DSH po autoryzacji ciasteczkiem. Zawsze zaczyna się od `/` i nie może być interpretowana jako URL. Companion przekazuje ją wyłącznie do `http://127.0.0.1:<dshPort>`, przepisuje `Host`, `Origin` i `Referer` na ten lokalny host i port oraz usuwa przekazane nagłówki identyfikacji proxy.

Odpowiedzi są strumieniowane jako początkowa ramka nagłówków, zero lub więcej ramek ciała i jedna ramka końcowa:

```json
{"v":1,"type":"http_res","channel":"ch_a1b2","id":"uuid","ts":1755500000000,"payload":{"status":200,"headers":{"content-type":"text/event-stream"},"bodyB64":"","seq":0,"final":false}}
```

Każda kolejna ramka zwiększa `seq`. `bodyB64` może być puste. `final:false` pozostawia odpowiedź Relay otwartą; dokładnie jedna ramka `final:true` ją kończy. Pozwala to przesyłać odpowiedzi startowe DSH i SSE bez buforowania całej odpowiedzi.

Jeśli przeglądarka zamknie trwające żądanie, Relay wysyła `http_close` na tym samym kanale. Companion niszczy odpowiadające mu żądanie loopback i zwalnia je. Spóźnione `http_res` dla zamkniętego kanału jest ignorowane.

## Przekazywanie WebSocket

1. Relay wysyła `ws_open {channel, path, headers}`.
2. Companion otwiera lokalny WebSocket DSH i odpowiada `ws_open_ok`; w razie niepowodzenia używa `ws_close`.
3. Każda strona wysyła ramki bezpieczne dla danych binarnych przez `ws_frame {channel, dataB64, opcode}`, gdzie `opcode` wynosi `1` dla tekstu lub `2` dla danych binarnych.
4. Każda strona zamyka połączenie przez `ws_close {channel, code, reason}`.

Relay mapuje uaktualnienie połączenia przeglądarki do WebSocket na jeden kanał. Przed utworzeniem kanału musi wymusić tę samą autoryzację ciasteczkiem WebView. Companion musi zamknąć wszystkie lokalne kanały po zamknięciu gniazda Relay.

## Granice zasobów

Egzekwowanie limitów zasobów nie zmienia koperty v1. Publiczny Relay MVP domyślnie ogranicza przychodzącą ramkę WebSocket do 4 MiB, przekazywane żądanie HTTP do 2 MiB, a całą odpowiedź HTTP do 32 MiB. Akceptuje najwyżej 32 kanały tunelu HTTP i 16 kanałów WebSocket na urządzenie, z zastrzeżeniem niższych globalnych limitów pojemności. Zbyt duża ramka WebSocket zamyka tylko to połączenie z kodem `1009`; nie może zakończyć procesu Relay.

Gdy kanał HTTP przekroczy limit odpowiedzi, Relay wysyła `http_close` do Companion i kończy odpowiedź przeglądarki. Companion musi zwolnić żądanie loopback po `http_close`. Odpowiedzi dla kanału należącego do innego uwierzytelnionego urządzenia są ignorowane.
