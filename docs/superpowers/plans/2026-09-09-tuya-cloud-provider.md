# Tuya Cloud Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a native `tuya` provider to Slate that polls Tuya/Smart Life cloud devices via the official Tuya OpenAPI and lets the editor bind tiles to them by friendly name.

**Architecture:** New firmware component `slate_tuya` in three files — `slate_tuya_client.c` (signed HTTPS transport + token lifecycle), `slate_tuya_map.c` (pure DP↔snapshot mapping, host-testable), `slate_tuya.c` (provider lifecycle: registration, settings routes, poller task, action dispatch) — plus dedicated secret accessors in `slate_store`, wiring in `main.c`, a Tuya card in the editor's Integrations dialog, and docs.

**Spec:** `docs/superpowers/specs/2026-09-09-tuya-cloud-provider-design.md` (normative; this plan is its decomposition).

**Tech Stack:** ESP-IDF v5.5.5 C (esp_http_client, esp_crt_bundle, mbedTLS HMAC-SHA256, cJSON, FreeRTOS), React/TypeScript editor, `node --test`.

## Global Constraints

- Firmware targets ESP-IDF v5.5.5; C, no C++; `extern "C"` guards in headers.
- **ESP-IDF is NOT installed in this environment.** Firmware tasks are verified by (a) careful adherence to existing component patterns, (b) host compilation of pure modules where feasible, and (c) code review. `idf.py build` must be run by the user afterwards; the plan's last task says so.
- Node 22 is present (project pins 24; editor tests run fine on 22 — use what exists).
- Style: follow the existing codebase — extensive rationale comments in file headers, `ESP_LOGE(... " — continuing")` degraded-mode logging, no `ESP_ERROR_CHECK` in app paths.
- Secret credentials are write-only: stored via dedicated `slate_store` accessors, never returned from any API route.
- `POST /api/v1/tuya` (not PUT) for connect — matches the `POST /ha` precedent. Spec was corrected.
- Never block the LVGL/binding task: `subscribe()` and `dispatch()` do bookkeeping only and hand network work to the provider's own task (Shelly pattern).
- Action dispatch returns `ESP_OK` to mean "took responsibility"; `slate_action_result()` reports delivery later (§5.3).
- Provider id: `"tuya"`. Resource id: raw Tuya device id (≤63 chars).
- Do NOT `git commit` anything — the user explicitly said no commits. Stage nothing; leave the working tree dirty.

---

### Task 1: `slate_store` Tuya credential accessors

**Files:**
- Modify: `firmware/components/slate_store/include/slate_store.h`
- Modify: `firmware/components/slate_store/slate_store.c`

**Context:** HA credentials use dedicated accessors (`slate_store_ha_set/url_get/token_get/token_is_set/clear`, slate_store.h:326-361) backed by a *private* NVS key for the secret (no `SLATE_KEY_*` constant; see the §12 rule comment near h:117-124: "a key constant a serialiser can name is a key constant a serialiser can read"). `slate_store_ha_set` commits both values in one NVS transaction. Tuya needs four values: region (non-secret but keep private for symmetry), access_id, access_secret (the actual secret), uid.

**Interfaces (produced — later tasks rely on these exact names):**

```c
/* slate_store.h — new limits near SLATE_HA_TOKEN_MAX_LEN */
#define SLATE_TUYA_REGION_MAX_LEN     8    /* "eu", "us", "cn", "in", ... */
#define SLATE_TUYA_ACCESS_ID_MAX_LEN  32
#define SLATE_TUYA_SECRET_MAX_LEN     64
#define SLATE_TUYA_UID_MAX_LEN        64

/* All four committed in one NVS transaction. Any NULL/empty arg -> ESP_ERR_INVALID_ARG. */
esp_err_t slate_store_tuya_set(const char *region, const char *access_id,
                               const char *secret, const char *uid);
/* ESP_ERR_NOT_FOUND when unset; each getter fails unless ALL four are present. */
esp_err_t slate_store_tuya_get(char *region, size_t region_len,
                               char *access_id, size_t access_id_len,
                               char *secret, size_t secret_len,
                               char *uid, size_t uid_len);
bool      slate_store_tuya_is_set(void);   /* cached like ha_token_is_set, no flash read */
esp_err_t slate_store_tuya_clear(void);
```

- [ ] Read `slate_store.c` around the HA accessors (grep `ha_set`, `key_is_secret`, the cached `s_ha_token_set` flag) and mirror the pattern exactly: private static key strings (`"tuya_rg"`, `"tuya_aid"`, `"tuya_sec"`, `"tuya_uid"`), length validation against the new limits, single-commit set, cached is_set flag refreshed at init and on set/clear.
- [ ] Add the four keys to the `key_is_secret` refusal list in `slate_store.c` (near line 431) so `slate_store_str_set/get` refuse them with `ESP_ERR_INVALID_ARG`.
- [ ] Verify: `grep -n "tuya" firmware/components/slate_store/slate_store.c firmware/components/slate_store/include/slate_store.h` shows the new API; check nothing else in the tree references the new names yet (`grep -rn slate_store_tuya firmware/ | grep -v slate_store` → empty).

---

### Task 2: `slate_tuya_map` — pure DP mapping module + host test

**Files:**
- Create: `firmware/components/slate_tuya/include/slate_tuya_map.h`
- Create: `firmware/components/slate_tuya/slate_tuya_map.c`
- Create: `firmware/components/slate_tuya/CMakeLists.txt` (final version in Task 4; create with map now)
- Test (host, throwaway dir): `/var/folders/h5/87gs6m6d76d7ht3k181k2zdh0000gn/T/opencode/tuya_map_test/`

**Context:** Tuya devices expose numbered DPs named by string `code` in the per-device functions spec (`GET /v1.0/devices/{id}/functions` → `{"result":[{"code":"switch_led","type":"Boolean","values":"{}"}, ...]}`) and report status as `{"result":[{"code":"switch_led","value":true}, ...]}`. DP *numbers* vary by model; `code` is the stable key. Scaling: `bright_value_v2` range 10–1000 (from `values` JSON `{"min":10,"max":1000}`), temperature sensors report ×10, plugs report power in deciwatts on some models — normalize to Slate's percent/kelvin/SI vocabulary.

**Interfaces (produced — Task 4 consumes these exact names):**

```c
/* slate_tuya_map.h */
#include "cJSON.h"
#include "slate_state.h"   /* slate_snapshot_t, SLATE_KIND_*, capabilities */

typedef enum {
    SLATE_TUYA_CLASS_UNSUPPORTED = 0,
    SLATE_TUYA_CLASS_LIGHT,
    SLATE_TUYA_CLASS_COVER,
    SLATE_TUYA_CLASS_SENSOR_TEMP_HUM, /* wsdcg-style: temperature+humidity */
    SLATE_TUYA_CLASS_SENSOR_POWER,    /* metered plug: power */
} slate_tuya_class_t;

/* Map a Tuya category string ("dj", "cl", "wsdcg", "cz"/"pc" plugs, ...) to a class. */
slate_tuya_class_t slate_tuya_classify(const char *category);

/* Parsed per-device DP layout, filled from the functions-spec JSON "result" array.
 * A code that is absent gets dp id 0 / present=false. */
typedef struct {
    slate_tuya_class_t cls;
    /* light */  bool has_switch, has_bright, has_temp; int bright_min, bright_max; int temp_min, temp_max;
    /* cover */  bool has_control, has_percent_control, has_percent_state;
    /* sensor temp/hum */ bool has_temperature, has_humidity; double temp_scale, hum_scale;
    /* sensor power */ bool has_power; double power_scale;
} slate_tuya_dps_t;

esp_err_t slate_tuya_map_functions(const char *category, const cJSON *functions_result,
                                   slate_tuya_dps_t *out);
/* Build a normalized snapshot from a status "result" array. available is set by caller.
 * Returns ESP_ERR_NOT_FOUND when nothing publishable is present. name is copied by the store. */
esp_err_t slate_tuya_map_status(const slate_tuya_dps_t *dps, const cJSON *status_result,
                                const char *resource, const char *name,
                                slate_snapshot_t *out);
/* Build the commands body for an action: {"commands":[{"code":...,"value":...}]}.
 * value_type/value mirror slate_action_request_t. Returns ESP_ERR_NOT_SUPPORTED for
 * actions the DPs don't cover. Caller frees the returned cJSON. */
esp_err_t slate_tuya_map_command(const slate_tuya_dps_t *dps, slate_action_t action,
                                 bool has_bool, bool bool_value, int32_t number_value,
                                 cJSON **out_body);
```

Mapping table (normative, from the spec):

| class | categories | codes |
|---|---|---|
| light | `dj`, `dd`, `fwd`, `gyd`, `xdd` | `switch_led`/`switch`/`switch_led_1`, `bright_value_v2`/`bright_value`, `temp_value_v2`/`temp_value` |
| cover | `cl`, `clkg` | `control` (enum open/stop/close), `percent_control`, `percent_state` |
| sensor temp/hum | `wsdcg`, `zndb` | `va_temperature`, `va_humidity` (scale from `values` if present, else ×10 → °C / %) |
| sensor power | `cz`, `pc` (when a power DP exists) | `cur_power` (scale per `values`, else ×0.1 → W; report W) |

Capabilities: light → `toggle|set_power` always, `set_brightness{min,max}` when has_bright, `set_color_temperature{min,max kelvin}` when has_temp; cover → `open|stop|close` + `set_position{0,100}` when has_percent_control; sensors → zeroed caps (read-only). Cover `position` from `percent_state` when present else `SLATE_STATE_ABSENT`; `motion` = `SLATE_COVER_IDLE` at rest (no motion DPs in v1).

- [ ] Write the header and `slate_tuya_map.c`. Pure C + cJSON only — **no esp_log, no FreeRTOS, no esp_err.h**: use plain return codes? No — keep `esp_err_t` for consistency; on host define it via a shim. Actually: use `int` returns with the same constant values (`0` ok, `0x105` not_found...) is ugly. Decision: include `esp_err.h` and shim it in the host test dir with `typedef int esp_err_t; #define ESP_OK 0 ...` — shim file lives only in the temp test dir, never in the tree.
- [ ] Host test: create the temp dir, fetch cJSON (`curl -L https://raw.githubusercontent.com/DaveGamble/cJSON/v1.7.18/cJSON.c` and `cJSON.h`), write `test_map.c` with fixture JSON strings for: a `dj` light with `bright_value_v2`+`temp_value_v2` (functions + status), a `cl` cover with percent, a `wsdcg` sensor, a metered `cz` plug, an unsupported category, and a light status missing brightness. Assertions: classify results, snapshot kinds/values/capabilities (incl. 10–1000 → 0–100 scaling and kelvin), command bodies for toggle/set_brightness(50%→500)/open/set_position(37), and ESP_ERR_NOT_SUPPORTED for `set_color_temperature` on a bright-only light.
- [ ] Compile and run: `cc -I. -I<firmware components slate_state/include> test_map.c cJSON.c <path>/slate_tuya_map.c -o test_map && ./test_map` — note `slate_state.h` is pure types (check its includes; if it pulls esp headers, copy the needed enum/struct definitions into the shim header instead). Iterate until all assertions pass.
- [ ] Write `firmware/components/slate_tuya/CMakeLists.txt`:
```cmake
idf_component_register(
    SRCS "slate_tuya.c" "slate_tuya_client.c" "slate_tuya_map.c"
    INCLUDE_DIRS "include"
    REQUIRES esp_event esp_http_client esp_timer json mbedtls slate_action slate_api slate_state slate_store slate_wifi
)
```
(Files `slate_tuya.c`/`slate_tuya_client.c` don't exist yet — that's fine, CMake is only read at firmware build time; Tasks 3–4 create them.)
- [ ] Report: host test output (all passing), any spec deviations discovered.

---

### Task 3: `slate_tuya_client` — OpenAPI transport

**Files:**
- Create: `firmware/components/slate_tuya/include/slate_tuya_client.h`
- Create: `firmware/components/slate_tuya/slate_tuya_client.c`

**Context:** Tuya OpenAPI signing (documented at developer.tuya.com — "Sign a request"): headers `client_id`, `sign` (uppercase hex HMAC-SHA256 of string-to-sign), `t` (ms timestamp), `sign_method: HMAC-SHA256`, `access_token` (all calls except token request), plus `nonce` optional. String-to-sign: `HTTP_METHOD\nSHA256_HEX(body)\n<sorted headers: client_id, access_token, t, ...>\npath-with-query`. Token: `GET /v1.0/token?grant_type=1` signed with secret → `{"result":{"access_token","refresh_token","expire_time":7200}}`. Refresh: `GET /v1.0/token/{refresh_token}`. Region endpoints: `https://openapi.tuyaus.com` (us), `tuyaeu.com` (eu), `tuyacn.com` (cn), `tuyain.com` (in), `openapi-ueaz.tuyaus.com` (ueaz), `openapi-weaz.tuyaeu.com` (weaz). HTTPS via `esp_http_client` with `.crt_bundle_attach = esp_crt_bundle_attach` (precedent: slate_update.c:469-482). HMAC-SHA256 via mbedTLS `mbedtls_md_hmac(mbedtls_md_info_from_type(MBEDTLS_MD_SHA256), key, keylen, data, datalen, out32)` — new to the tree but `mbedtls` is already a linked REQUIRES elsewhere. IMPORTANT: verify the exact string-to-sign construction against Tuya's docs via webfetch during implementation; sign errors are the #1 integration failure.

**Interfaces (produced — Task 4 consumes):**

```c
/* slate_tuya_client.h */
typedef struct slate_tuya_client *slate_tuya_client_handle_t;

/* cfg: region, access_id, secret, uid (copied). Does no I/O. */
esp_err_t slate_tuya_client_create(const char *region, const char *access_id,
                                   const char *secret, const char *uid,
                                   slate_tuya_client_handle_t *out);
void slate_tuya_client_destroy(slate_tuya_client_handle_t h);

/* Error classification, so the lifecycle can pick offline vs error (§Lifecycle). */
typedef enum {
    SLATE_TUYA_ERR_NONE = 0,
    SLATE_TUYA_ERR_NETWORK,   /* DNS/TLS/timeout — transient */
    SLATE_TUYA_ERR_AUTH,      /* token rejected, sign invalid — needs a person */
    SLATE_TUYA_ERR_QUOTA,     /* Tuya "permission deny"/trial-expired error codes */
    SLATE_TUYA_ERR_API,       /* other non-success API envelope */
} slate_tuya_error_kind_t;

/* One signed request. path includes the query string. body may be NULL (GET).
 * On ESP_OK *out is a parsed cJSON the caller deletes; *err_kind classifies failure.
 * Envelope {"success":false} maps to the matching kind, never ESP_OK.
 * out_body may be NULL for fire-and-forget. */
esp_err_t slate_tuya_client_request(slate_tuya_client_handle_t h,
                                    int method /*HTTP_METHOD_GET/POST/DELETE*/,
                                    const char *path, const char *body,
                                    cJSON **out_body, slate_tuya_error_kind_t *err_kind);

/* Validates credentials without a full client: fetch token + one device-list page.
 * Used by the POST /tuya route before storing. */
esp_err_t slate_tuya_client_test(const char *region, const char *access_id,
                                 const char *secret, const char *uid,
                                 slate_tuya_error_kind_t *err_kind);
```

Token lifecycle: lazy — the client holds token/expiry in memory; `request` refreshes when <20% lifetime remains (spec: refresh at ~80% consumed), retries once on token-invalid responses. One HTTPS call at a time; the only caller is the poller/action task, so a mutex around the client handle suffices for the POST /tuya validation path (which uses `slate_tuya_client_test`, not the live client).

- [ ] webfetch Tuya signing docs to confirm string-to-sign exact format before writing signing code.
- [ ] Implement; keep response buffer bounded (16 KB cap like Shelly's BODY_MAX rationale — catalog responses are the largest).
- [ ] Host-verification: none feasible (TLS+cloud); verify by review against Tuya docs + the esp_http_client pattern in slate_update.c. Note any API-shape uncertainties explicitly in the completion report.

---

### Task 4: `slate_tuya` — provider lifecycle, routes, poller

**Files:**
- Create: `firmware/components/slate_tuya/include/slate_tuya.h`
- Create: `firmware/components/slate_tuya/slate_tuya.c`

**Consumes:** Task 1 (`slate_store_tuya_*`), Task 2 (`slate_tuya_map_*`), Task 3 (`slate_tuya_client_*`), plus existing: `slate_state_provider_register/set_status/publish`, `slate_action_provider_register`, `slate_action_result`, `slate_action_provider_unavailable`, `slate_api_register_uri` (auth `SLATE_API_AUTH_DEVICE_TOKEN` only), `slate_api_send_json`, `slate_api_refuse`, `slate_wifi_status` + `SLATE_WIFI_EVENT` attach pattern.

**Interfaces (produced — Task 5 consumes):**

```c
/* slate_tuya.h */
#define SLATE_TUYA_PROVIDER_ID "tuya"
esp_err_t slate_tuya_init(void);   /* call after slate_state_init + slate_action_init,
                                      from start_api() (needs slate_api_init for routes) */
esp_err_t slate_tuya_start(void);  /* call from start_network(), after slate_wifi_init */
#ifdef SLATE_TUYA_SELFTEST
esp_err_t slate_tuya_selftest(void); /* exercises classify/map tables via slate_tuya_map */
#endif
```

Structure (mirror slate_shelly.c):
- One poller task (`xTaskCreate`, stack 8192 — TLS needs more than Shelly's 6144, priority 4), one command queue (`xQueueCreate(8, sizeof(command_t))`), bind handover via a static mutex held for pointer swaps only.
- `subscribe()` (UI task): copy bound ids into a fresh table (PSRAM `heap_caps_calloc`), swap under lock, poke `CMD_SWEEP`. No I/O.
- Per bound device the table keeps: device id, friendly name (filled from catalog fetch), cached functions spec (`slate_tuya_dps_t`), `ever_read`, `slate_state_value_t last`.
- Sweep: for each bound device `GET /v1.0/devices/{id}/status` (fetch functions spec once per device id, lazily on first sweep; refetch if classification was UNSUPPORTED and device re-appears), map via Task 2, `slate_state_publish`. Interval 5000 ms from end of sweep; abandon sweep early if a command is queued (Shelly pattern).
- `dispatch()` (action bus): validate provider/resource/action, copy to `command_t`, `xQueueSend(...,0)`, `ESP_ERR_NO_MEM` when full. Runner builds the command body via `slate_tuya_map_command`, `POST /v1.0/devices/{id}/commands`, reports `slate_action_result`, then re-polls that device promptly.
- Routes (all `SLATE_API_AUTH_DEVICE_TOKEN`, registered in `slate_tuya_init`):
  - `GET /api/v1/tuya` → `{"configured":bool,"region":"eu"|null,"uid":"…"|null}` — never the secret. 200 always.
  - `POST /api/v1/tuya` — body `{"region","access_id","secret","uid"}`. Validate lengths, call `slate_tuya_client_test` (async-park the request with `httpd_req_async_handler_begin` like slate_ha does for its credential test), on success `slate_store_tuya_set`, recreate the live client, poke sweep, 204. On failure refuse with codes: `invalid_config` (malformed), `tuya_auth_failed`, `tuya_unreachable`, `tuya_quota`.
  - `DELETE /api/v1/tuya` — `slate_store_tuya_clear`, destroy client, mark provider `unconfigured`, 204.
  - `GET /api/v1/resources?provider=tuya` catalog: register a catalog callback via `slate_api_resources_register` (slate_api.c:390-419 — read that function and its typedef first; nobody registers one today, so confirm the signature). The callback fetches `GET /v1.0/users/{uid}/devices`, maps each device to a Resource entry `{provider:"tuya", resource:<device id>, kind:<mapped slate kind>, name:<friendly name>, available:true, state:{}}`, marks unsupported-category devices with kind as mapped or omits with a count? — decision: include only mappable devices, since the picker filters by kind anyway; note this in API.md (Task 7). Cache the fetched list in PSRAM for 60 s so the picker's refetch-on-open doesn't hammer the cloud.
- Status transitions: `unconfigured` (no stored creds) → `connecting` (client created, first sweep in flight) → `online` (a sweep completed) / `offline` (SLATE_TUYA_ERR_NETWORK or wifi down — also `slate_action_provider_unavailable("offline")` on wifi drop) / `error` (AUTH or QUOTA kinds — log the Tuya error code so it's diagnosable). Wi-Fi event handler mirrors Shelly's (register, then read `slate_wifi_status` to close the startup race).
- Selftest: guarded `#ifdef SLATE_TUYA_SELFTEST`; drives classify/functions/status fixtures through Task 2's API against the *live store registration* (register-first rule) — keep it small since Task 2's host test covers the mapping logic.

- [ ] Implement `slate_tuya.h`, `slate_tuya.c`.
- [ ] Read `slate_api_resources_register` before writing the catalog callback; if its signature doesn't fit, fall back to letting the default `append_bound_resources` serve `?provider=tuya` and report that limitation.
- [ ] Cross-check every consumed symbol against the real headers (`grep -n` each in slate_state.h/slate_action.h/slate_api.h/slate_store.h).
- [ ] Report: route count added (3 — confirm `SLATE_API_MAX_URI_HANDLERS` 36 not exceeded; count existing registrations), uncertainties.

---

### Task 5: wiring — main.c, main CMakeLists, selftest knob

**Files:**
- Modify: `firmware/main/main.c` (add `slate_tuya_init()` in `start_api()` after `slate_shelly_init()` with the same log-and-continue pattern; add `slate_tuya_start()` in `start_network()` after `slate_shelly_start()`)
- Modify: `firmware/main/CMakeLists.txt` (add `slate_tuya` to REQUIRES; add `SLATE_TUYA_SELFTEST` to the selftest knob foreach if one exists — check how SLATE_SHELLY_SELFTEST is threaded)
- Modify: `firmware/components/slate_tuya/CMakeLists.txt` (append the selftest compile-definition block, mirroring slate_shelly's)

- [ ] Apply edits following the exact neighboring patterns.
- [ ] `grep -rn "slate_shelly_init\|slate_shelly_start" firmware/main/main.c` to confirm placement; verify include added (`#include "slate_tuya.h"`).
- [ ] Count registered URI handlers across the tree vs `SLATE_API_MAX_URI_HANDLERS` (36): `grep -rn "slate_api_register_uri" firmware/components --include=*.c` and tally distinct httpd_uri_t statics +3 for Tuya. If >36, bump the constant with a comment and report it.

---

### Task 6: editor — Tuya integration card

**Files:**
- Modify: `editor/src/lib/api.ts` (add `TuyaConfiguration { configured: boolean; region: string | null; uid: string | null }`; methods `tuyaConfiguration()`, `configureTuya(region, accessId, secret, uid)` → `POST /tuya` with 20000 ms timeout like HA, `disconnectTuya()` → `DELETE /tuya`)
- Modify: `editor/src/lib/providers.ts` (add `if (id === 'tuya') return 'Tuya'`)
- Modify: `editor/src/ui/IntegrationsDialog.tsx` (third `integration-card` after the HA card, replicating the HA card's state/busy/message/disconnect pattern exactly; fields: region `<select>` with options `eu,us,cn,in,ueaz,weaz` (labels "Europe", "Americas", "China", "India", "US East (Azure)", "EU West (Azure)"), Access ID text input, Access Secret password input, App account UID text input; chip from `providers.find(p => p.id === 'tuya')`; error mapper for codes `tuya_auth_failed` → "Tuya rejected those credentials", `tuya_unreachable` → "Could not reach the Tuya cloud", `tuya_quota` → "Tuya cloud subscription or quota problem — check iot.tuya.com", `invalid_config` → "Check the four fields"; note under the form (`.integration-note`): where to create the cloud project (iot.tuya.com) and link the Smart Life app)
- Test: `editor/tests/` — add a small test only if there's a pure-lib seam worth it (e.g. none for the dialog; skip UI tests — no framework exists). Run existing tests.

No `ResourcePicker.tsx`/`App.tsx` changes — `resources('tuya')` already takes the generic path; provider appears in pickers automatically once `/status` lists it.

- [ ] Implement the three file changes.
- [ ] Run `npm run check` and `npm test` in `editor/`; then `npm run build` and confirm `dist/index.html` is produced and its gzip size stays under the 409600 B budget (`gzip -c editor/dist/index.html | wc -c`).
- [ ] Report: check/test/build output, bundle size before/after.

---

### Task 7: docs

**Files:**
- Modify: `README.md` (Providers section: new "### Tuya — Smart Life cloud devices" after Shelly; and the Providers intro sentence "nothing outside the local network is involved" → mention the Tuya exception alongside the update check)
- Modify: `SECURITY.md` (read it first; add the cloud dependency + write-only credential posture in its existing style)
- Modify: `docs/DESIGN.md` (find the provider enumeration — §5.1/§4.1 — add `tuya`; keep edits minimal and in-style)
- Modify: `docs/API.md` (add `GET/POST/DELETE /tuya` endpoint docs near the `/ha` section; add `tuya` to the provider table near line 181; document `?provider=tuya` catalog behavior)

- [ ] Make the edits; match each file's prose voice (measured, no marketing).
- [ ] `grep -n "local network" README.md` — update every claim the Tuya provider falsifies.

---

### Task 8: final verification + handoff

- [ ] `grep -rn "slate_tuya\|slate_store_tuya" firmware/ editor/src | wc -l` sanity; re-run editor `npm run check && npm test && npm run build`.
- [ ] Re-run the Task 2 host test from its temp dir.
- [ ] `git status` review: list all touched files; confirm nothing committed.
- [ ] Handoff note to the user: firmware build requires ESP-IDF v5.5.5 — `tools/fonts/generate.sh && tools/editor/build.sh && cd firmware && idf.py build`, then flash/OTA; credential setup steps (iot.tuya.com project → link app → four fields in Integrations); what to check in logs on first run (`slate_tuya` TAG).
