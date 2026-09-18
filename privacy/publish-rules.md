# Publish rules (P1–P8)

The filter rules decide what the server may receive. These decide what the
server may show through the API and write into the public dump. Everything else
stays in the database, or is deleted on the schedule in P7.

| Rule | Requirement |
| --- | --- |
| **P1** | Publish a community-found network only after at least 3 observations, from at least 3 distinct rate-limit buckets, on at least 2 distinct UTC days. One contributor walking past three times is not a confirmation. The evidence is the tally in P9, not the bucket values |
| **P2** | For a community-found network publish: SSID, security type, captive-portal flag, the centre of its geohash-7 cell, `first_seen` and `last_seen` as dates, and the report counts. Never its BSSID |
| **P3** | For an owner-verified network publish: everything in P2 at full precision (5 decimal places), plus its BSSIDs, the venue name, and the credential only if the owner marked it public |
| **P4** | Never publish a network whose observations span more than 1 km. Mark it `mobile` and exclude it. A moving access point is a vehicle, a travel router or a phone, and publishing it would track its owner |
| **P5** | Never publish a network on the opt-out list, and delete its tally with everything else. The list is stored as HMAC-SHA256 of the BSSID under a server-held pepper, so the server can block a network it is no longer allowed to store in the clear. The check runs before anything is stored, which is stricter than the order below and is deliberate |
| **P6** | Unpublish a network not observed for 365 days, or that reaches 3 `private` or `gone` reports, or whose owner withdraws it |
| **P7** | Delete raw observations within 24 hours of the run that consumed them. Delete the rate-limit counters and the daily salt within 24 hours. Enforced by a scheduled job, not by hand |
| **P8** | A network's public id is random and carries no derivation from its BSSID, SSID or position. Ids are not reused after P6 |
| **P9** | Consume a UTC day only once it is **closed**: 8 days after it ended, past the 7-day window R7 lets a client upload within, so no observation for that day can still arrive. Record on the network only the day, the number of distinct buckets seen in it, and the observation count. Never keep the bucket values. A closed day is tallied once and never revised. P1 reads this tally, so the threshold outlives the raw data |
| **P10** | Compact tally rows older than 90 days into one row holding the summed observation count, summed bucket count and number of days. P1 reads those sums, so nothing about the threshold changes, and the per-day record of when a network was seen goes |

## How P1 and P7 fit together

P1 counts contributors across days; P7 deletes the evidence after a week. P9 is
what reconciles them, and it costs nothing, because of how the salt works.

The rate-limit salt rotates every UTC day and is deleted within 24 hours. A
bucket value is therefore **already day-scoped**: the same person contributing
on Monday and on Tuesday produces two unrelated values, and nobody, including
us, can tell they were the same person. "Distinct buckets" has only ever meant
distinct contributor-days.

So counting distinct buckets **within** each completed day and adding the
counts up gives the same number as counting them across the whole history,
while keeping none of the values. That is P9. What survives on a network is a
list like "2026-09-14: 2, 2026-09-15: 1", which says two people saw this
network on Monday and one on Tuesday, and nothing whatever about who they were.

Two consequences worth stating:

- A network whose third contributor arrives months after the first two **is**
  published. The tally waited for them.
- Aggregation processes only **completed** UTC days, so a day cannot be counted
  twice by two runs. Publication is therefore up to a day behind the last
  observation, which is immaterial next to P1's own two-day requirement.

## What P1 protects against, and what it does not

P1 is a noise filter. It stops a single stray sighting becoming a published
network: a bad fix, a network glimpsed from a passing train, one person's one
mistake.

**P1 does not stop a determined contributor, and cannot.** Because the salt
rotates daily, one person contributing on three days produces three distinct
buckets, and nothing here can tell that from three people. Three networks in an
afternoon (home line, mobile data, a VPN) does the same. The only way to catch
it would be to keep something that links a contributor across days, which is
exactly what this project refuses to keep.

So do not read P1 as an anti-abuse defence, and do not "strengthen" it by
storing contributor identity for longer: that trade is not on the table. What
defends the data against a determined actor is moderation, the reports in P6,
and, if the project ever needs it, anonymous tokens that prove a submission is
distinct without saying whose it is.

### The lag this creates

A network first seen today can be published nine days later at the earliest:
eight for its day to close under P9, and P1 still wants a second day. That is
the price of counting each day exactly once without keeping what would let us
revise it.

An app should therefore show a contributor their own pending sightings
locally. The map cannot show them for over a week, and a contributor who sees
nothing at all will reasonably conclude the app is broken.

## Choices these rules leave open, and how the server makes them

Two numbers are visible in the published output, so they are pinned here rather
than left to each implementation.

| Choice | Rule |
| --- | --- |
| RSSI weighting for the centroid | `weight = rssi + 101`, so −100 dBm weighs 1 and 0 dBm weighs 101. Monotonic and never zero |
| How P4 measures "span" | The diagonal of a bounding box kept cumulatively on the network, not the maximum pairwise distance. Raw observations are deleted at 7 days, so a running box is all that survives. It is never smaller than the true span, so the check errs towards not publishing |

## Ordering

The aggregation job applies the rules in this order, and stops at the first that
refuses:

```
consume a completed UTC day -> tally it (P9) -> opt-out (P5) -> mobile (P4)
  -> threshold (P1) -> precision (P2/P3) -> write
```

## Consequences worth stating

- The dump cannot be used to look up where a specific router is, because
  community-found BSSIDs never appear in it and the API offers no BSSID lookup.
- An owner who verifies a network is choosing to be precisely located. The
  claim flow must say so in those words before they confirm.
- P1 means a new free network takes at least two days to appear. That is a
  deliberate trade against a single contributor being able to place a network
  anywhere.
