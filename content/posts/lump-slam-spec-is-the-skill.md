+++
title = "The Spec Is the Skill: AI-Assisted Development With Real QA"
date = 2026-04-10T10:00:00-05:00
draft = false
tags = ["ai-engineering", "data-engineering", "testing", "methodology", "retirement-planning"]
summary = "Six silent bugs, none of them obvious. Here's what actually caught them when building a retirement calculator with AI."
+++

Most AI coding failures aren't syntax errors. The failure is analytically wrong output that looks plausible, the kind of bug you only catch if you already know the right answer.

I built a retirement calculator using AI for most of the implementation and ran into this exactly. Six silent bugs. Here's what fixed it.

---

## What I built

[Lump Slam](https://www.pablooverton.com/lumpslam/) models retirement as four sequential phases (COBRA, ACA, Medicare, RMD) and handles withdrawal sequencing, Roth conversion timing, and Social Security optimization. I built it because commercial tools either cost money, required an advisor, or gave me suspiciously reassuring answers. I have four kids and one more on the way. I wanted something I could actually verify.

---

## Tips if you're building something similar

**Spec the domain before writing any code.** Don't tell AI what to build. Tell it what correct looks like. Not: "implement a tax bracket lookup that takes income and filing status." Instead: "given $185k taxable income for a married couple, the marginal rate should be 22%, not 37%." AI is good at generating valid code. It's much less reliable at independently knowing whether financial or analytical output is actually right.

**Find a verified reference case.** I used a financial advisor video with known inputs and outputs. Encode those expected values before touching any implementation. If your tests are anchored to an external reference, they can't be gamed: they verify correctness, not the implementation.

**Don't trust plausible output.** Every bug I found produced numbers in a reasonable range. None threw exceptions. None triggered type errors. They all just quietly returned wrong answers.

---

## The bugs

Six of them, all silent:

- **getMarginalRate always returned 37%.** The bracket lookup iterated backward, so `Infinity` matched every income first.
- **Spending capacity was $82k instead of $156k.** Social Security wasn't counted. The portfolio covers $82k at 4% SWR; SS adds another $77k on top.
- **ACA MAGI spiked to $181k in years 3-5.** Withdrawal sequencing drew from brokerage first, then fell through to uncapped pretax after the brokerage ran dry. Every subsidy gone.
- **Widow's coverage showed 33% instead of 98%.** The calculation forgot the survivor inherits the portfolio. The tool showed "cannot maintain lifestyle" for a scenario where the survivor had 98% of income covered.
- A tax bracket off-by-one (`<` instead of `<=` at the ceiling value).
- A spending comparison against the wrong total, which made the surplus show negative and blocked all Roth conversions from firing.

None of these were detectable without knowing what the output should be. The reference case was the only thing that caught them.

---

## The code

Full project at [github.com/pablooverton/lumpslam](https://github.com/pablooverton/lumpslam). Domain engine in `src/domain/` with no framework dependencies, same code runs in the browser and CLI.
