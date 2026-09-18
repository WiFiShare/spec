# Areas and cells

WiFiShare uses [geohash](https://en.wikipedia.org/wiki/Geohash) prefixes for two
different jobs: splitting the world into downloadable areas, and blurring the
position of networks that no owner has verified.

## Prefix lengths

Cell sizes below are at the equator. A cell keeps its height everywhere, but
gets narrower towards the poles: the geohash-7 cell over Piazza Maggiore in
Bologna is 153 m tall and 109 m wide. Where one number is quoted, it is the
height, which is the larger of the two.

| Prefix length | Cell size (approx.) | Used for |
| --- | --- | --- |
| 2 | 1,250 × 625 km | First directory level in the dump |
| 3 | 156 × 156 km | Second directory level in the dump |
| 5 | 4.9 × 4.9 km | One area file, one API lookup |
| 7 | 153 × 153 m | Published position of a community-found network |

## Why an area is 5 characters

A `GET /v1/areas/{geohash5}` lookup tells the server only which 4.9 km cell the
caller cares about. That is coarse enough that the request is not a position
report, and small enough that the response stays a few tens of kilobytes.

Clients fetch the caller's own cell plus the eight neighbours, so a network just
across a cell boundary is still found.

## Why a published position is 7 characters

A community-found network is published as the centre of its geohash-7 cell, and
its BSSID is withheld. Someone reading the dump learns that free Wi-Fi exists on
roughly this block, which is what a person looking for Wi-Fi needs. They do not
learn which flat a router sits in.

Owner-verified networks are different: the owner asked to be found, so the
position is published to 5 decimal places (about 1 m) with the BSSIDs.

## Dump paths

An area file lives at `areas/<gh2>/<gh3>/<gh5>.geojson`, where each segment is
the geohash prefix of that length:

```
44.4938, 11.3426  (Piazza Maggiore, Bologna)
geohash7 = srbj45g
geohash5 = srbj4
path     = areas/sr/srb/srbj4.geojson
```

Each directory therefore holds at most 32 subdirectories or 1,024 files, which
stays under GitHub's limit of 3,000 entries per directory.
