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

