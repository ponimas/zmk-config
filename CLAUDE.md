# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

ZMK firmware configuration for the [TOTEM](https://github.com/GEIGEIGEIST/TOTEM) split keyboard — a 38-key wireless keyboard using Seeeduino XIAO BLE controllers. Builds are done via GitHub Actions (push triggers a build automatically).

## Building

There is no local build setup. Firmware is built by pushing to GitHub, which triggers the workflow at `.github/workflows/build.yml`. The workflow delegates to `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`.

The build matrix is defined in `build.yaml`:
- `totem_left` — with ZMK Studio support (`studio-rpc-usb-uart` snippet, `CONFIG_ZMK_STUDIO=y`, `CONFIG_ZMK_STUDIO_LOCKING=n`)
- `totem_right`
- `settings_reset` — for resetting paired BT devices

## File Layout

Two parallel keymap files exist with different levels of customization:

| File | Purpose |
|------|---------|
| `config/totem.keymap` | Primary keymap (used by build) |
| `boards/shields/totem/totem.keymap` | Shield-level keymap (older/reference) |

The `config/` directory is the active configuration; `boards/shields/totem/` contains the shield hardware definition.

**Shield hardware files:**
- `totem.dtsi` — key matrix transform (10 columns × 4 rows), kscan GPIO config
- `totem-layouts.dtsi` — physical layout definition
- `totem_left.overlay` / `totem_right.overlay` — per-side column GPIO pin assignments
- `Kconfig.defconfig` / `Kconfig.shield` — Kconfig shield declarations
- `totem.zmk.yml` — ZMK shield metadata (requires `seeeduino_xiao_ble`)

## Keymap Architecture

The keymap uses 4 layers (defined in `config/totem.keymap`):

- `BASE` (0) — QWERTY with mod-tap on home row modifiers (LCTRL/RCTRL on A/;), shift on outer keys
- `NAV` (1) — Arrow keys, brackets, numpad — activated by hold on space
- `SYM` (2) — Symbols, German umlauts (via `RA()` combos), media keys — activated by hold on enter
- `ADJ` (3) — System reset, bootloader, BT management, F-keys — accessed by simultaneous NAV+SYM hold

**Behavior config** (`&mt` override):
```
quick-tap-ms = <100>;
global-quick-tap;
flavor = "tap-preferred";
tapping-term-ms = <170>;
```

**Combo:** positions 0+1 → ESC

**Macro:** `gif` — types `@gif`

## Key Matrix

The TOTEM has 38 keys laid out as:
- 3 rows × 5 columns per side + 1 outer key per side on row 3
- 3 thumb keys per side
- Matrix uses `col2row` diode direction
- Left side columns: XIAO pins D4, D5, D10, D9, D8
- Right side columns: same pin numbers mirrored
- Rows (both sides): XIAO pins D0, D1, D2, D3

<!-- GSD:project-start source:PROJECT.md -->
## Project

**ZMK Config — Keymap Diagram CI**

ZMK firmware configuration for the TOTEM 38-key split keyboard. This project extends the existing build pipeline to automatically generate a visual keymap diagram (via keymap-drawer) on every push and commit it back to README.md at the top — so the readme always shows a live, up-to-date picture of all keyboard layers.

**Core Value:** The README always shows an accurate, auto-generated SVG diagram of all keymap layers after every push — no manual steps required.

### Constraints

- **CI**: Must work within GitHub Actions free tier
- **Permissions**: Workflow needs `contents: write` to push README changes back
- **Compatibility**: Must not break the existing firmware build job
- **No secrets**: No PAT required if using `GITHUB_TOKEN` with write permissions
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Languages
- DeviceTree Source (`.dtsi`, `.overlay`) - Hardware and keymap configuration
- Kconfig (`.conf`, `Kconfig.*`) - Firmware feature flags and build options
- YAML (`.yml`, `.yaml`) - Build matrix and module/manifest configuration
- C (via ZMK firmware internals, referenced through `#include` directives in `.keymap` and `.dtsi` files)
## Runtime
- Embedded bare-metal on Nordic nRF52840 (via Seeeduino XIAO BLE)
- Zephyr RTOS (pulled in via ZMK `west.yml` manifest at `v0.3`)
- West (Zephyr's meta-tool) — version pinned to `v0.3` via `config/west.yml`
- No lockfile — revision is pinned by string in manifest
## Frameworks
- ZMK Firmware `v0.3` — keyboard firmware framework for wireless keyboards
- Zephyr RTOS — pulled in transitively via ZMK's `app/west.yml`
- GitHub Actions — CI/CD build system; no local toolchain setup
- CMake — used internally by ZMK/Zephyr build system; invoked via `cmake-args` in `build.yaml`
## Key Dependencies
- ZMK `v0.3` — entire keyboard firmware; provides `behaviors.dtsi`, BT bindings, key bindings, combos, macros, ZMK Studio RPC support
- Seeeduino XIAO BLE board definition — provides `xiao_d` GPIO headers used in `totem.dtsi` and overlays; required by `totem.zmk.yml`
- `zmk,kscan-gpio-matrix` — key scanning via GPIO matrix (`boards/shields/totem/totem.dtsi`)
- `zmk,matrix-transform` — key position mapping (`boards/shields/totem/totem.dtsi`)
- `zmk,keymap` — layer/binding definitions (`config/totem.keymap`)
- `zmk,combos` — combo key (ESC on positions 0+1) (`config/totem.keymap`)
- `zmk,behavior-macro` — `gif` macro (`config/totem.keymap`)
- `zmk,physical-layout` — physical key layout (`boards/shields/totem/totem-layouts.dtsi`)
## Configuration
- `CONFIG_ZMK_USB_LOGGING=n` — USB logging disabled
- RGB underglow: disabled (commented out)
- OLED display: disabled (commented out)
- `totem_left` — includes ZMK Studio (`studio-rpc-usb-uart` snippet, `CONFIG_ZMK_STUDIO=y`, `CONFIG_ZMK_STUDIO_LOCKING=n`)
- `totem_right` — standard build, no Studio
- `settings_reset` — BT pairing reset utility build
- `ZMK_KEYBOARD_NAME` defaults to `"TOTEM"` when left shield active
- `ZMK_SPLIT_ROLE_CENTRAL` defaults to `y` for left side
- `ZMK_SPLIT` defaults to `y` for both sides
- `board_root: .` — registers project root as board/shield search path
## Platform Requirements
- No local toolchain required — all builds run in GitHub Actions
- Git push to any branch triggers build via `.github/workflows/build.yml`
- Firmware artifacts downloaded from GitHub Actions run artifacts
- Target MCU: Nordic nRF52840 (on Seeeduino XIAO BLE)
- Wireless protocol: Bluetooth Low Energy (BLE) split keyboard
- Flash method: UF2 bootloader drag-and-drop or `west flash`
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## Language Context
- **Devicetree Source (`.dtsi`, `.overlay`)** — hardware description and keymap binding declarations
- **ZMK Keymap (`.keymap`)** — keymap layer definitions (C preprocessor + Devicetree syntax)
- **Kconfig (`.conf`, `Kconfig.*`)** — firmware feature flags
- **YAML (`.yml`, `.yaml`)** — build matrix and ZMK shield metadata
## Naming Patterns
- Shield hardware files use `totem.` prefix: `totem.dtsi`, `totem-layouts.dtsi`, `totem.conf`, `totem.zmk.yml`
- Per-side files use `totem_left` / `totem_right` suffix: `totem_left.overlay`, `totem_right.overlay`, `totem_left.conf`
- Kconfig files use Zephyr convention: `Kconfig.defconfig`, `Kconfig.shield`
- SCREAMING_SNAKE_CASE, single word or abbreviation
- Example from `config/totem.keymap` and `boards/shields/totem/totem.keymap`:
- Numbers are right-aligned with spaces for visual alignment
- `snake_case` with descriptive suffixes: `base_layer`, `nav_layer`, `sim_layer`, `adjust_layer`
- Hardware nodes use `snake_case` with numeric suffix where needed: `kscan0`, `kscan_0`
- Labels mirror node names: `kscan0: kscan_0`
- Short all-caps abbreviations, max 4–5 characters: `"BASE"`, `"NAVI"`, `"SYM"`, `"ADJ"`
- `combo_` prefix + descriptive name: `combo_esc`
- Short lowercase identifiers matching their function: `gif`
- `SHIELD_TOTEM_LEFT`, `SHIELD_TOTEM_RIGHT` — `SHIELD_` prefix + `TOTEM_` infix + side suffix
## Code Style
- Each layer's `bindings = <...>` block is formatted as a visual grid, using spaces to align columns
- Left side (5 cols) and right side (5 cols) separated by double-space gap
- Row-outer keys (thumb/outer column) indented to the left of the 10-column block
- Example from `config/totem.keymap`:
- Multi-value properties use comma-separated lines with leading `,` on continuation:
- Devicetree files use 4-space indentation
- The older shield keymap (`boards/shields/totem/totem.keymap`) uses inconsistent indentation (mixed 2/4 spaces, sometimes inline with comments)
- The active config keymap (`config/totem.keymap`) uses consistent 4-space indentation with proper nesting
- Block separators between layers use full-width Unicode block-drawing lines:
- ASCII-art box-drawing comments document the physical layout of each layer above its bindings:
- Header comments use `//` in `.keymap` files
- Hardware files (`*.dtsi`, `*.overlay`) use `/* ... */` block comment for copyright header
- Declared at the top level before the `/` root node, outside `/ { ... }`
- Example from `config/totem.keymap`:
- The shield-level version uses multi-line style with additional properties (`global-quick-tap`, `flavor`, `tapping-term-ms`)
- `&lt LAYER KEY` — layer-tap (hold for layer, tap for key): used on thumb cluster
- `&mo LAYER` — momentary layer: used inside layers to reach ADJ
- `&mt MOD KEY` — mod-tap (hold for modifier, tap for key): used on home row and outer keys
- `&trans` used for all unassigned positions in non-base layers (not `&none`)
## Import Organization
## Kconfig Conventions
- Feature flags default to `y` or `n` using `default` keyword (not `=y`)
- Side-specific config guarded with `if SHIELD_TOTEM_LEFT` / `if SHIELD_TOTEM_RIGHT`
- Shared split config guarded with `if SHIELD_TOTEM_LEFT || SHIELD_TOTEM_RIGHT`
- `.conf` files in `boards/shields/totem/` are empty (present for ZMK scaffolding, not for user config)
- User config in `config/totem.conf`; disabled options shown as commented lines
## Build Matrix
- `build.yaml` uses `include:` list (not top-level `board:`/`shield:` arrays) to allow per-target cmake-args and snippets
- ZMK Studio enabled only for `totem_left` via `cmake-args: -DCONFIG_ZMK_STUDIO=y -DCONFIG_ZMK_STUDIO_LOCKING=n`
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## Pattern Overview
- No application code: all behavior is expressed as Devicetree (`.dtsi`, `.overlay`) and Kconfig (`.conf`) declarations consumed by the ZMK firmware build system
- Two-tier structure: `boards/shields/totem/` defines hardware; `config/` defines user keymap and build settings
- CI-only builds: no local toolchain — all compilation is delegated to GitHub Actions via `zmkfirmware/zmk`'s reusable workflow
## Layers
- Purpose: Describe physical key matrix, GPIO pins, and ZMK Studio physical layout metadata
- Location: `boards/shields/totem/`
- Contains: `totem.dtsi` (kscan + matrix transform), `totem-layouts.dtsi` (physical key positions), per-side overlays, Kconfig declarations, ZMK shield metadata
- Depends on: ZMK firmware internals (`zmk,kscan-gpio-matrix`, `zmk,matrix-transform`, `zmk,physical-layout`)
- Used by: Build system when composing a firmware image for a specific board+shield combination
- Purpose: Define keymap layers, behaviors, combos, and macros; set Kconfig options
- Location: `config/`
- Contains: `totem.keymap` (all keymap logic), `totem.conf` (Kconfig overrides), `west.yml` (ZMK dependency pinning)
- Depends on: Hardware definition layer via ZMK binding APIs (`&mt`, `&lt`, `&kp`, `&bt`)
- Used by: ZMK build system at compile time
- Purpose: Translate the user config into firmware artifacts
- Location: `build.yaml`, `.github/workflows/build.yml`
- Contains: Build matrix (board+shield+snippet+cmake-args per target), GitHub Actions workflow delegation
- Depends on: `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`
- Used by: GitHub Actions on every push/PR/manual dispatch
## Data Flow
- `BASE` (0): always active at bottom of stack
- `NAV` (1): activated by hold on left thumb space key (`&lt NAV SPACE`)
- `SYM` (2): activated by hold on right thumb enter key (`&lt SYM RET`)
- `ADJ` (3): accessed via `&mo ADJ` bound on both NAV and SYM layers' inner thumb keys — requires simultaneous hold of both
## Key Abstractions
- Purpose: Encapsulate hardware-specific GPIO wiring and key matrix so the board (Seeeduino XIAO BLE) stays generic
- Files: `boards/shields/totem/totem.dtsi`, `totem_left.overlay`, `totem_right.overlay`, `totem.zmk.yml`
- Pattern: Left overlay adds col-gpios in natural order; right overlay reverses them and sets `col-offset = <5>` to address the second half of the 10-column matrix transform
- Purpose: Map physical RC(row,col) scan coordinates to a flat logical key index used by the keymap
- File: `boards/shields/totem/totem.dtsi`
- Pattern: 10 columns × 4 rows; outer keys on row 3 (positions RC(3,0) and RC(3,9)) and inner thumb row; 38 total keys
- Purpose: Define what each key does per active layer
- File: `config/totem.keymap`
- Pattern: Layers declared as child nodes of `keymap { compatible = "zmk,keymap"; }` using `bindings = <...>` arrays
- Purpose: Reusable key actions (`&mt` mod-tap, `&lt` layer-tap, `&kp` key press, `&mo` momentary layer)
- File: `config/totem.keymap` overrides `&mt` timing; ZMK built-ins provide the rest
- Pattern: `&mt MODIFIER KEY` — tap for key, hold for modifier. Configured globally: `quick-tap-ms = <100>`, `flavor = "tap-preferred"`, `tapping-term-ms = <170>`
- Purpose: Multi-key simultaneous presses mapped to a single action
- File: `config/totem.keymap` under `combos {}` node
- Pattern: `key-positions = <0 1>` → `&kp ESC` with `timeout-ms = <50>`
- Purpose: Multi-step key sequences triggered by a single binding
- File: `config/totem.keymap` under `macros {}` node
- Pattern: `gif` macro presses LSHFT+2 then releases LSHFT, then taps G I F (producing `@gif`)
## Entry Points
- Location: `boards/shields/totem/totem_left.overlay` (hardware), `config/totem.keymap` (behavior)
- Triggers: `build.yaml` entry `shield: totem_left`
- Responsibilities: Central role (`ZMK_SPLIT_ROLE_CENTRAL=y`), BLE host connection, ZMK Studio USB UART support
- Location: `boards/shields/totem/totem_right.overlay`
- Triggers: `build.yaml` entry `shield: totem_right`
- Responsibilities: Peripheral split role, scans right-side columns with `col-offset = <5>`
- Location: ZMK built-in `settings_reset` shield
- Triggers: `build.yaml` entry `shield: settings_reset`
- Responsibilities: Clears paired BT devices when flashed
## Error Handling
- Invalid keymap bindings cause compile-time errors in ZMK's Zephyr build
- Hardware GPIO mismatches surface as boot-time scan failures on device
## Cross-Cutting Concerns
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
