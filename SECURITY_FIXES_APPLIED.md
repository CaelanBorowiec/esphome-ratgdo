# Security Fixes Applied: Uncommanded Garage Door Opening

This document describes the code changes made to address the **High** and **Medium** issues identified in `SECURITY_FINDINGS_UNCOMMANDED_OPEN.md`. Each section below corresponds to one finding and its fix.

---

## Fix 1: High — Reject move-to-position when door position is unknown

**Finding:** A client could send “set cover position” (e.g. to 50%) before the device had received any door state. With `door_position` still at `DOOR_POSITION_UNKNOWN` (-1.0), the code computed `delta = position - (-1) > 0` and sent **OPEN**, causing an uncommanded open after boot or reconnect.

**Fix:** Reject `door_move_to_position()` when the current door position is unknown. No open or close command is sent until the device has a known position (from protocol status or limit switches).

### Change

**File:** `components/ratgdo/ratgdo.cpp`  
**Function:** `RATGDOComponent::door_move_to_position(float position)`

- **What was added:** A guard at the start of the function (after handling OPENING/CLOSING) that returns immediately if `*this->door_position == DOOR_POSITION_UNKNOWN`, with a warning log.
- **Behavior:** Any call to move the cover to a numeric position (e.g. from Home Assistant or the API) is ignored until the firmware has set `door_position` to a known value (0.0–1.0). Once the door state has been reported (OPEN, CLOSED, or STOPPED), `door_position` is set and move-to-position works as before.

### Code diff (conceptual)

```cpp
// Added after the OPENING/CLOSING block, before computing delta:

if (*this->door_position == DOOR_POSITION_UNKNOWN) {
    ESP_LOGW(TAG, "Door position unknown, ignoring move to position %.2f (query door state first)", position);
    return;
}
```

### Notes

- `DOOR_POSITION_UNKNOWN` is defined in `components/ratgdo/ratgdo.h` as `-1.0`.
- Position becomes known when the protocol reports OPEN (→ 1.0), CLOSED (→ 0.0), or STOPPED (→ 0.5 if still unknown), or from dry-contact limit state.
- No change to direct `cover.open` / `cover.close` or to `door_toggle()`; only “set position” is gated on known position.

---

## Fix 2: Medium — Dry contact script boot delay

**Finding:** The dry contact door script runs on **on_press** of either the open or close GPIO. At power-up, GPIO and pull-ups can stabilize such that the “open” contact briefly appears ON and “close” OFF, causing the script to run and send **cover.open** — an uncommanded open at boot. Noise or contact bounce could have a similar effect.

**Fix:** Introduce a boot delay before the dry contact script is allowed to perform any door action. For the first 10 seconds after boot, the script does nothing when triggered. After that, behavior is unchanged.

### Changes

**Files:** `base.yaml`, `base_secplusv1.yaml`

1. **Global variable**
   - **Component:** `globals`
   - **Id:** `${id_prefix}_dry_contact_ready`
   - **Type:** `bool`
   - **Initial value:** `false` (not restored from flash)
   - **Purpose:** Represents “boot delay has elapsed; dry contact actions are allowed.”

2. **Boot automation**
   - **Component:** `on_boot`
   - **Actions:** Wait 10 seconds, then set `${id_prefix}_dry_contact_ready` to `true`.
   - **Effect:** After 10 s, the dry contact script is allowed to open/close/toggle.

3. **Script guard**
   - **Script:** `${id_prefix}_dry_contact_door_control`
   - **Change:** The entire existing logic (0.1 s delay + three `if` blocks for both-on/toggle, open-only, close-only) is wrapped in a single outer `if` whose condition is:
     - `lambda: "return id(${id_prefix}_dry_contact_ready);"`
   - **Effect:** When the script runs (on press of open or close contact), it first checks the global. If the boot delay has not yet completed, the script exits without sending any cover command.

### Behavior summary

| Time after boot | `dry_contact_ready` | Script on contact press |
|-----------------|---------------------|--------------------------|
| 0–10 s          | `false`             | No door action           |
| After 10 s      | `true`              | Open/close/toggle as before |

### Notes

- **base_drycontact.yaml** was not changed: it does not define the dry contact door control script (it uses the same pins as limit switches only), so there is no script to delay.
- The 10 s delay is a conservative default; it can be adjusted in the `on_boot` `delay` if needed for your hardware.
- The existing 500 ms `delayed_on_off` filter on the dry contact binary sensors is unchanged and still reduces bounce; the boot delay adds protection specifically for power-up and early GPIO settling.

---

## Summary table

| Severity | Issue | Fix | Files touched |
|----------|--------|-----|----------------|
| High     | Move-to-position with unknown position could open door | Reject move when `door_position == DOOR_POSITION_UNKNOWN` | `components/ratgdo/ratgdo.cpp` |
| Medium   | Dry contact script could open at boot or due to noise | 10 s boot delay before script can run door actions | `base.yaml`, `base_secplusv1.yaml` |

---

## Verification

- **High:** After flash, before the device has reported door state, send “set cover position” to e.g. 0.5 from HA or API. The door should not move, and the log should show the “Door position unknown, ignoring move to position” warning. After the door state is known, set position again; the door should move as expected.
- **Medium:** Power cycle the device. Press the dry contact (or short the open contact) within the first 10 seconds; the door should not move. After 10 seconds, the same action should open/close/toggle as before.

---

*These fixes address the High and Medium items from the security review. The Low and informational items in `SECURITY_FINDINGS_UNCOMMANDED_OPEN.md` remain as recommendations (API/auth, documentation) and were not changed in code.*
