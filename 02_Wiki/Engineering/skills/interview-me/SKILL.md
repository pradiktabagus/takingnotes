---
name: interview-me
description: Extracts what the user actually wants instead of what they think they should want. Achieves this through one-question-at-a-time interview until ~95% confidence about the underlying intent, while tracking decisions so the conversation converges instead of circling. Use when an ask is underspecified ("build me X" without "for whom" or "why now"), when brainstorming an open topic and the discussion keeps looping or drifting, when the user explicitly invokes ("interview me", "grill me", "are we sure?", "stress-test my thinking", "brainstorm", "kita muter-muter"), or when you catch yourself silently filling in ambiguous requirements before any plan, spec, or code exists.
---

# Interview Me

## Overview

What people ask for and what they actually want are different things. They ask for "a dashboard" because that's what one asks for, not because a dashboard solves their problem. They say "make it faster" without a number to hit.

The cheapest moment to find this gap is before any plan, spec, or code exists. Once you've started building, switching costs are real, and the user will rationalize the wrong thing into a "good enough" thing. The misfit gets locked in.

This skill closes the gap before it costs anything. Interview-me comes before planning or specs: ask one question at a time, with your best guess attached, until you can predict what the user is going to say before they say it. `doubt-driven-development` stress-tests a decision after it is drafted; interview-me finds the decision in the first place.

A second job: **keep the conversation converging.** Open-ended discussions loop because nothing records what was already settled. This skill keeps a running Decision Log (see "Keeping the Conversation Converging") so settled points stay settled and side topics don't hijack the thread.

## When to Use

Apply this skill when:

- The ask is missing at least one of: **who** the user is, **why** they want it, what **success** looks like, what the binding **constraint** is
- The request is conventional rather than specific ("build me X", "make it faster") and you can't unpack the convention without guessing
- You're tempted to start with assumptions you haven't surfaced
- The discussion is open-ended (brainstorming, exploring options) and is repeating, drifting, or reversing earlier points
- The user hasn't said which value they're optimizing for when two reasonable ones are in tension (simplicity vs. flexibility, cost vs. speed)
- The user explicitly invokes: "interview me", "grill me", "before we start, are we sure?", "stress-test my thinking"

**When NOT to use:**

- The ask is unambiguous and self-contained ("rename this variable", "fix this typo")
- The user has explicitly asked for speed over verification
- Pure information requests ("how does X work?", "what does this code do?")
- Mechanical operations (renames, formats, file moves)
- You already have ≥95% confidence (verify with the stop test below before assuming so)

## Loading Constraints

This skill needs a live, responsive user. **Do not invoke in non-interactive contexts** like CI pipelines, scheduled runs, `/loop`, or autonomous-loop. If you're in one of those and the ask is underspecified, flag that as a blocker for the user instead of guessing.

## The Process

### Step 0: Look before you ask

In an existing project, or when the user pointed at context (a spec, a handoff, files, earlier conversation), read it first, plus the code or material it connects to. Keep reading until you can say how the relevant parts work today and what the change would touch.

- **Never ask what the project or the context already answers.** A question the repo could have answered spends the user's attention on your homework.
- If you find something they may not have considered (a constraint, a conflict with existing code, a cheaper path), say it plainly in your first message. It may change the ask.
- **Fast exit:** if after looking everything that matters is in the request, this skill is done. Do the work and say what you did.

### Step 1: Hypothesize, with a confidence number

Before asking anything, write down your current best read of what the user wants in **one sentence**, plus an honest confidence number (0–100%):

```
HYPOTHESIS: You want a way to answer "how are we doing?" in standup, and "dashboard" was the convention that came to mind.
CONFIDENCE: ~30% — missing: who it's for, what "metrics" means in context, and what success looks like
```

The number forces honesty. If you wrote down a high number but can't actually predict the user's reactions to the next three questions you'd ask, the number is wrong. Start at the confidence level you can defend.

When confidence is below ~70%, append a brief reason on the same line — what's still unresolved or missing. This tells the user exactly what the interview needs to surface, and prevents the number from being a vague signal.

### Step 2: Ask one question at a time, each with a guess attached

Format:

```
Q: <one focused question>
GUESS: <your hypothesis for the answer, with the reasoning that produced it>
```

Wait for the user to react before asking the next question.

**Why one at a time, not a batch:**

- The user can't react to your hypotheses if you bury them in a list
- Batches encourage skim-reading and surface answers
- The third question often depends on the answer to the first; asking them all at once locks in the wrong framing
- The user's energy for thinking carefully is finite; spend it one question at a time

**Why attach a guess:**

- The user reacts faster to a wrong guess than they generate an answer from scratch
- It commits you to a hypothesis you can be visibly wrong about, which keeps you honest
- It surfaces *your* assumptions, which is what the interview is meant to expose

The risk here is a polite user agreeing with your guess to be agreeable. Mitigate by being visibly willing to be wrong, and occasionally guess in a direction you expect the user to push back on.

**Light opt-out on the first question.** If the user invoked the skill or asked to brainstorm, just ask. If the skill fired on its own, add one line after your first question: "Or if you'd rather I just start, say so." If they take it, the interview is over: state any significant choice you're making (platform, language, library) in one plain line, then do the work.

**Ask about how it gets made, early.** Find out which technical choices (stack, libraries, hosting, data store) the user wants to make, which they're handing to you, and what they already have. Someone with strong views wants to discuss them; someone counting on you needs sensible defaults said out loud. In existing work, build on what it already uses. A default you picked is logged as your call, never as something they agreed to.

**Offer recon when it would help.** Once you have some grounding, offer to look deeper (the codebase, their files or data, the web) and say what you'd look for. The user decides whether you go.

### Step 3: Listen for "want vs. should want"

The most dangerous answers are the ones where the user says what a thoughtful answer *sounds like* rather than what they actually want. Watch for:

- Answers that pattern-match best-practice talk ("I want it to be scalable", "clean architecture") without specifics
- Answers that defer to convention ("the way most apps do it", "the standard approach")
- Phrases like "I should probably…", "I think I'm supposed to…", "good engineering practice says…"
- Buzzwords as goals — when "modern", "scalable", "robust" are the answer instead of a specific outcome

When you hear these, the question to ask is:

> *"If you didn't have to justify this to anyone, what would you actually want?"*

That single question often does more work than the previous five.

### Step 4: Restate intent in the user's own words

When your confidence is high, write back what you now think the user wants. Keep it tight (5–8 lines), use their language where possible, and structure it so the user can confirm or correct line by line:

```
Here's what I now think you want:

- Outcome:      <one line>
- User:         <one line — who benefits>
- Why now:      <one line — what changed>
- Success:      <one line — how we know it worked>
- Constraint:   <one line — the binding limit>
- Out of scope: <one line — what we're explicitly not doing>

Yes / no / refine?
```

Including "Out of scope" is non-negotiable. Half of misalignment is silent disagreement about what is *not* being built.

### Step 5: Confirm — explicit yes, not "whatever you think"

The gate is an explicit "yes." The following are **not** yes:

- "Whatever you think is best." → The user is delegating, which means they don't have 95% confidence either. Re-ask with two concrete options framed as a choice.
- "Sounds good." → Ambiguous. Ask: "Anything you'd refine?" Silence isn't confirmation.
- "Sure, let's go." → Often a polite exit, not an endorsement. Same follow-up.
- Silence followed by "okay let's start." → The user has given up on the interview, not converged. Stop and ask whether you've missed something.

If they correct you, fold the correction in and restate. Loop until you get an explicit yes.

### Step 6: Lay out the approaches

The confirmed restate settles *what*. Before anything is built, settle *how*, in a message of its own:

1. Come up with several real ways to build it. Drop the weak ones, including any that work against the confirmed Outcome or that the existing code rules out.
2. Present the two or three that remain, each grounded in what the project actually does, with what it trades off.
3. Recommend one and say why, then ask which way they want to go. If there is genuinely one sensible way, say so and say why the others don't fit.

```
Approaches:
A. <name>: <how, in one line>. Trade-off: <cost or risk>.
B. <name>: <how>. Trade-off: <...>.
Recommend: A, because <reason tied to the confirmed Outcome or Constraint>.
Which way?
```

Log the pick in DECIDED. Skip this step in Exploration Mode, or when the work is a small change with one obvious way to do it.

**Size the work.** Once they've picked, say how big it is in plain words: a small change, or a project. The user can override. If it is really several independent pieces, say so, agree on an order, and take them one at a time. If it grows mid-task, stop and say so.

**Offer a spike when it's cheaper than talking.** A quick throwaway build is often the fastest way to learn what the user wants or whether something works. Offer one when it would help. A spike gets minimal or no tests and stays labeled throwaway. Bring what it taught you back into the conversation; keeping and building on the spike is a new request, not a continuation.

### The 95% Confidence Stop

You're done when you can answer yes to this:

> *Can I predict the user's reaction to the next three questions I would ask?*

If yes, you have shared understanding. Stop interviewing and produce the restate. If no, you're not done; ask the next question.

This is a checkable test, not a vibe. It also has a floor: if you've gone several rounds and still can't predict, that's information about the ask, not a reason to keep grinding. Stop and tell the user: "I've asked X questions and I still can't predict your reactions. Something foundational is missing. Want to step back?"

## Keeping the Conversation Converging

Brainstorming loops when nothing records what was already settled. Use these mechanics from the first question on.

### Decision Log

Keep a running log. Show it compactly every ~4 questions, whenever the user asks "sampai mana kita?", and before any restate:

```
DECIDED:  <settled points, one line each>
OPEN:     <the one question we're working on now, then others queued>
PARKED:   <side topics set aside, with where they came from>
```

Rules:
- A DECIDED item is not reopened silently. To change it, the user must say so, and you log the reversal ("D2 changed: A -> B because ...").
- Never ask a question whose answer is already in DECIDED. Check the log first.
- OPEN holds one active question. Everything else waits in the queue.

### Parking Lot

When the user raises a tangent, don't follow it and don't drop it: say "Parked: <topic>. Back to <active question>." and add it to PARKED. Revisit PARKED only after OPEN is empty, and ask whether each item still matters.

### Loop Signals

You're circling if any of these happen:

- The user restates a point already in DECIDED
- A decision is reversed twice
- The same question comes back in new words
- Answers get vaguer as questions get more specific

When one fires, stop asking and say so: "We're circling on <topic>. Current log: <log>. The real split is A vs B. Pick one, or tell me what's missing that makes it undecidable." Force a choice between two concrete options. If the user can't choose, name what information is missing and make getting it the next step (research, a quick test, asking someone else) instead of more discussion.

### Question Budget

After 5 questions on one topic, pause and give the log plus one sentence: "Still unresolved: <X>. Continue, decide with what we have, or park this?" Do not exceed 8 questions on one topic without the user explicitly choosing to continue.

### Exploration Mode

For open brainstorming with no deliverable to build ("let's think through X"), the Step 4 restate is too heavy. Use the lighter close:

```
Goal:        <what this discussion is for>
Decided:     <log>
Still open:  <what's unresolved>
Next step:   <one concrete action>
```

Same rule applies: confirm with an explicit yes.

## Output

The output of this skill is a **confirmed statement of intent**: the restate from Step 4, with an explicit yes from Step 5, plus the approach the user picked in Step 6 when building follows. That's the deliverable. Specs, plans, and task lists are downstream; they consume the intent this skill produces.

If the user wants the intent to persist (a multi-session project, a handoff to another collaborator), offer to save it to `docs/intent/[topic].md`. Only save if they confirm.

## Example

A short before-and-after.

**Without `interview-me`:**

```
User:  build me a dashboard for our metrics
Agent: <starts proposing chart libraries and layouts>
```

The agent has assumed: who it's for, what metrics, what "for our team" means, what success looks like. None of these are stated. Every assumption is a place the user can later say "that's not what I meant."

**With `interview-me`:**

```
User: build me a dashboard for our metrics

Agent:
HYPOTHESIS: You want a way to answer "how are we doing?" in standup, and "dashboard" came to mind because that's the conventional answer.
CONFIDENCE: ~30% — missing: who it's for, what "metrics" are, and what success looks like

Q:     When you say "how are we doing?", who's asking — you alone, the engineering team in standup, or up the chain (e.g., your manager, an exec, a board)?
GUESS: engineering team in standup, because "we" usually scopes that way and standups are where this question gets asked. If it's for execs the metrics and the framing change a lot.

User: It's actually for me. I keep losing track of which experiments are running and what their early signal looks like.

Agent:
Updated read: not a team dashboard at all. It's a personal experiment tracker. Confidence: ~60% — still missing: what "early signal" means and what done looks like.

Q:     Is the gap that you don't know which experiments exist, or that you can't see their results in one place?
GUESS: the second one. You have a list of experiments somewhere, but their results live in five different tools, and reconciling them by hand is what you're tired of.

User: First one actually. I literally don't have a list. They're spread across different docs.

Agent: <continues>
```

Two questions in, the agent has discovered the actual ask isn't "a dashboard." It's "a list." Different artifact, different scope, different work. The dashboard would have been wrong.

## Interaction with Other Skills

- **`doubt-driven-development`**: downstream. Interview-me is pre-decision intent extraction; doubt-driven is post-decision review. After the user confirms a significant decision, offer to stress-test it.
- **`context-engineering`**: pairs well. In a long session, the Decision Log doubles as the summary to carry into a fresh session.
- **`source-driven-development`**: orthogonal. Interview-me clarifies what the user wants; SDD verifies framework facts.
- **`incremental-implementation`**: downstream. Hand it the confirmed intent, the picked approach, and the size from Step 6; a project-sized piece becomes its slices.
- If spec, planning, or idea-refinement skills are installed, hand the confirmed intent to them. If not, write the next step directly from the restate.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The ask is clear enough" | If you can't write the user's desired outcome in one sentence right now, the ask isn't clear. Run Step 1 before deciding. |
| "Asking too many questions wastes their time" | Time wasted by 4–6 targeted questions is small. Time wasted by building the wrong thing is enormous, and the user is the one bearing that cost. |
| "I'll figure it out as I build" | Switching costs after code exists are 10x what they are now. Discovery during implementation is rework. |
| "They said 'whatever you think,' so I should just decide" | "Whatever you think" is delegation, not decision. Re-ask with two concrete options as a choice. |
| "I should give them several options to pick from" | Options work when the user knows what they want and is choosing between trade-offs. They don't know what they want yet. Listing options widens the search; asking narrows it. Options belong in Step 6, for *how*, after the intent is confirmed. |
| "I'll ask, it's faster than reading the code" | It's faster for you and slower for them. Run Step 0; ask only what the project can't answer. |
| "I'll pick sensible defaults quietly" | Say them out loud. A default the user never heard is one they couldn't reject. |
| "If I attach my guess, I'm leading them" | Leading is the point. Reacting is faster than generating from scratch. The risk is sycophancy, not leading; mitigate by being visibly willing to be wrong. |
| "We've talked enough, I get it" | Test it: can you predict their reaction to the next three questions? If not, you don't get it yet. |
| "The user said yes, we're done" | If the yes followed a vague restate or an open-ended "sounds good," the yes is hollow. Restate concretely and re-confirm. |

## Red Flags

- Three or more questions in a single message: that's batching, not interviewing
- A question without your hypothesis attached: that's surveying, not committing
- Accepting "whatever you think is best" as a terminal answer
- Producing a spec, plan, or task list before the user has explicitly confirmed your restate
- Questions framed as "what would be best practice?" instead of "what do you actually want?"
- The user gives a sophistication-signaling answer ("scalable", "clean", "modern") and you accept it without probing whether it's what they actually want
- Three or more rounds without your confidence visibly rising: you're asking the wrong questions, step back and reframe
- A confidence number below ~70% with no reason attached: the user can't help close the gap if they don't know what's missing
- Saving the intent doc before the user has confirmed (the doc itself implies a yes the user didn't give)
- Skipping the "Out of scope" line in the restate (silent disagreement about non-goals is half of misalignment)
- Asking a question whose answer is already in the Decision Log
- Following a tangent without parking it, or dropping it without saying so
- Going past 5 questions on one topic with no log or checkpoint
- Reopening a DECIDED item without logging the reversal
- Asking something the codebase or the user's own context already answers
- Starting to build after the restate without laying out approaches (unless it's a small change with one obvious way)
- Treating "sounds good" to an idea as approval of a design or approach you haven't shown
- Continuing to build on a spike as if it were the real thing

## Verification

After applying interview-me:

- [ ] An explicit hypothesis with a confidence number was stated in the first turn
- [ ] Every confidence number below ~70% was accompanied by a one-line reason (what's still unresolved or missing)
- [ ] Questions were asked one at a time, each with the agent's guess attached
- [ ] At least one "what would you actually want if you didn't have to justify it?" probe ran when the user gave a sophistication-signaling or convention-signaling answer
- [ ] A concrete restate (Outcome / User / Why now / Success / Constraint / Out of scope) was written back to the user
- [ ] The user confirmed the restate with an explicit yes (not "whatever you think," not "sounds good," not silence)
- [ ] At the stop point, the agent could predict reactions to the next three questions it would ask
- [ ] A Decision Log (DECIDED / OPEN / PARKED) was shown at least every ~4 questions in sessions longer than that
- [ ] No question repeated something already in DECIDED; tangents were parked, not followed or silently dropped
- [ ] Any loop signal (repeat, double reversal, vaguer answers) triggered a forced two-option choice instead of more questions
- [ ] No topic exceeded 8 questions without an explicit user choice to continue
- [ ] Any handoff to a downstream skill was framed in terms of the confirmed intent, not the original underspecified ask
- [ ] Relevant code or context was read before the first question; no question asked what the project already answers
- [ ] Before building, two or three approaches with trade-offs and a recommendation were shown, and the user picked one (or there was one obvious way and you said why)
- [ ] Defaults the agent chose were stated out loud and logged as the agent's call
