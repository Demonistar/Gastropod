# GASTROPOD LSL System Bible — Full Extraction Task
# G.A.S.T.R.O.P.O.D. (Guided Autonomous Security Tactical Response Operations Drone)
# Second Life LSL Scripting Project — Demonistar

---

## Source Files (in this repo)
- `Gasgropod first 3 chats.txt`
- `Gasgropod second 3 chats.txt`
- `Gasgropod last 3 chats.txt`

## Existing Files (already in repo — reference only, do NOT overwrite)
- `bible_1.md` — partial extraction from first 3 chats
- `bible_2.md` — partial extraction from second 3 chats
- `bible_3.md` — partial extraction from last 3 chats
- `GASTROPOD_Master_Bible.md` — incomplete master (use as gap reference only)

## Output Target
Produce: `GASTROPOD_Master_Bible_v2.md`

## CRITICAL — DO NOT START FROM SCRATCH
The following files already exist in this repo and contain completed work.
Use them. Do not ignore them. Do not redo them.
- `bible_1.md`, `bible_2.md`, `bible_3.md` — already extracted content
- `GASTROPOD_Master_Bible.md` — partially complete master bible

Your job is to produce `GASTROPOD_Master_Bible_v2.md` by:
1. Using the existing bibles as your foundation
2. Reading the original chat files to find everything that was MISSED
3. Adding the missing content into the correct sections
4. Never duplicating content that is already correctly documented

---

## EXECUTION RULES — FOLLOW EXACTLY OR THE TASK FAILS

1. Read ONE source chat file at a time. Never load multiple simultaneously.
2. Write ONE section at a time. Fully complete it, commit it, then
   immediately continue to the next section without stopping or waiting.
3. After every commit, log progress internally and keep going automatically.
   Do NOT stop. Do NOT wait for input. Do NOT ask for confirmation.
4. If context feels heavy at any point, commit immediately then continue.
5. Never truncate a code block for any reason. Ever. Full code blocks verbatim always.
6. Per section, read file 1 for relevant content → then file 2 → then file 3.
   Extract content for that section only. Write it. Commit. Move directly
   to the next section automatically.
7. Cross-reference each section against the existing GASTROPOD_Master_Bible.md
   and existing bible_1/2/3.md files to identify gaps and missing content.
   Fill gaps. Do not duplicate what is already correct.
8. Complete all 11 sections and push everything without stopping.
   Only report when ALL sections are fully done and pushed.

---

## EXTRACTION STANDARD

Read ALL THREE chat files completely before writing anything.
Do not summarize as you go. Tag internally every piece of content including:

- Every LSL script — complete, untruncated code blocks, every version
- Every global variable with type, purpose, and scope
- Every custom function: full signature, purpose, all parameters, return value, side effects
- Every LSL state and every event handled per state with trigger conditions
- Every communication channel number and its assignment
- Every llLinksetData key with prefix, format, and purpose
- Every link message number with sender, receiver, and meaning
- Every notecard name and its exact expected format
- Every system mentioned regardless of completion stage:
  solar panels, wind turbines, dark energy batteries, watering systems,
  charging infrastructure, fleet management hub, control panel,
  HUD systems, dock systems, battery visual prop, Eye prim, Hover prim,
  security lists, scanning system, teleportation system, diagnostics,
  visitor tracking — EVERYTHING mentioned, developed or not
- Every bug with its description, root cause if known, and resolution status
- Every failed approach and why it was abandoned
- Every design decision and the reasoning behind it
- Every constraint, limitation, and edge case discussed
- Every roadmap item and future concept regardless of development stage
- Every deprecated approach with the reason it was abandoned
- Every LSL forbidden rule and coding constraint established

Nothing is too minor to document. If it was discussed, it goes in.

---

## REQUIRED OUTPUT: GASTROPOD_Master_Bible_v2.md

### Format
- Markdown format (.md)
- Clean section headers with dividers
- Changelog table at the top
- Technical and direct tone — not academic, no padding
- Stands alone as a complete build reference

### Changelog Table (insert at top before Section 1)
| Version | Date | Section | Change Summary | Reason |
|---------|------|---------|----------------|--------|
| v2.0 | [query system for current date] | All | Full rebuild from source chats | v1 was incomplete |

DATE RULE: Before inserting any date or timestamp, query the system
for the current date/time. Do NOT guess or infer.

---

## REQUIRED SECTIONS (11) — ONE COMMIT PER SECTION

### Section 1 — Project Overview & Purpose
- Full name and acronym breakdown
- Purpose and core capabilities
- Complete component ecosystem table (drone, dock, battery, console, HUD, all variants)
- All drone display names and mark numbers
- Physical setup notes
- Autonomous operating cycle (full loop, all states)
- Development history summary by chat batch

### Section 2 — LSL Forbidden Rules & Coding Constraints
- All 9 forbidden rules with examples of forbidden vs required syntax
- Additional scope rules discovered during development
- Functions and constants confirmed absent from LSL (do not use list)
- Functions and constants confirmed present (verified working list)
- Standard script header format
- Variable naming constraints
- All coding conventions established during development

### Section 3 — Script Architecture
- Every script in the drone root prim with version and full responsibility
- Every child prim script with prim name, script name, version, responsibility
- Every dock object script with version and responsibility
- Every HUD script with version and responsibility
- All peripheral/proof-of-concept scripts
- Inter-script communication architecture (full link message map)
- External communication architecture (drone ↔ dock)
- Prim structure details for drone and dock
- Memory management approach and rationale
- Full version history table for every script across all 3 chat batches
- Why 9-script architecture was chosen (stack-heap collision history)
- llLinksetData storage schema complete

### Section 4 — Power System
- Design philosophy and hybrid model explanation
- All capacity design goals (all operating states)
- All power drain constants with exact values
- Rate breakdown (idle, patrol, distance, height adjustment)
- All avatar scan power costs by access tier
- All power thresholds and actions triggered at each
- Emergency mode full specification (visual, behavioral, power)
- Hover text color reference with exact vector values
- Power bar display format and formula
- Charging system: trigger conditions, proportional rate formula,
  production vs test mode values, 2.5x multiplier note
- Time to full charge table by starting level
- Visual states during charging
- Full charging cycle sequence step by step
- Charge status broadcasting format and frequency
- State persistence: LLD keys used, read/write logic
- Battery visual prop full specification:
  segment count, naming, face assignments, wave effect logic,
  visibility threshold, all status indicator states with colors/glow values
- PBR note
- All 7 planned battery types (Base, Dark Energy, Nuclear, Solar,
  Wall Outlet, Wind, Zero Point) — full specs for any that were discussed
- Solar panel system — everything discussed
- Wind turbine system — everything discussed
- Dark energy system — everything discussed
- Watering/irrigation system — everything discussed
- Any other charging infrastructure discussed

### Section 5 — Movement System
- All rejected movement approaches with reasons
- KFM (llSetKeyframedMotion) approach rationale
- Z-axis lock rule and reason
- Rise on start sequence (exact values, timing)
- Lower on stop sequence (exact values, timing)
- Height adjustment system (bridge crossing, delayed lowering flag)
- Waypoint system (notecard format, parsing logic)
- Patrol speed (normal and emergency)
- Resume TP system — complete specification including:
  the bug history, all failed approaches, working solution
- Emergency return sequence step by step
- Emergency departure sequence step by step
- State persistence keys (all LLD keys used by movement)
- Notecard format (exact format with all fields)
- All movement-related link messages (sent and received)
- RP mode behavior differences

### Section 6 — Dock System
- Dock pairing process complete step by step
- Dock Config Script full specification
- Dock Pairing Script full specification
- Unique channel calculation method
- Dock rotation alignment offsets (all known values)
- Drone_Home prim specification
- Position/rotation broadcasting protocol
- Emergency coordinate request/response protocol
- Channel architecture (all channels used)
- Rotation-aware alignment logic
- All dock-related link messages

### Section 7 — Security System
- Full access hierarchy (all tiers with permissions)
- llLinksetData security list schema
- Scan logic for each avatar tier
- Power costs per tier (already in Section 4 but reference here)
- Admin notification system
- Detection persistence behavior
- TempGuest clearing logic (when/how cleared)
- Banned avatar response sequence
- Unknown avatar response sequence
- Group UUID configuration
- All planned security features not yet built

### Section 8 — All Code Blocks
- EVERY code block from all three chat files
- Label each: CB-[number] | Script Name vX.Y.Z | Source file
- COMPLETE and UNTRUNCATED — no exceptions
- Include partial/broken versions if they were discussed
  (label them [BROKEN] or [DEPRECATED] with reason)

### Section 9 — Bugs & Current Status
- EVERY bug mentioned across all chats
- Format: Bug ID | System | Description | Root Cause | Status
- Status symbols: ✅ Resolved | 🔄 In Progress | ❌ Open | ⚠️ Workaround
- Group by system
- Include all failed approaches tried per bug
- Flag the 4 currently open critical items

### Section 10 — Changelog
- Full version history table for the project
- Every script version change across all chat batches
- All deprecated approaches with reasons
- Format: Version | Date | Component | Change | Reason

### Section 11 — Conclusion & Roadmap
- Phase 1 complete checklist
- Phase 2 full specification
- Phase 3 full specification
- Phase 4 full specification
- Phase 5 full specification
- ALL future features regardless of development stage
- Solar panels, wind turbines, dark energy, watering system,
  fleet hub (5-port dock pillar), control panel broadcast spec,
  charging infrastructure items, visitor tracking,
  diagnostics logging, HTTP server integration — ALL of it
- Production readiness checklist
- Known outstanding issues

---

## COMPLETION CONFIRMATION

When all 11 sections are committed and pushed, report:
- Full file name
- Total line count
- Line count per section
- Confirm all source chat content was covered
- List any content found in chats that did not fit a section
  (append as Section 12 — Miscellaneous if needed)
