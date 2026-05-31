# Workout Coach App — Onboarding Flow

> Onboarding has one job: **populate the memory layers** (identity, constraints,
> goals) so the very first coaching interaction already feels personal. It is
> also the user's first taste of the product thesis — *you can see and shape
> what the model knows about you.*

See `architecture.md` for the data model these screens write to.

## Design principles

1. **Every screen writes to a memory row.** No throwaway questions. If we ask
   it, the coach uses it.
2. **Show the payoff early.** End onboarding with a real, tailored coaching
   moment — not a "you're all set!" dead end.
3. **Constraints are sacred.** The injury step gets extra care; it's the
   safety layer and the trust moment.
4. **Legible from minute one.** Make it visible that answers become an editable
   profile the user controls — set the expectation that this is *their* model.
5. **Short, momentum-first.** Ask the few things that change the first response;
   defer the rest to "you can add this anytime."

## Flow overview

```
1. Welcome / promise
2. Sign in with Apple
3. Who are you        → profile (identity)
4. How you train      → profile (persona + tone)
5. Equipment & schedule → profile
6. Injuries & limits  → constraints   ⚠ the trust moment
7. Your goal          → goals
8. "Here's what I know about you"  → review screen (the legibility reveal)
9. First coaching moment → live, tailored response
10. Soft paywall intro (trial framing, no card yet)
```

## Screen-by-screen

### 1. Welcome / promise
- **Purpose:** State the wedge in one line.
- **Copy direction:** "A coach that actually knows you — your goals, your body,
  your history. And you can see and edit everything it knows."
- **Writes:** nothing.

### 2. Sign in with Apple
- **Purpose:** Auth + create the user record.
- **Writes:** `user`.
- **Note:** single tap; no email/password forms.

### 3. Who are you (Identity)
- **Asks:** age, training experience level.
- **Writes:** `profile.age`, `profile.experience_level`.
- **Why first:** cheap, non-threatening, immediately shapes tone (a program for
  a 45-year-old advanced lifter differs from a 22-year-old beginner).

### 4. How you train (Persona + voice)
- **Asks:** training style and how you want to be coached.
  - e.g. "How hard do you go?" → cautious / balanced / **runs into the fire**
  - e.g. "How should your coach talk to you?" → gentle / straight / no-nonsense
- **Writes:** `profile.training_persona`, `profile.tone_preference`.
- **Why it matters:** this is the step that later makes the coach *sound* like
  it gets you. It's the personality dial.

### 5. Equipment & schedule
- **Asks:** where you train (full gym / home rack / minimal) and days/week.
- **Writes:** `profile.equipment_access`, `profile.schedule`.
- **Why:** prevents the coach from prescribing things you can't do.

### 6. Injuries & limits (Constraints) ⚠ the trust moment
- **Asks:** any injuries, pain, or movements to avoid. Free-text plus optional
  body-area tags. "Anything I should work around to keep you safe?"
- **Writes:** `constraints` (one row per issue: `body_area`, `description`,
  `movements_to_avoid`, `severity`, `status`).
- **Design care:**
  - Reassure: "I'll factor this into every single session." (And mean it —
    architecturally this layer is injected on every request.)
  - Make "none right now" a first-class, one-tap answer.
  - Frame as care, not paperwork. This is where trust is won or lost.

### 7. Your goal (Goals)
- **Asks:** the primary thing you're driving toward. Offer a metric where
  natural (lift X, lose Y, train consistently) but allow qualitative goals.
- **Writes:** `goals` (`description`, optional `metric`/`target_value`/
  `current_value`/`target_date`, `priority = primary`).
- **Keep it to one** primary goal in onboarding; more can be added later.

### 8. "Here's what I know about you" — the legibility reveal
- **Purpose:** The signature screen. Show the assembled profile back to the
  user as clean, editable cards: Identity · How you train · Equipment ·
  Injuries · Goal.
- **Interaction:** every card is tappable → edits the underlying row.
- **Copy direction:** "This is your coach's understanding of you. It's yours —
  edit it anytime, and your coaching changes with it."
- **Why it's the heart:** this is the product thesis delivered as an experience,
  not a claim. It also previews the always-available "About Me" surface.

### 9. First coaching moment
- **Purpose:** Pay off the whole flow with a real, tailored response.
- **Interaction:** prompt the user with something like "Ask me what to train
  today" (or auto-generate a first suggestion). The backend assembles context
  from everything just captured and streams a genuinely personalized reply that
  visibly references their persona, constraints, and goal.
- **Why:** the "it knows me" wow moment must happen *inside* onboarding, before
  any paywall.

### 10. Trial framing (soft paywall)
- **Purpose:** Set expectations, not extract a card.
- **Copy direction:** "You're on the free plan — 3 months free, then $2.99/mo.
  No card needed today."
- **Note:** keep the actual StoreKit/RevenueCat purchase out of onboarding;
  introduce it contextually later. Maximize time-to-value first.

## What we deliberately defer

To protect momentum, these are *not* in onboarding — they live in the always-on
profile and are surfaced via gentle prompts later:

- Detailed training history / past programs
- Body metrics beyond what a goal needs
- Multiple/secondary goals
- Nutrition (if ever in scope)

## Success criteria

- Time to first tailored coaching response: **under ~2 minutes.**
- Onboarding completion produces a non-empty `profile`, at least the
  `constraints` decision (even if "none"), and one primary `goal`.
- The user has seen — and ideally touched — the editable "what I know about you"
  surface before finishing.

## Open questions for refinement

- How much of step 4 (persona/voice) is preset chips vs. free text? Free text is
  richer for the model but higher friction.
- Should the "first coaching moment" be a fixed prompt or fully open-ended?
- Do we capture a baseline lift number in onboarding (helps goal tracking) or
  defer to first logged session?
