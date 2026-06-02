+++
title = "Your Personality Is the Input: An AI Config That Argues With You"
date = 2026-06-02T10:00:00-04:00
draft = false
tags = ["ai-engineering", "llm", "agents-md", "tools"]
summary = "Most AI personality prompts aim the wrong way. typecast inverts it: your type is the input, and the output is a config for how the assistant should treat you. Satisfaction mode flatters you; growth mode argues with you."
+++

Most prompts that give an AI a personality are aimed the wrong way. "Act like a friendly ENFP." "Be a blunt senior engineer." You are casting the character the model plays, directing a tiny improv troupe of one. When I sit down to do real work with Claude or ChatGPT, I could not care less which character it performs. I care how it treats me. Those are different problems, and only the second one changes the output I get.

So I built the inverse and called it typecast. Your personality is the input. What comes out is an operating manual for how the assistant should behave toward you. Pick your type, pick a mode, get a config file you can drop straight into your tools.

## Satisfaction mode flatters you

There are two modes, and the distance between them is the entire idea.

I ran it on myself first, both modes, which is the fastest way to feel the difference. Satisfaction mode handed back everything I already believe about how I like to work. It felt great. It was also useless in the exact way a horoscope is useless: it told me what I wanted to hear and asked nothing of me.

Take INTJ as the worked example. Satisfaction mode says lead with the conclusion, assume competence, bring evidence instead of encouragement, praise nothing and move on. Comfortable. Growth mode flips it. If you have asked for more options or more rigor twice on the same decision, it refuses and makes you pick. It steelmans the low-status option you waved away. It names the one assumption your whole conclusion is balanced on and asks whether you have actually tested it. That version is the keeper, because it makes you do the thing you were avoiding.

A horoscope flatters you. Growth mode is built to do the opposite.

## MBTI is the skin, Big Five is the engine

I will be straight about the machinery, because that is where most personality content gets lazy.

MBTI is widely treated as pseudoscience, and that is a fair charge. I am not using it as truth. I am using it as a door people will actually walk through. "I'm an INTJ" is something people put in their dating profiles. "I'm high Openness and low Agreeableness" is not, even though it describes them better. So every type maps to a Big Five profile underneath, and the directives are written against that profile's documented failure modes.

The mapping is lossy on purpose. Low Conscientiousness tends to leave projects unfinished, so growth mode forces a deadline before the conversation closes. High Openness tends toward analysis paralysis, so growth mode forces a ship decision and stops you piling on rigor. The personality label is just the delivery mechanism. The directives are the product.

## Vibes do not survive contact with a model

"Be supportive" does nothing. The model already believes it is being supportive. Every directive in typecast has to be behavioral and checkable, something the model can catch itself doing or failing to do. "If I've opened a third thread before closing the first, pull me back to one." "Before I act, ask what the consequence is two moves out." "When I'm using fun to dodge a decision, stop playing hype man and ask the real question." You can verify whether the model did those things. You cannot verify "supportive."

## What it does not do

The limits matter, because overselling this would wreck the part that is actually real.

Trait-conditioning fades. Over a long session the model drifts back to its defaults, so the config is a starting posture, not a permanent rewiring. When it drifts, paste it again. Self-typing is noisy too, and people mistype themselves constantly. That matters less than you would think for growth mode: if the failure mode it leans on is not yours, the friction feels wrong and you ignore it. A wrong type costs you a few directives you roll your eyes at, not a working assistant.

This is a useful toy, not a validated instrument. Saying that out loud makes it more credible, not less. The interesting part has nothing to do with whether personality tests are science. It is that the highest-value way to set up an assistant is against your own weak spots, and that you have to write the instructions concretely enough for a model to grade itself against them.

## Using it

The output is an `AGENTS.md` file, the cross-tool convention that Cursor, Windsurf, and others already read, and that Claude Code picks up through import. It is plain markdown, so it works just as well pasted into ChatGPT or Claude custom instructions. Pick a type, pick a mode, save the file. No build step, no dependency, no account.

typecast is live at [pablooverton.com/typecast](https://www.pablooverton.com/typecast), and the source is at [github.com/pablooverton/typecast](https://github.com/pablooverton/typecast). Pick satisfaction if you want to feel understood. Pick growth if you want to get something finished.
