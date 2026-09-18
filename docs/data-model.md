# Data model

Seven entities. Three travel from a phone to the server, three travel back out
to anyone, and one is the venue owner's side of the deal.

```
Observation  --+
               |  200 max, shuffled
Batch  --------+--> Envelope (HPKE) --> POST /v1/batches
                                            |
                                     aggregation job
                                            |
                                            v
                               PublishedNetwork --> GET /v1/areas/{gh5}
                                                --> areas/**.geojson in the dump

Report   --> POST /v1/networks/{id}/reports
OptOut   --> POST /v1/optout
Claim    --> POST /v1/claims
```

## Observation

One sighting of one network, after the filter rules. Schema:
`schemas/observation.schema.json`.

| Field | Type | Notes |
| --- | --- | --- |
| `ssid` | string, 1–32 bytes UTF-8 | Required. Hidden networks are dropped by R1 |
| `bssid` | string | Lowercase, colon-separated (R9) |
| `security` | `open` \| `owe` | Phase 1 collects nothing else (R5) |
| `captive_portal` | `unknown` \| `detected` \| `none` | `detected` only when the device actually met a portal |
| `lat`, `lon` | number | 4 decimal places (R10) |
| `accuracy_m` | number, ≤ 50 | Of the device's own position fix (R6) |
| `rssi` | integer, −100…0 | Used to weight the position estimate |
| `frequency_mhz` | integer, optional | 2.4 or 5 GHz channel centre |
| `observed_at` | string | RFC 3339, truncated to the hour, UTC (R11) |
| `source` | `scan` \| `connected` | iOS can only ever send `connected` |

## Batch

What gets encrypted. Schema: `schemas/batch.schema.json`. Carries no id and no
timestamp of its own, so two batches from one phone cannot be linked (R12).

```json
{
  "schema": "wifishare.batch/1",
  "client": { "platform": "android", "version": "1.0" },
  "observations": [ { "...": "..." } ]
}
```

## Envelope

What is posted. Schema: `schemas/envelope.schema.json`. HPKE, RFC 9180, base
mode, suite X25519-SHA256-ChaCha20Poly1305, with `info` set to the batch schema
id and the key id as additional authenticated data.

```json
{
  "schema": "wifishare.envelope/1",
  "key_id": "2026q4",
  "suite": "x25519-sha256-chacha20poly1305",
  "enc": "<base64url, 32 bytes>",
  "ct": "<base64url>"
}
```

The server publishes current public keys at `GET /v1/keys`. Apps pin the keys
they shipped with and accept a rotation only over TLS from that endpoint.

## PublishedNetwork

The only shape the outside world sees. Schema:
`schemas/network.schema.json`. Fields marked *verified only* appear when
`verification` is `owner-verified`.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string, 12 chars base32 | Random, unrelated to the BSSID (P8) |
| `ssid` | string | |
| `security` | `open` \| `owe` \| `shared` | `shared` means a password the owner published |
| `captive_portal` | `unknown` \| `detected` \| `none` | |
| `verification` | `community` \| `owner-verified` | |
| `lat`, `lon` | number | Centre of the geohash-7 cell, or exact for verified |
| `precision_m` | integer | 150 for community, 10 for verified |
| `cell` | string | The geohash-7 cell |
| `first_seen`, `last_seen` | string, `YYYY-MM-DD` | Day precision only |
| `reports` | object | `{ "works": n, "fails": n }` |
| `bssids` | array of string | *Verified only* |
| `venue` | object | *Verified only*: `{ "name": "...", "kind": "cafe" }` |
| `credential` | object | *Verified only*, and only if published: `{ "type": "wpa2-psk", "secret": "...", "note": "..." }` |

## Area file

A GeoJSON `FeatureCollection` of published networks for one geohash-5 cell, with
foreign members naming the cell and the license. Schema:
`schemas/area.schema.json`. The API returns the same document at
`GET /v1/areas/{geohash5}`, so a client has one parser for both sources.

## Report, OptOut, Claim

| Entity | Purpose | Notes |
| --- | --- | --- |
| `Report` | A user says a network `works`, `fails`, is `not_free`, is `private` or is `gone` | Anonymous, rate-limited |
| `OptOut` | Remove a network from the database and the dump | Takes a BSSID, or an SSID plus a cell. Not cryptographically proven: removal is low-risk, and an owner who wants back in can claim the network |
| `Claim` | A venue proves it controls a network and shares access | Proof is an SSID challenge: the owner appends a short code to the SSID, and the app confirms seeing it |
