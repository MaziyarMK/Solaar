# Solaar MX Master 3 — Settings Reset on Wake Fix

## Problem

MX Master 3 (Unifying receiver) loses all Solaar settings — DPI, key overrides,
divert-keys — every time the mouse wakes from deep sleep. The mouse pointer freezes
briefly, then comes back at default DPI (1000) with stock button behaviour. Restarting
Solaar temporarily restores settings until the next sleep/wake cycle (~10 min).

Reproducible trigger: `pkill solaar; solaar &` → settings correct → middle-click
(wakes mouse from sleep) → settings gone.

## Root Cause

Two bugs in Solaar that together prevent settings from being re-pushed on wake:

### Bug 1 — `notifications.py`: powered-on event is silently ignored

When the mouse wakes from deep sleep it sends a `WIRELESS_DEVICE_STATUS` HID++
notification with `data[2] = 1` ("powered on"). Solaar's handler only pushes settings
if the device simultaneously sets `data[1] = 1` ("please reconfigure me"). The MX
Master 3 firmware sends `data[1] = 0` on wake, so the handler does nothing.

### Bug 2 — `device.py`: operator precedence makes `push=True` a no-op

Even if `push=True` were passed, the condition in `device.changed()` is:

```python
or push
and (not self.features or SupportedFeature.WIRELESS_DEVICE_STATUS not in self.features)
```

Python's operator precedence makes `and` bind tighter than `or`, so this evaluates as
`push AND (device lacks WIRELESS_DEVICE_STATUS)`. The MX Master 3 *has* this feature,
so the clause is always `False` and `push=True` is silently ignored for this mouse.

This operator-precedence bug has existed in Solaar since December 2021. It was fixed in
Solaar master in May 2026 (PR #3173) but the fix had not yet been packaged for Ubuntu
at the time these changes were applied (`1.1.19+202603022202~ubuntu24.04.1`).

## Files Changed

Both files are in `/usr/share/solaar/lib/logitech_receiver/`.

### 1. `notifications.py` — line ~322

**Before:**
```python
if notification.data[1] == 1:  # device is asking for software reconfiguration so need to change status
```

**After:**
```python
if notification.data[1] == 1 or notification.data[2] == 1:  # requesting reconfiguration or powered on from sleep
```

Also expressible as a sed one-liner:
```bash
sudo sed -i 's/if notification\.data\[1\] == 1:  # device is asking for software reconfiguration so need to change status/if notification.data[1] == 1 or notification.data[2] == 1:  # requesting reconfiguration or powered on from sleep/' \
  /usr/share/solaar/lib/logitech_receiver/notifications.py
```

### 2. `device.py` — lines ~453-454

**Before:**
```python
                if (
                    was_active is None
                    or not was_active
                    or push
                    and (not self.features or SupportedFeature.WIRELESS_DEVICE_STATUS not in self.features)
                ):
```

**After:**
```python
                if (
                    was_active is None
                    or not was_active
                    or push
                ):
```

Delete the `and (not self.features ...)` line entirely. Also expressible as:
```bash
sudo sed -i '/and (not self\.features or SupportedFeature\.WIRELESS_DEVICE_STATUS not in self\.features)/d' \
  /usr/share/solaar/lib/logitech_receiver/device.py
```

## Startup Wrapper Fix

The Ubuntu/PPA package installs Solaar's Python modules and metadata under
`/usr/share/solaar/lib`, but the generated `/usr/bin/solaar` console-script wrapper
does not add that directory to `sys.path`. If Python cannot see the package metadata,
startup fails before Solaar code runs:

```text
importlib.metadata.PackageNotFoundError: No package metadata was found for solaar
```

The local wrapper patch is tracked in this repo as `solaar-wrapper-pythonpath.patch`.
It adds this line near the top of `/usr/bin/solaar`, immediately after `import sys`:

```python
sys.path.insert(0, '/usr/share/solaar/lib')
```

## Applying After a Solaar Package Upgrade

Ubuntu package upgrades will overwrite these files. Re-apply with:

```bash
sudo sed -i 's/if notification\.data\[1\] == 1:  # device is asking for software reconfiguration so need to change status/if notification.data[1] == 1 or notification.data[2] == 1:  # requesting reconfiguration or powered on from sleep/' \
  /usr/share/solaar/lib/logitech_receiver/notifications.py

sudo sed -i '/and (not self\.features or SupportedFeature\.WIRELESS_DEVICE_STATUS not in self\.features)/d' \
  /usr/share/solaar/lib/logitech_receiver/device.py

sudo patch /usr/bin/solaar -i ./solaar-wrapper-pythonpath.patch

pkill -f '/usr/bin/solaar'; solaar &
```

Check whether the upstream fix has landed before re-applying: if `notifications.py`
already contains `data[2] == 1` in that condition, both patches are no longer needed.

## Other Changes Made During Diagnosis

**`~/.config/solaar/config.yaml`** — removed 4 stale duplicate device profiles (old
MX Master 2S entries, a ghost Unifying profile, a Bluetooth profile for the same mouse).
Backup saved at `~/.config/solaar/config.yaml.bak`. Only the active Unifying profile
(`_wpid: '4082'`) and the G815 entry remain.

## Verification

```bash
# Watch DPI live — should stay at 1900 after mouse wakes from sleep
watch -n 3 'solaar config "MX Master 3" dpi'

# Leave mouse idle ~10 min, move to wake, confirm still 1900
```
