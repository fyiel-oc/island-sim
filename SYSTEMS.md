# Island Sim v5 — Systems Reference

Deep technical documentation for every simulation system. All code lives in `index.html`.

---

## Table of Contents

1. [Simulation Loop](#1-simulation-loop)
2. [Agent System](#2-agent-system)
3. [Economy Engine](#3-economy-engine)
4. [Pricing & Futures](#4-pricing--futures)
5. [Career & Election System](#5-career--election-system)
6. [Interaction Matrix (Metrics)](#6-interaction-matrix-metrics)
7. [Building Degradation & Maintenance](#7-building-degradation--maintenance)
8. [Criminal System](#8-criminal-system)
9. [Agent Personal Finance](#9-agent-personal-finance)
10. [World Events](#10-world-events)
11. [Ollama AI Integration](#11-ollama-ai-integration)
12. [HUD & Rendering](#12-hud--rendering)
13. [Data Structures Reference](#13-data-structures-reference)

---

## 1. Simulation Loop

### Time Model

- **Logical resolution:** 1280×720 px canvas, scaled via `Phaser.Scale.FIT`
- **Sim time multiplier:** 1 real-second = 4 sim-minutes at 1× speed
- **Speed settings:** 0× (pause), 0.5×, 1×, 2×, 4×
- **Day length:** 6 real-minutes = 24 sim-hours at 1× speed

### Engine Tick Structure

`WorldEngine.tick(dtMs, simSpeed)` runs every Phaser frame:

```js
// Sub-tick accumulator pattern
this._acc += dtMs * simSpeed;
while(this._acc >= STEP_MS) {
  this._acc -= STEP_MS;
  this._tickNeeds(STEP_MS * simSpeed / 1000 * 4);  // per step
  this._buildingInteractions(STEP_MS * simSpeed / 1000 * 4);
  this._updateStates();
  this._recalc();     // metrics interaction matrix
  this._checkGO();    // game-over check
  while(t.minute >= 60) { t.minute -= 60; t.hour++; this._hourTick(); }
  while(t.hour >= 24)   { t.hour -= 24; this._dayTick(); }
}
```

### Hour Tick Events (`_hourTick`)

Fires once per sim-hour using a dedup Set (`_fired`). Pruned every 200 entries.

| Trigger | Action |
|---|---|
| Mon 08:00 | `_paySalaries()` + `_agentFinanceWeekly()` |
| Thu 08:00 | `_collectTax()` |
| Supply day 09:00 | `_shipment()` |
| Sun 09:00 | `_sundayChurch()` + `_churchRedemption()` |
| Sun 20:00 | `_taraReview()` |
| Day % 30 == 0, 00:00 | `_crisisRoll()` + `_careerElection()` |
| Day % 7 == 0, 06:00 | Snap election check (happiness < 35%) |
| Every hour | `_thiefTick()` + `_policePpatrol()` |

### Day Tick Events (`_dayTick`)

Runs at 00:00 each sim day.

```
_dailyFood → _updateDynamicPrices → _updateFutures → _dailyExports
→ _tourism → _buildingDegradation → _roadDecay → _expireCrises
→ _museumDaily → _agentFinanceDaily → _zackFenceLoot
→ experience accumulation → weekly events (if Sunday)
```

---

## 2. Agent System

### Agent State Machine

```
sleeping → commuting → working
                    ↘ idle (bad mood + chance)
         → lunch
         → social
         → church (Sunday)
         → hospital (health < 25%)
         → stealing (thief, night)
         → detained (arrested)
         → patrolling (police)
```

State is set in `_updateStates()` every engine step. Detained and thief agents bypass the normal schedule.

### Pathfinding (BFS)

`bfsPath(c0, r0, c1, r1)` — breadth-first search on road cells only. Buildings are not traversable (agents teleport to adjacent road cell when arriving at a building). Road cells precomputed in `ROAD_CELLS[]`.

### Agent Movement

`GameScene._advanceAgentWalk(agent)` — Phaser tween per step. Duration = `max(60ms, STEP_MS / simSpeed)`. Overlap offset: agents sharing a cell spread ±15px horizontally.

### Mood Calculation

```js
base = 0.5
     - hunger×0.35
     - sleep×0.25
     - (1−health)×0.40
     - (cash<30 ? 0.20 : 0)
mood = clamp(base, -1, 1)
```

### Crime Probability

```js
crimProb = max(0,
  0.02
  + (1 − (mood+1)/2) × 0.20
  + (cash<20 ? 0.12 : 0)
  + (isThief ? 0.30 : 0)
  + (bankrupt ? 0.08 : 0)
  + (60−socialCredit)/100 × 0.20
)
```

### Needs Decay

Per sim-second of non-sleep time:
- `hunger += 0.05 × r` where `r = seconds/3600`
- `sleep += 0.035 × r`

During sleep: `sleep -= 0.12 × r`, `hunger += 0.01 × r` (slower)

---

## 3. Economy Engine

### Salary System (`_paySalaries`, Monday 08:00)

Salary cut applied when deficit > thresholds:
- deficit > 40%: pay at 40%
- deficit > 25%: pay at 65%
- deficit > 10%: pay at 85%
- else: full salary

Agents with salary cut get `mood -= 0.12`.

### Tax Collection (`_collectTax`, Thursday 08:00)

`weekly = house.value × taxRate / 52`

Agents with `noTax = true` (thieves) are exempt. Agents who can't pay get `mood -= 0.08`.

### Rent Collection (`_collectRent`, Thursday 09:00)

`rent = house.rentPerWeek × rentMult`

`rentMult = clamp(0.7 + 0.6 × (reputation/100), 0.7, 1.3)` — high island reputation = higher rent.

Fay keeps 70% of rent income; 30% to treasury. Agents with `noRent = true` exempt.

### Food Consumption (`_dailyFood`)

17 units consumed per day (1/agent). If inventory < 17: agents start getting hungry (hunger rate increases). If food < 5: health damage begins.

### Export Production (`_dailyExports`)

Runs Mon–Fri only (working days). Output formula:

```js
prod = dailyProd × factoryExp_efficiency × factory.outputRate × factory.condMult
```

- `factoryExp_efficiency`: 0.4–1.0 based on factory_mgr days in role
- `factory.outputRate`: 0.55–1.30 from morale interaction
- `factory.condMult`: 0.30–1.0 from building condition

Export revenue = `prod × exportPrice × futures.multiplier`

### Hotel Revenue

$60/guest/night, 30% to treasury. Hotel guests = `floor(tourists × 0.7)`. Charged 21:00–22:00 if agent (`Fay`) in hotel cell.

### Museum Revenue

$15/visitor. `visits = floor(tourists × 0.8)`. Charged daily via `_museumDaily`.

### Park Revenue

$2/tourist/hour when Tara is in park cell (probabilistic, 10% per tick).

### Bank Interest

10% APR ≈ 0.19%/week on outstanding loans. Interest paid to treasury each Monday repayment.

---

## 4. Pricing & Futures

### Dynamic Price Engine (`_updateDynamicPrices`)

Runs daily before exports. Computes:

**Export price multiplier per good:**
```
ratio = qty / (dailyProd × 3)   // healthy = 3-day buffer

priceMult =
  ratio == 0   → 1.50  (recovery premium)
  ratio < 0.5  → 1.0 to 1.5 (sliding)
  ratio <= 1   → 1.0  (nominal)
  ratio > 1    → max(0.60, 1.0 − 0.13×(ratio−1))  (surplus discount)

+ crisis modifiers: world_war×0.1, recession×0.75, trade_boom×1.25
+ futures.multiplier (eM component)
```

**Import price multiplier:**
```
daysLeft = qty / use

buyMult =
  ≥7 days → 1.0
  3–7     → 1.2
  1–3     → 1.5
  0       → 2.0

+ crisis: world_war×3.0, pandemic×4.0 (medicine only), recession×1.4
+ reputation discount: max(0.6, 1.0 − 0.15×((rep−50)/50))
```

**Flight cost per shipment:**
- Normal: $50 — Storm: $120 — Recession: $80 — War: $300

### Futures Index (`_updateFutures`)

```js
seed = realDate_YYYYMMDD (recomputed each day)
r1 = LCG(seed + simDay×7)   // in [0,1]
r2 = LCG(seed + simDay×13)

islandBias = (economy−50)/100 × 0.008
crisisBias = −crises.length × 0.005
move       = (r1−0.5)×0.04 + islandBias + crisisBias + (r2−0.5)×0.01

value = max(200, value × (1+move))
changePct = move × 100
multiplier = clamp(value/1000, 0.85, 1.15)
```

The multiplier feeds into `_eM` (export multiplier) with 90/10 weighted smoothing.

---

## 5. Career & Election System

### Career Definitions

```js
CAREERS = {
  mayor:          { sal:150, wp:{3,3}, fit:[...], unique:true, public:true, minExp:0 },
  banker:         { sal:140, wp:{0,5}, fit:[...], unique:true, public:true, minExp:0 },
  police:         { sal:100, wp:{6,5}, fit:[...], unique:true, public:true, minExp:0 },
  doctor:         { sal:130, wp:{3,5}, fit:[...], unique:true, public:true, minExp:0 },
  airport_master: { sal:90,  wp:{3,2}, fit:[...], unique:true, public:true, minExp:0 },
  upkeep_worker:  { sal:80,  wp:{6,2}, fit:[...], unique:true, public:true, minExp:0 },
  priest:         { sal:60,  wp:{3,4}, fit:[...], unique:true, public:true, minExp:0 },
  factory_mgr:    { sal:110, wp:{6,3}, fit:[...], unique:true, public:false, minExp:30 },
  factory_worker: { sal:70,  wp:{6,3}, fit:[...], unique:false,public:false, minExp:7  },
  shopkeeper:     { sal:120, wp:{0,4}, fit:[...], unique:true, public:false, minExp:14 },
  landlord:       { sal:100, wp:{0,2}, fit:[...], unique:true, public:false, minExp:14 },
  journalist:     { sal:90,  wp:null,  fit:[...], unique:false,public:false, minExp:7  },
  wanderer/housewife/tourist_rep: unassigned
}
```

### Fit Score Algorithm

```js
function careerFitScore(agent, roleKey) {
  const per = agent.per.toLowerCase()
  const keyHits = career.fit.filter(k => per.includes(k)).length
  return Math.round(
    keyHits × 20
    + ((agent.mood+1)/2) × 10        // mood bonus 0–10
    + agent.health × 8                // health bonus 0–8
    + (agent.socialCredit−50) × 0.1  // credit bonus
    + (agent.role===roleKey ? 15 : 0) // tenure bonus
  )
}
```

### Public Election (`_careerElection`, day%30==0)

1. Filter roles: `public && unique && role !== 'mayor'`
2. For each role: score all agents (add 0–6 random, +8 if broke wanderer)
3. Sort descending
4. Winner displaces incumbent only if `winnerScore ≥ incumbentScore + 12 + 15 (tenure bonus)`
5. Displaced agent → wanderer/housewife, mood −0.15
6. Winner gets role, salary, workplace, mood +0.20, socialCredit +8

### Mayor Election (`_mayorElection`)

- Normal: challenger needs `incumbentScore + 15` margin
- Snap (happiness < 35%): any challenger wins immediately
- Winner: mood +0.30, socialCredit +12

### Private Role Hiring (`_agentFinanceWeekly`)

Broke wanderers/housewives (cash < $50) scan for open private roles. Best `careerFitScore` wins. Hire note includes efficiency warning if `exp < minExp`.

### Experience Accumulation

`_dayTick` increments `agent.experience[role]` by 1 for every day agent is in `state === 'working'`.

### Efficiency Calculation

```js
eff = exp >= minExp ? 1.0 : 0.4 + 0.6 × (exp / minExp)
// ramps from 40% at day-0 to 100% at minExp days
```

---

## 6. Interaction Matrix (Metrics)

### Raw Input Calculation

```js
rawEC = min(1, balance/4000)×40 + (workingAgents/total)×35 + max(0,1−deficitPct/100)×25
rawMO = clamp(50 + avgMood×50, 0, 100)
rawHA = (avgHealth×0.45 + (1−avgHunger)×0.35 + (avgMood+1)/2×0.20) × 100
rawRE = metric.reputation × 0.97 + tara.satScore × 0.03   // sticky social credit
rawCR = avgCrimProb × 100
```

### Cross-Influence Pass

```js
ec0=rawEC/100, mo0=rawMO/100, ha0=rawHA/100, re0=rawRE/100, cr0=rawCR/100

ecDelta = 0.20×(mo0−0.5) + 0.10×(re0−0.5) + 0.15×(ha0−0.5) − 0.15×(cr0−0.3)
moDelta = 0.25×(ec0−0.5) + 0.30×(ha0−0.5) − 0.10×(cr0−0.3)
haDelta = 0.20×(ec0−0.5) + 0.35×(mo0−0.5) − 0.25×(cr0−0.3) + 0.05×(re0−0.5)
reDelta = 0.15×(ec0−0.5) + 0.10×(ha0−0.5) − 0.20×(cr0−0.3)
crDelta = −0.35×(ha0−0.5) − 0.10×(ec0−0.5) − 0.15×(re0−0.5) − 0.20×(mo0−0.5)

// Apply, capped at ±15 pts
economy    = clamp(rawEC + ecDelta×15, 0, 100)
morale     = clamp(rawMO + moDelta×15, 0, 100)
happiness  = clamp(rawHA + haDelta×15, 0, 100)
crimeRate  = clamp(rawCR + crDelta×15, 0, 100)
reputation = clamp(reputation + reDelta×6, 0, 100)  // ±3 max per tick
```

### Downstream Effects

```js
// Morale → factory output
factory.outputRate = clamp(0.55 + 0.75 × (morale/100), 0.55, 1.30)

// Reputation → rent multiplier
treasury.rentMult = clamp(0.7 + (reputation/100)×0.6, 0.7, 1.3)

// Reputation → import price
repDiscount = 1.0 − 0.15 × ((reputation−50)/50)  // ±15%

// All → tourists
tBase = 3 × (0.25×ec0 + 0.15×mo0 + 0.30×(happiness/100) + 0.25×(reputation/100) + 0.05)
      × crimeP × taraScore
tourists = crisisBlock ? 0 : floor(tBase × 8 × tourismMultiplier)
```

### Tara's Review (`_taraReview`, Sunday 20:00)

```js
score = 50
  + (crimeRate<15 ? +15 : crimeRate<30 ? +8 : crimeRate>50 ? −18 : 0)
  + (upkeep.efficiency>80 ? +8 : upkeep.efficiency<50 ? −10 : 0)
  + (park.eventToday ? +10 : 0)
  + (happiness>70 ? +8 : happiness<50 ? −8 : 0)
  + (economy>70 ? +6 : economy<40 ? −6 : 0)
  + (morale>65 ? +5 : morale<40 ? −5 : 0)
  + (crises.length===0 ? +5 : 0)
  + tara.mood × 12

tara.satScore = clamp(score, 0, 100)
reputation = clamp(reputation×0.6 + satScore×0.4, 0, 100)
```

---

## 7. Building Degradation & Maintenance

### Daily Decay Formula

```js
decay = 0.4                         // natural wear
      + (vandalism×1.5)             // vandalism amplifier
      + random(0, 0.3)              // noise

condition = max(0, condition − decay)
```

### Pat's Repair

When Pat is in `state === 'working'`:
```js
repairPower = 0.3 + (experience.upkeep_worker/100) × 0.7  // 0.3–1.0 base
repairPerBuilding = repairPower × 3  // ×3 to all buildings

// Vandalism repair is 50% faster
if(vandalism > 0) repair × 1.5
vandalism = max(0, vandalism − 1)  // one point removed per day
```

### Condition → Output Multiplier

`condMult = max(0.30, condition/100)`

Applied to: factory goods production, hotel revenue calculation, Tara satisfaction score.

### Maintenance Cost

```js
cost = maintenanceCost × (1 + vandalism×0.30)
```

Total charged to treasury daily. If treasury < $100 → no repair, all buildings decay extra −0.5/day.

### Vandalism from Zack

| Building | Vandalism on success | Vandalism on caught |
|---|---|---|
| Bank | +3 | +1 |
| Factory | +2 | +1 |
| Hospital | +2 | +1 |
| Shop | +1 | +1 |
| Others | +1 | +1 |
| Houses | +1 | — |

---

## 8. Criminal System

### Thief Activation (organic)

Every hour for non-thief, non-detained agents:
```js
shouldTurn = (socialCredit < 25)
          && (cash < 0)
          && (bankrupt || role==='wanderer' || role==='housewife')
          && (random() < 0.08)
```

On activation: `isThief=true, noRent=true, noTax=true, squatter=true`

### Zack's Starting State

```js
{ isThief:true, thieveryExp:70, successfulSteals:12,
  socialCredit:15, noRent:true, noTax:true, squatter:true }
```

### Night Operation Schedule

`_thiefTick()` runs every sim-hour. At night (21:00–05:00):
- 45%: `_thiefRobBuilding`
- 30%: `_thiefRobResident`
- 25%: `_thiefRobTreasury`

During day (06:00–20:00): routes to empty house to sleep.

### Empty-Location Enforcement

Before every theft attempt and routing decision:
```js
const agentPos = new Set(agents.filter(a=>a.id!==thief.id&&!a.detained)
                               .map(a=>`${a.pos.col},${a.pos.row}`))
```

Building/house selected only if not in `agentPos`. On arrival, re-checked — aborts if occupied mid-route.

### Detection Probability

```js
// Building (standard)
detectProb = max(4%, 75% − thieveryExp×0.70%)
// Bank alarm bonus
detectProb × 1.4
// Police station
detectProb × 1.6
// Resident house
detectProb = max(4%, 65% − thieveryExp×0.60%)
// Town hall (hardest)
detectProb = max(10%, 90% − thieveryExp×0.75%)
```

### Police Patrol (`_policePpatrol`)

Runs every hour during night and work hours. Routes to:
- 40%: random road cell near known thief position (±2 tiles)
- 60%: random road cell anywhere

Detection on same/adjacent tile:
```js
policeSkill = 0.30 + min(0.50, policeExp/100×0.50)   // 30%–80%
thiefStealth = thieveryExp/100 × 0.70                  // 0%–70%
detectChance = max(5%, policeSkill − thiefStealth)
```

### Arrest Consequences (`_arrestThief`)

```js
penaltyRate = clamp(0.50 + (random()−0.5)×0.40, 0.30, 0.70)  // 30–70%
confiscateCash  = thief.cash × penaltyRate
confiscateStash = thief.stolenCash          // all hot cash
confiscateGoods = stolenGoods × faceValue × 0.50

treasury += (confiscateCash + confiscateStash + confiscateGoods) × 0.95
```

Additionally: `socialCredit −40`, `reputation −4`, `crimeRate +5`, `detainedDays = 3–7`.

### Fencing (`_zackFenceLoot`, daily)

```js
// Physical goods
fenceRate = 0.40 + (thieveryExp/100)×0.25  // 40–65%
// 60% chance per item type per day
// Goods flow back into island inventory at 70% quantity

// Cash stash
fenceCashRate = 0.50 + (thieveryExp/100)×0.20  // 50–70%
// Up to $70/day, fence takes (1−rate) as margin
```

### Church Redemption (`_churchRedemption`, Sunday)

```js
chance = 0.15 + (prayer/100)×0.30
// On success:
socialCredit += 15–35
mood += 0.20
if(isThief && socialCredit >= 40): isThief = false
// Father Jo at church: detained agents −2 days (25% chance)
```

---

## 9. Agent Personal Finance

### Weekly Finance (`_agentFinanceWeekly`, Monday)

**Loan repayment:**
```js
payment = min(loanWeekly, loan)
if(cash >= payment):
  cash -= payment; loan -= payment
  interest = loan × 0.0019  // 10% APR ÷ 52 weeks
  treasury += interest
  if(loan ≤ 0): socialCredit += 5
else:
  missed = payment − cash
  cash = 0; loan += round(missed × 1.05)  // 5% penalty
  socialCredit -= 8
```

**Job seeking (when cash < $50):**
1. Try bank loan: `socialCredit ≥ 40` required. Amount = `min(sal×4, 200)`, repay 8 weeks
2. Try open private role: best fit score wins, includes low-exp warning

### Daily Finance (`_agentFinanceDaily`)

- Detention countdown: `detainedDays` → 0 → release
- Squatters/thieves bypass all debt tracking
- Cash < 0 → `debtDays++`, `mood −0.08`
- 7+ debtDays + loan maxed → bankruptcy
- Cash < $15: auto-withdraw up to $50 savings
- Cash > $150 + salary > 0: 15% auto-save (30% chance)

### Bankruptcy

```js
bankrupt = true
cash = sav = loan = 0
mood = −0.8; socialCredit −30
role → wanderer, wp=null, sal=0
```

---

## 10. World Events

### Crisis Roll (`_crisisRoll`, day%30==0)

```js
prayerReduction = prayer / 100 × 0.10  // church attendance reduces crisis chance
for each crisis/good:
  if(random() < (prob − prayerReduction) / 30):
    _applyEvent(ev)
```

### Crisis Effects (`_applyEvent`)

```js
if(fx.importBlocked): supply.status = 'blocked'
if(fx.moraleHit):     metrics.morale -= fx.moraleHit
if(fx.moraleBonus):   metrics.morale += fx.moraleBonus
if(fx.crimeBonus):    metrics.crimeRate += fx.crimeBonus
if(fx.importMult):    _iM *= fx.importMult
if(fx.exportMult):    _eM *= fx.exportMult
if(fx.tourMult):      _tM *= fx.tourMult
if(fx.factDrop):      factory.outputRate *= (1−fx.factDrop)
if(fx.injure):        agent[id].health -= 0.3
if(fx.foodBonus):     inv.food.qty += fx.foodBonus
if(fx.medBonus):      inv.medicine.qty += fx.medBonus
```

Crises have duration `[minDays, maxDays]`. `_expireCrises` removes ended ones and resets multipliers.

### Community Events (`_weeklyCommEvents`, Sunday)

1 or 2 negative events (55% chance each). 1 positive event (40% chance). Weighted random selection. Events target random eligible agents for personal effects, or island-wide for metrics effects.

### Weekly Summary (`_buildWeeklySummary`, Sunday)

Snapshot of all metrics, treasury, agents (employed count, bankruptcies), mayor name, active crises. Displayed as modal overlay on right panel for 12 seconds. Auto-dismissed or click ✕.

---

## 11. Ollama AI Integration

### System Architecture

```
GameScene.update()
  ├─ (every frame) if(ollamaOnline && random < 0.0003×speed):
  │    pick random agent → queueThink(agent, worldCtx, onDone)
  └─ (on agent click) → queueThink(agent, worldCtx, refreshPanel)

THINK_QUEUE (serial):
  _processThinkQueue()
    ├─ shift job from queue
    ├─ await ollamaThink(agent, ctx)
    ├─ agent.lastThought = response
    ├─ agent.memory.push("[thought] " + response)
    ├─ mood shift based on sentiment
    ├─ if(onDone) onDone(thought)
    └─ setTimeout(_processThinkQueue, 500)  // 500ms gap
```

### Prompt Template

```
You are [name], [role] on a small island community.
Personality: [per]
Status: mood [X]% health [Y]% cash $[Z]
Currently: [state] at [col],[row]
Experience: [role Nd, ...]
Recent memory: [last 2 entries]
World: Day [N], treasury $[B], morale [M]
Respond with ONE sentence (max 15 words), first person, present tense.
```

### Sentiment Mood Shift

```js
if(/worried|fear|danger|crisis|fail|starv|broke/.test(thought)):
  mood -= 0.04
else if(/happy|great|good|hope|excit|proud|love/.test(thought)):
  mood += 0.03
```

### Health Check

`checkOllama()` polls `localhost:11434/api/tags` every 15 seconds with 2-second timeout. Updates all `.od` dots and `.ai-lbl` spans via `querySelectorAll`.

### Status Display

- Green pulsing dot = Ollama live, model loaded
- Grey dot = offline or no models
- "Ollama offline — run: `ollama pull llama3.2`" shown in agent panel when offline

---

## 12. HUD & Rendering

### Layout System

All CSS uses `vw`/`vh` units. Conversion: `1px at 1280w = 0.078125vw`.

```
Grid rows: 8.33vh / 1fr / 5.56vh  (≈60/main/40 at 720h)
Grid cols: 25vw / 1fr / 27.34vw   (≈320/map/350 at 1280w)
```

HUD overlay is positioned via `ResizeObserver` + `getBoundingClientRect` to match the Phaser canvas after FIT scaling.

### Agent Dot Colours

| State | Glow Colour |
|---|---|
| working | Green `#22c55e` |
| church | Gold `#fcd34d` |
| sleeping | Dark blue `#1e3a8a` |
| hospital | Red `#ef4444` |
| patrolling | Indigo `#6366f1` |
| stealing (night) | Bright red `#dc2626` |
| detained | Orange `#f97316` |
| default | Agent colour |

Thieves pulse at 0.45 opacity at night (vs 0.22 normal).

### Selection Ring

`Graphics.strokeCircle()` — 3px white ring, depth 8. Redrawn on every tween completion and heartbeat tick. Cleared on deselection or new selection.

### News Fade System

Each news entry stores `data-day`. Age calculation: `now − entryDay` sim-days.

| Age | Opacity |
|---|---|
| Same day | 1.0 |
| 1 day | 0.75 |
| 2 days | 0.55 |
| 3 days | 0.35 |
| 4+ days | 0.18 |

CSS: `transition: opacity 1.5s ease`

---

## 13. Data Structures Reference

### INIT (global state object)

```js
{
  time: { day, week, dayOfWeek, hour, minute, simSpeed },
  inv: {
    food:     { qty, use, buy, sell, nominalBuy, nominalSell, unit },
    medicine: { qty, use, buy, sell, nominalBuy, nominalSell, unit },
    fuel:     { qty, use, buy, nominalBuy, unit },
    rawMat:   { qty, use, buy, nominalBuy, unit },
    goods:    { qty, dailyProd, exportPrice, nominalExport, dockStock },
    fish:     { qty, dailyProd, exportPrice, nominalExport, dockStock },
    minerals: { qty, dailyProd, exportPrice, nominalExport, dockStock },
    alcohol:  { qty, use, buy, sell, nominalBuy, nominalSell },
  },
  treasury: {
    balance, income, expense, taxRate, deficitPct,
    rentIncome, rentMult,
  },
  metrics: {
    economy, morale, happiness, reputation, crimeRate, tourists
  },
  deltas: { treasury, economy, morale, happiness, reputation, crimeRate, tourists },
  supply: { status, interval, nextDay, theft },
  prayer, crises, log,
  bldg: {
    [buildingKey]: {
      open, condition, maintenanceCost, vandalism, condMult,
      // building-specific: guests, outputRate, loanBook, ...
    },
    houses: {
      H1..H13: { condition, maintenanceCost, vandalism }
    },
    dockPriceIndex, airportFlightCost, airportPriceIndex,
  },
  futures: {
    value, prev, change, changePct, history[14], multiplier, seed
  },
  weeklySummary: { ... },
}
```

### Agent Object (mkA defaults)

```js
{
  // Identity
  id, name, ini, g, age, role, sal,
  col, ch,   // Phaser colour int, CSS hex string
  per,       // personality string (searched for career keywords)
  ns,        // news sensitivity multiplier
  sp,        // spouse agent id or null
  wp,        // workplace cell {col,row} or null

  // Position
  pos: {col, row},
  houseCell: {col, row},
  houseId,

  // State machine
  state, destination, path, pathStep,

  // Stats
  mood,     // -1.0 to 1.0
  health,   // 0.0 to 1.0
  needs: { hunger, sleep },  // 0.0 to 1.0
  crimProb,

  // Finances
  cash, sav,
  loan, loanWeekly, debtDays, bankrupt,
  socialCredit,  // 0–100, personal (≠ island reputation)
  noRent, noTax, squatter,  // thief exemptions

  // Career
  experience,  // { roleKey: daysWorked }

  // Crime
  criminalRecord,
  isThief, detained, detainedDays,
  thieveryExp,  // 0–100 stealth skill
  successfulSteals, detectedSteals,
  stolenCash, stolenGoods,  // fencing stash
  squatterCell,   // current sleep cell
  _lastTarget,    // routing dedup key

  // AI
  memory,      // string[], max 8 entries
  lastThought, // latest Ollama response
  fuelUsedToday,
}
```

### HOUSES

```js
{
  H1: { col, row, residents:['maya','ken'], value:900, rentPerWeek:0, label:'Maya+Ken' },
  // ...
  H13: { col:4, row:7, residents:['marco'], value:420, rentPerWeek:0, label:'Marco' },
}
```

Rent = 0 means owner-occupied. Rent > 0 means tenants pay Fay.

### CAREERS

```js
{
  roleKey: {
    sal,            // weekly salary
    wp,             // workplace {col,row}
    fit,            // personality keywords for scoring
    unique,         // only one holder at a time
    public,         // elected (true) or hired (false)
    minExp,         // days needed for full efficiency
  }
}
```

### CRISES / GOODS

```js
{
  id, prob, dur:[min,max], sev,
  type: 'world'|'local',
  note: { type, icon, title, body },
  fx: {
    importBlocked, importMult, exportMult, tourMult,
    moraleHit, moraleBonus, crimeBonus,
    factDrop, sickPct, foodMult, medMult,
    injure, foodBonus, medBonus, roadDrop,
    tourStop,
  }
}
```

---

## Engine Method Index

| Method | Trigger | Purpose |
|---|---|---|
| `_tickNeeds` | Every step | Hunger, sleep, mood, crimProb |
| `_updateStates` | Every step | Agent state/destination routing |
| `_buildingInteractions` | Every step | Hospital, shop, church, hotel, park |
| `_recalc` | Every step | Interaction matrix, metrics, tourists |
| `_checkGO` | Every step | Game-over detection |
| `_hourTick` | Each sim-hour | Salaries, tax, shipment, elections, crime |
| `_dayTick` | Each sim-day | Food, exports, pricing, degradation, finance |
| `_dailyFood` | Daily | Food consumption and health effects |
| `_updateDynamicPrices` | Daily | Supply/demand repricing |
| `_updateFutures` | Daily | Futures index random walk |
| `_dailyExports` | Daily (weekdays) | Factory production and export revenue |
| `_buildingDegradation` | Daily | Condition decay, maintenance cost |
| `_tourism` | Daily | Airport tax, hotel occupancy |
| `_museumDaily` | Daily | Museum visitor revenue |
| `_roadDecay` | Daily | Upkeep efficiency |
| `_agentFinanceDaily` | Daily | Detention, debt, auto-save, bankruptcy |
| `_zackFenceLoot` | Daily | Fence stolen goods and cash stash |
| `_agentFinanceWeekly` | Monday | Loan repayment, job seeking, bank loans |
| `_paySalaries` | Monday | Weekly salaries with deficit cut |
| `_collectTax` | Thursday | House value tax |
| `_collectRent` | Thursday | Rental income to Fay |
| `_shipment` | Every 3 days | Restock imports, pay flight cost |
| `_sundayChurch` | Sunday 09:00 | Mood boost, prayer accumulation |
| `_churchRedemption` | Sunday | Thief/convict redemption rolls |
| `_taraReview` | Sunday 20:00 | Tourist satisfaction, reputation update |
| `_crisisRoll` | Day % 30 | World/local crisis and good event rolls |
| `_careerElection` | Day % 30 | Public role elections |
| `_mayorElection` | Day % 30 + snap | Mayor democratic vote |
| `_mayorTaxAdjust` | Sunday | Automatic tax rate adjustment |
| `_weeklyCommEvents` | Sunday | Community random events |
| `_buildWeeklySummary` | Sunday | Compile weekly stats snapshot |
| `_thiefTick` | Every hour | Thief targeting, routing, steal attempts |
| `_thiefRobBuilding` | Night | Break into empty buildings |
| `_thiefRobResident` | Night | Enter empty houses to steal cash |
| `_thiefRobTreasury` | Night | Raid town hall when empty |
| `_arrestThief` | On detection | Detain, confiscate, news |
| `_policePpatrol` | Every hour | Eve routes toward thieves on roads |
| `_mayaAutoTrade` | Work hours | cash↔goods trading via airport |
| `_expEfficiency` | On demand | Private role output multiplier |
| `_expireCrises` | Daily | Remove ended crises, reset multipliers |
| `_applyEvent` | On crisis roll | Apply crisis effects to state |
| `_news` | On events | Add to news list, emit to HUD |
| `_ledger` | On transactions | Cash flow log entry |
