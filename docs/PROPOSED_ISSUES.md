# Project Analysis — Proposed Improvements

> Generated from a full review of `custom_components/tuya_smart_ir_ac` (version `2026.6.4`).
> GitHub Issues are disabled on this repository, so each proposed issue is captured below in
> Markdown, ready to be copied into the upstream tracker or addressed directly via PR.
> Ordered by severity. Each entry lists the precise location, impact, and a proposed fix.

| # | Severity | Area | Title |
|---|----------|------|-------|
| 1 | 🔴 Critical | Correctness | `NameError: Callable` in `bridge.py` prevents the whole integration from loading |
| 2 | 🟠 High | Packaging | `pycryptodome` runtime dependency not declared in manifest `requirements` |
| 3 | 🟠 Medium | Security | Pulsar WebSocket disables TLS verification; message signature check never called |
| 4 | 🟡 Medium | Stability | Pulsar reconnect task can be garbage-collected; loop-unsafe dispatch via `add_job` |
| 5 | 🟡 Medium | Performance | Sensor smart-polling never recognizes Pulsar DPS codes → redundant cloud polling |
| 6 | 🟡 Medium | UX / Data | Editing options wipes every device-registry entry on each update |
| 7 | 🔵 Low | Quality | No automated tests or linting in CI; plus housekeeping cleanups |

---

## 1. 🔴 Critical: `NameError: Callable` in `bridge.py` prevents the whole integration from loading

### Summary
`custom_components/tuya_smart_ir_ac/bridge.py` references the name `Callable` in a function
annotation, but `Callable` is **never imported** and the module does **not** use
`from __future__ import annotations`. Python evaluates function-parameter annotations eagerly at
definition time, so importing `bridge.py` raises `NameError` immediately.

`bridge.py` is imported unconditionally by `__init__.py`
(`from .bridge import TuyaPulsarBridge`), so this propagates and **the entire integration fails to
set up** — not just the Pulsar feature.

### Location
`custom_components/tuya_smart_ir_ac/bridge.py:19`
```python
def register_handler(self, dev_id: str, callback: Callable) -> None:
```
The file imports only `logging`, `json`, `HomeAssistant`, and `TuyaOpenPulsar` — no
`from typing import Callable` and no `from __future__ import annotations`.

### Reproduction
Executing the class body with external imports stubbed (so only the annotation matters) fails
deterministically:
```
FAILED AT IMPORT/CLASS-DEF: NameError: name 'Callable' is not defined
```
Minimal standalone repro:
```python
class T:
    def f(self, x: Callable): pass   # NameError: name 'Callable' is not defined
```

### Impact
- **Critical.** The integration cannot be imported/loaded in any environment, regardless of whether
  Pulsar is enabled.
- An import smoke test or linter (see Issue 7) would catch this instantly.

### Proposed fix
Add at the top of `bridge.py`:
```python
from collections.abc import Callable
```
Optionally use a precise type matching the async handler, e.g.
`Callable[[str, dict], Awaitable[None]]` (consistent with `openpulsar.py`, which already imports
`Callable`/`Awaitable` from `typing`).

---

## 2. 🟠 High: `pycryptodome` runtime dependency is not declared in manifest `requirements`

### Summary
`custom_components/tuya_smart_ir_ac/tuya_connector/openpulsar.py:9` imports the AES primitives:
```python
from Crypto.Cipher import AES
```
The `Crypto` namespace is provided by **`pycryptodome`**, but `manifest.json` declares:
```json
"requirements": [],
```
`openpulsar.py` is imported at integration load time (via `tuya_connector/__init__.py` →
`__init__.py`), so the import is **not** lazy. On a Home Assistant install without `pycryptodome`
present, this raises `ModuleNotFoundError` and the integration fails to load.

### Impact
- **High.** HA does not guarantee `pycryptodome` in core; relying on it being incidentally installed
  breaks clean/minimal installs and containers. HA installs declared `requirements` automatically
  before loading the integration.

### Proposed fix
Declare it in `manifest.json`:
```json
"requirements": ["pycryptodome>=3.20.0"]
```
Pin a sensible minimum. Alternatively migrate AES-GCM/ECB handling to `cryptography` (already a
core HA dependency), but declaring `pycryptodome` is the smallest correct change.

---

## 3. 🟠 Medium: Pulsar WebSocket disables TLS verification (`ssl=False`); signature check never called

### Summary
Two related weaknesses in the Pulsar stream (`tuya_connector/openpulsar.py`):

**3a. TLS certificate verification disabled** — `openpulsar.py:111-116`:
```python
async with session.ws_connect(
    self._topic_url,
    headers=headers,
    heartbeat=PING_INTERVAL_SECONDS,
    ssl=False,          # <-- disables certificate validation
) as ws:
```
`ssl=False` turns off certificate validation for the `wss://` connection, exposing the event stream
to MITM interception/tampering. The endpoints are well-known public Tuya hosts
(`wss://mqe.tuya*.com:8285/`) with valid certificates, so verification should succeed normally.

**3b. Message signature never verified** — `_verify_v2_sign()` (`openpulsar.py:185-189`) implements
Tuya's MD5 v2-envelope signature check but is **never called** in `_process_message()`. Payloads
are decrypted and dispatched without integrity verification (dead code + missed defense).

### Impact
- **Medium** (defensive). Data is AES-encrypted, but disabling TLS removes transport protection and
  the unused signature check leaves envelope integrity unverified.

### Proposed fix
- Remove `ssl=False`; rely on default verification or pass a context from
  `homeassistant.util.ssl.client_context()`.
- Wire `_verify_v2_sign()` into `_process_message()` for `encryptVersion == "v2"` messages and
  drop/log failures — or remove the method if intentionally unused.

---

## 4. 🟡 Medium: Pulsar reconnect task can be garbage-collected; loop-unsafe dispatch via `add_job`

### Summary
**4a. Fire-and-forget task** — `openpulsar.py:83-86`:
```python
async def start(self):
    self._stop_event.clear()
    asyncio.create_task(self._connect_loop())   # return value discarded
```
The event loop keeps only a *weak* reference to tasks. A bare `asyncio.create_task` whose result is
not stored can be garbage-collected mid-flight, silently killing the reconnect loop (this is exactly
what `ruff` rule `RUF006` flags). Store the handle (`self._task = ...`) and cancel it in `stop()`.

**4b. Dispatch helper choice** — `bridge.py:31`:
```python
self.hass.add_job(handler, dev_id, data)
```
`_on_message` already runs on the event loop, and `handler` is a coroutine function. Prefer the
explicit `self.hass.async_create_task(handler(dev_id, data))` (or `async_run_hass_job`) over the
legacy `add_job` dispatcher for clarity and forward compatibility.

**4c. Single handler per device** — `bridge.register_handler` stores handlers in a dict keyed by
`dev_id`, so a second `register_handler` for the same device silently overwrites the first. Today
climate and sensor coordinators register disjoint device IDs, so it's latent rather than active, but
a `dict[str, list[callback]]` (or a guard) would make it robust.

### Impact
- **Medium.** 4a can cause the real-time stream to stop without any error after some time under GC
  pressure — hard to diagnose. 4b/4c are robustness/maintainability hardening.

### Proposed fix
```python
async def start(self):
    self._stop_event.clear()
    self._task = asyncio.create_task(self._connect_loop())

async def stop(self):
    self._stop_event.set()
    if self._task:
        self._task.cancel()
    ...
```
Switch the bridge dispatch to `async_create_task`, and store handlers as a list per device.

---

## 5. 🟡 Medium: Sensor smart-polling never recognizes Pulsar DPS codes → redundant cloud polling

### Summary
The sensor coordinator implements a "skip API poll if Pulsar already delivered fresh data"
optimization, but the freshness bookkeeping compares **raw Tuya codes** against **normalized field
names**, so they never match for temperature/humidity.

`coordinator.py:293-296` (Pulsar path) records the raw codes:
```python
codes = [item.get("code") for item in new_status.get("status", [])]
self._update_dps_timestamp(device_id, codes)
```
For T&H sensors the raw codes are `va_temperature` / `va_humidity` (see
`const.TUYA_CODE_MAPPING`). But `_update_dps_timestamp` (`coordinator.py:242-250`) only keeps codes
that are in `TuyaSensorData.get_dps_codes()`, which returns the **normalized** field names
`["temp_current", "humidity_value", "battery_state"]`:
```python
for code in [c for c in codes if c in monitored_codes]:
    self._pulsar_last_updates[device_id][code] = now
```
`va_temperature`/`va_humidity` are filtered out, so their freshness is never recorded. As a result
`_needs_api_refresh` keeps returning `True` for those DPS and the coordinator keeps polling the
cloud even while Pulsar is actively delivering the data — defeating the optimization for the two
most important metrics. (The periodic path at `coordinator.py:282-283` passes canonical codes, so it
appears to work there, masking the bug.)

### Impact
- **Medium (performance / API quota).** Unnecessary Tuya cloud calls when Pulsar is enabled,
  increasing the chance of throttling — the opposite of this feature's intent.

### Proposed fix
Normalize codes before comparison in the Pulsar path, e.g. reuse
`helpers.normalize_tuya_payload` / `TUYA_CODE_MAPPING`:
```python
codes = [TUYA_CODE_MAPPING.get(item.get("code"), item.get("code"))
         for item in new_status.get("status", [])]
```
Add a unit test asserting that a `va_temperature` Pulsar update marks `temp_current` fresh and
suppresses the next poll.

---

## 6. 🟡 Medium: Editing options wipes every device-registry entry on each update

### Summary
`__init__.py:206-211` (`async_update_entry`) removes **all** device-registry entries for the config
entry on every non-reboot options change (adding a device, editing a preset, etc.):
```python
if entry.disabled_by is None:
    device_registry = dr.async_get(hass)
    devices = dr.async_entries_for_config_entry(device_registry, entry.entry_id)
    for device_entry in devices:
        device_registry.async_remove_device(device_entry.id)
await hass.config_entries.async_unload_platforms(entry, PLATFORMS)
...
await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)
```
Entities re-register with stable `unique_id`s (so entity settings survive), but removing the
**device** entries discards user-assigned **area**, custom device name, and other device-level
customizations, and causes device churn in the registry on every edit.

### Impact
- **Medium (UX / data loss).** Users lose area assignments and device-level tweaks whenever they
  touch options. Surprising and frustrating for a config-flow-driven integration.

### Proposed fix
Reconcile instead of nuking: only remove device entries that no longer correspond to a configured
sub-device (stale cleanup), and let still-configured devices persist. Compute the set of valid
`(DOMAIN, identifier)` tuples from current options and remove only the difference.

---

## 7. 🔵 Low: No automated tests or linting in CI; housekeeping cleanups

### Summary
CI (`.github/workflows/`) runs only `hassfest` and the HACS action — both validate manifest/repo
structure, not Python correctness. There are **no unit tests** and no linter/type checker. A trivial
import smoke test or `ruff` run would have caught Issue 1 (undefined `Callable`) and would surface
Issue 4a (`RUF006`).

### Proposed additions
- Add `pytest` + [`pytest-homeassistant-custom-component`](https://github.com/MatthewFlamm/pytest-homeassistant-custom-component)
  with, at minimum: an import smoke test for every module, config-flow tests, and coordinator
  parsing/freshness tests (covering Issue 5).
- Add `ruff` (lint + format) and a `mypy`/`pyright` pass to CI.
- A CI job that simply imports the package would be a cheap, high-value guard.

### Housekeeping cleanups (bundle into the same PR)
- **`const.py:24`** — `TEST_MODE = True` appears to be an unused debug constant; remove it (or wire
  it up intentionally). A `True` "test mode" flag shipping in releases is a smell.
- **`manifest.json`** — `"iot_class": "cloud_polling"`, but with Pulsar enabled the integration is
  push-based. Consider `"cloud_push"` (or document the polling-default rationale) and add a
  `"loggers"` key for proper log namespacing.
- **Type hints** — `entity.get_preset_modes()` is annotated `-> str` but returns
  `list[str] | None`; `get_preset_mode()` similarly. Tighten these (a type checker would flag them).
- **Stray comments** — `openapi.py` contains leftover Italian inline comments
  (lines ~73, ~216, ~232, ~247); translate or remove for consistency.
- **Empty-padding edge case** — `openpulsar._decrypt_ecb` does `decrypted[:-padding_len]`; if a
  malformed frame yields `padding_len == 0`, this returns an empty string. A defensive guard
  (`1 <= padding_len <= block_size`) would harden it.
