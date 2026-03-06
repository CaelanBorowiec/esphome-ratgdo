# How to compile, bundle, and flash the firmware

This project is **ESPHome**-based. You can build and flash with your local changes (including the security updates) either **locally** with the ESPHome CLI or via **GitHub Actions** (CI). Flashing can be done over **USB** or **OTA** (Wi‑Fi).

---

## Prerequisites

- **Python 3.9+** (3.11 recommended)
- **ESPHome CLI**: `pip install esphome`
- For **USB flashing**: a USB cable and the correct port (e.g. `COM3` on Windows, `/dev/ttyUSB0` on Linux)
- For **OTA**: device and computer on the same network; device already has WiFi/API configured (e.g. from a previous flash)

---

## Option 1: Build and flash locally (with your repo changes)

Use this when you have edited `components/`, `base.yaml`, or `base_secplusv1.yaml` and want to compile and flash from your machine.

### 1. Use local components and local base

From the **repository root**:

```bash
python scripts/update_refs_for_ci.py
```

This script:

- Switches **external_components** in `base.yaml` (and related base files) from the git repo to **local** `components/`, so your C++ and YAML changes are used.
- In root board YAMLs (e.g. `v25iboard.yaml`), replaces the remote package with **local** `!include <path>/base.yaml`.

**Note:** This **modifies** YAML files in the repo. To revert after building:  
`git checkout -- base.yaml base_secplusv1.yaml base_drycontact.yaml *.yaml`  
(or restore the files you changed).

### 2. Choose a board config

Pick the YAML that matches your hardware (board + protocol). Root-level examples:

| Board / protocol              | Config file (in repo root)        |
|------------------------------|-----------------------------------|
| V2.5i Security+ 2.0          | `v25iboard.yaml`                  |
| V2.5i Security+ 1.0          | `v25iboard_secplusv1.yaml`       |
| V2.5i Dry contact            | `v25iboard_drycontact.yaml`      |
| V2.5 ESP8266 D1 Mini Sec+ 2.0| `v25board_esp8266_d1_mini.yaml`  |
| V3.2 board Sec+ 2.0          | `v32board.yaml`                   |
| …                            | (see repo root and `static/` for more) |

If your board config lives only in `static/`, copy it to the repo root first (or run the script in a way that processes `static/`; the script currently only updates `*.yaml` in the root).

### 3. Compile

From the **repository root**:

```bash
esphome compile v25iboard.yaml
```

Replace `v25iboard.yaml` with your chosen config. The first compile can take a few minutes (dependencies and toolchains). Output is under `.esphome/build/` and includes the firmware `.bin`.

### 4. Flash

**Over USB (new install or wired update):**

```bash
esphome run v25iboard.yaml
```

This compiles (if needed) and then prompts for the port. You can also compile then upload in one step:

```bash
esphome upload v25iboard.yaml
```

Select the correct port when asked (e.g. `COM3`, `/dev/ttyUSB0`).

**Over OTA (device already on WiFi):**

```bash
esphome upload v25iboard.yaml --device <hostname-or-ip>
```

Example: `--device ratgdov25i.local` or `--device 192.168.1.50`.

**Single command (compile + upload in one go):**

```bash
esphome run v25iboard.yaml --device ratgdov25i.local
```

---

## Option 2: Build via GitHub Actions (CI)

Use this to build **all** board variants with your branch’s code (including security updates) and download pre-built firmware.

1. **Push your branch** (with the security fixes and any other changes) to GitHub.
2. Open **Actions** on the repo and run or find the **Build** workflow for your branch.
3. After the run completes, open the workflow run and **download the artifact(s)** for the board(s) you need (e.g. “V2.5i Board Security+ 2.0”). Each artifact contains the firmware and manifest for that variant.
4. **Flash** using one of:
   - **Web Installer**: [ratgdo.github.io/esphome-ratgdo](https://ratgdo.github.io/esphome-ratgdo/) (for new installs / Improv; use the built `.bin` if you have a way to feed it).
   - **ESPHome Dashboard**: Add device → “Install” → “Upload existing binary” and select the `.bin` from the artifact.
   - **ESPHome CLI**:  
     `esphome upload <path-to-downloaded-config>.yaml --device <hostname-or-ip>`  
     (use the config that matches the artifact, or a minimal config that matches the build).

On **non‑main** branches, the CI script updates refs so the build uses **local components and base** from that branch; you don’t need to run `update_refs_for_ci.py` yourself for CI.

---

## Quick reference

| Goal                         | Command / step |
|-----------------------------|----------------|
| Use local components + base | `python scripts/update_refs_for_ci.py` (from repo root) |
| Compile one board           | `esphome compile <board>.yaml` |
| Compile + USB flash         | `esphome run <board>.yaml` (choose port when prompted) |
| Compile + OTA flash         | `esphome run <board>.yaml --device <hostname-or-ip>` |
| Revert script changes       | `git checkout -- base.yaml base_secplusv1.yaml base_drycontact.yaml *.yaml` |
| Build all boards (no local) | Push branch → Actions → Build → download artifacts |

---

## Troubleshooting

- **“No such file or directory” for base.yaml**  
  Run `update_refs_for_ci.py` from the **repository root** so the script can resolve paths and update the right files.

- **Wrong board or protocol**  
  Pick the YAML that matches your hardware (see table in § “Choose a board config”). Configs in `static/` are the same as the root copies used by CI; use the root copy if the script only touches root.

- **ESPHome version**  
  The project sets `esphome: min_version: 2026.2.0` in the base. Install a matching or newer ESPHome:  
  `pip install "esphome>=2026.2.0"` or use the version from the project’s CI.

- **USB port not found**  
  Install the correct USB‑serial driver for your board (e.g. CP210x, CH340). On Windows, check Device Manager for the COM port.

- **OTA fails**  
  Ensure the device is powered and on the same network; use the device hostname (e.g. `ratgdov25i.local`) or its IP. If the device was never configured for WiFi, flash once over USB with a config that includes WiFi/API.
