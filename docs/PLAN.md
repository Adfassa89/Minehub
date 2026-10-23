# MineHub — Master Plan
### v0.2 — rebuilt from your Gemini conversations ("DESKTOP APP" and "Streamlining Mining Data with Microsoft Tools")

> **Status:** Draft for review. **No code is written until you approve this plan.**
> Items marked **❓ DECISION** need your answer.

---

## Part A — Understanding

### A1. The problem (in your words)
At the mine, every department keeps its own Excel files — **Planning, Dispatch, Geology (quality & seam quantities), Survey (actuals)**. Combining them is slow, error-prone and messy, and subcontractors deliver their data in their own formats. Nobody sees one trusted picture of the day.

### A2. What MineHub is
**One cloud platform where every department of every company at a mine enters its data once, in a controlled way, and everyone gets the same trusted reports.**

- **Multi-company (SaaS):** used by many mining companies, not just one. Each company only sees what it is allowed to see.
- **Owner + subcontractor model:** the mine owner defines the mine (locations, blocks, materials, qualities). Subcontractors use the owner's definitions, bring their own fleet and people, and their data flows to the owner automatically.
- **Any commodity:** coal, gold, copper, other minerals. Nothing is hard-coded for coal or for one mine.
- **"Master data first":** nobody types free text for important things (truck IDs, seams, locations). They pick from lists, so the data is clean.
- **Two kinds of data:**
  - **Post-processed / manual data first:** plans, dispatch shift logs, geology quality, survey actuals, stockpile balances, daily approval.
  - **Real-time data later:** equipment location and status from the **Data Collector App** on handheld devices in the cabs.
- **Reports people can trust:** plan vs dispatch vs survey reconciliation, stockpile balance, ROM quality, KPIs, Excel export, and later a flexible pivot/report builder.

### A3. Everything you asked for (collected from both conversations)
**Platform & access**
- Sign up / sign in with username or email + password, plus password reset by email. Email or phone verification.
- Registration choice: **join an existing company** (6-digit company ID → request goes to that company's admin) **or create a new company**:
  - owner or subcontractor (a subcontractor enters its head company's ID);
  - owner companies enter mine details: name, type (coal / metals / minerals), country, state.
- **Super admin** (you, the platform owner):
  - sees all companies;
  - approves new companies;
  - gives **licences** (start date, end date, remaining days are calculated);
  - can delete users and companies;
  - can switch into any company to help it.
- **Licence check:** no licence, or an expired one, means no access. The message says "No valid licence" or "Licence expired", followed by "contact admin@minehub.mn, +976…".
- **User management:** users list, join requests (approve / reject), and **custom roles per company** (not hard-coded).
  - Permissions are set per tab: view / edit.
  - Permissions can be given **per user**, and a user-level grant overrides the role.
  - Normal users can't see user management or approve themselves.
- **Setup wizard** on a company admin's first login:
  - owner company: locations, materials, qualities, roles and permissions;
  - subcontractor: roles and permissions only.

**Master data ("Project setup")**
- **Fleet:** equipment ID, type, brand, model, capacity (m³ / t), notes. Also **operators**.
- **Locations:**
  - type is **source** (where material is dug) or **destination** (where it is dumped);
  - name, block, elevation;
  - for a destination, whether it is a **stockpile**.
- **Materials:**
  - type, is it a resource (yes/no), unit (m³ / t), default density, default swell factor, notes;
  - **coal:** seam (0–10) and sub-seam (A, B, C, D, U);
  - **other commodities:** category and sub-category.
- **Qualities:** name (ash, CV, sulphur, Au g/t…), unit, notes.
- **Products:** classify resource into products or grades by quality ranges.
- **Custom fields:** name, unit (length / volume / area / mass / custom), notes. Example: haul distance per cycle or per shift.
- **Bulk edit:** **copy/paste straight from Excel**, with save and cancel.
- Microsoft Office **import/export** everywhere.

**Operations modules:** Plan, Dispatch, Geology, Survey, Stockpile management, Reconciliation, Reports.
- Dispatch form: selecting a truck fills its capacity automatically.
- Reconciliation: dispatch total vs survey actual per block/period, with variance %.
- **Stockpile:** day-to-day balance.
- **ROM quality.**

**Daily approval workflow**
- Each day's data is entered in sequence: **planners → dispatchers → geologists → surveyors**.
- Everyone can see the day's checkpoint status.
- After some days, **department chiefs approve** the reports.

**Map**
- Satellite basemap.
- **DWG/DXF CAD import** to show pits, roads and boundaries.
- Draw and highlight areas, with view/edit permission.
- Later: **live equipment positions**.

**Data Collector App (handheld in each machine)**
- Sends **real-time** location and status.
- Sends the **activity log and production log** at the end of the shift.
- Always talks to the system **through the web API**, which keeps it secure and maintainable.
- Must stay flexible, so it can be rebuilt and connected easily.

**Integration**
- Final shift data (total material moved, equipment activity log) can go to the **company's own local SQL Server** if the company wants it.

### A4. Why the previous attempts kept breaking (honest review)

| What happened | Root cause | What we do now |
|---|---|---|
| Restarted 4 times: Windows Forms → .NET MAUI → React → Antigravity | Platform choices changed mid-way; each restart threw away work | **Choose the platform once (Part C), with reasons, and keep it** |
| "The old version worked like a charm, why did you mess up the working code?" | No version history; the AI rewrote whole files and silently dropped features (for example, the user management tab disappeared) | **Git on GitHub:** every change is saved, small and reversible. Nothing is ever "lost" again |
| The same syntax error came back again and again | Copy-pasting giant files from chat into your PC by hand | **I write, run and test the code directly in this repository.** You never copy-paste code |
| Everything in one `App.js` | No structure | Modular project: one folder per module, shared components |
| "SQL and Firebase are out of sync", "where() called with undefined", sign-up requests not appearing | **Two databases holding overlapping data** (users and companies in Firebase, permissions and data in Supabase) | **One database is the single source of truth** (decision C2) |
| Users could approve their own requests | Permission rules only in the screen code, not enforced on the server | **Every rule is enforced on the server and in the database**, and screens only hide or show |
| "I deleted my old tables, give me new SQL" | No managed database changes | **Database migrations:** each change is a versioned script, applied automatically and never destroys data |
| Copilot said "all phases complete" but the app showed "Hello World" | Nobody verified; no tests | **Automated tests + I run the app and check it myself** before saying something is done |
| Features (map, live updates) added before basics were solid | No phase discipline | You already said it best: *"solidify each step, working robust"* — strict phases below |

**Security note:** your Google Docs contain your Supabase project key and Firebase settings. If those docs are shared publicly ("anyone with the link"), please make them private. When we rebuild, we'll create fresh projects and keys anyway, so the old keys will no longer matter.

---

## Part B — Core Design

### B1. Who's who (tenancy model)
```
Platform (MineHub — Super Admin)
└── Company  (type: OWNER or SUBCONTRACTOR, 6-digit unique ID, licence)
    ├── Owner company ──owns──► Mine / Site (name, commodity, country, state, coordinate system)
    │                              ├── Master data owned by the mine: locations, blocks, materials, qualities, products
    │                              └── Contracts: which subcontractor works at this mine (start/end)
    ├── Subcontractor ──works at──► Mine (via contract)
    │      ├── Own: fleet, operators, users, roles
    │      └── Uses (read-only): the mine's locations, materials, qualities
    └── Users ── membership(company, role) ── permissions
```
**Visibility rules (enforced in the database):**
- An owner sees its own data **plus all its subcontractors' data at its mines**.
- A subcontractor sees **only its own data**, never another subcontractor's data.
- The super admin can see everything, but only inside an explicit "support mode" that is logged.

### B2. Permissions model
- Each company creates its own **roles** (e.g. "Senior Geologist", "Shift Dispatcher").
- Each role has, per module: **None / View / Edit / Approve**.
- **User-level overrides** win over the role (as you specified).
- Built-in starter roles are created by the setup wizard. Admins can change them.
- Every action is checked **on the server**. A user can never approve their own request or raise their own rights.

### B3. Data structure (main categories)
**1. Platform:** companies, mines/sites, contracts, users, memberships, roles, permissions, licences, audit log.

**2. Master data (the "nouns"):**

| Table | Key fields |
|---|---|
| Equipment types | truck, excavator, loader, dozer, drill, grader, light vehicle… (configurable) |
| Equipment | code, type, brand, model, capacity + unit, owning company, status |
| Operators | name, employee ID, company, licences/qualified equipment |
| Locations | name, **source/destination**, is stockpile, block, elevation, map shape (point/polygon) |
| Blocks | block ID (the **key that links geology, plan, dispatch and survey**), pit, bench/elevation, seam/zone, planned tonnes/volume, planned qualities |
| Materials | name, class (ore/coal, waste, topsoil…), is resource, unit, **group 1 / group 2** (labels set by the mine: "Seam / Sub-seam" for coal, "Category / Sub-category" for metals), density, swell factor |
| Quality parameters | ash, CV, sulphur, moisture, Au g/t, Cu %… with units |
| Products | name + quality range rules |
| Shift patterns | e.g. Day 08:00–20:00, Night 20:00–08:00 |
| Activity / delay codes | each mapped to a time category (see B5) |
| Custom fields | name, unit type, notes; can be attached to equipment, locations, loads… |

**3. Operational data (the "verbs"), all tied to mine + date + shift + company:**

| Table | Who enters | What |
|---|---|---|
| Plan | Planners | targets per period/shift: block, material, quantity, qualities, destination |
| Production loads | Dispatch / Data Collector App | loader, truck, operator, source/block, destination, material, quantity, time, distance |
| Equipment activity log | Dispatch / App | every status period per machine: start, end, activity code |
| Geology quality | Geologists | sample/analysis results per block/seam/stockpile |
| Survey actuals | Surveyors | measured volume/tonnes per block/period, stockpile surveys |
| Stockpile movements | calculated + adjustments | in / out / survey adjustment → **balance** |
| Telemetry (later) | Data Collector App | GPS position, speed, status heartbeats |

**4. Workflow:** daily checkpoints (per mine/day/department: Not started → Submitted → Approved), approvals, corrections (always with reason, old/new value kept).

### B4. Key business logic
- **Quantity of a load:** the default is truck capacity (auto-filled). It can be adjusted. Later it can come from payload sensors or bucket counts.
- **Unit conversion:** bcm ↔ loose m³ ↔ tonnes, using material density and swell. Values are stored as entered and conversions are calculated, so a factor can be corrected later without changing the original entries.
- **Quality of mined material:** comes from the block the load came from. Dispatchers never type ash or CV.
- **Stockpile balance:** opening + received − reclaimed ± survey adjustment = closing.
  - It's calculated per day and quality is blended by weighted average.
- **Reconciliation:** compares plan vs dispatch vs survey per block and period. Variance and %, with a drill-down to the individual loads.
- **Daily checkpoint:**
  1. The day can only be approved when every required department has submitted.
  2. Approved days are locked.
  3. A change to a locked day needs a correction request with a reason, approved by the chief.
- **Equipment time:** uses the GMG standard time model (B5).

### B5. Equipment time model (industry standard — GMG Time Usage Model)
```
Calendar time
├── Unscheduled
└── Scheduled
    ├── Down: unplanned (breakdown) / planned (service)
    └── Available
        ├── Standby (no operator, weather, blasting, waiting plan…)
        └── Utilised
            ├── Operating delay (queue, refuel, shift change, meal…)
            └── Working / Production
```
- **Physical availability** = Available ÷ Scheduled.
- **Utilisation** = Utilised ÷ Available.
- **Productivity** = quantity ÷ working hour.

Every activity code a company creates must be put into one of these boxes, so KPIs are always comparable between companies.

---

## Part C — Technology

### C1. Shape of the system
```
                  ┌──────────────────────── CLOUD ────────────────────────┐
 Office PCs  ───► │  MineHub Web App  (React)                              │
 (any browser,    │        │                                               │
  installable as  │        ▼                                               │
  a desktop app)  │  MineHub API  (all rules, permissions, calculations)   │ ──► optional export to a
                  │        │                                               │     company's own SQL Server
 Handheld in ───► │        ▼                                               │
 each machine     │  ONE PostgreSQL database (+ PostGIS for maps)          │
 (Data Collector  │  • row-level security: companies can't see each other  │
  App, works      │  • automatic daily backups                             │
  OFFLINE, syncs) └────────────────────────────────────────────────────────┘
```
- **"Desktop app" = the web app.** It opens in a browser on any PC and can be "installed" with its own icon and window. One version to maintain, not two. A true desktop program can be added later if ever needed.
- **The cloud** solves your "LAN 1 / LAN 2" question: every computer anywhere just connects to the same cloud system over the internet.
- **Data Collector App:**
  - stores everything on the device first, then syncs through the API;
  - if there is no signal, it keeps working and sends the data later, so **nothing is lost**;
  - "end of shift" means the operator closes the shift and all remaining data is confirmed as received.

### C2. ❓ DECISION — one database instead of Firebase + SQL
The out-of-sync bugs came from splitting users and companies (Firebase) from everything else (SQL). Options:

| Option | Description | My view |
|---|---|---|
| **A. One system (recommended)** | PostgreSQL holds everything; login handled by the same platform (e.g. Supabase Auth or our own) | Simplest, most reliable, no sync problems |
| B. Firebase for login only | Firebase only checks email/password. **All** profile, company, licence and permission data lives in PostgreSQL | Acceptable if you want to keep Firebase |
| C. Firebase holds profiles/companies/licences + SQL holds the rest | What you had | **Not recommended**: this caused the bugs |

### C3. Proposed stack (I will build and maintain it; you don't need to learn it)
| Part | Choice | Why |
|---|---|---|
| Language | **TypeScript** everywhere | One language for web, API and handheld app; catches many errors before running |
| Web app | **React** + Tailwind CSS (+ a component library) | You already know how it looks; huge ecosystem |
| Tables / bulk edit | Spreadsheet-style grid component with **Excel copy/paste** | Your bulk-edit requirement |
| Charts | Recharts / ECharts | Dashboards |
| Map | **MapLibre/Leaflet** + satellite imagery + DXF import (converted to map shapes using the mine's coordinate system) | Free, works offline later |
| API | **Node.js** (Fastify or NestJS) | Same language; enforces rules; same API for web and handheld |
| Database | **PostgreSQL + PostGIS** with **row-level security** and **migrations** | Rock-solid, free software, handles maps, scales to millions of loads |
| Handheld app | **React Native (Expo)** for Android, local SQLite storage | Offline-first, background GPS, shares code with web |
| Live updates | WebSockets | Live map and dashboards (later phase) |
| Hosting | Free/cheap tiers during development (e.g. Supabase or Neon for the DB, Vercel/Render/Fly or Google Cloud for the app); paid tier only at go-live | Low cost to start; move up as usage grows |
| Quality | Automated tests + GitHub Actions run on every change; separate **test** and **live** environments | Things stop breaking silently |

### C4. Reliability & security rules
- Every record has an owner company, a creator and a timestamp, and changes are kept in the **audit log**.
- No hard deletes of operational data: records are "voided" with a reason.
- Records from handhelds have unique IDs, so duplicates are ignored. Offline data syncs in order.
- Daily automated backups, and the restore procedure is tested.
- Licence and permission checks happen on the server for every request.
- Passwords are never stored by us (handled by the auth provider). All connections are encrypted.

---

## Part D — UI / UX

**Principles**
1. **One clear layout:**
   - left menu with modules (shown only if permitted);
   - top bar with company/mine selector, date and shift selector, and user;
   - content in the middle.
2. **The date + shift + mine you choose stays selected across modules.** You don't re-enter it on every screen.
3. **Tables behave like Excel:** paste from Excel, edit in place, sort, filter, export. Invalid cells are highlighted in red *before* saving, with the reason.
4. **Forms use dropdowns from master data** and auto-fill (truck → capacity, block → material/seam).
5. **Status everywhere:** the day checkpoint bar shows Plan ✔, Dispatch ✔, Geology ⏳, Survey ✖.
6. **Every number in a report can be clicked down** to the rows behind it.
7. **Languages:** ❓ DECISION — Mongolian + English, switchable.
8. **Handheld app:** huge glove-friendly buttons, 1-tap status change, day/night mode, clear "saved / waiting to send" indicator, no typing during the shift.

**Main screens**
- Login / Register (join company / create company)
- Setup wizard
- Dashboard
- Project setup: fleet, operators, locations, blocks, materials, qualities, products, custom fields, shifts, activity codes
- Plan
- Dispatch: shift entry, loads, equipment activity
- Geology
- Survey
- Stockpiles
- Reconciliation
- Reports
- Map
- Daily approval
- User management: users, requests, roles and permissions
- Super admin: companies, licences, support mode

---

## Part E — Roadmap (each phase ends with something working that you test)

| Phase | What you get | "Done" means |
|---|---|---|
| **0. Foundation** | Project structure, test and live environments, automatic tests, **clickable mock-ups** of the main screens for your feedback | You approve the mock-ups and Part B |
| **1. Platform core** | Register / login / reset; companies (owner/subcontractor, 6-digit ID); licences; super admin; join requests; custom roles and permissions; setup wizard | Every scenario from A3 works, with automated tests proving a subcontractor can't see other companies' data |
| **2. Master data** | All project-setup lists with **Excel paste**, import/export; owner → subcontractor inheritance; commodity-agnostic materials | You can load your real mine's fleet, locations, blocks and materials from Excel in minutes |
| **3. Daily data entry** | Plan, Dispatch (loads + equipment activity), Geology, Survey; daily checkpoint and approval; corrections and audit | One real day of your mine's data entered by each department and approved |
| **4. Reports & reconciliation** | Shift/daily/monthly production, KPIs (availability, utilisation, productivity), plan vs dispatch vs survey, stockpile balance, ROM quality, Excel/PDF export, pivot explorer | Reports match your current Excel reports for a test month |
| **5. Data Collector App** | Android handheld app: login, shift start, status buttons, loads, offline storage, end-of-shift submission feeding phase 3 automatically | Pilot on a few machines for 2 weeks, zero lost records |
| **6. Live map & tracking** | Satellite map + DXF overlays, drawn areas with permissions, live positions and status, geofences, automatic load/dump detection, haul distance | Live map used by dispatch; auto cycles match manual counts |
| **7. Advanced** | Report builder, Power BI connection, export to company SQL Server, dispatch optimisation, maintenance/fuel, on-premise option for big customers | Per feature |

**How we work, every phase:**
1. I build a small piece.
2. Tests run automatically.
3. I check it myself in a browser.
4. You get a link to try it on the **test** site.
5. Once you approve, it goes to **live**.

If something breaks, we roll back to the last good version in one step.

---

## Part F — ❓ Questions for you
1. **Database decision (C2):** A (one system, recommended), or B (keep Firebase for login only)?
2. **Data Collector App:** was it actually built? If yes, with what (Kotlin, Flutter, MIT App Inventor…), and is the code somewhere I can see? If not, we build it in phase 5.
3. **First real user:** which mine/company and department will test first? (Your own department at your mine would be ideal.)
4. **Sample files:** please share examples of your current Excel files:
   - dispatch shift report;
   - plan;
   - geology quality;
   - survey actuals;
   - stockpile balance;
   - the reports management receives.

   These define the exact columns and calculations.
5. **Activity/delay codes** you use today.
6. **Coordinate system** of your CAD/DXF drawings (for example UTM zone 48N, or a local mine grid)?
7. **Languages:** Mongolian + English?
8. **Domain:** do you own `minehub.mn` (for the website and emails like admin@minehub.mn)?
9. **Existing data:** is there anything in your old Firebase/Supabase you need to keep, or was it all test data?
