# MineHub — Mining Fleet Management System
## Master Plan (v0.1 — for review before any code is written)

> **Status:** Draft for discussion. Nothing in here is final. Sections marked **❓ DECISION** need an answer from the mine side before we build that part.

---

## 1. What we are building, in one paragraph

MineHub is a system that tells everyone at the mine, in real time, **what every machine is doing, where it is, and how much material has moved** — and turns that into reliable shift reports and KPIs without anyone retyping numbers into Excel. Operators press a few big buttons on a tablet in the cab. Dispatchers watch a live board and map and reassign trucks. Supervisors, maintenance, planners and managers get accurate reports and dashboards from the same data.

## 2. Why earlier attempts struggle (and what we will do differently)

AI-generated attempts at "desktop + cloud + mobile" apps usually fail for the same few reasons. We design against each one from day one:

| Common failure | What we do instead |
|---|---|
| Three separate apps built separately (desktop, mobile, cloud) that don't share data or logic | **One system, one database, one set of rules.** Different screens for different people, all talking to the same server. |
| Assumes perfect internet | **Offline-first cab app.** Tablet keeps working with no signal and syncs automatically when it reconnects. **Nothing is ever lost.** |
| Assumes the cloud is reachable from the pit | **Server can run on-site at the mine** (works when the internet link to the city is down). The cloud is optional, for HQ reporting. |
| Pretty screens with no real mining logic behind them | Built around a **standard time usage model** (GMG), real delay codes, real load/haul cycles, and tonnage factors from your mine. |
| Tries to build everything at once, nothing works properly | **Small working pieces, one at a time**, each tested at the mine before the next one starts. |
| UI designed for an office, not a vibrating cab at night with gloves | **Cab UI rules:** huge buttons, almost no typing, day/night mode, sound and vibration feedback, works in your language. |
| No history of who changed what | **Every change is logged** (who, when, old value, new value). Reports can be trusted and audited. |

## 3. Who uses it (users and what they need)

| User | Where | Main device | What they need from MineHub |
|---|---|---|---|
| **Truck / excavator / dozer operator** | In the cab | Rugged Android tablet | Log in, pre-start checklist, change status in 1 tap (Working / Delay / Standby / Breakdown), see current assignment, count loads, get messages from dispatch |
| **Dispatcher** | Dispatch room | Desktop PC, 2 screens | Live board of all equipment and status, live map, assign trucks to shovels and dump points, see delays immediately, message operators |
| **Shift supervisor / foreman** | Pit + office | Tablet or phone + PC | Start/end shift, crew assignment, approve/correct shift data, shift report |
| **Maintenance planner / mechanic** | Workshop | PC + phone | Breakdown alerts, equipment hours (SMU), downtime history, fuel |
| **Mine planning engineer** | Office | PC | Load the shift/day plan (targets by pit, block, material), compare plan vs actual, export data |
| **Management** | Office / remote | PC / phone | KPI dashboard: production, availability, utilisation, productivity, trends |
| **Administrator** | Office | PC | Equipment list, users, delay codes, material types, locations, tonnage factors |

## 4. Core mining logic (the "brain" of the system)

This is the part that makes the system correct. These rules are configured by the mine, not hard-coded.

### 4.1 Time usage model
We follow the **GMG (Global Mining Guidelines Group) Time Usage Model**, so KPIs are calculated the same way the industry does. Every second of every machine's shift falls into exactly one category:

```
Calendar time
├── Unscheduled time                    (no shift planned, public holidays)
└── Scheduled time
    ├── Unavailable / Downtime
    │   ├── Unplanned maintenance       (breakdown)
    │   └── Planned maintenance         (service, PM)
    └── Available time
        ├── Standby / Idle              (no operator, waiting on plan, weather, blasting...)
        └── Utilised time
            ├── Operating delays        (waiting at shovel, queuing at dump, refuel, shift change, meal)
            └── Production / Working time
```

From this we compute (consistently, for every machine, shift, day, month):
- **Physical Availability (PA)** = Available time ÷ Scheduled time
- **Use of Availability (UoA)** = Utilised time ÷ Available time
- **Utilisation** = Utilised time ÷ Scheduled time
- **Operating efficiency** = Working time ÷ Utilised time
- **Productivity** = tonnes or BCM per working hour
- **OEE-style effective utilisation** (optional)

**Every status button the operator presses maps to one delay code, and every delay code maps to one category above.** The mine owns this table (❓ DECISION: we need your current delay code list).

### 4.2 Production (load & haul)
- Every load is a record: **truck, loader, operator(s), material type, source (pit/bench/block), destination (dump/stockpile/ROM/crusher), time loaded, time dumped**, payload.
- Payload comes from (in order of preference, configurable): truck payload sensor → bucket count × bucket factor → nominal truck capacity × fill factor.
- Tonnes ↔ BCM conversion uses **density and swell factors per material** set by the planning engineer (coal, overburden by type, etc.).
- **Cycle time** split into: queue at loader → spot → load → haul loaded → queue at dump → dump → return empty. Phase 1 uses operator buttons; Phase 2 detects these automatically from GPS geofences.

### 4.3 Dispatch logic
- **Phase 1 (fixed assignment):** dispatcher assigns trucks to a loader and dump; system shows the queue and warns about problems (loader waiting with no trucks, 4 trucks queued at one shovel).
- **Phase 3 (assisted dispatch):** system *suggests* the next destination per truck to minimise loader idle and truck queue, using **match factor** (= trucks × loading time ÷ (loaders × truck cycle time)) and plan priorities. **The dispatcher always has the final say.**

### 4.4 Plan vs actual
- Planner enters (or imports from Excel/planning software) the shift targets: tonnes/BCM by loader, material, and destination.
- Live progress bars on the dispatch board and in reports: plan vs actual, and the gap.

### 4.5 Data correctness rules
- One machine has one status at any moment (no overlaps, no gaps).
- A shift can't be closed while machines still have open statuses — supervisor resolves them first.
- Corrections after the fact are allowed for supervisors only, and **always logged with a reason**.
- Time is stored in one standard form and shown in mine local time; shift boundaries (e.g. 08:00/20:00) are configurable.

## 5. Modules

| # | Module | Summary |
|---|---|---|
| M1 | **Master data / Admin** | Equipment (type, model, capacity, ID), users & roles, crews, delay codes, materials, locations (pits, benches, dumps, stockpiles), shift patterns |
| M2 | **Cab app (operator)** | Login with PIN/badge, pre-start checklist, status buttons, load counter, assignment display, messages, works offline |
| M3 | **Dispatch board** | Live status of every machine, colour-coded; assignment drag-and-drop; alerts; messaging |
| M4 | **Production tracking** | Loads, tonnes, BCM, cycle times, by machine/operator/material/location |
| M5 | **Shift management** | Shift start/end, crew line-up, handover notes, supervisor review & sign-off |
| M6 | **Reports & KPIs** | Shift report, daily report, monthly report, PA/UA/productivity, Excel/PDF export |
| M7 | **Live map (GPS)** | Machine positions on mine map/drone image, geofences for loaders/dumps, automatic cycle detection |
| M8 | **Maintenance & fuel** | Breakdown notification to workshop, SMU hours, PM due alerts, fuel issues per machine |
| M9 | **Plan integration** | Shift/day targets, plan vs actual, import from Excel or planning software |
| M10 | **Safety & alerts** | Fatigue/long-shift alerts, speed/zone alerts (with GPS), emergency button |

## 6. Technical architecture (plain-language version)

```
 ┌──────────────── In the pit ─────────────────┐        ┌──────── Mine office ────────┐
 │  Cab tablets (Android app, works offline)   │        │  Dispatch room PCs          │
 │  - stores everything locally first          │  Wi-Fi │  Supervisor / planner PCs   │
 │  - sends to server when signal is available │  /LTE  │  (just a web browser)       │
 └──────────────────────┬──────────────────────┘        └──────────────┬──────────────┘
                        │                                                │
                        ▼                                                ▼
              ┌──────────────────────────────────────────────────────────────┐
              │            MineHub SERVER (runs ON-SITE at the mine)          │
              │  - one database (all equipment, statuses, loads, shifts)      │
              │  - all mining rules and KPI calculations                      │
              │  - live updates pushed to dispatch screens instantly          │
              │  - nightly automatic backups                                  │
              └───────────────────────────────┬──────────────────────────────┘
                                              │ (optional, when internet available)
                                              ▼
                              ┌───────────────────────────────┐
                              │  Cloud copy for HQ / phones   │
                              │  (reports & dashboards only)  │
                              └───────────────────────────────┘
```

**Key point: "desktop", "cloud" and "mobile" are not three separate apps.** They are:
1. **One server** (the brain + database).
2. **One web application** for dispatch, supervisors, planners, managers, admin — opens in any browser on any PC. (Can be wrapped as a "desktop app" icon later if wanted; it's the same thing.)
3. **One Android tablet app** for cabs — because it must work offline and use GPS reliably in the background, which a browser cannot do well.

### Proposed technology (for the developer side — you don't need to learn these)
| Part | Choice | Why |
|---|---|---|
| Language everywhere | **TypeScript** | One language for server, web and tablet → shared rules, fewer bugs |
| Server | **Node.js** (Fastify or NestJS) | Mature, fast, handles live updates well |
| Database | **PostgreSQL + PostGIS** | Rock-solid, free, handles maps/GPS natively, easy backups |
| Live updates | **WebSockets** | Dispatch board updates in under a second |
| Web app | **React** | Most widely supported UI toolkit; easy to find developers later |
| Cab tablet app | **React Native (Expo)** + local **SQLite** | Real Android app, offline storage, background GPS, shares code with web |
| Maps | **MapLibre** with your own mine map / drone orthophoto / DXF outlines | Works offline, no licence fees |
| Deployment | **Docker** on one on-site server (+ optional cloud copy) | Install/upgrade with one command; easy to move to cloud later |
| Testing | Automated tests for every KPI formula and sync rule | KPIs must be provably correct |

### 6.1 Reliability design (the "never lose data" rules)
- Every event (status change, load) is created **on the tablet with its own unique ID and timestamp** and saved locally *before* it is sent.
- Sending is retried until the server confirms. Sending the same event twice is harmless (the server recognises duplicates).
- If a tablet is offline for a whole shift, all events sync later in the correct order and the reports fix themselves.
- Tablet clocks are checked against the server to avoid wrong times.
- Server database backed up automatically every night (and copied off-site); restore procedure tested.
- If the server goes down, tablets keep working offline; dispatch falls back to radio until it's back. (Possible later: a second standby server.)
- Health page shows: which tablets are online, last sync time, battery, app version.

### 6.2 Security
- Every user has a role (operator, dispatcher, supervisor, maintenance, planner, manager, admin) that controls what they can see and change.
- Operators log in with a short PIN or badge (fast, glove-friendly); office users with password.
- Encrypted connections; audit log of every change.

## 7. UI / UX design principles

### Cab tablet (most important — if operators don't use it, the whole system fails)
- **6–8 huge buttons max on the main screen**, usable with gloves on a vibrating machine.
- Current state shown in giant colour + text: 🟢 WORKING / 🟡 DELAY: Queue at shovel / 🔴 BREAKDOWN.
- **Status change = 1 tap** (2 taps max for a specific delay reason). No typing during the shift.
- **Day / night mode** (dim dark theme at night so it doesn't blind the operator).
- Sound + vibration confirmation on every press; big "Saved ✓ / Waiting to send ⏳" indicator.
- Language: ❓ DECISION (e.g. Mongolian + English, switchable per user).
- Works on cheap-to-replace rugged Android tablets; screen stays readable in sunlight.
- Operator never needs to "save" or "sync" — it happens automatically.

### Dispatch board (desktop)
- **Exception-first:** problems jump out (red = breakdown, flashing = delay longer than X minutes, loader idle with no trucks).
- One screen = live board grouped by loader (each loader with its trucks), second screen = live map.
- Drag a truck onto a loader to reassign it. Undo is always available.
- Running totals vs plan at the top: tonnes so far, % of shift target, trucks working/down.

### Office / reports
- Shift report generated automatically at shift end, supervisor reviews and signs off.
- Every number is **clickable down to the individual loads/statuses** behind it (so people trust it).
- One-click export to Excel and PDF in your current report layout.

## 8. Phased roadmap

Each phase ends with a **working, tested system used at the mine**, not just a demo. We don't move on until the current phase is reliable.

| Phase | Goal | Contents | "Done" means |
|---|---|---|---|
| **0. Foundation** (1–2 weeks) | Agree on rules & look | Answer decisions in §10, collect delay codes/equipment/report examples, clickable UI mock-ups of cab + dispatch screens, set up the code project | You approve the mock-ups and the time model table |
| **1. MVP — "Digital time & load sheet"** | Replace paper/radio logging | M1 Admin, M2 Cab app (status + manual load count, offline), M3 Dispatch board (status only, manual assignment), M5 Shifts, M6 shift report & basic KPIs | **Pilot: 1 loader + 4–6 trucks for 2 weeks**, reports match manual check within agreed tolerance, zero lost data |
| **2. GPS & live map** | See where everything is, automatic cycles | M7 map with your mine outlines, geofences, automatic load/dump detection, cycle time breakdown, speed | Auto-detected loads match operator counts ≥ 98% |
| **3. Full fleet & smarter dispatch** | Whole mine on the system | All equipment types (dozers, graders, water carts, drills), assisted dispatch suggestions, M9 plan vs actual, M8 maintenance & fuel | Whole fleet live, dispatchers use suggestions |
| **4. Integrations & extras** | Connect to the rest of the business | Cloud HQ dashboards, phone app for managers, payload sensors / OEM telematics, ERP export, planning software import, M10 safety alerts | Agreed per integration |

## 9. Risks and how we handle them

| Risk | Mitigation |
|---|---|
| Poor network coverage in the pit | Offline-first tablets; measure coverage map during pilot; only sync needs network |
| Operators don't adopt it | Extremely simple cab UI, operators involved in design testing, run pilot with supportive crew first |
| Wrong KPIs → people stop trusting it | Use GMG model, automated tests for every formula, drill-down to raw data, parallel run with existing reports during pilot |
| Tablets break / dust / cold | Rugged tablets, spare pool, app re-installs and re-syncs in minutes, no data lives only on one tablet for long |
| Scope creep (trying to do everything) | Strict phases; new ideas go to a list for the next phase |
| Single developer / AI dependency | Clean, documented, standard technology so any developer can maintain it |

## 10. ❓ Decisions & information needed from the mine

**About the operation**
1. Mine type and size: how many **trucks, excavators/loaders, dozers, other**? Models and capacities?
2. Shift pattern (e.g. 2 × 12 h, 08:00–20:00)? Number of crews?
3. Current **delay code list** and how you report shift production today (please share a sample shift report / Excel).
4. Materials tracked (coal grades, overburden types) and their density/swell factors — or where these come from.
5. Destinations: dumps, stockpiles, ROM pad, crusher, wash plant?
6. Which KPIs does management actually look at today?

**About infrastructure**
7. Is there Wi-Fi / LTE / radio network coverage in the pit? How good?
8. Is there internet at the mine site? Reliable?
9. Is there a server room / can the mine provide a PC/server on-site, or must it be cloud-only?
10. Do the trucks already have GPS, payload systems, or OEM telematics (e.g. Komatsu, CAT, Hitachi)?
11. Budget for tablets (rugged Android ~ one per machine)?

**About users**
12. Languages needed on screens?
13. How many dispatchers per shift? Is there a dispatch room?
14. Who will be the "owner" on the mine side to test and approve each phase?

## 11. What happens next

1. You review this plan and answer the questions in §10 (rough answers are fine).
2. I adjust the plan and produce **clickable mock-ups** of the cab screen and dispatch board so you can see and feel them before any real building.
3. After you approve the mock-ups, we build **Phase 1** step by step, and I explain each step in plain language.
