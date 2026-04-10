+++
title = "The Spec Is the Skill: AI-Assisted Development With Real QA"
date = 2026-04-10T10:00:00-05:00
draft = false
tags = ["ai-engineering", "data-engineering", "testing", "methodology", "retirement-planning"]
summary = "I built a retirement calculator with AI assistance, then hit a wall: analytically wrong output that looked correct. Here's how spec-first design and integration tests anchored to a verified reference caught six silent bugs — and what it generalizes to."
+++

Most AI coding failures aren't the ones you'd expect. The code usually runs. Types check out, syntax is fine, maybe unit tests pass. The failure is analytically wrong output that looks plausible — the kind of bug you'd only notice if you already knew what the right answer was supposed to be.

I built a retirement calculator using AI for most of the implementation, and hit this exactly. Six bugs. All silent. None threw exceptions or failed any test I'd written before I understood the problem.

---

## What I built

I needed to model my own retirement — not as an exercise, but as a real plan. Four kids, one more on the way, and I wanted a tool I could actually hand to my wife and say "here's what this looks like under different scenarios." Commercial tools either cost money, required an advisor, or gave me suspiciously reassuring answers.

I knew the domain well enough — ACA subsidy cliffs, Roth conversion windows, RMD mechanics, widow's penalties — to know when a tool was lying to me. So I built [Lump Slam](https://www.pablooverton.com/lumpslam/) instead.

It's fully static, browser-only. It models retirement as four sequential phases:

- **COBRA** (first 18 months): No income constraints. Draw freely, build Roth conversion headroom.
- **ACA** (until Medicare): MAGI must stay strictly below $84,600 for a couple or you lose all subsidies instantly — not gradually. The "$1 over the cliff" problem.
- **Medicare** (65+): Fixed healthcare costs, IRMAA surcharges above certain income thresholds. Prime Roth conversion window.
- **RMD** (73+): Required Minimum Distributions from pre-tax accounts whether you need them or not.

Given a profile — ages, balances, spending goals, Social Security claim ages — the engine projects retirement year by year, handles withdrawal sequencing for each phase, and fires Roth conversions when the math says to.

I used AI to implement most of it. And before shipping, I hit the problem.

---

## The methodology

Before asking AI to write code for anything with real consequences, there are two things I won't skip.

**Spec the domain, not the implementation.** Not: "implement a tax bracket lookup that takes income and filing status." That tells AI what to build. Instead: "given $185k taxable income for a married couple in 2024, the marginal rate should be 22%, not 24%, and certainly not 37%." That tells AI what *correct* looks like.

AI is very good at turning instructions into syntactically valid code. It's less reliable at independently knowing whether financial or analytical output is actually correct — not because it's broken, but because that's a structural property of how it works. Tax law, actuarial tables, ACA cliff logic: these are domains where "plausible-sounding" and "correct" diverge silently. Encoding what correct looks like is the non-automatable step. That's the spec.

**Anchor integration tests to a verified reference before writing any code.** I had a reference scenario: a financial advisor video walking a couple through their retirement math on screen. Known inputs, known outputs. I encoded those expected values as failing integration tests before touching any code:

```typescript
it('spending capacity includes SS income and is ~$156k', () => {
  expect(result.spendingCapacity).toBeGreaterThanOrEqual(140_000);
  expect(result.spendingCapacity).toBeLessThanOrEqual(165_000);
});

it('ACA years stay below $84,600 MAGI cliff', () => {
  for (const year of acaYears) {
    expect(year.magi).toBeLessThan(84_600);
  }
});

it("widow's coverage includes portfolio withdrawal capacity", () => {
  expect(result.survivorCoveragePercent).toBeGreaterThan(0.9);
});
```

Tests anchored to an external reference can't be gamed — they're not verifying the implementation, they're verifying correctness. Then: implement until they pass.

---

## Six bugs the tests caught

All silent. None threw exceptions, none triggered type errors, none produced obviously wrong output. They all landed in plausible ranges.

### getMarginalRate always returned 37%

```typescript
// Wrong — iterates backward; Infinity ceiling matches every income first
for (const bracket of [...brackets].reverse()) {
  if (taxableIncome <= ceiling) return bracket.rate;
}

// Fixed — forward iteration, first bracket whose ceiling exceeds income
for (const bracket of brackets) {
  if (taxableIncome <= ceiling) return bracket.rate;
}
```

The top bracket's ceiling is `Infinity`. Iterating backward means `taxableIncome <= Infinity` is always true, so the 37% bracket matched first every time. Every Roth conversion was grossed up at the top marginal rate. No warning. Wrong answer, plausible magnitude.

### Spending capacity was $82k instead of $156k

Social Security was missing. The portfolio generates ~$82k at 4% SWR. Social Security adds another $77k — income the portfolio doesn't need to generate. The tool was counting only the portfolio half.

### ACA MAGI spiked to $181k in years 3–5

Withdrawal sequencing drew from brokerage first (no MAGI impact), then fell through to uncapped pretax once the brokerage ran dry after 1–2 ACA years. Every subsidy gone.

Fix: always enforce the MAGI ceiling in ACA years regardless of brokerage balance. Cap pretax at `$84,600 - passiveMagi - 1` — the cliff is exclusive, so exactly $84,600 disqualifies you. Use Roth built up during COBRA as overflow.

### Widow's coverage showed 33% instead of 98%

The calculation counted only survivor Social Security. It forgot the survivor inherits the portfolio. `$41k SS / $126k desired spending = 33%`. Correct: `($41k SS + $82k portfolio × 4%) / $126k = 98%`.

The tool was showing "cannot maintain lifestyle" for a survivor who had 98% of desired income covered. A frightening result that was simply wrong.

The remaining two: a tax bracket off-by-one (`<` instead of `<=` at the ceiling value) and a desired spending comparison against total spending including charitable ($161k) instead of essential only ($126k), which made the surplus show as -$2k and blocked all Roth conversions from firing.

---

## What I take from this

I write data pipelines professionally. This failure mode lives there too. An ETL transform can be type-safe, unit-tested at the function level, and still produce analytically wrong output — from a join type, a null handling assumption, a date truncation that didn't match the business definition. You only catch it if you have a verified reference case and fail explicitly when output diverges.

The tests are the easy part once you have the reference. Knowing what they should assert — what "correct" actually looks like for this domain — that's the harder work. It's also the part AI can't do for you.

---

## The code

Full project at [github.com/pablooverton/lumpslam](https://github.com/pablooverton/lumpslam). Domain engine lives in `src/domain/` with no React dependencies — same code runs in the browser and in the CLI. 63 tests covering tax math, SS actuarial adjustments, RMD tables, ACA cliff behavior, season classification, and the full reference integration scenario.

```
npx tsx cli/run.ts profile.json scenarios
npx tsx cli/run.ts profile.json seasons 10
npx tsx cli/run.ts profile.json roth
```
