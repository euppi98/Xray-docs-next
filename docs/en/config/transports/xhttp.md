# XHTTP

XHTTP (also known internally as SplitHTTP) is a transport that carries proxy traffic over standard HTTP requests and responses. It was designed around one core insight: CDN and middleboxes that buffer uploads will still stream downloads. XHTTP exploits this asymmetry — uplink data is sent as ordinary HTTP requests that any middlebox can handle, while downlink data is streamed continuously, preserving full download speed.

Three transmission modes are available:

- **`packet-up`** — uplink split into sequential POST requests, downlink streamed via GET. Maximum CDN compatibility.
- **`stream-up`** — uplink sent as a single streaming POST (masked as gRPC), downlink streamed via GET. Best uplink efficiency.
- **`stream-one`** — a single bidirectional POST carries both directions. Simplest path, fewest requests.

XHTTP supports H1, H2, and H3 (QUIC), works behind real Nginx/Caddy reverse proxies and CDNs, includes built-in connection multiplexing (XMUX), and optionally splits uplink and downlink across entirely different network paths.

::: tip
Do not enable `mux.cool` when using XHTTP. The server rejects anything other than pure XUDP.

To verify the actual HTTP version, host, mode, and upstream/downstream split in use, set log level to `"info"`.
:::

::: warning
Cloudflare drops connections with no actual HTTP data transferred for 100 seconds. For long-lived proxy connections (e.g. SSH), configure application-level keepalive on the remote service (e.g. `ClientAliveInterval` in `sshd`).
:::

## XHTTPObject

`XHTTPObject` corresponds to the `xhttpSettings` item in [`StreamSettingsObject`](../transport.md#streamsettingsobject).

```jsonc
{
  // outbound example — most fields also apply to inbound
  "outbounds": [
    {
      "streamSettings": {
        "network": "xhttp",
        // [!code focus:start]
        "xhttpSettings": {
          "host": "example.com",
          "path": "/yourpath",
          "mode": "auto",
          "headers": {
            "User-Agent": "chrome"
          },
          "extra": {
            "xPaddingBytes": "100-1000",
            "xPaddingObfsMode": false,
            "xPaddingKey": "x_padding",
            "xPaddingHeader": "X-Padding",
            "xPaddingPlacement": "queryInHeader",
            "xPaddingMethod": "repeat-x",
            "uplinkHTTPMethod": "POST",
            "sessionIDPlacement": "path",
            "sessionIDKey": "",
            "sessionIDTable": "",
            "sessionIDLength": "0",
            "seqPlacement": "path",
            "seqKey": "",
            "uplinkDataPlacement": "auto",
            "uplinkDataKey": "",
            "uplinkChunkSize": "",
            "noGRPCHeader": false,
            "noSSEHeader": false,
            "scMaxEachPostBytes": "1000000",
            "scMinPostsIntervalMs": "30",
            "scMaxBufferedPosts": 30,
            "scStreamUpServerSecs": "20-80",
            "serverMaxHeaderBytes": 8192,
            "xmux": {
              "maxConcurrency": "16-32",
              "maxConnections": "0",
              "cMaxReuseTimes": "0",
              "hMaxRequestTimes": "600-900",
              "hMaxReusableSecs": "1800-3000",
              "hKeepAlivePeriod": 0
            },
            "downloadSettings": {
              "address": "",
              "port": 443,
              "network": "xhttp",
              "security": "tls",
              "tlsSettings": {},
              "xhttpSettings": {
                "path": "/yourpath"
              },
              "sockopt": {}
            }
          }
        }
        // [!code focus:end]
      }
    }
  ]
}
```

::: tip About `extra`
`extra` is a raw JSON object that carries all parameters except `host`, `path`, and `mode`. When `extra` is present, only those three top-level fields are read; everything else must live inside `extra`.

This separation exists so that share links and GUIs only need to expose the three user-facing fields, while the operator controls all advanced behaviour through `extra` embedded in the link.
:::

Throughout this page, **Client** and **Server** columns indicate which side of the connection actually reads and uses the field. A field marked Client-only is harmless (but ignored) if also present on the server side, and vice versa — except `uplinkDataKey`, which must be set identically on both sides (see below).

---

## Top-level fields

These three fields are read directly from `xhttpSettings`, even when `extra` is present.

---

> `host`: string

| Client | Server |
|--------|--------|
| sends as `Host` header | validates received value |

The HTTP host header value sent by the client.

Priority when sending (client): `host` > `serverName` (SNI) > `address`.

If set on the server, incoming requests must match this value; otherwise the connection is rejected. Leave empty on both sides unless you have a specific reason (traffic is already distinguished by `path`).

Cannot be placed inside `headers`.

Default: `""` (empty — derived from `serverName` or `address`)

---

> `path`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

The HTTP path prefix. Must match on client and server.

A trailing slash is appended automatically if absent. Session ID and sequence number are appended after this prefix by default (see `sessionIDPlacement`).

Default: `""` (equivalent to `"/"`)

---

> `mode`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

Selects the transmission mode.

| Value | Uplink | Downlink |
|-------|--------|----------|
| `"auto"` | chosen automatically (see below) | streamed GET |
| `"packet-up"` | sequential POST requests | streamed GET |
| `"stream-up"` | single streaming POST | streamed GET |
| `"stream-one"` | single bidirectional POST | same POST response |

**Client `"auto"` resolution** (exact order, from `Dial()`):
1. Default: `"packet-up"`
2. If REALITY is in use: `"stream-one"`
3. If REALITY **and** `downloadSettings` are both in use: `"stream-up"`

::: warning
Plain TLS (no REALITY) always resolves to `"packet-up"` under `"auto"` — including over H2. To get `"stream-up"` over TLS+H2 (e.g. to pass Cloudflare as masked gRPC), you must set `mode` to `"stream-up"` explicitly; `"auto"` will not pick it for you in that case.
:::

This is independent of the HTTP version negotiation (H1.1 / H2 / H3), which is decided separately based on TLS/REALITY/ALPN and does not affect which `mode` `"auto"` resolves to.

**Server behaviour:** the server does not pre-select a mode. For every incoming request it inspects the actual HTTP method and the presence of a session ID / sequence number to classify the request, then checks that classification against the server's configured `mode`:

- A POST/PUT/etc. request with a session ID but **no** sequence number is treated as `stream-up` (or `stream-one`, see below). If the server's `mode` is explicitly set and is neither `""`, `"auto"`, nor `"stream-up"`, the request is rejected with `400 Bad Request` ("stream-up mode is not allowed").
- A request with **no** session ID at all is treated as `stream-one`. If the server's `mode` is explicitly set and is none of `""`, `"auto"`, `"stream-one"`, `"stream-up"`, the request is rejected with `400 Bad Request` ("stream-one mode is not allowed").
- A POST/PUT/etc. request with both a session ID **and** a sequence number is treated as `packet-up`. If the server's `mode` is explicitly set and is neither `""`, `"auto"`, nor `"packet-up"`, the request is rejected with `400 Bad Request` ("packet-up mode is not allowed").
- A GET request with a sequence number is treated as a `packet-up` uplink fragment (same check as above); a GET without one is treated as the downlink stream and always accepted regardless of `mode`.

In short: leaving the server's `mode` as `"auto"` (or empty) accepts requests of any shape; setting it explicitly narrows which client modes are allowed to connect, with `"stream-up"` alone also accepting `"stream-one"` requests.

A number of fields below are only meaningful in one specific mode (noted individually). Setting them while running a different mode is harmless — they're simply not read by that code path.

::: warning
`downloadSettings` cannot be used with `"stream-one"` mode. Setting both will cause an error.
:::

::: warning
[Browser Dialer](https://xtls.github.io/config/features/browser_dialer.html) cannot perform bidirectional streaming — it can only issue separate, independent request/response pairs. This means `stream-up` and `stream-one` are unusable through Browser Dialer; only `packet-up` works. If a client may be using Browser Dialer, ensure `mode` resolves to (or is explicitly set to) `"packet-up"`.
:::

Default: `"auto"`

---

> `headers`: object

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Custom HTTP headers added to every outgoing request. Keys and values are plain strings.

Cannot contain a `"host"` key — use the top-level `host` field instead.

Example:
```json
"headers": {
  "User-Agent": "chrome"
}
```

Accepted values for `User-Agent`: `"chrome"`, `"firefox"`, `"edge"`, `"golang"`.

Default: `{}` (empty)

---

## Fields inside `extra`

All remaining parameters belong inside the `extra` object.

---

### Padding (`xPadding*`)

XHTTP randomises the length of HTTP request and response headers to avoid fixed-size fingerprints.

::: warning Writing is strict, reading is tolerant
There is an important asymmetry between how padding is **sent** and how it's **parsed**, and it's easy to misconfigure if you assume they're symmetric:

- **Sending** (what the client puts in a request, what the server puts in a response) always follows `xPaddingPlacement` exactly — there is no fallback on the write side.
- **Parsing incoming requests on the server** is a tolerant, ordered fallback chain that checks several possible locations regardless of what `xPaddingPlacement` is configured to. This means a server can often still validate padding sent by a client whose `xPaddingPlacement` doesn't exactly match the server's, as long as it lands somewhere the fallback chain checks.

Server-side request parsing order:
1. **If `xPaddingObfsMode` is `false`:** look at the `Referer` header first; extract `x_padding` from its query string. If `Referer` is empty, fall back to reading `x_padding` directly from the request's own URL query string. *(The server's own response padding in this mode always goes in the `X-Padding` header — that header is never read back from incoming requests; it's write-only, response-side.)*
2. **If `xPaddingObfsMode` is `true`:** check, in order — a cookie named `xPaddingKey`; then the header named `xPaddingHeader` (read directly if `xPaddingPlacement` is `"header"`, otherwise parsed as a URL and the `xPaddingKey` query parameter extracted from it, covering `"queryInHeader"`); then finally a query parameter named `xPaddingKey` on the request URL itself.
:::

---

> `xPaddingBytes`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✔ |

The target size of the padding value. Accepts a fixed number or a range string (`"100-1000"`). A new random value within the range is chosen for each request and each response, independently on each side.

::: tip
For `xPaddingMethod: "repeat-x"`, this is the literal number of characters generated — straightforward.

For `xPaddingMethod: "tokenish"`, this is **not** the raw string length. It's the target size in bytes *after* HPACK/QPACK Huffman compression (the encoding H2 and H3 apply to header values on the wire). Since `repeat-x`'s repeated `X`/`Z` characters happen to compress 1:1 in HPACK's static Huffman table, `xPaddingBytes` means the same thing on the wire either way — but the raw string Xray generates internally for `tokenish` will usually be longer than `xPaddingBytes`, because mixed Base62 characters compress better than repeated letters. Validation on the receiving end accounts for this difference (see `xPaddingMethod` below).
:::

Cannot be set to zero or a negative value.

Default: `"100-1000"`

---

> `xPaddingObfsMode`: boolean

| Client | Server |
|--------|--------|
| ✔ | ✔ |

Enables padding obfuscation, switching both the generation method and the placement of padding away from the recognisable legacy pattern. See the parsing-order box above for exactly what changes on each side.

Both sides should be configured consistently — while the server's fallback parsing is somewhat tolerant of mismatches (see above), the *response* side (server → client) has no such fallback, so if the client expects obfuscated response padding and the server isn't sending it in the expected place, validation can fail.

Enable this when a CDN is filtering traffic based on the distinctive `X...` padding pattern or the `Referer`-based legacy format.

Default: `false`

---

> `xPaddingMethod`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

How the padding value is generated, and consequently how it's validated on receipt. Only effective when `xPaddingObfsMode` is `true` — the legacy (non-obfs) pattern always uses an implicit `repeat-x`-equivalent value.

| Value | Generation | Validation |
|-------|-----------|-------------|
| `"repeat-x"` | A string of repeated `X`/`Z` characters (default, recognisable) | Raw string length must fall within `[from, to]` of `xPaddingBytes` |
| `"tokenish"` | A random Base62 string, length-adjusted so its HPACK Huffman-encoded size approximates `xPaddingBytes` (iteratively trimmed/padded, ±2 byte tolerance) | HPACK Huffman-encoded length must fall within `[from − 2, to + 2]` of `xPaddingBytes` (lower bound clamped to 0) |

`"tokenish"` looks like a real CDN cache key or auth token on the wire rather than an obvious padding marker, at the cost of slightly more CPU work to generate (iterative length adjustment, capped at 150 attempts) and slightly looser size validation (±2 bytes).

Default: `"repeat-x"`

---

> `xPaddingPlacement`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

Where the padding value is written when *sending* a request or response. Only effective when `xPaddingObfsMode` is `true`. This governs the write side strictly (see the asymmetry warning above) — the read side on the server is more tolerant.

| Value | Description |
|-------|-------------|
| `"queryInHeader"` | As a query parameter inside a URL string, which is itself placed as the value of an HTTP header (default) |
| `"header"` | Directly as a plain HTTP header value |
| `"query"` | As a URL query parameter on the request/response URL |
| `"cookie"` | As a cookie |

Default: `"queryInHeader"`

---

> `xPaddingHeader`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

The HTTP header name that carries the padding when `xPaddingPlacement` is `"header"` or `"queryInHeader"`, and `xPaddingObfsMode` is `true`.

::: tip
When `xPaddingObfsMode` is `false`, this field is unused — see the parsing-order box at the top of this section for what each side does instead (client sends via `Referer`; server's own response padding always uses a fixed `X-Padding` header, which is never read back from incoming requests).
:::

Default: `"X-Padding"`

---

> `xPaddingKey`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

The key name for the padding value when `xPaddingObfsMode` is `true`. Its meaning depends on `xPaddingPlacement`:
- `"queryInHeader"` or `"query"` — the URL query parameter name
- `"cookie"` — the cookie name
- `"header"` — not used (the header name itself is `xPaddingHeader`)

Without `xPaddingObfsMode`, the client always uses the fixed key `x_padding` inside its `Referer` pattern (or, as a fallback if `Referer` is absent on the receiving end, directly in the request's own query string).

Default: `"x_padding"`

---

### Uplink method (`uplinkHTTPMethod`)

---

> `uplinkHTTPMethod`: string

| Client | Server |
|--------|--------|
| ✔ | ✗ |

The HTTP method used for uplink (upload) requests. The server accepts whatever method the request arrives with — it doesn't validate or restrict the method on receipt — so this is purely a client-side choice.

| Value | Allowed in which `mode` | Notes |
|-------|--------------------------|-------|
| `"POST"` (default) | any | conventional choice, used for `packet-up` and `stream-up` |
| `"PUT"` | any | alternative if a CDN/WAF blocks `POST` specifically |
| `"PATCH"` | any | same use case as `PUT` |
| `"DELETE"` | any | same use case, despite the unusual semantics of sending a body on `DELETE` — Xray doesn't treat it specially |
| `"GET"` | **`packet-up` only** | rejected at config-build time in `auto`, `stream-up`, `stream-one` |

The config validator does not maintain an allowlist of methods — any string other than `"GET"` is accepted without restriction, in any mode. The *only* method-specific rule enforced is the `GET` → `packet-up` requirement above.

`"GET"` is a special case in practice, separate from the formal restriction: since a `GET` request conventionally carries no body, sending uplink data via plain `GET` only makes sense when paired with `uplinkDataPlacement` set to `"header"` or `"cookie"` — see the table under `uplinkDataPlacement` below for how that field's own restriction (also keyed on `mode`, not on this field) combines with each method.

::: tip Uplink connection reuse on HTTP/1.1 specifically
On H1.1, the downlink GET and the uplink POST/PUT/etc. behave differently with respect to connection reuse, since [XMUX](#xmux) doesn't apply here at all:

- **Downlink** opens a fresh TCP connection per GET request — no pooling.
- **Uplink**, in `packet-up` mode, maintains a small internal pool of raw, hand-rolled HTTP/1.1 connections specifically to avoid a full TCP+TLS handshake before every chunk. Each request is pre-serialised to raw bytes (so it can be safely retried on a different pooled connection if a write fails) and written directly to the socket. Responses to earlier requests on a reused connection aren't necessarily read immediately — they're drained lazily, just before the next write on that same connection. This pool isn't exposed through any `xhttpSettings` field; it's automatic and not user-configurable.
:::

Default: `"POST"`

---

### Session ID placement (`sessionID*`)

By default, XHTTP appends a UUID session ID and a sequence number to the URL path, producing paths like `/yourpath/507d9107-8354-4ccf-b174-96387334b3b0/2`. The following fields move and rename these values to blend in with real CDN traffic.

::: tip Session lifecycle on the server
The server creates a session entry the first time it sees a given session ID and tears it down once the connection is no longer needed:
- If the downlink GET request for a session never arrives, the server expires that session automatically after **30 seconds** and discards any buffered uplink data.
- Once the downlink GET does arrive, the session is considered fully connected and the 30-second reaper no longer applies — the session lives as long as that GET connection does, and is cleaned up when it closes.

This is why uplink and downlink must reach the server within a reasonably short window of each other; very long delays between opening the uplink and the downlink (e.g. across a slow or congested split path) can cause the session to be reaped before the GET connects.

A session ID's uplink is bound to **one** transport pattern for its lifetime: the first uplink request the server sees for a given session ID — whether it's a `stream-up` POST or a `packet-up` chunk — fixes how all subsequent uplink data for that session is expected to arrive. Sending a `stream-up`-style request and `packet-up`-style chunks under the same session ID concurrently is rejected by the server (surfacing as `409 Conflict`); this isn't something you'd normally hit through ordinary `mode` configuration, but it's worth knowing if you're debugging a custom client or proxy in front of XHTTP that might accidentally fork a session's uplink across two different request shapes.
:::

---

> `sessionIDPlacement`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

Where the session ID is placed. Must match between client and server, or the server won't be able to locate it.

| Value | Description |
|-------|-------------|
| `"path"` | Appended to the URL path (default, classic XHTTP behaviour) |
| `"query"` | As a URL query parameter |
| `"header"` | As an HTTP request header |
| `"cookie"` | As a cookie |

::: warning
If `sessionIDPlacement` is `"path"`, `seqPlacement` must also be `"path"`.
:::

Default: `"path"`

---

> `sessionIDKey`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

The key name for the session ID. Not used when `sessionIDPlacement` is `"path"`. Must match between client and server.

Default by placement:
- `"header"` → `"X-Session"`
- `"cookie"` or `"query"` → `"x_session"`

---

> `sessionIDTable`: string

| Client | Server |
|--------|--------|
| ✔ | ✗ |

The character set used to generate session IDs. Accepts a predefined name or a custom ASCII string. When empty, standard UUID format is used. This only affects how the client *generates* the ID — the server reads it back as an opaque string, so this field is client-only and need not match the server.

Predefined tables:

| Name | Characters |
|------|-----------|
| `"ALPHABET"` | `A-Z` (26 chars) |
| `"Alphabet"` | `A-Za-z` (52 chars) |
| `"BASE36"` | `0-9A-Z` (36 chars) |
| `"Base62"` | `0-9A-Za-z` (62 chars) |
| `"HEX"` | `0-9A-F` (16 chars) |
| `"alphabet"` | `a-z` (26 chars) |
| `"base36"` | `0-9a-z` (36 chars) |
| `"hex"` | `0-9a-f` (16 chars) |
| `"number"` | `0-9` (10 chars) |

Custom example: `"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_"`

::: warning
The total ID space (table size ^ session ID length) must exceed 2.1 billion possibilities. If `sessionIDTable` is set, `sessionIDLength` must also be set with a non-zero `from` value.
All characters in a custom table must be ASCII (< 0x80).
:::

Default: `""` (UUID format)

---

> `sessionIDLength`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

The length of the generated session ID, in characters. Accepts a fixed number or a range string (`"16-32"`). Only effective when `sessionIDTable` is set. Client-only for the same reason as `sessionIDTable`.

The `from` value must be greater than 0.

Default: `"0"` (UUID format used when `sessionIDTable` is empty)

---

### Sequence number placement (`seq*`)

---

> `seqPlacement`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

Where the packet sequence number is placed. Options and constraints are identical to `sessionIDPlacement`. Must match between client and server.

::: warning
If `sessionIDPlacement` is `"path"`, `seqPlacement` must also be `"path"`.
:::

Default: `"path"`

---

> `seqKey`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

The key name for the sequence number. Not used when `seqPlacement` is `"path"`. Must match between client and server.

Default by placement:
- `"header"` → `"X-Seq"`
- `"cookie"` or `"query"` → `"x_seq"`

---

### Uplink data placement (`uplinkData*`)

These fields are only used in `packet-up` mode, either when `uplinkHTTPMethod` is `"GET"` or when you need to move the uplink payload out of the request body entirely.

---

> `uplinkDataPlacement`: string

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Where uplink data fragments are placed in the outgoing request.

| Value | Allowed in which `mode` | Description |
|-------|--------------------------|-------------|
| `"auto"` (default) | any | Uses the request body — normal behaviour |
| `"body"` | any | Explicitly uses the request body |
| `"header"` | **`packet-up` only** | Data fragments placed in HTTP headers |
| `"cookie"` | **`packet-up` only** | Data fragments placed in cookies |

::: tip This is independent of `uplinkHTTPMethod`
The restriction above is gated on `mode`, not on which HTTP method you've chosen — `"header"`/`"cookie"` placement works with **any** `uplinkHTTPMethod` (`"POST"`, `"PUT"`, `"PATCH"`, `"DELETE"`, or `"GET"`), as long as `mode` is `"packet-up"`. It isn't a GET-only feature: e.g. `uplinkHTTPMethod: "PUT"` combined with `uplinkDataPlacement: "header"` is a perfectly valid `packet-up` configuration — useful if a CDN allows `PUT` but inspects/limits body size more aggressively than headers, for instance.

Full combination matrix for `mode: "packet-up"` (the only mode where `uplinkHTTPMethod` and `uplinkDataPlacement` both have meaningful, non-default choices):

| `uplinkHTTPMethod` | `uplinkDataPlacement: "body"`/`"auto"` | `uplinkDataPlacement: "header"`/`"cookie"` |
|---------------------|------------------------------------------|----------------------------------------------|
| `POST` / `PUT` / `PATCH` / `DELETE` | ✔ normal, conventional | ✔ valid — useful if the CDN inspects/limits the body specifically |
| `GET` | ✗ not practical — `GET` has no conventional body to carry data in | ✔ the only practical pairing for `GET` |

`"GET"` simply happens to be the method that's functionally *unusable* without `"header"` or `"cookie"`, since it has no conventional body to fall back on — but that's a consequence of how `GET` works, not a restriction this field enforces.
:::

The server doesn't need this field set to a specific value — it inspects the body, headers, and cookies of incoming requests and reassembles whatever it finds using `uplinkDataKey` as the lookup prefix, regardless of what the client's `uplinkDataPlacement` says.

::: warning
`"header"` and `"cookie"` require `mode` to be explicitly set to `"packet-up"` on both client and server.
:::

Default: `"auto"` (body)

---

> `uplinkDataKey`: string

| Client | Server |
|--------|--------|
| ✔ | ✔ |

Base name for the keys used to pass data fragments. The client automatically appends a numeric index (e.g. `X-Data-0`, `X-Data-1`). Not used when `uplinkDataPlacement` is `"body"` or `"auto"`.

::: warning
This field has no built-in default and **must be set explicitly on both client and server** when used — Xray has no way to guess a custom key name, and an empty/mismatched key means the server can't find the fragments.
:::

---

> `uplinkChunkSize`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Maximum size of each uplink data chunk, in bytes. Accepts a fixed number or a range string. Only effective when `uplinkDataPlacement` is not `"body"` or `"auto"`. `packet-up` mode only. Client-only — the server simply reassembles whatever chunks arrive.

Minimum value: 64 bytes.

Defaults by placement:
- `"cookie"` → `"2048-3072"` (2–3 KB)
- `"header"` → `"3000-4000"` (3–4 KB)

---

### Streaming behaviour

---

> `noGRPCHeader`: boolean

| Client | Server |
|--------|--------|
| ✔ | ✗ |

When `false` (default), XHTTP sets `Content-Type: application/grpc` on streaming uplink requests (`stream-up` and `stream-one` modes), masquerading as gRPC traffic. This is what allows H2 `stream-up` to pass through Cloudflare. Client-only — it controls what the client sends; the server accepts the request regardless of this header.

Set to `true` only if the gRPC header causes problems with your specific reverse proxy.

Default: `false`

---

> `noSSEHeader`: boolean

| Client | Server |
|--------|--------|
| ✗ | ✔ |

When `false` (default), the server sends `Content-Type: text/event-stream` on the downlink GET response, masquerading as Server-Sent Events (SSE). This improves compatibility with many middleboxes. Server-only — it controls what the server sends in response headers.

Set to `true` if SSE masquerading conflicts with your reverse proxy configuration.

Default: `false`

---

> `scMaxEachPostBytes`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✔ |

Maximum bytes of data in a single POST request body. `packet-up` mode only.

**On the client**, a new random value is drawn from the range for *each individual upload connection* and used as that connection's POST size ceiling. Outbound bytes are funnelled through an internal buffered pipe that automatically batches multiple small `Write()` calls together — the client does not wait to accumulate the full ceiling before sending; it flushes a POST as soon as enough data is queued, splitting into multiple POSTs only if more data arrives than fits in one. In other words, setting this very high does not introduce extra latency for small writes; it only raises the upper bound.

**On the server**, the upper bound of the configured range (the `to` value) is used as a single hard limit for all incoming requests, regardless of what range was configured. Any POST whose `Content-Length` — or, if chunked, actual body size — exceeds this limit is rejected with `413 Payload Too Large`, and the server logs a message explicitly telling you to adjust the server's value to be at least as large as the client's.

::: warning
Client and server values do not need to match, but the **server's value must be ≥ the client's value**, since the server enforces an upper bound on whatever the client sends. A server limit lower than the client's will cause uploads to fail once the client sends a POST larger than the server allows.
:::

Accepts a fixed number or a range string; using a range randomises the limit per request, reducing fingerprint.

Default: `"1000000"` (1 MB)

---

> `scMinPostsIntervalMs`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Minimum interval between consecutive POST requests for a single proxy sub-connection, in milliseconds. `packet-up` mode only. Client-only — it paces how fast the client issues POSTs; the server has no equivalent throttle.

This is not a flat `sleep(interval)` before every POST — it's a compensating wait that accounts for time already spent since the previous POST: `sleep(random(interval) - time_since_last_write)`. If sending and processing the previous chunk already took longer than the interval, the next POST fires immediately with no extra delay.

Default: `"30"` (30 ms)

---

> `scMaxBufferedPosts`: number

| Client | Server |
|--------|--------|
| ✗ | ✔ |

`packet-up` mode only. Server-only.

Uplink POST chunks can legitimately arrive out of sequence (different HTTP requests racing over different connections, or through different CDN edges). The server reassembles them using a small priority structure keyed by sequence number: a chunk that arrives exactly in order is delivered immediately and never touches the buffer; only chunks that arrive **ahead of** the one currently expected get held until the gap is filled.

This field caps how many such out-of-order chunks may be held at once. If a chunk arrives ahead of schedule and the number of already-held chunks exceeds this limit, the server tears down the connection rather than continuing to buffer ("packet queue is too large"), under the assumption that the client (or your application) will retry.

::: tip
In practice, gaps are usually small and short-lived — most chunks land in order or only slightly out of order, so this buffer rarely fills up under normal conditions. This is consistent with field reports of little to no perceivable speed difference between low (e.g. `1`) and default (`30`) values: the buffer mostly sits empty regardless, and only matters when your network path (or CDN) is reordering uplink chunks more aggressively than usual.
:::

Default: `30`

---

> `scStreamUpServerSecs`: string or number

| Client | Server |
|--------|--------|
| ✗ | ✔ |

How often the server sends keepalive padding bytes (a chunk of `'X'` bytes, sized per `xPaddingBytes`) to the client while the `stream-up` uplink connection is idle, in seconds. `stream-up` mode only. Server-only — the client has no corresponding setting.

This prevents Cloudflare (and similar CDNs) from dropping the connection after 100 seconds of no downlink data.

::: tip
The keepalive loop only starts if **both** of these are true:
- the range's upper bound (`to`) is greater than `0` (i.e. not disabled), **and**
- the request was recognisable as XHTTP padding — either it carried a non-empty legacy `Referer` padding marker, or `xPaddingObfsMode` was enabled and its padding was accepted.

In practice this means keepalive activates normally for standard XHTTP clients; it's a defensive check against starting a keepalive loop on a connection the server can't actually confirm is an XHTTP stream-up request.
:::

Set to `-1` to disable keepalive entirely. In that case the server will not send response headers until actual data arrives, matching pre-keepalive behaviour.

Default: `"20-80"` (random interval between 20 and 80 seconds)

---

> `serverMaxHeaderBytes`: number

| Client | Server |
|--------|--------|
| ✗ | ✔ |

Maximum size of HTTP request headers the server will accept, in bytes. Server-only — it's passed straight to Go's `http.Server.MaxHeaderBytes` and only has any effect there; setting it on the client does nothing (the JSON is accepted but unused).

Must be a non-negative value. A value of `0` or less uses the default.

Default: `8192` (8 KB)

---

### XMUX

XMUX controls how HTTP/2 and HTTP/3 connections are multiplexed and reused. It is read entirely by the client — the server has no XMUX configuration of its own, since connection reuse is a dialing decision. All range-typed values accept either a single number or a range string (e.g. `"16-32"`) unless noted.

::: warning
XMUX (`xmux.*`) only has an effect when the client is dialing over **H2 or H3**. Plain HTTP/1.1 connections (no TLS, or TLS with `alpn` forced to `"http/1.1"`) don't read any `xmux.*` field — H1.1 has no concept of multiplexed streams to manage. This does **not** mean H1.1 has no connection reuse at all, though: the uplink (`packet-up`) path maintains its own separate, internal connection pool of raw TCP sockets, used to avoid a full handshake per chunk. That pool isn't configurable via JSON and works independently of XMUX. The downlink (`GET`) side on H1.1 opens a fresh connection per request and does not pool at all.

XMUX is also bypassed entirely when [Browser Dialer](https://xtls.github.io/config/features/browser_dialer.html) is active (and REALITY is not in use) — connection management is delegated to the browser. Separately, and more significantly: **Browser Dialer cannot perform bidirectional streaming at all**, which means `stream-up` and `stream-one` are not usable through it — only `packet-up`'s pattern of separate request/response pairs works. If you're relying on Browser Dialer, make sure `mode` resolves to (or is explicitly set to) `"packet-up"`.
:::

::: tip
If the entire `xmux` object is omitted (all fields zero or absent), three defaults apply automatically:
- `maxConnections`: `6`
- `hMaxRequestTimes`: `"600-900"`
- `hMaxReusableSecs`: `"1800-3000"`

This produces smooth periodic connection rotation and is the recommended starting point.

If **any** single field is set, the automatic defaults for the other fields are disabled and you must configure them manually (except the mutually exclusive pair `maxConcurrency` / `maxConnections`).
:::

::: tip How connection selection actually works
All `xmux.*` settings (except `hKeepAlivePeriod`) are ranges. The client doesn't re-roll these ranges on every request — each value is rolled **once** when the connection pool for a given destination is first created, and stays fixed for as long as that pool exists.

For every outgoing request, the client picks (or opens) a connection like this:
1. Drop any pooled connection that's already closed, has used up `cMaxReuseTimes`, has used up `hMaxRequestTimes`, or has outlived `hMaxReusableSecs`.
2. If the pool is now empty, open a new connection.
3. If `maxConnections` is set and the pool has fewer connections than that limit, open a new connection (this is how the pool fills up to `maxConnections` before anything is reused).
4. Otherwise, build a shortlist of connections currently below the `maxConcurrency` threshold of simultaneously in-flight requests. If `maxConcurrency` is unset, every remaining connection qualifies.
5. If that shortlist is empty (everything is at capacity), open a new connection.
6. Otherwise, pick **one connection at random** (cryptographically random, not round-robin) from the shortlist and use it.

The practical effect: `maxConnections` controls how many connections the pool grows to before reuse kicks in; `maxConcurrency` controls how many in-flight requests are allowed to share one connection before another is needed. They describe the same pool from two different angles, which is why only one of them may be set at a time.
:::

---

> `xmux.maxConcurrency`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Maximum number of simultaneously in-flight requests allowed on a single connection before the pool looks for (or opens) another one. This number is rolled once per destination when the connection pool is created, not re-rolled per request.

Mutually exclusive with `maxConnections` — set only one.

For multi-threaded speed tests, set `"maxConcurrency": 1` so each parallel thread is forced onto its own connection.

Default: `0` (treated as unlimited; connection growth is governed by `maxConnections` instead)

---

> `xmux.maxConnections`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Maximum number of connections the pool is allowed to grow to for a given destination. Below this number, every new request that needs a connection opens a fresh one; once the pool reaches this size, requests are distributed across the existing connections at random instead (see the connection-selection box above). This number is rolled once per destination when the pool is created.

Mutually exclusive with `maxConcurrency`.

For a single persistent upstream carrying all proxy traffic: `"maxConnections": 1`.

Default: `6` (when all xmux fields are zero/omitted)

---

> `xmux.cMaxReuseTimes`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

How many total times a single connection may be selected from the pool (including its first use) before it's retired from reuse. Rolled once per connection when that connection is created. Once exhausted, the connection isn't immediately closed — it simply becomes ineligible for new requests and closes naturally once its in-flight requests finish.

Default: `0` (unlimited reuse)

---

> `xmux.hMaxRequestTimes`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Maximum number of individual HTTP requests that may be sent over a single connection before it's retired from the pool. Rolled once per connection when that connection is created; decremented on every actual request sent. Counted per logical HTTP request, which differs by mode:
- `stream-one` sends 1 request total for the whole proxy connection
- `stream-up` sends 2 (one upstream POST, one downstream GET)
- `packet-up` sends one request per chunk on the uplink, plus one GET for the downlink — so N, growing with how much data is transferred

Nginx defaults to 1000 requests per connection; this field lets you stay comfortably below that.

::: tip
Mid-stream rotation onto a new connection — triggered automatically once this limit (or `hMaxReusableSecs`) is hit — is only possible in `packet-up` mode, because only `packet-up` sends a continuous sequence of separate HTTP requests for one logical proxy connection that the client can reschedule onto a different pool entry between chunks. `stream-up` and `stream-one` each acquire one connection up front for the life of the stream and don't re-check this mid-flight; on those two, this field only affects which connection is initially selected to start the stream, plus the request-counting for subsequent independent streams that reuse the same pool.
:::

Default: `"600-900"` (when all xmux fields are zero/omitted), otherwise `0` (unlimited)

---

> `xmux.hMaxReusableSecs`: string or number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Maximum lifetime of a connection, in seconds, measured from the moment it's created (not from its last use, and not a sliding idle timeout). Rolled once per connection at creation time as an absolute cutoff timestamp; once that timestamp passes, the connection is no longer eligible for new requests and closes naturally once its in-flight requests finish.

Nginx defaults to roughly 1 hour per connection. The same `packet-up`-only mid-stream rotation behaviour described under `hMaxRequestTimes` applies here.

Default: `"1800-3000"` (when all xmux fields are zero/omitted), otherwise `0` (unlimited)

---

> `xmux.hKeepAlivePeriod`: number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Interval at which the client sends keepalive packets on an idle H2/H3 connection, in seconds.

- `0` — protocol default (45 s for Chrome-style H2 `ReadIdleTimeout`; for H3 this defers to quic-go's own default unless overridden elsewhere in `quicParams`)
- Negative (e.g. `-1`) — disable keepalive entirely (for H2 this clamps `ReadIdleTimeout` to `0`, which disables the HTTP/2 health-check ping; for H3 it simply does not set an explicit keepalive, falling back to the same protocol default as `0`)
- Any positive integer — explicit interval in seconds, applied to both H2's `ReadIdleTimeout` and H3's QUIC keepalive period

::: warning
This is the only XMUX field that does not accept a range string. Randomising keepalive intervals would itself be a fingerprint.
:::

Default: `0`

---

### Upstream/Downstream split (`downloadSettings`)

XHTTP can route uplink and downlink through completely different network paths — different IP addresses, protocols, CDNs, or even different countries. This is possible because the server correlates uplink and downlink connections using only the session ID in the path (or wherever `sessionIDPlacement` puts it).

`downloadSettings` is a nested `StreamSettingsObject` with two extra fields: `address` and `port`. It is entirely client-side: the server has no concept of "downloadSettings" — it simply receives an uplink connection and a separate downlink connection and links them by session ID, regardless of where each one physically came from. The block is a fully independent configuration; it does not inherit any settings from the parent uplink configuration.

::: warning
`downloadSettings` cannot be used with `"stream-one"` mode.
:::

---

> `downloadSettings.address`: string

| Client | Server |
|--------|--------|
| ✔ | ✗ |

The address of the downlink endpoint. Can be a domain name or IP address. Leave empty to use the same address as the uplink.

Default: `""` (same as uplink)

---

> `downloadSettings.port`: number

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Port of the downlink endpoint.

Default: `443`

---

> `downloadSettings.network`: string

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Must be `"xhttp"`. Required; cannot be omitted.

---

> `downloadSettings.security`: string

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Security layer for the downlink connection. Accepts `"tls"` or `"reality"`.

---

> `downloadSettings.tlsSettings` / `downloadSettings.realitySettings`

| Client | Server |
|--------|--------|
| ✔ | ✗ |

TLS or REALITY configuration for the downlink connection. Standard `TLSObject` or `REALITYObject`.

---

> `downloadSettings.xhttpSettings`

| Client | Server |
|--------|--------|
| ✔ | ✗ |

XHTTP configuration for the downlink connection. The `path` must match the server's `path`. All other fields are independent.

Even if `xmux` is not explicitly set here, the random values chosen for downlink XMUX ranges are independent from the uplink — connections rotate at different times on each side, making timing correlation harder.

---

> `downloadSettings.sockopt`

| Client | Server |
|--------|--------|
| ✔ | ✗ |

Socket options for the downlink connection. Independent of the uplink `sockopt`.

Exception: if the uplink `sockopt` sets `"penetrate": true`, the uplink `sockopt` overrides the downlink `sockopt`. This is useful for applying `mark` consistently.

---

## Mode selection summary

```
client mode = "auto"
│
├── REALITY in use?
│   ├── No  → "packet-up"
│   └── Yes
│       ├── downloadSettings present → "stream-up"
│       └── downloadSettings absent  → "stream-one"
│
└── (plain TLS or no security: always "packet-up", regardless of H1.1/H2/H3)
```

HTTP version (H1.1 / H2 / H3) is a separate decision driven by TLS/REALITY/ALPN — it does not influence which `mode` `"auto"` resolves to. To combine TLS+H2 with `stream-up` (e.g. for masked-gRPC passage through Cloudflare), set `mode` to `"stream-up"` explicitly.

## Quick-start checklist

1. Fill in `path`; all other fields have working defaults.
2. For QUIC H3 via CDN: set `alpn` to `["h3"]` on the client.
3. For IP selection (e.g. Cloudflare): set `address` to the IP, `serverName` (SNI) to the domain.
4. If Cloudflare blocks the connection: enable gRPC support in the CF dashboard.
5. If Nginx rejects the connection: change `proxy_pass` to `grpc_pass`.
6. If other CDNs or reverse proxies reject the connection: set `mode` to `"packet-up"` explicitly — it has the broadest compatibility.
7. For speed tests: set `"maxConcurrency": 1` inside `xmux`.
8. If a CDN filters on X-Padding: enable `xPaddingObfsMode` and set `xPaddingMethod` to `"tokenish"` on both client and server.
9. If POST requests are blocked: change `uplinkHTTPMethod` to `"PUT"`, `"PATCH"`, or `"DELETE"` on the client — any works in any mode.
10. If only GET is allowed: set `mode` to `"packet-up"` on both client and server, `uplinkHTTPMethod` to `"GET"` on the client, and `uplinkDataPlacement` to `"header"` or `"cookie"` on the client, with a matching `uplinkDataKey` on both sides.

## Full annotated example

The following shows a client outbound with upstream/downstream split. Server configuration uses only the parameters that actually take effect server-side — the rest are defaults.

```jsonc
// CLIENT outbound
{
  "outbounds": [
    {
      "protocol": "vless",
      "settings": { /* ... */ },
      "streamSettings": {
        "network": "xhttp",
        "security": "tls",
        "tlsSettings": {
          "serverName": "example.com",
          "alpn": ["h2"]
        },
        "xhttpSettings": {
          "host": "example.com",  // shared with downlink endpoint
          "path": "/api/data",    // must match server
          "mode": "stream-up",     // explicit — "auto" would give packet-up here (no REALITY)
          "extra": {
            // --- Padding obfuscation (CDN fingerprint bypass, must match server) ---
            "xPaddingBytes": "100-1000",
            "xPaddingObfsMode": true,
            "xPaddingMethod": "tokenish",
            "xPaddingPlacement": "queryInHeader",
            "xPaddingKey": "_dc",
            "xPaddingHeader": "X-Cache",

            // --- Session/seq moved out of path (must match server) ---
            "sessionIDPlacement": "header",
            "sessionIDKey": "X-Client-ID",
            "seqPlacement": "query",
            "seqKey": "chunk",

            // --- Client-only: generation format for the session ID above ---
            "sessionIDTable": "Base62",
            "sessionIDLength": "16-32",

            // --- XMUX (client-only) ---
            "xmux": {
              "maxConnections": "6",
              "hMaxRequestTimes": "600-900",
              "hMaxReusableSecs": "1800-3000",
              "hKeepAlivePeriod": 0
            },

            // --- Downlink via a different CDN edge IP (client-only) ---
            "downloadSettings": {
              "address": "2606:4700:4700::1111",  // Cloudflare IPv6 edge
              "port": 443,
              "network": "xhttp",
              "security": "tls",
              "tlsSettings": {
                "serverName": "example.com",
                "alpn": ["h3"]                    // QUIC H3 for downlink
              },
              "xhttpSettings": {
                "path": "/api/data"               // must match server
              }
            }
          }
        }
      }
    }
  ]
}
```

```jsonc
// SERVER inbound
{
  "inbounds": [
    {
      "protocol": "vless",
      "settings": { /* ... */ },
      "streamSettings": {
        "network": "xhttp",
        "security": "tls",
        "tlsSettings": { /* ... */ },
        "xhttpSettings": {
          "path": "/api/data",
          "mode": "auto",          // accepts the client's stream-up request as-is
          "extra": {
            // --- Must mirror the client's padding settings above ---
            "xPaddingObfsMode": true,
            "xPaddingMethod": "tokenish",
            "xPaddingPlacement": "queryInHeader",
            "xPaddingKey": "_dc",
            "xPaddingHeader": "X-Cache",

            // --- Must mirror the client's session/seq placement above ---
            "sessionIDPlacement": "header",
            "sessionIDKey": "X-Client-ID",
            "seqPlacement": "query",
            "seqKey": "chunk",

            // --- Server-only: stream-up keepalive ---
            "scStreamUpServerSecs": "20-80"
          }
        }
      }
    }
  ]
}
```

## Troubleshooting: server response codes

These are the exact HTTP status codes the server returns for common failure conditions, useful when reading reverse-proxy or browser network logs:

| Code | Cause |
|------|-------|
| `404 Not Found` | `host` didn't match the configured value, or the request path didn't start with the configured `path` |
| `400 Bad Request` | Padding value failed validation (wrong length or, in `xPaddingObfsMode`, wrong format); the request's implied mode (`stream-one`/`stream-up`/`packet-up`, inferred from session ID and sequence number) conflicted with an explicitly configured `mode`; malformed base64 in header/cookie-carried uplink data; malformed sequence number |
| `405 Method Not Allowed` | A request method other than `GET`/`POST`/`OPTIONS` arrived without matching any recognised pattern |
| `409 Conflict` | A duplicate or conflicting uplink request landed on a session ID that already has an active `stream-up` reader bound to it — for example, two `stream-up` POSTs for the same session ID arriving concurrently, or a `packet-up` chunk arriving on a session ID already locked to `stream-up` (see the session-lifecycle note above) |
| `413 Request Entity Too Large` / `413 Payload Too Large` | Uploaded data — whether via the request body, headers, or cookies — exceeded `scMaxEachPostBytes` as configured on the server |
| `500 Internal Server Error` | Failed to queue an uplink payload internally, or failed to parse an internal sequence number — generally indicates a bug or a corrupted request rather than a client misconfiguration |

`OPTIONS` requests are always answered with `200 OK` and the CORS headers described under [Browser Dialer](https://xtls.github.io/config/features/browser_dialer.html) compatibility, regardless of `mode` or session state.

## Parameter scope reference

| Parameter | Client | Server | Mode |
|-----------|--------|--------|------|
| `host` | ✔ | ✔ | all |
| `path` | ✔ | ✔ | all |
| `mode` | ✔ | ✔ | all |
| `headers` | ✔ | ✗ | all |
| `xPaddingBytes` | ✔ | ✔ | all |
| `xPaddingObfsMode` | ✔ | ✔ | all |
| `xPaddingMethod` | ✔ | ✔ | all |
| `xPaddingPlacement` | ✔ | ✔ | all |
| `xPaddingHeader` | ✔ | ✔ | all |
| `xPaddingKey` | ✔ | ✔ | all |
| `uplinkHTTPMethod` | ✔ | ✗ | all (`GET` requires `packet-up`) |
| `sessionIDPlacement` | ✔ | ✔ | all |
| `sessionIDKey` | ✔ | ✔ | all |
| `sessionIDTable` | ✔ | ✗ | all |
| `sessionIDLength` | ✔ | ✗ | all |
| `seqPlacement` | ✔ | ✔ | all |
| `seqKey` | ✔ | ✔ | all |
| `uplinkDataPlacement` | ✔ | ✗ | packet-up |
| `uplinkDataKey` | ✔ | ✔ | packet-up |
| `uplinkChunkSize` | ✔ | ✗ | packet-up |
| `noGRPCHeader` | ✔ | ✗ | stream-up, stream-one |
| `noSSEHeader` | ✗ | ✔ | all |
| `scMaxEachPostBytes` | ✔ | ✔ | packet-up |
| `scMinPostsIntervalMs` | ✔ | ✗ | packet-up |
| `scMaxBufferedPosts` | ✗ | ✔ | packet-up |
| `scStreamUpServerSecs` | ✗ | ✔ | stream-up |
| `serverMaxHeaderBytes` | ✗ | ✔ | all |
| `xmux.*` | ✔ | ✗ | all (H2/H3) |
| `downloadSettings` | ✔ | ✗ | packet-up, stream-up |
