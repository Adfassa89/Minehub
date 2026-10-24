# MineHub — Build Plan v0.3 (simple and focused)

> All the big ideas from the Gemini chats are kept safely in [VISION.md](VISION.md).
> **This file is the only plan we follow.** Everything not listed here waits.

---

## 1. Where we start — one sentence

**Phase 1 replaces the East and West dispatcher Excel workbooks with one web system. Phase 2 adds the Data Collector App, which fills in that same data automatically from the cabs.**

Why start here:
- It's real work that happens **every shift**.
- You already have the data structure (your `DM` and `DB` sheets).
- We can test against your real October 2026 numbers.

Everything else (planning, geology, survey, licences, maps, live tracking) is built **on top of** this foundation later.

---

## 2. Technical decisions — FINAL (you don't need to decide these)

| Topic | Decision | In plain words |
|---|---|---|
| Database | **One database: PostgreSQL, hosted on Supabase for the MVP.** No Firebase | All data in one place, so it can never get "out of sync". Supabase *is* PostgreSQL in the cloud |
| Login | Built into MineHub itself | Same database, no second system |
| "Desktop app" | **Web app** (React + TypeScript), opened in a browser on dispatcher PCs | One program to maintain; works on any PC; can have its own icon |
| Server / rules | **API server** (Node.js + TypeScript) | All calculations and checks happen here, the same for the web and the cab app |
| Cab app (phase 2) | **Android app** (React Native) that works **offline** | Keeps working with no signal and sends the data later |
| Languages | **Mongolian by default**, English switchable | Screens are written in English in the code with a Mongolian translation file. Your own data names (e.g. "Тэсэлсэн хөрс") stay exactly as you enter them |
| Coordinates | **WGS 84 / UTM zone 48N** (EPSG:32648) | Used later for the map and CAD drawings |
| Safety | Code on GitHub with full history, automatic tests, separate **test** and **live** systems, daily backups | Nothing is lost, and every change can be undone |
| Where it runs | **MVP: Supabase cloud** (region close to Mongolia, e.g. Singapore or Tokyo). MineHub talks to the database only through its own API server, using standard PostgreSQL, so it can move to an ETT server later with a backup and restore and no code changes | When ETT IT reviews the working MVP, both options stay open |

---

## 3. What your Excel does today (I studied both files)

Each workbook (13–14 MB, 27–29 sheets) is a small database built by hand:

| Your sheet | What it is | What it becomes in MineHub |
|---|---|---|
| `DM01_Dump` | Truck list: contract, company, fleet № (Парк№), plate, brand, model, year, serial, capacity, axle, trip factor code (Par) | **Equipment list** |
| `DM02_EXTus` | Excavators, dozers, graders, loaders, water trucks | **Equipment list** (same list, different type) |
| `DM04_DaPar` | Trip factor per truck model: m³ per trip for overburden, t per trip for coal, start date | **Trip factors with history**: a change from a new date never changes old shifts |
| `DM03_DaValid` | Codes: materials, seams, companies, crews, shifts, **time allocation codes** | **Code lists** (dropdowns) |
| `DM00_Date` | Calendar: which crew (А Б В Г) works DS/NS each day | **Crew roster**: the crew fills in automatically |
| `DM05_ParaRom`, `DM_06_BunkerTejeel` | ROM stockpiles: location → coal type, stockpile number, valid dates; ТНАЛТ bunker feed | **Destinations & stockpiles** |
| `DB01_Dump_data` (~11,900 rows in October, East) | Truck shift log: moto-hours, km, minutes per time code, excavator, material, seam, trips, destination, volume | **Truck shift record + trip lines** |
| `DB02_EX_tus` (~4,000 rows) | Excavator shift log: moto-hours, minutes per time code, block, material, target, actual, % | **Excavator shift record** |
| `DB03_Plan` | Daily / weekly / monthly plan per company: overburden m³, coking coal t, thermal coal t, rock mass m³ | **Plan targets** |
| `Report…` sheets | Rock mass, overburden, coal by company and day; equipment count; per-excavator, per-block, monthly | **Reports**, generated automatically |
| `Check`, `Check2` | Manual checking | **Automatic checks** while entering data |

### Rules I confirmed from your East data
- **Volume = trips × trip factor.** For example, a Liebherr T264 truck (A81): 14 trips × 81 = 1,134 m³.
- **Excavator volume = the sum of its trucks' trip volumes.** This matches in **1,128 of 1,144** excavator shifts. The rest are typing errors (see below). MineHub will calculate it automatically, so nobody types it twice.
- **Every shift adds up to exactly 720 minutes:** working time plus all time codes, for every truck and excavator. MineHub will check this on every entry.

### Problems I found in the data (exactly what MineHub will prevent)
| Problem found | Example | MineHub fix |
|---|---|---|
| Same machine typed two ways | `LO-131` and `lo-131` on 14 Oct night shift, so the volume landed on the "wrong" loader | Pick the machine from a list, no typing |
| Block names with **Cyrillic letters** mixed with Latin | `А8.3.1_1514` (Cyrillic А) vs `A8.3.1_1514`, `Т2.5.2` vs `T2.5.2`, lowercase `c3.4.5_1494`, leading spaces | Block list with a fixed format; blocks are selected, not typed |
| Two blocks in one cell | `T2.5.3, Т2.5.2_1459` | One block per line; add a second line if needed |
| Crew in lowercase | `в` instead of `В` (99 rows) | Crew comes automatically from the roster |
| Stockpile numbers used as locations | `20095` in the destination column | Destination and stockpile number are separate fields |
| A truck working for 2 excavators gets 2 rows, each repeating the full 720 minutes | Hours can be counted twice if rows are added up | The truck's hours are stored **once**, with trips as separate lines |
| Crew letters typed in **Latin** instead of Cyrillic (West) | `B` / `A` instead of `Б` / `А` (861 rows) | Crew comes automatically from the roster |

### East vs West — same process, slightly different files
| | East | West |
|---|---|---|
| Companies | 2 (TET, KAM) | 4 (YMT, YOT, OTR, YUT) |
| Trucks active in October | 165 | 496 |
| Truck trip lines, 1–22 Oct | ~11,900 | ~29,100 |
| Areas | East | West, West2, Borteeg |
| Extra | — | Light tower log (`DB02_Ltower`), km on excavators |
| Time code columns | Same idea, but **different column names and order** (e.g. "Автосамосвал хүлээлт" vs "Дамп хүлээлт") | |

In Excel, every column difference means separate formulas and reports. In MineHub the **time codes are a list, not columns**: each pit can have its own codes, but reports work the same for every pit and company.

---

## 4. Time allocation model — your question answered

**Yes: "delay/activity codes" means exactly your time allocation model (цаг ашиглалт).** Your mine already has a good one in `DM03_DaValid`:

| Category | Mongolian | Examples from your file |
|---|---|---|
| **Working** | Бүтээлтэй ажилласан | productive minutes |
| **NSP** | Төлөвлөгөөт сул зогсолт | shift change, inspection, meal, safety time, refuelling |
| **OD** | Бүтээлийн бус ажилласан | floor/wall cleaning, excavator relocation, waiting for trucks, geology/survey measurement |
| **OS** | Төлөвлөгөөт бус сул зогсолт | blasting, dust, weather, radio, safety stoppage, power limit, not enough trucks, fuel shortage |
| **SLM** | Төлөвлөгөөт засвар | planned repair, lubrication |
| **ULM** | Төлөвлөгөөт бус засвар | breakdown |

From these, MineHub calculates for every machine, shift, day and month:
- **Availability** = (720 − SLM − ULM) ÷ 720;
- **Utilisation** = (Working + OD) ÷ available time;
- **Productivity** = volume ÷ working hour.

The codes stay **editable by the mine**. You can add a reason and choose its category; nothing is hard-coded.

In phase 2, operators will press these same codes as big buttons on the cab tablet, and the minutes are recorded automatically.

---

## 5. Phase 1 — "Digital dispatcher report" (East + West)

### How data flows today, and in MineHub
- **Today:** operator fills a paper sheet → the **subcontractor's dispatcher** makes a report → the **ETT dispatcher** types or pastes it into the Excel workbook.
- **MineHub:** the subcontractor's dispatcher enters their own company's shift directly (with all checks) → the **ETT dispatcher reviews and approves** → reports update automatically. One retyping step disappears.
- Management looks first at the **production report**, then the **performance report**.

Clickable mock-ups of these screens: see `docs/mockups/` (published as a private link for review).

### Facts confirmed from your data while building the mock-ups
- Coal trip volumes are in **m³**; the report converts to tonnes at **1.4 t/m³** (exactly 1.4000 for both TET and KAM on 22 Oct).
- 22 Oct night shift has rows for **two crews** (Г and Б), although the roster has one crew per shift.
- Three excavator time columns are **not in the time-code list** (`DM03_DaValid`): "ТНАЛТ-н дамп хүлээсэн", "ТНАЛТ зогсож дамп хүлээсэн", "Ажил хаялт". They need a category.

### Screens
1. **Login.** Roles: admin, dispatcher, supervisor, viewer.
2. **Shift entry:**
   - choose **pit** (East / West), **date** and **shift**, and the crew fills in automatically;
   - three tabs: **Trucks | Excavators | Support equipment** (dozers, graders, loaders, water trucks, light towers);
   - each tab is an Excel-like table with dropdowns;
   - you can **paste from Excel**, and mistakes turn red immediately;
   - truck row: moto-hours start/end, km start/end, minutes per time code, then **trip lines** (excavator, material, seam, destination, trips). The volume is calculated for you.
   - excavator row: moto-hours, minutes per time code, block, material, target. Actual volume is calculated from the trucks.
3. **Shift check & close:**
   - list of problems: shift not 720 minutes, trips with no excavator, missing moto-hours, unusual values;
   - the dispatcher closes the shift and the supervisor approves it;
   - changes after approval need a reason and are logged.
4. **Reports:**
   - the same ones you use now: rock mass / overburden / coal by company and day vs plan, equipment count, per excavator, per block, daily coal, ТНАЛТ, monthly;
   - **one click to export to Excel**.
5. **Master lists (admin):** equipment, trip factors, materials, seams, blocks, destinations / stockpiles, crews & roster, time codes, plan. These are first **imported from your existing DM sheets**.

### Steps (each one is checked by you before the next starts)
| Step | Result | How we know it's right |
|---|---|---|
| 1.1 | **Clickable mock-ups** of shift entry and one report | You click through them and say "yes, that's how dispatchers work" |
| 1.2 | Master lists + import from your `DM` sheets | The full truck list (232 trucks in `DM01_Dump`) and excavator/support list (`DM02_EXTus`) appear correctly |
| 1.3 | Shift entry with all checks | A dispatcher can enter one real shift faster than in Excel |
| 1.4 | Import 1–22 October 2026 from both workbooks | The import reports every problem row (like `lo-131`) for you to fix |
| 1.5 | Reports | Numbers match your Excel. Example: East, 22 Oct, total rock mass **232,293 m³** |
| 1.6 | Pilot | Dispatchers enter real shifts in MineHub **alongside** Excel for 2 weeks, then compare |

---

## 6. Phase 2 — Data Collector App (cab tablet)

- Operator logs in → chooses the machine → enters moto-hours / km at the start.
- **Big buttons for each time code.** One tap = status change, and the start and end times are recorded automatically.
- **Trip button** (+1 trip with the current excavator / material / destination). Later, GPS detects trips automatically.
- Works **offline** and sends when there is signal.
- At the end of the shift: moto-hours / km, and the summary is sent.
- The dispatcher sees the result in the **same "Shift check & close" screen** from phase 1, so reports don't change at all.

That's why phase 1 comes first: it defines exactly the data the app must produce.

---

## 7. Later phases (only after 1 and 2 work) — details in [VISION.md](VISION.md)
3. Plan module for planners (replaces typing the plan into `DB03_Plan`)
4. Geology quality, survey actuals, reconciliation, stockpile balance, ROM quality
5. Multi-company platform (sign-up, licences, subcontractor logic)
6. Map with satellite imagery + DXF drawings (UTM 48N), live tracking from the cab app

---

## 8. What I need from you now (only 3 things)
1. **Approve phase 1** as described in section 5, or tell me what's wrong or missing.
2. **How dispatchers work today:**
   - who types the data into Excel, and when (during the shift or at the end)?
   - where does it come from (radio, paper sheets from operators)?

   This decides how the entry screen should look.
3. **Your 3 most important report sheets** (the ones management looks at first).

When you approve, I'll build step 1.1: clickable mock-ups you can open on your phone or PC.
