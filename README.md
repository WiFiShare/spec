# WiFiShare spec

The contract between the WiFiShare apps, the API and the public dump. If an
implementation and this repo disagree, this repo is right.

Version: `1` (draft). Nothing here is frozen until the first public beta.

## What is in here

| Path | What it defines |
| --- | --- |
| `docs/data-model.md` | The entities: observation, batch, envelope, published network, report, opt-out, claim |
| `docs/geohash.md` | Why areas are geohash cells, and which prefix length means what |
| `privacy/filter-rules.md` | Rules R1–R12: what an app may upload, and what it must drop on the device |
| `privacy/publish-rules.md` | Rules P1–P8: what the server may publish, and at what precision |
| `privacy/fixtures/filter-cases.json` | Test vectors for R1–R12. Every client and the server must pass these |
| `schemas/*.json` | JSON Schema (2020-12) for each entity |
| `openapi.yaml` | The HTTP API, version 1 |

## The three rules everything else serves

1. Collect only what helps someone connect.
2. Publish less than is collected.
3. Keep nothing that ties an observation to a person.

## Using the fixtures

`privacy/fixtures/filter-cases.json` is the acceptance test for the on-device
privacy filter. Each case has an input observation and the expected decision,
including the rule that decided it. A client that drops a network the fixtures
keep is merely less useful; a client that keeps a network the fixtures drop is a
privacy bug and must not ship.

Implementations in this project that consume the fixtures:

- `api` — `tests/test_filter_fixtures.py`
- `android` — planned, phase 2
- `ios` — planned, phase 3

## Changing the spec

1. Open an issue describing the behaviour change.
2. Change the rule text and the fixtures in the same pull request. A rule
   without a fixture is not a rule.
3. Bump `CHANGELOG.md`. Breaking changes to a wire format get a new schema id
   (`wifishare.batch/2`), never a silent change to an existing one.

## License

Apache-2.0. See `LICENSE`.
