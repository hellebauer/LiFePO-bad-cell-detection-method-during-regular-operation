# Finding a bad cell in a parallel LiFePO4 stack

A method for diagnosing single-cell degradation in parallel LiFePO4 battery stacks
(Pylontech US2000/US3000/US5000 and similar) using nothing but the telemetry the BMS
already gives you.

This exists because the information isn't out there. Pylontech doesn't publish a
diagnostic procedure. The forums have folklore — "your cells are drifting, charge to
100% more often" — but no method for working out *what is actually wrong* and
distinguishing a genuinely failing cell from normal manufacturing spread.

It was worked out on a real stack: 4× US3000C on a Victron MultiPlus-II 3000,
~475 cycles, using ~11 months of logged data plus targeted tests. All numbers below
are from that stack.

---

## Contents

- [The thing that makes this hard](#the-thing-that-makes-this-hard)
- [The core insight](#the-core-insight)
- [When your measurement is valid](#when-your-measurement-is-valid)
- [Getting the data](#getting-the-data)
- [The tests](#the-tests)
  - [1. Rule out the boring explanations first](#1-rule-out-the-boring-explanations-first)
  - [2. Estimate per-pack resistance from your logs](#2-estimate-per-pack-resistance-from-your-logs)
  - [3. The current-reversal test](#3-the-current-reversal-test)
  - [4. The sustained-load test — the one that actually matters](#4-the-sustained-load-test--the-one-that-actually-matters)
  - [5. The recovery test](#5-the-recovery-test)
- [The passive balancer is making it worse](#the-passive-balancer-is-making-it-worse)
- [What you can honestly conclude](#what-you-can-honestly-conclude)
- [Scope and limitations](#scope-and-limitations--read-this)
- [Using this with Claude](#using-this-with-claude)

---

## The thing that makes this hard

LiFePO4 has a famously flat discharge curve. Between roughly 20% and 90% state of
charge, cell voltage barely moves. That flatness is the whole problem — and, once you
understand it, the whole solution.

**At mid SOC, real differences between cells are invisible.** Two cells whose charge
levels differ by several percent will read within a millivolt of each other. The curve
is too flat to show it.

**At the ends, tiny differences are amplified.** Near full or empty, the curve is
steep. A cell that's barely different from its neighbours suddenly reads 30 mV off.
This is why people panic about "cell imbalance" at 100% SOC when nothing is wrong.

**Under current, resistance adds a spread that has nothing to do with charge level.**
A cell with slightly higher internal resistance will droop under load and rise under
charge — purely from `U = I·R`, regardless of how full it is.

Put those together and you get the trap: **the same cell can look like the worst in the
pack, the best in the pack, or perfectly ordinary, depending entirely on when you
happen to look.**

I went through this personally. My first read of the data cleared the suspect cell
completely. My second blamed the cabling. My third blamed a different mechanism. Only
by testing across *states* did the real answer emerge. Any diagnosis from one screenshot
is worthless.

---

## The core insight

There are three distinct reasons a cell can deviate, and **they have different
signatures across current direction:**

| | Discharging | At rest (0 A) | Charging |
|---|---|---|---|
| **High internal resistance** | reads **low** | reads **normal** | reads **high** |
| **Low charge / lost capacity** | reads low | reads **low** | reads low |
| **Diffusion limitation** | drops **further and further** over time | recovers slowly and incompletely | rises progressively |

The rest reading is the discriminator. A cell that is *perfectly normal at zero current*
but deviates under load does not have a charge problem. It has a **resistance** problem.

This is the single most useful thing in this document. It costs nothing to test: take
one reading under load, one under charge, one at rest, all at mid SOC.

---

## When your measurement is valid

Most of the effort here is not measuring — it's knowing when *not* to trust what you
measured.

| Situation | Use it for | Don't use it for |
|---|---|---|
| Mid SOC (30–80%), high current | **Resistance** | — |
| Mid SOC, zero current, settled | **True charge state** | — |
| SOC above ~90% | *nothing reliable* | Curve is steep; everything is contaminated |
| Balancer running (cells >3.45 V) | *nothing* | The balancer distorts the reading of whichever cell it's bleeding |
| Within ~2 min of a load change | *nothing* | Cells haven't settled yet |
| Low current (<5 A per pack) | Charge state | Resistance — the signal is down at the BMS's 1 mV resolution |

The last row bit me repeatedly. At 4 A per pack I was seeing 4–6 mV spreads and
computing resistance values that scattered between 0.7 and 1.5 mΩ. The numbers were
noise. At 13 A the same measurement was clean.

**Rule of thumb: measure resistance under as much current as your system will give you,
in the middle of the SOC range, and never while the balancer is active.**

---

## Getting the data

You need two different things, and one tool won't give you both.

### The CSV gives you pack-level data only

MultiSIBControl (see [multisibcontrol.net](https://www.multisibcontrol.net) — setup is
documented there, not repeated here) can log your stack and export CSV. That export
contains, per pack: voltage, current, temperature, SOC.

That's enough for **Test 2** (per-pack resistance) and for tracking the long-term trend.
It is *not* enough for anything else, because —

### The CSV does **not** contain cell voltages

Individual cell voltages appear only in the live monitoring view, not in the export.
And cell voltages are where the entire diagnosis lives.

So: **you have to take screenshots.** There is no way around it.

### What to screenshot, and when

Not one screenshot per test. A *series*.

| Test | Screenshot cadence |
|---|---|
| Current-reversal (Test 3) | One under discharge, one at rest, one under charge |
| Sustained load (Test 4) | **Every 10–15 minutes for the whole hour** |
| Recovery (Test 5) | At 1, 10, 30, 60, 90 min after load ends — then again after a recharge |

The sustained-load test in particular is worthless with two data points. The whole finding
is the *shape* of the curve, and you only see a shape if you sample it.

### Record the context with every screenshot

A cell voltage on its own means nothing. The same 3.24 V reading is unremarkable at rest
and alarming under 13 A. Note, for each capture:

- **Current** (per pack — not just stack total)
- **SOC** of the reference packs (not the suspect pack; its SOC reading is skewed)
- **How long** the current has been flowing at that level
- **Cell temperatures** (warmth lowers internal resistance and will mask the effect)

If your screenshots include the table with pack current, SOC and temperatures alongside
the cell voltages — as MultiSIBControl's monitoring view does — that's all captured for
free. Which is the main reason to use it.

---

## The tests

### 1. Rule out the boring explanations first

Before you go hunting for a bad cell, check two things.

**Is the stack ever actually reaching full?** If your system rarely charges all packs
to a true 100%, the coulomb counters never reset, and packs drift apart in reported SOC
for entirely innocent reasons. In my logs, only 2.5% of the time were all four packs
simultaneously above 99%. That alone explains a lot of apparent "drift."

**Is it just a bad terminal?** A high-resistance cable or connector is symmetric — it
restricts current equally in both directions — and it produces heat (`P = I²R`). Check
for load-correlated warming, but check it *properly*: thermal time constants are
minutes to tens of minutes, so correlating instantaneous temperature against
instantaneous current will find nothing even when a hotspot exists. Smooth the heat
term over a realistic thermal window, or look at whether the pack warms *relative to its
neighbours* across a sustained high-load block.

In my case this test came back negative — no load-correlated heating — which is what
eventually pushed the diagnosis from the wiring into the cells.

### 2. Estimate per-pack resistance from your logs

If your monitoring tool exports CSV (MultiSIBControl does), you can estimate each pack's
effective resistance without any special equipment.

Within a narrow SOC band, open-circuit voltage is roughly constant. So within that band,
plot each pack's terminal voltage against its own current: the slope is its resistance.

```python
slopes = []
for lo in range(30, 100, 5):
    m = (soc >= lo) & (soc < lo + 5)
    if m.sum() > 200 and (I[m].max() - I[m].min()) > 15:  # need real current spread
        slope, _ = np.polyfit(I[m], V[m], 1)
        slopes.append(slope)
R_eff = np.average(slopes)   # ohms
```

**Do not trust the absolute value.** BMS voltage resolution is coarse, and there's
residual SOC movement inside each band. My absolute figures ranged from 60 to 104 mΩ
across datasets — the spread is methodological, not physical.

**Do trust the relative comparison and the trend.** Four identical packs, same
measurement, same conditions: any one of them standing out is real.

Here's what that produced on my stack over eleven months:

| | Aug 2025 | May 2026 | Jul (early) | Jul (late) |
|---|---|---|---|---|
| **P4 excess resistance vs. the other three** | **−4%** | **+12%** | **+24%** | **+27%** |

Pack 4 started out *better* than average. That's the part that matters. A systematic
error in the method would have shown P4 as elevated from day one. Instead it crossed
from negative to strongly positive, monotonically. That's a real, progressing change.

### 3. The current-reversal test

This is the cheapest and most decisive test in the whole method.

Take three readings of the cell voltages, all at mid SOC:
- Under meaningful discharge (>10 A per pack)
- At rest, five minutes after the load stops
- Under meaningful charge (>10 A per pack)

A cell with elevated resistance will **flip sign**. On my stack, cell 13 of pack 4:

| State | Cell 13 | Its position in the pack |
|---|---|---|
| Discharging, 13 A | 3.245 V | **lowest** |
| At rest | 3.321 V | indistinguishable from the rest |
| Charging, 9–21 A | 3.349 V | **highest** |

Same cell. Reverses completely with current direction. Returns to the pack average when
no current flows. That is a textbook resistance signature and it cannot be anything else.

Meanwhile the other three packs sat at 1–2 mV spread in every state.

### 4. The sustained-load test — the one that actually matters

Everything above finds a resistance difference. This test tells you whether that
difference is *benign spread* or *real degradation*, and it's the test almost nobody runs.

**The principle:** ohmic resistance is time-invariant. `U = I·R` doesn't care how long
you've been pulling current. If a cell's voltage gap is caused by plain resistance, that
gap will hold steady at constant current, indefinitely.

Diffusion limitation is different. If a cell's internal ion transport can't keep up, the
concentration gradient at the electrode builds over minutes, and the voltage droop
**grows** — even at constant current.

**How to run it:** start from mid SOC (~60%), apply as much constant discharge as your
system allows, and record cell voltages every 10–15 minutes for at least an hour. Track
the gap between your suspect cell and the highest cell in the same pack.

Here's what happened on my stack, at roughly constant current:

| Elapsed | Current through P4 | Cell 13 gap | Implied resistance |
|---|---|---|---|
| 2 min | 13.5 A | 12 mV | 0.9 mΩ |
| 17 min | 13.2 A | 19 mV | 1.4 mΩ |
| 23 min | 13.0 A | 22 mV | 1.7 mΩ |
| 37 min | 12.2 A | 29 mV | 2.4 mΩ |
| 42 min | 11.9 A | 34 mV | 2.9 mΩ |
| 55 min | 11.6 A | 40 mV | 3.4 mΩ |
| 61 min | 11.6 A | 44 mV | 3.8 mΩ |
| 66 min | 11.1 A | 46 mV | 4.1 mΩ |

Read that carefully. **The current went *down* by 18%. The voltage gap went *up* by
283%.** Ohm's law says the gap should have shrunk. It nearly quadrupled instead.

That is diffusion limitation. It is a genuine ageing mechanism, and you will not see it
in any snapshot — at two minutes in, this cell looked like a mildly elevated resistance,
nothing alarming. An hour in, it was unmistakable.

**Important honesty:** the other three packs also drifted, just far less. P1 went from
1 to 3 mV, P2 from 2 to 5 mV, P3 from 2 to 6 mV over the same hour. A small progressive
spread under sustained load is *normal* — it happens in healthy cells too. What matters
is the magnitude. Pack 4's suspect cell moved 34 mV where its healthy counterparts moved
2–4 mV. It's a factor of 8–16, not a binary healthy/broken distinction. Say it that way.

**Stop the test** before your reference packs drop below ~30–35% SOC, or you leave the
flat region and the comparison stops being clean.

### 5. The recovery test

After the load, kill the current and watch. Does the suspect cell come back?

| Time after load ends | Cell 13 gap |
|---|---|
| under load | 46 mV |
| 1 min | 35 mV |
| 10 min | 22 mV |
| 18 min | 21 mV |
| 50 min | 17 mV |
| ~90 min | 15 mV |

Before the test, at rest, this pack's spread was **2 mV**. Ninety minutes after, it was
still at 15 mV and barely moving.

At that point I concluded the cell had a permanent capacity deficit. **I was wrong.**
After recharging back into the mid-SOC range, the spread had fully collapsed: at rest at
61–63% SOC, the pack was back to **2 mV** — identical to before the test and to the
healthy packs.

So the recovery is *complete*, just extremely slow. The cell has a badly stretched
recovery time constant, but it does not permanently lose charge. That is a meaningfully
milder finding than it first appeared, and it belongs in the record.

**The lesson:** a recovery test that stops after 90 minutes can look like a capacity
deficit when it is really just slow recovery. Give it a full recharge cycle before
concluding anything.

**One caveat you must account for.** In a parallel stack, once the load stops, the packs
quietly exchange small balancing currents (I measured ~0.3–0.5 A) as their resting
voltages equalise. This *does* muddy a naive recovery measurement.

But it doesn't invalidate the important number. That balancing current flows through all
15 cells of a pack in series, lifting them roughly equally. It shifts the pack's
*absolute* level relative to the other packs — but it barely touches the **spread
between cells inside that pack**, which is what you're actually measuring. Track
intra-pack spread, not pack-versus-pack voltage, and the test survives.

(Credit where due: I missed this initially and had to be pulled up on it.)

---

## The passive balancer is making it worse

This is the part that surprised me most, and it has real consequences.

Pylontech — and essentially every LFP BMS in this class — uses **passive** balancing.
It can bleed charge off a high cell through a resistor (about 50 mA). It **cannot** push
charge into a low cell. There is no mechanism for that.

Now think about what a high-resistance cell does while charging. `U = I·R` drives its
terminal voltage **above** its neighbours. So it reaches the balancing threshold
(~3.45 V) **first** — even though it is, in reality, the *emptiest* cell in the pack.

The BMS sees a high voltage and does the only thing it knows: **it bleeds the weakest
cell in the pack.**

That closes a loop:

```
higher resistance
    → reads artificially high while charging
        → balancer bleeds it
            → genuinely loses charge
                → droops harder under load
                    → measured resistance rises
                        → (repeat, every cycle)
```

This mechanism explains the eleven-month acceleration in my data better than ageing
alone does. And it is *not* something the user causes or can switch off. It's how the
product behaves.

**The diagnostic tell:** during absorption, the suspect cell reads as the **lowest** in
its pack — while under charging current, minutes earlier, it read as the **highest**. If
you see that inversion, the balancer has been working against that cell.

On my stack, during absorption at 99% SOC, cell 13 read 3.485 V against a pack average
around 3.50 V. Lowest in the pack. It had already been bled down.

**Practical consequence:** do not expect the balancer to fix a resistance-degraded cell.
It will do the opposite. Lowering the charge current limit (via DVCC or equivalent)
reduces the `I·R` distortion and makes the balancer act on something closer to the cell's
true state — that's a genuine mitigation, though it treats the symptom.

---

## What you can honestly conclude

Be careful about what you claim.

**Well-founded:**
- Comparing identical packs against each other, under identical conditions
- The trend over time — this is the strongest evidence you'll have
- Directly measured cell voltages
- The current-reversal signature
- The sustained-load progression

**Not well-founded:**
- Absolute resistance values from BMS telemetry — treat as order-of-magnitude only
- Any extrapolation to remaining lifetime
- Calling a module "defective" when it throws no errors and delivers rated capacity

The claim that actually holds up is comparative: *this module is degrading measurably
faster than identical modules under identical conditions, and the mechanism is
identified.* That's defensible. "It's broken" is not, if it's still working.

In my case the stack still functions fine. No errors, no protection events, and because
it never runs down into the low SOC region in normal use, the capacity deficit isn't even
noticeable day to day. The problem is the *trajectory*, not the current state.

---

## Scope and limitations — read this

This method was derived from **one stack**. Four Pylontech US3000C modules, one inverter,
one household load profile, eleven months of data. That is enough to establish the method
but not enough to make the specific numbers authoritative.

What I'm confident generalises:
- The three-signature framework (resistance / capacity / diffusion)
- The current-reversal test
- The requirement to test under *sustained* load, not snapshots
- The validity windows (flat region, high current, balancer off)
- The passive-balancer trap — this follows from how passive balancing works and is
  confirmed by the absorption-phase inversion, but I have **not** verified it against
  Pylontech's actual firmware. It is mechanistically sound inference, not a datasheet fact.

What I'd want more cases before trusting:
- Specific mΩ thresholds for "concerning"
- How fast diffusion limitation typically progresses
- Whether the same signatures appear in other BMS families

If you run this on your own stack, the numbers you get will differ. The *shapes* should
not.

---

## Using this with Claude

`SKILL.md` in this repo is written for Claude, not for you. It encodes the decision tree,
the validity windows and the traps — so the model doesn't have to rediscover them by
getting it wrong six times first.

(Which, for the record, is exactly how this method got written.)

**To use it:**

1. Download `SKILL.md` from this repo.
2. Start a conversation with Claude and attach it, along with your data.
3. Say something like: *"Use the attached skill to analyse my battery data."*

Then feed it what you've collected:

- **Your CSV export** — Claude can run the resistance regression on it directly and give
  you the per-pack comparison and the trend.
- **Your screenshots** — paste them in as you take them. Claude reads the cell voltage
  tables directly from the image.

The screenshots are the part people skip, and it's the part that matters. Keep sending
them throughout a test. Tell Claude the current, the SOC and how long the load has been
running — it needs that context to interpret what it's looking at, and it will otherwise
guess (badly).

If you have historical logs from months back, include those too. **The trend over time is
the strongest evidence you will produce.** A single day of data can show you *that*
something is off; only the trend shows you that it's getting worse.

---

*Data from a 4× Pylontech US3000C stack on a Victron MultiPlus-II 3000, ~475 cycles.
Serial numbers and network details removed. Timestamps relative.*
