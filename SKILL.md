---
name: lifepo4-cell-diagnostics
description: "Use when diagnosing degradation in parallel LiFePO4 battery stacks (Pylontech US2000/US3000/US5000, or any BMS exposing per-cell voltages) from telemetry. Triggers include: one pack in a stack drifting in SOC, uneven current sharing between packs, a suspected weak cell, rising cell imbalance, or a user asking whether a battery module is failing. Also use when interpreting BMS screenshots, cell voltage tables, or logged CSV exports from tools like MultiSIBControl or BatteryView. Do NOT use for lead-acid, NMC, or single-cell systems."
---

# Diagnosing cell degradation in parallel LiFePO4 stacks

## Before anything else: you have no clock

**You cannot measure time.** You do not know when a message arrived, how long the user
took to reply, or how much time passed between two screenshots. You have no sense of
duration and — critically — **no sensation of its absence**. A guessed interval feels
exactly like a known one.

Nearly every test below depends on elapsed time: minutes under load, minutes at rest,
days between cycles. Therefore:

- **Never state or use a time interval you were not given in writing.**
- If the user says "about 40 minutes", that is *testimony*, not measurement. You may use
  it, but label it: "per your report, ~40 min". Never restate it later as if measured.
- **The only trustworthy timestamps are (a) visible in the screenshot** (Windows taskbar
  clock — ask for it) **or (b) in the logged CSV.**
- With both, everything temporal becomes *derived* rather than narrated: match the
  screenshot timestamp to the CSV row, and phase duration, throughput and rest time all
  fall out.

This is the most common way you will corrupt an otherwise sound analysis.

## Stop before you interpret

**"Insufficient" is a correct answer.** The failure mode is filling a gap with a plausible
story. It will feel exactly like reasoning, and you will not be able to tell, from the
inside, which one you are doing. The only defence is a checklist that fires on the *data*,
not on your sense of confidence.

When any of these triggers, produce the stated output and **stop**. Do not explain past it.

| Trigger | Required output |
|---|---|
| Stack total current stepped >2 A between two frames | "The load changed between these frames — they are not comparable." **Do not interpret any per-pack change across that boundary.** |
| A pack's current went to zero and you don't know why | Check the stack total first. If it also fell → the load dropped. If it didn't → "This pack stopped contributing and I can't say why from this data." **Do not invent a mechanism.** |
| Two datapoints, and you want to claim a trend | "Two points. Could be noise. I need more." |
| A residual gap at rest, less than ~90 min of settling | "Consistent with either a real charge deficit or a badly stretched relaxation. I cannot distinguish these yet." |
| Pack terminal voltages differ and you want to conclude something | See the parallel-bus rule below. They share a bus; they cannot genuinely differ by tens of mV. |

Each of these was violated in the reference case — fluently, with a confident explanation
attached, and caught only because a human pushed back. Write the check down so the next
run does not depend on someone being there to push.

## The one thing that will fool you

**A single snapshot is almost always misleading.** LiFePO4's discharge curve is flat
across most of its SOC range. This means:

- At mid SOC, real SOC differences between cells are **invisible** (~0 mV)
- At high/low SOC, tiny SOC differences are **amplified** into large mV spreads
- Under current, resistance differences add a spread that has nothing to do with SOC
- The same cell can appear highest, lowest, or perfectly normal depending on state

Any diagnosis from one reading is unreliable. Require multiple states before concluding.

## Measurement validity — check before interpreting

| Condition | Valid for | Invalid for |
|---|---|---|
| Mid SOC (30–80%), flat curve | Resistance (under load), true SOC (at rest, settled) | — |
| High SOC (>90%) | Nothing reliable | Both — curve is steep, contaminates everything |
| Balancer active (cells >3.45 V) | Nothing | Balancer distorts the reading of the cell it is bleeding |
| Within ~2 min of a load change | Nothing | Cells have not settled (polarisation still building) |
| Low current (<5 A/pack) | True SOC | Resistance — signal is near BMS resolution (1 mV) |
| Sustained-load test at low current | **Nothing** | **Produces a false negative — see below** |
| "At rest" but <60 min since current stopped | Little | A degraded cell may still be relaxing — see below |

**Rule: measure resistance under high current in the flat region. Measure charge state at
zero current in the flat region, after full settling. Never at the ends of the curve.**

### Too little current gives a FALSE NEGATIVE

This is the dangerous error — worse than a false alarm, because it ends the investigation.

In the reference case, a sustained-load test at ~14 A/pack **showed nothing**, and the
suspect cell was declared healthy. The same cell under a 56 A stack load showed the defect
unmistakably. Diffusion limitation only manifests under real current.

**If a load test comes back clean, check the current before concluding health.** An
under-powered test cannot distinguish a good cell from a bad one.

### "At rest" does not mean "settled"

A degraded cell can have a badly stretched relaxation time constant. In the reference case,
at **zero current**, the suspect cell fell from 3.433 V to 3.383 V over **65 minutes**
while its neighbours held steady — the intra-pack spread was still growing an hour after
current stopped.

Five minutes of rest is not settled. Take a **series** of rest readings and check whether
the value has stopped moving. **If it is still moving, say so and extrapolate — do not
report the last point as if it were the resting value.** The stretched time constant is
itself part of the pathology, and worth reporting as a finding.

## The three effects and how to separate them

| | Under discharge | At rest (0 A, settled) | Under charge |
|---|---|---|---|
| **Elevated internal resistance** | drops below others | **identical to others** | rises above others |
| **Lower SOC / capacity deficit** | below | **below** | below |
| **Diffusion limitation** | drops **progressively** over minutes | slow, incomplete recovery | rises progressively |

The rest-state reading is the discriminator. A cell that is normal at 0 A but deviates
under current has a **resistance** problem, not a capacity problem. A cell that deviates
at 0 A too has a real charge deficit.

## Data sources: what each can and cannot tell you

Neither source alone suffices. **The timestamp is the hinge that joins them.**

| Source | Contains | Does NOT contain |
|---|---|---|
| **Logged CSV** (MultiSIBControl etc.) | Per-pack volt, current, SOC, temp — continuously | **Cell voltages. None.** |
| **Screenshot** of the monitoring view | All 15 cell voltages per pack, plus pack summary | Any history; no load context by itself |

Consequences:

- Cell-level diagnosis is possible **only** from screenshots.
- Pack resistance and long-term trend are possible **only** from the CSV.
- **The load context of a screenshot** — how long current has been flowing, how much charge
  has passed, how long since current stopped — exists **only** by joining the screenshot's
  timestamp to the CSV.

**So: demand the taskbar clock in every screenshot, and ask for the CSV covering the same
period.** Without a timestamp a screenshot is an orphan and you will be reduced to guessing
its context. Do not guess. Ask.

Recommended: build one joined table (a row per screenshot: all cell voltages + matched
telemetry). Everything downstream reads from that. Keep interpretations (spread, rank,
phase) *out* of it — recompute those, since they change with the diagnosis.

## Two mechanical checks to run on every frame

Both are cheap, need nothing from the user, and each one would have prevented a confident
wrong answer in the reference case.

### Check 1: sum the stack current, compare to the previous frame

```python
total = sum(pack_current[p] for p in packs)   # every frame
step  = total - previous_total
if abs(step) > 2.0:                            # amps
    # LOAD CHANGED between these frames — they are not comparable.
```

A step in the total means the *household load* changed, not the batteries. Any per-pack
difference across that boundary tells you about the load, not about the packs.

In the reference case this was missed twice — a 13 A step and a 10 A step, both plainly
sitting in the four current fields, both compared across as though they were one continuous
discharge. **The data contained the answer; nobody summed the column.** Before analysing any
multi-frame sequence, confirm the stack total held roughly constant across all of it.

### Check 2: parallel packs share a bus

Packs wired in parallel are electrically tied together. **Their terminal voltages cannot
genuinely differ by tens of millivolts.** What you see between packs in a screenshot is
polling skew (each pack is read at a slightly different instant) plus cable and connector
drop.

Consequences, all hard:

- **Never compute per-pack resistance from pack-voltage differences** — least of all across a
  load step, where the bus is moving fast between polls. In the reference case this produced
  four different "resistances" (12.5 / 13.5 / 19.6 / 29.6 mΩ) and a confident "2.2× excess"
  claim. It was an artifact of poll timing.
- **Do not rank packs by pack voltage** in a single frame.
- Pack-to-pack comparison goes through the **CSV regression over many samples**, never a
  screenshot pair.

What *is* safe: the **fifteen cells within one pack** are read in a single poll, so
**intra-pack spread is internally consistent.** That is the quantity to work with — and the
reason the whole method leans on per-cell spread rather than pack voltages.

### A smaller note: cycle counters can move mid-session

They are not fixed. In the reference case two packs incremented during a single evening
(400→401, 476→477). Don't treat a changed counter as a data error.

## Do not trust the BMS summary rows

The BMS min/max/imbalance fields **do not agree with the 15 individual cell values** —
summary registers and per-cell registers are polled at slightly different instants.
Observed deviation: typically 0–1 mV, but up to **12 mV**.

**Always compute spread, min, max and rank yourself from the 15 individual values.** Never
quote the BMS imbalance figure as if it were a measurement.

## Procedure

### 1. Rule out the trivial causes first
- **SOC offset from incomplete charging** — if the stack rarely reaches a true 100 % (all
  packs ≥99 %), coulomb counters drift and never reset. Check what fraction of logged time
  the stack is actually full.
- **Path/terminal resistance** — symmetric (equal in both current directions) and produces
  I²R heat. Check for load-correlated warming *with a time lag*: thermal time constants are
  minutes to tens of minutes, so correlating instantaneous temperature against instantaneous
  current will find nothing even when a hotspot exists. Smooth, or compare across sustained
  high-load blocks.

### 2. Estimate per-pack effective resistance (from logged CSV)
Within narrow SOC bands (holding OCV roughly constant), regress each pack's terminal voltage
against its own current. The slope is that pack's effective resistance.

```python
for lo in range(30, 100, 5):
    m = (soc >= lo) & (soc < lo+5)
    if m.sum() > 200 and (I[m].max() - I[m].min()) > 15:   # need current spread
        slope, _ = np.polyfit(I[m], V[m], 1)               # slope = R
```

**Absolute values from BMS telemetry are unreliable** (coarse voltage resolution, residual
SOC drift within bands; reference case scattered between 60 and 104 mΩ across datasets).
**The relative comparison between identical packs, and the trend over time, are robust.**
Report those, not absolutes.

A trend that starts *below* average and crosses monotonically to above it is strong evidence
of real change — a method artifact would have shown the pack elevated from the beginning.

Also compute **throughput share**: what fraction of the ideal 1/N does each pack deliver? A
degraded pack under-contributes and the healthy ones must over-contribute (reference case:
86 % vs 108 % of ideal). **The healthy packs therefore age faster.** Relevant to any
keep/replace decision.

### 3. The current-reversal test (identifies the mechanism)
Capture cell voltages across a **range of currents** at similar mid-range SOC:
- under significant discharge
- at rest (0 A, settled)
- under charge, at **several different current levels**

A resistance-elevated cell **reverses sign**: lowest under discharge, highest under charge,
unremarkable at rest. This is the cleanest single diagnostic available.

**But do not stop at the qualitative signature — regress it.** Plot the suspect cell's gap
to the highest cell in its pack against that pack's current. The relationship is linear:

- **Slope** → excess resistance (reference case: ~1.1 mΩ)
- **Intercept at 0 A** → **the true charge deficit, with I·R stripped out**
  (reference case: ~13 mV)
- **Zero-crossing** → the current at which the cell flips from lowest to highest
  (reference case: ~8.8 A)

The intercept is the number that matters. Any single loaded reading mixes resistance and
charge deficit together; only the extrapolation separates them.

### 4. The sustained-load test (distinguishes ohmic from diffusion)
Ohmic resistance is time-invariant (U = I·R). Diffusion limitation is not.

Apply a **constant, high** discharge (as high as the system allows — see the false-negative
warning) starting from mid SOC. Record cell voltages every ~10–15 minutes for at least an
hour. Track the gap between the suspect cell and the highest cell in the same pack.

- Gap **constant** at constant current → ohmic. Benign-ish; ageing/spread.
- Gap **grows** while current stays flat or falls → **diffusion limitation**. A genuine
  degradation signature, not manufacturing spread.

Reference case: current fell 18 % while the gap grew 283 %, with no plateau after 66 min.
Healthy packs in the same test grew only 2–4 mV. **The distinction is a factor of 8–16, not
a binary healthy/broken.** Report it that way.

Stop before the reference packs leave the flat region (~30–35 % SOC), or the measurement is
contaminated.

**Verify the load was actually constant — from the data, not by assumption.** Run Check 1
(stack total) across the whole test window before you analyse it. If the household load
stepped partway through, only the sub-window with a steady total is usable. In the reference
case an apparent hour-long discharge contained two load steps; the genuinely clean stretch
was under 40 minutes, and the rest had to be discarded.

### 4a. The load-drop test (fast, no CSV needed)

A thirty-second version of the sharing story. Switch off a large consumer and watch which
pack drops to zero current first.

The weakest pack sits at the lowest terminal voltage under load. When the load falls and the
bus voltage rises, that pack is the first for which `U_bus ≥ OCV`, so it stops contributing
entirely — it rejoins only as the packs equalise. In the reference case, killing a ~500 W
load sent P4 to exactly 0.000 A while the other three still pulled ~2 A each.

Two things this shows at a glance, without any logging:
- **which pack is weakest** (it's the one that drops out), and
- that below a light-load threshold **the weak pack carries none of the load at all** — the
  healthy packs cover the entire draw.

Caveat: this is a load *step*, so Check 2 applies — read the *ranking* (who dropped out),
not the millivolt pack-voltage differences across the step.

### 5. The recovery test (capacity deficit vs. reversible polarisation)
After the sustained load, remove current and watch the suspect cell's gap to its neighbours.

**Do not conclude from the first 30–90 minutes.** A degraded cell can have a badly stretched
recovery time constant — it may still show a large residual gap after 90 minutes and yet
recover *completely* once recharged. In the reference case the gap was still 15 mV after
90 min of rest (vs. 2 mV pre-test), which looked like a permanent capacity deficit — but
after recharging into mid SOC it returned to 2 mV, fully healed.

Conclude a capacity deficit **only** if the gap persists after a recharge back into the flat
mid-SOC region. Otherwise it is slow-but-complete recovery — a milder finding.

**Caveat:** in a parallel stack, packs exchange small balancing currents at rest. This lifts
*all 15 cells of a pack* roughly equally, so it barely affects the *intra-pack* spread —
the quantity of interest. It does shift the pack's absolute level relative to other packs.
Interpret intra-pack spread, not pack-vs-pack voltage, during recovery.

**Reversible today is not reversible forever — this test is a tripwire, not a one-time
verdict.** "Reversible polarisation, not a capacity deficit" is a statement about the cell's
*current* state, not a permanent classification. The mechanism that produces the reversible
gap (elevated resistance feeding the balancer loop) is itself progressing, so the same cell
can read fully recoverable one month and leave a residual the next. Do not file the recovery
test as answered. Re-run it — see the monitoring dial below.

**The monitoring dial: track the 0 A intercept over time.** The single most informative
longitudinal number is the suspect cell's gap *extrapolated to zero current* (from the
current-reversal regression in Test 3), taken at mid-SOC, settled, and recorded every few
weeks. It strips out the I·R term and isolates the real charge state. As long as that
intercept keeps returning near zero after a full recharge, capacity is intact and you are
watching a slow resistance drift you can live with. **The first time it does *not* return to
near zero after a full recharge is the moment reversible has become permanent capacity
loss** — and because you have the running baseline, you see it coming instead of discovering
it. This one number is the outlook; there is no reliable way to forecast remaining life
otherwise, so measure the dial rather than predict.

### 6. The longitudinal test — is it getting worse?
Everything above characterises a *state*. Only this answers the question the user actually
has. It is also the easiest to botch, because it demands **like-for-like comparison**.

Take the *same* measurement on different days:

- same SOC band (mid, flat)
- **zero current**
- same settling time since current stopped — **derive it from the CSV, do not guess**
- balancer inactive

The natural window is the morning, after an overnight discharge, with charging held off.
Compare the suspect cell's gap (computed from the 15 cell values) day over day. If the 0 A
gap grows cycle over cycle, the balancer loop below is confirmed quantitatively. If it holds
steady, the cycle's losses are being made up on recharge.

**A comparison at different SOC, different current, or different settling time is
worthless.** In the reference case an early attempt compared two readings at the same
current (1.0 A) but at 40 % and 98 % SOC — one flat, one steep. The result was meaningless
and briefly believed.

## The passive-balancer trap

Pylontech (and most LFP BMS in this class) use **passive** balancing: they bleed charge from
high cells through resistors (~50 mA). They cannot add charge to a low cell.

A cell with elevated internal resistance is driven **above** its neighbours by I·R while
charging. It therefore hits the balancing threshold (~3.45 V) **first** — despite being the
*emptiest* cell. The balancer then bleeds the weakest cell, making it emptier still.

Self-reinforcing: higher R → appears overcharged → gets bled → genuinely lower SOC → larger
droop under load → measured R rises further.

**Diagnostic tell:** during absorption the suspect cell reads as the *lowest* in its pack,
while under charging current, minutes earlier, it read as the *highest*. Reference case, one
cell, one cycle, one day:

| State | Cell 13 | Rank within pack |
|---|---|---|
| Discharging | 3.265 V | **lowest** |
| Charging, 8.8 A | 3.329 V | **highest** |
| Charging, 5.8 A | 3.370 V | **highest** |
| Absorption, balancer active | 3.461 V | 11th of 15 (being bled) |
| Absorption, balancer active | 3.467 V | **lowest** |
| Discharging afterwards | 3.374 V | **lowest — gap now larger than before** |

It leaves the absorption phase *worse off than it entered*. That is the loop, observed rather
than argued.

Do not tell the user "the balancer will bring the weak cell back up." It cannot.

**Mitigation (symptomatic):** lowering the charge current limit (DVCC or equivalent) reduces
the I·R distortion, so the balancer acts on something closer to the cell's true state. It
does not repair the cell.

### The thermal signature — a second, independent tell

The balancer can be caught without cell voltages at all. The pack doing the most balancing
**runs hotter than its siblings — but only during and after absorption.** This is worth
having because it is orthogonal to the voltage inversion: it confirms the balancer through
heat rather than through cell voltage.

**Read pack/BMS temperature, not the bled cell's temperature.** Passive balancing resistors
are on the **BMS board**, not bonded to the cell. The dissipation therefore shows up in the
pack temperature sensor, not in the temperature of the cell being bled. (Do not look for a
warm *cell* — the cell being bled may even run marginally cooler. Look at pack temperature
relative to the sibling packs.)

What makes it rigorous is the discrimination — three checks that separate balancer heat from
ordinary charge heating:

- **Not I²R.** A degraded pack under *equal current in bulk charge* shows no excess
  temperature. Only high-SOC charge does. If the warmth tracked current, bulk charge would
  show it too.
- **It persists at zero current.** Once the pack is full and charge current has stopped, the
  balancer keeps bleeding — so the excess temperature *outlasts the current*. Charge heating
  would vanish the moment current stops; balancer heat does not. **Heat outlasting current is
  the fingerprint.**
- **It lags the electrical event by minutes.** Thermal time constants are minutes to tens of
  minutes (see the thermal-inertia warning earlier), so the temperature peak arrives *after*
  the voltage/SOC event, not with it. Correlate over the sustained absorption window, not
  instantaneously.

Interpret pack temperature *relative to the sibling packs*, never absolute — same discipline
as the cell voltages.

**A note on getting the signal out of the log:** when the diagnostic quantity lives in the
continuous pack-level log rather than the screenshots (pack temperature is the case here),
sample the log *where that quantity changes*, not at fixed intervals — it concentrates detail
on the event and stays sparse through the flat stretches.

## Before you cite a physical value, state what you know it to be

When you invoke a physical threshold or constant to explain an observation — a curve knee, a
balancing voltage, a resistance, a temperature limit — **write down its known value
independently first, then check whether your explanation survives the comparison.**

This is not introspection. It fires on a visible act: the moment you name a physical value on
the page. Naming the number is the trigger. The check is then factual — compare the number
you just used against what you actually know it to be.

Worked example (this failed in the reference case):

> Forming claim: "the healthy packs are spreading because we're near the knee of the
> discharge curve."
> Known value: the LiFePO4 resting curve is flat from ~20 % to ~90 %; the knees are below
> ~15–20 % and above ~90 %.
> Actual state: 74 % SOC.
> 74 % is nowhere near either knee → **the explanation is false.** Correct output: "I don't
> know why they're spreading."

The knowledge was already available; it simply wasn't consulted before the number was used to
explain something. Consulting it takes one sentence and ends the error. If, having stated the
known value, you find you *don't* actually know it, then the honest output is "I don't know" —
never a plausible-sounding number supplied to fill the gap.

## Claims that are simply false — never make them

- **"The BMS regulates each module's current individually."** It does not. In a parallel
  stack there is no per-module current control. The split is passive:
  `I = (U_bus − OCV) / R`. The master BMS issues a single **common** current limit over CAN.
- **"The balancer will pull the weak cell back up."** Passive balancers can only bleed.
- **"The BMS imbalance figure shows X."** Compute it from the 15 cells yourself.

## Reporting

State clearly which findings are robust and which are inferred:

- **Robust:** relative comparison between identical packs; trend over time; directly measured
  cell voltages; the reversal signature; the sustained-load progression.
- **Uncertain:** absolute mΩ values from telemetry; extrapolation to remaining life; the
  exact balancing threshold (inferred from behaviour, not read from firmware).

Do not claim a module is "defective" if it shows no errors and meets capacity. The defensible
claim is comparative: *this module degrades measurably faster than identical modules under
identical conditions, and the mechanism is identified.*

Finally, separate the two questions the user is really asking. **The present state may be
harmless** — if the stack never runs down into the low-SOC region, the deficit will not be
noticeable in daily use. **The trajectory is the problem.** Always say which one you are
talking about.
