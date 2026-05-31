# CLAUDE.md — Workout Coach project

Guidance for working in this repo. Read before designing or editing any
prototype.

## What this is

An LLM-powered weightlifting coach for iOS (currently explored via HTML/CSS
prototypes in `prototypes/`). The product wedge is **legibility and personal
experience**: the user can see and edit what the model knows about them. See
`docs/architecture.md` and `docs/onboarding-flow.md` for the full thinking.

The owner is a UX designer. He is the primary user and the design authority.
His lived experience as a 45-year-old serious lifter is the spec — defer to it
over generic fitness-app conventions.

## Design principles (learned from the owner — do not violate)

These come from real gym use. They override default UI instincts.

1. **Legible from 6–8 feet away.** The phone is frequently on the floor, not in
   hand, during a set. Core workout numbers must be readable at a glance from a
   distance. This is the dominant constraint for the logging screen — when in
   doubt, favor legibility over density or polish.

2. **Warmups matter and must show reps and sets.** Do not collapse, hide, or
   reduce warmup sets to a summary strip. Their reps/sets are real information
   the lifter needs. Express them — just slightly less boldly than the working
   sets (a hierarchy of emphasis, not omission).

3. **Don't oversize the working set.** The earlier "huge bold working set" was
   too large. Big enough to read from across the room, not dominating.

4. **No per-set check-off interaction.** Lifters count their own sets in their
   head — people are perfectly capable of tracking how many sets they've done.
   Do NOT add checkboxes, status dots, or tap-to-complete affordances per set.
   That's unnecessary noise. Sets are a display of information, not a checklist
   to tick.

## Working norms

- These are embodied preferences from real training. When a design choice
  touches how the screen is used mid-workout, ask rather than assume.
- HTML/CSS prototypes are feel-tests for layout and legibility, not the native
  app. The real app is SwiftUI.
- Default branch is `master`. The repo is published via GitHub Pages from
  `master` / root; `index.html` is the landing page.
