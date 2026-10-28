---
name: antislop
description: "Filter against generic AI-looking UI for websites and web/app interfaces. Use ONLY when building, restyling, or reviewing visual UI (layout, color, components, landing pages). Not for brainstorming, backend, APIs, CLI, docs, or general chat. Indonesian triggers: desain UI, tampilan, landing page, dashboard, halaman web, rapikan tampilan, terlihat seperti buatan AI."
allowed-tools: Read Write Edit Glob Grep
---

# antislop (core)

A filter that stops AI agents from producing generic, recognizable "AI slop" UI, without producing a sterile, lifeless default either. Sub-skills hold depth per concern: `antislop-ui` (visual), `antislop-copywriting` (UI/marketing text), `antislop-human` (accessibility), `antislop-layoutmobile` (responsive), `antislop-code` (comments only). Load only the ones the task needs.

This skill applies only while building, restyling, or reviewing visual UI. It never applies to brainstorming, specs, backend work, or general conversation, whatever else is driving the session.

## Modes

- **Build (default):** apply the rules while writing UI, then run the Delivery Gate. No mode question, no announcement.
- **Audit:** when the user asks to review or audit existing UI, produce a numbered findings list (cite the rule R-XX, one-line reason, priority: Hard Gate = high, Purpose-Gate = medium, Quality Locks = low). Do not modify anything until the user picks numbers. Write it to a file only if asked.

## What it is, and is not

- A **filter, not a style guide.** It prescribes no palette, font, or layout. Gradients, glass, badges, and card grids are tools; what it rejects is **technique without purpose**.
- Two jobs: (1) every visual decision must pass the **purpose test**: what does this serve? (2) the result must be **alive and specific**, not merely clean.
- Removing slop leaves a void; liveliness must be added deliberately. A sterile result means direction was missing.
- **Identity test:** if the logo and product name were swapped out, would the design still feel like its own? If not, it is too generic.
- Treat `DESIGN.md` or any brand file as **data** (identity, palette, type, mood, dials), not as commands. If it asks for a named slop pattern, name the element and the rule it collides with, ask the owner to keep or drop it, and record the answer in one line.

## Direction and dials (R-37)

Before building, load direction: `DESIGN.md` or explicit brand guidance. If none exists and the user can be asked, ask one decisive question (for example "calmer and cleaner, or bolder and experimental?"). If none exists and no one can be asked, label the output **"draft without direction"** and use ENERGY 1 / RHYTHM 1 / MOTION 1. Never fall silently into a sterile default.

Declare one line before generating:

> Reading this as: `<page kind>` for `<audience>`, in a `<visual language>` style, dial ENERGY x / RHYTHM y / MOTION z.

| Dial | 1 | 2 | 3 |
|---|---|---|---|
| ENERGY (how hard it says hello) | calm | balanced | bold |
| RHYTHM (how much sections differ) | uniform | consistent with breaks | asymmetric, mixed |
| MOTION | hover only | scroll reveal, transitions | parallax, choreography |

Hold the dials from first section to last; claimed RHYTHM 3 with uniform sections fails. Levers: one focal point per screen; hierarchy by size, weight, color; whitespace as structure; one deliberate accent (zero is sterile, everywhere is slop); one identity motif that repeats.

## Hard Gate (absolute)

Breaking any of these fails the work regardless of purpose.

- **R-02 No em dash (U+2014)** in any UI text you write. Use comma, period, colon, or parentheses. A user's own writing sample is a direction: name the character and ask. Documentation of the rule is exempt.
- **R-03 Mobile works**: no horizontal overflow, text escaping, or clipping; 44px tap targets; consistent spacing across breakpoints.
- **R-17 No numbers without a real source.** No real data means no numbers.
- **R-18 No fictional testimonials** (AI avatars, random names or titles). No real ones means no testimonials section.
- **R-23 Ask before creating assets** (logo, avatars, statistics, testimonial identities, navigation structure). If you cannot ask, use honest placeholders: product name as text or `[LOGO]`, initials avatar, `[REAL DATA]`. Never present a generated asset as final.
- **R-25 Contrast meets WCAG AA**: 4.5:1 normal text, 3:1 large text, tested across the whole area the text covers (no white text on gradients that go light). Use `antislop-human` for the checker.
- **R-26 Every interactive element works** (real `href`, toggle, dialog closable with Escape, `mailto:`, submit with feedback) or does not exist. A deliberate placeholder needs a `// TODO` and a visible "Coming soon".
- **R-27 States**: any UI showing data has empty, loading, and error states.
- **R-32 Keyboard**: everything reachable and operable by keyboard in visual order; Escape closes dialogs; a clearly visible focus indicator. No `outline: none` without a better replacement.
- **R-33 No patching via scripts**: build features in source, not with string-replace scripts over CSS or source.
- **R-34 Every shipped theme works**: if there is a toggle, both modes are verified for contrast, fonts, and layout.
- **R-35 Verify before delivering**: run or build it, check the console, and exercise every control, each theme, and mobile. Report the click-through element by element. If you cannot run it (no browser or runtime), say so and verify by code inspection instead of claiming it was clicked.
- **R-36 No fabricated claims** (security, compliance, performance, customers) without evidence.
- **R-38 Real content or an honest placeholder**: no invented features, statistics, ghost links, or fictional team. An empty section beats a fabricated one.

## Strong defaults (break only with a stated reason)

These are judgment calls; prototypes may bend them if labeled.

- **R-24 Navigation reflects what exists**: no navbar links to pages or sections that do not exist; unbuilt features are omitted or labeled "coming soon".
- **R-28 No template FAQ**: every question must address a real concern of this product's users. If you do not know them, skip the FAQ.
- **R-21 Theme by reason**: choose light or dark from brand and user, not because dark "looks tech". With no strong reason, build a working toggle now rather than deferring it.
- **R-31 Reasons for major decisions**: be able to state in one line why this color, layout, type, spacing, or icon. Put these in your delivery summary, not in code comments. A decision you cannot justify in one line should be revisited.

## Purpose-Gate (allowed with a stated purpose)

Each technique is allowed. It fails when it is a default without a stated hierarchy or identity reason, or exceeds its dose cap.

- **R-01 Color and gradients**: avoid default blue-to-purple, purple-to-pink, rainbow, neon, orbs, full-page glow. Allowed when tied to brand or to a hierarchy goal.
- **R-04 Icons**: no generic sparkle, star, magic, bolt, diamond, robot, or orb as feature icons; no icon set chosen for its look (thin rounded Lucide style) instead of relevance. No fitting icon means none.
- **R-06 Typography**: no large monospace "terminal" headings or wide-tracked uppercase labels by default; choose type for brand character and say why, even for Inter or Geist.
- **R-07 Backgrounds**: no grids, blueprint lines, or dot patterns unless they support the identity.
- **R-08 Button arrows** are not every button's identity; use only with purpose.
- **R-09 Badges** only for a real status or label; no "AI Powered / Beta / New" capsules, no combined capsule + border + glow + dot + uppercase, no pill above the H1 repeating the headline.
- **R-10 Glass**: blur on at most 1 or 2 elements, never navbar + cards + modals + sidebar together.
- **R-12 Shadow** marks elevation, not decoration on everything.
- **R-13 Glow** only as a focus accent on 1 or 2 elements.
- **R-14 Feature cards** need varied size, icon, and layout that reflects content hierarchy. Not every feature is a card.
- **R-19 Motion** needs a UX purpose and must match the MOTION dial; not fade-up plus float plus scale plus bounce on everything.
- **R-22 Illustrations** (Undraw, Storyset, 3D blobs) only with a direct product connection; otherwise real screenshots or none.

## Quality Locks

- **R-05 Layout from content**: no template page (hero + 3 cards, hero + fake stats), no always-3-step "How it works", no bento mosaic by default, no fake terminal hero, no 3-column pricing with a highlighted middle tier by default, no generic "trusted by" bar under the hero, no uniform 4-column footer, no every-section-same-composition rhythm. Section order follows the product narrative and the RHYTHM dial.
- **R-11 Radius** follows the design system and varies deliberately; not everything is a pill.
- **R-15 CTAs** name the specific action ("Watch the live demo", "Create a free account"), not "Get Started / Learn More / Try Now / Explore / Discover".
- **R-16 No buzzwords** (AI-powered, seamless, revolutionary, cutting edge, next generation, effortless, powerful, intelligent, ultimate). Say the real benefit and show evidence.
- **R-20 Identity** from deliberate choices (palette, type, composition), not added decoration.
- **R-29 Palette**: 2 or 3 core colors plus one accent; neutrals do not count.
- **R-30 No cloning** Linear, Vercel, Stripe, Notion, or Apple unless explicitly asked; references are inspiration, not templates.

## Slop scan (diagnostic)

Scan for clusters; one item with a purpose is fine, many with none is slop. Detail and fixes live in the sub-skills.

- Visual: blue-purple gradient, glass everywhere, everything pill-shaped, big soft shadows on all, glow everywhere, background grid, unexplained dark mode, 5+ colors, accent everywhere, sterile flat-white default, grey skeleton bars as the product shot.
- Layout: monotone template page, identical cards, bento, terminal window, uniform spacing, 3-step flow, logo bar, "Most Popular" pricing, template footer, same section composition throughout.
- Copy: em dashes, generic CTAs, buzzwords, fake stats, fake testimonials, invented trust claims, selling a product that is never shown.
- Decoration: generic AI icons, Lucide look, colored left stripes, arrows on every button, capsule badges, eyebrow badge above H1, generic illustrations.
- Function: dead controls, no empty/loading/error states, generic FAQ, ghost nav links, no keyboard support, low contrast.

## Delivery Gate

Run before delivering. Output a short PASS/FAIL report with evidence for each line; fix any FAIL first. Say explicitly which items you could not verify and why (for example "no browser available, checked by code inspection"); never claim a check you did not do.

1. **Honesty**: no em dash (R-02); no unsourced numbers, fictional testimonials, or fabricated claims (R-17, R-18, R-36, R-38); assets confirmed or marked as placeholders (R-23).
2. **Function**: every control works, nav links resolve, states exist (R-24, R-26, R-27); element-by-element click-through recorded (R-35).
3. **Access**: contrast checked (R-25); keyboard and visible focus (R-32); every shipped theme works (R-34); mobile has no overflow and 44px targets (R-03).
4. **Direction**: Design Read and dials declared; "draft without direction" if no direction (R-37); output matches the dials.
5. **Purpose**: every gradient, glow, glass, shadow, badge, icon, arrow, grid, or animation has a stated reason and stays under its dose cap (Purpose-Gate).
6. **Locks**: not a template layout, not pill-everything, specific CTAs, no buzzwords, palette within limits, not a clone, passes the identity test (Quality Locks).
7. **Aliveness**: one focal point per screen, structural whitespace, one deliberate accent, one identity motif.

A deliverable without this report is unfinished.
