# GASTROPOD Master Bible

> Merged from `bible_1.md`, `bible_2.md`, and `bible_3.md`
> Source chats: September 4–26, 2025
> Status: In progress — sections added incrementally

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [LSL Forbidden Rules](#2-lsl-forbidden-rules)
3. [Script Architecture](#3-script-architecture)
4. [Power System](#4-power-system)
5. [Movement System](#5-movement-system)
6. [Dock System](#6-dock-system)
7. [Security System](#7-security-system)
8. [All Code Blocks](#8-all-code-blocks)
9. [Bugs & Current Status](#9-bugs--current-status)
10. [Roadmap](#10-roadmap)

---

## 1. System Overview

### Project Identity

**Full Name:** G.A.S.T.R.O.P.O.D.
**Acronym:** Guided Autonomous Security Tactical Response Operations Drone
**Platform:** Second Life (Linden Scripting Language — LSL)
**Object:** Full-perm mechanical snail mesh object
**Display Names:** "::DisturbeD:: Sci-Fi Robot Attack Snail" / "DEATH SNAIL DRONE OF ULTIMATE SECURITY" / "G.A.S.T.R.O.P.O.D. mrk I" (and mrk II, III, IV, etc.)

---

### Purpose

GASTROPOD is a roaming security drone for Second Life land parcels. It autonomously patrols a pre-planned waypoint route, monitors for unknown or banned avatars, manages its own power supply via a dock charging system, and operates as a self-sustaining security unit with RP (roleplay) narrative flavor.

**Core capabilities:**
- Pre-planned waypoint patrol along owner-defined routes
- Power management with RP-themed drain and recharge simulation
- Avatar detection, scanning, and access control
- Dynamic dock system for emergency return and recharging
- Particle beam and visual effects (eye glow, hover underglow)
- Multi-drone fleet support (each drone independently named and paired)
- Admin command interface via a unique per-pair communication channel

---

### Component Ecosystem

The system is composed of multiple paired objects that communicate over a unique calculated channel:

| Component | Object Name | Role |
|-----------|-------------|------|
| **Drone** | "G.A.S.T.R.O.P.O.D. mrk I/II/III/IV" | Patrol and security unit; contains all drone scripts |
| **Dock** | "G.A.S.T.R.O.P.O.D. Dock" | Charging hub and communication relay; contains a Drone_Home prim for alignment |
| **Battery** | "Drone Battery" | Standalone visual prop; also designed for hot-swap charging (Phase 2) |
| **Console Panel** | (planned) | Multi-drone management display; listens on paired channel for status |
| **Pathway HUD** | "Pathway" (worn) | Owner-worn HUD for recording patrol routes and outputting "Tour" notecard data |

---

### Drone Script Contents

The drone linkset contains the following scripts in its root prim:

| Script | Current Version | One-Line Purpose |
|--------|----------------|-----------------|
| Configuration Script | v3.1.0 | Central hub: dock pairing, command dispatch, RP mode, patrol start/stop, charge coordination |
| Movement Script | v4.2.0 | Notecard reading, waypoint patrol, height adjustment, state persistence, emergency return |
| Power Script | v2.3.0 | Power drain/charge tracking, hover text display, emergency thresholds, charge broadcasting |
| Pairing Script | v1.2.0 (temporary) | Initial dock-to-drone pairing; deleted after successful pairing |

Child prim scripts (in linked sub-prims, not the root):
- **Eye script** — child prim named "Eye": glow effects and particle beam targeting
- **Hover script** — child prim named "Hover": underglow and teleport burst effects

---

### Autonomous Operating Cycle

The full intended loop (Phase 1, confirmed working):

```
Patrol → 25% power (coordinate update, continue patrol) → 5% power
→ Emergency TP home → Back into dock → Land
→ LANDING_COMPLETE → Begin charging (1 hour, or 15 min in test mode)
→ 100% power → CHARGING_COMPLETE → Auto-resume patrol
→ [Repeat indefinitely]
```

Key design notes:
- Power is always forced to exactly **1%** before emergency TP (RP theme: "just enough to return home")
- At 25% power, dock coordinates are silently updated in the background — drone continues patrol
- At 5% power, emergency TP is triggered using the most recently updated dock coordinates
- After charging, drone automatically resumes from the **saved waypoint index** (not from the start)
- Fleet operation: multiple drones (mrk I, II, III, IV) can operate simultaneously, each paired to its own dock

---

### Physical Setup Notes

- Drone object is **non-physical** (`STATUS_PHYSICS = FALSE`) at all times
- Drone rotation: uses **Z-axis only** for facing direction (snail mesh geometry)
- Dock contains a child prim named **Drone_Home** which defines the docking target position and orientation
- Drone_Home prim always uses `0, 0, Z` rotation format (no X or Y tilt)
- Physical Y-axis offset between drone center and dock center: **0.10075 m**

---

### Development History Summary

| Chat Batch | Sessions | Key Milestone |
|------------|----------|---------------|
| First 3 chats | Sept 4–8, 2025 | Core architecture: movement, power, dock design; movement script working |
| Second 3 chats | ~Sept 8–13, 2025 | Phase 1 complete: full autonomous cycle patrol→emergency return→charge→resume working |
| Last 3 chats | Sept 13–26, 2025 | Dock Config/Pairing script fixes, Pathway HUD, RP Mode system, Power Script overhaul |

---

*Section 1 of 10 complete*

---

## 2. LSL Forbidden Rules

These rules were established early and enforced throughout all development sessions. **Every GASTROPOD script must obey them without exception.**

---

### Core Forbidden Rules

**Rule 1 — No ternary operators**
LSL does not support the C-style ternary operator.
```lsl
// FORBIDDEN:
string name = (id == llGetOwner()) ? "Owner" : "Guest";

// REQUIRED:
string name;
if (id == llGetOwner())
{
    name = "Owner";
}
else
{
    name = "Guest";
}
```

---

**Rule 2 — No foreach loops**
LSL has no `foreach`. Always use traditional indexed `for` loops.
```lsl
// FORBIDDEN:
foreach (key id in myList) { ... }

// REQUIRED:
integer i;
for (i = 0; i < llGetListLength(myList); ++i)
{
    key id = llList2Key(myList, i);
    // ...
}
```

---

**Rule 3 — No global assignments from function calls**
Global variable declarations may only use literals or empty values. Assignment from functions must happen inside `state_entry` or other event handlers.
```lsl
// FORBIDDEN (at global scope):
string ownerName = llKey2Name(llGetOwner());
vector myPos = llGetPos();

// REQUIRED:
string ownerName;   // empty global declaration

default
{
    state_entry()
    {
        ownerName = llKey2Name(llGetOwner());  // assign inside event
    }
}
```

---

**Rule 4 — No `void` keyword**
LSL does not use C-style `void`. Functions that return nothing are declared with no return type.
```lsl
// FORBIDDEN:
void doSomething() { ... }

// REQUIRED:
doSomething() { ... }
```

---

**Rule 5 — No `PRIM_HOVER_HEIGHT` in `llSetPrimitiveParams`**
Must use the dedicated functions instead.
```lsl
// FORBIDDEN:
llSetPrimitiveParams([PRIM_HOVER_HEIGHT, 1.0]);

// REQUIRED:
llSetHoverHeight(1.0, FALSE, 0.0);
// or read with:
float h = llGetHoverHeight();
```

---

**Rule 6 — No runtime values in global scope**
Globals are only for config constants and empty placeholders. No function return values at global scope.

---

**Rule 7 — No invisible storage via object name, description, or notecards**
Persistent cross-script data must use `llLinksetData*`. Object name/description must not be repurposed as a data channel unless explicitly required by design.

---

**Rule 8 — No sneaky shorthand**
Always expand logic clearly. No collapsing of logic into unsupported operators, non-standard syntax, or shorthand assignments.

---

**Rule 9 — No reserved keywords as variable names**
The following are reserved LSL data types and cannot be used as variable names:
`state`, `rotation`, `vector`, `integer`, `float`, `string`, `key`, `list`

```lsl
// FORBIDDEN:
rotation rotation = llGetRot();    // "rotation" is a type name
integer state = 1;                 // "state" is a keyword

// REQUIRED:
rotation rot_angle = llGetRot();
integer stateValue = 1;
```

---

### Additional LSL Scope Rules (Discovered During Development)

These were confirmed through trial and error:

- Global variables can **only** be accessed directly within event handlers (`state_entry`, `timer`, `touch_start`, etc.)
- User-defined functions **cannot** access global variables — they can only use parameters passed to them and return values
- Variables declared inside blocks (`if`/`else`) are not accessible outside those blocks
- Functions must be declared **before** the `default` state block, after global variable declarations
- All functions must have explicit return types (or use the no-type pattern for void-equivalent)

---

### Functions and Constants That Do NOT Exist in LSL

Confirmed absent — do not use:

| Does Not Exist | Notes |
|---------------|-------|
| `llSlerp()` | No such function; use `llEuler2Rot()` for rotation |
| `at_target()` event | No such event; use timer-based distance check |
| `not_at_target()` event | No such event |
| `llGetTimerEvent()` | No such function |
| `llKeyframedMotion()` | Wrong name; correct is `llSetKeyframedMotion()` |

---

### Functions and Constants That DO Exist (Confirmed)

| Function/Constant | Notes |
|------------------|-------|
| `llSetKeyframedMotion(list keyframes, list options)` | Correct name for keyframed movement |
| `KFM_DATA`, `KFM_TRANSLATION`, `KFM_MODE`, `KFM_FORWARD` | All valid constants |
| `llGetTimeOfDay()` | Returns `float` — must cast to string before concatenation |

---

### Script Header Format (Standard)

All scripts must use this versioned header:
```lsl
// Script Name vX.Y.Z
// [One-line purpose description]
// Enhancements: [list new features added]
// Modifications: [list code changed/moved/adjusted]
// Cleaned: [list features removed]
// Fixed: [list broken features corrected]
// PHASE X: [current development phase]
```

Sections within scripts are marked with `//[SECTION NAME]` comments.

---

*Section 2 of 10 complete*

---

## 3. Script Architecture

### Drone Root Prim — Core Scripts

All scripts below reside in the **root prim** of the drone linkset:

| Script | Latest Version | Responsibility |
|--------|---------------|----------------|
| **Configuration Script** | v3.1.0 | Central coordination hub: dock pairing data management, notecard loading coordination, unique channel management, admin command dispatch, RP mode control, patrol start/stop, charge broadcasting |
| **Movement Script** | v4.2.0 | Notecard reading, waypoint patrol, smooth KFM movement, height adjustment, rise/lower sequences, state persistence and resume, emergency return execution, notecard change detection |
| **Power Script** | v2.3.0 | Hybrid power drain tracking (time + distance), charging cycle management, hover text display with unicode power bar, emergency threshold detection, charge broadcasting, battery swap communication prep |
| **Pairing Script** | v1.2.0 | Initial dock-to-drone pairing only; calculates unique `comm_channel`; stores pairing data; **temporary** — designed to delete itself after pairing (self-delete disabled in test builds) |

> **Note on 9th script:** The original design called for 9 scripts. The Pairing Script was intended as temporary, the Security, Scan, Diagnostics, and Communications scripts are planned but not yet built. The current functional system uses 4 drone scripts.

---

### Drone Child Prim Scripts

These scripts reside in named **child prims** of the drone linkset (not the root prim):

| Child Prim Name | Script | Responsibility |
|----------------|--------|----------------|
| `Eye` | Eye script | Glow effects and particle beam targeting |
| `Hover` | Hover script | Underglow and teleport burst effects |

---

### Dock Object Scripts

The dock is a **separate object** from the drone. It communicates over the calculated `comm_channel`.

| Script | Latest Version | Responsibility |
|--------|---------------|----------------|
| **Dock Config Script** | v1.5.0 | Receives coordinate requests from drone; broadcasts current Drone_Home prim position and rotation; message filtering (ignores non-dock messages); RP_MODE awareness; paired-touch channel display |
| **Dock Pairing Script** | v1.4.0 | Owner initiates pairing via touch; discovers nearby drone; calculates unique `comm_channel`; saves pairing data to linkset; admin reset/status commands via calculated channel |

---

### HUD Object Scripts

The HUD is a **worn attachment** for the drone owner/builder:

| Script | Version | Location | Responsibility |
|--------|---------|----------|----------------|
| **Pathway (root)** | v1.0+ | Root prim of HUD | Full waypoint recording: route name, RP mode, rotation axis, start/waypoint/end recording, output generation, resume beacon, minimize/maximize toggle, light system |

---

### Peripheral / Proof-of-Concept Scripts

| Script | Version | Object | Responsibility |
|--------|---------|--------|----------------|
| **Battery Visual Charging** | v2.2 | "Drone Battery" | 10-segment visual battery prop: bidirectional charging wave effect (bottom-up fill / top-down drain), status indicator prims named "charge" |

---

### Script Communication Architecture

**Internal (drone scripts → each other):**
- `llMessageLinked(LINK_THIS, num, str, id)` — all inter-script communication within the drone linkset
- No script directly calls another; all coordination is event-driven via link messages

**External (drone ↔ dock):**
- `llRegionSay(comm_channel, message)` — drone → dock
- `llRegionSay(comm_channel, message)` — dock → drone (region-wide, not distance-limited)
- `comm_channel` is unique per drone-dock pair, calculated during pairing

**RP emotes:**
- `llSay(0, "/me ...")` — used for RP_MODE output to local public chat

---

### Prim Structure — Dock

The dock linkset contains a named child prim:

| Prim Name | Purpose |
|-----------|---------|
| `Drone_Home` | Defines the exact docking target position and orientation. Always uses `0, 0, Z` rotation (Z-axis only). The drone aligns to this prim's position + 0.10075m offset. |

---

### Memory Management

The multi-script architecture was mandated by LSL's 64KB per-script memory limit:
- A single-script approach caused Stack-Heap Collision errors
- Splitting into modular scripts keeps each well under the limit
- Trade-off: some inter-script communication latency accepted as necessary

**llLinksetData storage:**
- 64KB total per linkset (shared across ALL prims — adding prims does NOT increase storage)
- Estimated capacity: ~1000+ UUIDs within the 64KB limit
- Used for: pairing data, persistent state, power level, debug flags

---

### Version History Summary

| Script | Bible 1 State | Bible 2 State | Bible 3 State |
|--------|--------------|--------------|--------------|
| Configuration | Monolithic (pre-split) → modular design | v2.1.0 → v3.0.0 | v3.0.0 → v3.1.0 |
| Movement | Single script (v2.0.1 working) | v3.1.0 → v4.1.0 | v4.1.0 → v4.2.0 |
| Power | Designed, partial | v1.4.0 → v2.0.0 | v2.0.0 → v2.3.0 |
| Pairing | Designed | v1.2.0 (working) | v1.2.0 → v1.4.0 (dock side) |
| Dock Config | Designed | Initial impl | v1.2.0 → v1.5.0 |
| Battery Visual | Designed | v1.0 → v2.2 | Referenced, not changed |
| Waypoint HUD | Designed | v1.0 → v1.1.1 | Full Pathway HUD rewrite |

---

*Section 3 of 10 complete*
