+++
title = "The Spec Is the Skill: AI-Assisted Development With Real QA"
date = 2026-04-10T10:00:00-05:00
draft = false
tags = ["ai-engineering", "data-engineering", "testing", "methodology", "retirement-planning"]
summary = "I built a retirement calculator with AI assistance, then hit a wall: analytically wrong output that looked correct. Here's how spec-first design and integration tests anchored to a verified reference caught six silent bugs — and what it generalizes to."
+++

There's a version of AI-assisted development that ends at the prompt. You describe what you want, the model generates code, you ship it. Fast, satisfying, occasionally correct.

The failure mode isn't syntax errors or type errors. Those surface immediately. The failure mode is analytically wrong output that looks plausible. The kind of bug you only find if you already know what the right answer is.

This is a post about how to handle that — using a retirement calculator I built as the case study.

---

## The project

A few months ago I needed to model my own retirement. The commercial tools either cost money or required an advisor. I knew the domain well enough — ACA subsidy cliffs, Roth conversion windows, RMD mechanics, widow's penalties — to have a precise model in my head. I built one instead.

[Lump Slam](https://www.pablooverton.com/lumpslam/) is a fully static, browser-only retirement calculator. It models retirement as four sequential phases with different rules for income, tax exposure, and withdrawal strategy:

- **COBRA** (first 18 months): No income constraints. Draw freely, set up Roth conversion headroom.
- **ACA** (until Medicare): MAGI must stay strictly below $84,600 for a couple or you lose all subsidies instantly — not gradually. The "$1 over the cliff" problem.
- **Medicare** (65+): Fixed healthcare costs, IRMAA surcharges above income thresholds. Prime Roth conversion window.
- **RMD** (73+): Required Minimum Distributions from pre-tax accounts whether you need them or not.

Given a profile — ages, account balances, spending goals, Social Security claim ages — the engine projects retirement year by year, picking the right withdrawal sequencing for each phase and firing Roth conversions when the math says to.

I used AI to implement most of it. And before shipping, I hit the problem I want to talk about.

---

## The methodology

Before AI writes code for anything with real consequences, I do two things that aren't optional.

### 1. Spec the domain, not the implementation

Not: "implement a tax bracket lookup that takes income and filing status." That tells AI what to build.

Instead: "given $185k taxable income for a married couple in 2024, the marginal rate should be 22%, not 24%, and certainly not 37%." That tells AI what *correct* looks like.

The distinction matters. AI is excellent at turning instructions into syntactically valid code. It's unreliable at independently knowing whether the output is financially or analytically correct. That's not a criticism — it's a structural property of the problem. Tax law, actuarial tables, ACA subsidy cliffs: these are domains where "plausible-sounding" and "correct" can diverge silently for months.

The spec is domain judgment written down. It's the non-automatable part.

### 2. Anchor integration tests to a verified reference case before writing any code

I had a reference scenario: a financial advisor video walking a couple through their retirement math on screen. Known inputs, known outputs. I encoded those expected values as failing integration tests *before* touching any code:

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

These tests define success. They can't be gamed — they're anchored to an external reference, not to the implementation. Then: implement until they pass.

---

## What the tests caught

Six bugs. All silent. None threw exceptions, none triggered type errors, none produced obviously nonsense output. All of them were analytically wrong in ways that compound.

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

The top bracket's ceiling is `Infinity`. Iterating backward means `taxableIncome <= Infinity` is always true, and the 37% bracket matches first. Every Roth conversion was grossed up at the top rate. No exception, no warning. Wrong answer, plausible magnitude.

### Spending capacity was $82k instead of $156k

Social Security income was missing. The portfolio generates $82k at 4% SWR. Social Security adds another $77k — income the portfolio doesn't need to generate. The tool was only counting the portfolio half.

### ACA MAGI spiked to $181k in years 3–5

Withdrawal sequencing drew from brokerage first (no MAGI impact), then fell through to uncapped pretax once the brokerage ran out after 1–2 ACA years. Every subsidy lost.

The fix: always enforce the MAGI ceiling in ACA season regardless of brokerage balance, cap pretax at `$84,600 - passiveMagi - 1` (the cliff is *exclusive* — exactly $84,600 disqualifies you), and use Roth built up during COBRA as overflow.

### Widow's coverage showed 33% instead of 98%

The calculation only counted survivor Social Security. It forgot the survivor inherits the full portfolio. `$41k SS / $126k desired spending = 33%`. The correct calculation: `($41k SS + $82k portfolio × 4%) / $126k = 98%`.

The tool was showing "cannot maintain lifestyle" for a scenario where the survivor has 98% of desired income. A frightening result that was simply wrong.

The remaining two bugs: a tax bracket off-by-one (`<` instead of `<=` at the exact ceiling value) and a desired spending comparison against total spending including charitable ($161k) instead of essential only ($126k), which made the surplus look like -$2k and prevented any Roth conversions from firing.

None of these were detectable without knowing what the numbers should be. They all produced output in a plausible range. The integration tests — specifically the tests anchored to the reference case — were the only mechanism that surfaced them.

---

## What this generalizes to

I write attribution models and data pipelines professionally. The same failure mode exists at every layer.

An ETL transform can be syntactically correct, type-safe, and unit-tested at the function level — and still produce analytically wrong output because of a join type, a null handling assumption, or a date truncation that didn't match the business definition. The only way to catch it is to have a verified reference case and fail explicitly when output diverges.

The methodology is the same everywhere:

1. **Spec the domain requirements** as assertions against known-correct values — not as implementation instructions
2. **Write integration tests anchored to a verified reference** before any implementation
3. **Use AI to implement fast**, evaluate against the tests
4. **Domain expertise evaluates** — that's the non-automatable loop

The integration tests aren't the hard part. Knowing what they should assert — knowing what "correct" looks like — is the hard part. That's domain knowledge. That's the work AI doesn't replace. That's the skill.

---

## The code

The full project is at [github.com/pablooverton/lumpslam](https://github.com/pablooverton/lumpslam). The domain engine lives in `src/domain/` with no React dependencies — same code runs in the browser and on the CLI. 63 tests covering tax math, SS actuarial adjustments, RMD tables, ACA cliff behavior, season classification, and the full reference case integration scenario.

```
npx tsx cli/run.ts profile.json scenarios
npx tsx cli/run.ts profile.json seasons 10
npx tsx cli/run.ts profile.json roth
```

The CLI is useful for scripting comparisons across parameter variations without loading the browser UI.
