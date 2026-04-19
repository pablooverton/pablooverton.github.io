+++
title = "The Bug You Can't Find With Tests: IRMAA's Two-Year Lookback"
date = 2026-04-18T10:00:00-04:00
draft = false
tags = ["typescript", "testing", "domain-modeling", "retirement", "lumpsum"]
summary = "How a timing rule invisible to the code — and to most retirement calculators — changes the math on Roth conversions at 65."
+++

Most bugs you find with tests are bugs you can specify. You know what the right answer should be, you write an assertion, and when the code disagrees you have a failure to chase. I wrote about that approach in ["The Spec Is the Skill"](/posts/lump-slam-spec-is-the-skill/) — anchor tests to a verified reference case, and they'll surface the silent, plausible-looking wrong answers where nothing throws and the numbers look reasonable.

There's a harder category. The rule that changes the answer isn't visible in the code at all. It doesn't show up in tax tables, it doesn't change the formulas, and none of the inputs look wrong. Your tests pass. Your output is off by thousands of dollars per year, and you'd never know unless you'd read a specific paragraph in Medicare's premium rules.

This is a post about one of those.

---

## The setup

I've been building [Lump Slam](https://www.pablooverton.com/lumpslam/), a browser-based retirement calculator. It models retirement as four sequential seasons — COBRA, ACA, Medicare, RMD — each with its own withdrawal strategy, MAGI constraint, and optimization lever. The big optimization during the Medicare season is the Roth conversion: move money from pretax (traditional IRA, 401k) into Roth, pay the tax now at a known rate, and never pay tax on that money again.

The cost of a Roth conversion in Medicare years isn't just the federal tax bill. It's also **IRMAA**: the Income-Related Monthly Adjustment Amount on Medicare Part B and Part D premiums. Above certain MAGI thresholds, your Medicare premiums step up — and IRMAA is a cliff at each tier, not a phase-in. One dollar over the floor charges the entire tier's surcharge for the whole year.

So the naive model is: for each Medicare year, compute MAGI, look up the IRMAA surcharge, add it to the cost of the conversion, done.

That's what every retirement calculator I've looked at does. And it's wrong.

---

## The rule that isn't in the code

Medicare premiums in year Y are not priced on your year-Y MAGI. They're priced on your MAGI from **two tax years earlier** — year Y minus 2. From [SSA's own page on the topic](https://www.ssa.gov/benefits/medicare/medicare-premiums.html): *"To determine your 2026 IRMAA, we'll use your tax return from 2024."*

This is the two-year lookback. It exists because Medicare needs a finalized tax return to price next year's premiums, and the IRS doesn't finalize a year until it's already over.

Practically, this means:

- A Roth conversion at age 65 shows up in premiums at age 67
- There are two Medicare years at the start of retirement where IRMAA is priced on your working-income MAGI — which is likely already above every tier
- A retiree who drops their MAGI at 65 keeps paying high-income premiums until 67

None of this is visible in the tax formulas. It's a timing rule. The marginal rate math doesn't change. The MAGI calculation doesn't change. What changes is **which year's MAGI the surcharge formula reads**.

If you model it in the same year (MAGI-now → surcharge-now), you'll systematically overstate the immediate cost of aggressive early conversions and mis-time when that cost actually hits the cash flow.

---

## Why tests don't catch this

The failure mode I wrote about in the previous post was silent analytical wrong answers — bugs that don't throw exceptions but produce plausible, off-by-a-factor output. The fix there was anchoring integration tests to a reference scenario with known-correct numbers. You look at the failing assertion, you trace the bug.

That methodology assumes you have something to assert against. Here I didn't.

My reference scenario came from a financial advisor walking a couple through their retirement math on video. The advisor discussed IRMAA generally, but didn't show the lookback mechanic — he stated surcharges as a year-by-year cost, not as a two-year-displaced cost. If I'd written an integration test against his stated numbers without knowing about the lookback, the test would have *confirmed* the same-year model. The code would match the spec. The spec itself would be wrong.

This is the limit of reference-anchored testing. Tests validate the code against your belief about correctness. If your belief is wrong because the source material was imprecise, the tests enforce the wrong belief with the full confidence of a green CI run.

The only way out is reading the underlying source — in this case, the SSA's own rules — not just the secondary material that interprets them.

---

## The fix

The engine tracks MAGI year by year. Adding the lookback was two changes.

First, retain a history:

```typescript
// IRMAA surcharges are based on MAGI from 2 years prior (the "lookback MAGI").
// Tracking per-year MAGI history lets us price this correctly: a big Roth
// conversion in year N triggers an IRMAA increase in year N+2, not year N.
const magiHistory: number[] = [];
const getLookbackMagi = (currentMagi: number): number =>
  magiHistory.length >= 2 ? magiHistory[magiHistory.length - 2] : currentMagi;
```

The fallback to `currentMagi` for the first two years is deliberate. Before retirement, the household has working-income MAGI that's probably already saturating the tiers — using current MAGI in years 1–2 of Medicare is a defensible approximation and avoids the calculator showing a suspicious zero-IRMAA transition.

Second, read the lookback value when computing the surcharge:

```typescript
const lookbackMagi = getLookbackMagi(magi);
const irmaaSurcharge = calculateIrmaaSurcharge(lookbackMagi, filingStatus);
```

And then `magiHistory.push(magi)` at the end of each year's accounting.

The change is five lines. The effect on retirement projections is large — particularly for anyone running the engine past age 65 with any conversion activity.

---

## What it reveals

Running the same profile before and after the fix, the most visible change is at age 65. Before: the engine showed a conversion at 65 immediately priced with the full IRMAA surcharge, so the tool was treating the first post-COBRA year as expensive for conversions.

After: the cost at 65 drops, because the premium being charged that year is based on MAGI from 63 — which for most profiles is still working-income years with employer insurance, where IRMAA doesn't apply anyway. The surcharge shows up at 67, where it should, reflecting the 65-year conversion.

The practical planning consequence: **there is a window at 65–66 to do aggressive conversions before IRMAA has any bite**. If you're also below the tier by 67 (because the conversion reduced your pretax balance, reducing RMDs and MAGI), you may never pay the surcharge at all. The conventional advice to "be careful about conversions at 65 because of IRMAA" is imprecise — you should be careful about conversions at *63*, which is too early to do conversions for different reasons (still working, high marginal rate).

The engine now flags this explicitly. The `classifyIrmaaTier` helper exposes tier index, magi ceiling, and room-to-next-tier so the UI can show "you have $6,000 of conversion room before the next IRMAA jump" — using the correct lookback year.

---

## Generalizing: domain rules vs. code rules

I write attribution pipelines and data platforms professionally. The parallel in that domain is the **business rule that isn't in the data**. A pipeline can be correctly joining tables, correctly aggregating metrics, correctly handling nulls — and still produce the wrong number because the business defines a "qualified lead" differently than the source system does, or because finance uses a 4-5-4 retail calendar rather than standard months, or because an attribution window is counted from touchpoint rather than conversion.

None of those are code bugs. They're not visible in the schemas, not derivable from the column names, not testable against the data itself. They live in the sales ops' head, or in a policy doc from 2019, or in the CFO's expectations. Your tests will pass and the number will be wrong.

The defense is the same as for IRMAA: **read the primary source**. Talk to the person who defined the business rule, not the person who built the last pipeline. Read the IRS publication, not the financial planning book. Tests validate code against your model of the world. If your model is incomplete, the tests encode the incompleteness.

AI is remarkably good at generating code from a spec. It cannot check whether the spec is missing a rule. That's still our job.

---

*The IRMAA lookback change is in Lump Slam as of April 2026, along with 12 tests locking the lookback contract between the engine and the simulation runner. Full source at [github.com/pablooverton/lumpslam](https://github.com/pablooverton/lumpslam). If you're modeling your own retirement and want to see what a corrected lookback does to your conversion plan, the tool is free and runs in-browser.*
