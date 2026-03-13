# GASTROPOD Master Bible v2.0

> **Full system reference for G.A.S.T.R.O.P.O.D.**
> Produced from complete re-read of all three source chat files.
> Cross-referenced against bible_1.md, bible_2.md, bible_3.md, and GASTROPOD_Master_Bible.md.

---

## Changelog

| Version | Date | Section | Change Summary | Reason |
|---------|------|---------|----------------|--------|
| v1.0 | 2025-09 | All | Initial extraction from bibles 1–3 | First pass |
| v2.0 | 2026-03-12 | All | Full rebuild from source chats | v1 was incomplete — missed battery types, full code blocks, complete function specs, all LLD keys, all channels, infrastructure systems |

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

### Additional Known LSL Constraints Observed in This Project

| Issue | Cause | Correct Approach |
|-------|-------|-----------------|
| Type mismatch when concatenating float to string | `llGetTimeOfDay()` returns `float`, not `string` | `(string)llGetTimeOfDay()` |
| "Name previously declared within scope" | Variable declared twice in same scope | Rename one declaration or move to outer scope |
| Stack-heap collision at 64KB | Monolithic script exceeded LSL memory limit | Split into modular scripts |
| Return type mismatch | Function declared without return type but uses `return value;` | Declare return type explicitly: `float my_function()` |
| LSL timer fires slower than expected at 0.1s intervals | LSL timer unreliable at very short intervals | Use 1-second intervals; adjust increment multipliers |
| `llSetRegionPos()` returns TRUE but doesn't move object | Called in conflict with active KFM timer | Stop timer completely before TP; wait 3 seconds before next movement message |

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
| Phase 1 | ✅ Complete | Autonomous charging cycle: patrol → emergency return → charge → resume |
| Phase 1.5 | ✅ Complete | Multiple drones, RP Mode, HUD output chunking, dock filtering |
| Phase 2 | 🔄 In Progress | Battery system, solar panel integration, Control Panel, hot-swap charging |
| Phase 3 | 📋 Planned | Eye Security integration, laser targeting, blast power costs |

**Phase 1 confirmed working:** September 9, 2025. Drone lands at dock at 1% power, charges 1%→100%, then auto-resumes at last saved waypoint.

**Multiple drones confirmed simultaneous operation:** mrk I (at WP5) and mrk III (at WP33) both charging in the same session.

### 1.7 System Components

| Component | Object Name | Purpose |
|-----------|-------------|---------|
| Drone | `G.A.S.T.R.O.P.O.D. mrk [N]` | The main patrol unit |
| Dock | `G.A.S.T.R.O.P.O.D. Dock` | Charging station, coordinate provider |
| HUD | `Pathway` | Route creation, drone control |
| Control Panel | TBD | Real-time status display, countdown timers |
| Battery | TBD | Optional fast-charge / hot-swap unit |
| Eye Security | TBD | Laser targeting system (Phase 3) |

### 1.8 Communication Overview

| Channel | Type | Purpose |
|---------|------|---------|
| Unique `comm_channel` | `llListen()` | Per-drone channel from UUID hash — all drone↔dock messages |
| `0` | `llSay()` / `llRegionSay()` | Pairing broadcast, RP emotes |
| `-8890`, `-8889` | `llListen()` | HUD dialog channels |

Comm channel calculation (from UUID):
```
integer comm_channel = (integer)("0x" + llGetSubString((string)llGetKey(), 0, 6)) * -1;
```
Example: drone UUID `d4f00f32-...` → channel `-233798746`

---
