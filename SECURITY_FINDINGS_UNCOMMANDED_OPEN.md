# Security & Safety Review: Uncommanded Garage Door Opening

This document summarizes findings from an in-depth review of the esphome-ratgdo firmware for conditions that could cause the garage door to **open without explicit user command**. It is intended for developers and users to understand risks and mitigations.

---

## Executive Summary

| Severity | Issue | Location | Mitigation / status |
|----------|--------|----------|----------------------|
| **High** | Move-to-position with unknown door position can open the door | `ratgdo.cpp` | Reject move when position is unknown (see below) |
| **Medium** | Dry contact script can open on boot or noise | `base.yaml` / script | Delay script eligibility after boot; consider edge-trigger logic |
| **Medium** | Dry contact "both on" triggers toggle (can open if closed) | `base.yaml` script | Document; optional edge-trigger or debounce |
| **Low** | Cover open/position via API — no extra auth | API / cover | Rely on API password and network security |
| **Info** | Protocol RX never sends open | Sec+ / dry_contact | Confirmed safe |

---

## 1. Code Paths That Can Open the Door

The following are the **only** code paths that result in an open (or open-direction) command to the door:

1. **Cover open**  
   - **File:** `components/ratgdo/cover/ratgdo_cover.cpp`  
   - **Trigger:** `control()` when `call.get_position() == COVER_OPEN` (position 1.0).  
   - **Call chain:** `RATGDOCover::control()` → `RATGDOComponent::door_open()` → `door_action(DoorAction::OPEN)`.

2. **Cover set position (open direction)**  
   - **File:** `components/ratgdo/cover/ratgdo_cover.cpp` → `components/ratgdo/ratgdo.cpp`  
   - **Trigger:** `control()` with a position in `(0, 1)` (not fully open or closed) → `door_move_to_position(position)`.  
   - **Logic:** `delta = position - *door_position`. If `delta > 0`, `door_action(DoorAction::OPEN)` is called.  
   - **Call chain:** Cover → `door_move_to_position()` → `door_action(OPEN)` (and later STOP).

3. **Dry contact script — open**  
   - **Files:** `base.yaml`, `base_secplusv1.yaml` (and dry-contact variants).  
   - **Trigger:** Script `dry_contact_door_control` runs on **on_press** of `dry_contact_open` or `dry_contact_close`.  
   - **Condition:** `dry_contact_open` is ON and `dry_contact_close` is OFF → `cover.open`.

4. **Dry contact script — toggle**  
   - **Same script.** When **both** `dry_contact_open` and `dry_contact_close` are ON → `cover.toggle`.  
   - If the door is closed, toggle **opens** the door.

5. **Template button “Toggle door”**  
   - **File:** `base.yaml` (template button).  
   - **Trigger:** User presses the button in the UI → `door_toggle()`.  
   - Considered **user-initiated**; no uncommanded open from firmware logic.

**Confirmed:** Protocol receive paths (Security+ 1.0/2.0 and dry contact state updates) only call `ratgdo_->received(...)` to update door state. They **never** call `door_action(OPEN)` or any command that opens the door.

---

## 2. High: Move-to-Position With Unknown Door Position

**File:** `components/ratgdo/ratgdo.cpp` — `door_move_to_position(float position)`

**Issue:**  
`door_position` is initialized to `DOOR_POSITION_UNKNOWN` (`-1.0`) and remains unknown until the device has received a door state (OPEN, CLOSED, or STOPPED). The function does **not** check for unknown position:

```cpp
auto delta = position - *this->door_position;   // door_position can be -1.0
// ...
this->door_action(delta > 0 ? DoorAction::OPEN : DoorAction::CLOSE);
```

So when `door_position` is still `-1.0`:

- Any requested `position` in `(0, 1]` (e.g. 0.5 for “50%”) gives `delta = position - (-1) > 0` → **OPEN** is sent.
- A client (e.g. Home Assistant) that sends “set cover position to 50%” or “restore last position” **before** the device has reported state can therefore cause an **uncommanded open** after boot or reconnect.

**Recommendation:**  
Reject move-to-position when door position is unknown, e.g.:

- If `*this->door_position == DOOR_POSITION_UNKNOWN` (or equivalent check), log a warning and **return** without calling `door_action(OPEN)` or `door_action(CLOSE)`.
- Optionally: query status first and wait for a known state before allowing move-to-position.

---

## 3. Medium: Dry Contact Script at Boot or Due to Noise

**Files:** `base.yaml`, `base_secplusv1.yaml`, `base_drycontact.yaml` (script and binary_sensor `on_press`).

**Setup:**

- Two GPIO binary sensors: `dry_contact_open`, `dry_contact_close` with `delayed_on_off: 500ms`.
- On **on_press** of **either** sensor, the script `dry_contact_door_control` runs and evaluates:
  - Both on → `cover.toggle`
  - Open on, close off → `cover.open`
  - Close on, open off → `cover.close`

**Issues:**

1. **Boot:**  
   After power-up, GPIO and pull-ups can stabilize in a way that one sensor (e.g. open) briefly appears “on” and the other “off.” Once the 500 ms filter allows it, **on_press** can fire and the script can run. If the condition “open on, close off” holds at that moment, the script sends **cover.open** → potential **uncommanded open on power-up**.

2. **Noise / bounce:**  
   Contact bounce or electrical noise can cause a transient “open on, close off” and again trigger **cover.open**.

3. **Both contacts on:**  
   Miswiring, both buttons pressed, or noise can make both sensors “on.” The script then runs **cover.toggle**. If the door was closed, that **opens** the door.

**Recommendations:**

- **Boot:** Delay eligibility for the dry-contact script for a short period after boot (e.g. 5–10 seconds), or require both sensors to be stable for a period before accepting open/close/toggle.
- **Logic:** Consider edge-triggered behavior (e.g. “open” only when the open contact **transitioned** to on) rather than level-based “open on and close off” at script run time, to reduce sensitivity to noise and “both on.”
- **Documentation:** Clearly document that dry contact “open” is equivalent to a physical open button and that wiring and environment can affect false triggers.

---

## 4. Low: API and Cover Control

**Mechanism:**  
The cover is a standard ESPHome cover. Any client that can reach the device and authenticate (e.g. with the device API password) can send:

- `cover.open`, or  
- `cover.set_position` with position 1.0 or any value in (0, 1) that yields open-direction in `door_move_to_position`.

**Notes:**

- There is **no** cover-specific or “open”-specific authentication in this codebase; security is by API password and network segmentation.
- Some example configs use `authorizer: none` under **esp32_improv** (provisioning), not under the main **api**; the main API may still require a password depending on user config.
- Best practice: strong API password, locked-down network, and awareness that anyone with API access can open/close the door.

---

## 5. Boot and Restore Behavior (Safe)

- **Cover restore:**  
  In `ratgdo_cover.cpp`, `setup()` calls `restore_state_()` and then `set_door_position(state.value().position)`. `set_door_position()` only updates internal `door_position`; it does **not** call `door_open()` or any protocol command. **Restore does not send open on boot.**

- **Sync (including dry contact):**  
  `sync()` calls `protocol_->set_open_limit()` / `set_close_limit()` with the current binary sensor state and updates door state via `received()`. It does **not** invoke the dry contact script or `door_action(OPEN)`.

- **Timeout in `door_open()`:**  
  The timeout that runs after sending open only updates internal state and may call `query_status()`; it does **not** send another open command.

---

## 6. Summary Table: Paths That Can Open the Door

| Path | Source | Condition | Uncommanded risk |
|------|--------|-----------|-------------------|
| Cover open | API / HA / dashboard | `position == 1.0` | Only if client sends it; see move-to-position bug for unknown position |
| Cover set position | API / HA / dashboard | `position in (0,1)` and `position > door_position` | **Yes** when `door_position` is unknown (e.g. after boot) |
| Dry contact script | GPIO `on_press` | Open on, close off | **Yes** at boot or due to noise/bounce |
| Dry contact script | GPIO `on_press` | Both on → toggle | **Yes** if door was closed (toggle opens) |
| Toggle button | User | Button press | No (user-initiated) |
| Protocol RX | Sec+ / dry contact | — | **None** (state only) |

---

## 7. Recommendations Checklist

1. **Firmware:** In `door_move_to_position()`, reject the move (and optionally log) when `door_position` is unknown (e.g. `DOOR_POSITION_UNKNOWN` or equivalent), and do not call `door_action(OPEN)` or `door_action(CLOSE)` in that case.
2. **Dry contact:** Add a boot delay or stability requirement before the dry contact script can trigger open/close/toggle; consider edge-triggered logic and document wiring/noise considerations.
3. **Documentation:** Document that the cover is controllable by any client with API access and that dry contact open is equivalent to a physical open button.
4. **Deployment:** Use a strong API password and restrict network access to the device where possible.

---

*This review is based on the repository code and YAML as of the review date. Actual ESPHome core behavior (e.g. when binary_sensor fires on_press at boot) may affect risk in practice.*
