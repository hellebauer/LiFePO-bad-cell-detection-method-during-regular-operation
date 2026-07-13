---
name: lifepo4-cell-diagnostics
description: "Use when diagnosing degradation in parallel LiFePO4 battery stacks (Pylontech US2000/US3000/US5000, or any BMS exposing per-cell voltages) from telemetry. Triggers include: one pack in a stack drifting in SOC, uneven current sharing between packs, a suspected weak cell, rising cell imbalance, or a user asking whether a battery module is failing. Also use when interpreting BMS screenshots, cell voltage tables, or logged CSV exports from tools like MultiSIBControl or BatteryView. Do NOT use for lead-acid, NMC, or single-cell systems."
---

# Diagnosing cell degradation in parallel LiFePO4 stacks

## The one thing that will fool you

**A single snapshot is almost always misleading.** LiFePO4's discharge curve is flat
across most of its SOC range. This means:

- At mid SOC, real SOC differences between cells are **invisible** (they produce ~0 mV)
- At high/low SOC, tiny SOC differences are **amplified** into large mV spreads
- Under current, resistance differences add a spread that has nothing to do with SOC
- The same cell can appear highest, lowest, or perfectly normal depending on state

Any diagnosis from one reading is unreliable. Require multiple states before concluding.

## Measurement validity — check before interpreting

| Condition | Valid for | Invalid for |
|---|---|---|
| Mid SOC (30–80%), flat curve | Resistance (under load), true SOC (at rest) | — |
| High SOC (>90%) | Nothing reliable | Both — curve is steep, contaminates everything |
| Balancer active (cells >3.45 V) | Nothing | Balancer distorts the reading of the cell it is bleeding |
| Within ~2 min of a load change | Nothing | Cells have not settled (polarisation still building) |
| Low current (<5 A/pack) | True SOC | Resistance — signal is near BMS resolution (1 mV) |

**Rule: measure resistance under high current in the flat region. Measure SOC at rest in the flat region. Never at the ends of the curve.**

## The three effects and how to separate them

A cell can deviate for three distinct reasons. They are separable by their signature:

| | Under discharge | At rest (0 A) | Under charge |
|---|---|---|---|
| **Elevated internal resistance** | drops below others | **identical to others** | rises above others |
| **Lower SOC / capacity deficit** | below | **below** | below |
| **Diffusion limitation** | drops **progressively** over minutes | slow, incomplete recovery | rises progressively |

The rest-state reading is the discriminator. A cell that is normal at 0 A but deviates
under current has a **resistance** problem, not a capacity problem. A cell that deviates
at 0 A too has a real charge deficit.

## Procedure

### 1. Rule out the trivial causes first
Before blaming a cell, confirm the pack-level effect is not simply:
- **SOC offset from incomplete charging** — if the stack rarely reaches a true 100 %
  (all packs ≥99 %), coulomb counters drift and never reset. Check what fraction of
  logged time the stack is actually full.
- **Path/terminal resistance** — this is symmetric (equal in both current directions)
  and produces I²R heat. Check for load-correlated warming with a time lag (thermal
  time constants are minutes; instantaneous correlation will miss it).

### 2. Estimate per-pack effective resistance (from logged CSV)
Within narrow SOC bands (to hold OCV constant), regress each pack's terminal voltage
against its own current. The slope is that pack's effective resistance.

```python
for lo in range(30, 100, 5):
    m = (soc >= lo) & (soc < lo+5)
    if m.sum() > 200 and (I[m].max() - I[m].min()) > 15:   # need current spread
        slope, _ = np.polyfit(I[m], V[m], 1)               # slope = R
```

**Absolute values from BMS telemetry are unreliable** (limited voltage resolution,
residual SOC drift within bands). **The relative comparison between identical packs,
and the trend over time, are robust.** Report those, not absolutes.

### 3. The current-reversal test (identifies the mechanism)
Capture cell voltages in three states at similar mid-range SOC:
- Under significant discharge (>10 A per pack)
- At rest, ≥5 min after load ends
- Under significant charge (>10 A per pack)

A resistance-elevated cell **reverses sign**: lowest under discharge, highest under
charge, unremarkable at rest. This is the cleanest single diagnostic available.

### 4. The sustained-load test (distinguishes ohmic from diffusion)
Ohmic resistance is time-invariant (U = I·R). Diffusion limitation is not.

Apply a **constant** discharge (as high as the system allows) starting from mid SOC.
Record cell voltages every ~10–15 minutes for at least an hour. Track the gap between
the suspect cell and the highest cell in the same pack.

- Gap **constant** at constant current → ohmic. Benign-ish; ageing/spread.
- Gap **grows** while current stays flat or falls → **diffusion limitation**.
  This is a genuine degradation signature, not manufacturing spread.

Stop before the reference packs leave the flat region (~30–35 % SOC), or the
measurement is contaminated.

### 5. The recovery test (capacity deficit vs. reversible polarisation)
After the sustained load, remove current and watch the suspect cell's gap to its
neighbours.

**Do not conclude from the first 30–90 minutes.** A degraded cell can have a badly
stretched recovery time constant — it may still show a large residual gap after 90
minutes and yet recover *completely* once recharged. In the reference case the gap was
still 15 mV after 90 min of rest (vs. 2 mV pre-test), which looked like a permanent
capacity deficit — but after recharging into mid SOC it returned to 2 mV, fully healed.

Conclude a capacity deficit **only** if the gap persists after a recharge back into the
flat mid-SOC region. Otherwise it is slow-but-complete recovery, which is a milder
finding.

**Caveat:** in a parallel stack, packs exchange small balancing currents at rest.
This lifts *all 15 cells of a pack* roughly equally, so it barely affects the
*intra-pack* spread — which is the quantity of interest. It does affect the pack's
absolute level relative to other packs. Interpret intra-pack spread, not pack-vs-pack
voltage, during recovery.

## The passive-balancer trap

Pylontech (and most LFP BMS in this class) use **passive** balancing: they bleed charge
from high cells through resistors (~50 mA). They cannot add charge to a low cell.

A cell with elevated internal resistance is driven **above** its neighbours by I·R while
charging. It therefore hits the balancing threshold (~3.45 V) **first** — despite being
the *emptiest* cell. The balancer then bleeds the weakest cell, making it emptier still.

This is self-reinforcing: higher R → appears overcharged → gets bled → genuinely lower
SOC → larger droop under load → measured R rises further.

**Diagnostic tell:** during absorption, the suspect cell reads as the *lowest* in its
pack, while under charging current it reads as the *highest*. If you see that inversion,
the balancer is working against the cell.

Do not tell the user "the balancer will bring the weak cell back up." It cannot.

## Reporting

State clearly which findings are robust and which are inferred:
- **Robust:** relative comparison between identical packs; trend over time; directly
  measured cell voltages; the reversal signature; the sustained-load progression.
- **Uncertain:** absolute mΩ values from telemetry; extrapolation to remaining life.

Do not claim a module is "defective" if it shows no errors and meets capacity. The
defensible claim is comparative: *this module degrades measurably faster than identical
modules under identical conditions.*
