# GASTROPOD Bible 3 — Extraction from "Gasgropod last 3 chats.txt"

> Source file: `Gasgropod last 3 chats.txt` (2154 lines)
> Date range of conversations: September 13–26, 2025
> Topics: Dock Config fixes, Dock Pairing Script, Pathway Waypoint HUD, RP Mode system, Power Script overhaul

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [LSL Forbidden Rules](#2-lsl-forbidden-rules)
3. [Script Architecture](#3-script-architecture)
4. [Dock System](#4-dock-system)
5. [Pathway Waypoint HUD](#5-pathway-waypoint-hud)
6. [Power System](#6-power-system)
7. [Movement System](#7-movement-system)
8. [RP Mode System](#8-rp-mode-system)
9. [Link Message Strings and Channels](#9-link-message-strings-and-channels)
10. [Bugs and Resolution Status](#10-bugs-and-resolution-status)
11. [Design Decisions](#11-design-decisions)
12. [All Code Blocks](#12-all-code-blocks)

---

## 1. System Overview

G.A.S.T.R.O.P.O.D. (Guided Autonomous Security Tactical Response Operations Drone) is a Second Life scripted object system consisting of multiple paired components:

- **Drone** (e.g., "G.A.S.T.R.O.P.O.D. mrk IV") — the patrol/security unit
- **Dock** ("G.A.S.T.R.O.P.O.D. Dock") — charging and communication hub
- **Battery** — paired to dock, provides hot-swap charging
- **Console Panel** — paired to dock, monitors drone status
- **Pathway HUD** — worn by owner/builder, records patrol routes

The drone contains multiple scripts:
- Configuration Script (v3.x)
- Movement Script (v4.x)
- Power Script (v2.x)
- (Security scripts referenced but not fully shown in this file)

---

## 2. LSL Forbidden Rules

The following rules were repeatedly enforced throughout all sessions:

1. **No ternary operators**
   - Forbidden: `string name = (id == llGetOwner()) ? "Owner" : "Guest";`
   - Must use standard `if/else`

2. **No foreach loops**
   - Must always use traditional indexed loops:
     ```lsl
     integer i;
     for (i = 0; i < llGetListLength(myList); ++i) { ... }
     ```

3. **No global assignments from function calls**
   - Forbidden: `string ownerName = llKey2Name(llGetOwner());`
   - Only declare globals with literals or empty values
   - Assignments from functions must occur inside `state_entry` or other events

4. **No `void` keyword** for event/function declarations (LSL doesn't use C-style void)

5. **No direct use of `PRIM_HOVER_HEIGHT`** — must use `llSetHoverHeight()` / `llGetHoverHeight()`

6. **All variable declarations must be scoped inside functions/events** unless they are constants
   - Globals only for config constants or empty placeholders
   - Avoid runtime values in global space

7. **No reliance on object name, description, or notecards for invisible storage** unless explicitly required
   - Persistent or cross-script data must use `llLinksetData*` or `llGetObjectDesc()` only when explicitly desired

8. **No sneaky shorthand**

9. **No reserved keywords as variable names**
   - `state`, `rotation`, `vector`, `integer`, `float`, `string`, `key`, `list` — all are reserved
   - Confirmed fix: `state` → `stateValue`; `rotation` → `rotAngle` / `angleValue`

10. **`llGetTimeOfDay()` returns float** — must cast to string before string concatenation

---

## 3. Script Architecture

### Drone Scripts

| Script | Version | Responsibility |
|--------|---------|----------------|
| Configuration Script | v3.0.0 / v3.0.1 / v3.1.0 | Central command hub: pairing, coordination, admin commands, patrol start/stop, RP mode, debug toggle |
| Movement Script | v4.1.0 / v4.2.0 | Reads "Tour" notecard, manages waypoints, rotation axis conversion, RP mode support, notecard change detection, broadcasts RP_MODE to dock |
| Power Script | v2.0.0 / v2.1.0 / v2.2.0 / v2.3.0 | Power management, charging cycles, visual indicators, charge broadcasting, battery swap communication, security power costs, control panel broadcasting |

### Dock Scripts

| Script | Version | Responsibility |
|--------|---------|----------------|
| Dock Config Script | v1.2.0 / v1.3.0 / v1.4.0 / v1.5.0 | Communication with paired drone, message filtering, channel display on touch, RP_MODE listening and output suppression |
| Dock Pairing Script | v1.1.0 / v1.2.0 / v1.4.0 | Pairing dock with drone, unique channel generation, admin listener for reset/status commands |

### HUD Scripts

| Script | Version | Responsibility |
|--------|---------|----------------|
| Pathway (root prim controller) | v1.0 (multiple iterations) | Full waypoint recording HUD: route name, RP mode, rotation axis, start/waypoint/end, resume, output generation, min/max toggle, light system |

---

## 4. Dock System

### Dock Config Script — Key Behaviors

**Version history and key changes:**

- **v1.2.0**: Initial version with region-wide communication, basic command handling
- **v1.3.0**: Added `is_dock_message()` filtering — only accepts DOCK:-prefixed or legacy commands; silently ignores all other messages (CHARGE_STATUS, BATTERY:, etc.)
- **v1.4.0**: Added paired-touch behavior — when paired and owner touches, uses `llRegionSayTo(llGetOwner(), 0, ...)` to privately show communication channel only to owner. Added explicit post-pairing instruction to owner.
- **v1.5.0**: Added RP_MODE listening and output filtering; `dock_output()` function respects RP_MODE; persistent storage of drone RP settings

**Commands the dock recognizes (v1.3.0+):**
- `REQUEST_HOME_COORDINATES`
- `STATUS` / `DOCK:STATUS`
- `PING` → responds `PONG` / `DOCK:PING` → responds `DOCK:PONG`
- `COORDS` / `DOCK:COORDS` → responds `DOCK:CURRENT_COORDS:`
- `RP_MODE:YES` or `RP_MODE:NO` (v1.5.0+)

**Silent ignore list:** `CHARGE_STATUS`, `BATTERY:`, and any message not prefixed with `DOCK:` or in the legacy list.

**Touch behavior (paired):**
- Uses `llRegionSayTo(llGetOwner(), 0, ...)` to privately tell owner the communication channel
- Only owner sees the channel info

**RP_MODE output control (v1.5.0):**
- `dock_output()` function — when `drone_rp_mode == "NO"`, suppresses operational messages
- Always shows: pairing events, setup, errors regardless of RP_MODE
- Persistent storage of RP_MODE setting in linkset data

### Dock Pairing Script — Key Behaviors

**Version history:**

- **v1.1.0**: Basic pairing, finds Drone_Home prim, sends coordinates
- **v1.2.0**: Added `pairing_completed` flag to prevent re-pairing after success; no self-delete (stays active for testing)
- **v1.4.0 (final working)**: Full rewrite with proper reset. Added admin channel listener on calculated comm channel. Commands: `/<channel> reset` (clears all linkset data), `/<channel> status`.

**Pairing process:**
1. Owner touches dock → enters pairing mode for 60 seconds
2. Dock broadcasts ping to nearby drones on channel 0
3. Drone responds with UUID
4. Unique comm channel calculated from UUID hash
5. Coordinates of Drone_Home prim sent to drone
6. Pairing data saved to linkset data

**Reset mechanism:**
- Script listens on calculated comm channel for admin commands
- `/<channel> reset` — clears all linkset data, resets pairing state
- `/<channel> status` — shows current pairing status
- Touch always enters pairing mode (no blocking after successful pairing in v1.4.0)

**Problem that required v1.4.0:**
- `llLinksetDataRead()` data survives script resets
- Earlier versions checked linkset data on startup and blocked re-pairing
- Fix: Admin listener always set up on paired channel regardless of pairing state

### Battery Pairing (Planned/Designed, Not Yet Implemented in These Chats)

- Battery is a **separate paired item** (as is the console panel)
- Dock does NOT manage battery directly — battery is its own paired component
- Dock should silently ignore `CHARGE_STATUS` and `BATTERY:` messages not directed at it
- Battery swap protocol designed in Power Script (see Section 6)

---

## 5. Pathway Waypoint HUD

### Purpose
A worn HUD for recording drone patrol routes and outputting formatted notecard data.

### Prim Structure
- **Root prim**: `Pathway` — contains all scripts (no scripts in child prims except root)
- **Child prims** (no scripts, controlled via link messages and `llSetLinkPrimitiveParamsFast`):
  - `btn_route` — opens route name dialog
  - `btn_rp` — opens RP mode dialog (Yes/No)
  - `btn_rotation` — opens rotation axis dialog (X/Y/Z)
  - `btn_start` — flags start waypoint
  - `btn_waypoint` — records current position as waypoint
  - `btn_end` — flags end waypoint
  - `btn_resume` — shows beacon + SURL to last waypoint
  - `btn_output` — calculates rotations and outputs formatted text
  - `btn_new` — purges data, resets state
  - `btn_waypnt` — displays hover text with last waypoint XYZ coords (no script; updated via `PRIM_TEXT`)
  - `btn_toggle` — min/max button (two textures: `btn_minimize`, `btn_maximize`)

- **Light prims** (9 total, named `light_[buttonname]`):
  - `light_route`, `light_rp`, `light_rotation`, `light_start`, `light_new`
  - `light_waypoint`, `light_end`, `light_resume`
  - `light_output`

### Button Layout
```
Row 1: btn_route | btn_rp | btn_rotation
Row 2: btn_start | btn_waypoint | btn_end
Row 3: btn_resume | btn_output | btn_new
Separate: btn_waypnt (coordinate display), btn_toggle (min/max)
```

### Button Visibility Logic
- Initially visible: Route, RP, Rotation, Start, New
- After Start clicked: also show Waypoint, End, Resume
- After End clicked: also show Output
- When minimized: all buttons invisible except btn_toggle; root prim alpha = 0

### Light System
- Glow value: **0.25** (orange)
- Initially glowing (when maximized): `light_route`, `light_rp`, `light_rotation`, `light_start`, `light_new`
- After Start: additionally `light_waypoint`, `light_end`, `light_resume`
- After End: additionally `light_output`
- When minimized: all lights invisible (alpha 0.0) and glow off (0.0)

### Storage
- `llLinksetData` — persistent 64KB storage, survives resets
- Keys stored: route name, RP mode, rotation axis, start point, waypoint list, resume point

### Channels
- Dialog channel 1: `-8890`
- Dialog channel 2: `-8889`

### Rotation Calculation
- Uses `llAtan2(next_y - current_y, next_x - current_x)` to calculate heading angle
- Adjusted based on selected rotation axis (Z/X/Y)
- Each waypoint's rotation points toward the next waypoint
- Function name: `calculateAngle()` (renamed from `calculateRotation()` to avoid LSL reserved word conflict)
- Variable: `angleValue` / `angleToNext` / `startAngle` (renamed from `rotation` to avoid LSL reserved type conflict)

### Output Format
```
=====Copy and Paste the following=====
# Route: "Route Name"
# Generated: 2025-09-13 181313.300000
RP_MODE: YES
ROTATION_AXIS: Z
HOME: 116, 107, 1501
START: 116, 107, 1501, -0.047079
WAYPOINTS:
125, 107, 1501, -0.007931
144, 107, 1504, 0.000432
...
END: 126, 103, 1501
```

- Output via `llOwnerSay()` (higher character limit than `llRegionSayTo`)
- Long routes split into chunks of 30 waypoints per message
- Continuation chunks prefixed with `=====Continued=====`
- Header message: Route info + START + "WAYPOINTS:"
- Subsequent chunks: waypoints 1-30, 31-60, 61-82, etc.
- End message: "END:" line

### Resume Feature
- Stores last waypoint as separate XYZ coordinate
- On Resume click: shows beacon to that spot, outputs SURL link via `llRegionSayTo` so owner can TP directly
- Re-enables waypoint recording from that position

### Min/Max Toggle
- `toggleHUD()` function controls:
  - All child prims alpha (0.0 = invisible, 1.0 = visible)
  - Root prim alpha: invisible when minimized, visible when maximized
  - Light prim alpha and glow state
  - `btn_toggle` texture: switches between `btn_minimize` and `btn_maximize` textures

---

## 6. Power System

### Version History
- v2.0.0 — Phase 2 Enhanced Charging System
- v2.1.0 — Added persistent debug toggle, communication channel listening, consistent RP_MODE usage, enhanced charge broadcasting
- v2.2.0/v2.3.0 — Rebuilt from scratch: 24-hour base runtime, 22-hour with movement, full battery hot-swap system, solar panel integration, control panel broadcasting

### Power Constants (Final/Target Values)

| Constant | Value | Notes |
|----------|-------|-------|
| BASE_POWER_DRAIN | 0.069% per minute | 24-hour base runtime |
| MOVEMENT_POWER_PER_METER | 0.00006% per meter | 22-hour runtime during active patrol |
| HEIGHT_ADJUSTMENT_COST | 1.5% | Per terrain change |
| CHARGING_INTERVAL | 36.0 seconds per 1% | 1-hour full charge (36 × 100 = 3600s = 60min) |
| SOLAR_CHARGING_INTERVAL | 18.0 seconds per 1% | 30-minute solar charge |
| TP_TELEPORT_COST | 2.0% | Security TP to avatar |
| AVATAR_SCAN_COST | 0.5% | Scanning an avatar |
| LASER_TARGET_COST | 1.0% | Initial laser targeting (stage 1) |
| LASER_BLAST_COST | 5.0% | Full security blast (stage 2) |

### Power Thresholds

| Threshold | Value | Behavior |
|-----------|-------|----------|
| Emergency Mode | 25% | Speed reduced 50%, power consumption doubled (~6 hours remaining) |
| Critical Return | 5% | Emergency dock return triggered (~1.2 hours remaining) |
| Landing/Charging | 1% | Triggers charging sequence |

### Runtime Estimates
- Idle/standby: 24+ hours
- Normal patrol: ~22 hours (base + movement costs)
- Active security: ~12-16 hours
- Heavy combat: ~8-12 hours
- Emergency mode: ~8-10 hours (doubled consumption)

### Charging Formulas
- Normal charge: 36 seconds per 1% = **60 minutes total**
- Solar charge: 18 seconds per 1% = **30 minutes total**
- Hot-swap: ~30 seconds (instant to 100%, with 3 emote steps at 10s intervals)

### Charging Formula Derivation
```
Base runtime = 24 hours = 1440 minutes
Required drain rate = 100% ÷ 1440 = 0.069% per minute

Movement budget for 22hr runtime:
Total drain at 22hr = 100% ÷ 1320min = 0.0758% per minute
Movement budget = 0.0758% - 0.069% = 0.0068% per minute
At 0.25s waypoint pause (95% moving time, 2.0 m/s, ~114m/min):
Cost per meter = 0.0068% ÷ 114 = ~0.00006%
```

### Battery Hot-Swap Protocol
**Trigger:** Drone docks at 1% power, battery is at 100%

**Sequence (30 seconds total):**
- t=0s: Broadcast power status to dock/battery
- t=10s: `/me disconnecting depleted battery pack`
- t=20s: `/me installing fully charged battery pack [ID]`
- t=30s: `/me battery hot-swap complete - systems fully powered`
- Result: Instant 100% power restoration
- Sends: `HOT_SWAP_COMPLETE:[drone_id]:[battery_id]`

**If battery < 100%:**
- Battery stops charging
- Drone charges normally (36s/1%) or at solar speed (18s/1%) if solar panel connected
- On charge complete: Sends notification to dock/battery to resume battery charging

**If no battery paired:** No change to normal charge times

### Visual Indicators
- Normal charging: Blue eye light
- Solar charging: Yellow eye light
- Hot-swap: Cyan eye light
- Emergency mode: Red indicator
- Lights stay ON when: moving, charging, or hot-swapping
- Lights OFF only when: truly idle (not moving, charging, or swapping)

### Control Panel Broadcasting
**Regular broadcasts (every 5 minutes):**
```
POWER_UPDATE:[drone_id]:[power%]:[runtime_estimate]
```

**Charging start notification:**
```
CHARGING_START:[drone_id]:[type]:[minutes]:[timestamp]
```
- Types: `NORMAL` (60min), `SOLAR` (30min), `HOT_SWAP` (1min)

**Power milestone broadcasts (every 10% drop):**
```
POWER_MILESTONE:[drone_id]:[milestone%]:[runtime_remaining]:[estimated_return_time]
```

**Charging complete:**
```
CHARGING_COMPLETE:[drone_id]:[completion_timestamp]
```

### Security System Power Communication
The power system listens for link messages from security scripts:
- `SECURITY_TELEPORT:[target_id]` → deducts `TP_TELEPORT_COST` (2%)
- `SECURITY_SCAN:[target_id]` → deducts `AVATAR_SCAN_COST` (0.5%)
- `SECURITY_TARGET:[target_id]` → deducts `LASER_TARGET_COST` (1%)
- `SECURITY_BLAST:[target_id]` → deducts `LASER_BLAST_COST` (5%)

Function signature fix: `float handle_security_power_drain(...)` — must declare float return type.

### Persistent Debug Toggle
- `load_debug_state()` and `save_debug_state()` functions
- Debug setting stored in linkset data, survives script restarts
- Commands via paired channel: `debug on`, `debug off`, `recharge`, `status`

---

## 7. Movement System

### Version: v4.1.0 / v4.2.0

### Notecard Format: "Tour"
```
# Route: "Master Patrol"
# Generated: 2025-09-14 9794.600000
RP_MODE: NO
ROTATION_AXIS: Z
HOME: 108, 154, 1501
START: 108, 154, 1501, 1.324121
WAYPOINTS:
109, 158, 1501, 1.160719
...
END: 108, 139, 1501
```

### Key Features
- Reads RP_MODE from notecard and applies it
- Reads ROTATION_AXIS from notecard (X, Y, or Z)
- Pairing-aware: waits for pairing completion before enabling movement
- Automatic notecard change detection
- Manual reload command
- Smart route reloading without losing operational state

### RP_MODE Broadcasting (v4.2.0 addition)
- `broadcast_rp_mode()` function sends `RP_MODE:YES` or `RP_MODE:NO` to dock when notecard loads
- Broadcasts after pairing completes AND when notecard reloads

### Waypoint Pause Timing
- Current (test): 1.0 second per waypoint
- Target (production): **0.25 seconds** per waypoint
- Change location: `llSetTimerEvent(1.0)` → `llSetTimerEvent(0.25)` at waypoint transitions
- Effect: ~95% moving time vs ~50% moving time; much more fluid patrol pattern

### Control Panel Commands (Prep, v4.2.0)
- `CONTROL_PANEL_RETURN` — Emergency return without auto-resume
- `CONTROL_PANEL_STOP` — Land immediately and stop
- `CONTROL_PANEL_START` — Resume patrol operations

### Communication Channel
- Listens on calculated paired channel for admin commands
- Commands: `debug on/off`, `reload`, `status`
- No more hardcoded channel 1000

---

## 8. RP Mode System

### Defined Values
- `RP_MODE: YES` — Local emotes (channel 0) for status updates; immersive flavor text
- `RP_MODE: NO` — Suppress all informational/operational messages entirely; clean output

### How RP_MODE Flows Through System
1. Notecard "Tour" contains `RP_MODE: YES` or `RP_MODE: NO`
2. Movement Script reads it on notecard load
3. Movement Script broadcasts `RP_MODE:YES` or `RP_MODE:NO` on paired comm channel
4. Dock Config Script receives broadcast, stores in `drone_rp_mode` variable and linkset data
5. All subsequent dock output filtered through `dock_output()` function
6. Drone scripts use `movement_output()`, `power_output()`, `config_output()` functions

### Output Functions Pattern
Each script has a centralized output function:
- `dock_output(message)` — Dock Config Script
- `movement_output(message)` — Movement Script
- `power_output(message)` — Power Script
- `config_output(message)` — Configuration Script

These functions check RP_MODE and either:
- Suppress the message (RP_MODE NO)
- Output as normal system message (RP_MODE NO for critical, or always for critical)
- Output as `/me` emote (RP_MODE YES)

**Critical messages always shown regardless of RP_MODE:** Pairing events, setup messages, errors

### RP_MODE Emote Examples
- `/me starting patrol`
- `/me emergency power mode activated`
- `/me charging initiated`
- `/me disconnecting depleted battery pack`
- `/me installing fully charged battery pack [ID]`
- `/me battery hot-swap complete - systems fully powered`

---

## 9. Link Message Strings and Channels

### Communication Channel
- Unique per drone/dock pair, calculated from UUID hash
- Example channel seen in logs: `-233798746`
- Used for: admin commands, RP_MODE broadcasts, charge status, power updates

### Channel 0
- Public chat (local)
- Used for RP emotes when RP_MODE is YES
- Used by HUD dialogs and output

### Hardcoded Channels
- HUD dialog channel 1: `-8890`
- HUD dialog channel 2: `-8889`
- Pairing channel: `0` (public, range-limited)
- Previous (deprecated) admin channel: `1001` / `/1001` — replaced by calculated paired channel

### CHARGE_STATUS Message Format
```
CHARGE_STATUS:[drone_uuid]:[current_power]:[max_power]
```
Example: `CHARGE_STATUS:d4f00f32-30f9-7c82-1cc1-298cc7662b0c:14:13`

The dock silently ignores these messages (does not respond, does not spam).

### Power Broadcast Formats
```
POWER_UPDATE:[drone_id]:[power%]:[runtime_estimate]
CHARGING_START:[drone_id]:[type]:[minutes]:[timestamp]
POWER_MILESTONE:[drone_id]:[milestone%]:[runtime_remaining]:[estimated_return_time]
CHARGING_COMPLETE:[drone_id]:[completion_timestamp]
HOT_SWAP_COMPLETE:[drone_id]:[battery_id]
```

### Battery Communication Formats
```
BATTERY_STATUS:[battery_id]:[charge_level]
BATTERY_SWAP_REQUEST:[drone_id]
SOLAR_AVAILABLE:YES
SOLAR_AVAILABLE:NO
```

### Security System Formats (Inbound to Power Script)
```
SECURITY_TELEPORT:[target_id]
SECURITY_SCAN:[target_id]
SECURITY_TARGET:[target_id]
SECURITY_BLAST:[target_id]
```

### RP_MODE Broadcast Format
```
RP_MODE:YES
RP_MODE:NO
```

### Dock Admin Commands (on paired channel)
```
/<channel> reset     — Clears all pairing data
/<channel> status    — Shows pairing status
/<channel> debug on  — Enables debug output
/<channel> debug off — Disables debug output
/<channel> reload    — Reload notecard
/<channel> recharge  — Trigger recharge
```

### CONTROL_PANEL Formats
```
CONTROL_PANEL_RETURN
CONTROL_PANEL_STOP
CONTROL_PANEL_START
LANDING_1_PERCENT    — (older, from battery swap concept)
```

---

## 10. Bugs and Resolution Status

### Bug 1: Dock Spamming CHARGE_STATUS Unknown Message
**Symptom:** Dock constantly outputs:
```
DOCK CONFIG: Received message on channel -233798746: CHARGE_STATUS:d4f00f32-...:14:12
DOCK CONFIG: Unknown message: CHARGE_STATUS:d4f00f32-...:14:12
DOCK CONFIG: Available commands: REQUEST_HOME_COORDINATES, STATUS, PING, COORDS
```
**Cause:** Dock script had no handler for CHARGE_STATUS messages; treated them as unknown.
**Resolution:** RESOLVED in v1.3.0. Added `is_dock_message()` filtering — all non-dock messages are silently ignored (no output at all).

### Bug 2: First Touch After Pairing Stops (Non-Existent) Patrol
**Symptom:** After pairing completes, first touch on drone gives "CONFIG: Stopping patrol mode" even though patrol was never active. Second touch then starts patrol correctly.
**Cause:** After pairing, `patrol_active` was not explicitly set to `FALSE`. State confusion caused first touch to treat patrol as active.
**Resolution:** RESOLVED in Configuration Script v3.0.1. Added `patrol_active = FALSE;` explicitly in the pairing completion logic.

### Bug 3: Dock Pairing Script Ignores Reset Commands
**Symptom:** After using `/-233798746 reset`, only the drone reset. The dock pairing script said "DOCK: Pairing already completed. Reset dock to re-pair with different drone."
**Cause:** Earlier versions (v1.2.0) stored `pairing_completed` flag and checked it but had no admin listener on the paired channel to receive reset commands.
**Resolution:** RESOLVED in Dock Pairing Script v1.4.0. Always sets up admin listener on paired channel; `/<channel> reset` properly clears all linkset data.

### Bug 4: Linkset Data Survives Script Resets
**Symptom:** After resetting the dock pairing script, it still reads old pairing data and refuses to re-pair.
**Cause:** `llLinksetDataRead()` data persists through script resets; only `llLinksetDataDelete()` removes it.
**Resolution:** RESOLVED in v1.4.0. Reset function explicitly calls `llLinksetDataDelete()` to clear all stored data.

### Bug 5: HUD Waypoint Output Shows Zeros (x,y,z all 0,0,0)
**Symptom:** Route output showed:
```
117, 0, 0, 3.141593
108, 0, 0, 0.000000
```
**Cause:** Waypoints were stored as CSV strings, then parsed back using `llList2Float()` which doesn't work correctly when list contains strings.
**Resolution:** RESOLVED. Changed to store actual vectors in `waypointPositions` list. Uses `(float)llList2String()` for parsing, or direct `llList2Vector()`. Final fix: store vectors directly, flatten to CSV only for persistent storage, rebuild with `llList2Vector()` on load.

### Bug 6: HUD Output Truncated at ~34 Waypoints
**Symptom:** Route output cut off mid-waypoint at `88, 99` (missing Z and rotation) for an 82-waypoint route.
**Cause:** `llRegionSayTo()` has a character limit per message. Output was one large string that exceeded the limit.
**Resolution:** RESOLVED. Switched to `llOwnerSay()` (higher limit), then when that also truncated, implemented **chunked output**: 30 waypoints per message. First chunk has no header; subsequent chunks prefixed with `=====Continued=====`.

### Bug 7: HUD Syntax Error at Line 6,7 — Before `PRIM_`
**Symptom:** Syntax error before string constant declarations.
**Cause:** String constants declared as global assignments from literal expressions still caused issues in certain LSL compiler versions.
**Resolution:** RESOLVED. All button name strings used as literal strings directly (not as named constants), with all assignments moved into `state_entry()`.

### Bug 8: HUD Syntax Error — `string state = ""`
**Symptom:** Syntax error because `state` is a reserved LSL keyword.
**Resolution:** RESOLVED. Renamed to `stateValue`.

### Bug 9: HUD Syntax Error — `float rotation = llAtan2(...)`
**Symptom:** Syntax error because `rotation` is a reserved LSL data type.
**Resolution:** RESOLVED. Renamed to `rotAngle`, `angleValue`, `angleToNext`.

### Bug 10: HUD Type Mismatch — `llGetTimeOfDay()`
**Symptom:** Type mismatch at `string timeStamp = llGetDate() + " " + llGetTimeOfDay();`
**Cause:** `llGetTimeOfDay()` returns a `float`, not a `string`; cannot concatenate without casting.
**Resolution:** RESOLVED. Changed to `(string)llGetTimeOfDay()`.

### Bug 11: RP_MODE from Notecard Not Being Respected
**Symptom:** Both drones (mrk II and mrk III) spamming operational messages even when notecard has `RP_MODE: NO`.
**Cause:** Movement script read RP_MODE from notecard but did not broadcast to dock, and did not have a centralized `movement_output()` filtering function consistently applied.
**Resolution:** RESOLVED in Movement Script v4.2.0 and Dock Config Script v1.5.0. Movement script now has `broadcast_rp_mode()` function. Both scripts use centralized output functions that respect RP_MODE.

### Bug 12: Power Script Return Type Error (Row 875)
**Symptom:** `Return Statement type doesn't match function return type` at `return current_power;`
**Cause:** `handle_security_power_drain` function missing explicit `float` return type declaration.
**Resolution:** RESOLVED. Added `float` return type to function declaration.

### Bug 13: Power Script Hovertext/Lights Turn Off Entirely
**Symptom:** After applying enhanced power script, hovertext and lights turn off and never come back on. Battery swap mode also turns off all lights/hovertext and never restores them.
**Cause:** Multiple scattered calls to `set_eye_on()` and `set_eye_off()` conflicting; no centralized control.
**Resolution:** RESOLVED in Power Script v2.x. Centralized all visual control in `update_power_display()` function. Logic: lights stay ON when moving, charging, or hot-swapping; OFF only when truly idle.

### Bug 14: Variable Name Declared Twice — `last_power_milestone`
**Symptom:** `Name previously declared within scope at 82, 34` — `integer last_power_milestone = 100;`
**Cause:** LSL forbidden rule violation — variable declared twice in same scope, or declared globally with a non-literal default value.
**Resolution:** IN PROGRESS at end of chat (v2.3.0 being rebuilt from scratch). Conversation reached length limit before final version confirmed.

---

## 11. Design Decisions

### Dock Communication Architecture
- **Decision:** Dock only responds to messages specifically directed at it (DOCK:-prefixed or explicit legacy commands). Silently ignores everything else.
- **Rationale:** Battery, console panel, and other components communicate on the same channel; dock should not intercept or respond to their traffic.
- **Decision:** Battery is a **separate paired item** — not managed by dock config script.
- **Decision:** Console panel is also a separate paired item.

### Pairing Script: No Self-Delete
- **Decision:** Pairing script does NOT self-delete after successful pairing (for testing purposes).
- **Decision:** Instead uses `pairing_completed` flag to prevent accidental re-pairing.
- **Decision:** Admin listener always active so owner can `/<channel> reset` to re-pair.

### Channel Communication Strategy
- **Decision:** All admin commands moved to the calculated unique paired channel (not hardcoded `/1001`).
- **Rationale:** Multiple drone instances would conflict if all used same hardcoded channel.
- **Decision:** Dock privately tells owner the channel via `llRegionSayTo(llGetOwner(), 0, ...)` on touch.

### Power System Philosophy
- **Decision:** Base runtime 24 hours, 22 hours with normal patrol movement.
- **Decision:** 1-hour charging time (not fast) with battery hot-swap as the "rapid recovery" option.
- **Decision:** Solar panel integration for 30-minute charging.
- **Decision:** Movement cost calculated per-meter (not per-time) to account for variable speed.
- **Decision:** Security actions (TP, scan, laser, blast) have specific power costs to make security use meaningful.
- **Decision:** Two security blast stages: initial laser target (1%) and massive blast (5%).

### HUD Architecture
- **Decision:** Single root prim script handles everything; no scripts in child prims.
- **Decision:** `llLinksetData` for 64KB persistent storage that survives resets.
- **Decision:** `llOwnerSay()` preferred over `llRegionSayTo()` for output due to higher character limit.
- **Decision:** Output chunked at 30 waypoints per message to avoid any character limits.
- **Decision:** Light prims behind buttons provide visual state feedback (orange glow at 0.25 intensity).
- **Decision:** Root prim visible when maximized, invisible when minimized.
- **Decision:** Resume feature stores last waypoint as separate coordinate for beacon/SURL.

### RP_MODE Design
- **Decision:** RP_MODE stored in notecard, read by Movement Script, broadcast to dock on notecard load.
- **Decision:** Every script has a centralized output function (`dock_output()`, `movement_output()`, etc.).
- **Decision:** Critical messages (pairing, errors, setup) always shown regardless of RP_MODE setting.

### Control Panel Architecture (Planned)
- **Decision:** Control panel pairs to dock to get unique communication channel.
- **Decision:** Control panel receives charging countdown data from Power Script broadcasts.
- **Decision:** Power Script broadcasts at charging start, every 5 minutes, every 10% power drop.
- **Decision:** Countdown timer types: NORMAL (60min), SOLAR (30min), HOT_SWAP (1min).
- **Decision:** Timestamps use Unix time for compatibility with existing clock script (supports time zones, DST, military time).
- **Decision:** Last Full Charge timestamp updated when charging completes; Estimated Return updated every 10% power drop.

---

## 12. All Code Blocks

The chat file for this session primarily contains discussion of code changes, version-over-version iterations, and test output analysis. The actual complete script bodies were pasted in truncated form (visible only as the first ~100 characters in the transcript: "// Script Name v1.x.x ...pasted"). The full bodies of the following scripts were referenced as "Code" or "Code ∙ Version N" blocks that were not captured in the exported chat text.

However, all architecture, constants, function names, message formats, and behavioral logic have been fully extracted above. Below are the code fragments that DID appear in full in the chat.

---

### Code Block 1: Dock Quick Fix — CHARGE_STATUS Handler Snippet
**Context:** Proposed quick fix to silence CHARGE_STATUS spam (before full redesign)

```lsl
else if (llSubStringIndex(processed_message, "CHARGE_STATUS:") >= 0)
{
    // Extract charging data
    list charge_parts = llParseString2List(processed_message, [":"], []);
    if (llGetListLength(charge_parts) >= 4)
    {
        string drone_uuid = llList2String(charge_parts, 1);
        integer current_power = (integer)llList2String(charge_parts, 2);
        integer max_power = (integer)llList2String(charge_parts, 3);

        // Optional: Store or display charging status
        // llOwnerSay("DOCK CONFIG: Drone charging: " + (string)current_power + "%");
    }
    // Don't spam "unknown message" for charge status updates
}
```

---

### Code Block 2: Configuration Script — Patrol Active Fix
**Context:** Fix for double-touch issue after pairing (v3.0.1)

```lsl
// After pairing completion, patrol_active should be FALSE
// So first touch after pairing should START patrol, not stop it

// WRONG - assumes patrol is active after pairing
if (patrol_active)
{
    // Stop patrol
}
else
{
    // Start patrol
}

// CORRECT FIX — explicitly set in pairing completion logic:
patrol_active = FALSE;
```

---

### Code Block 3: HUD Output — Chunking Logic (Conceptual)
**Context:** Fixing character limit issue for 82-waypoint routes

```lsl
// Send header first
llRegionSayTo(owner, 0, headerPart);
// Then send waypoints in chunks of ~30 waypoints per message
llRegionSayTo(owner, 0, waypointChunk1);
llRegionSayTo(owner, 0, waypointChunk2);
// etc.
```

---

### Code Block 4: Rotation Calculation — Core Logic
**Context:** Waypoint HUD rotation angle calculation

```lsl
// Function to calculate angle from one waypoint to next
// Uses llAtan2(delta_y, delta_x) for 2D heading
// "rotation" is LSL reserved type — use "rotAngle" or "angleToNext"

float calculateAngle(vector fromPos, vector toPos)
{
    float angleToNext = llAtan2(toPos.y - fromPos.y, toPos.x - fromPos.x);
    return angleToNext;
}
```

---

### Code Block 5: Movement Script — Waypoint Pause Timing Change
**Context:** Speeding up patrol from 1.0s to 0.25s pause at each waypoint

```lsl
// OLD (test value):
llSetTimerEvent(1.0);

// NEW (production value):
llSetTimerEvent(0.25);
```
**Location:** In Movement Script, the line that sets timer at waypoints during patrol.

---

### Code Block 6: Power Script — Key Constants (Final Target Values)
**Context:** Power System v2.3.0 target values

```lsl
// Base runtime: 24 hours
float BASE_POWER_DRAIN = 0.069;        // % per minute

// Movement: target 22hr runtime during patrol
float MOVEMENT_POWER_PER_METER = 0.00006;  // % per meter

// Height changes
float HEIGHT_ADJUSTMENT_COST = 1.5;    // % per terrain change

// Charging
float CHARGING_INTERVAL = 36.0;        // seconds per 1% — 1 hour total
float SOLAR_CHARGING_INTERVAL = 18.0;  // seconds per 1% — 30 minutes total

// Power thresholds
integer EMERGENCY_THRESHOLD = 25;      // % — ~6 hours remaining
integer CRITICAL_THRESHOLD = 5;        // % — ~1.2 hours remaining

// Security costs
float TP_TELEPORT_COST = 2.0;         // % per TP
float AVATAR_SCAN_COST = 0.5;         // % per scan
float LASER_TARGET_COST = 1.0;        // % initial targeting
float LASER_BLAST_COST = 5.0;         // % full blast
```

---

### Code Block 7: Dock Config — RP_MODE Variable Storage
**Context:** Dock Config Script v1.5.0 RP_MODE tracking

```lsl
// Global variables (declared empty)
string drone_rp_mode;       // "YES" or "NO"
integer rp_mode_received;   // FALSE until broadcast received

// In state_entry:
drone_rp_mode = llLinksetDataRead("drone_rp_mode");
if (drone_rp_mode == "") drone_rp_mode = "YES";  // Default to verbose

// In listen handler:
if (llSubStringIndex(message, "RP_MODE:") == 0)
{
    drone_rp_mode = llGetSubString(message, 8, -1);
    llLinksetDataWrite("drone_rp_mode", drone_rp_mode);
    rp_mode_received = TRUE;
}
```

---

### Code Block 8: Battery Hot-Swap Emote Sequence (Power Script Design)
**Context:** Power Script v2.3.0 hot-swap sequence

```lsl
// Hot-swap takes 30 seconds with RP emotes every 10 seconds
// Only when battery is at 100% and drone docks at 1%

// t=10s:
llSay(0, "/me disconnecting depleted battery pack");

// t=20s:
llSay(0, "/me installing fully charged battery pack " + (string)battery_id);

// t=30s:
llSay(0, "/me battery hot-swap complete - systems fully powered");
current_power = 100;
llRegionSay(comm_channel, "HOT_SWAP_COMPLETE:" + (string)llGetKey() + ":" + (string)battery_id);
```

---

### Code Block 9: Power Milestone Broadcast (Every 10% Drop)
**Context:** Control panel integration — power milestone broadcasting

```lsl
// Broadcast when power crosses 10% milestones (100, 90, 80, etc.)
string broadcast_msg = "POWER_MILESTONE:"
    + (string)llGetKey() + ":"
    + (string)milestone_percent + ":"
    + runtime_remaining + ":"
    + estimated_return_time;
llRegionSay(comm_channel, broadcast_msg);
```

---

### Code Block 10: Charging Start Notification Format
**Context:** Control panel countdown timer trigger

```lsl
// Sent when charging begins — control panel uses to start countdown
string charge_msg = "CHARGING_START:"
    + (string)llGetKey() + ":"
    + charge_type + ":"       // "NORMAL", "SOLAR", or "HOT_SWAP"
    + (string)charge_minutes + ":"
    + (string)llGetUnixTime();
llRegionSay(comm_channel, charge_msg);
```

---

### Code Block 11: HUD getLinkByName() Pattern
**Context:** Referenced from existing AO controller code; used in Pathway HUD

```lsl
// Find prim number by name in linkset
integer getLinkByName(string primName)
{
    integer linkCount = llGetNumberOfPrims();
    integer i;
    for (i = 1; i <= linkCount; ++i)
    {
        if (llGetLinkName(i) == primName)
        {
            return i;
        }
    }
    return -1;
}
```

---

### Code Block 12: HUD toggleHUD() Function (Final Version with Root Prim + Lights)
**Context:** Pathway HUD minimize/maximize with root prim and light prim control

```lsl
// toggleHUD() — controls all prim visibility and light states
// alphaVal = 0.0 (minimized) or 1.0 (maximized)

toggleHUD()
{
    float alphaVal;
    if (hudMinimized)
    {
        alphaVal = 1.0;
        hudMinimized = FALSE;
        llSetLinkPrimitiveParamsFast(getLinkByName("btn_toggle"),
            [PRIM_TEXTURE, ALL_SIDES, btn_minimize_texture, <1,1,0>, <0,0,0>, 0.0]);
    }
    else
    {
        alphaVal = 0.0;
        hudMinimized = TRUE;
        llSetLinkPrimitiveParamsFast(getLinkByName("btn_toggle"),
            [PRIM_TEXTURE, ALL_SIDES, btn_maximize_texture, <1,1,0>, <0,0,0>, 0.0]);
    }

    // Control root prim visibility
    llSetLinkAlpha(LINK_ROOT, alphaVal, ALL_SIDES);

    // Hide/show all buttons (except btn_toggle)
    // ... (iterate button list, set alpha)

    // Handle lights
    if (hudMinimized)
    {
        // All lights off
        // set alpha 0.0, glow 0.0 on all light_ prims
    }
    else
    {
        // Restore lights based on current recording state
        updateButtons();
    }
}
```

---

### Code Block 13: Confirmed Working HUD Sample Output (Master Patrol - Chunked)
**Context:** Verified working output from the HUD after all fixes applied, 82-waypoint route

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
134, 212, 1501, 0.205665
155, 216, 1501, -0.026304
196, 215, 1501, -0.548191
207, 208, 1501, -1.050767
213, 198, 1501, -1.490732
214, 186, 1501, -2.314907
207, 179, 1501, -2.055863
204, 173, 1501, -1.325173
205, 167, 1501, -0.931972
211, 159, 1501, -1.241156
213, 153, 1501, -1.571245
213, 111, 1501, -1.572087
213, 102, 1501, -1.452964
214, 92, 1501, -1.662965
212, 68, 1501, -1.608220
211, 41, 1501, -2.076768
207, 34, 1501, -2.386295
200, 27, 1501, -3.119003
190, 27, 1501, -3.022968
180, 25, 1501, 3.129821
153, 26, 1501, 2.858182
143, 29, 1501, 2.449690
136, 34, 1501, 2.345404
104, 67, 1501, 2.003455
102, 72, 1501, 1.592332
102, 79, 1501, 3.078545
[18:10] Pathway HUD: 94, 80, 1501, 2.437584
88, 85, 1501, 1.910957
85, 93, 1501, 1.105052
88, 99, 1501, 1.464742
89, 110, 1501, 3.133508
85, 110, 1501, -3.114855
77, 109, 1503, 3.107529
72, 110, 1503, 3.132341
63, 110, 1501, 3.049677
56, 110, 1501, 2.434514
51, 115, 1501, 2.038174
46, 124, 1501, 1.745896
45, 132, 1501, 1.515623
46, 151, 1501, 1.106585
51, 160, 1501, 0.803542
67, 178, 1501, 1.078291
72, 186, 1501, 2.956942
70, 187, 1501, -2.129747
64, 176, 1501, -2.349341
51, 163, 1501, -2.127301
45, 154, 1501, -1.840956
43, 147, 1501, -1.582871
43, 134, 1501, -2.710983
34, 130, 1501, -2.848706
25, 127, 1501, 0.302197
34, 130, 1501, 0.480853
43, 134, 1501, 0.320045
54, 138, 1501, 0.106493
63, 139, 1501, -0.015560
69, 139, 1503, -1.582670
[18:10] Pathway HUD: 69, 132, 1503, -3.073587
66, 132, 1503, -0.037487
69, 132, 1503, 1.550749
69, 139, 1503, -3.102163
63, 139, 1501, -3.123250
56, 139, 1501, -2.807532
43, 134, 1501, -1.458152
44, 124, 1501, -1.079528
49, 115, 1501, -0.732405
56, 109, 1501, -2.539031
51, 105, 1501, -2.146412
47, 99, 1501, -0.303171
49, 98, 1501, 0.975437
54, 105, 1501, 0.444713
60, 107, 1501, 0.009482
64, 107, 1501, 0.027666
71, 108, 1503, -0.001625
77, 108, 1503, 0.000426
85, 108, 1501, -0.035825
115, 107, 1501, 1.624769
115, 120, 1501, 1.915623
112, 127, 1501, 1.901253
[18:10] Pathway HUD: END: 108, 139, 1501
```

---

### Code Block 14: Confirmed Working HUD Sample Output (New Test — Shorter Route)
**Context:** First successful clean output after coordinate parsing fix

```
[12:21] Pathway HUD: =====Copy and Paste the following=====
# Route: "New Test"
# Generated: 2025-09-13 181313.300000
RP_MODE: YES
ROTATION_AXIS: Z
HOME: 116, 107, 1501
START: 116, 107, 1501, -0.047079
WAYPOINTS:
125, 107, 1501, -0.007931
144, 107, 1504, 0.000432
203, 107, 1504, 0.010238
214, 107, 1501, -1.556970
214, 81, 1501, -1.655830
213, 68, 1501, 3.102501
203, 68, 1501, 3.043990
144, 74, 1501, 3.121790
141, 74, 1501, 2.375371
123, 92, 1501, 0.736029
131, 99, 1501, 2.532982
END: 126, 103, 1501
```

---

### Code Block 15: Confirmed Drone Debug Output — Paired System Status
**Context:** Verified drone + dock output after full reset and re-init, shows system version and initialization sequence

```
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: CONFIG: Initiating full system reset...
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: MOVEMENT: System reset received - reinitializing...
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: POWER: System reset received - reinitializing...
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: Security Drone starting...
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: Current version: 4.1.0 - Notecard Change Detection
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: MOVEMENT: Waiting for pairing completion before enabling movement
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: Loading: Tour
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: Power Script v2.0.0 - Phase 2 Enhanced Charging System
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: Hybrid Power System: ONLINE
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: CONFIG: Configuration system v3.0.0 initialized
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: Route loaded! Waypoints: 5
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: MOVEMENT: Rotation axis: : Z
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: MOVEMENT: RP mode: 0
[16:23] G.A.S.T.R.O.P.O.D. mrk IV: MOVEMENT: Notecard loaded - waiting for pairing completion
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK CONFIG: Broadcasting on channel -233798746
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK CONFIG: === PAIRED DOCK STATUS ===
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK CONFIG: Paired drone: d4f00f32-30f9-7c82-1cc1-298cc7662b0c
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK CONFIG: Communication channel: -233798746
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK: Pairing already completed. Reset dock to re-pair with different drone.
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK CONFIG: Drone_Home link: 2
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK CONFIG: Current Drone_Home position: <127.60080, 153.97060, 3800.32000>
[16:23] G.A.S.T.R.O.P.O.D. Dock: DOCK CONFIG: Current Drone_Home rotation: <0.00000, 0.00000, 0.70711, 0.70711>
```

---

## Appendix: Known Pending Items at End of Chat File

1. **Power Script v2.3.0 rebuild** — was being rebuilt from scratch to fix `last_power_milestone` double-declaration (LSL scope violation). Conversation reached length limit before final version confirmed. Version 16 of the script was the final iteration shown.

2. **Battery Pairing Script** — planned as next task ("we will create this pairing option shortly") but not begun in this chat.

3. **Control Panel** — architecture fully designed; will pair to dock for unique channel; countdown timer, last charge timestamp, estimated return time. Not yet implemented.

4. **Movement Script waypoint timing** — confirmed target value of 0.25s per waypoint but not confirmed as implemented in a final version.

5. **Eye Security system integration** — security blast stages (laser target + massive blast) and their power costs designed, but "eye security" communication protocol left for a future session.

6. **Solar Panel pairing** — mentioned as future component; power script prepared to receive `SOLAR_AVAILABLE:YES/NO` broadcasts.

---

*End of bible_3.md*
