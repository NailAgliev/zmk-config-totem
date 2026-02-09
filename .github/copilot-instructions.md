# ZMK Totem Keyboard Configuration Guide

## Project Overview

This is a **ZMK firmware configuration** for the TOTEM split keyboard (38 keys, 4 rows × 10 columns). The TOTEM is a compact column-staggered split keyboard powered by SEEEDUINO XIAO BLE or RP2040 microcontrollers. Builds are automated via GitHub Actions, which compiles firmware for left/right halves separately.

## Layout: Hands Down Vibranium (Neu-vv) with P↔F Swap

### Core Design Ideas

**Hands Down philosophy prioritizes whole-hand ergonomics over isolated metrics:**

- **High Alternation** — Hands ping-pong back and forth, creating smooth, rhythmic flow and preventing same-hand fatigue
- **High Rolling** — Continuous rolling motions (especially in→out) minimize redirects; out-rolling is gentler on ring/pinky than inward
- **Extremely Low SFBs** (~0.725%) — Same-finger bigrams cause stuttering and tension; Vibranium achieves this without sacrificing rolling
- **Graduated Hand Burden** — Load distribution from pinky (lightest) → middle (heaviest) → index, respecting hand anatomy
- **Whole-Hand Rhythm** — Considers stretches, palm dislocations, shoulder impact, and inter-hand coordination—not just individual finger statistics

### Design Decisions for TOTEM Split

- **R on Thumb** — Enables the steady rhythm characteristic of Vibranium; critical for split ergo keyboards
- **Low Center Column Usage** — Reduces the "awkward reach" penalty on split boards
- **Custom P↔F Swap** — Optimization for the project's specific typing corpus; trades off one SFB pair for better rolling patterns elsewhere
- **Balanced Trade-offs** — Unlike layouts optimizing for single metrics, Hands Down resists extremes (e.g., ultra-low SFBs at the cost of redirects)

## Architecture & Key Concepts

### Hardware Configuration (Device Tree)

- **[totem.dtsi](config/boards/shields/totem/totem.dtsi)** - Shared hardware definition
  - Defines 4×10 matrix layout with `matrix-transform`
  - GPIO row/column scanning (4 rows, col2row diode direction)
  - Key position mapping (0-37 system, position 0 = top-left key)

- **[totem_left.overlay](config/boards/shields/totem/totem_left.overlay)** - Left half specifics
  - GPIO pin assignments for 5 columns (xiao_d pins: 4,5,10,9,8)

- **[totem_right.overlay](config/boards/shields/totem/totem_right.overlay)** - Right half specifics
  - GPIO pins in reversed order (8,9,10,5,4) due to pin layout
  - `col-offset = <5>` mirrors the matrix on firmware side (logical columns 5-9 map to physical left's 0-4)

**Key principle**: Both halves share keymap/behavior code but have different GPIO mappings. The right overlay's `col-offset` handles the mirroring mathematically.

### Keymap Structure ([totem.keymap](config/totem.keymap))

**Layers** (935 lines total):
```
#define BASE 0  // Base typing layer
#define NAV  1  // Navigation/cursor layer  
#define SYM  2  // Symbol layer
#define ADJ  3  // Adjustment/settings layer
```

**Three main sections**:
1. **Combos** - Multi-key press triggers (e.g., `key-positions = <10 11 12>` → Sch ligature)
   - Set `timeout-ms` and `slow-release` for reliable detection
   - Positions are 0-indexed from the matrix (see dtsi)

2. **Behaviors** - Reusable key actions (hold-tap, mod-morph, adaptive-key)
   - `HMR_L`/`HMR_R` - Hold-tap behaviors with hold-trigger ranges to prevent misfires
   - `mod-morph` - Single key outputs different symbols when shifted (e.g., `.` → `:` on shift)
   - `adaptive-key` - Context-aware bindings based on previous key (e.g., `H` → `U` after `A`)
   - Commented-out experimental behaviors show intent; common patterns for future reference

3. **Keymap data** - 4  layer definitions, each with positions 0-37

**Behavior tuning parameters** (critical for typing feel):
- `tapping-term-ms` - Hold duration before registering as hold (typically 170-200ms)
- `quick-tap-ms` - Repeat presses register immediately (55-100ms)
- `flavor` - Tap vs. hold priority behavior; usually "balanced" or "tap-preferred"
- `hold-trigger-key-positions` - Hand-side detection; prevents accidental combinations

### Build & Firmware

- **build.yaml** - Simple GitHub Actions trigger that defers to ZMK's standard CI
- **Outputs** - Two .uf2 files after build: `totem_left` and `totem_right`
- **Flash method** - Reset device twice → appears as USB drive → drag-drop .uf2 file

## Project-Specific Patterns

### Custom Behavior Naming Convention

Behaviors follow patterns for clarity:
- `HMR_L`/`HMR_R` - Home-row modifiers (Left/Right)
- `ak_*` - Adaptive-key behaviors (context-aware)
- `lk_*` - Hold-tap behaviors (linger-key patterns like "th", "sh", "gh")
- `*Pair` - Paired bracket/brace macros with auto-insertion

### Combo Position Reference

The 38-key TOTEM layout maps as:
```
Row 0:  0  1  2  3  4  |  5  6  7  8  9      (number row)
Row 1:  10 11 12 13 14 | 15 16 17 18 19     (home row)
Row 2:  20 21 22 23 24 | 25 26 27 28 29 30  (lower row, 20 & 30 outer)
Row 3:        32 33 34 | 35 36 37           (thumb cluster, 20 left outer, 31 right outer)
```

**Common combos**:
- `<10 11 12>` - S+C+N (left middle area) → **Sch**
- `<12 13>` - N+T (right of middle) → **Th**
- `<11 12>` - C+N (left middle) → **Ch**
- `<1 2>` - W+M (top row outer) → **Wh**
- `<30 29>` - P+Y (right lower row) → **Ph**
- `<1 3>` - W+G (top row skipped) → **Qu**

### Macro Syntax

Macros are defined inline in behaviors. Example:
```
macro_Sch {
    compatible = "zmk,behavior-macro";
    #binding-cells = <0>;
    bindings = <&macro_press &mo>, <&kp S C H>, <&macro_release>;
    label = "SCH";
};
```

Avoid long macro strings; bind via hold-tap behaviors instead for better responsiveness.

## Common Development Tasks

### Modify a Single Key

Edit [totem.keymap](config/totem.keymap) → find layer (e.g., BASE keymap block) → replace binding. Push to GitHub → check "Actions" tab → download `.uf2` from latest build.

### Add a New Combo

1. Add to `combos` section with `key-positions`, `bindings`, `timeout-ms`
2. Reference an existing behavior or create a new macro
3. Test `timeout-ms` gradually (45-60ms typical); lower = stricter timing

### Tune Behavior Timing

Edit `tapping-term-ms` in behavior definition. For slow typists, increase to 200-220ms; for fast, reduce to 150ms. Test on device before pushing.

### Add a New Layer

1. Add `#define LAYER_NAME N` near top
2. Create keymap block in keymap data
3. Add layer-switch keys (e.g., `&mo LAYER_NAME`) to existing layers

### Flash Left/Right Half

After build completes:
1. Connect left via USB + press reset twice → copy `totem_left-seeeduino_xiao_ble-zmk.uf2`
2. Connect right + press reset twice → copy `totem_right-seeeduino_xiao_ble-zmk.uf2`

Device recognition requires proper GPIO wiring per [Kconfig.shield](config/boards/shields/totem/Kconfig.shield).

## Critical Files Checklist

| File | Purpose |
|------|---------|
| [totem.keymap](config/totem.keymap) | Layers, combos, behaviors, macros |
| [totem.dtsi](config/boards/shields/totem/totem.dtsi) | Matrix layout, GPIO scanning |
| [totem_left.overlay](config/boards/shields/totem/totem_left.overlay) | Left GPIO pins |
| [totem_right.overlay](config/boards/shields/totem/totem_right.overlay) | Right GPIO pins + col-offset |
| [totem.conf](config/totem.conf) | ZMK feature flags |
| [build.yaml](build.yaml) | CI trigger |

## External References

- [ZMK Documentation](https://zmk.dev/docs) - Keycodes, behaviors, device tree
- [TOTEM Hardware Repo](https://github.com/GEIGEIGEIST/totem) - PCB/wiring details
- [ZMK Behaviors](https://zmk.dev/docs/behaviors) - Hold-tap, combos, macros reference
