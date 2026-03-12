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

---

## 4. Power System

### Design Philosophy

The power system uses a **hybrid model** to simulate realistic power consumption:
- **Base time drain** — power depletes over time regardless of activity (idle drain)
- **Distance drain** — additional cost per meter traveled
- **Activity costs** — scanning avatars, height adjustments, TP operations each have fixed costs

This was chosen over a simple timer-only drain because a flat rate would incorrectly charge the same power for a 10m hop and a 50m run.

---

### Capacity Design Goals

| Operating State | Target Battery Life |
|----------------|-------------------|
| Idle (on, stationary) | 24 hours |
| Patrol (normal roaming) | 12 hours |
| Active Scanning | 6 hours |
| Heavy Activity | 4 hours |
| Extreme Activity | 2 hours |

---

### Power Drain Constants

```lsl
float BASE_POWER_DRAIN = 0.14;          // % per minute at rest (patrol rate)
                                         // 0.07% per minute = idle (24hr rate)
float MOVEMENT_POWER_PER_METER = 0.05;  // additional % per meter traveled
float HEIGHT_ADJUSTMENT_COST = 8.0;     // % cost per height adjustment event
float EMERGENCY_THRESHOLD = 25.0;       // % — triggers coordinate pre-fetch
float CRITICAL_THRESHOLD = 5.0;         // % — triggers emergency return TP
```

**Rate breakdown:**
- Idle drain: 4.17% per hour = **0.07% per minute**
- Patrol drain: 8.33% per hour = **0.14% per minute**
- Distance: **0.05% per meter** traveled
- Height adjustment: **8 units** per event (testing rate)

---

### Avatar Scan Power Costs

| Avatar Type | Power Cost | Notes |
|-------------|-----------|-------|
| Owner | 1% | Identification only |
| Admin | 1% | Identification only |
| Group member | 1% | Identification only |
| Guest (permanent or temp) | 1% | Identification only |
| Banned user | 0.5% | Quick detection, security response triggered |
| Unknown avatar | 1% + (0.3% × script count) | Full scan; script count = number of scripts detected on avatar |
| TP operations | 15% | Per sequence (future feature) |

---

### Power Thresholds and Actions

| Power Level | Action |
|------------|--------|
| > 50% | Hover text green; normal patrol operation |
| 25–50% | Hover text yellow; normal patrol operation |
| ≤ 25% | **Emergency mode activates:** speed reduced to 50%, power consumption increased to 200%, coordinate request sent region-wide to dock, drone continues patrol. Hover text red. |
| 5% | **Emergency return triggered:** drone TPs to updated dock coordinates, lands, begins charging |
| 1% | Power forced to exactly 1% before emergency TP (intentional RP design: "just enough to return home") |

---

### Emergency Mode Visual (≤ 25%)

Hover text flashes alternating colors every 1 second:

| State | Color | Vector Value |
|-------|-------|-------------|
| Flash A | Red | `<1.0, 0.255, 0.212>` |
| Flash B | Orange | `<1.0, 0.522, 0.106>` |

---

### Hover Text Color Reference

| Power Level | Color | Vector Value |
|------------|-------|-------------|
| > 50% | Green | `<0.18, 0.8, 0.251>` |
| 25–50% | Yellow | `<1.0, 0.863, 0.0>` |
| < 25% | Red | `<1.0, 0.255, 0.212>` |
| Emergency | Flashing red/orange | Alternates every 1 second |

Hover text format: `[Object Name]\nPower: [bar] [X]%`
Uses `llGetObjectName()` (not hardcoded) so each named drone shows its own name.

---

### Power Bar Display Format

```lsl
// 20-character bar: █ = filled, ░ = empty
// bar_filled = (integer)(current_power / 5.0)
// Example at 60%: [████████████░░░░░░░░] 60%
```

---

### Charging System

#### Charging Trigger

Charging starts when:
1. Movement Script sends `LANDING_COMPLETE` link message after drone lands
2. Power Script detects `current_power <= 10.0` at landing → initiates charging

#### Charge Rate (Proportional)

```
charge_time_needed = (100 - current_power) / 100 * FULL_CHARGE_TIME
charge_rate = (100 - current_power) / charge_time_needed
            = 1 / FULL_CHARGE_TIME  (always the same rate per second)
```

| Mode | FULL_CHARGE_TIME | Rate | Seconds per 1% |
|------|-----------------|------|----------------|
| Production | 3600 s (1 hour) | 0.0278%/sec | ~36 sec |
| Test | 900 s (15 min) | 0.111%/sec × 2.5× multiplier | ~9 sec |

> **IMPORTANT:** The 2.5× multiplier in test mode **must be removed** before switching to production 1-hour mode. It compensates for LSL timer unreliability at 0.1-second intervals (which were replaced by 1.0-second intervals).

#### Time to Full Charge by Starting Level

| Starting Power | Time to Full (Production) |
|---------------|--------------------------|
| 0% → 100% | 60 minutes |
| 25% → 100% | 45 minutes |
| 50% → 100% | 30 minutes |
| 75% → 100% | 15 minutes |
| X% → 100% | `(100 - X) / 100 × 60 minutes` |

#### Visual During Charging

- Hover text shows "CHARGING"
- Eye and hover platform glow **blue**
- Battery visual prop shows filling segments (if deployed)

---

### Charging Cycle States (Full Sequence)

1. Drone lands → Movement Script sends `LANDING_COMPLETE`
2. Power Script detects low power at landing → calls `start_charging()`
3. Visual: blue glow, "CHARGING" text
4. Power increments by `charge_rate` each timer tick (1.0 second interval)
5. At 100%: Power Script sends `CHARGING_COMPLETE` link message
6. Config Script receives → sends `START_PATROL` link message
7. Movement Script rises, resumes from saved waypoint

---

### Charge Status Broadcasting

While charging, Power Script broadcasts status once per minute:
- Via link message to Config Script
- Config Script re-broadcasts on `comm_channel` to paired listening devices
- Format includes: current power% + estimated minutes remaining
- Estimated minutes formula: `minutes = (100 - current_power) / 100 * 60`

---

### State Persistence

Power level is stored between script resets:
```lsl
llLinksetDataWrite("current_power", (string)current_power);
// Restored in state_entry():
string saved = llLinksetDataRead("current_power");
if (saved != "") { current_power = (float)saved; }
```

---

### Battery Visual Prop (Standalone)

A separate "Drone Battery" object with 10 child prims (named `1` through `10`):
- Each segment = 10% of total battery
- Only **face 1** of each prim is modified; faces 0 and 2 are left to manual setup
- **Bidirectional wave effect:**
  - Charging: fills bottom-up (1→10) with cascade anticipation upward
  - Draining: fades top-down (10→1) with cascade anticipation downward
- `VISIBILITY_THRESHOLD = 0.05` (5%) — segments only show glow above this threshold

**Status indicator prims** (named `charge`):

| State | Color | Glow |
|-------|-------|------|
| Draining | Orange `<1.0, 0.5, 0.0>` | 0.25 |
| Drained (0%) | Red `<1.0, 0.0, 0.0>` | 0.15 |
| Charging | Blue `<0.0, 0.5, 1.0>` | 0.15 |
| Charged (100%) | Green `<0.0, 1.0, 0.0>` | 0.25 |

> **PBR Note:** PBR must be **disabled** on the glass container prim to allow discrete segment visibility. PBR light blending causes all segments to appear lit simultaneously.

---

*Section 4 of 10 complete*

---

## 5. Movement System

### Approach History — What Was Rejected

Before the final system, several movement approaches were tried and abandoned:

| Approach | Reason Rejected |
|----------|----------------|
| Physics-based movement | Object flopped, lost orientation, face dragged on ground, rotation spun to non-zero X/Y |
| `llMoveToTarget()` | Requires physics (rejected for same reasons) |
| Random roaming | Caused scope errors in LSL, unpredictable path, hard to control |
| `llSetRegionPos()` for patrol | Teleports instantly — no smooth movement |

---

### Adopted Approach: Pre-Planned Notecard Waypoint System

- Owner **walks a path** with the Pathway HUD worn, stopping at each desired point to record it
- Coordinates stored in a notecard named **"Tour"** inside the drone
- Drone follows waypoints in sequence: `0 → 1 → 2 → ... → N → 0` (loops forever)
- `llSetKeyframedMotion()` provides smooth horizontal movement between waypoints
- No collision detection needed — path is pre-walked and confirmed clear by the owner

---

### Movement Constants

```lsl
string NOTECARD_NAME = "Tour";
float MOVEMENT_SPEED = 2.0;          // meters per second (normal)
float HOVER_HEIGHT = 0.25;           // meters above ground during patrol
float height_change_threshold = 1.0; // minimum terrain delta to trigger rise/lower
```

---

### Z-Axis Rule (Critical)

The drone **never changes its Z coordinate** during normal patrol:
- Z is locked to the `ground_level` value recorded when the drone was rezzed
- During movement: `Z = ground_level.z + HOVER_HEIGHT` (0.25m above ground)
- During idle/docked: `Z = ground_level.z` (on the ground)
- `llSetRegionPos()` is used **only** for height adjustments (rise/lower), not horizontal patrol

---

### Timer Intervals

| Activity | Timer Interval |
|----------|---------------|
| Patrol (between waypoint checks) | 1.0 second |
| Rising sequence | 0.1 second |
| Lowering sequence | 0.1 second |

---

### Rise / Lower Sequences

The drone rises before patrolling and lowers when stopping. Each takes 3 seconds:

```
Rise: 0.0m → 0.25m over 3 seconds (0.1s timer, 0.01m increments)
Lower: 0.25m → 0.0m over 3 seconds (same rate in reverse)
```

- Timer fires at 0.1s intervals during rise/lower
- Height calculated proportionally: `current_height = (rise_timer / 3.0) * HOVER_HEIGHT`
- Rise complete → starts patrol timer at 1.0s
- Lower complete → stops timer entirely

---

### Waypoint Advancement

When the drone reaches the current waypoint (within 0.5m threshold):
```lsl
current_waypoint++;
if (current_waypoint >= total_waypoints)
{
    current_waypoint = 0;
}
```

---

### Rotation

Drone uses **Z-axis only** rotation to face the direction of travel:
```lsl
vector direction = llVecNorm(movement);
float z_angle = llAtan2(direction.y, direction.x);
rotation new_rot = llEuler2Rot(<0.0, 0.0, z_angle>);
llSetRot(new_rot);
```

Compass mapping (radians):
- `0.0` = facing East
- `1.57` = facing North (π/2)
- `3.14` = facing West (π)
- `4.71` = facing South (3π/2)
- `0.785` = NE, `2.36` = NW, `3.93` = SW, `5.49` = SE

---

### Emergency Mode Speed

When Power Script sends `EMERGENCY_MODE`:
- Speed reduced to **50% of normal** = 1.0 m/s
- Power consumption increased to 200%
- Drone continues patrol (does not stop)

When Power Script sends `NORMAL_MODE`:
- Speed restored to 2.0 m/s
- Power consumption restored to 100%

---

### Height Adjustment (Bridge Crossing)

Each waypoint in the Tour notecard stores a terrain Z value (the third column). The Movement Script uses this to handle elevation changes:

1. Movement Script tracks `working_terrain_level` (current terrain Z)
2. At each waypoint: compare `working_terrain_level` vs next waypoint's terrain Z
3. If difference > `height_change_threshold` (1.0m): trigger rise before crossing, lower after
4. **v4.0.4 fix — Delayed lowering:** The lower command is delayed by +1 waypoint to prevent premature descent mid-bridge (flag: `lowering_delayed`)

---

### Tour Notecard Format

Notecard named **"Tour"** inside the drone:

```
# Route: "Test Patrol"
# Generated: 2025-09-04 18:58:00
RP_MODE: YES
ROTATION_AXIS: Z
HOME: 112, 140
START: 107, 141, 3.14159
WAYPOINTS:
107, 157, 0.0
111, 166, 0.785
142, 166, 1.57
142, 124, 3.14
122, 108, 3.14
116, 108, 4.71
114, 126, 5.49
END: 107, 141
```

**Field notes:**
- `RP_MODE: YES/NO` — controls whether status messages are RP emotes or debug output
- `ROTATION_AXIS: Z` — which axis controls the drone's facing direction (snail uses Z)
- `HOME:` line — became **obsolete** with dock pairing system (dock provides coordinates dynamically)
- Waypoint columns: `X, Y, terrain_Z` (the terrain_Z is used for bridge/height detection)
- Angle column in START line is facing direction in radians

---

### Notecard Change Detection (v4.1.0+)

Movement Script monitors `changed()` event:
```lsl
changed(integer change)
{
    if (change & CHANGED_INVENTORY)
    {
        key current_key = llGetInventoryKey(NOTECARD_NAME);
        if (current_key != stored_notecard_key && current_key != NULL_KEY)
        {
            stored_notecard_key = current_key;
            reload_notecard();  // clears old waypoints, preserves position state
        }
    }
}
```

Manual force-reload: `/<channel> reload` command → Config sends `RELOAD_NOTECARD` link message.

---

### State Persistence (llLinksetData Keys)

| Key | Type | Purpose |
|-----|------|---------|
| `"patrol_active"` | "0"/"1"/"" | Patrol state: 0=paused, 1=patrolling, ""=fresh |
| `"current_waypoint"` | integer string | Current waypoint index |
| `"working_terrain"` | float string | Current terrain level |
| `"current_power"` | float string | Current power level |
| `"emergency_mode"` | integer string | Emergency mode flag |
| `"saved_position"` | vector string | Last known position |

---

### Resume Behavior on Script Reset

| Saved State | Behavior |
|-------------|----------|
| No saved state (fresh) | Waits for manual start command |
| `patrol_active = "0"` (paused) | Loads position, waits for start command |
| `patrol_active = "1"` (active) | Auto-resumes from saved waypoint on script reset |

Resume sequence output:
```
Resume enabled - will return to WP16
Rising...
Starting patrol
TP: Resume successful - starting movement in 3 seconds
```

---

### Resume TP Issue (Critical Design Note)

`llSetRegionPos()` behaves differently depending on context:
- In **link_message handler**: works reliably — drone TPs to target position
- In **timer handler**: returns TRUE but drone does NOT move

**Working solution:** Timer sends a link message to self, resume TP executes in the link_message handler:
```lsl
// Timer context (unreliable for TP):
llMessageLinked(LINK_THIS, saved_waypoint, "RESUME_TP", NULL_KEY);

// link_message handler (reliable for TP):
if (str == "RESUME_TP")
{
    current_waypoint = num;
    llSetRegionPos(saved_position);
}
```

---

### Reset Command

Chat commands on the paired channel:
- `/<channel> reset` — clears all llLinksetData, forces fresh start, resets script
- `/<channel> fresh` — same as reset (alias)

---

### Emergency Return Sequence (Phase 1 — Teleport Based)

Triggered when Power reaches 5% (`EMERGENCY_RETURN` link message):

1. Power forced to exactly 1%
2. Patrol state saved to linkset data
3. Motion stopped: `llSetKeyframedMotion([], [])`
4. Approach position calculated: 1 meter in front of Drone_Home prim using Z-axis trig:
   ```lsl
   float z_radians = euler.z;
   vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
   vector approach_pos = emergency_home_position + approach_offset;
   ```
5. Drone TPs to approach position, facing same direction as dock (matching Z rotation)
6. Dock notified: `DRONE_APPROACHING` (via `llRegionSayTo`)
7. Over **10 seconds**, drone backs into dock using keyframed motion
8. Emergency landing sequence timer fires → final position at dock + HOVER_HEIGHT
9. Drone lowers to ground → `"Landed"` message
10. `LANDING_COMPLETE` link message sent → Power Script initiates charging

---

### Departure Sequence (after charging)

1. Rise to hover height (standard 3-second rise)
2. Move 1m forward (away from dock, using dock's orientation)
3. Send `DRONE_DEPARTING` to dock (via channel)
4. TP to saved patrol position (emergency return) OR wait for touch (manual return)
5. Resume patrol from saved waypoint

---

### Copied Drone Fix

When a drone is paired for the first time:
- Config Script sends `CLEAR_SAVED_STATE` link message
- Movement Script clears stored waypoint index and terrain data
- Prevents copied drones from resuming at the original drone's last patrol position

---

*Section 5 of 10 complete*

---

## 6. Dock System

### Design Principles

- One dock pairs with one drone (no multi-drone docking on a single dock)
- Pairing is owner-validated (only pairs objects with matching owners)
- Dock provides its own coordinates dynamically — no hardcoded HOME in the drone's notecard
- All dock-drone communication uses a unique calculated `comm_channel` (not a hardcoded channel)
- Dock communicates region-wide (`llRegionSay`) so distance doesn't break the connection

---

### Dock Object Structure

The dock linkset contains one special child prim:

| Prim Name | Purpose |
|-----------|---------|
| `Drone_Home` | Defines the exact docking target position and orientation. Always set to `0, 0, Z` rotation (no X/Y tilt). The Drone_Home prim's position is the coordinate the drone aligns to. |

**Physical offset:** Drone center sits **0.10075m** away from Drone_Home prim center (Y-axis in local space).

---

### Dock Scripts

#### Dock Config Script (v1.5.0)

**Responsibilities:**
- Listens on calculated `comm_channel` for drone requests
- Responds to `REQUEST_HOME_COORDINATES` with current Drone_Home position and rotation
- Handles `PING`, `STATUS`, `COORDS`, `RP_MODE` commands
- Silently ignores unrecognized messages (`CHARGE_STATUS`, `BATTERY:`, etc.)
- On owner touch (when paired): privately reports the `comm_channel` via `llRegionSayTo(llGetOwner(), 0, ...)`
- Respects drone's `RP_MODE` setting for output suppression

**Version history:**

| Version | Key Change |
|---------|-----------|
| v1.2.0 | Initial: region-wide communication, basic command handling |
| v1.3.0 | Added `is_dock_message()` filtering — silently ignores unknown messages |
| v1.4.0 | Paired-touch shows channel privately to owner; post-pairing instructions |
| v1.5.0 | RP_MODE listening; `dock_output()` function respects RP_MODE; persistent storage of drone RP setting |

**Commands recognized (v1.3.0+):**

| Command | Response |
|---------|----------|
| `REQUEST_HOME_COORDINATES` | Broadcasts `HOME_COORDINATES:<pos>\|<rot>` region-wide |
| `STATUS` / `DOCK:STATUS` | Status report |
| `PING` | `PONG` |
| `DOCK:PING` | `DOCK:PONG` |
| `COORDS` / `DOCK:COORDS` | `DOCK:CURRENT_COORDS:<pos>` |
| `RP_MODE:YES` or `RP_MODE:NO` | Stored and applied to output filtering |

**Silent ignore list:** `CHARGE_STATUS`, `BATTERY:`, and any message not prefixed with `DOCK:` or in the recognized legacy list.

**RP_MODE output rules (v1.5.0):**
- `dock_output()` function — when `drone_rp_mode == "NO"`, suppresses operational messages
- Always shows: pairing events, setup messages, errors (regardless of RP_MODE)

---

#### Dock Pairing Script (v1.4.0)

**Responsibilities:**
- Manages the initial one-time dock-to-drone pairing process
- Calculates the unique `comm_channel` from a hash of the drone UUID
- Saves pairing data (drone UUID, channel, home coordinates) to linkset data
- Provides admin reset/status commands via the calculated channel

**Pairing process:**
1. Owner touches dock → dock enters pairing mode for **60 seconds**
2. Dock broadcasts ping to nearby drones on channel `0` (public)
3. Nearby drone's Pairing Script responds with its UUID
4. Unique `comm_channel` calculated from UUID hash
5. Drone_Home prim position and rotation sent to drone
6. Pairing data saved to linkset data on both sides

**Version history:**

| Version | Key Change |
|---------|-----------|
| v1.1.0 | Basic pairing, finds Drone_Home prim, sends coordinates |
| v1.2.0 | Added `pairing_completed` flag to prevent re-pairing; self-delete disabled for testing |
| v1.4.0 | Full rewrite: proper reset mechanism, admin listener on calculated channel, touch always re-enters pairing mode |

**Why v1.4.0 was required:**
- `llLinksetDataRead()` data survives script resets
- Earlier versions checked linkset data on startup and **blocked re-pairing permanently**
- Fix: Admin listener is always set up on the calculated channel regardless of pairing state
- `/<channel> reset` clears all linkset data and resets pairing state
- `/<channel> status` shows current pairing status

---

### Channel Architecture

#### Unique `comm_channel` Calculation

The communication channel is derived from the drone's UUID, making each drone-dock pair unique:
```lsl
// Pattern from bible_1 / bible_2:
integer pairing_channel = (integer)("0x" + llGetSubString((string)llGetOwner(), 0, 7));
// Example result: -381513550
```

This ensures:
- Multiple drones on the same parcel do not interfere with each other
- Admin commands on one drone's channel do not affect other drones
- Battery and console panel can listen on the same channel without ambiguity

#### Channel Usage

| Channel | Used For |
|---------|----------|
| `0` (public) | Initial pairing broadcast (temporary, during pairing only) |
| `comm_channel` (calculated) | All ongoing drone↔dock communication; admin commands |
| `0` (owner-private via `llRegionSayTo`) | Dock quietly tells owner the channel number on touch |

---

### Dock Communication Protocol

#### Coordinate Request (at 25% power)

1. Power Script sends `POWER_CRITICAL` link message to Config Script
2. Config Script sends on `comm_channel`: `REQUEST_HOME_COORDINATES:<drone_UUID>`
3. Dock Config Script receives → logs `*** COORDINATE REQUEST RECEIVED ***`
4. Dock broadcasts: `HOME_COORDINATES:<x,y,z>|<rotation_quaternion>` (region-wide on same channel)
5. Config Script receives coordinates → calls `update_home_position(pos, rot, TRUE)` (emergency = TRUE, no movement)
6. Config sends `UPDATE_HOME_COORDINATES` link message to Movement Script (coordinates only)
7. Movement Script updates `emergency_home_position` and `emergency_home_rotation`
8. Config logs: `"Emergency coordinates updated - continuing patrol until 5% power"`

**Example observed coordinate message:**
```
HOME_COORDINATES:<101.10080, 143.86980, 1500.57500>|<0.00000, 0.70711, 0.00000, 0.70711>
```

#### Emergency Return Communication

| Message | Direction | Trigger |
|---------|-----------|---------|
| `DRONE_APPROACHING` | Drone → Dock | After TP to approach position |
| `DRONE_DEPARTING` | Drone → Dock | After departure sequence begins |
| `CHARGING_COMPLETE` | (Power → Config internally) | After 100% charge reached |

---

### Docking Alignment (Rotation-Aware)

The dock can be placed at any compass orientation. The drone calculates the correct approach using the Drone_Home prim's Z-axis rotation:

```lsl
rotation convert_dock_rotation(rotation dock_rot)
{
    vector euler = llRot2Euler(dock_rot);
    rotation converted = llEuler2Rot(<0.0, 0.0, euler.z>);
    return converted;
}
```

**Approach vector calculation (handles all compass angles via trigonometry):**
```lsl
float z_radians = euler.z;
vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
vector approach_pos = emergency_home_position + approach_offset;
// approach_pos = 1 meter in front of dock's facing direction
```

Cardinal direction mapping:
| Dock Facing | euler.z | approach_offset |
|-------------|---------|----------------|
| East | 0° | `<1.0, 0.0, 0.0>` |
| North | 90° | `<0.0, 1.0, 0.0>` |
| West | 180° | `<-1.0, 0.0, 0.0>` |
| South | 270° | `<0.0, -1.0, 0.0>` |
| NE | 45° | `<0.707, 0.707, 0.0>` |

---

### Dock Pairing Data (llLinksetData Keys — Drone Side)

| Key | Content |
|-----|---------|
| `"paired_dock_uuid"` | UUID of paired dock |
| `"dock_channel"` | Calculated `comm_channel` |
| `"home_position"` | Vector string: dock Drone_Home position |
| `"home_rotation"` | Rotation string: dock Drone_Home orientation |

---

### Phase 2: Proximity Docking (Planned, Not Yet Implemented)

Trigger conditions:
- Power < 25% **AND** distance to dock < 20 meters

Behavior:
- If both conditions met: initiate **controlled docking** (no emergency TP, smooth approach)
- If power < 25% but distance ≥ 20m: continue normal patrol (rely on Phase 1 emergency TP at 5%)
- If distance < 20m but power ≥ 25%: continue normal patrol

This replaces the abrupt TP-home with a graceful approach when the drone happens to be near the dock as power gets low.

---

*Section 6 of 10 complete*

---

## 7. Security System

> **Status:** Designed and partially specified. Not yet fully implemented as a separate script. Power costs are wired in the Power Script. Access list structure is defined. Full Scan and Security scripts are planned for future development.

---

### Access Control Hierarchy

Avatars are evaluated in priority order. Higher priority = processed first.

| Priority | Class | Storage | Notes |
|----------|-------|---------|-------|
| 1 | **Owner** | `llGetOwner()` (live check) | Never stored; always checked at runtime |
| 2 | **Admin** | `llLinksetData("admin_list")` | Pipe-delimited UUIDs |
| 3 | **Group** | `llLinksetData("security_group")` | Single group UUID; matched via `llSameGroup()` |
| 4 | **Guest** | `llLinksetData("guest_list")` | Permanent approval |
| 5 | **Temp Guest** | `llLinksetData("temp_guest_list")` | Cleared when avatar leaves region |
| 6 | **Banned** | `llLinksetData("banned_list")` | Permanent ban; triggers security response |
| 7 | **Unknown** | (not stored until decision) | Full scan; triggers admin notification |

---

### Security Lists (llLinksetData Keys)

| List | Key | Format |
|------|-----|--------|
| Admin list | `"admin_list"` | Pipe-delimited UUIDs: `"uuid1|uuid2|uuid3"` |
| Guest list | `"guest_list"` | Pipe-delimited UUIDs |
| Temp Guest list | `"temp_guest_list"` | Pipe-delimited UUIDs; purged on region leave |
| Banned list | `"banned_list"` | Pipe-delimited UUIDs |
| Security group | `"security_group"` | Single group UUID |

---

### Scanning Logic (Priority Order)

```lsl
key avatar_id = llDetectedKey(i);

if (avatar_id == llGetOwner())
{
    consume_power(1.0, "owner identification");
}
else if (llListFindList(admin_list, [avatar_id]) != -1)
{
    consume_power(1.0, "admin identification");
}
else if (security_group != NULL_KEY && llSameGroup(avatar_id))
{
    consume_power(1.0, "group member identification");
}
else if (llListFindList(guest_list, [avatar_id]) != -1 ||
         llListFindList(temp_guest_list, [avatar_id]) != -1)
{
    consume_power(1.0, "guest identification");
}
else if (llListFindList(banned_list, [avatar_id]) != -1)
{
    consume_power(0.5, "banned user detection");
    // Trigger security response
}
else
{
    // Full scan for unknown avatar
    integer script_count = detected_scripts;
    float scan_cost = 1.0 + (script_count * 0.3);
    consume_power(scan_cost, "full scan (" + (string)script_count + " scripts)");
    // Trigger admin notification
}
```

---

### Power Costs for Security Actions

| Action | Power Cost |
|--------|-----------|
| Owner/Admin/Group/Guest identification | 1.0% |
| Banned user detection | 0.5% |
| Unknown avatar full scan | 1.0% + (0.3% × script count) |
| Avatar scan (flat rate, Power Script v2.3.0) | 0.5% |
| Security TP to avatar | 2.0% |
| Laser target (stage 1) | 1.0% |
| Laser blast (stage 2) | 5.0% |

---

### Admin Notification Format

When an unknown avatar is detected:
```
Security Alert: Unknown Avatar
Name: [DisplayName]
Profile: secondlife:///app/agent/[UUID]/about
Location: [Region] ([X], [Y], [Z])

[PERMANENT] [TEMPORARY] [BAN]
```

Admin response buttons:
- **PERMANENT** — adds avatar to permanent guest list
- **TEMPORARY** — adds to temp guest list (cleared when avatar leaves region)
- **BAN** — adds to banned list

---

### Warning / Ejection System

- Minimum warning time: **10 seconds**
- Available presets: 10s, 20s, 30s, or custom (user-specified, min 10s)
- After warning timer: ejection or ban action executes

---

### Detection Persistence Rules

| List | Persistence |
|------|------------|
| Admin / Guest / Banned | Permanent (survives resets; stored via llLinksetData) |
| Temp Guest | Cleared when avatar detected leaving the region |
| Scanned list (already-checked cache) | Purged hourly OR when avatar leaves region |

---

### Security System Power Communication (Power Script v2.3.0)

The Power Script listens for link messages from security scripts:

| Link Message | Power Cost Deducted |
|-------------|-------------------|
| `SECURITY_TELEPORT:[target_id]` | 2.0% (`TP_TELEPORT_COST`) |
| `SECURITY_SCAN:[target_id]` | 0.5% (`AVATAR_SCAN_COST`) |
| `SECURITY_TARGET:[target_id]` | 1.0% (`LASER_TARGET_COST`) |
| `SECURITY_BLAST:[target_id]` | 5.0% (`LASER_BLAST_COST`) |

---

### Future: ID Card System (Planned)

- An optional attachment for pre-authorized visitors
- Broadcasts on the security channel when entering the monitored zone: auto-approves without scan
- Reduces power cost to 1% base for pre-authorized avatars
- Eliminates need for admin notification for known trusted visitors

---

### RP Mode Behavior (Security Output)

When `RP_MODE: YES`:
- Security events output as `/me` emotes on local channel 0
- Examples: `/me detects unknown presence`, `/me initiating security scan`
- No raw debug output in local chat

When `RP_MODE: NO`:
- Debug-style `llOwnerSay()` output only (visible to owner only)
- No public chat

---

*Section 7 of 10 complete*

---

## 8. All Code Blocks

---

### CB-01: Notecard Reader / Waypoint Parser

*Source: bible_1 — v2.0.1 dataserver handler*

```lsl
dataserver(key query_id, string data)
{
    if (query_id == notecard_query)
    {
        if (data == EOF)
        {
            notecard_loaded = TRUE;
            llOwnerSay("Route loaded - Waypoints: " + (string)total_waypoints);
            return;
        }

        data = llStringTrim(data, STRING_TRIM);

        if (llGetSubString(data, 0, 0) == "#" || data == "")
        {
        }
        else if (llSubStringIndex(data, "RP_MODE:") == 0)
        {
            string mode = llStringTrim(llGetSubString(data, 8, -1), STRING_TRIM);
            rp_mode = (llToUpper(mode) == "YES");
        }
        else if (llSubStringIndex(data, "HOME:") == 0)
        {
            string home_data = llStringTrim(llGetSubString(data, 5, -1), STRING_TRIM);
            list parts = llParseString2List(home_data, [","], []);
            if (llGetListLength(parts) == 2)
            {
                float x = llList2Float(parts, 0);
                float y = llList2Float(parts, 1);
                home_position = <x, y, ground_level.z>;
            }
        }
        else if (llSubStringIndex(data, "START:") == 0 ||
                 llSubStringIndex(data, "WAYPOINTS:") == 0 ||
                 llSubStringIndex(data, "END:") == 0)
        {
        }
        else
        {
            list parts = llParseString2List(data, [","], []);
            if (llGetListLength(parts) >= 2)
            {
                float x_pos = llList2Float(parts, 0);
                float y_pos = llList2Float(parts, 1);
                vector wp = <x_pos, y_pos, 0.0>;
                waypoints += [wp];
                total_waypoints++;
            }
        }

        notecard_line++;
        notecard_query = llGetNotecardLine(NOTECARD_NAME, notecard_line);
    }
}
```

---

### CB-02: Timer — Rise / Lower / Patrol Logic

*Source: bible_1 — v2.0.1 / v4.0.4 working version*

```lsl
timer()
{
    if (is_rising)
    {
        rise_timer += 0.1;
        current_height = (rise_timer / 3.0) * HOVER_HEIGHT;

        if (current_height >= HOVER_HEIGHT)
        {
            current_height = HOVER_HEIGHT;
            is_rising = FALSE;
            llOwnerSay("Starting patrol");

            if (total_waypoints > 0)
            {
                current_waypoint = 0;
                is_moving_to_waypoint = TRUE;
                llSetTimerEvent(1.0);
            }
        }
        else
        {
            vector current_pos = llGetPos();
            vector new_pos = <current_pos.x, current_pos.y, ground_level.z + current_height>;
            llSetRegionPos(new_pos);
        }
    }
    else if (is_lowering)
    {
        rise_timer += 0.1;
        current_height = HOVER_HEIGHT - ((rise_timer / 3.0) * HOVER_HEIGHT);

        if (current_height <= 0.0)
        {
            current_height = 0.0;
            is_lowering = FALSE;
            llSetTimerEvent(0.0);
            llOwnerSay("Landed");
        }
        else
        {
            vector current_pos = llGetPos();
            vector new_pos = <current_pos.x, current_pos.y, ground_level.z + current_height>;
            llSetRegionPos(new_pos);
        }
    }
    else if (is_moving && is_moving_to_waypoint)
    {
        vector wp = llList2Vector(waypoints, current_waypoint);
        vector target = <wp.x, wp.y, ground_level.z + current_height>;
        vector current_pos = llGetPos();
        vector movement = target - current_pos;
        float distance = llVecMag(movement);

        if (distance > 0.5)
        {
            float time_to_move = distance / MOVEMENT_SPEED;

            vector direction = llVecNorm(movement);
            float z_angle = llAtan2(direction.y, direction.x);
            rotation new_rot = llEuler2Rot(<0.0, 0.0, z_angle>);
            llSetRot(new_rot);

            llOwnerSay("Moving to waypoint " + (string)(current_waypoint + 1));

            llSetKeyframedMotion([movement, time_to_move], []);

            llSetTimerEvent(time_to_move + 1.0);
        }
        else
        {
            current_waypoint++;
            if (current_waypoint >= total_waypoints)
            {
                current_waypoint = 0;
            }
            llSetTimerEvent(1.0);
        }
    }
}
```

---

### CB-03: Z-Only Rotation Calculation

*Source: bible_1*

```lsl
// Calculate rotation to face direction of travel
vector direction = llVecNorm(movement);
float z_angle = llAtan2(direction.y, direction.x);
rotation new_rot = llEuler2Rot(<0.0, 0.0, z_angle>);
llSetRot(new_rot);
```

---

### CB-04: Touch Start — Stop / Start with Rise / Lower

*Source: bible_1*

```lsl
touch_start(integer total_number)
{
    if (llDetectedKey(0) == llGetOwner())
    {
        if (is_moving)
        {
            is_moving = FALSE;
            is_moving_to_waypoint = FALSE;
            llSetKeyframedMotion([], []);

            is_rising = FALSE;
            is_lowering = TRUE;
            rise_timer = 0.0;
            current_height = HOVER_HEIGHT;

            llOwnerSay("Landing...");
            llSetTimerEvent(0.1);
        }
        else
        {
            if (total_waypoints > 0)
            {
                is_moving = TRUE;
                is_rising = TRUE;
                is_lowering = FALSE;
                rise_timer = 0.0;
                current_height = 0.0;

                llOwnerSay("Rising...");
                llSetTimerEvent(0.1);
            }
        }
    }
}
```

---

### CB-05: State Persistence — Save / Load / Clear

*Source: bible_1*

```lsl
save_movement_state()
{
    llLinksetDataWrite("patrol_active", (string)is_moving);
    llLinksetDataWrite("current_waypoint", (string)current_waypoint);
    llLinksetDataWrite("working_terrain", (string)working_terrain_level);
    llLinksetDataWrite("current_power", (string)current_power);
    llLinksetDataWrite("emergency_mode", (string)emergency_mode);
}

integer load_movement_state()
{
    string active = llLinksetDataRead("patrol_active");
    if (active == "")
    {
        return FALSE;  // Fresh deployment
    }

    string saved_wp = llLinksetDataRead("current_waypoint");
    if (saved_wp != "")
    {
        current_waypoint = (integer)saved_wp;
    }

    string saved_terrain = llLinksetDataRead("working_terrain");
    if (saved_terrain != "")
    {
        working_terrain_level = (float)saved_terrain;
    }

    string saved_power = llLinksetDataRead("current_power");
    if (saved_power != "")
    {
        current_power = (float)saved_power;
    }

    is_moving = (integer)active;
    return TRUE;
}

clear_saved_state()
{
    llLinksetDataDelete("patrol_active");
    llLinksetDataDelete("current_waypoint");
    llLinksetDataDelete("working_terrain");
    llLinksetDataDelete("current_power");
    llLinksetDataDelete("emergency_mode");
    llLinksetDataDelete("saved_position");
    llOwnerSay("STATE: All saved data cleared - next start will be fresh");
    llResetScript();
}
```

---

### CB-06: Power System — Hybrid Drain Pattern

*Source: bible_1 — v2.0.2*

```lsl
// Global power variables (Power script)
float current_power;
float base_power_drain;        // % per minute
float distance_power_rate;     // % per meter
float last_power_update;       // llGetTime() timestamp
integer emergency_mode;        // TRUE/FALSE
integer flash_state;           // for flashing hover text

// In state_entry():
current_power = 100.0;
base_power_drain = 0.14;       // 0.14% per minute = 12hr patrol capacity
distance_power_rate = 0.05;    // 0.05% per meter
last_power_update = llGetTime();
emergency_mode = FALSE;
flash_state = FALSE;

// Timer (runs continuously):
float current_time = llGetTime();
if ((current_time - last_power_update) >= 60.0)
{
    float drain_amount = base_power_drain;
    if (emergency_mode == TRUE)
    {
        drain_amount = base_power_drain * 2.0;
    }
    current_power -= drain_amount;
    last_power_update = current_time;
    if (current_power < 0.0)
    {
        current_power = 0.0;
    }
}
```

---

### CB-07: Power Display with Color and Unicode Bar

*Source: bible_1 — v2.0.1*

```lsl
// update_power_display() — must run inside an event handler
string power_bar;
integer bar_filled;
integer bar_empty;
integer b;
power_bar = "[";
bar_filled = (integer)(current_power / 5.0);
bar_empty = 20 - bar_filled;
for (b = 0; b < bar_filled; b++)
{
    power_bar += "█";
}
for (b = 0; b < bar_empty; b++)
{
    power_bar += "░";
}
power_bar += "] " + (string)((integer)current_power) + "%";

// Color selection
vector text_color;
if (emergency_mode == TRUE)
{
    if (flash_state == TRUE)
    {
        text_color = <1.0, 0.255, 0.212>;  // Red
        flash_state = FALSE;
    }
    else
    {
        text_color = <1.0, 0.522, 0.106>;  // Orange
        flash_state = TRUE;
    }
}
else if (current_power > 50.0)
{
    text_color = <0.18, 0.8, 0.251>;       // Green
}
else if (current_power > 25.0)
{
    text_color = <1.0, 0.863, 0.0>;        // Yellow
}
else
{
    text_color = <1.0, 0.255, 0.212>;      // Red
}

// Uses object name dynamically (not hardcoded):
string display_text = llGetObjectName() + "\nPower: " + power_bar;
llSetText(display_text, text_color, 1.0);
```

---

### CB-08: Security Scan Power Consumption

*Source: bible_1*

```lsl
key avatar_id = llDetectedKey(i);

if (avatar_id == llGetOwner())
{
    consume_power(1.0, "owner identification");
}
else if (llListFindList(admin_list, [avatar_id]) != -1)
{
    consume_power(1.0, "admin identification");
}
else if (security_group != NULL_KEY && llSameGroup(avatar_id))
{
    consume_power(1.0, "group member identification");
}
else if (llListFindList(guest_list, [avatar_id]) != -1 ||
         llListFindList(temp_guest_list, [avatar_id]) != -1)
{
    consume_power(1.0, "guest identification");
}
else if (llListFindList(banned_list, [avatar_id]) != -1)
{
    consume_power(0.5, "banned user detection");
}
else
{
    integer script_count = detected_scripts;
    float scan_cost = 1.0 + (script_count * 0.3);
    consume_power(scan_cost, "full scan (" + (string)script_count + " scripts)");
}
```

---

### CB-09: Docking Alignment — Rotation-Aware Position Calculation

*Source: bible_1*

```lsl
vector calculate_docking_position(vector dock_pos, rotation dock_rot)
{
    vector dock_euler = llRot2Euler(dock_rot) * RAD_TO_DEG;
    vector position_offset = ZERO_VECTOR;

    if (llFabs(dock_euler.x) > 45.0)  // X-axis rotation (270°)
    {
        position_offset.x = -0.10075;
    }
    else if (llFabs(dock_euler.y) > 45.0)  // Y-axis rotation
    {
        position_offset.y = 0.10075;
    }
    else if (llFabs(dock_euler.z) > 45.0)  // Z-axis rotation
    {
        position_offset.z = 0.10075;
    }

    vector rotated_offset = position_offset * dock_rot;
    return dock_pos + rotated_offset;
}

vector calculate_approach_position(vector dock_pos, rotation dock_rot)
{
    // Approach from 2m in front of dock's facing direction
    vector approach_vector = <2.0, 0.0, 0.0> * dock_rot;
    return dock_pos + approach_vector + <0.0, 0.0, HOVER_HEIGHT>;
}
```

---

### CB-10: llSetKeyframedMotion — Correct Usage Patterns

*Source: bible_1*

```lsl
// Start keyframed motion to a waypoint
vector movement = target - current_pos;
float time_to_move = distance / MOVEMENT_SPEED;
llSetKeyframedMotion([movement, time_to_move], []);

// Stop keyframed motion immediately
llSetKeyframedMotion([], []);

// Emergency backup into dock (10-second sequence)
vector backup_movement = home_pos - approach_pos;
float backup_time = 10.0;
list keyframes = [backup_movement, ZERO_ROTATION, backup_time];
llSetKeyframedMotion(keyframes, [KFM_MODE, KFM_FORWARD]);
```

---

### CB-11: Dock Pairing Communication Pattern

*Source: bible_1*

```lsl
// Pairing channel calculation (from owner UUID hash)
integer pairing_channel = (integer)("0x" + llGetSubString((string)llGetOwner(), 0, 7));

// Dock broadcasts pairing signal:
llRegionSay(pairing_channel, "DOCK_PAIRING:" + (string)llGetKey());

// Drone listens and responds:
listen(integer channel, string name, key id, string message)
{
    if (llSubStringIndex(message, "DOCK_PAIRING:") == 0)
    {
        string dock_key_str = llGetSubString(message, 13, -1);
        key dock_key = (key)dock_key_str;
        if (llGetOwnerKey(dock_key) == llGetOwner())
        {
            // Store dock reference, start glowing
        }
    }
}

// After successful pairing — store in linkset data:
llLinksetDataWrite("paired_dock_uuid", (string)dock_uuid);
llLinksetDataWrite("dock_channel", (string)dock_channel);

// Requesting dock position at runtime:
llRegionSayTo(paired_dock_uuid, dock_channel, "REQUEST_DOCK_POSITION");
// Dock responds:
llRegionSayTo(paired_drone_uuid, dock_channel, "DOCK_POSITION:" + (string)llGetPos() + ":" + (string)llGetRot());
```

---

### CB-12: Emergency Return Sequence (link_message Context)

*Source: bible_1*

```lsl
// In link_message handler (llSetRegionPos works reliably here):
else if (str == "EMERGENCY_RETURN")
{
    // 1. Save patrol state
    llLinksetDataWrite("patrol_active", "0");
    llLinksetDataWrite("current_waypoint", (string)current_waypoint);

    // 2. Stop current motion
    llSetKeyframedMotion([], []);

    // 3. Request dock position
    llRegionSayTo(paired_dock_uuid, dock_channel, "REQUEST_DOCK_POSITION");
}

// When DOCK_POSITION received:
listen(integer channel, string name, key id, string message)
{
    if (llSubStringIndex(message, "DOCK_POSITION:") == 0)
    {
        // Parse position and rotation, calculate approach position
        vector approach_pos = dock_pos + approach_offset;

        // TP to approach position
        llSetKeyframedMotion([], []);
        llSetRegionPos(approach_pos);
        llSetRot(dock_rot);

        // Notify dock
        llRegionSayTo(paired_dock_uuid, dock_channel, "DRONE_APPROACHING");

        // Begin 10-second backup sequence into dock
        vector backup_movement = final_dock_pos - approach_pos;
        float backup_time = 10.0;
        llSetKeyframedMotion([backup_movement, ZERO_ROTATION, backup_time], [KFM_MODE, KFM_FORWARD]);

        emergency_landing_sequence = TRUE;
        llSetTimerEvent(backup_time + 0.5);
    }
}
```

---

### CB-13: Global Variables Declaration Pattern (Movement Script)

*Source: bible_1 — v2.0.3+*

```lsl
// Movement Script - Global Variables
string NOTECARD_NAME = "Tour";
float MOVEMENT_SPEED = 2.0;
float HOVER_HEIGHT = 0.25;
float height_change_threshold = 1.0;

// Basic state
list waypoints;
integer current_waypoint;
integer total_waypoints;
integer is_moving;
integer is_charging;
integer notecard_loaded;
integer is_rising;
integer is_lowering;
vector ground_level;
float current_height;
float rise_timer;
integer is_moving_to_waypoint;

// Notecard
key notecard_query;
integer notecard_line;

// Communication
integer chat_channel;
integer listen_handle;

// Resume system
integer resume_last_waypoint;
integer saved_waypoint;
float saved_terrain;
integer post_tp_delay;

// Emergency home
vector emergency_home_position;
rotation emergency_home_rotation;
integer emergency_landing_sequence;

// Debug
integer DEBUG_MODE;

// Working terrain
float working_terrain_level;
integer lowering_delayed;
```

---

### CB-14: update_home_position() — Emergency vs Normal Update

*Source: bible_2 — Config Script*

```lsl
update_home_position(vector new_pos, rotation new_rot, integer emergency_update)
{
    home_position = new_pos;
    home_rotation = new_rot;
    save_pairing_data();

    if (emergency_update == FALSE)
    {
        // Normal pairing - enable movement system
        string coord_message = (string)home_position + "|" + (string)home_rotation;
        llMessageLinked(LINK_THIS, 0, "HOME_POSITION_DATA", coord_message);
    }
    else
    {
        // Emergency coordinate update - just update stored data, no movement trigger
        string coord_message = (string)home_position + "|" + (string)home_rotation;
        llMessageLinked(LINK_THIS, 0, "UPDATE_HOME_COORDINATES", coord_message);
    }

    if (debug_mode) llOwnerSay("CONFIG: Home position updated to " + (string)home_position);
}
```

---

### CB-15: UPDATE_HOME_COORDINATES Handler — Movement Script

*Source: bible_2*

```lsl
else if (str == "UPDATE_HOME_COORDINATES")
{
    list coord_data = llParseString2List((string)id, ["|"], []);
    if (llGetListLength(coord_data) >= 2)
    {
        emergency_home_position = (vector)llList2String(coord_data, 0);
        rotation dock_rotation = (rotation)llList2String(coord_data, 1);
        emergency_home_rotation = convert_dock_rotation(dock_rotation);
        home_coordinates_received = TRUE;
        // Just update coordinates - don't execute emergency return
    }
}
```

---

### CB-16: LANDING_COMPLETE → CHARGING_COMPLETE → AUTO-RESUME

*Source: bible_2*

```lsl
// Movement Script — after drone lands:
llOwnerSay("Landed");
llMessageLinked(LINK_THIS, 0, "LANDING_COMPLETE", NULL_KEY);

// Power Script — on LANDING_COMPLETE:
else if (str == "LANDING_COMPLETE")
{
    if (current_power <= 10.0)
    {
        llOwnerSay("POWER: Landing detected at " + (string)((integer)current_power) + "% power - initiating charging sequence");
        start_charging();
    }
}

// Config Script — on CHARGING_COMPLETE:
else if (str == "CHARGING_COMPLETE")
{
    llOwnerSay("CONFIG: Charging complete - automatically resuming patrol");
    patrol_active = TRUE;
    llMessageLinked(LINK_THIS, 0, "START_PATROL", NULL_KEY);
}
```

---

### CB-17: Rotation Conversion — Z-Axis Only

*Source: bible_2 — Movement Script*

```lsl
rotation convert_dock_rotation(rotation dock_rot)
{
    vector euler = llRot2Euler(dock_rot);
    // Use Z-axis rotation only (dock always uses 0,0,Z format)
    rotation converted = llEuler2Rot(<0.0, 0.0, euler.z>);
    if (debug_mode) llOwnerSay("DEBUG: Using dock Z-axis rotation: " + (string)converted);
    return converted;
}
```

---

### CB-18: Approach Vector — Trig-Based for Any Compass Angle

*Source: bible_2 — Movement Script (execute_emergency_return)*

```lsl
vector euler = llRot2Euler(emergency_home_rotation);
float z_radians = euler.z;
// Calculate approach direction (handles all compass angles)
vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
vector approach_pos = emergency_home_position + approach_offset;
// approach_pos = 1 meter in front of dock's facing direction
```

---

### CB-19: Battery Charging Rate Constants (Production Mode)

*Source: bible_2*

```lsl
float FULL_CHARGE_TIME = 3600.0;   // 1 hour = production mode
// float FULL_CHARGE_TIME = 900.0; // 15 minutes = test mode

float VISIBILITY_THRESHOLD = 0.05; // 5% — minimum to show segment glow

// Charge rate calculation:
float power_needed = 100.0 - current_power;
float charge_time = (power_needed / 100.0) * FULL_CHARGE_TIME;
charge_rate = power_needed / charge_time;
// = 0.0277778 per second for 1-hour full charge

// Per timer tick (every 1.0 second):
current_power = current_power + charge_rate;
if (current_power >= 100.0)
{
    current_power = 100.0;
    // trigger CHARGING_COMPLETE
}
```

> **Test mode only:** `charge_rate = (power_needed / charge_time) * 2.5;` — the 2.5× multiplier compensates for LSL timer unreliability at 0.1-second intervals. **Must be removed for production.**

---

### CB-20: Battery Status Indicator Colors

*Source: bible_2*

```lsl
update_status_indicators()
{
    integer total_links;
    integer i;
    vector indicator_color;
    float indicator_glow;
    string link_name;
    integer prim_num;

    total_links = llGetNumberOfPrims();

    if (is_charging)
    {
        indicator_color = <0.0, 0.5, 1.0>;  // Blue
        indicator_glow = 0.15;
    }
    else if (is_draining)
    {
        indicator_color = <1.0, 0.5, 0.0>;  // Orange
        indicator_glow = 0.25;
    }
    else if (current_power >= 100.0)
    {
        indicator_color = <0.0, 1.0, 0.0>;  // Green
        indicator_glow = 0.25;
    }
    else
    {
        indicator_color = <1.0, 0.0, 0.0>;  // Red
        indicator_glow = 0.15;
    }

    for (i = 1; i <= total_links; i++)
    {
        link_name = llGetLinkName(i);
        if (link_name == "charge")
        {
            llSetLinkColor(i, indicator_color, ALL_SIDES);
            llSetLinkPrimitiveParamsFast(i, [PRIM_GLOW, ALL_SIDES, indicator_glow]);
        }
    }
}
```

---

### CB-21: HUD Glow-Before-Alpha Fix (Two-Pass)

*Source: bible_2 — Waypoint HUD*

```lsl
// When minimizing — MUST turn off glow before setting alpha to 0
// Pass 1: Turn off all glows
integer j;
for (j = 0; j < llGetListLength(light_links); j++)
{
    integer link_num = llList2Integer(light_links, j);
    if (link_num > 0)
    {
        llSetLinkPrimitiveParams(link_num, [PRIM_GLOW, ALL_SIDES, 0.0]);
    }
}
llSleep(0.01);  // Brief delay to let glow changes render
// Pass 2: Set all alphas to 0
for (j = 0; j < llGetListLength(light_links); j++)
{
    integer link_num = llList2Integer(light_links, j);
    if (link_num > 0)
    {
        llSetLinkAlpha(link_num, 0.0, ALL_SIDES);
    }
}
```

---

### CB-22: HUD loadData() — Correct Default Logic

*Source: bible_2*

```lsl
// CORRECT — only applies defaults if truly no data found:
string stored_rp = llLinksetDataRead("rp_mode");
if (stored_rp != "")
{
    rpMode = stored_rp;
}
else
{
    rpMode = "YES";  // Default only when no stored value exists
}

string stored_axis = llLinksetDataRead("rotation_axis");
if (stored_axis != "")
{
    rotationAxis = stored_axis;
}
else
{
    rotationAxis = "Z";  // Default only when no stored value exists
}
```

---

### CB-23: Notecard Change Detection — Movement Script (v4.1.0)

*Source: bible_2*

```lsl
// Global variable to track notecard key
key stored_notecard_key;

changed(integer change)
{
    if (change & CHANGED_INVENTORY)
    {
        key current_key = llGetInventoryKey(NOTECARD_NAME);
        if (current_key != stored_notecard_key && current_key != NULL_KEY)
        {
            stored_notecard_key = current_key;
            reload_notecard();  // clears old waypoints, preserves position state
        }
    }
}
```

---

### CB-24: CLEAR_SAVED_STATE Handler — Movement Script

*Source: bible_2*

```lsl
else if (str == "CLEAR_SAVED_STATE")
{
    // Prevents copied drones from resuming at the original drone's last position
    llLinksetDataDelete("saved_waypoint");
    llLinksetDataDelete("saved_terrain");
    current_waypoint = 0;
    if (debug_mode) llOwnerSay("MOVEMENT: Saved state cleared for fresh start");
}
```

---

### CB-25: Type Conversion Fix — key → integer

*Source: bible_2*

```lsl
// WRONG (type mismatch — cannot cast key directly to integer):
integer minutes_remaining = (integer)id;

// CORRECT (key → string → integer):
string minutes_str = (string)id;
integer minutes_remaining = (integer)minutes_str;
```

---

### CB-26: Dock Config — is_dock_message() Filtering (v1.3.0)

*Source: bible_3*

```lsl
else if (llSubStringIndex(processed_message, "CHARGE_STATUS:") >= 0)
{
    // Extract charging data if needed — but do NOT log "unknown message"
    list charge_parts = llParseString2List(processed_message, [":"], []);
    if (llGetListLength(charge_parts) >= 4)
    {
        string drone_uuid = llList2String(charge_parts, 1);
        integer current_power = (integer)llList2String(charge_parts, 2);
        integer max_power = (integer)llList2String(charge_parts, 3);
    }
    // Silently ignore — do not respond
}
```

---

### CB-27: Patrol Active Fix After Pairing (Config Script v3.0.1)

*Source: bible_3*

```lsl
// After pairing completion — explicitly set patrol_active to FALSE
// so that first touch after pairing STARTS patrol (not stops it)
patrol_active = FALSE;
```

---

### CB-28: Waypoint HUD — Rotation Angle Calculation

*Source: bible_3 — Pathway HUD*

```lsl
// "rotation" is a reserved LSL type — use "rotAngle" or "angleToNext"
float calculateAngle(vector fromPos, vector toPos)
{
    float angleToNext = llAtan2(toPos.y - fromPos.y, toPos.x - fromPos.x);
    return angleToNext;
}
```

---

### CB-29: Power Script — Final Constants (v2.3.0 Target)

*Source: bible_3*

```lsl
// Base runtime: 24 hours
float BASE_POWER_DRAIN = 0.069;            // % per minute

// Movement: target 22hr runtime during patrol
float MOVEMENT_POWER_PER_METER = 0.00006; // % per meter

// Height changes
float HEIGHT_ADJUSTMENT_COST = 1.5;       // % per terrain change

// Charging
float CHARGING_INTERVAL = 36.0;           // seconds per 1% — 1 hour total
float SOLAR_CHARGING_INTERVAL = 18.0;     // seconds per 1% — 30 minutes total

// Power thresholds
integer EMERGENCY_THRESHOLD = 25;         // % — ~6 hours remaining
integer CRITICAL_THRESHOLD = 5;           // % — ~1.2 hours remaining

// Security costs
float TP_TELEPORT_COST = 2.0;            // % per security TP
float AVATAR_SCAN_COST = 0.5;            // % per avatar scan
float LASER_TARGET_COST = 1.0;           // % initial laser targeting
float LASER_BLAST_COST = 5.0;            // % full security blast
```

---

### CB-30: Waypoint Pause Timing — Production vs Test

*Source: bible_3*

```lsl
// TEST (current) — 1 second pause between waypoints:
llSetTimerEvent(1.0);

// PRODUCTION — 0.25 second pause (much smoother patrol):
llSetTimerEvent(0.25);
// Location: Movement Script, at the end of waypoint-advance block
// Effect: ~95% moving time instead of ~50%, far more fluid appearance
```

---

*Section 8 of 10 complete*
