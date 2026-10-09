---
name: grilling
description: Grill the user relentlessly about a plan, decision, or idea. Use when the user wants to stress-test their thinking, or uses any 'grill' trigger phrases.
---

# Grilling

The user is interviewed relentlessly until a shared understanding is reached, mapped as a design tree in which every decision branches into the decisions that hang off it.

## One question at a time

The tree is worked one question at a time. The frontier is every decision whose prerequisites are already settled, the questions askable now without guessing at answers not yet heard.

The single most load-bearing frontier question is asked with its recommended answer and ends with the confirmation prompt "Are you okay with this proposition?", and then the interview waits.

Pushback or a re-ask is resolved before moving on; only a confirmation unlocks the next question.

## The shape of a question

Every question has the same layout: the question is a `##`, each option a `###`, and each option's four ideas a `####` with a short prose paragraph under it.

```
## Q1 · <question title>

<one or two sentences framing the decision>

<artifact: what the situation looks like today>

⠀

⠀

### Option A: <name>

#### What it changes

<prose>

<artifact: A rendered, every line labelled A>

#### 🟢 Standard: aligns best with <X>

<the definition, quoted with its citation>

#### Why it's a candidate

<prose>

#### 🟢 VALUE ▰▰▰ HIGH · 🟡 EFFORT ▰▰▱ MEDIUM

<prose>

⠀

⠀

### Option B: <name>
…

⠀

⠀

➡️ <your recommended answer, one or two sentences>

Are you okay with this proposition?

## What I need from you

1. <what to pick>:

A
B
```

The closing section is the one [writing/SKILL.md](../writing/SKILL.md) fixes for every reply, with each option letter as a reply line.

Spacing is a token, a line holding only the blank braille pattern U+2800, standing alone as its own paragraph with a blank line on each side: the terminal collapses plain blank lines, prints HTML entities verbatim and trims a non-breaking space, while U+2800 is a symbol it draws as empty space. Two tokens separate the framing from the first option, one option from the next, and the last option from the recommendation; an option's `####` subsections take only the ordinary paragraph gap.

```
between options          U+2800  U+2800    (2, each its own paragraph)
between #### children    none              (the ordinary gap)
```

The value-and-effort heading carries both grades twice over: a three-block meter for magnitude and, before it, an emoji that reads as good or bad for the maintainer, so high value and low effort are green. The prose under it says what the value buys and what the effort costs.

```
VALUE    🟢 ▰▰▰ HIGH   🟡 ▰▰▱ MEDIUM   🔴 ▰▱▱ LOW
EFFORT   🟢 ▰▱▱ LOW    🟡 ▰▰▱ MEDIUM   🔴 ▰▰▰ HIGH
```

The Standard heading carries the verdict as a coloured emoji, so the fit scans by colour down the margin.

```
🟢 aligns best    🟡 partial    🔴 contradicts
```

Artifacts are mandatory: every option is shown under its "What it changes", never described and left to the imagination. The options' artifacts carry the option letter and the same columns, so scanning down the margin compares them; the recommendation comes only after every option has been seen.

What counts as an artifact, how small it is and the say-a-little-draw-a-little rhythm are fixed in [artifact/SKILL.md](../artifact/SKILL.md), read before the first question.

## Standard fit

The repo profiles four standards, each with a skill and a `STANDARD.md` quoting the source: [akn](../akn/SKILL.md), [skos](../skos/SKILL.md), [otel](../otel/SKILL.md) and [retrieval-telemetry](../retrieval-telemetry/SKILL.md). A question that touches ground one of them defines or has an opinion on is asked only after that skill and its `STANDARD.md` are read in full; what they leave open is checked against the official spec by a research agent, as a fact found rather than asked.

Every option's Standard subsection grades it against that ground: the definition it falls under, quoted with its citation (`XSD:3244`, `P1 §5.10.11`), and whether it fits fully, partially or contradicts it. The verdict rides the heading, so the best fit names the standard there.

```
#### 🟢 Standard: aligns best with AKN

`num` is "a number or any other ordered mechanism to identify the structure" (XSD:3244).
```

A question no standard speaks to says so once, in its framing, naming what was checked, and its options carry no Standard subsection.

```
The standard is silent here: AKN fixes num and heading as elements and says nothing about how a reader joins them (STANDARD.md, P1 §4.7).
```

The recommendation is a separate judgment from the fit: when it goes against the best-aligned option, it says why.

## The frontier moves

Each confirmed answer reshapes the tree: settled decisions push the frontier outward and unblock the questions that depended on them, so the frontier is recomputed and the next single question asked. A question whose answer depends on one still unconfirmed is never asked.

## Facts are found, decisions are asked

Finding facts is the interviewer's job, never the user's. When a frontier question needs a fact from the environment (filesystem, tools), a sub-agent is dispatched to find it rather than asking the user for anything that could be looked up, and nothing blocks on it: a running exploration is an unsettled prerequisite, so the questions downstream of it wait for the sub-agent to report while the next independent frontier question is asked.

The decisions are the user's, each put to them with a wait for the answer.

## When it ends

The session is done when the frontier is empty, every branch of the design tree visited and nothing left silently assumed. Nothing is acted on until the user confirms the shared understanding has been reached.
