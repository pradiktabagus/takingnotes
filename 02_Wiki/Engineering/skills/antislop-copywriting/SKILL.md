---
name: antislop-copywriting
description: "Anti-AI-writing rules for UI and marketing copy: headlines, CTAs, product text, tone. Use when writing interface or landing-page text. Not for emails, docs, chat answers, or brainstorming. Indonesian triggers: teks UI, copy bahasa Indonesia, judul, tombol CTA, teks landing page, copywriting."
allowed-tools: Read Write Edit Glob Grep
---

# antislop-copywriting

Copy and text depth for the `antislop` core skill: headlines, CTAs, product and landing-page text, and the patterns that make AI-written prose easy to spot. The core holds the hard bans (R-02, R-15, R-16, R-17, R-18, R-36, R-38); this file adds the prose patterns it does not cover.

- **Indonesian copy:** also read `references/indonesian.md` (register `Anda` or `kamu`, Indonesian buzzwords and CTAs, formats like `Rp 1.500.000`).
- **Never invent facts** (R-17, R-36, R-38). A rewrite adds no fact, name, number, date, quote, or citation that is not in the source or supplied by the user. If a sentence needs real detail, ask for it or write the plain version without it.
- **Do not over-sterilize.** Copy with no voice is as machine-made as copy full of tells (R-37). When the user supplies a voice, keep it.
- Look for **clusters, not isolated tells.** One "however" or one em dash proves nothing; em dashes plus a forced trio plus "vibrant tapestry" plus a generic conclusion does.

## Tone and voice

- **Empty AI vocabulary**: unlock, elevate, empower, delve, showcase, testament, landscape, journey, robust, game-changer, next-level, seamless, cutting-edge, revolutionary. They signal intent to impress, not inform. Say what the thing does. (R-16)
  - Before: "Unlock the power of seamless collaboration to elevate your team's journey." After: "Work with your team in one shared space."
- **Significance inflation**: "the future of X", "a pivotal moment", "a new era of". Replace with the concrete effect. (R-36, C-5)
- **Empty social proof**: "trusted by thousands", "industry-leading", "world-class", with nothing named. Name real customers or cut the claim. (R-17, R-18, R-36)
- **Weasel attributions**: "experts say", "industry observers", "studies show" with no source. Name the source or cut it. (R-36)
- **Authority tropes**: "at its core", "the real question is", "what really matters", "fundamentally". Say the point directly.
- **Chatbot closers**: "I hope this helps!", "Let me know if you have questions". Conversation artifacts, not copy.
- **Fake-candid openers**: "Honestly?", "Here's the thing", "Real talk". A person being honest just says it.
- **Signposting**: "Let's dive in", "Here's what you need to know", "Without further ado". Do the thing instead of announcing it.
- **All-caps emphasis** inside prose. Write the emphasis into the sentence. A real headline, a voice that shouts, or one accent word may keep caps.
- **Actorless passive** by default ("the pricing page was updated"). Name who acted: "We rewrote the pricing page". Passive is fine when the actor is unknown, irrelevant, or the object is the real subject.
- **Inanimate subject, human verb**: "the dashboard understands what your team needs". Say what it does: "The dashboard opens on the three metrics your team checks every morning." Ordinary verbs like "the report shows" are fine; the tell is understands, knows, decides, wants, cares.

## Rhythm and structure

- **Rule of three** forced onto every idea ("innovation, inspiration, and insights"). Lists have the number of items the content needs.
- **Negative parallelism and tailing negations**: "It's not just X, it's Y", "no guessing", "no wasted motion".
- **Aphorism formulas**: "X is the language of Y", "Efficiency becomes a trap when". State the actual claim.
- **Staccato drama**: runs of short fragments ("No templates. No defaults. No safety."). One short sentence for emphasis is fine; a run is engineered.
- **Synonym cycling**: "fast / quick / speedy" to dodge repetition. Repeat the clearest word.
- **False ranges**: "from onboarding to scale", "from first touch to final invoice, and everything in between". List what it actually handles.

## Honesty and evidence

- **Fabricated specifics**: invented numbers, testimonials, names, dates, features. The worst pattern, a defect even when it sounds human. No real data means no number and no quote; say what the product does. (R-17, R-18, R-36, R-38, C-5)
- **Speculative gap-filling**: a guess dressed as fact ("the founder likely started small"). State only what a source supports, or omit.
- **Generic positive conclusion**: "The future looks bright". Cut it; end on the last concrete fact.

## Hygiene and markdown

- **Em dashes.** R-02 forbids the em dash character (—) in any text you write, as well as spaced double hyphens used the same way. Replace in rough order of preference: a period, a comma (tight aside), a colon (introduces an explanation), parentheses (true aside), or restructure. Example: "The policy, announced without warning, affects thousands of workers."
  - An em dash alone is not proof of AI (editors use them), but R-02 still bans it in your output. Do not rewrite a user's deliberate style silently.
  - **Voice sample:** if the user's sample uses em dashes, it is a direction, not agent copy. Name the character and the rule, ask (R-37), and match the sample only if the owner keeps it. Never keep or cut them silently.
- **Boldface overuse**: every key term bolded. Emphasis everywhere is emphasis nowhere.
- **Excessive quotation marks**: scare quotes and quotes as default emphasis ("the "solution" "streamlines" your "workflow""). Dialogue, real citations, and titles keep theirs.
- **Inline-header lists**: bullets that start with a bolded header restating the item. Write a sentence or a plain list.
- **Emojis in headings and bullets**: they decorate, they do not inform.
- **Filler**: "in order to" (to), "due to the fact that" (because), "at this point in time" (now), "it is important to note that" (nothing).
- **Stacked hedging**: "could potentially possibly". One qualifier does the work.

## What NOT to flag

Do not gut legitimate prose. These are not reliable indicators alone: perfect grammar and consistent style; mixed casual and formal register; plain or dry writing; formal vocabulary (AI overuses specific words, not all fancy ones); a single transition word; curly quotes; one short emphatic sentence; unsourced claims; secondhand text inside quotations, titles, proper names, or examples being discussed.

## Signs of human writing (preserve)

Specific, unusual, hard-to-fabricate detail; mixed feelings and unresolved tension; dated or era-bound references; varied sentence length; genuine asides and self-corrections. Lean toward leaving prose alone when you see these.

## Voice calibration

If the user supplies a writing sample, read it first (sentence length, vocabulary, openings, punctuation, recurring phrases) and match it instead of merely deleting patterns. The sample outranks these style rules, except R-02, which goes through the R-37 conflict protocol above. Without a sample, use the defaults here.

## Draft, audit, final

1. **Draft**: rewrite applying the patterns. It should read naturally aloud, vary sentence length, and prefer specific detail and simple constructions.
2. **Audit**: answer briefly, "What makes this obviously AI-generated?" and "Does it state any fact, name, number, date, or citation that is not in the source?" A fabrication is a defect even when it sounds more human.
3. **Final**: fix both answers, then check for em dashes one last time (R-02). A hit means it is not done.

## Copywriting checklist (supplement to the core Delivery Gate)

- [ ] No fabricated numbers, testimonials, names, dates, or claims; everything real or a labeled placeholder (R-17, R-18, R-36, R-38)
- [ ] Buzzwords and empty AI vocabulary replaced with specific, evidenced language (R-16)
- [ ] No em dashes (R-02), unless the owner kept them from their own sample under R-37
- [ ] No excess quotation marks, all-caps emphasis, mechanical bold, or emoji decoration
- [ ] Sentences name their actor; no abstraction given a human verb where a real subject exists
- [ ] CTAs specific to the action (R-15)
- [ ] No forced trio, negative parallelism, staccato run, aphorism formula, or false range
- [ ] A real voice is present (the user's sample or a deliberately chosen tone), not a sterile default (R-37)
- [ ] Read aloud: it sounds like a person wrote it
