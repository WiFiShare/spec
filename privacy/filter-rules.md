# Filter rules (R1–R12)

These rules run **on the device**, before anything is encrypted or queued. An
observation that any rule drops must never reach the network. The server applies
the same rules again on arrival and rejects a batch that breaks them, because a
client cannot be trusted to be a current version.

Test vectors: `privacy/fixtures/filter-cases.json`. Rule ids are stable; a rule
that is withdrawn keeps its number and is marked withdrawn.

## Drop rules

| Rule | Drop the observation when | Why |
| --- | --- | --- |
| **R1** | The SSID is absent, empty, or the network is hidden | A hidden network is not offering itself to anyone |
| **R2** | The SSID ends in `_nomap` or `_optout`, compared case-insensitively | The owner used the standard opt-out. Only the suffix counts: a network called `Bar Nomap` is not opted out |
| **R3** | The BSSID is locally administered, that is bit `0x02` of the first octet is set | Phone hotspots and randomized MACs live there, and they follow a person around |
| **R4** | The SSID matches the personal-hotspot patterns in `hotspot_patterns` below | A hotspot named after its owner is personal data, and it is not a place |
| **R5** | The security type is anything other than `open` or `owe` | Phase 1 collects only networks that can be shared. A password-protected network enters only through its owner, never through a scan. See D5 in the plan |
| **R6** | There is no location fix, or its accuracy is worse than 50 m | A position we cannot trust would blur the wrong block |
| **R7** | The observation is more than 7 days old in the queue | Stale data is not worth the privacy cost of sending it |
| **R8** | A field is present that is neither in `schemas/observation.schema.json` nor one of the raw platform fields these rules consume. Today that is exactly one field, `hidden`, which R1 reads | Belt and braces against a future contributor adding a device id |

`hotspot_patterns`, each anchored and matched case-insensitively against the
whole SSID:

```
^(iphone|ipad)([-_ ]?[0-9]{1,4})?$
^(iphone|ipad|galaxy|pixel) (di|de|van|von) .+
.+['`’]s (iphone|ipad|phone|galaxy|hotspot)$
^androidap[0-9]*$
^galaxy[-_ ]?[a-z]?[0-9]{1,3}([-_ ].*)?$
^(mifi|myhotspot|personal ?hotspot)([-_ 0-9].*)?$
```

The patterns are anchored on purpose. An unanchored `iphone` would also drop a
shop called `iPhone Repair Shop WiFi`, which is a place and not a person, and
the fixtures pin that case.

The patterns will never be complete. They are a floor, not a claim of safety.
When in doubt, add a pattern and a fixture: a missed venue costs a listing, a
missed hotspot costs somebody's privacy.

## Normalisation rules

Applied to every observation that survives the drop rules.

| Rule | Normalise |
| --- | --- |
| **R9** | BSSID to lowercase hexadecimal with colons: `b8:27:eb:1a:2b:3c` |
| **R10** | Latitude and longitude to 4 decimal places, about 11 m. Implementations must agree to within `1e-9`. Exact halfway values are undefined and the fixtures avoid them |
| **R11** | `observed_at` down to the hour, in UTC: `2026-09-17T14:00:00Z` |

## Batch rules

| Rule | Requirement |
| --- | --- |
| **R12** | At most 200 observations per batch. Within a batch, keep one observation per (BSSID, rounded position, hour): the one with the strongest RSSI. Shuffle the order before sending. Carry no batch id, sequence number, session id or timestamp of your own |

## What the server does with a broken rule

The two documents have to agree on this, so it is stated once, here.

- A violation of **R1–R11** in any observation rejects the **whole batch**. The
  server answers `400` with the first offending rule id in the problem
  document and stores nothing. One bad observation loses the batch, which is
  the safe direction: the client that sent it is out of date.
- **R12's dedupe is not a violation.** Duplicates are removed, the batch is
  accepted, and the `202` body reports how many went and names `R12`.
- A batch carrying a **linkage field** breaks R12 and is rejected whole.

## What never goes in a batch

Not a rule with a number, because it is not negotiable: no device identifier,
model, OS build, IP address, account, advertising id, installation id, list of
saved networks, or anything derived from them. The client field carries only the
platform name and the app's major.minor version, so that the server can reject a
version with a known filter bug.

## Why SSIDs are collected at all

[Opinion 13/2011](https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2011/wp185_en.pdf)
(section 5.3.2) says collecting SSIDs is excessive *for offering geolocation
services*, because a positioning database does not need the name. WiFiShare is
not a positioning database: the SSID is the thing a person selects to connect,
and the name is what tells them the network is a library rather than a flat. It
is collected because it is necessary for this purpose, and for no other.

## Running the fixtures

`privacy/fixtures/filter-cases.json` holds two lists.

- `observation_cases`: each case has an `input`, a raw platform scan result, and
  an `expect`. A kept case names the exact `normalized` output, down to the
  rounded coordinates and the truncated hour. A dropped case names the rule.
- `batch_cases`: each case gives the observations of one batch and the expected
  `count` after R12. The `generate` form builds `count` observations from
  `template`, each with a distinct BSSID under `bssid_prefix`.

Two things implementations must not get creative about:

1. Evaluate R7 against the `now` field in the fixture file, never the wall
   clock. Otherwise the suite rots.
2. Apply the drop rules in numeric order and report the first one that matches.
   Several rules often apply to the same network, and the fixtures pin which one
   is reported.
