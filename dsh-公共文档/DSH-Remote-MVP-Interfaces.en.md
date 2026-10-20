# DSH Remote MVP Cross-Platform Interfaces

[Chinese](DSH-远程MVP-跨端接口.md) | [English](DSH-Remote-MVP-Interfaces.en.md) | [Polski](DSH-Remote-MVP-Interfaces.pl.md)

This is a reference translation of the Chinese summary.

This file is the shared contract for the mobile client, computer-side Companion, and Relay. Business payloads are not written to Relay logs.

## Address conventions

- Local development Relay: `http://127.0.0.1:8787`
- Production Relay: a Railway HTTPS domain, for example `https://dsh-relay.example`
- Companion device connection: replace the Relay address scheme with `ws` / `wss`, using path `/device`
- WebView entry: the Mobile device's local `http://127.0.0.1:<ephemeral>/{path}`; Relay provides only the `/client-tunnel` ciphertext WSS

## Authentication

`POST /auth/register` `{ email, password }`, with a password of at least 8 characters, returns `{ accessToken, refreshToken }`.

`POST /auth/login` is the same as above. `POST /auth/refresh` `{ refreshToken }` rotates the refresh token. The access token uses `Authorization: Bearer <token>` and is valid for 7 days; the refresh token is valid for 30 days.

## Pairing

1. Companion calls `POST /pair/session` and receives `{ code, deviceId, deviceSecret, expiresAt }`. deviceSecret is stored only locally on the computer.
2. Companion locally generates a 32-byte master key and encodes `{"v":2,"relay":"...","code":"...","e2eeKey":"<base64url>"}` as a QR code; the master key is not uploaded to Relay, and production pairing does not accept a six-digit code alone.
3. The phone calls `POST /pair/claim` `{ code }` with the user JWT.
4. Companion calls `POST /pair/confirm` `{ deviceId, deviceSecret, deviceName }` every 2 seconds. Before the code is claimed, it returns `202 { status: "pending" }`; after the claim, it returns `{ deviceToken }`.
5. Companion subsequently uses only `deviceId + deviceToken` to establish the `/device` WSS connection.

Pairing codes are valid for 5 minutes and single-use; deviceToken becomes invalid immediately after the device is unbound.

## Device management

- `GET /devices`: returns `{ devices: [{ id, name, online, dshStatus, lastSeenAt }] }`
- `PATCH /devices/:id` `{ name }`
- `DELETE /devices/:id`
- `POST /device-management/:id/unbind`: Companion uses `Authorization: Device <deviceToken>` locally on the computer to remove the computer-wide pairing; Relay revokes the device token, ends access sessions, and disconnects the current connection
- `POST /web-ticket` `{ deviceId }`: returns a single-use ticket valid for 60 seconds

Mobile uses the single-use ticket in `Authorization: WebTicket` to connect to `/client-tunnel`, then starts a local HTTP/WS gateway bound only to loopback. WebView accesses only the local gateway; HTTP, SSE, and WebSocket are forwarded as encrypted inner envelopes. A ticket can be used only once and must not appear in a URL.

## Tunnel envelope

Companion, Mobile, and Relay outer frames all remain v1 JSON; business envelopes are encrypted according to `Tunnel-E2EE-v1.md`:

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

The first frame after the device connection is established must be:

```json
{"v":1,"type":"auth","id":"...","ts":0,"payload":{"deviceId":"dev_x","deviceToken":"..."}}
```

The decrypted HTTP request payload is `{ method, path, headers, bodyB64 }`; the response payload is `{ status?, headers?, bodyB64, seq, final }`. Responses are chunked with increasing `seq`, supporting SSE and long responses. Companion rewrites `Host`, `Origin`, and `Referer` to the local DSH address. When Mobile's local browser aborts a request, it sends encrypted `http_close`, and Companion cancels the local request.

WebSocket uses the same encrypted inner `channel`: the Mobile local gateway sends `ws_open`, both sides exchange `ws_frame`, and either side sends `ws_close` when closing. Relay sees only sealed frames.

Status frame: `status { dsh: "online" | "offline" }`. Relay returns `503 {"reason":"device_offline"}` when there is no device connection; it returns `503 {"reason":"dsh_offline"}` when the device is online but the DSH probe fails.

## Known MVP limitations

- Business content uses end-to-end encryption from Mobile to Companion; Relay can still see accounts, devices, online state, ciphertext length, and timing.
- Relay is a single instance; SQLite requires a Railway persistent volume.
- HTTP request bodies are buffered by the Mobile loopback gateway before encryption; HTTP responses and SSE are streamed in sealed-frame chunks, and Relay does not see plaintext.
- The Relay events table retains at most the latest 50 `kind` metadata entries, with `payload_json` fixed to an empty object; business event bodies must travel through the sealed data plane.
