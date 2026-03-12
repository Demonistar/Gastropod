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

*Section 1 of 10 — more sections to follow*
