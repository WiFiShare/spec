# Publish rules (P1–P8)

The filter rules decide what the server may receive. These decide what the
server may show through the API and write into the public dump. Everything else
stays in the database, or is deleted on the schedule in P7.

| Rule | Requirement |
| --- | --- |
| **P1** | Publish a community-found network only after at least 3 observations, from at least 3 distinct rate-limit buckets, on at least 2 distinct UTC days. One contributor walking past three times is not a confirmation |
| **P2** | For a community-found network publish: SSID, security type, captive-portal flag, the centre of its geohash-7 cell, `first_seen` and `last_seen` as dates, and the report counts. Never its BSSID |
| **P3** | For an owner-verified network publish: everything in P2 at full precision (5 decimal places), plus its BSSIDs, the venue name, and the credential only if the owner marked it public |
| **P4** | Never publish a network whose observations span more than 1 km. Mark it `mobile` and exclude it. A moving access point is a vehicle, a travel router or a phone, and publishing it would track its owner |
| **P5** | Never publish a network on the opt-out list. The list is stored as HMAC-SHA256 of the BSSID under a server-held pepper, so the server can block a network it is no longer allowed to store in the clear |
| **P6** | Unpublish a network that has not been observed for 12 months, or that reaches 3 `private` or `gone` reports, or whose owner withdraws it |
| **P7** | Delete raw observations within 7 days of the aggregation that consumed them. Delete rate-limit bucket hashes within 24 hours. Both are enforced by a scheduled job, not by hand |
| **P8** | A network's public id is random and carries no derivation from its BSSID, SSID or position. Ids are not reused after P6 |

## Ordering

The aggregation job applies the rules in this order, and stops at the first that
refuses:

```
opt-out (P5) -> mobile check (P4) -> threshold (P1) -> precision (P2/P3) -> write
```

## Consequences worth stating

- The dump cannot be used to look up where a specific router is, because
  community-found BSSIDs never appear in it and the API offers no BSSID lookup.
- An owner who verifies a network is choosing to be precisely located. The
  claim flow must say so in those words before they confirm.
- P1 means a new free network takes at least two days to appear. That is a
  deliberate trade against a single contributor being able to place a network
  anywhere.
