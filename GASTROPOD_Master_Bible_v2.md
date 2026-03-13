# GASTROPOD Master Bible v2.0

> **Full system reference for G.A.S.T.R.O.P.O.D.**
> Produced from complete re-read of all three source chat files.
> Cross-referenced against bible_1.md, bible_2.md, bible_3.md, and GASTROPOD_Master_Bible.md.
> Date produced: 2026-03-13

---

## Table of Contents

1. [Project Overview & Purpose](#1-project-overview--purpose)
2. [LSL Forbidden Rules & Coding Constraints](#2-lsl-forbidden-rules--coding-constraints)
3. [Script Architecture](#3-script-architecture)
4. [Power System](#4-power-system)
5. [Movement System](#5-movement-system)
6. [Dock System](#6-dock-system)
7. [Security System](#7-security-system)
8. [All Code Blocks](#8-all-code-blocks)
9. [Bugs & Current Status](#9-bugs--current-status)
10. [Changelog](#10-changelog)
11. [Conclusion & Roadmap](#11-conclusion--roadmap)

---

## 1. Project Overview & Purpose

### 1.1 Name & Acronym

**G.A.S.T.R.O.P.O.D.** — Guided Autonomous Security Tactical Response Operations Drone

### 1.2 Platform

Second Life (Linden Lab virtual world), scripted in LSL (Linden Scripting Language).

### 1.3 Physical Object

A snail mesh object operating in a skybox at altitude ~1500m. The drone is non-physical and moves via `llSetKeyframedMotion()` (KFM). The drone nameplate format is `G.A.S.T.R.O.P.O.D. mrk [I/II/III/IV]`. Confirmed deployed units: mrk I, mrk II, mrk III, mrk IV.

### 1.4 Purpose

An autonomous security drone that:

- Patrols a user-defined waypoint route 24/7
- Scans avatars entering its range
- Engages unauthorized/banned avatars via an integrated Eye Security system (laser target + blast)
- Returns to dock autonomously when power is low, charges, then resumes patrol
- Reports status to a HUD and Control Panel
- Operates in RP Mode (immersive emotes) or silent mode depending on notecard config

### 1.5 Architecture Philosophy

The system was originally designed as a single monolithic LSL script. A stack-heap collision error (script exceeded the 64KB LSL memory limit) forced a split into **nine modular scripts**. Each script handles one domain and communicates with others via `llMessageLinked()` (internal) or `llRegionSay()` / `llRegionSayTo()` (external).

Cross-script persistent state is stored in `llLinksetData*` (64KB per linkset).

### 1.6 Development Phases

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1 | Complete | Autonomous charging cycle: patrol → emergency return → charge → resume |
| Phase 1.5 | Complete | Multiple drones, RP Mode, HUD output chunking, dock filtering |
| Phase 2 | In Progress | Battery system, solar panel integration, Control Panel, hot-swap charging |
| Phase 3 | Planned | Eye Security integration, laser targeting, blast power costs |

**Phase 1 confirmed working:** September 9, 2025. Drone lands at dock at 1% power, charges 1%→100%, then auto-resumes at last saved waypoint.

**Multiple drones confirmed simultaneous operation:** mrk I (at WP5) and mrk III (at WP33) both charging in the same session.

### 1.7 System Components

| Component | Object Name | Purpose |
|-----------|-------------|---------|
| Drone | `G.A.S.T.R.O.P.O.D. mrk [N]` | The main patrol unit |
| Dock | `G.A.S.T.R.O.P.O.D. Dock` | Charging station, coordinate provider |
| HUD | `Pathway` (worn) | Route creation, drone control |
| Control Panel | TBD | Real-time status display, countdown timers |
| Battery | TBD | Optional fast-charge / hot-swap unit |
| Eye Security | TBD | Laser targeting system (Phase 3) |

### 1.8 Communication Overview

| Channel | Type | Purpose |
|---------|------|---------|
| Unique `comm_channel` | `llListen()` | Per-drone channel from UUID hash — all drone/dock messages |
| `0` | `llSay()` / `llRegionSay()` | Pairing broadcast, RP emotes |
| `-8890`, `-8889` | `llListen()` | HUD dialog channels |

Comm channel calculation (from UUID):
```lsl
integer comm_channel = (integer)("0x" + llGetSubString((string)llGetKey(), 0, 6)) * -1;
```
Example: drone UUID `d4f00f32-30f9-7c82-1cc1-298cc7662b0c` → channel `-233798746`
Another example: channel `-381513550`

### 1.9 Autonomous Operating Cycle

The full intended loop (Phase 1, confirmed working):

```
Patrol → 25% power (coordinate update, continue patrol) → 5% power
→ Emergency TP home → Back into dock → Land
→ LANDING_COMPLETE → Begin charging (1 hour production / 15 min test)
→ 100% power → CHARGING_COMPLETE → Auto-resume patrol at saved waypoint
→ [Repeat indefinitely]
```

Key design notes:
- Power is always forced to exactly **1%** before emergency TP (RP theme: "just enough to return home")
- At 25% power, dock coordinates are silently updated in the background — drone continues patrol
- At 5% power, emergency TP is triggered using the most recently updated dock coordinates
- After charging, drone automatically resumes from the **saved waypoint index** (not from WP0)

### 1.10 Development History

| Chat Batch | Dates | Key Milestone |
|------------|-------|---------------|
| First 3 chats | Sept 4–8, 2025 | Core architecture: movement, power, dock design; movement script working |
| Second 3 chats | Sept 8–13, 2025 | Phase 1 complete: full autonomous cycle confirmed working |
| Last 3 chats | Sept 13–26, 2025 | Dock script fixes, Pathway HUD, RP Mode system, Power Script v2.3.0 overhaul |

---

## 2. LSL Forbidden Rules & Coding Constraints

These rules were established during the project to prevent LSL compilation errors, runtime errors, and scope bugs. All scripts in the GASTROPOD system must comply.

### Rule 1 — No Ternary Operators

LSL does not support the `?:` ternary operator.

**Wrong:**
```lsl
integer x = (condition) ? 1 : 0;
```
**Correct:**
```lsl
integer x;
if (condition) { x = 1; } else { x = 0; }
```

### Rule 2 — No Foreach Loops

LSL has no `foreach` construct. Use indexed `for` or `while` loops with integer counters.

**Wrong:**
```lsl
foreach (item in list) { ... }
```
**Correct:**
```lsl
integer i = 0;
while (i < llGetListLength(myList)) {
    string item = llList2String(myList, i);
    i++;
}
```

### Rule 3 — No Global Assignments from Function Calls

Global variable declarations may not be initialized by function calls. Functions may only be called inside event handlers or user-defined functions.

**Wrong (top of script):**
```lsl
string my_key = (string)llGetKey();  // runtime call at global scope
```
**Correct:**
```lsl
string my_key;  // declare empty globally

default {
    state_entry() {
        my_key = (string)llGetKey();  // assign inside event handler
    }
}
```

### Rule 4 — No `void` Keyword

LSL has no `void` return type. Functions that return nothing simply have no return type specifier.

**Wrong:**
```lsl
void my_function() { ... }
```
**Correct:**
```lsl
my_function() { ... }
```

### Rule 5 — No `PRIM_HOVER_HEIGHT` in `llSetPrimitiveParams`

`PRIM_HOVER_HEIGHT` is not a valid parameter for `llSetPrimitiveParams` in LSL. Do not use it.

### Rule 6 — No Runtime Values in Global Scope

No expression that requires runtime evaluation may be placed in the global scope. Global variables must have compile-time constant values or be declared empty.

**Wrong:**
```lsl
float my_val = llFrand(1.0);  // runtime value
```
**Correct:**
```lsl
float my_val;  // assigned in state_entry()
```

### Rule 7 — No Invisible Storage via Object Name/Description/Notecards

Do not use `llSetObjectName()`, `llSetObjectDesc()`, or hidden notecards to pass state between scripts. Use `llLinksetDataWrite()` / `llLinksetDataRead()` for all persistent cross-script storage.

### Rule 8 — No Sneaky Shorthand

Write explicit, readable code. No bitwise tricks, no collapsed conditionals, no undocumented magic numbers. Every constant must be named.

### Rule 9 — No Reserved Keywords as Variable Names

The following LSL type names and keywords are reserved and must NOT be used as variable names:

| Reserved Word | Bad Variable Name | Correct Alternative |
|---------------|-------------------|---------------------|
| `state` | `string state` | `string stateValue` |
| `rotation` | `float rotation` | `float rotAngle` / `float angleValue` |
| `vector` | `vector vector` | `vector pos` |
| `integer` | `integer integer` | `integer count` |
| `float` | `float float` | `float val` |
| `string` | `string string` | `string text` |
| `key` | `key key` | `key target_key` |
| `list` | `list list` | `list items` |

### Additional LSL Constraints Observed in This Project

| Issue | Cause | Correct Approach |
|-------|-------|-----------------|
| Type mismatch concatenating float to string | `llGetTimeOfDay()` returns `float`, not `string` | `(string)llGetTimeOfDay()` |
| "Name previously declared within scope" | Variable declared twice in same scope | Rename one or move to outer scope |
| Stack-heap collision at 64KB | Monolithic script exceeded LSL memory limit | Split into modular scripts |
| Return type mismatch | Function declared without return type but uses `return value;` | Declare return type explicitly: `float my_function()` |
| LSL timer fires slower than expected at 0.1s intervals | LSL timer unreliable at very short intervals | Use 1-second intervals; adjust increment multipliers |
| `llSetRegionPos()` returns TRUE but doesn't move object | Called in conflict with active KFM timer | Stop timer first; wait 3 seconds before next movement message |
| Global variable inaccessible from user-defined function | LSL scope rules: globals only visible in event handlers | Pass as parameter to function or use linkset data |

### Script Header Format (Standard)

All scripts use this versioned header:
```lsl
// Script Name vX.Y.Z
// [One-line purpose description]
// Enhancements: [list new features added]
// Modifications: [list code changed/moved/adjusted]
// Cleaned: [list features removed]
// Fixed: [list broken features corrected]
// PHASE X: [current development phase]
```

### LSL Functions That Do NOT Exist (Confirmed Absent)

| Does Not Exist | Notes |
|----------------|-------|
| `llSlerp()` | No such function; use `llEuler2Rot()` for rotation |
| `at_target()` event | No such event; use timer-based distance check |
| `not_at_target()` event | No such event |
| `llGetTimerEvent()` | No such function |
| `llKeyframedMotion()` | Wrong name; correct is `llSetKeyframedMotion()` |

---

## 3. Script Architecture

### 3.1 Drone Root Prim Scripts

All scripts below reside in the **root prim** of the drone linkset:

| Script | Latest Version | Responsibility |
|--------|---------------|----------------|
| **Configuration Script** | v3.1.0 | Central coordination hub: dock pairing data management, notecard loading coordination, unique channel management, admin command dispatch, RP mode control, patrol start/stop, charge broadcasting, config_output() for consistent RP-aware messaging |
| **Movement Script** | v4.2.0 | Notecard reading, waypoint patrol, smooth KFM movement, height adjustment, rise/lower sequences, state persistence and resume, emergency return execution, notecard change detection, broadcast_rp_mode() to dock on notecard load |
| **Power Script** | v2.3.0 | Hybrid power drain tracking (time + distance + activity), charging cycle management, hover text display with unicode power bar, emergency threshold detection, charge broadcasting, battery swap communication, control panel broadcasting, security power costs |
| **Pairing Script** | v1.2.0 (temporary) | Initial dock-to-drone pairing only; calculates unique `comm_channel`; stores pairing data; designed to delete itself after pairing |

> **Note:** The original architecture called for 9 scripts. The current working system uses 4 drone scripts. Security, Scan, Diagnostics, and Communications scripts are planned but not yet built.

### 3.2 Drone Child Prim Scripts

These scripts reside in named **child prims** of the drone linkset:

| Child Prim Name | Script | Responsibility |
|----------------|--------|----------------|
| `Eye` | Eye script | Glow effects and particle beam targeting |
| `Hover` | Hover script | Underglow and teleport burst effects |

### 3.3 Dock Object Scripts

The dock is a **separate object** from the drone. It communicates over the calculated `comm_channel`.

| Script | Latest Version | Responsibility |
|--------|---------------|----------------|
| **Dock Config Script** | v1.5.0 | Receives coordinate requests from drone; broadcasts current Drone_Home prim position and rotation; message filtering (ignores non-dock messages); RP_MODE awareness via `drone_rp_mode` variable and `dock_output()` function; owner-touch displays paired channel via `llRegionSayTo()` |
| **Dock Pairing Script** | v1.4.0 | Owner initiates pairing via touch; discovers nearby drone; calculates unique `comm_channel`; saves pairing data to linkset; admin reset/status commands via calculated channel; `pairing_completed` flag prevents re-pairing |

### 3.4 HUD Object Scripts

The HUD is a **worn attachment** for the drone owner/builder. Object name: `Pathway`.

| Script | Location | Responsibility |
|--------|----------|----------------|
| **Pathway (root)** | Root prim `Pathway` | Full waypoint recording: route name, RP mode, rotation axis, start/waypoint/end recording, output generation (chunked 30 waypoints per message), resume beacon, minimize/maximize toggle, light system |

**HUD Prim Structure:**

| Prim Name | Type | Purpose |
|-----------|------|---------|
| `Pathway` | Root prim | Contains all HUD logic |
| `btn_route` | Button | Route name entry |
| `btn_rp` | Button | Toggle RP_MODE YES/NO |
| `btn_rotation` | Button | Rotation axis selection |
| `btn_start` | Button | Record START position |
| `btn_waypoint` | Button | Record WAYPOINT |
| `btn_end` | Button | Record END position |
| `btn_resume` | Button | Drop resume beacon |
| `btn_output` | Button | Generate notecard output |
| `btn_new` | Button | New route |
| `btn_waypnt` | Button | (alternate waypoint button) |
| `btn_toggle` | Button | Minimize/Maximize HUD |
| `light_route` | Light indicator | Orange glow 0.25 when btn_route active |
| `light_rp` | Light indicator | Orange glow 0.25 when RP active |
| `light_rotation` | Light indicator | Orange glow 0.25 when rotation active |
| `light_start` | Light indicator | Orange glow 0.25 when start recorded |
| `light_waypoint` | Light indicator | Orange glow 0.25 when waypoint recorded |
| `light_end` | Light indicator | Orange glow 0.25 when end recorded |
| `light_resume` | Light indicator | Orange glow 0.25 when resume set |
| `light_output` | Light indicator | Orange glow 0.25 when output active |
| `light_new` | Light indicator | Orange glow 0.25 when new route |

**HUD Notes:**
- No scripts in child button prims — all logic in root `Pathway` prim
- Button light prims are found by name using `getLinkByName()` function
- Dialog channels: `-8890` and `-8889`
- Minimize: root prim invisible, all lights off
- Maximize: root prim visible, lights on per state
- Output uses `llOwnerSay()` (not `llRegionSayTo()`) for higher character limit
- Large routes are chunked: header message + chunks of 30 waypoints + "=====Continued=====" prefix on each chunk after the first + END message

### 3.5 Peripheral / Proof-of-Concept Scripts

| Script | Version | Object | Responsibility |
|--------|---------|--------|----------------|
| **Battery Visual Charging** | v2.2 | "Drone Battery" | 10-segment visual battery prop: bidirectional charging wave effect (bottom-up fill / top-down drain), status indicator prims named "charge" |

**Battery Visual Notes:**
- 10 child prims named `1` through `10` (numbers as names)
- Only face 1 of each prim is modified (not face 0 or 2)
- `VISIBILITY_THRESHOLD = 0.05` — minimum alpha before segment becomes visible
- Status prim named `charge` (2 such prims): orange=draining 0.25 glow, red=drained 0%, blue=charging 0.15 glow, green=charged 0.25 glow
- PBR (Physically Based Rendering) must be **disabled** on the glass container prim for correct segment visibility
- 2.5x timing multiplier needed in test mode (timer fires slower than expected at short intervals)
- Production mode: no timing multiplier needed (1-hour charging)

### 3.6 Script Communication Architecture

**Internal (drone scripts to each other):**
- `llMessageLinked(LINK_THIS, num, str, id)` — all inter-script communication within drone linkset
- No script directly calls another; all coordination is event-driven via link messages

**External (drone/dock):**
- `llRegionSay(comm_channel, message)` — both directions (region-wide, not distance-limited)
- `comm_channel` is unique per drone-dock pair, calculated during pairing

**RP emotes:**
- `llSay(0, "/me ...")` — used for RP_MODE output to local public chat

**HUD output:**
- `llOwnerSay()` — used for route data output (higher character limit than `llRegionSayTo()`)

### 3.7 Prim Structure — Dock

| Prim Name | Purpose |
|-----------|---------|
| `Drone_Home` | Defines the exact docking target position and orientation. Always uses `0, 0, Z` rotation (Z-axis only). The drone aligns to this prim's position + 0.10075m offset. |

### 3.8 Memory Management

| Item | Limit | Notes |
|------|-------|-------|
| LSL script memory | 64KB per script | Exceeding causes Stack-Heap Collision |
| `llLinksetData` storage | 64KB per linkset (shared) | Adding prims does NOT increase storage |
| `llLinksetData` capacity | ~1000+ UUIDs | Estimated within 64KB limit |

### 3.9 LLD (LinksetData) Key Registry

| Key | Set By | Purpose |
|-----|--------|---------|
| `paired_drone_uuid` | Pairing Script | UUID of paired drone |
| `comm_channel` | Pairing Script | Calculated unique channel |
| `drone_home_coords` | Dock Config | Last known dock coordinates |
| `debug_mode` | Power / Movement / Config | Persistent debug toggle |
| `current_waypoint` | Movement Script | Saved waypoint index for resume |
| `patrol_active` | Config / Movement | Whether patrol is in progress |
| `emergency_return` | Movement Script | Whether emergency return was active |
| `power_level` | Power Script | Current power percentage |
| `rp_mode` | Config / Dock Config | RP_MODE setting from notecard |

### 3.10 Version History

| Script | After First Chats | After Second Chats | After Last Chats |
|--------|------------------|-------------------|-----------------|
| Configuration | Modular design, no version | v2.1.0 → v3.0.0 | v3.0.0 → v3.1.0 |
| Movement | v2.0.1 (working) | v3.1.0 → v4.1.0 | v4.1.0 → v4.2.0 |
| Power | Designed, partial | v1.4.0 → v2.0.0 | v2.0.0 → v2.3.0 |
| Pairing | Designed | v1.2.0 (working) | v1.2.0 (drone), v1.4.0 (dock) |
| Dock Config | Designed | Initial impl | v1.2.0 → v1.5.0 |
| Battery Visual | Designed | v1.0 → v2.2 | Referenced, not changed |
| Waypoint HUD | Designed | v1.0 → v1.1.1 | Full Pathway HUD rewrite |

---

## 4. Power System

### 4.1 Design Philosophy

The power system uses a **hybrid drain model** to simulate realistic power consumption:

- **Base time drain** — power depletes over time regardless of activity
- **Distance drain** — additional cost per meter traveled
- **Activity costs** — fixed costs for specific operations (TP, scan, laser, blast)

A flat timer-only drain was rejected because it would charge the same power for idle hovering and active combat operations. The hybrid model gives the drone meaningful operational trade-offs.

### 4.2 Final Power Constants (v2.3.0 — Production Values)

```lsl
// === POWER DRAIN CONSTANTS ===
float BASE_POWER_DRAIN = 0.069;             // % per minute — 24-hour base runtime
float MOVEMENT_POWER_PER_METER = 0.00006;   // % per meter — 22-hour with movement
float HEIGHT_ADJUSTMENT_COST = 1.5;         // % per terrain height change

// === THRESHOLDS ===
float EMERGENCY_THRESHOLD = 25.0;           // % — triggers coordinate pre-fetch
float CRITICAL_THRESHOLD = 5.0;             // % — triggers emergency return TP

// === CHARGING ===
float CHARGING_INTERVAL = 36.0;             // seconds per 1% charge (1-hour to full)
float SOLAR_CHARGING_INTERVAL = 18.0;       // seconds per 1% with solar (30 min to full)

// === SECURITY COSTS ===
float TP_TELEPORT_COST = 2.0;               // % per TP operation
float AVATAR_SCAN_COST = 0.5;               // % per avatar scanned
float LASER_TARGET_COST = 1.0;              // % for initial laser targeting
float LASER_BLAST_COST = 5.0;               // % for full security blast
```

### 4.3 Runtime Economics

| Operating State | Runtime | Notes |
|----------------|---------|-------|
| Idle (stationary, on) | 24 hours | Base drain only |
| Normal patrol (movement) | ~22 hours | Base + movement cost |
| Emergency mode (25%) | ~8–10 hours remaining | Double consumption |
| Active security (TPs, scans) | ~12–16 hours | Depends on encounter frequency |
| Heavy combat (frequent blasts) | ~8–12 hours | Each blast = 5% cost |

### 4.4 Power Thresholds and Actions

| Power Level | Action | Visual |
|------------|--------|--------|
| > 50% | Normal patrol operation | Hover text green |
| 25–50% | Normal patrol operation | Hover text yellow |
| ≤ 25% | **Emergency mode**: speed reduced to 50%, consumption doubled, coordinate request sent to dock, patrol continues | Hover text red / flashing |
| 5% | **Emergency return triggered**: power forced to 1%, TP to dock coordinates | Emergency emotes |
| 1% | Power forced to exactly 1% before emergency TP (RP design: "just enough to get home") | — |
| Docked | Charging begins, 1% → 100% over 1 hour | Blue glow |

### 4.5 Emergency Mode Detail (≤ 25%)

When power drops to 25%:
1. Drone broadcasts `REQUEST_HOME_COORDINATES:[drone_uuid]` region-wide
2. Dock responds with `HOME_COORDINATES:<pos>|<rot>`
3. Drone stores coordinates silently — does NOT trigger emergency return
4. Patrol continues at reduced speed (50%) with doubled consumption
5. Hover text flashes alternating red/orange every 1 second

Hover text flash colors:
- Flash A: Red `<1.0, 0.255, 0.212>`
- Flash B: Orange `<1.0, 0.522, 0.106>`

### 4.6 Emergency Return Sequence (5%)

1. Power forced to exactly 1% (RP design)
2. Patrol state saved (current waypoint index)
3. KFM timer stopped completely (`llSetTimerEvent(0.0)`)
4. `llSetRegionPos(dock_pos)` executed — teleports drone to dock
5. 3-second delay before sending `MOVEMENT_STARTED` link message (allows state to stabilize)
6. Drone lands at dock, begins charging

### 4.7 Hover Text Display

Format: `[Object Name]\nPower: [bar] [X]%`

Uses `llGetObjectName()` (not hardcoded "SECURITY DRONE") so each named drone shows its own name.

Power bar (20-character unicode bar):
```lsl
// bar_filled = (integer)(current_power / 5.0)
// Example at 60%: Power: [████████████░░░░░░░░] 60%
```

Hover text colors:
| Power Level | Color | Vector |
|------------|-------|--------|
| > 50% | Green | `<0.18, 0.8, 0.251>` |
| 25–50% | Yellow | `<1.0, 0.863, 0.0>` |
| < 25% | Red | `<1.0, 0.255, 0.212>` |

### 4.8 Charging System

**Charge trigger:** Movement Script sends `LANDING_COMPLETE` link message after drone lands at dock.

**Charge rate formula:**
```
charge_rate = 1 / FULL_CHARGE_TIME  (constant regardless of starting level)
FULL_CHARGE_TIME = 3600 seconds (production) / 1800 seconds (solar)
```

| Mode | Seconds per 1% | Total to Full |
|------|---------------|---------------|
| Normal | 36 seconds | 60 minutes |
| Solar panel | 18 seconds | 30 minutes |
| Battery hot-swap (100% battery) | Instant (30s RP sequence) | ~30 seconds |

**Charging visual:**
- Hover text: "CHARGING"
- Eye and hover platform glow: **blue**
- Solar charging: **yellow** glow
- Hot-swap: **cyan** glow

**Charging cycle states:**
1. Drone lands → Movement Script sends `LANDING_COMPLETE`
2. Power Script detects low power → calls `start_charging()`
3. Power Script broadcasts `CHARGING_START:[drone_id]:[type]:[minutes]:[timestamp]` to control panel
4. Power increments by `charge_rate` each second (timer interval = 1.0 second)
5. At 100%: Power Script sends `CHARGING_COMPLETE` link message
6. Config Script receives → sends `START_PATROL` link message
7. Movement Script rises, resumes from saved waypoint

### 4.9 Battery Hot-Swap System (Phase 2)

When the drone docks, it broadcasts its power level (always 1%) to the dock/battery system.

**If battery is at 100%:** Hot-swap sequence begins (30 seconds total):
```
t=0s:  Begin hot-swap
t=10s: /me disconnecting depleted battery pack
t=20s: /me installing fully charged battery pack [battery_id]
t=30s: /me battery hot-swap complete - systems fully powered
       → Power set to 100% instantly
       → Broadcasts HOT_SWAP_COMPLETE:[drone_id]:[battery_id]
```

**If battery is less than 100%:** Battery stops charging. Drone charges normally (36s per 1%). When drone reaches 100%, it sends notification to battery to resume charging.

**If no battery paired:** Normal charging only (no change).

### 4.10 Solar Panel Integration

If a solar panel object is paired to the dock:
- Listens for `SOLAR_AVAILABLE:YES/NO`
- Solar = 18 seconds per 1% (30-minute full charge)
- Visual: yellow eye glow during solar charging

### 4.11 Control Panel Broadcasting

Power Script broadcasts status at multiple intervals for Control Panel integration:

| Message | Trigger | Format |
|---------|---------|--------|
| Regular update | Every 5 minutes | `POWER_UPDATE:[drone_id]:[power%]:[runtime_estimate]` |
| Charging start | When charging begins | `CHARGING_START:[drone_id]:[type]:[minutes]:[timestamp]` |
| Power milestone | Every 10% drop | `POWER_MILESTONE:[drone_id]:[milestone%]:[runtime_remaining]:[estimated_return_time]` |
| Charging complete | When 100% reached | `CHARGING_COMPLETE:[drone_id]:[completion_timestamp]` |

Charging types in broadcast: `NORMAL` (60min), `SOLAR` (30min), `HOT_SWAP` (1min)

Unix timestamps are used for clock script integration (time zones, DST, military time).

### 4.12 Security Power Costs

The drone loses power for every security action. The Power Script listens for these messages from the Eye Security system:

| Message | Power Cost | Action |
|---------|-----------|--------|
| `SECURITY_TELEPORT:[target_id]` | 2.0% | TP to scan an avatar's location |
| `SECURITY_SCAN:[target_id]` | 0.5% | Avatar scan completed |
| `SECURITY_TARGET:[target_id]` | 1.0% | Initial laser targeting (Stage 1) |
| `SECURITY_BLAST:[target_id]` | 5.0% | Full security blast (Stage 2) |

Eye Security has two stages (confirmed from Eye Security session references):
- **Stage 1:** Laser targeting — initial lock-on, small power cost
- **Stage 2:** Massive blast — primary security response, significant power cost

### 4.13 Charge Status Broadcast Format

While charging, drone broadcasts status once per minute:
- Format: `CHARGE_STATUS:[drone_uuid]:[current_power]:[max_power]`
- Example: `CHARGE_STATUS:d4f00f32-30f9-7c82-1cc1-298cc7662b0c:45:100`
- **Dock should IGNORE this message** — it is for the Control Panel only
- Dock Config Script message filter: only responds to `DOCK:`-prefixed commands, `REQUEST_HOME_COORDINATES`, `STATUS`, `PING`, `COORDS`

### 4.14 State Persistence

Power level persists through script resets via linkset data:
```lsl
// Save:
llLinksetDataWrite("current_power", (string)current_power);
// Restore in state_entry():
string saved = llLinksetDataRead("current_power");
if (saved != "") { current_power = (float)saved; }
```

### 4.15 RP_MODE Power Output

When `RP_MODE = YES`, power messages use emote format:
```
/me emergency power mode activated - speed reduced, power consumption doubled
/me connecting to charging station... initiating power transfer
/me fully charged - resuming patrol
```

When `RP_MODE = NO`, all operational power messages are suppressed. Critical errors always display regardless of RP_MODE.


---

## 5. Movement System

### 5.1 Approach History — What Was Rejected

| Approach | Reason Rejected |
|----------|----------------|
| Physics-based movement | Object flopped, lost orientation, rotation spun to non-zero X/Y |
| `llMoveToTarget()` | Requires physics (same problems) |
| Random roaming | LSL scope errors, unpredictable path |
| `llSetRegionPos()` for patrol | Teleports instantly — no smooth movement |

### 5.2 Adopted Approach

Owner **walks a path** with the Pathway HUD worn, stopping at each desired point to record coordinates. These are written to a notecard named **"Tour"** inside the drone. The drone follows waypoints in sequence using `llSetKeyframedMotion()` for smooth movement.

- No collision detection needed — path is pre-walked by owner
- Route loops forever: `0 → 1 → 2 → ... → N → 0`
- Waypoints confirmed clear by the human who walked the path

### 5.3 Movement Constants (v4.2.0)

```lsl
string NOTECARD_NAME = "Tour";
float MOVEMENT_SPEED = 2.0;          // meters per second (normal)
float EMERGENCY_SPEED = 1.0;         // meters per second (emergency mode = 50%)
float HOVER_HEIGHT = 0.25;           // meters above ground during patrol
float height_change_threshold = 1.0; // minimum terrain delta to trigger rise/lower
float WAYPOINT_PAUSE = 0.25;         // seconds at each waypoint before next move (changed from 1.0s)
```

**Note:** Waypoint pause was changed from 1.0 second to **0.25 seconds** to make patrol movement more fluid. With 0.25s pauses the drone spends ~90–95% of its time in motion, requiring the adjusted `MOVEMENT_POWER_PER_METER = 0.00006` to maintain 22-hour runtime.

### 5.4 Tour Notecard Format

Notecard named **"Tour"** inside the drone root prim:

```
# Route: "Master Patrol"
# Generated: 2025-09-14 9794.600000
RP_MODE: YES
ROTATION_AXIS: Z
HOME: 108, 154, 1501
START: 108, 154, 1501, 1.324121
WAYPOINTS:
109, 158, 1501, 1.160719
113, 167, 1501, 1.696881
111, 181, 1501, 1.284317
...
END: 108, 139, 1501
```

**Field notes:**
- `RP_MODE: YES/NO` — controls output format (emotes vs debug or silent)
- `ROTATION_AXIS: Z` — which axis controls facing direction (snail always Z)
- `HOME:` line — **obsolete** since dock pairing provides coordinates dynamically
- Waypoint columns: `X, Y, Z, rotation_angle_radians`
- Rotation angle calculated by HUD: `llAtan2(next_y - from_y, next_x - from_x)`
- `START:` line includes initial facing angle

**Confirmed route examples:**
- "Master Patrol" — 82 waypoints covering a large skybox area
- "New Test" — 12 waypoints for testing

### 5.5 Z-Axis Rotation Rule

The drone only ever rotates on the Z-axis (snail mesh geometry). Setting X or Y rotation causes visual problems.

```lsl
// Correct rotation facing direction of travel:
float z_angle = llAtan2(direction.y, direction.x);
rotation new_rot = llEuler2Rot(<0.0, 0.0, z_angle>);
llSetRot(new_rot);
```

Compass reference (radians): 0.0=East, 1.57=North, 3.14=West, 4.71=South

### 5.6 Timer Intervals

| Activity | Timer Interval |
|----------|---------------|
| Patrol (between waypoint checks) | 0.25 seconds (updated from 1.0s) |
| Rising sequence | 0.1 second |
| Lowering sequence | 0.1 second |
| TP delay (before MOVEMENT_STARTED) | 3.0 seconds |

### 5.7 Rise / Lower Sequences

The drone rises before patrolling and lowers when stopping. Each takes 3 seconds:

```
Rise: 0.0m → 0.25m over 3 seconds (0.1s timer, proportional increments)
Lower: 0.25m → 0.0m over 3 seconds (same rate in reverse)
```

Height formula: `current_height = (rise_timer / 3.0) * HOVER_HEIGHT`

### 5.8 Height Adjustment (Elevation Changes)

Each waypoint stores a terrain Z value (third column). The Movement Script tracks `working_terrain_level` and compares to the next waypoint's Z:

1. Compare `working_terrain_level` to next waypoint terrain Z
2. If difference > `height_change_threshold` (1.0m): trigger rise before crossing, lower after
3. **Delayed lowering fix (v4.0.4):** Lower command delayed by +1 waypoint to prevent premature descent mid-bridge (`lowering_delayed` flag)

### 5.9 Waypoint Advancement

```lsl
current_waypoint++;
if (current_waypoint >= total_waypoints)
{
    current_waypoint = 0;  // loop back to start
}
// Save waypoint to linkset data on every advance:
llLinksetDataWrite("current_waypoint", (string)current_waypoint);
```

Waypoint arrival threshold: within **0.5m** of target coordinates.

### 5.10 Notecard Change Detection (v4.1.0+)

```lsl
changed(integer change)
{
    if (change & CHANGED_INVENTORY)
    {
        key current_key = llGetInventoryKey(NOTECARD_NAME);
        if (current_key != stored_notecard_key && current_key != NULL_KEY)
        {
            stored_notecard_key = current_key;
            reload_notecard();  // clears old waypoints, preserves operational state
        }
    }
}
```

Manual force-reload: `/<channel> reload` command via paired channel.

When notecard is loaded, Movement Script calls `broadcast_rp_mode()` which sends `RP_MODE:YES` or `RP_MODE:NO` to the dock so the dock can suppress/enable its own output.

### 5.11 RP_MODE Communication to Dock

After notecard loads and after pairing completes, Movement Script broadcasts:
```
llRegionSay(comm_channel, "RP_MODE:" + rp_mode);  // "RP_MODE:YES" or "RP_MODE:NO"
```
Dock Config Script v1.5.0 receives this and adjusts its own output accordingly.

### 5.12 State Persistence (LLD Keys — Movement)

| Key | Type | Purpose |
|-----|------|---------|
| `patrol_active` | "0"/"1"/"" | 0=paused, 1=patrolling, ""=fresh |
| `current_waypoint` | integer string | Current waypoint index for resume |
| `working_terrain` | float string | Current terrain level |
| `saved_position` | vector string | Last known position before emergency |
| `emergency_mode` | integer string | Emergency mode flag |
| `debug_mode` | integer string | Persistent debug toggle |

### 5.13 Resume Behavior on Reset

| Saved State | Behavior |
|-------------|----------|
| No saved state (fresh) | Waits for manual start command |
| `patrol_active = "0"` (paused) | Loads position, waits for start |
| `patrol_active = "1"` (active) | Auto-resumes from saved waypoint |

### 5.14 Resume TP Bug and Fix (Critical)

`llSetRegionPos()` behaves differently depending on context:
- In **link_message handler**: works reliably
- In **timer handler**: returns TRUE but drone does NOT actually move

**Root cause:** Rising sequence timer was calling `llSetRegionPos()` continuously; the TP attempt in the same timer context was immediately overridden by the next timer tick.

**Fix:** Stop timer completely before TP, execute TP, then wait 3 seconds before sending `MOVEMENT_STARTED`:
```lsl
// Correct resume TP sequence:
llSetTimerEvent(0.0);               // stop all timers
llSetRegionPos(saved_position);     // teleport
llSleep(3.0);                       // wait for state stabilization
llMessageLinked(LINK_THIS, 0, "MOVEMENT_STARTED", NULL_KEY);
```

Confirmed working: positions before=home, after=WP14, WP33, WP43.

### 5.15 Emergency Return Sequence

Triggered when Power Script sends `EMERGENCY_RETURN` link message (at 5% power):

1. Power Script forces current_power to exactly 1%
2. Movement Script saves current waypoint index to linkset data
3. KFM stopped: `llSetKeyframedMotion([], [])`
4. Approach position calculated (1m in front of dock's Drone_Home prim):
   ```lsl
   float z_radians = dock_euler.z;
   vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
   vector approach_pos = dock_home_position + approach_offset;
   ```
5. Timer stopped; `llSetRegionPos(approach_pos)` executed
6. 3-second delay; dock notified: `DRONE_APPROACHING`
7. Over 10 seconds, drone backs into dock using KFM
8. Drone lowers to ground; `LANDING_COMPLETE` link message sent
9. Power Script initiates charging

### 5.16 Departure Sequence (After Charging)

Triggered by `CHARGING_COMPLETE` link message (NOT by script resets):
1. Rise to hover height (standard 3-second rise)
2. Move 1m forward in dock's facing direction (using dock's Z-axis orientation)
3. Send `DRONE_DEPARTING` to dock via `comm_channel`
4. TP to saved patrol position (if emergency return) OR fresh start from WP0 (if manual)
5. Resume patrol from saved waypoint with 3-second delay before `MOVEMENT_STARTED`

### 5.17 Copied Drone Fix

When a drone is first paired to its dock:
- Config Script sends `CLEAR_SAVED_STATE` link message
- Movement Script clears stored waypoint index and terrain data
- Prevents copied drones from inheriting the original's last patrol position

### 5.18 Control Panel Commands

Movement Script handles these control panel commands (Phase 2):
- `CONTROL_PANEL_RETURN` — Emergency return without auto-resume
- `CONTROL_PANEL_STOP` — Land immediately and stop
- `CONTROL_PANEL_START` — Resume patrol operations

### 5.19 Emergency Mode Speed

When Power Script sends `EMERGENCY_MODE` link message:
- Movement speed reduced to **50% = 1.0 m/s**
- Power consumption doubled
- Patrol continues (drone does not stop)

When Power Script sends `NORMAL_MODE`:
- Speed restored to 2.0 m/s

---

## 6. Dock System

### 6.1 Design Principles

- One dock pairs with one drone (one-to-one, never shared)
- Pairing is owner-validated (only pairs with objects that have the same owner)
- Dock provides its own coordinates dynamically — no hardcoded HOME in the drone notecard
- All dock-drone communication uses the unique calculated `comm_channel`
- Dock communicates region-wide (`llRegionSay`) — distance doesn't break the connection

### 6.2 Dock Object Structure

The dock linkset contains one special child prim:

| Prim Name | Purpose |
|-----------|---------|
| `Drone_Home` | Defines exact docking target position and orientation. Always uses `0, 0, Z` rotation — no X or Y tilt. |

**Physical offset:** Drone center sits **0.10075m** from Drone_Home prim center (Y-axis in local space).

Dock test positions observed in development:
- Drone: `<103.50000, 144.00000, 1500.31970>`
- Drone_Home prim: `<103.39925, 143.89925, 1500.31970>` (offset 0.10075m)

### 6.3 Dock Config Script (v1.5.0)

**Responsibilities:**
- Listens on calculated `comm_channel` for drone requests
- Responds to `REQUEST_HOME_COORDINATES` with current Drone_Home prim position and rotation
- Message filtering — silently ignores unrecognized messages
- RP_MODE awareness: when drone sets `RP_MODE: NO`, dock suppresses operational output
- On owner touch (when paired): privately displays `comm_channel` to owner via `llRegionSayTo(llGetOwner(), 0, ...)`

**Version history:**

| Version | Key Change |
|---------|-----------|
| v1.2.0 | Region-wide communication, basic command handling |
| v1.3.0 | `is_dock_message()` filter — silently ignores unknown messages (e.g., `CHARGE_STATUS`) |
| v1.4.0 | Owner touch shows channel privately; post-pairing instructions |
| v1.5.0 | RP_MODE listening; `drone_rp_mode` variable; `dock_output()` for filtered output; persistent RP storage |

**Commands recognized (v1.3.0+):**

| Command | Response |
|---------|----------|
| `REQUEST_HOME_COORDINATES` | Broadcasts `HOME_COORDINATES:<pos>\|<rot>` on `comm_channel` |
| `STATUS` / `DOCK:STATUS` | Status report |
| `PING` | `PONG` |
| `DOCK:PING` | `DOCK:PONG` |
| `COORDS` / `DOCK:COORDS` | `DOCK:CURRENT_COORDS:<pos>` |
| `RP_MODE:YES` or `RP_MODE:NO` | Stored; adjusts dock output filtering immediately |

**Silent ignore list:** `CHARGE_STATUS`, `BATTERY:`, and anything not prefixed with `DOCK:` or not in the legacy command list.

**RP_MODE output rules (v1.5.0):**
- `dock_output()` function: when `drone_rp_mode == "NO"`, suppresses operational messages
- Critical messages (pairing events, errors) always display regardless of RP_MODE

**Variables stored in linkset data (dock side):**
- `paired_drone_uuid` — UUID of paired drone
- `comm_channel` — calculated unique channel
- `drone_home_coords` — last known Drone_Home position
- `rp_mode` — received RP_MODE from drone (persists across resets)

### 6.4 Dock Pairing Script (v1.4.0)

**Responsibilities:**
- Owner touches dock → dock enters pairing mode for 60 seconds
- Broadcasts ping to nearby drones on channel 0 (public)
- Drone's Pairing Script responds with its UUID
- Calculates unique `comm_channel` from UUID hash
- Sends Drone_Home prim position and rotation to drone
- Saves pairing data to linkset data on both sides
- Admin listener on calculated channel: `reset` and `status` commands

**Pairing process detail:**
1. Owner touches dock
2. Dock broadcasts: `DOCK_PAIRING:[dock_key]` on channel 0
3. Nearby drone Pairing Script responds: `DRONE_PAIR:[drone_key]`
4. Both sides validate owner UUIDs match (no cross-owner pairing)
5. Unique `comm_channel` calculated from drone UUID hash
6. Drone_Home prim position and rotation sent to drone
7. Pairing data saved to linkset data on both sides
8. Pairing Scripts intended to delete themselves on success (disabled in test builds)
9. 1-minute timeout on dock side — but script persists to allow retry

**Version history:**

| Version | Key Change |
|---------|-----------|
| v1.1.0 | Basic pairing, finds Drone_Home prim, sends coordinates |
| v1.2.0 | `pairing_completed` flag prevents re-pairing; self-delete disabled for testing |
| v1.4.0 | Admin listener on calculated channel; `reset` clears all linkset data; `status` reports pairing state |

**Why v1.4.0 was needed:**
- Linkset data survives script resets — earlier versions blocked re-pairing permanently
- Fix: Admin listener always set up on calculated channel regardless of pairing state
- `reset` command: `llLinksetDataDelete()` clears all stored pairing data

### 6.5 Channel Architecture

```lsl
// Unique comm_channel calculation:
integer comm_channel = (integer)("0x" + llGetSubString((string)llGetKey(), 0, 6)) * -1;
```

Example results: `-381513550` and `-233798746`

| Channel | Used For |
|---------|----------|
| `0` (public) | Initial pairing broadcast (60 seconds only) |
| `comm_channel` (calculated) | All ongoing drone/dock communication; admin commands |
| `0` via `llRegionSayTo(owner)` | Dock privately tells owner the channel number on touch |

### 6.6 Dock Communication Protocol

**Coordinate request flow (at 25% power):**
1. Power Script sends `POWER_CRITICAL` link message to Config Script
2. Config Script sends on `comm_channel`: `REQUEST_HOME_COORDINATES:[drone_UUID]`
3. Dock Config receives → logs `*** COORDINATE REQUEST RECEIVED ***`
4. Dock broadcasts: `HOME_COORDINATES:<x,y,z>|<rotation_quaternion>`
5. Config receives → calls `update_home_position(pos, rot, TRUE)` (emergency flag = TRUE, no movement)
6. Config sends `UPDATE_HOME_COORDINATES` link message to Movement Script
7. Movement Script updates `emergency_home_position` and `emergency_home_rotation`

**Example coordinate message observed:**
```
HOME_COORDINATES:<101.10080, 143.86980, 1500.57500>|<0.00000, 0.70711, 0.00000, 0.70711>
```

**Config Script fix (v3.0.0+):** Coordinate update sends `UPDATE_HOME_COORDINATES` (not `HOME_POSITION_DATA`) so it doesn't accidentally trigger movement.

**Emergency approach/departure messages:**

| Message | Direction | Trigger |
|---------|-----------|---------|
| `DRONE_APPROACHING` | Drone → Dock | After TP to approach position |
| `DRONE_DEPARTING` | Drone → Dock | After departure sequence begins |

### 6.7 Docking Alignment — Rotation-Aware Approach

The dock can be placed at any compass orientation. The drone calculates the correct approach using the Drone_Home prim's Z-axis rotation:

```lsl
// Extract Z-only rotation from dock:
rotation convert_dock_rotation(rotation dock_rot)
{
    vector euler = llRot2Euler(dock_rot);
    rotation converted = llEuler2Rot(<0.0, 0.0, euler.z>);
    return converted;
}

// Calculate approach position (1m in front of dock):
float z_radians = euler.z;
vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
vector approach_pos = dock_home_position + approach_offset;
```

Cardinal direction mapping:
| Dock Facing | euler.z | approach_offset |
|-------------|---------|----------------|
| East | 0° | `<1.0, 0.0, 0.0>` |
| North | 90° | `<0.0, 1.0, 0.0>` |
| West | 180° | `<-1.0, 0.0, 0.0>` |
| South | 270° | `<0.0, -1.0, 0.0>` |
| NE | 45° | `<0.707, 0.707, 0.0>` |

### 6.8 Dock Pairing LLD Keys (Both Sides)

**Dock side:**

| Key | Content |
|-----|---------|
| `paired_drone_uuid` | UUID of paired drone |
| `comm_channel` | Calculated unique channel |
| `drone_home_coords` | Last known Drone_Home position |
| `rp_mode` | RP_MODE received from drone |

**Drone side:**

| Key | Content |
|-----|---------|
| `paired_dock_uuid` | UUID of paired dock |
| `dock_channel` | Calculated `comm_channel` |
| `home_position` | Vector string: dock Drone_Home position |
| `home_rotation` | Rotation string: dock Drone_Home orientation |

### 6.9 Config Script Bugs Fixed (Related to Dock)

**Double-click to start bug (v3.0.0 → v3.0.1):**
- Cause: `patrol_active` not explicitly set to `FALSE` after pairing completion
- Fix: Added `patrol_active = FALSE;` in Config Script pairing completion handler

**Copied drone resumes at original's position:**
- Cause: Linkset data from original drone persists in the copy
- Fix: Send `CLEAR_SAVED_STATE` link message after successful pairing; Movement Script clears state

### 6.10 Phase 2: Planned Dock Enhancements

**Battery pairing:** Battery object pairs to dock separately from drone. Dock mediates between drone power needs and battery state.

**Control Panel pairing:** Control Panel pairs to dock to receive the `comm_channel`. Then listens on that channel for status broadcasts from the drone.

**Proximity docking (planned):** When power < 25% AND distance to dock < 20m, initiate smooth approach instead of emergency TP. Replaces the abrupt TP-home with a graceful approach.

---

## 7. Security System

> **Status:** Designed and power costs wired into Power Script v2.3.0. Full Security and Scan scripts are planned for Phase 3. Access list structure is defined. Eye Security integration (laser targeting) is confirmed planned but not yet implemented.

### 7.1 Access Control Hierarchy

Avatars are evaluated in priority order. Higher priority = processed first.

| Priority | Class | Storage | Notes |
|----------|-------|---------|-------|
| 1 | **Owner** | `llGetOwner()` (live check) | Never stored; always checked at runtime |
| 2 | **Admin** | `llLinksetData("admin_list")` | Pipe-delimited UUIDs |
| 3 | **Group** | `llLinksetData("security_group")` | Single group UUID; `llSameGroup()` check |
| 4 | **Guest** | `llLinksetData("guest_list")` | Permanent approval |
| 5 | **Temp Guest** | `llLinksetData("temp_guest_list")` | Cleared when avatar leaves region |
| 6 | **Banned** | `llLinksetData("banned_list")` | Permanent ban; triggers security response |
| 7 | **Unknown** | Not stored until decision | Full scan; triggers admin notification |

### 7.2 Security Lists (LLD Keys)

| List | Key | Format |
|------|-----|--------|
| Admin list | `admin_list` | Pipe-delimited UUIDs: `"uuid1\|uuid2\|uuid3"` |
| Guest list | `guest_list` | Pipe-delimited UUIDs |
| Temp Guest list | `temp_guest_list` | Pipe-delimited UUIDs; purged on region leave |
| Banned list | `banned_list` | Pipe-delimited UUIDs |
| Security group | `security_group` | Single group UUID |

### 7.3 Power Costs for Security Actions

| Action | Power Cost | Notes |
|--------|-----------|-------|
| Owner/Admin/Group/Guest identification | 1.0% | Identification only |
| Banned user detection | 0.5% | Quick detection, response triggered |
| Unknown avatar full scan | 1.0% + (0.3% × script count) | Script count = scripts on avatar |
| Avatar scan (Power Script flat rate) | 0.5% (`AVATAR_SCAN_COST`) | Used in v2.3.0 simplified model |
| Security TP to avatar location | 2.0% (`TP_TELEPORT_COST`) | Per TP operation |
| Laser targeting (Stage 1) | 1.0% (`LASER_TARGET_COST`) | Initial lock-on |
| Laser blast (Stage 2) | 5.0% (`LASER_BLAST_COST`) | Full security response |

### 7.4 Eye Security Integration (Two-Stage System)

The Eye Security system (a separate integrated object) provides two stages of response to banned avatars:

- **Stage 1 — Laser Target:** Initial lock-on; small power cost (1.0%); visual targeting effect
- **Stage 2 — Massive Blast:** Full security blast; significant power cost (5.0%); major visual effect

The Power Script listens for these messages from the Eye Security system:

```lsl
// Power Script listener for Eye Security:
"SECURITY_TELEPORT:[target_id]"  → deduct 2.0%
"SECURITY_SCAN:[target_id]"      → deduct 0.5%
"SECURITY_TARGET:[target_id]"    → deduct 1.0%
"SECURITY_BLAST:[target_id]"     → deduct 5.0%
```

### 7.5 Admin Notification Format

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

### 7.6 Warning / Ejection System

- Minimum warning time: **10 seconds**
- Available presets: 10s, 20s, 30s, or custom (user-specified, min 10s)
- After warning timer: ejection or ban action executes

### 7.7 Detection Persistence Rules

| List | Persistence |
|------|------------|
| Admin / Guest / Banned | Permanent — survives resets via `llLinksetData` |
| Temp Guest | Cleared when avatar is detected leaving the region |
| Scanned cache | Purged hourly OR when avatar leaves region |

### 7.8 RP_MODE Security Output

When `RP_MODE: YES`:
- Security events output as `/me` emotes on local channel 0
- Examples: `/me detects unknown presence`, `/me initiating security scan`
- No raw debug output in local chat

When `RP_MODE: NO`:
- `llOwnerSay()` debug output only (owner-visible only)
- No public chat messages

### 7.9 Security TP System

When a security TP is required (scanning a suspicious avatar at their location):
1. Power Script deducts `TP_TELEPORT_COST` (2.0%)
2. Movement Script TPs drone to avatar location
3. Security scan performed
4. Power Script deducts `AVATAR_SCAN_COST` (0.5%)
5. Movement Script TPs drone back to last patrol position (3-second delay before `MOVEMENT_STARTED`)

This TP pattern uses the same fix as the resume TP: stop timer, execute TP, wait 3 seconds.

### 7.10 Future: ID Card System (Planned)

- Optional attachment for pre-authorized visitors
- Broadcasts on security channel when entering monitored zone — auto-approves without full scan
- Reduces power cost for pre-authorized avatars
- Eliminates admin notification for known trusted visitors

---

## 8. All Code Blocks

All code blocks are preserved completely and untruncated. Organized by source and system.

---

### CB-01: Notecard Reader / Waypoint Parser

*Source: Movement Script v2.0.1 — dataserver handler*

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

*Source: Movement Script v2.0.1 / v4.0.4*

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

*Source: Movement Script*

```lsl
// Calculate rotation to face direction of travel
vector direction = llVecNorm(movement);
float z_angle = llAtan2(direction.y, direction.x);
rotation new_rot = llEuler2Rot(<0.0, 0.0, z_angle>);
llSetRot(new_rot);
```

---

### CB-04: Touch Start — Stop / Start with Rise / Lower

*Source: Movement Script*

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

*Source: Movement Script*

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

*Source: Power Script (early design)*

```lsl
// Global power variables
float current_power;
float base_power_drain;        // % per minute
float distance_power_rate;     // % per meter
float last_power_update;       // llGetTime() timestamp
integer emergency_mode;        // TRUE/FALSE
integer flash_state;           // for flashing hover text

// In state_entry():
current_power = 100.0;
base_power_drain = 0.069;      // 0.069% per minute = 24hr base runtime (FINAL)
distance_power_rate = 0.00006; // 0.00006% per meter = 22hr with movement (FINAL)
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

*Source: Power Script v2.0.0+*

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

// Uses object name dynamically (not hardcoded "SECURITY DRONE"):
string display_text = llGetObjectName() + "\nPower: " + power_bar;
llSetText(display_text, text_color, 1.0);
```

---

### CB-08: Security Scan Power Consumption

*Source: Security system design*

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

*Source: Movement Script (early design)*

```lsl
vector calculate_docking_position(vector dock_pos, rotation dock_rot)
{
    vector dock_euler = llRot2Euler(dock_rot) * RAD_TO_DEG;
    vector position_offset = ZERO_VECTOR;

    if (llFabs(dock_euler.x) > 45.0)  // X-axis rotation (270 deg)
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

*Source: Movement Script*

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

*Source: Pairing Script*

```lsl
// Pairing channel calculation (from owner UUID hash)
integer pairing_channel = (integer)("0x" + llGetSubString((string)llGetOwner(), 0, 7));

// Dock broadcasts pairing signal on channel 0:
llRegionSay(0, "DOCK_PAIRING:" + (string)llGetKey());

// Drone Pairing Script listens and responds:
listen(integer channel, string name, key id, string message)
{
    if (llSubStringIndex(message, "DOCK_PAIRING:") == 0)
    {
        string dock_key_str = llGetSubString(message, 13, -1);
        key dock_key = (key)dock_key_str;
        if (llGetOwnerKey(dock_key) == llGetOwner())
        {
            // Owner validated — respond with drone UUID
            llRegionSay(0, "DRONE_PAIR:" + (string)llGetKey());
            // Store dock reference, start glowing
        }
    }
}

// After successful pairing — store in linkset data:
llLinksetDataWrite("paired_dock_uuid", (string)dock_uuid);
llLinksetDataWrite("dock_channel", (string)dock_channel);
```

---

### CB-12: Emergency Return Sequence (link_message Context)

*Source: Movement Script*

```lsl
// In link_message handler (llSetRegionPos works reliably here):
else if (str == "EMERGENCY_RETURN")
{
    // 1. Save patrol state
    llLinksetDataWrite("patrol_active", "0");
    llLinksetDataWrite("current_waypoint", (string)current_waypoint);

    // 2. Stop current motion
    llSetKeyframedMotion([], []);

    // 3. Stop timer before TP (critical fix)
    llSetTimerEvent(0.0);

    // 4. Calculate approach position (1m in front of dock)
    vector euler = llRot2Euler(emergency_home_rotation);
    float z_radians = euler.z;
    vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
    vector approach_pos = emergency_home_position + approach_offset;

    // 5. TP to approach position
    llSetRegionPos(approach_pos);
    llSetRot(emergency_home_rotation);

    // 6. Notify dock
    llRegionSay(dock_channel, "DRONE_APPROACHING");

    // 7. Begin 10-second backup sequence into dock
    vector backup_movement = emergency_home_position - approach_pos;
    float backup_time = 10.0;
    llSetKeyframedMotion([backup_movement, ZERO_ROTATION, backup_time], [KFM_MODE, KFM_FORWARD]);

    emergency_landing_sequence = TRUE;
    llSetTimerEvent(backup_time + 0.5);
}
```

---

### CB-13: Global Variables Declaration Pattern (Movement Script)

*Source: Movement Script v2.0.3+*

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
key stored_notecard_key;

// Communication
integer comm_channel;
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
integer home_coordinates_received;

// Debug
integer DEBUG_MODE;

// Working terrain
float working_terrain_level;
integer lowering_delayed;
```

---

### CB-14: update_home_position() — Emergency vs Normal Update

*Source: Config Script*

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

*Source: Movement Script*

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

### CB-16: LANDING_COMPLETE → CHARGING_COMPLETE → AUTO-RESUME Chain

*Source: Movement Script / Power Script / Config Script*

```lsl
// Movement Script — after drone lands:
llOwnerSay("Landed");
llMessageLinked(LINK_THIS, 0, "LANDING_COMPLETE", NULL_KEY);

// Power Script — on LANDING_COMPLETE:
else if (str == "LANDING_COMPLETE")
{
    if (current_power <= 10.0)
    {
        llOwnerSay("POWER: Landing detected at " + (string)((integer)current_power) + "% - initiating charging");
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

*Source: Movement Script*

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

*Source: Movement Script (execute_emergency_return)*

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

*Source: Power Script*

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

> **Test mode only:** `charge_rate = (power_needed / charge_time) * 2.5;` — 2.5x multiplier compensates for LSL timer unreliability. **Remove for production.**

---

### CB-20: Battery Status Indicator Colors

*Source: Battery Visual Script*

```lsl
update_status_indicators()
{
    integer total_links;
    integer i;
    vector indicator_color;
    float indicator_glow;
    string link_name;

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

### CB-21: HUD Glow-Before-Alpha Fix (Two-Pass Minimize)

*Source: Waypoint HUD v1.1.1*

```lsl
// When minimizing — MUST turn off glow before setting alpha to 0
// Pass 1: Turn off all glows first
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

*Source: Waypoint HUD*

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

*Source: Movement Script v4.1.0*

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

*Source: Movement Script*

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

### CB-25: Type Conversion Fix — key to integer

*Source: Config Script (bug fix)*

```lsl
// WRONG (type mismatch — cannot cast key directly to integer):
integer minutes_remaining = (integer)id;

// CORRECT (key to string to integer, two steps):
string minutes_str = (string)id;
integer minutes_remaining = (integer)minutes_str;
```

---

### CB-26: Dock Config — is_dock_message() Filtering (v1.3.0)

*Source: Dock Config Script v1.3.0*

```lsl
// Silently ignore CHARGE_STATUS broadcasts from drone (dock does not need them)
else if (llSubStringIndex(processed_message, "CHARGE_STATUS:") >= 0)
{
    // Parse if needed but do NOT log "unknown message"
    list charge_parts = llParseString2List(processed_message, [":"], []);
    if (llGetListLength(charge_parts) >= 4)
    {
        string drone_uuid = llList2String(charge_parts, 1);
        integer current_p = (integer)llList2String(charge_parts, 2);
        integer max_p = (integer)llList2String(charge_parts, 3);
    }
    // Silently ignore — do not respond
}
```

---

### CB-27: Patrol Active Fix After Pairing (Config Script v3.0.1)

*Source: Config Script v3.0.1*

```lsl
// After pairing completion — explicitly set patrol_active to FALSE
// so that first touch after pairing STARTS patrol (not stops it)
patrol_active = FALSE;
```

---

### CB-28: Waypoint HUD — Rotation Angle Calculation

*Source: Pathway HUD*

```lsl
// "rotation" is a reserved LSL type — use "angleToNext" or similar
float calculateAngle(vector fromPos, vector toPos)
{
    float angleToNext = llAtan2(toPos.y - fromPos.y, toPos.x - fromPos.x);
    return angleToNext;
}
```

---

### CB-29: Power Script — Final Constants (v2.3.0)

*Source: Power Script v2.3.0*

```lsl
// === POWER DRAIN ===
float BASE_POWER_DRAIN = 0.069;            // % per minute — 24-hour base runtime
float MOVEMENT_POWER_PER_METER = 0.00006; // % per meter — 22-hour with movement
float HEIGHT_ADJUSTMENT_COST = 1.5;       // % per terrain change

// === CHARGING ===
float CHARGING_INTERVAL = 36.0;           // seconds per 1% — 1 hour total
float SOLAR_CHARGING_INTERVAL = 18.0;     // seconds per 1% — 30 minutes total

// === THRESHOLDS ===
integer EMERGENCY_THRESHOLD = 25;         // % — coord request, speed reduced
integer CRITICAL_THRESHOLD = 5;           // % — emergency TP triggered

// === SECURITY COSTS ===
float TP_TELEPORT_COST = 2.0;            // % per security TP
float AVATAR_SCAN_COST = 0.5;            // % per avatar scan
float LASER_TARGET_COST = 1.0;           // % initial laser targeting (Stage 1)
float LASER_BLAST_COST = 5.0;            // % full security blast (Stage 2)
```

---

### CB-30: Waypoint Pause Timing — Production vs Test

*Source: Movement Script*

```lsl
// TEST (old) — 1 second pause at each waypoint:
llSetTimerEvent(1.0);

// PRODUCTION (updated) — 0.25 second pause (much smoother patrol):
llSetTimerEvent(0.25);
// Location: Movement Script, at end of waypoint-advance block
// Effect: ~95% moving time instead of ~50%, far more fluid appearance
```

---

### CB-31: RP_MODE Broadcast to Dock (v4.2.0 — New)

*Source: Movement Script v4.2.0 — broadcast_rp_mode()*

```lsl
// Called after notecard loads AND after pairing completes
broadcast_rp_mode()
{
    string rp_message;
    if (rp_mode == TRUE)
    {
        rp_message = "RP_MODE:YES";
    }
    else
    {
        rp_message = "RP_MODE:NO";
    }
    llRegionSay(comm_channel, rp_message);
    if (debug_mode) llOwnerSay("MOVEMENT: Broadcast " + rp_message + " to dock");
}
```

---

### CB-32: dock_output() — RP-Aware Output Filter (Dock Config v1.5.0 — New)

*Source: Dock Config Script v1.5.0*

```lsl
// Variables at global scope:
string drone_rp_mode;     // "YES" or "NO"
integer rp_mode_received; // TRUE/FALSE

dock_output(string message, integer always_show)
{
    if (always_show == TRUE)
    {
        llOwnerSay(message);
        return;
    }
    if (drone_rp_mode == "NO")
    {
        return;  // Suppress operational messages when RP is off
    }
    llOwnerSay(message);
}

// Listener for RP_MODE broadcasts from drone:
// In listen() event:
if (message == "RP_MODE:YES" || message == "RP_MODE:NO")
{
    drone_rp_mode = llGetSubString(message, 8, -1);
    rp_mode_received = TRUE;
    llLinksetDataWrite("rp_mode", drone_rp_mode);
    llOwnerSay("DOCK: RP_MODE set to " + drone_rp_mode);
}
```

---

### CB-33: Resume TP Fix — Stop Timer + 3-Second Delay (New)

*Source: Movement Script (critical bug fix)*

```lsl
// BROKEN (timer context — returns TRUE but drone doesn't move):
// llSetRegionPos(saved_position);  // Called inside timer handler — UNRELIABLE

// CORRECT:
execute_resume_tp()
{
    llSetTimerEvent(0.0);               // Stop ALL timers first
    llSetRegionPos(saved_position);     // Now TP works reliably
    llSetRot(saved_rotation);
    // Wait 3 seconds before resuming movement:
    llSleep(3.0);
    llMessageLinked(LINK_THIS, 0, "MOVEMENT_STARTED", NULL_KEY);
}
```

---

### CB-34: Battery Hot-Swap Emote Sequence (Phase 2 Design — New)

*Source: Power Script v2.3.0 design*

```lsl
// Hot-swap sequence (30 seconds, RP_MODE emotes every 10 seconds)
// Triggered when: drone docks at 1% AND battery is at 100%

initiate_hot_swap()
{
    hot_swap_active = TRUE;
    hot_swap_step = 0;
    hot_swap_timer = 0;
    // Set cyan eye glow for hot-swap visual
    llMessageLinked(LINK_THIS, 0, "SET_EYE_COLOR", "CYAN");
    llSetTimerEvent(10.0);  // Step every 10 seconds
}

// In timer handler:
if (hot_swap_active)
{
    hot_swap_step++;
    if (hot_swap_step == 1)
    {
        llSay(0, "/me disconnecting depleted battery pack");
    }
    else if (hot_swap_step == 2)
    {
        llSay(0, "/me installing fully charged battery pack " + hot_swap_battery_id);
    }
    else if (hot_swap_step == 3)
    {
        llSay(0, "/me battery hot-swap complete - systems fully powered");
        current_power = 100.0;
        hot_swap_active = FALSE;
        llRegionSay(comm_channel, "HOT_SWAP_COMPLETE:" + (string)llGetKey() + ":" + hot_swap_battery_id);
        // Resume patrol
        llMessageLinked(LINK_THIS, 0, "CHARGING_COMPLETE", NULL_KEY);
    }
}
```

---

### CB-35: Control Panel Power Broadcast Functions (Phase 2 Design — New)

*Source: Power Script v2.3.0 (enhanced)*

```lsl
// Broadcast every 5 minutes for Control Panel:
broadcast_power_update()
{
    string runtime_est = get_runtime_estimate();
    llRegionSay(comm_channel, "POWER_UPDATE:" + (string)llGetKey() + ":" +
                (string)((integer)current_power) + ":" + runtime_est);
}

// Broadcast at charging start:
broadcast_charging_start(string charge_type, integer total_minutes)
{
    string timestamp = (string)llGetUnixTime();
    llRegionSay(comm_channel, "CHARGING_START:" + (string)llGetKey() + ":" +
                charge_type + ":" + (string)total_minutes + ":" + timestamp);
}

// Broadcast every 10% power drop:
broadcast_power_milestone(integer milestone)
{
    string runtime_remaining = get_runtime_estimate();
    string est_return = calculate_estimated_return_time();
    llRegionSay(comm_channel, "POWER_MILESTONE:" + (string)llGetKey() + ":" +
                (string)milestone + ":" + runtime_remaining + ":" + est_return);
}

// Broadcast at charging complete:
broadcast_charging_complete()
{
    string timestamp = (string)llGetUnixTime();
    llRegionSay(comm_channel, "CHARGING_COMPLETE:" + (string)llGetKey() + ":" + timestamp);
}
```

---

### CB-36: Comm Channel Calculation (Both Sides)

*Source: Pairing Script / Dock Pairing Script v1.4.0*

```lsl
// From drone UUID:
integer comm_channel = (integer)("0x" + llGetSubString((string)llGetKey(), 0, 6)) * -1;

// Example:
// Drone UUID: d4f00f32-30f9-7c82-1cc1-298cc7662b0c
// Prefix: d4f00f3
// Hex to int: 0xd4f00f3 = 222972147
// * -1 = -222972147
// Actual observed: -233798746 (slight variation based on exact UUID prefix used)

// Alternative pattern observed (from owner UUID):
integer pairing_channel = (integer)("0x" + llGetSubString((string)llGetOwner(), 0, 7));
// Example result: -381513550
```

---

### CB-37: Waypoint HUD Output Format (82-Waypoint Route Example)

*Source: Pathway HUD — confirmed working output*

```
[18:10] Pathway HUD: =====Copy and Paste the following=====
# Route: "Master Patrol"
# Generated: 2025-09-14 9656.548000
RP_MODE: YES
ROTATION_AXIS: Z
HOME: 108, 154, 1501
START: 108, 154, 1501, 1.324121
WAYPOINTS:
[18:10] Pathway HUD: 109, 158, 1501, 1.160719
113, 167, 1501, 1.696881
111, 181, 1501, 1.284317
115, 194, 1501, 0.751799
... (30 waypoints per chunk)
[18:10] Pathway HUD: =====Continued=====
94, 80, 1501, 2.437584
88, 85, 1501, 1.910957
... (30 more waypoints)
[18:10] Pathway HUD: =====Continued=====
69, 132, 1503, -3.073587
... (remaining waypoints)
[18:10] Pathway HUD: END: 108, 139, 1501
```

Output rules:
- Header message: route metadata + "WAYPOINTS:" line
- First chunk: waypoints 1-30 (no continuation header)
- Subsequent chunks: "=====Continued=====" prefix + next 30 waypoints
- Final message: "END: X, Y, Z" line
- Uses `llOwnerSay()` (not `llRegionSayTo()`) for higher character limit

---

