---
name: antislop-ui
description: "Visual rules for antislop: color, layout, components, motion. Use when building or editing a web/app interface, together with the antislop core. Indonesian triggers: desain tampilan, warna, tata letak, komponen, animasi, landing page, dashboard."
allowed-tools: Read Write Edit Glob Grep
---

# antislop-ui

Visual and structural depth for the `antislop` core skill. Load it together with the core when building or editing a website, web app, or dashboard. The core holds the mechanism (purpose test, rule tiers, dials, Delivery Gate); this file gives the concrete tells and fixes, citing core rules as R-XX and craft principles as C-X.

Each entry is **Tell** (what it looks like) and **Fix** (what to do). The shared reason: the pattern comes from the model's default, not from the product's identity or content. Technique is allowed when it has a stated purpose; clusters of defaults are slop.

## Visual and color

- **Blue-purple gradient and its family** (blue-cyan, purple-pink, rainbow, purple-and-black, neon or pastel palettes, radial orbs, full-page glow). Fix: take the palette from `DESIGN.md` or the product's identity. Keep a gradient only where it separates a level of hierarchy, with the reason written down (R-01).
- **Glassmorphism everywhere** (blur on navbar, cards, modals, sidebar). Fix: glass is an accent on at most 1 or 2 elements (R-10).
- **Everything pill-shaped.** Fix: a small set of radii from the design system, applied deliberately; one generous radius on the primary action reads as intent (R-11).
- **Large soft shadows on every component.** Fix: shadow marks elevation only; most elements sit flat, with the reason for each lifted element written down (R-12).
- **Glow on cards, buttons, icons, badges, borders at once.** Fix: glow only as a focus accent on 1 or 2 elements (R-13).
- **Background grid, blueprint lines, dot patterns.** Fix: only if it supports the product's identity, with the reason written down (R-07). A real identity motif beats a stock grid.
- **Dark mode for no reason.** Fix: choose the theme from brand and audience (developer and creative tools have legitimate reasons for dark); otherwise build a working toggle (R-21, R-34).
- **5+ colors.** Fix: 2 or 3 core colors plus one accent, one core being the neutral base (R-29).
- **Accent on everything** (buttons, icons, badges, links, lines, glows). Fix: the accent belongs at the key moment only; zero accents is sterile, everywhere is slop.
- **Sterile default** (flat white, thin grey borders, small radius, generic font, no identity). This is a direction problem, not a filter problem: add `DESIGN.md` or resolve the Design Read, then raise the dials. The fix is never more bans (R-37).

## Layout and components

- **Monotonous template page** (hero, subtitle, 2 CTAs, screenshot, feature grid, testimonials, FAQ, CTA, footer). Fix: structure from content; no testimonials means no testimonials section; section order follows the product's story and the RHYTHM dial (R-05, C-3).
- **Copy-paste feature cards.** Fix: vary size and treatment by content weight (a flagship full-width, supporting features as a list); not every feature is a card (R-14).
- **Bento grid.** Fix: only when the content has elements of genuinely different sizes; otherwise a simple grid or list is more honest (R-05).
- **Uniform spacing.** Fix: a spacing scale used at different levels to separate and connect; uniform rhythm only when the dial says 1 (R-05).
- **"How it works" always 3 steps with round icons.** Fix: show the real process with its real number of steps (R-05).
- **"Trusted by" logo bar.** Fix: only real, verifiable logos; none otherwise (R-18, R-36).
- **"Most popular" middle pricing tier, always three columns.** Fix: as many tiers as exist; highlight the one that serves the product's goal with a stated reason, or none (R-05, R-31).
- **Demo without a product** (promises only; no Terms or Privacy). Fix: show the real product working or say honestly it is not shipped; if the page takes signups or payment, Terms and Privacy must exist (R-38).
- **4-column template footer.** Fix: link groups the product actually has (R-05).
- **Uniform section rhythm** (centered title + subtitle + identical grid every time). Fix: vary composition per the RHYTHM dial: text-heavy and visual, symmetric and asymmetric (R-05).

## Decorative elements

- **Generic AI icons** (sparkle, star, magic, lightning, diamond, cube, robot, orb). Fix: relevant icons with the relevance written down, or none; the label does the work (R-04).
- **One recognizable icon library look** (thin-stroke rounded, Lucide or a clone). Fix: pick for relevance first, then decide whether the library's weight suits the brand (R-04).
- **Emoji as decoration** (a rocket in a headline, a check beside every bullet, fire on a CTA). Fix: remove from UI text; use a relevant icon or nothing (R-04).
- **Arrows on every button.** Fix: only where a direction cue helps, sized proportionally, with the purpose written down (R-08).
- **Colored left stripe on cards or rows.** Fix: it must mark real state (active, warning, new) or go (R-01, R-31).
- **AI capsule badges** ("AI Powered", "Beta", "New"; pill + border + glow + dot + uppercase). Fix: badges only for a real status, never the full combination (R-09).
- **Eyebrow badge above the H1** repeating what the headline says (for example "The platform for teams"). Fix: cut it; fold any real information into the headline or subheadline (R-09).
- **Decorative status dot** that glows or pulses and marks no state. Fix: a dot marks a real state, with no glow and no endless pulse; otherwise remove it (R-19, R-31).
- **Generic AI typography** (large monospace headings, wide-tracked uppercase labels, the default roster Inter, Geist, Space Grotesk, Geist Mono, JetBrains Mono, Fira Code). None is banned; each needs a brand reason, written down (R-06).
- **Fake terminal window** as a hero or feature visual. Fix: a real screenshot of the real product; a real CLI may show a real terminal (R-06, C-5).
- **Generic illustrations** (Undraw, Storyset, 3D blobs). Fix: a direct connection to the product, written down, or real screenshots, or none (R-22).

## Structural and flow

- **Dead navigation.** Every nav item needs a real destination; omit unbuilt features or label them "Coming soon" (R-24).
- **Non-functional controls** (buttons that do nothing, dropdowns that do not open, forms that cannot submit). Every control works or is removed; a deliberate placeholder needs `// TODO` plus a visible label (R-26).
- **Sections that fill a template.** Every section earns its place from the product's content; fewer, purposeful sections beat all the defaults (C-3).

## App and dashboard

The same rules in app-side shapes.

- **Default dashboard shell** (sidebar, top bar, four stat cards, a chart, a table). Fix: name the screen's job and the one decision the user makes on it, and build the hierarchy around that; if the job is "spot the failing job and retry it", the failing jobs are the page and the stat row is a footnote (C-3, R-20).
- **Stat cards with invented numbers and deltas.** Fix: real data or none; a delta only when the comparison period is real and named; prototypes label values as placeholders visibly (R-17, R-38).
- **Filler activity feed** (invented names, rotating avatars). Fix: real events, or an honest empty state that says what to do first (R-18, R-27, R-38).
- **Charts without a question.** Fix: write the question the chart answers and put it in the title ("Failed jobs per hour, last 24h"); if a sentence answers it better, write the sentence; segments need 3:1 contrast (C-3, R-25).
- **Generic table columns** (Name, Status, Date, Actions, a three-dot menu on every row). Fix: columns from the decision made in this table, the deciding field early; row-menu items must exist (R-26).
- **Filler data in fields and columns** (`John Doe`, `johndoe@example.com`, "Let's build something"). Fix: leave cells empty or use honest placeholders (`Your Name`, `email@example.com`, `[REAL DATA]`); generic filler copy is buzzword slop (R-16, R-23, R-38).
- **Placeholder empty and loading states** ("No data available" plus an illustration, a bare spinner, a skeleton the data never fills). Fix: an empty state says why it is empty and gives the one action that fills it; a loading state says what is loading; an error state says what failed and what to do next. First run, filtered-to-nothing, and permission-denied are different screens (R-27).

## Motion

- **Endless pulses and loops** with no user trigger. Fix: motion has a written UX purpose and guides attention to a moment; with MOTION 1 (hover only), a loop fails the declared dial (R-19).
- **Template animations stacked** (fade up, fade in, float, scale, bounce on everything). Fix: choreograph to a purpose and to the dial; the hero speaks, supporting elements stay calm; a claimed "cinematic" page must actually move and a claimed "static" page must not (R-19).

## UI checklist (supplement to the core Delivery Gate)

Report each as PASS/FAIL with evidence; say what you could not verify.

- [ ] Palette comes from `DESIGN.md` or a written identity, not the default gradient set; accent used at the key moment only (R-01, R-29)
- [ ] No decorative emoji, eyebrow pill, or decorative status dots (R-04, R-09, R-31)
- [ ] Section compositions vary per the declared RHYTHM dial; no bento, fake terminal, 3-column pricing, or meaningless left stripes by default (R-05, R-01)
- [ ] Every nav item and control has a real destination or behavior, or a visible "Coming soon" (R-24, R-26)
- [ ] Motion follows the MOTION dial and has a written purpose, with no endless loops (R-19)
- [ ] Glass, glow, shadow, radius within dose caps (R-10, R-11, R-12, R-13)
- [ ] App screens are built around the user's decision, not the sidebar-stats-chart-table default (C-3, R-20)
- [ ] Every number, delta, feed entry, and table row is real or a labelled placeholder; empty fields carry honest placeholders (R-17, R-18, R-23, R-38)
- [ ] Empty, loading, and error states name the cause and the next action (R-27)
- [ ] Holds up at every breakpoint, in every shipped theme, and with keyboard only (R-03, R-34, C-4)
