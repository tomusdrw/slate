# Design: native Tuya cloud provider (`slate_tuya`)

Date: 2026-09-09
Status: approved (design review passed in brainstorming session)

## Summary

A fourth provider for Slate that controls Tuya / Smart Life devices through the
official Tuya OpenAPI (cloud), so a panel with Tuya devices works with nothing
else running — the same motivation as the Shelly provider. The panel polls
bound devices over signed HTTPS every few seconds, publishes normalized
snapshots to the state store, and translates semantic actions into Tuya
commands. Credentials (region, Access ID, Access Secret, app account UID) are
entered on the editor's Integrations page and stored in NVS; Access ID and
Access Secret are write-only through Slate APIs. An
editor resource picker lists Tuya devices by friendly name.

## Decisions locked during brainstorming

- Native firmware provider, not a `direct`-provider bridge script — the panel
  must work with no companion process and no Home Assistant.
- Official Tuya OpenAPI only. The unofficial Smart Life app API is rejected:
  undocumented and has been killed by Tuya before.
- REST polling every ~5 s, not the cloud message-queue push. Push adds a second
  TLS connection and subscription management for freshness a wall panel does
  not need.
- Editor picker with friendly names (HA-style catalog), not manual device-id
  entry.
- Device scope: Slate's tile types — `light` (power, brightness, color
  temperature), `cover` (open/stop/close/position), `sensor` (temperature,
  humidity, power).

## Architecture fit

The provider plugs into the existing ADR-3 seams with no core changes:

- Registers provider id `tuya` with the state store (`slate_state_provider_register`)
  and the action bus (`slate_action_provider_register`), following Shelly's
  registration-first rule: registration happens even when the poller cannot
  start, so a tile shows a missing resource rather than a missing provider.
- `subscribe()` hands it the bound resource-id set; only bound devices are
  polled (§5.1's "what is bound is what exists").
- Publishes §5.2 normalized snapshots (`light`/`cover`/`sensor` kinds,
  capability bits, percent/kelvin vocabulary).
- Catalog served on the existing `GET /resources?provider=tuya` route for the
  editor picker.

### Honest costs (accepted)

- **Breaks the LAN-only property.** The update check is no longer the only
  traffic leaving the LAN. README, SECURITY.md and DESIGN.md must be updated;
  §5.2's per-provider offline semantics already keep a Tuya outage from
  staling Shelly/HA tiles.
- **Tuya trial/quota expiry becomes a wall-visible failure.** The provider must
  report this as a distinct `error` reason, not generic unreachability.
- **DP mapping is the real work.** Tuya has no uniform device vocabulary; the
  mapping below can only be fully validated against real devices.

## Components

### `firmware/components/slate_tuya/` — three files

- **`slate_tuya_client.c`** — OpenAPI transport. HMAC-SHA256 request signing
  (mbedTLS), HTTPS via `esp_http_client` with the ESP-IDF CA bundle, token
  acquisition (`/v1.0/token?grant_type=1`) and refresh at ~80% of token
  lifetime. Exposes a small `tuya_get(path)` / `tuya_post(path, body)`
  interface; nothing above this layer sees a signature.
- **`slate_tuya_map.c`** — DP→snapshot vocabulary mapping. Pure functions over
  parsed JSON (functions spec, status payloads) → normalized snapshots,
  capabilities and command payloads. No network, no RTOS dependencies;
  directly unit-testable.
- **`slate_tuya.c`** — lifecycle. `slate_tuya_init()` registers with store and
  action bus (`unconfigured` until credentials exist); `slate_tuya_start()`
  attaches to the Wi-Fi lifecycle and runs the poller task. Called from
  `main.c` next to the Shelly provider.

## Credentials and setup

Settings: **region** (one of Tuya's endpoint regions: `us`, `eu`, `cn`, `in`,
…, selects the API host), **Access ID**, **Access Secret**, **app account
UID** (Smart Life / Tuya Smart account linked to the cloud project).

- Stored in NVS like the HA token. They only go in: `GET` returns
  `{"configured":true,"region":"eu","uid":"…"}` and never the secret.
- Endpoints mirror the HA shape: `GET /api/v1/tuya`,
  `POST /api/v1/tuya`, `DELETE /api/v1/tuya`.
- `POST` validates against the cloud (fetch a token, list devices) **before**
  replacing working credentials — a bad secret never strands a working setup.
- Editor: a Tuya card on the Integrations page with connect form (region
  dropdown, three text fields), status display, disconnect button.

## Bindings, catalog, picker

- Resource id is the Tuya device id (within `SLATE_RESOURCE_ID_MAX`) except that
  a dual sensor's independently bindable humidity reading uses
  `<device-id>/humidity`. The suffix is Slate-only and is stripped before every
  cloud request; existing raw-id temperature bindings need no migration.
- Catalog: `GET /resources?provider=tuya` relays
  `GET /v1.0/users/{uid}/devices` as normalized entries (`id`, `name`,
  `kind`). Account discovery runs on a dedicated worker and publishes one
  bounded cache snapshot, so the shared HTTP task never waits on Tuya TLS.
- Devices whose category or specification has no honest Slate kind are omitted
  from the typed picker. Version 1 has no `unsupported` kind and no manual Tuya
  id fallback; inventing either would make an unrenderable resource look usable.

## DP mapping

The mapping keys on each device's **device specification**
(`GET /v1.1/devices/{id}/specifications`): reportable `status` entries prove
read-only readings and `functions` entries prove actions. It keys on DP codes,
never hardcoded DP numbers, which vary across models.

| Slate kind | Tuya categories | DP codes |
|---|---|---|
| `light` | `dj`, `dd`, `fwd`, … | `switch_led`, `bright_value(_v2)`, `temp_value(_v2)` |
| `cover` | `cl`, `clkg` | `control` (open/stop/close), `percent_control`, `percent_state` |
| `sensor` | `wsdcg`, metered plugs | `va_temperature`, `va_humidity`, `cur_power` |

- Capabilities derive from the functions spec: a light without `bright_value`
  publishes `toggle` only, and the tile hides the slider (§7.1).
- Scaling is normalized here: Tuya's temperature ×10 and brightness 10–1000
  ranges become Slate's percent and kelvin vocabulary before the store sees
  them.
- A device the table cannot classify is never published (the store has no
  `unknown` kind by design).

## Actions

`toggle` / `set_power` / `set_brightness` / `set_color_temperature` /
`open` / `stop` / `close` / `set_position`
→ `POST /v1.0/devices/{id}/commands` with the mapped DP codes and inverse
scaling. Delivery semantics are the existing rule: HTTP success acknowledges
the command; the tile's pending state clears on the next published snapshot
showing the new value. After a dispatched action the poller re-polls that
device promptly (Shelly's post-action sweep pattern), so pending clears in
~1 s rather than waiting out the interval.

## Lifecycle and error handling

`unconfigured` (no credentials) → `connecting` (token fetch) → `online`
(polling). Failures:

- `offline` — DNS/TLS/network failure (including Wi-Fi station loss and ISP
  outage; the panel reports what it can observe). Only this provider's
  resources stale, per §5.2.
- `error` — needs a person: credentials rejected, **Tuya trial/quota expiry**.
  Each carries a specific reason string visible via `GET /status`, so a dead
  dashboard is diagnosable from the editor.

## Polling

5 s sweep over bound devices, mirroring Shelly's `POLL_INTERVAL_MS`. Quota
note: ~10 devices at 5 s ≈ 173 k calls/day, which may exceed some Tuya trial
quotas — the interval is a documented constant in v1; if quota proves to be a
problem, make it adaptive (slow down when nothing changes) as a follow-up
rather than shipping the complexity unmeasured.

## Testing

- On-device selftest for the mapping (`-DSLATE_TUYA_SELFTEST=1`, Shelly's
  pattern): fixture JSON (captured functions specs and status payloads per
  category) → expected snapshots. Needs no network; covers the code where "a
  wrong number on a wall" lives.
- Client and lifecycle validated on real hardware against real devices; there
  is no trustworthy Tuya simulator.

## Documentation updates

- README — Providers section (new `tuya` entry) and the "nothing outside the
  local network" claim, which becomes "two things": the update check and Tuya
  cloud traffic, when configured.
- SECURITY.md — cloud dependency and credential handling (write-only secrets,
  same posture as the HA token).
- DESIGN.md — provider table gains `tuya`.
- docs/API.md — `/api/v1/tuya` endpoints, resource-id format, catalog entry
  shape.

## Resource budget

CA bundle ≈ 64 KB flash; signing/HTTP/mapping code modest — the 6 MB
application slot has ample headroom. One TLS session ≈ 17 KB RAM while a poll
is in flight. `SLATE_STATE_MAX_PROVIDERS` (currently 8) already has slots for
a fourth provider.

## Explicitly out of scope

- Tuya message-queue push (Pulsar/MQTT) for instant state.
- Local (TinyTuya-style LAN) control protocol.
- Scenes/automations sync from the Tuya cloud.
- Adaptive polling (follow-up only if quota demands it).
