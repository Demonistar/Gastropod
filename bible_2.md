# GASTROPOD Bible 2 — Second 3 Chats Extraction

Source file: `Gasgropod second 3 chats.txt` (3594 lines)
Sessions covered: Snail Drone 10.txt through end of second batch

---

## 1. Script Names and Responsibilities

### Core Drone Scripts (4 total)

| Script | Version at End of Session | Responsibility |
|---|---|---|
| Configuration Script | v3.0.0 | Central coordination hub: dock pairing, notecard loading, channel management, admin commands, RP mode, control panel command dispatch |
| Movement Script | v4.1.0 | Waypoint patrol, emergency return/landing, height adjustment, state persistence/resume, notecard change detection |
| Power Script | v2.0.0 | Power drain tracking, charging cycle, emergency thresholds, hover text display, charge broadcasting |
| Pairing Script | v1.2.0 | Initial dock-to-drone pairing, unique channel establishment, range enforcement |

### Peripheral / Proof-of-Concept Scripts

| Script | Version | Responsibility |
|---|---|---|
| Battery Visual Charging System | v2.2 (approx.) | Standalone battery prop: 10 child prims (named 1–10) showing charge level with bidirectional wave effect; status indicator prims named "charge" |
| Waypoint HUD Controller | v1.1.1 | Records waypoints, outputs Tour notecard data, manages UI visibility and RP/rotation settings |
| Dock Config Script | (dock-side) | Receives coordinate requests from drone; broadcasts current dock position and rotation region-wide |

---

## 2. System Architecture Decisions

### Multi-Script Design
- The drone is split into exactly 4 scripts to keep each script under LSL memory limits and to allow independent development/debugging.
- Inter-script communication is exclusively via `llMessageLinked(LINK_THIS, ...)` (link messages). No script calls another directly.
- The dock is a separate object communicating over a shared calculated channel.

### Channel Architecture
- A unique `comm_channel` is calculated during pairing (derived from drone UUID and dock UUID). This ensures only the matched pair communicates.
- All admin commands were **migrated away from hardcoded channel /1001** to the calculated `comm_channel`. This allows each drone to be controlled independently even when multiple drones are deployed.
- Channel format used in debug output: `channel -381513550` (example of a calculated channel value).

### Persistence
- Pairing data (dock UUID, channel, home position, home rotation) is persisted using `llLinksetData*` so it survives script resets.
- Debug mode flag is also persisted via linkset data (added in v3.0.0 of Config and v2.0.0 of Power).
- Saved state (waypoint index, terrain level) is written by the Movement Script and read on resume.

### Region-Wide Communication
- Coordinate requests from drone to dock use region-wide `llRegionSay` on the calculated channel (not limited by range).
- This enables the dock to respond even when the drone is far away on its patrol route.

### RP Mode
- `RP_MODE: YES` or `RP_MODE: NO` is read from the Tour notecard.
- When RP mode is active, status messages are delivered as `/me` emotes on local channel 0 instead of `llOwnerSay` debug-style messages.
- "Verbosity" = RP-themed messages (e.g., "Emergency power low, returning to base"). These are separate from debug output.
- Debug output is controlled by a toggle and should never appear in local chat unless explicitly enabled.

### Notecard Change Detection (added v4.1.0 of Movement Script)
- Movement Script listens for `changed()` event with `CHANGED_INVENTORY`.
- Compares current notecard key (`llGetInventoryKey(NOTECARD_NAME)`) with stored key.
- If changed, reloads the Tour notecard, clears old waypoints, preserves position state.
- A manual `reload` command on the paired channel forces an immediate reload.

### Copied Drone Fix
- When a drone is paired for the first time, the Config Script sends `CLEAR_SAVED_STATE` link message to Movement Script.
- Movement Script clears stored waypoint index and terrain data so copies do not resume from the original's position.

### Hover Text
- Updated in Power Script to use `llGetObjectName()` instead of hardcoded "SECURITY DRONE".
- Each drone (e.g., "G.A.S.T.R.O.P.O.D. mrk I", "mrk II", "mrk III", "mrk IV") displays its own name in hover text.

---

## 3. Link Message Strings and Channel Numbers

### Link Messages (LINK_THIS internal communication)

| Message String | Direction | Purpose |
|---|---|---|
| `HOME_POSITION_DATA` | Config → Movement | Sends home position + rotation; **triggers** emergency return movement |
| `UPDATE_HOME_COORDINATES` | Config → Movement | Sends updated home coordinates only; does **NOT** trigger movement (emergency update at 25% power) |
| `START_PATROL` | Config → Movement | Begin patrol from current or saved waypoint |
| `LANDING_COMPLETE` | Movement → All | Drone has fully landed; triggers charging cycle |
| `CHARGING_COMPLETE` | Power → Config | Charging reached 100%; Config auto-resumes patrol |
| `POWER_CRITICAL` | Power → Config | Power at emergency threshold (25%) |
| `EMERGENCY_RETURN` | Power → Movement | Power at critical threshold (5%); triggers emergency TP home |
| `CLEAR_SAVED_STATE` | Config → Movement | Clear persisted waypoint/terrain data (used after pairing) |
| `RELOAD_NOTECARD` | Config → Movement | Force reload of Tour notecard |
| `CONTROL_PANEL_RETURN` | Config → Movement | Remote return command (dock but do not auto-resume patrol) |
| `CONTROL_PANEL_STOP` | Config → Movement | Remote stop: land in place and disable |
| `BATTERY_STATUS` / `BATTERY_AVAILABLE` | External → Power | Battery swap communication prep |
| `DEBUG_ON` / `DEBUG_OFF` | Config → All | Toggle debug output |
| `RP_MODE_STATUS` | Movement → Config | Reports RP_MODE value read from notecard |

### External Communication Channels

| Channel | Purpose |
|---|---|
| Calculated `comm_channel` | All admin commands (debug on/off, reload, return, stop, start); also charge status broadcasts |
| `-381513550` (example) | Calculated comm channel seen in debug output |
| `0` (public local) | RP emotes (`llSay(0, "/me ...")`) when RP_MODE = YES |
| Region-wide on `comm_channel` | Dock responds to `REQUEST_HOME_COORDINATES` with `HOME_COORDINATES:...` |

### Dock Communication Protocol Message Format

Request (drone → dock):
```
REQUEST_HOME_COORDINATES:<drone_UUID>
```

Response (dock → drone, region-wide):
```
HOME_COORDINATES:<x,y,z>|<rotation_quaternion>
```

Example observed:
```
HOME_COORDINATES:<101.10080, 143.86980, 1500.57500>|<0.00000, 0.70711, 0.00000, 0.70711>
```

---

## 4. Power System Values and Formulas

### Power Drain Constants (from Power Script v1.4.0 / v2.0.0)
```lsl
float BASE_POWER_DRAIN = 0.14;          // % per second at rest
float MOVEMENT_POWER_PER_METER = 0.05;  // additional % per meter traveled
float HEIGHT_ADJUSTMENT_COST = 8.0;     // % cost per height adjustment event
float EMERGENCY_THRESHOLD = 25.0;       // % — triggers coordinate request
float CRITICAL_THRESHOLD = 5.0;         // % — triggers emergency return
```

### Power Thresholds and Actions

| Threshold | Action |
|---|---|
| 25% | Emergency mode activates: speed reduced to 50%, power consumption increased to 200%, coordinate request sent to dock region-wide, drone continues patrol |
| 5% | Emergency return triggered: drone TPs to updated dock position, lands |
| 1% | Power is always set to exactly 1% before emergency TP (RP theme: "just enough to return home") |

### Charging System

| Parameter | Testing Mode | Normal Mode |
|---|---|---|
| Full charge time | 900 seconds (15 minutes) | 3600 seconds (1 hour) |
| Rate (1%→100%) | 0.111%/second (originally), corrected to ~0.278%/sec with 2.5x multiplier | 0.0278%/second |
| Seconds per 1% | ~9 seconds (test) | ~36 seconds (normal) |
| Minutes per 10% segment | ~1.5 minutes (test) | ~6 minutes (normal) |

**Proportional Charging Formula:**
```
charge_time_needed = (100 - current_power) / 100 * FULL_CHARGE_TIME
charge_rate = (100 - current_power) / charge_time_needed
```

Examples at normal (1-hour) rate:
- 0% → 100%: 60 minutes
- 25% → 100%: 45 minutes
- 50% → 100%: 30 minutes
- 75% → 100%: 15 minutes
- Any X% → 100%: `(100 - X) / 100 * 60 minutes`

### Charging Cycle States

1. Drone lands (`LANDING_COMPLETE` message sent)
2. Power Script detects landing at ≤5% power; starts charging
3. Visual indicators: hover text shows "CHARGING", eye/hover platform glow **blue**
4. Power increments each timer tick
5. At 100%: `CHARGING_COMPLETE` sent to Config Script
6. Config Script sends `START_PATROL` → drone auto-resumes

### Emergency Power Behavior
- At 25%: Power sets speed to 50%, consumption to 200% — drone continues patrol
- At 5%: Power sends `EMERGENCY_RETURN` — drone TPs home using stored coordinates
- Power is forced to 1% regardless of actual level before TP (intentional design for RP)

### Battery Swap Prep (planned, not yet implemented in session)
- If external battery at 100%: drone charges at fast rate (15 minutes instead of 1 hour)
- Battery and drone communicate on the same calculated `comm_channel`
- Battery level is inverse of drone level during charging

---

## 5. Dock Pairing Protocol Details

### Pairing Script (v1.2.0) Behavior
- Pairing range: **20 meters** (`PAIRING_RANGE = 20.0`)
- Pairing channel: **0** (public channel for initial handshake, `PAIRING_CHANNEL = 0`)
- Owner must touch drone to initiate pairing
- Visual indicator: drone glows yellow during pairing mode
- Drone verifies distance < 20m before completing pair
- On pairing completion:
  - Unique `comm_channel` calculated from drone + dock UUIDs
  - Channel saved to linkset data
  - `CLEAR_SAVED_STATE` sent to reset any copied state
  - Visual indicators reset to normal
- Note: In v1.2.0 (testing version), self-delete of pairing script is disabled for debugging

### Coordinate Request Protocol (at 25% power)
1. Power Script sends `POWER_CRITICAL` link message to Config Script
2. Config Script sends `REQUEST_HOME_COORDINATES:<drone_UUID>` on `comm_channel` region-wide
3. Dock CONFIG script receives message, logs: `*** COORDINATE REQUEST RECEIVED ***`
4. Dock broadcasts: `HOME_COORDINATES:<pos>|<rot>` region-wide on same channel
5. Config Script receives coordinates, calls `update_home_position(pos, rot, TRUE)` (TRUE = emergency, no movement trigger)
6. Config sends `UPDATE_HOME_COORDINATES` link message to Movement Script (coordinates only, no movement)
7. Movement Script updates `emergency_home_position` and `emergency_home_rotation`
8. Config logs: "Emergency coordinates updated - continuing patrol until 5% power"

---

## 6. Movement and Waypoint System Details

### Tour Notecard Format
```
# Route: "Test Patrol with Bridges"
# Generated: 2025-09-04 19:30:00
RP_MODE: YES
ROTATION_AXIS: Z
HOME: 112, 140, 1501
START: 107, 141, 1501, 3.14159
WAYPOINTS:
<waypoint data lines>
```

### Movement Constants (from script headers)
```lsl
string NOTECARD_NAME = "Tour";
float MOVEMENT_SPEED = 2.0;
float HOVER_HEIGHT = 0.25;
float height_change_threshold = 1.0;
```

### Rotation System
- Drone uses **Z-axis only** for rotation (0, 0, Z format)
- Drone_Home prim always aligned to `0, 0, ###` (only Z rotation relevant)
- Compass mapping:
  - `0, 0, 0` = facing East
  - `0, 0, 90` = facing North
  - `0, 0, 180` = facing West
  - `0, 0, 270` = facing South
  - Intermediate angles (NE, NW, SE, SW) supported via trigonometry

### Rotation Conversion Function
The `convert_dock_rotation()` function extracts Z-axis rotation:
```lsl
rotation convert_dock_rotation(rotation dock_rot)
{
    vector euler = llRot2Euler(dock_rot);
    // Use Z-axis rotation directly (dock always uses 0,0,Z format)
    rotation converted = llEuler2Rot(<0.0, 0.0, euler.z>);
    return converted;
}
```

### Emergency Return Sequence (execute_emergency_return)
1. Drone power forced to 1%
2. Approach position calculated: 1 meter **in front** of Drone_Home prim using Z-axis trig:
   ```lsl
   float z_radians = euler.z;  // from emergency_home_rotation
   vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
   vector approach_pos = emergency_home_position + approach_offset;
   ```
3. Drone TPs to approach position, **facing same direction** as dock (matching Z rotation)
4. Over **10 seconds**, drone backs up into dock (keyframe movement)
5. Emergency landing sequence timer fires
6. Final positioning at exact Drone_Home prim coordinates + HOVER_HEIGHT
7. Drone lowers to ground → `llOwnerSay("Landed")`
8. `LANDING_COMPLETE` link message sent → triggers charging

### State Persistence / Resume
- Current waypoint index and terrain level saved via `llLinksetData*`
- On `CHARGING_COMPLETE` → `START_PATROL` → Movement Script reads saved waypoint index
- Debug output observed: `"Resume enabled - will return to WP16"` / `"Resume enabled - will return to WP5"`
- Observed resume sequence:
  ```
  Resume enabled - will return to WP16
  Rising...
  Starting patrol
  TP: Resume successful - starting movement in 3 seconds
  ```

### Notecard Reload (v4.1.0 addition)
- `changed()` event monitors `CHANGED_INVENTORY`
- On change: compares stored notecard key with `llGetInventoryKey(NOTECARD_NAME)`
- If different: clears waypoint list, reloads notecard, preserves operational state
- Manual: `reload` command on paired channel → Config sends `RELOAD_NOTECARD` link message

### Proximity Docking (planned Phase 2, not yet implemented)
- Trigger: power < 25% **AND** distance to dock < 20 meters
- Drone checks distance to stored dock coordinates during patrol
- If both conditions met: initiate controlled docking (not emergency TP)
- If power ≥ 25% OR distance ≥ 20m: continue normal patrol

---

## 7. Bug Descriptions and Resolution Status

### Bug 1: Emergency return triggered at 25% instead of 5%
**Status: FIXED**
- **Cause:** `update_home_position()` in Config Script always sent `HOME_POSITION_DATA` link message. Movement Script treated this as "start emergency return now."
- **Fix:** Added `emergency_update` parameter to `update_home_position()`. When `TRUE`, sends `UPDATE_HOME_COORDINATES` instead of `HOME_POSITION_DATA`. Only `HOME_POSITION_DATA` triggers movement.
- **Fix confirmed by debug output:**
  ```
  CONFIG: Emergency coordinates updated - continuing patrol until 5% power
  ```

### Bug 2: Drone landing at wrong coordinates when dock is rotated
**Status: FIXED (partially)**
- **Cause:** Emergency return logic used hardcoded `<1.0, 0.0, 0.0>` approach vector, which broke at non-standard dock rotations.
- **Fix:** Approach vector now calculated using trigonometry from Z-axis rotation:
  ```lsl
  vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
  ```
- Additional fix: `convert_dock_rotation()` changed from extracting `euler.y` to `euler.z` to match new dock rotation system.
- **Design change:** Dock's Drone_Home prim simplified to always use `0, 0, Z` rotation format only.

### Bug 3: Rotation conversion wrong when dock rotated beyond 90 degrees
**Status: FIXED**
- **Cause:** Old `convert_dock_rotation()` used `euler.y` (Y-axis), which only worked for specific dock orientations.
- **Fix:** Updated to use `euler.z` (Z-axis) since dock Drone_Home prims are now always `0,0,Z` format.

### Bug 4: Autonomous charging cycle not starting
**Status: FIXED**
- **Cause:** Movement Script did not send `LANDING_COMPLETE` after drone landed.
- **Fix:** Added `llMessageLinked(LINK_THIS, 0, "LANDING_COMPLETE", NULL_KEY)` after `llOwnerSay("Landed")`.
- **Full cycle confirmed working:**
  ```
  [10:56] Landed
  [10:56] POWER: Landing detected at 1% power - initiating charging sequence
  [10:56] POWER: Charging initiated - 1% to 100%
  [11:11] POWER: Charging complete - 100% power restored
  [11:11] CONFIG: Charging complete - automatically resuming patrol
  [11:11] Resume enabled - will return to WP16
  [11:11] Rising...
  [11:11] Starting patrol
  [11:11] TP: Resume successful - starting movement in 3 seconds
  ```

### Bug 5: Emergency landing sequence feature removed without permission
**Status: RESOLVED (feature restored)**
- AI erroneously removed the approach/backup/10-second landing sequence when asked to fix positioning.
- Feature was fully restored. The sequence (appear 1m in front, back up over 10 seconds, land) is intentional RP design.
- **Resolution:** User provided original working code; AI was instructed never to remove features without explicit discussion.

### Bug 6: Battery visual charging rate too slow / too fast
**Status: FIXED**
- At normal `0.1`-second timer intervals with `0.111%/sec` rate, actual measured rate was ~40% of target.
- Root cause: LSL timers do not fire reliably at 0.1-second intervals.
- **Fix 1:** Changed timer interval from `0.1` seconds to `1.0` seconds.
- **Fix 2:** Applied 2.5x multiplier to charge rate for 15-minute test mode to compensate for observed lag.
- **Note:** The 2.5x multiplier must be **removed** when switching to production 1-hour mode.

### Bug 7: Battery segments showing green immediately (1-hour mode)
**Status: FIXED**
- Even at 0.028% power, `segment_partial = 0.0028` was enough to create visible glow.
- **Fix:** Added `VISIBILITY_THRESHOLD = 0.05` (5%). Segments only appear when `segment_partial > VISIBILITY_THRESHOLD`.

### Bug 8: Battery cascade effect lighting all segments instead of filling bottom-up
**Status: FIXED**
- Cascade multiplier `(11.0 - i) / 10.0` was backward, giving segment 10 the highest values.
- **Fix:** Simplified cascade logic so primary charging segment gets full alpha/glow, only the next segment gets a light preview when current >50% charged.

### Bug 9: Battery visual "reset" flash between segments
**Status: PARTIALLY FIXED / PBR Issue**
- When transitioning from segment N fully charged to segment N+1 starting, a brief visual reset occurred.
- Investigation found the glass cylinder container with PBR enabled was causing light blending artifacts, not a script bug.
- **Fix:** User disabled PBR on the glass container. Script behavior was already correct.

### Bug 10: Battery status indicator (prims named "charge") syntax error at line 121
**Status: FIXED**
- `update_status_indicators()` function was placed in the wrong location in the script.
- **Fix:** Recreated script from scratch with all functions before the `default` state block.

### Bug 11: Copied drones resume from original drone's patrol position
**Status: FIXED**
- **Cause:** Linkset data from original drone carries over to copies, including saved waypoint index.
- **Fix:** After pairing, Config Script sends `CLEAR_SAVED_STATE` link message. Movement Script clears all persistent state on receipt.

### Bug 12: Double-click required to start patrol after pairing
**Status: ADDRESSED (by design, clarified)**
- First touch initiates/confirms pairing; second touch starts patrol.
- This is intentional (prevents immediate takeoff during pairing).
- Message updated to clearly indicate "Pairing complete - touch to start patrol."

### Bug 13: HUD light prims stay ON when minimized after re-attach
**Status: FIXED**
- `state_entry()` was not forcing visibility state on attachment.
- **Fix:** Added `forceVisibilityUpdate()` called immediately in `state_entry()`. HUD visibility state persisted in linkset data. Default state is minimized (FALSE).

### Bug 14: HUD orange glow flicker when minimizing
**Status: FIXED**
- Glow was being turned off after alpha set to 0, causing brief orange squares.
- **Fix:** Two-pass approach: turn off ALL glows first, then set ALL alphas to 0. Added `llSleep(0.01)` between passes.

### Bug 15: HUD RP mode and rotation axis always defaulting to "YES" and "Z"
**Status: FIXED**
- `loadData()` function had unconditional default assignments at the bottom that overrode loaded values:
  ```lsl
  if (rpMode == "") rpMode = "YES";
  if (rotationAxis == "") rotationAxis = "Z";
  ```
  These ran even when valid data had been loaded.
- **Fix:** Proper conditional logic: only apply defaults if storage read returned empty string.

### Bug 16: Battery charging not working (showing "Cycle stopped at 0%" after 3 minutes)
**Status: FIXED**
- User touched again to stop; the stop event reported `(integer)current_power` which rounded 0.41% to 0.
- **Fix:** Stop message changed to report actual `current_power` float value.

### Bug 17: Type mismatch at Config Script line 508, col 51
**Status: FIXED**
- Code attempted `(integer)id` where `id` is of type `key`.
- **Fix:** Proper two-step conversion: `string minutes_str = (string)id; integer minutes_remaining = (integer)minutes_str;`

---

## 8. Design Decisions Made

### Autonomous Cycle Design
The complete intended autonomous cycle:
```
Patrol → 25% power (coordinate update, continue patrol) → 5% power
→ Emergency TP home → Back into dock → Land
→ LANDING_COMPLETE → Begin charging (1hr or 15min test)
→ 100% power → CHARGING_COMPLETE → Auto-resume patrol
→ [Repeat indefinitely]
```

### Power Levels are Intentional RP
- Drone always has exactly 1% power for emergency return (RP theme: "just enough to return home").
- This bypasses calculation complexity and creates consistent narrative.

### Phase Structure
- **Phase 1 (COMPLETE):** Basic autonomous cycle — patrol → coordinate update → emergency return → charge → auto-resume
- **Phase 2 (PLANNED):** Proximity docking — if power < 25% AND within 20m of dock, initiate controlled docking before hitting 5% emergency threshold

### Emergency Return vs Proximity Docking
| Feature | Emergency Return | Proximity Docking |
|---|---|---|
| Trigger | 5% power (anywhere on patrol) | <25% power AND <20m from dock |
| Method | Teleport (TP) | Controlled movement |
| Post-dock | Charge + auto-resume | Charge + auto-resume |
| Phase | Phase 1 (DONE) | Phase 2 (PLANNED) |

### Script Documentation Standard (established in session)
All scripts must include this header format:
```lsl
// Script Name vX.Y.Z
// [One-line purpose description]
// Enhancements: [list new features added]
// Modifications: [list code changed/moved/adjusted]
// Cleaned: [list features removed]
// Fixed: [list broken features corrected]
// PHASE X: [current development phase]
```
Sections within script marked with `//[SECTION NAME]` comments.

### LSL Forbidden Rules (locked in, must be followed 100%)
1. **No ternary operators** — use `if/else` instead
2. **No foreach loops** — use traditional indexed `for` loops
3. **No global assignments from function calls** — only literals or empty values at global scope; assignments inside `state_entry` or events
4. **No `void` keyword** — LSL does not use C-style `void`
5. **No `PRIM_HOVER_HEIGHT` in `llSetPrimitiveParams`** — use `llSetHoverHeight()` / `llGetHoverHeight()`
6. **All variables declared with type** — `integer`, `string`, `key`, `list`, `vector`, `rotation`
7. **Functions declared before `default` state block**
8. **No runtime values in global space** — use literals only
9. **No reliance on object name/description for invisible storage** — use `llLinksetData*`

### Battery Visual System Design
- 10 child prims named `1` through `10` each represent 10% of battery
- Only **face 1** of each prim is modified; faces 0 and 2 are left to manual setup
- Two "status indicator" prims named `charge` change color by state:
  - Draining: Orange, glow 0.25
  - Drained (0%): Red, glow 0.15
  - Charging: Blue, glow 0.15
  - Charged (100%): Green, glow 0.25
- Bidirectional wave effect:
  - Charging: fills bottom-up (1→2→3...→10) with cascade anticipation upward
  - Draining: fades top-down (10→9→8...→1) with cascade anticipation downward
- PBR should be **disabled** on the glass container prim to allow discrete segment visibility

### Admin Command Architecture
- All drone commands moved from hardcoded `/1001` to the calculated `comm_channel`
- `comm_channel` is unique per drone-dock pair
- Commands available on paired channel:
  - `debug on` / `debug off`
  - `reload` (force Tour notecard reload)
  - `return` (dock + charge, do not auto-resume)
  - `stop` (land in place, disable)
  - `start` (rise and resume patrol)

### Control Panel Prep
- Config Script added handlers for `return`, `stop`, `start` commands from future control panel
- `return` command: initiates emergency TP home, docks, charges, but **does NOT auto-resume patrol** (sets flag requiring manual touch to restart)
- Distinct from normal charging cycle (`CHARGING_COMPLETE` → auto-resume) vs manual return (charge → wait for touch)

### Charge Status Broadcasting
- Power Script broadcasts charge percentage + estimated minutes remaining once per minute during charging
- Format: sent to Config Script via link message; Config broadcasts on `comm_channel`
- Estimated minutes formula: `minutes = (100 - current_power) / 100 * 60`
- Future: countdown timer object listens on channel and updates display by the minute

---

## 9. Code Blocks (Complete)

### Code Block 1: update_home_position() Fix — Config Script
Version showing the emergency update parameter added to prevent premature return:

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
        // Emergency coordinate update - just update stored data, no movement
        string coord_message = (string)home_position + "|" + (string)home_rotation;
        llMessageLinked(LINK_THIS, 0, "UPDATE_HOME_COORDINATES", coord_message);
    }

    if (debug_mode) llOwnerSay("CONFIG: Home position updated to " + (string)home_position);
}
```

Emergency call site:
```lsl
// In the HOME_COORDINATES handler:
update_home_position(new_pos, new_rot, TRUE);  // TRUE = emergency update, don't trigger movement
```

---

### Code Block 2: UPDATE_HOME_COORDINATES Handler — Movement Script
Handles emergency coordinate update without triggering movement:

```lsl
else if (str == "UPDATE_HOME_COORDINATES")
{
    // Handle emergency coordinate updates without triggering movement
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

### Code Block 3: LANDING_COMPLETE Message — Movement Script
Added after landing is confirmed:

```lsl
// PHASE 1 ADDITION: Send landing complete message for charging system
llOwnerSay("Landed");
llMessageLinked(LINK_THIS, 0, "LANDING_COMPLETE", NULL_KEY);
```

---

### Code Block 4: CHARGING_COMPLETE Handler — Config Script
Phase 1 automatic patrol resumption:

```lsl
else if (str == "CHARGING_COMPLETE")
{
    // PHASE 1: Automatic patrol resumption after charging complete
    llOwnerSay("CONFIG: Charging complete - automatically resuming patrol");
    patrol_active = TRUE;
    llMessageLinked(LINK_THIS, 0, "START_PATROL", NULL_KEY);
}
```

---

### Code Block 5: LANDING_COMPLETE Handler — Power Script
```lsl
else if (str == "LANDING_COMPLETE")
{
    if (current_power <= 10.0) // Start charging if power is very low
    {
        llOwnerSay("POWER: Landing detected at " + (string)((integer)current_power) + "% power - initiating charging sequence");
        start_charging();
    }
}
```

---

### Code Block 6: Rotation Conversion — Movement Script (Z-axis only)
```lsl
rotation convert_dock_rotation(rotation dock_rot)
{
    vector euler = llRot2Euler(dock_rot);
    // Use Z-axis rotation directly (dock always uses 0,0,Z format)
    rotation converted = llEuler2Rot(<0.0, 0.0, euler.z>);
    if (debug_mode) llOwnerSay("DEBUG: Using dock Z-axis rotation: " + (string)converted);
    return converted;
}
```

---

### Code Block 7: Approach Vector Calculation (trig-based for any compass angle)
```lsl
// In execute_emergency_return():
vector euler = llRot2Euler(emergency_home_rotation);
float z_radians = euler.z;
// Calculate approach direction using trigonometry (handles all compass angles)
vector approach_offset = <llCos(z_radians), llSin(z_radians), 0.0>;
vector approach_pos = emergency_home_position + approach_offset;
```

Cardinal direction mapping:
- 0° (East): `<1.0, 0.0, 0.0>` (+X)
- 90° (North): `<0.0, 1.0, 0.0>` (+Y)
- 180° (West): `<-1.0, 0.0, 0.0>` (-X)
- 270° (South): `<0.0, -1.0, 0.0>` (-Y)
- 45° (NE): `<0.707, 0.707, 0>`
- 135° (NW): `<-0.707, 0.707, 0>`
- etc.

---

### Code Block 8: Battery Charging Constants (15-minute test mode)
```lsl
float FULL_CHARGE_TIME = 900.0;   // 15 minutes in seconds (TEST MODE)
// For production 1-hour mode:
// float FULL_CHARGE_TIME = 3600.0;  // 1 hour in seconds
```

Proportional calculation:
```lsl
// charge_rate = power_needed / charge_time
// For full charge: 100 / 3600 = 0.0277778% per second (1-hour mode)
// Timer fires every 1.0 second, adds charge_rate each tick
// After 3600 timer events = 100%
```

Note on 2.5x multiplier for test mode:
```lsl
// For 15-minute test mode only - compensates for LSL timer unreliability:
charge_rate = (power_needed / charge_time) * 2.5;
// This multiplier MUST be removed for 1-hour production mode:
charge_rate = power_needed / charge_time;
```

---

### Code Block 9: Battery Visual System — update_battery_display() Logic

Charging (bottom-up) logic:
```lsl
// At current_power = 15%:
// segment = 1 (floor(15/10) = 1, meaning segment 1 is complete)
// primary_charging = segment 2 (segment + 1)
// segment_partial = (15 - 10) / 10 = 0.5 (50% through segment 2)
// Segment 1: fully lit (alpha=1.0, glow=MAX_GLOW)
// Segment 2: segment_partial alpha and glow (primary charging)
// Segment 3: cascade effect if segment_partial > 0.5
// Segments 4-10: transparent
```

---

### Code Block 10: Battery Status Indicator Colors
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

### Code Block 11: Type Conversion Fix for link_message key→integer
```lsl
// WRONG (type mismatch):
integer minutes_remaining = (integer)id;

// CORRECT (key → string → integer):
string minutes_str = (string)id;
integer minutes_remaining = (integer)minutes_str;
```

---

### Code Block 12: Hover Text Using Object Name — Power Script
```lsl
// CORRECT - uses object name dynamically:
string display_text = llGetObjectName() + "\nPower: " + power_bar + " " + (string)((integer)current_power) + "%" + status_text;

// OLD (hardcoded - do not use):
// string display_text = "SECURITY DRONE\nPower: " + power_bar + ...
```

---

### Code Block 13: HUD Glow-Before-Alpha Fix (two-pass approach)
```lsl
// Minimized state - MUST turn off glow before setting alpha to 0
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

### Code Block 14: HUD loadData() Default Fix
```lsl
// WRONG - overwrites loaded data:
// if (rpMode == "") rpMode = "YES";
// if (rotationAxis == "") rotationAxis = "Z";

// CORRECT - only applies defaults if truly no data found:
string stored_rp = llLinksetDataRead("rp_mode");
if (stored_rp != "")
{
    rpMode = stored_rp;
}
else
{
    rpMode = "YES";  // Default only when no stored value
}

string stored_axis = llLinksetDataRead("rotation_axis");
if (stored_axis != "")
{
    rotationAxis = stored_axis;
}
else
{
    rotationAxis = "Z";  // Default only when no stored value
}
```

---

### Code Block 15: Notecard Change Detection — Movement Script (v4.1.0)
```lsl
// Global variable to track notecard key
key stored_notecard_key;

// In changed() event:
changed(integer change)
{
    if (change & CHANGED_INVENTORY)
    {
        key current_key = llGetInventoryKey(NOTECARD_NAME);
        if (current_key != stored_notecard_key && current_key != NULL_KEY)
        {
            stored_notecard_key = current_key;
            // Reload notecard - clear old waypoints, preserve position
            reload_notecard();
        }
    }
}
```

---

### Code Block 16: CLEAR_SAVED_STATE Handler — Movement Script
```lsl
else if (str == "CLEAR_SAVED_STATE")
{
    // Clear persisted state so copies don't resume from original drone's position
    llLinksetDataDelete("saved_waypoint");
    llLinksetDataDelete("saved_terrain");
    current_waypoint = 0;
    if (debug_mode) llOwnerSay("MOVEMENT: Saved state cleared for fresh start");
}
```

---

### Code Block 17: Battery Cycle — Charge Rate Calculation (production)
```lsl
// Global constants
float FULL_CHARGE_TIME = 3600.0;  // 1 hour in seconds
float VISIBILITY_THRESHOLD = 0.05; // 5% of a segment before showing glow

// Charging start:
float power_needed = 100.0 - current_power;
float charge_time = (power_needed / 100.0) * FULL_CHARGE_TIME;
charge_rate = power_needed / charge_time;
// = 0.0277778 per second for full charge

// Per timer tick (every 1.0 second):
current_power = current_power + charge_rate;
if (current_power >= 100.0)
{
    current_power = 100.0;
    // trigger CHARGING_COMPLETE
}
```

---

## 10. Complete Observed Debug Log Sequences

### Successful Emergency Return + Charge + Auto-Resume (key test)
```
[09:01] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: Received Drone_Home coordinates from dock
[09:01] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: New position: <101.10080, 143.86980, 1500.57500>
[09:01] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: New rotation: <0.00000, 0.70711, 0.00000, 0.70711>
[09:01] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: Home position updated to <101.10080, 143.86980, 1500.57500>
[09:01] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: Emergency coordinates updated - continuing patrol until 5% power
```

```
[10:56] DEATH SNAIL DRONE OF ULTIMATE SECURITY: Landed
[10:56] DEATH SNAIL DRONE OF ULTIMATE SECURITY: POWER: Landing detected at 1% power - initiating charging sequence
[10:56] DEATH SNAIL DRONE OF ULTIMATE SECURITY: POWER: Charging initiated - 1% to 100%
[11:11] DEATH SNAIL DRONE OF ULTIMATE SECURITY: POWER: Charging complete - 100% power restored
[11:11] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: Charging complete - automatically resuming patrol
[11:11] DEATH SNAIL DRONE OF ULTIMATE SECURITY: Resume enabled - will return to WP16
[11:11] DEATH SNAIL DRONE OF ULTIMATE SECURITY: Rising...
[11:11] DEATH SNAIL DRONE OF ULTIMATE SECURITY: Starting patrol
[11:11] DEATH SNAIL DRONE OF ULTIMATE SECURITY: TP: Resume successful - starting movement in 3 seconds
```

### Dock Coordinate Update at 25% (moved dock test)
```
[11:20] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: Received Drone_Home coordinates from dock
[11:20] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: New position: <112.02940, 154.10080, 1500.57500>
[11:20] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: New rotation: <-0.49999, 0.49997, 0.50001, 0.50003>
[11:20] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: Home position updated to <112.02940, 154.10080, 1500.57500>
[11:20] DEATH SNAIL DRONE OF ULTIMATE SECURITY: CONFIG: Emergency coordinates updated - continuing patrol until 5% power
```

### Multiple Drones Charging Simultaneously
```
[02:17] G.A.S.T.R.O.P.O.D. mrk I: POWER: Charging complete - 100% power restored
[02:17] G.A.S.T.R.O.P.O.D. mrk I: CONFIG: Charging complete - automatically resuming patrol
[02:17] G.A.S.T.R.O.P.O.D. mrk I: Resume enabled - will return to WP5
[02:20] G.A.S.T.R.O.P.O.D. mrk III: POWER: Charging complete - 100% power restored
[02:20] G.A.S.T.R.O.P.O.D. mrk III: CONFIG: Charging complete - automatically resuming patrol
[02:20] G.A.S.T.R.O.P.O.D. mrk III: Resume enabled - will return to WP33
```

### Battery Visual Segment Completion Timing (15-min test mode)
```
[02:12] Drone Battery: Charging started from 0.000000% to 100.000000% - Expected time: 15m | Corrected rate: 0.277778/sec
[02:13] Drone Battery: SEGMENT 1 COMPLETE - Power: 10.277770% | Timer: 38
[02:15] Drone Battery: SEGMENT 2 COMPLETE - Power: 20.000010% | Timer: 73
[02:16] Drone Battery: SEGMENT 3 COMPLETE - Power: 30.000040% | Timer: 109
[02:18] Drone Battery: SEGMENT 4 COMPLETE - Power: 40.000070% | Timer: 145
[02:19] Drone Battery: SEGMENT 5 COMPLETE - Power: 50.000100% | Timer: 181
[02:20] Drone Battery: SEGMENT 6 COMPLETE - Power: 60.000130% | Timer: 217
[02:22] Drone Battery: SEGMENT 7 COMPLETE - Power: 70.000160% | Timer: 253
[02:23] Drone Battery: SEGMENT 8 COMPLETE - Power: 80.000190% | Timer: 289
[02:24] Drone Battery: SEGMENT 9 COMPLETE - Power: 90.000220% | Timer: 325
[02:26] Drone Battery: Charging complete - 100% power reached!
[02:26] Drone Battery: Pausing for 5m at 100.000000%
```
Total time: ~14 minutes (close to 15-minute target, slight variance from LSL timer behavior).

---

## 11. Script Version History Summary

| Script | Previous Version | Version in This Session | Key Changes |
|---|---|---|---|
| Configuration Script | v2.1.0 | v3.0.0 | Removed /1001, added debug toggle (persistent), RP mode, control panel commands, charge broadcasting, notecard reload command, CLEAR_SAVED_STATE |
| Movement Script | v3.1.0 | v4.1.0 | Z-axis rotation fix, trig-based approach vector, RP mode, CLEAR_SAVED_STATE handler, CONTROL_PANEL_RETURN/STOP, notecard change detection, auto-reload |
| Power Script | v1.4.0 | v2.0.0 | Object name hover text, RP mode emotes, charge broadcasting, battery communication prep, persistent debug |
| Pairing Script | v1.2.0 | v1.2.0 | No changes needed (kept as-is) |
| Battery System | v1.0 | v2.2 (approx) | Bidirectional wave effect, face 1 only, status indicators (charge prims), 1-hour timing, VISIBILITY_THRESHOLD, stop/start toggle |
| Waypoint HUD | v1.0 | v1.1.1 | Fixed re-attach visibility, glow-before-alpha ordering, RP mode/rotation axis persistence fix |

---

## 12. Roadmap / Planned Features

### Phase 2 (Next)
- **Proximity docking:** When power < 25% AND within 20m of dock → initiate controlled docking
- Requires: distance check during patrol waypoint updates, `PROXIMITY_DOCK` message from Movement to Config

### Future
- **Control panel:** Multi-drone management panel with return/stop/start controls per drone
- **Battery swap mechanism:** Physical battery object exchanges charge with drone
- **Countdown timer display:** Shows estimated charge time remaining, updates per minute via channel broadcast
- **HUD integration with control panel:** Real-time patrol status, battery levels, drone IDs
- **Clock script integration** for control panel time display

---

*End of bible_2.md — extracted from Gasgropod second 3 chats.txt (3594 lines)*
