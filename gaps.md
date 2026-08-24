# wsbaga — language & protocol gaps

Probe log from apps-roadmap №3 (WebSocket). Same shape as kvbaga/pgbaga.

## W1 — ~~serial connections (K1 again)~~ — CLOSED

**Symptom.** `ws_serve` accepts one connection at a time; a slow client
blocks the rest. `go()` still carries only `i64`, so a shared listener
state can't move to workers without per-connection spawning.

**Closed (2026-08-04, chatbaga).** `std/net/poll.baga` + `chat_serve`
watch the listener and every client fd on one thread; residual buffers are
`Map<i64, bytes>`. The echo path (`ws_serve`) remains serial by design —
multi-connection products use the poll loop. kvbaga can adopt the same
pattern without further language changes.

## W2 — fragmented message reassembly

**Status 2026-08-04: closed.** `ws_read_message` reassembles continuation
frames into one message (original opcode, full payload); control frames
are delivered immediately even mid-message (RFC 6455 §5.4); a lone
continuation or a new data frame mid-message is a protocol violation.
The accumulator keeps the 64 MiB sanity cap. `ws_handle_conn` (echo
server) uses it. Live tests in `tests/ws_test.baga`: fragmented echo,
interleaved ping, lone-continuation close. `ws_read_frame` stays the
frame-level API (chatbaga uses it directly).

## W3 — sha1 exists now, but std/crypto has no HMAC-SHA1

**Symptom.** Only `hmac_sha256` exists. No protocol in the current stack
needs HMAC-SHA1, so it wasn't added.

**Severity.** Low until a protocol demands it (old OAuth 1.0a, TOTP-SHA1).

**Verdict.** Add when needed — `hmac_sha256` is the template.

## W4 — close status / RFC handshake strictness — **closed 2026-08-24**

**Shipped.** `ws_close_code` / `ws_close_reason` / `ws_close_payload` /
`ws_send_close_code`. Handshake requires `Connection: Upgrade` and
`Sec-WebSocket-Version: 13`. `ws_try_parse_frame` rejects RSV ≠ 0,
reserved opcodes, fragmented or >125-byte control frames, 64-bit length
with MSB set, and non-minimal 126/127 encodings.

## Closed / fine

- **SHA-1 in std** (the gap this app was built around): `std/crypto/sha1.baga`,
  RFC 3174 vectors + RFC 6455 accept-key vector in `tests/std/sha1_test.baga`.
- Frame codec over the `bytes` API is clean; masking is one XOR pass.
- Handshake parse reused httpdbaga (`http_read_request`) — no duplication.
- Real interop: `wscat` (Node.js) echoes UTF-8 text and 900-byte payloads.
- 101 / client GET built with `bufbaga` (G1).
