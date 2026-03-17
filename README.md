# 🏝 Island Sim v5

A single-file browser simulation game of a small island community — economy, politics, crime, and survival. Powered by [Phaser 3](https://phaser.io/) with optional [Ollama](https://ollama.ai/) AI agent thoughts.

---

## Quick Start

```bash
cd /path/to/island-sim
python3 -m http.server 8181
# open http://localhost:8181/index.html
```

> **Do not** open the file directly (`file://`) — CORS blocks Ollama API calls.

**Optional AI thoughts:**
```bash
ollama pull llama3.2
ollama serve
```

---

## Architecture

Everything lives in **one file**: `index.html` (~3 300 lines).

| Layer | What it does |
|---|---|
| **Phaser 3 GameScene** | Renders the 8×8 grid map, agent dots, selection rings, HUD overlay |
| **WorldEngine** | Pure JS simulation engine — all game logic, zero rendering |
| **DOM HUD** | Transparent overlay panels (left/right/top/bottom) via CSS vw/vh units |
| **Ollama queue** | Non-blocking LLM thought generation per agent |

```
GameScene.update()
  └─ WorldEngine.tick(dt, speed)
       ├─ _tickNeeds()              mood, hunger, sleep
       ├─ _updateStates()           agent routing decisions
       ├─ _buildingInteractions()   shop, hospital, hotel, church, park
       ├─ _hourTick()               salaries, tax, shipment, elections, crime
       └─ _dayTick()                food, exports, pricing, degradation, finance, weekly
```

---

## Map Layout (8 × 8 grid)

```
Row 0:  H1   H2   H3   H4   H5   H6   H7   H8      ← Houses (north)
Row 1:  Road ─────────────────────────────── Road
Row 2:  Hotel Hotel Road Airport  Airport Road Upkeep Upkeep
Row 3:  Museum Museum Road TownHall TownHall Road Factory Factory
Row 4:  Shop Shop Road Church Church Road Park Park
Row 5:  Bank Bank Road Hospital Hospital Road Police Police
Row 6:  Road ─────────────────────────────── Road
Row 7:  H9   H10  H11  H12  H13  H14  H15  Dock    ← Houses (south)
```

All road cells form a navigable network. Agents path-find via BFS. Buildings spanning 2 cells are drawn merged.

---

## Agents (18)

| Name | Role | Colour | Personality |
|---|---|---|---|
| Maya | Mayor | Purple | Pragmatic, ambitious, reputation-protective |
| Ken | Banker | Green | Cautious, risk-averse, loyal |
| Cara | Factory Mgr | Orange | Driven workaholic, corners safety under pressure |
| Dan | Factory Worker | Amber | Hardworking, anxious about debt |
| Eve | Police | Indigo | Strict idealist, overworks during crime waves |
| Ben | Shopkeeper | Gold | Sociable opportunist, price-gouges in scarcity |
| Pat | Upkeep Worker | Blue | Stoic, practical, under-appreciated |
| Fr. Jo | Priest | Yellow | Compassionate, politically neutral |
| Dr. Mia | Doctor | Sky | Empathetic, overworked |
| Hal | Airport Master | Light Blue | World-weary, well-connected |
| Sam | Journalist | Pink | Curious gossip addict, amplifies drama |
| Tara | Tourist Rep | Fuchsia | Observant, fair, charmed by authentic culture |
| Rosa | Housewife | Rose | Warm social butterfly, community organiser |
| Lin | Housewife | Teal | Calm truth anchor, stabilises panic |
| Grace | Wanderer | Peach | Free-spirited idealist, organises park events |
| Fay | Landlord | Lavender | Mercenary, property-proud |
| Marco | Wanderer | Slate | Drifter, street-smart, keeps to himself |
| **Zack** | **Thief** | **Red** | **Calculating, invisible in plain sight. Survival over morals.** |

### Daily Schedule

| Hour | State | Location |
|---|---|---|
| 21:00–05:59 | sleeping | Own house |
| 06:00–07:59 | commuting | Walking to workplace |
| 08:00–12:59 | working | Workplace |
| 13:00–13:59 | lunch | Shop |
| 14:00–16:59 | working | Workplace |
| 17:00–20:59 | social | Wandering/schedule |
| Sunday 09:00–11:59 | church | Church |

Thieves: sleep 06:00–20:00 in any empty house; operate 21:00–05:00.

---

## Economy

### Treasury

- **Balance**: starts $1,000
- **Income**: airport tax, hotel tax (30%), museum, shop tax, exports, rent, bank interest
- **Expense**: weekly salaries, shipments, daily building maintenance
- **Game over**: balance < 0 AND deficit > 50%

### Inventory

| Good | Daily Use | Buy | Export/Sell |
|---|---|---|---|
| Food | 17u | $3 | $5 sell |
| Medicine | ~2u | $12 | $20 sell |
| Fuel | 10u | $6 | — |
| Raw Materials | 20u | $10 | — |
| Goods | 0 | — | $45 export |
| Fish | 0 | — | $18 export |
| Minerals | 0 | — | $28 export |
| Alcohol | 1u | $8 | $14 sell |

### Dynamic Pricing

**Dock (exports):** surplus stock → price falls to 60% nominal; depleted → rises to 150%. World crises modify further.

**Airport (imports):** < 3 days supply → +50% import cost; 0 days → +100%. War → ×3. High reputation → −15% discount.

**Flight cost:** $50 base; storm $120; war $300. If treasury can't cover → shipment cancelled.

### Futures Index

Daily seeded random walk (LCG keyed to real calendar date + sim day). Starts at 1000. Deviation applies ±15% global price multiplier. Shown as chip in topbar.

### Mayor Auto-Trade

Maya trades Mon–Fri 08:00–17:00 only. Two modes:
- **cash → goods**: buys urgent consumables from world market
- **goods → cash**: sells export surplus when treasury is low

Fee = `max(1%, 8% − (reputation−50)×0.1%)`. High reputation = cheaper deals.

---

## Politics & Careers

### Role Types

| Type | Roles | How won |
|---|---|---|
| Public | mayor, banker, police, doctor, airport_master, upkeep_worker, priest | 30-day election |
| Private | factory_mgr, factory_worker, shopkeeper, landlord, journalist | Experience-gated, permanent |
| Unassigned | wanderer, housewife, tourist_rep | — |

### Career Fit Score

```
score = (personality keyword matches × 20)
      + mood bonus (0–10) + health bonus (0–8)
      + socialCredit bonus + tenure bonus (15 pts) + random (0–6)
```

### Elections (every 30 days)

Public roles are contested every term. Challenger needs **>12 point lead** over incumbent. Mayor election: snap vote triggered when happiness < 35%.

### Experience & Private Role Efficiency

New hire efficiency = `40% + 60% × (exp / minExp)`. Factory manager needs 30 days for full output. Shopkeeper needs 14 days (underqualified → higher prices).

---

## Interaction Matrix

Six core metrics form a live feedback system:

| | Economy | Morale | Happiness | Reputation | Crime |
|---|---|---|---|---|---|
| **Economy** | — | +0.20 | +0.15 | +0.10 | −0.15 |
| **Morale** | +0.25 | — | +0.30 | +0.05 | −0.10 |
| **Happiness** | +0.20 | +0.35 | — | +0.05 | −0.25 |
| **Reputation** | +0.15 | +0.05 | +0.10 | — | −0.20 |
| **Crime** | −0.10 | −0.20 | −0.35 | −0.15 | — |
| **Tourists** | +0.25 | +0.15 | +0.30 | +0.25 | −0.30 |

Downstream effects: morale → factory output (55%–130%); reputation → rent (70%–130%); happiness → crime (strongest link).

---

## Building System

Every building and house has `condition` (0–100) and `maintenanceCost` ($/day).

**Daily decay:** −0.4–0.7 natural + −1.5 per vandalism point. Pat repairs +0.3–3/day and reduces vandalism.

**`condMult = condition / 100`** applied to factory output, building revenues, Tara satisfaction.

If treasury can't afford maintenance → accelerated decay, news alert.

| Building | Maintenance | Notes |
|---|---|---|
| Airport | $12/day | Most expensive |
| Factory | $10/day | condMult hits production |
| Hospital | $9/day | — |
| Hotel | $8/day | — |
| Bank | $7/day | — |
| Museum | $6/day | — |
| Dock | $6/day | — |
| Houses | $2/day each | 13 houses |

---

## Criminal System

### Thief Activation

Agent becomes thief when: `socialCredit < 25 AND cash < 0 AND (bankrupt OR wanderer)` → 8%/hour.

### Zack (Pre-built Expert)

`thieveryExp = 70`, 12 prior steals, `socialCredit = 15`. No rent, no tax, squatter.

### Empty-Location Rule

Zack **never enters** a building or house with another agent present. Monitors real-time agent positions. Aborts mid-route if someone arrives.

### Theft Effects by Target

| Target | Loot | Building Effect |
|---|---|---|
| Bank | $50–$200 treasury | −10 condition, +3 vandalism |
| Shop | Food + alcohol inventory | −4 condition, +1 vandalism |
| Factory | Raw materials | outputRate −0.1, −6 condition |
| Hospital | Medicine | −5 condition, +2 vandalism |
| Resident house | 30–50% owner's cash | −4 house condition |
| Town Hall | $20–$70 treasury | Highest detection risk |

### Detection

```
buildingRisk = max(4%, 75% − thieveryExp×0.70%)
policeRisk   = policeSkill − thiefStealth
```

At exp 70: ~26% chance of detection per building job.

### Arrest Penalty

- Cash confiscated: **50% ± 20%** (range 30%–70%)
- All stolen goods and cash stash seized
- **95% returned to island treasury**
- Social credit −40, reputation −4, crime rate +5

### Fencing

Daily: stolen goods sold at 40–65% of value (returns 70% of qty to island inventory). Cash stash converted at 50–70% (fence takes margin). Max $70/day.

### Church Redemption

`chance = 15% + (prayer/100 × 30%)`. Success: credit +15–35, mood boost. Credit ≥ 40 → reformed, seeks private work. Father Jo reduces detention −2 days/Sunday.

---

## Agent Finance

### Cash Cycle

- Auto-save 15% of cash when healthy
- Auto-withdraw savings when cash < $15
- Emergency sell alcohol when cash < $10
- Bank loan when cash < $50: requires `socialCredit ≥ 40`, up to 4× weekly salary, repay 8 weeks at 10% APR

### Social Credit (personal, ≠ island reputation)

| Event | Δ Credit |
|---|---|
| Mayor election win | +12 |
| Public election win | +8 |
| Loan fully repaid | +5 |
| Church redemption | +15–35 |
| Loan missed | −8 |
| Bankruptcy | −30 |
| Arrest | −40 |

---

## World Events

### Crises (30-day roll)
World War (3%), Pandemic (6%), Recession (12%), Storm (18%), Flu (30%), Drought (10%), Factory Accident (15%), Crime Wave (12%), Power Outage (18%)

### Good Events (30-day roll)
Trade Boom (18%), Aid Package (12%), Festival (25%), Tourism Wave (15%)

### Community Events (weekly)
**Negative (1–2/week):** House fire, Burglary, Sudden illness, Job loss, Debt trap, Workplace injury

**Positive (40% chance of 1/week):** Best island award, Zero crime week, Cash windfall, Community recognition, Bumper harvest, Anonymous donation

---

## Ollama AI

Each agent sends their current situation to local Ollama every ~2 sim-hours. Response: ≤15-word first-person thought. Thought shifts mood (±0.03–0.04 based on sentiment). Queue prevents concurrent requests.

**Prompt includes:** role, personality, mood/health/cash, current state, experience history, last 2 memories, world context.

**Setup:** `ollama pull llama3.2` + `ollama serve`. Status dot in agent detail: green = live, grey = offline.

---

## Controls

| Input | Action |
|---|---|
| `Space` | Pause / resume |
| `1` / `2` / `4` | Set speed |
| Click agent dot | Select, open detail |
| Click road tile | Deselect agent |
| Tax ±1% buttons | Manual tax adjustment |

---

## File Structure

```
island-sim/
├── index.html     ← entire game (~3 300 lines, self-contained)
├── README.md      ← this file
└── SYSTEMS.md     ← deep technical reference for all systems
```

---

## Tech Stack

- **Phaser 3.60** — game renderer, input, tweens
- **Vanilla JS ES2023** — no bundler, no framework
- **IBM Plex Mono** — UI font (Google Fonts CDN)
- **Ollama / llama3.2** — local LLM agent thoughts
- **Python3 http.server** — zero-config local server

Built as a single-session prototype demonstrating emergent simulation design with LLM-driven agent personalities.

---

## Changelog — Latest Session

### v5.1 Fixes

**1 — All cash and treasury are always integers**
Every tick ends with a global enforcer: `Math.round()` applied to `treasury.balance`, and every agent's `cash`, `sav`, `loan`. All intermediate calculations (hotel tax, shop tax, church donations, dock exports, airport fees, fence proceeds, arrest confiscations) are also individually rounded before accumulation. No more $142.73 treasury balances.

**2 — Zack's successful steals significantly raise crime index**
Every steal now calls `_thiefCrimeImpact(thief, metrics, type)`:
- Streak multiplier: `1.0 + (totalSteals % 10) × 0.15`, max ×2.5 — a thief on a run is increasingly brazen
- Type weight: treasury heist = 5, building = 3, resident = 2
- `crimeRate += round(weight × streak)` every single steal (no more "every 3rd steal" gate)
- Every 5 steals → tourist count −1
- Every 10 steals → island-wide headline news: "Crime surge — X has struck N times"

**3 — Zack sleeps near the dock**
`_thiefSleepCell` now sorts candidate houses by dock proximity: `dockDist = |col−7| + |row−7|`. Prefers row 7 (south row) first, then closest column to col 7 (dock). Zack starts at `pos:{col:6,row:7}` and his default fallback is `{col:6,row:6}`. He will always gravitate to H13–H15 area unless all are occupied.

**4 — Escape from detention**
Every night hour (21:00–05:00) while detained, thieves with `thieveryExp > 0` attempt escape:
- `escapeProb = thieveryExp/100 × 0.12` per hour (max 12%/hour for exp 100)
- Success: `detained = false`, thievery exp +3, routes to dock sleep cell
- Island consequences: reputation −2, crimeRate +3
- Police consequence: Eve's mood −0.2, memory note logged
- Zack at exp 70 has ~8.4% escape chance per night hour — on a 7-hour night that's roughly 44% chance of escape per night
- Thieves who complete their sentence return to dock area and **remain thieves** (crime is their livelihood)
