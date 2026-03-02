+++
title = "Building an AI-Powered ICP Signal Monitor: Encoding Sales Judgment into Code"
date = 2026-03-02T10:00:00-05:00
draft = false
tags = ["ai-engineering", "llm", "gtm", "signal-intelligence", "claude", "python"]
summary = "How I built a system that uses Claude to score companies against an ideal customer profile — and why the interesting part isn't the AI."
+++

# Building an AI-Powered ICP Signal Monitor

If you've spent time in B2B sales or GTM data, you know that ICP fit scoring is mostly vibes. Every sales team has a signal hierarchy in their head — funding rounds matter more than press releases, executive hires signal new mandates, M&A creates integration needs. But that judgment lives in Slack threads and pipeline reviews, not in systems.

I wanted to see what happens when you encode that judgment explicitly and let an LLM apply it. The result is [icp-signal-monitor](https://github.com/pablooverton/icp-signal-monitor): a Python tool that fetches public signals for a list of target companies, scores them against a configurable ICP using Claude, and produces a weekly digest.

The interesting part isn't that it uses AI. It's *how* it uses AI — and what it doesn't ask AI to do.

## Signal hierarchy as domain expertise

Most AI projects fail not because the model is wrong, but because nobody encoded what "right" looks like. In sales, the signal hierarchy is the domain expertise. Not all signals are created equal:

| Signal Type | Weight | Why |
|-------------|--------|-----|
| Funding round (Series B+) | 0.9 | Budget event. New buying centers form. |
| Executive hire (CTO/CDO/VP Data) | 0.85 | New leader = new mandate, often new spend. |
| M&A activity | 0.85 | Integration creates data and tooling needs. |
| Product launch / expansion | 0.6 | Growth phase indicator. |
| Tech change (migration, cloud) | 0.6 | Active build cycle. |
| General news | 0.3 | Context only. |

A funding round at weight 0.9 and general news at 0.3 encodes a real insight: budget events are leading indicators of buying intent. Job postings and press coverage are lagging and noisy. Every experienced sales leader knows this, but it's rarely formalized.

This hierarchy lives in `icp_config.yaml` — a plain YAML file that anyone on a GTM team could read and adjust:

```yaml
trigger_signals:
  - type: "funding"
    weight: 0.9
    description: "Series B+ or growth equity; signals new budget"
  - type: "executive_hire"
    weight: 0.85
    description: "CTO, CDO, VP Data hire; new leader = new mandate"
  - type: "merger_acquisition"
    weight: 0.85
    description: "M&A activity; integration creates data needs"
```

The config also defines the ICP itself — target industries, company size range, geography, positive and negative keywords. Everything that matters is explicit and version-controlled. When the team's understanding evolves, you update a config file, not a prompt buried in application code.

## Architecture: prompt-as-code

The system is deliberately simple: **config → fetch → score → digest.**

1. Load the ICP definition and target companies from YAML
2. Fetch signals from public sources (news, funding data) for each company
3. Deduplicate and sort signals by reliability weight
4. Send each company's signals to Claude with a versioned scoring rubric
5. Render a markdown digest ranked by score

The interesting architectural decision is externalizing the scoring rubric into a separate, versioned prompt file (`prompts/scorer_v1.md`). This file defines a 0–100 scoring scale with explicit thresholds:

- **80–100 (Strong fit):** Multiple high-reliability signals aligned with ICP. Active transformation or budget event. Clear buying intent.
- **60–79 (Moderate fit):** Some aligned signals but missing key indicators. Growth phase but no confirmed budget event.
- **40–59 (Weak fit):** Limited signal alignment. General news only.
- **0–39 (Poor fit):** No meaningful signals or negative indicators.

Claude isn't deciding the rubric. It's applying one. The model reads the signals, maps them against the weights and thresholds I defined, and produces structured JSON with a score, reasoning, top signals, risk factors, and a recommendation (monitor, engage, or deprioritize).

This is the pattern I keep coming back to in AI engineering: the value isn't in the model's judgment — it's in making *your* judgment legible to the model. The prompt is code. The config is the business logic. The LLM is the execution engine.

## What Claude actually produces

Here's a trimmed example from the digest — Dutch Bros, a rapidly scaling coffee chain with ~1,000 employees:

> **Score:** 68/100 (Moderate)
> **Recommendation:** Monitor
>
> Dutch Bros shows moderate ICP fit as a rapidly scaling retail chain in target geography with confirmed data infrastructure needs, but signals lack explicit confirmation of active modernization budgets or senior data/tech leadership hires.

The reasoning section is where the nuance shows up:

- Operations chief hire and 900+ location scaling indicates active growth phase requiring data infrastructure for loyalty, inventory, and real-time ops
- Signals are dominated by investor sentiment and real estate transactions rather than high-reliability budget events
- Strong positive keyword alignment ('data capability,' 'loyalty,' 'real-time ops') but these are *inferred from context* rather than confirmed by press signals

And the risk factors:

- No explicit Series B+ funding round or growth equity raise announced
- No CTO, CDO, or VP Data hire explicitly confirmed; operations chief hire is relevant but not a data-specific mandate
- Inferred data need is strong contextually but lacks confirmation of active modernization project or budget allocation

What I find valuable here is the distinction Claude draws between contextual inference and confirmed signals. Dutch Bros *probably* needs data infrastructure — they're scaling 900+ locations with loyalty and real-time ops. But "probably needs" and "has allocated budget for" are different conversations, and the system surfaces that gap explicitly.

A sales rep reading this digest gets a prioritized list with reasoning they can act on, not a black-box score they have to trust blindly.

## What I'd build next

The current version monitors five companies using public news as the primary signal source. The obvious extensions:

- **Richer signal sources.** SEC filings for budget language. LinkedIn hiring patterns for team-building signals. Glassdoor for internal transformation chatter.
- **Historical trend tracking.** A single snapshot is useful; a score trending from 52 to 78 over six weeks tells you when to engage.
- **Delivery integration.** Push the digest to Slack or email. Alert on score jumps above a threshold.
- **Multi-persona rubrics.** The same company scores differently depending on whether you're selling data infrastructure vs. security tooling. Swap the config, keep the pipeline.

But the tool is less interesting than the pattern. The approach — encode domain expertise into an explicit rubric, version it, let the model apply it — works for any domain where expert judgment exists but isn't systematized. Underwriting. Competitive analysis. Vendor evaluation. Content moderation. The common thread is: humans define what "right" looks like, and AI applies that definition at scale.

The alternative — fine-tuning a model or hoping it figures out your domain from a vague prompt — is slower, more expensive, and harder to debug. When the scoring is wrong, I don't retrain a model. I read the rubric, find the gap, and fix it in a text file.

That's the whole point: make the judgment layer explicit, keep the AI on the execution layer, and version everything in between.

*The code is on [GitHub](https://github.com/pablooverton/icp-signal-monitor).*
