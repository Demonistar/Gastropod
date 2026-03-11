# GASTROPOD Bible 1 — Extracted from "Gasgropod first 3 chats.txt"

---

## 1. LSL FORBIDDEN RULES (Coding Constraints)

These rules must never be violated in any GASTROPOD script:

1. **No ternary operators** — `(id == llGetOwner()) ? "Owner" : "Guest"` is forbidden. Use standard if/else.
2. **No foreach loops** — LSL has no foreach. Always use indexed loops:
   ```lsl
   integer i;
   for (i = 0; i < llGetListLength(myList); ++i) { ... }
   ```
3. **No global assignments from function calls** — `string ownerName = llKey2Name(llGetOwner());` at global scope is forbidden. Only declare globals with literals or empty values. Assign from functions only inside state_entry or other events.
4. **No void keyword** — LSL doesn't use void (C-style). Declare with the return type or nothing for default event blocks.
5. **No direct use of PRIM_HOVER_HEIGHT** — Must use `llSetHoverHeight()` or `llGetHoverHeight()`. `llSetPrimitiveParams([PRIM_HOVER_HEIGHT, 1.0])` is forbidden.
6. **All variable declarations must be scoped inside functions/events unless they are constants** — Globals only for config constants or empty placeholders. Avoid runtime values in global space.
7. **No reliance on object name, description, or notecards for invisible storage unless explicitly required** — Persistent or cross-script data must use `llLinksetData*` or `llGetObjectDesc()` only when explicitly wanted.
8. **No sneaky shorthand** — Always expand logic clearly. No collapsing of logic into unsupported operators, shorthand assignments, or non-standard syntax.

### Additional LSL Scope Rules Discovered During Development

- Global variables can **only** be accessed directly within event handlers (state_entry, timer, touch_start, etc.)
- User-defined functions **cannot** access global variables at all — they can only use parameters passed to them and return values
- Functions are completely isolated from global scope
- Variables declared inside blocks (if/else) are not accessible outside those blocks
- The word `rotation` is a reserved LSL data type — cannot be used as a variable name
- `llSlerp()` does not exist in LSL
- `at_target()` and `not_at_target()` are not valid LSL events
- `llGetTimerEvent()` does not exist in LSL
- `llKeyframedMotion()` does not exist — the correct function is `llSetKeyframedMotion()`
- `KFM_DATA`, `KFM_TRANSLATION`, `KFM_MODE`, `KFM_FORWARD` — these constants DO exist (per later confirmation); the syntax for llSetKeyframedMotion is: `llSetKeyframedMotion([movement_vector, time_duration, rotation, time], [KFM_MODE, KFM_FORWARD])`
- All functions must have explicit return types declared
- All function declarations must come **before** the default state block, after global variables

---

## 2. SYSTEM OVERVIEW

**Project Name:** G.A.S.T.R.O.P.O.D. (Guided Autonomous Security Tactical Response Operations Drone)

**Object:** Full-perm mechanical snail mesh object named "::DisturbeD:: Sci-Fi Robot Attack Snail" / "DEATH SNAIL DRONE OF ULTIMATE SECURITY"

**Purpose:** A roaming security drone for Second Life with:
- Pre-planned waypoint patrol
- Power management (RP-based)
- Security access control (Owner/Admin/Group/Guest/TempGuest/Banned)
- Avatar detection and scanning
- Particle beam effects
- Dynamic dock system for recharging
- Modular multi-script architecture

---

## 3. SCRIPT ARCHITECTURE

### Final Decided Architecture: Nine Scripts in Root Prim

| Script | Responsibility |
|--------|---------------|
| **Movement** | Notecard reading, waypoint management, patrol movement, height adjustments, bridge crossing, state persistence, link message coordination |
| **Power** | All power calculations (hybrid system), hover text display (unicode power bar), emergency mode detection and flashing colors, power state broadcasting |
| **Docking** | Dock pairing data management, emergency return calculations (rotation-aware positioning), dock communication protocol, departure sequences |
| **Scan** | Avatar detection, scanning logic, TP command execution for target investigation, security scanning power consumption |
| **Permissions** | Security lists (Owner/Admin/Group/Guest/Banned), llLinksetData storage for lists, permission validation, chat command handling for list management |
| **Communications** | Panel pairing and command relay, HUD command processing, fleet coordination messages, status reporting |
| **Diagnostics** | Error handling and recovery, performance monitoring, activity/visitor logging, system health checks |
| **Configuration** | Patrol parameters (speed, thresholds, timers), power system tuning, scanning sensitivity settings, behavior mode switching |
| *(9th script TBD)* | Future expansion |

### Child Prim Scripts (separate linked prims)
- **Eye script** — child prim named "Eye": handles glow effects and particle beam targeting
- **Hover script** — child prim named "Hover": manages underglow and teleport burst effects

---

## 4. LINK MESSAGE STRINGS AND CHANNELS

### Movement → Power
- `"MOVEMENT_STARTED"` — drone has started moving
- `"MOVEMENT_STOPPED"` — drone has stopped
- `"MOVEMENT_DISTANCE"` (with float parameter) — distance traveled for power consumption
- `"HEIGHT_ADJUSTMENT"` — height change occurred (for power cost)

### Power → Movement
- `"EMERGENCY_MODE"` — activates 50% speed reduction
- `"NORMAL_MODE"` — restores normal speed
- `"POWER_CRITICAL"` — prepares for emergency return (sent at <2% power)
- `"POWER_RESTORED"` — clears critical state

### Movement → Docking
- `"EMERGENCY_RETURN"` — triggers emergency return to dock

### Docking → Movement
- `"RESUME_PATROL:waypoint"` — resume after charging
- `"RETURN_TO_DOCK"` — dock return trigger

### Movement internal (link to self)
- `"RESUME_TP"` (with saved_waypoint as integer parameter) — triggers resume TP in link_message context
- `"DEBUG_ON"` / `"DEBUG_OFF"` — toggle debug mode

### Hover/Eye prims
- `"HOVER_ON"` / `"HOVER_OFF"` — early system (later replaced by power script control)
- `"HOVER_DEBUG"` messages from child prim debug output

### Dock Communication Protocol
- Drone → Dock: `"REQUEST_DOCK_POSITION"`
- Dock → Drone: `"DOCK_POSITION:<pos>:<rot>"`
- Drone → Dock: `"DRONE_APPROACHING"`
- Drone → Dock: `"DRONE_DEPARTING"`
- Dock → Drone: `"CHARGING_COMPLETE"`

### Dock Pairing Protocol
- Dock broadcasts: `"DOCK_PAIRING:[dock_key]"` on pairing channel
- Drone responds: `"DRONE_PAIR:[drone_key]"`

### Chat Channel
- Default command channel: **1000** (configurable via `channel XXX` command)

---

## 5. POWER SYSTEM VALUES AND FORMULAS

### Capacity Design Goals
| State | Capacity |
|-------|----------|
| Idle (on, not moving) | 24 hours |
| Patrol (normal roaming) | 12 hours |
| Active Scanning | 6 hours |
| Heavy Activity | 4 hours |
| Extreme Activity | 2 hours |

### Drain Rates
| Type | Rate |
|------|------|
| Base idle drain | 4.17% per hour = **0.07% per minute** |
| Patrol drain | 8.33% per hour = **0.14% per minute** |
| Distance movement | **0.05% per meter** traveled |
| Height adjustment | **8 units** per event (testing rate) |
| Avatar scan (owner/admin/group/guest) | **1%** (identification only) |
| Avatar scan (banned user) | **0.5%** (quick detection) |
| Avatar scan (unknown) | **1% + (0.3% × script count)** |
| TP operations | **15%** per sequence (future) |

### Emergency Power Mode (≤25%)
- Hover text flashes red ↔ orange (1-second alternation)
- Movement speed reduced to **50%** of normal
- Power drain rate increased to **200%**
- Status announcements in chat
- Emergency return **triggers at 1-2%** power (not 25%)

### Emergency Mode Colors
- Red: `<1.0, 0.255, 0.212>`
- Orange: `<1.0, 0.522, 0.106>`
- Alternates every 1 second

### Hover Text Colors
- Green (>50%): `<0.18, 0.8, 0.251>`
- Yellow (25-50%): `<1.0, 0.863, 0.0>`
- Red (<25%): `<1.0, 0.255, 0.212>`
- Emergency (flashing): alternates red/orange

### Recharge System
- Full charge (0→100%): **1 hour** = +1.67% per minute
- Proportional: 25% power needs ~45 minutes to full charge
- While charging: drone acts as stationary security (still scans, no patrol)
- Active scanning during charge reduces net recharge rate
- Future: Charging enhancers (solar panels +50%, generators, battery banks for instant swap)

### Power Persistence
- Stored via `llLinksetData` key: `"current_power"`
- Restored on script reset/resume

---

## 6. MOVEMENT SYSTEM

### Approach Abandoned
- Physics-based movement rejected (object flopped, lost orientation, jerked)
- Random roaming rejected (caused scope errors, hard to control)
- `llMoveToTarget()` requires physics (rejected)
- `llSetRegionPos()` used only for height adjustments (rise/fall) — teleports instantly

### Adopted Approach: Pre-Planned Waypoint System
- Owner walks a path with a HUD, recording X/Y coordinates at each stop
- Coordinates stored in a notecard named **"Tour"** inside the drone
- Drone follows waypoints in sequence: 1→2→3→...→1 (loop)
- No collision detection needed (path is pre-walked and confirmed clear)
- `llSetKeyframedMotion()` used for smooth horizontal movement between waypoints

### Movement Parameters
| Parameter | Value |
|-----------|-------|
| Normal patrol speed | 2.0 m/s |
| Hover height (moving) | +0.25m above ground_level |
| Ground level (idle) | 0.0 (stored Z at rez time) |
| Rise time (0→0.25m) | 3 seconds (0.01m increments at 0.1s intervals) |
| Lower time (0.25m→0) | 3 seconds (same rate) |
| Timer interval (patrol) | 1.0 second |
| Timer interval (rising/lowering) | 0.1 second |
| Emergency mode speed | 50% of normal = 1.0 m/s |

### Z Axis Rule
- Drone **never changes its Z coordinate** during patrol
- Z is locked to the ground_level value recorded at rez time
- During movement: Z = `ground_level.z + HOVER_HEIGHT` (0.25m above ground)
- During idle/docked: Z = `ground_level.z` (on ground)

### Height Adjustment (Bridge Crossing)
- Each waypoint stores a terrain Z value
- System compares `working_terrain_level` (current) vs next waypoint terrain
- If difference > `height_change_threshold` (1.0m): rises or lowers before/after crossing
- **Delayed lowering fix (v4.0.4):** Lower command delayed by +1 waypoint to prevent premature descent

### Waypoint Advancement
- When reaching current waypoint (within threshold distance), advances to next
- Wraps: `if (current_waypoint >= total_waypoints) { current_waypoint = 0; }`

### State Persistence
Stored in `llLinksetData`:
- `"patrol_active"` — "0"=paused, "1"=patrolling, ""=fresh
- `"current_waypoint"` — integer index
- `"working_terrain"` — float terrain level
- `"current_power"` — float power level
- `"emergency_mode"` — integer flag
- `"saved_position"` — vector coordinates

### Resume Behavior
- Fresh deployment (no saved state): waits for manual start
- Paused state: loads position, waits for start command
- Active patrol state: auto-resumes from saved waypoint on script reset
- Resume with TP: attempts `llSetRegionPos()` to last waypoint position (link_message context preferred for reliability)

### Reset Command
- Chat: `/1000 reset` or `/1000 fresh` — clears all llLinksetData, forces fresh start

### Emergency Return to Dock
- Triggered at 1-2% power
- Saves patrol state before return
- Requests dock position from paired dock: `"REQUEST_DOCK_POSITION"`
- TPs to approach position (2m in front of dock)
- Backs into dock using keyframed motion
- Enters charging mode
- On charge complete: rise → move 1m forward → TP to saved patrol position → resume

---

## 7. DOCK PAIRING PROTOCOL

### Design Principles
- One pairing script in each (drone and dock) — temporary, deleted after successful pairing
- Scripts persist (NOT deleted) until pairing is SUCCESSFUL
- One dock : one drone (no multi-drone dock)
- Pairing is owner-validated (only pairs matching-owner objects)
- Maximum pairing range: 50m for drone/dock, 100m for panel/dock

### Pairing Process
1. Click dock → dock starts glowing + broadcasts `"DOCK_PAIRING:[dock_key]"` on pairing channel
2. Drone in range detects signal → drone starts glowing
3. Click drone → sends `"DRONE_PAIR:[drone_key]"` response to dock
4. Dock validates ownership, confirms pairing
5. Both exchange UUIDs and generate unique private communication channel
6. Both store pairing data in `llLinksetData`
7. **Only then** do both pairing scripts delete themselves (3-second delay)

### Timeout Behavior
- 1-minute timeout: dock stops glowing BUT keeps pairing script
- Drone stops glowing if not clicked BUT keeps pairing script
- Can retry unlimited times until successful

### Post-Pairing Communication
- Drone stores: paired dock UUID + private channel number (in llLinksetData)
- Dock stores: paired drone UUID
- Private channel derived from pairing exchange (unique per pair)
- Dock position requested dynamically — no hardcoded HOME coordinates needed in notecard

### Docking Alignment (Rotation-Aware)
- Dock rotation analyzed to determine which axis needs offset compensation
- Drone physical offset from dock center: **0.10075m**
- If dock has X-axis rotation (270°): offset on X-axis = -0.10075
- If dock has Y-axis rotation (90°): offset on Y-axis = +0.10075
- Dock rotation matrix applied to transform offset into world coordinates
- Approach position: 2m in front of dock (based on dock's facing direction)

### Departure Sequence (from dock)
1. Rise to hover height (standard 3-second rise)
2. Move 1m forward (away from dock, using dock's orientation)
3. Send `"DRONE_DEPARTING"` to dock
4. TP to saved patrol position (emergency return) or START (manual return)
5. Resume patrol

---

## 8. NOTECARD FORMAT (Route Configuration)

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

**Notes:**
- Angles are in radians (0.0=East, 1.57=North, 3.14=West, 4.71=South)
- HOME line became obsolete with dock pairing system (dock provides position dynamically)
- ROTATION_AXIS: Z means Z-axis controls the drone's facing direction (snail-specific)
- RP_MODE: YES means drone will use home location for recharging (RP feature)

---

## 9. SECURITY SYSTEM

### Security Lists (Stored via llLinksetData)
| List | Storage Key | Notes |
|------|-------------|-------|
| Owner | automatic (`llGetOwner()`) | Not stored, always checked live |
| Admin list | `"admin_list"` | Pipe-delimited UUIDs |
| Guest list | `"guest_list"` | Permanent approved |
| Temp Guest list | `"temp_guest_list"` | Cleared when avatar leaves region |
| Banned list | `"banned_list"` | Permanent ban |
| Group | `"security_group"` | Single group UUID for auto-approval |

### Scanning Logic (Priority Order)
```lsl
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
}
```

### Admin Notification Format
```
Security Alert: Unknown Avatar
Name: [DisplayName]
Profile: secondlife:///app/agent/[UUID]/about
Location: [Region] ([X], [Y], [Z])

[PERMANENT] [TEMPORARY] [BAN]
```

### Warning/Ejection System
- Variable warning time: minimum 10 seconds
- Options: 10s, 20s, 30s, or custom (user-specified, min 10s)
- Admin actions: Permanent approval, Temporary approval (until avatar leaves region), Ban

### Detection Persistence
- Scanned list purged hourly OR when avatar leaves region
- Temp guest list purged when avatar detected leaving region
- Permanent lists persist via llLinksetData

### Future: ID Card System
- Optional attachment for pre-authorized users
- Broadcasts on security channel when entering zone: auto-approves without scan
- Reduces to 1% base power cost for pre-authorized avatars

---

## 10. DESIGN DECISIONS

### Movement Architecture Decision
- Rejected physics-based movement: caused rotation instability, dragging face on ground
- Rejected random roaming: scope errors in LSL, unpredictable behavior
- **Chosen: Pre-planned notecard waypoint system** — owner walks path, drone follows precisely

### Script Architecture Decision
- Initially considered single monolithic script
- Hit LSL memory limit (64KB) → Stack-Heap Collision error
- **Decision: Split into 9 modular scripts** in root prim + child prim scripts for Eye and Hover
- Rationale: Memory management, logical organization, easier debugging
- Trade-off acknowledged: some inter-script communication latency

### Storage Decision
- Rejected notecards for persistent data (too much overhead)
- Rejected prim parameters (visible to builders, limited)
- **Chosen: llLinksetData** for all persistent cross-script data
- 64KB total limit per linkset (shared across all prims — adding more prims does NOT increase storage)
- ~1000+ UUIDs can fit within 64KB limit

### Home Location Decision
- Initially: hardcoded HOME coordinates in notecard
- **Changed: Home location provided dynamically by dock** via paired channel
- Reasoning: Dock can be moved without requiring notecard updates

### Orientation/Facing Decision
- Snail's Z-axis controls its facing direction (not X or Y)
- HUD will have X/Y/Z axis selection buttons to support any object type
- Once axis selected, other buttons fade to 50% alpha

### TP Resume Issue
- `llSetRegionPos()` works in link_message context but fails in timer context (despite returning TRUE)
- Emergency return TP works because it executes in link_message handler
- Resume TP should also be triggered via link message for consistency
- Working pattern: timer sends `llMessageLinked(LINK_THIS, saved_waypoint, "RESUME_TP", NULL_KEY)` → link_message handler executes actual TP

### Power Drain Decision
- Rejected simple timer-only drain (same cost for 10m and 50m travel)
- **Chosen: Hybrid system** = base time drain + distance-based movement cost + activity costs
- Rationale: Realistic power simulation, RP value

---

## 11. BUGS AND RESOLUTION STATUS

| Bug | Status | Resolution |
|-----|--------|-----------|
| Object dragging face on ground (physical) | RESOLVED | Removed physics; use llSetKeyframedMotion instead |
| Rotation going to non-zero X/Y (physics spin) | RESOLVED | Non-physical movement; Z-only rotation with llEuler2Rot |
| Instant teleport to waypoints (llSetRegionPos) | RESOLVED | llSetKeyframedMotion for smooth horizontal movement |
| llSlerp() not in LSL | RESOLVED | Removed; use llEuler2Rot for rotation |
| at_target() event not in LSL | RESOLVED | Removed; use timer-based distance check |
| llKeyframedMotion() wrong name | RESOLVED | Correct: llSetKeyframedMotion() |
| KFM_* constants not defined | RESOLVED | Correct syntax confirmed |
| "rotation" reserved word as variable name | RESOLVED | Renamed to rot_angle or similar |
| Global variable access in functions (scope errors) | RESOLVED | Move all logic to event handlers; functions take params only |
| Physics re-enabling during movement | RESOLVED | Removed all llSetStatus(STATUS_PHYSICS, ...) calls |
| Script forcing drone to WP0 on reset | RESOLVED | State persistence: only auto-resume if patrol_active="1" |
| Drone not resuming from saved waypoint | RESOLVED | load_movement_state() with linkset data |
| Hover text always white | RESOLVED | Use vector color parameter in llSetText() properly |
| Emergency mode not activating | RESOLVED | Added emergency mode check to update_power_display() |
| Rising slower than lowering | PARTIALLY INVESTIGATED | May be related to emergency mode speed affecting height adjustments |
| Power only updating at waypoints (not hybrid) | RESOLVED | Base power drain moved to beginning of timer(), runs continuously |
| 0.5% per minute rate wrong for 24hr target | RESOLVED | Corrected to 0.07%/min (idle) or 0.14%/min (patrol) |
| Resume TP not working (llSetRegionPos in timer) | PARTIALLY RESOLVED | Moved to link_message context; still debugging |
| Emergency return loop after resume TP | PARTIALLY RESOLVED | Added delay before MOVEMENT_STARTED; cause: power state conflict |
| Resume TP shows success but drone doesn't move | OPEN | llSetRegionPos returns TRUE but drone stays at home; timer context issue |
| execute_teleport function scope error | RESOLVED | Renamed unified_teleport, declared before default{} block |
| Bridge descent 1 waypoint early | RESOLVED | "lowering_delayed" flag: lower command delayed by +1 waypoint (v4.0.4) |
| Stack-Heap Collision memory error | RESOLVED | Split into modular multi-script architecture |
| Drone reverting name to "Untitled" | NOT APPLICABLE | Artifact naming issue in Claude's interface only |

---

## 12. ALL CODE BLOCKS

### Code Block 1: Working Notecard Reader / Waypoint Parser (from v2.0.1 dataserver)

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

### Code Block 2: Working Timer with Rise/Lower/Patrol Logic (from v2.0.1 / v4.0.4 working version)

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

### Code Block 3: Z-Only Rotation Calculation

```lsl
// Calculate rotation to face direction of travel
vector direction = llVecNorm(movement);
float z_angle = llAtan2(direction.y, direction.x);
rotation new_rot = llEuler2Rot(<0.0, 0.0, z_angle>);
llSetRot(new_rot);
```

---

### Code Block 4: Touch Start (Stop/Start with Rise/Lower)

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

### Code Block 5: State Persistence (Save/Load)

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

### Code Block 6: Power System - Hybrid Drain (from v2.0.2)

```lsl
// Global power variables (declared at top of Power script)
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
// Base drain check (every 60 seconds)
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

// Distance-based drain (called when MOVEMENT_DISTANCE received):
consume_power(float amount, string reason)
{
    // Note: this is a function pattern — in actual LSL,
    // this logic must be in an event handler
    if (emergency_mode == TRUE)
    {
        amount = amount * 2.0;
    }
    current_power -= amount;
    if (current_power < 0.0)
    {
        current_power = 0.0;
    }
    llOwnerSay("POWER: -" + (string)amount + " units for " + reason + " | Remaining: " + (string)((integer)current_power) + "%");
}
```

---

### Code Block 7: Power Display with Color (from v2.0.1 Fresh)

```lsl
// update_power_display() — must be in an event handler context in actual script
// Power bar display
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
    // Flashing emergency colors
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

llSetText("SECURITY DRONE\nPower: " + power_bar, text_color, 1.0);
```

---

### Code Block 8: Security List Power Consumption Logic

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

### Code Block 9: Docking Alignment Calculation (Rotation-Aware)

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

### Code Block 10: llSetKeyframedMotion Correct Usage

```lsl
// Start keyframed motion to a waypoint
vector movement = target - current_pos;
float time_to_move = distance / MOVEMENT_SPEED;
llSetKeyframedMotion([movement, time_to_move], []);

// Stop keyframed motion
llSetKeyframedMotion([], []);

// Emergency backup into dock
vector backup_movement = home_pos - approach_pos;
float backup_time = 10.0;
list keyframes = [backup_movement, ZERO_ROTATION, backup_time];
llSetKeyframedMotion(keyframes, [KFM_MODE, KFM_FORWARD]);
```

---

### Code Block 11: Dock Pairing Communication Pattern

```lsl
// Pairing channel calculation (using hash of both UUIDs)
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
        // Validate owner matches
        if (llGetOwnerKey(dock_key) == llGetOwner())
        {
            // Store dock reference, start glowing
        }
    }
}

// After successful pairing - store in linkset data:
llLinksetDataWrite("paired_dock_uuid", (string)dock_uuid);
llLinksetDataWrite("dock_channel", (string)dock_channel);

// Requesting dock position at runtime:
llRegionSayTo(paired_dock_uuid, dock_channel, "REQUEST_DOCK_POSITION");
// Dock responds:
llRegionSayTo(paired_drone_uuid, dock_channel, "DOCK_POSITION:" + (string)llGetPos() + ":" + (string)llGetRot());
```

---

### Code Block 12: Emergency Return Sequence (Working Pattern)

```lsl
// In link_message handler (this context makes llSetRegionPos work reliably):
else if (str == "EMERGENCY_RETURN")
{
    // 1. Save patrol state
    llLinksetDataWrite("patrol_active", "0");
    llLinksetDataWrite("current_waypoint", (string)current_waypoint);
    // ... save other state ...

    // 2. Stop current motion
    llSetKeyframedMotion([], []);

    // 3. Request dock position
    llRegionSayTo(paired_dock_uuid, dock_channel, "REQUEST_DOCK_POSITION");
    // (dock responds with DOCK_POSITION message, handled in listen())
}

// When DOCK_POSITION received:
listen(integer channel, string name, key id, string message)
{
    if (llSubStringIndex(message, "DOCK_POSITION:") == 0)
    {
        // Parse position and rotation
        // Calculate approach position (2m in front of dock)
        vector approach_pos = dock_pos + approach_offset;

        // TP to approach position
        llSetKeyframedMotion([], []);
        llSetRegionPos(approach_pos);
        llSetRot(dock_rot);

        // Notify dock
        llRegionSayTo(paired_dock_uuid, dock_channel, "DRONE_APPROACHING");

        // Begin backup sequence into dock
        vector backup_movement = final_dock_pos - approach_pos;
        float backup_time = 10.0;
        llSetKeyframedMotion([backup_movement, ZERO_ROTATION, backup_time], [KFM_MODE, KFM_FORWARD]);

        emergency_landing_sequence = TRUE;
        llSetTimerEvent(backup_time + 0.5);
    }
}
```

---

### Code Block 13: Global Variables Declaration Pattern (from working v2.0.3+)

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

## 13. ROADMAP (From Session 3 Summary)

### Immediate Priority (Current Work)
1. **State Persistence System** — ✅ Largely working (resume TP still being debugged)
2. **Flashing Emergency Colors** — ✅ Working (red/orange alternating)
3. **Emergency Return to Dock** — ✅ Emergency home TP works; resume TP still debugging

### Intermediate Features
4. **Charging System** — Designed, not coded yet
5. **Security Lists & Scanning** — Designed, partial implementation
6. **HUD Integration** — Planned (Stop/Start/Return commands)

### Advanced Features
7. **Teleportation System** — For target investigation (15% power per TP)
8. **Charging Infrastructure** — Solar panels, generators, battery banks
9. **Fleet Management Hub** — Central pillar with 5 dock ports, auto-alignment, battery management

---

## 14. PHYSICAL SETUP NOTES

### Object Details
- Object name: "::DisturbeD:: Sci-Fi Robot Attack Snail" / "DEATH SNAIL DRONE OF ULTIMATE SECURITY"
- Object rotation: 0,0,0 (standard)
- Object is NOT physical (STATUS_PHYSICS = FALSE at all times)

### Dock Details (Test Setup)
- Object name: "Drone Dock"
- Dock rotation tested: `<0, 90, 270>` and `<270, 0, 0>`
- Dock position (one test): `<103.50000, 144.00000, 1500.31970>`
- Drone aligned position: `<103.50000, 143.89925, 1500.56091>`
- Physical Y-axis offset between drone center and dock center: **0.10075m**

### Test Route Coordinates (from "Tour" notecard)
```
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
Total waypoints in use during testing: **50** (expanded from initial test)

---

*End of Bible 1 — Extracted from "Gasgropod first 3 chats.txt" (8439 lines)*
*Date of chat sessions: September 4-8, 2025*
