# Workout Coach App — Architecture

> An LLM-powered weightlifting coach for iOS. The product's wedge is
> **legibility and personal experience**: the user can see, understand, and
> edit what the model knows about them, and the coaching is genuinely tailored
> as a result.

## 1. Product thesis

Most AI fitness apps hide the model's understanding of the user behind a chat
box. This app makes that understanding the centerpiece. The user maintains a
structured, editable picture of who they are, what they're working toward, and
what they must avoid — and the coach uses that picture on every interaction.

The differentiation is not "an LLM in a fitness app." It is **a beautiful,
editable interface to the model's memory**, with coaching layered on top.

## 2. Platform & stack

- **Client:** Native iOS, **SwiftUI**. Built "right" as a real App Store
  binary. Local persistence via **SwiftData**.
- **Auth:** **Sign in with Apple** (best privacy story, lowest friction,
  expected for a portfolio-grade app).
- **Backend:** Serverless (recommended: **Supabase** — bundles Postgres, auth,
  and edge functions). Holds the LLM API key, assembles prompts, enforces usage
  caps. Alternatives: Cloudflare Workers, Vercel functions.
- **LLM:** Accessed server-side via API. Default to a **managed key** (the app
  never holds it). "Bring your own API key" is a later, optional power-user
  feature.

### Why a backend is non-negotiable

The LLM API key must never ship inside the app. The backend is where the key
lives, where the per-request context is assembled, and where daily usage caps
are enforced. Even in a future "bring your own key" mode, the server still
verifies users and centralizes prompt logic; the user's key would live in the
iOS Keychain and be passed per request.

## 3. System shape

```
┌─────────────────────────┐
│   iOS app (SwiftUI)      │
│  • Onboarding flow       │
│  • Workout logging UI    │
│  • Coaching / chat       │
│  • Sign in with Apple    │
│  • Local store (SwiftData)│
└───────────┬─────────────┘
            │  HTTPS (user session token, NOT an API key)
            ▼
┌─────────────────────────┐
│   Backend (Supabase)     │
│  • Verifies user (Apple) │
│  • Holds LLM API key  🔑 │
│  • Assembles context     │
│  • Rate limit / usage cap│
│  • Prompt caching        │
│  • Authoritative memory  │
└───────────┬─────────────┘
            │  HTTPS + API key
            ▼
┌─────────────────────────┐
│   LLM API (Anthropic/etc)│
└─────────────────────────┘
```

## 4. The core architectural principle

**Do not store memory as a chat transcript. Extract it into structured fields
and inject them deterministically on every request.**

A long conversation as "memory" is lossy (it forgets), unsafe (it may forget
the injury that matters most), expensive (you resend the whole transcript every
call), and illegible (the user can't see or edit a blob of chat). Structured,
deterministically-injected memory fixes all four — and the same mechanism
delivers cost control, safety, and the legibility thesis at once.

## 5. Data model

Five layers of memory, each of which is **also a screen the user can open and
edit**.

```
IDENTITY ───────► who you are, how you train, your voice        (stable)
CONSTRAINTS ────► injuries & limits — never forgotten, safety   (protected)
GOALS ──────────► what you're driving toward                    (evolving)
HISTORY ────────► what you actually did                         (append-only)
EPISODIC ───────► what just happened / recent context          (expiring)
```

### `profile` — Identity (the voice)
| field | example |
|---|---|
| `age` | 45 |
| `experience_level` | advanced |
| `training_persona` | "runs into the fire, goes hard, wants to lift more" |
| `tone_preference` | direct, intense, no hand-holding |
| `equipment_access` | full gym |
| `schedule` | 4 days/week |

Screen: **About Me.** This is what makes the coach talk to you like it knows you.

### `constraints` — Safety (the protected layer)
| field | example |
|---|---|
| `body_area` | left shoulder |
| `description` | impingement, flares on heavy overhead |
| `movements_to_avoid` | barbell OHP, behind-neck |
| `severity` | monitoring / active / resolved |
| `status` | active |

Screen: **Injuries & Limits.** Injected on **every** request, no exceptions.
This is a row in a table, not a hope that the model remembers — which is what
guarantees the coach never programs around a forgotten injury.

### `goals` — Direction (evolving)
| field | example |
|---|---|
| `description` | add 20 lb to squat |
| `metric` / `target_value` | squat 1RM / 315 |
| `current_value` | 295 |
| `target_date` | fall 2026 |
| `priority` | primary |

Screen: **Goals** — editable, so "my plan changed" updates a row and the next
response changes accordingly.

### `workout_sessions` + `sets` — History (proof of progress)
```
session:  date · perceived_effort · how_it_felt · notes
  └─ set: exercise · weight · reps · rpe
```
Screen: workout logger + history view. Feeds the long-term arc — the model and
the user can see that this week's squat beat last month's.

### `coaching_interactions` — Episodic + audit trail
| field | purpose |
|---|---|
| `user_message` / `assistant_message` | the exchange |
| `context_snapshot_used` | exactly what memory was fed in |
| `model_used`, `tokens` | cost tracking |

`context_snapshot_used` powers a rare legibility feature: tap any past reply and
see "here's what the model knew about me when it said this."

### `usage_counters` — cost cap
`user_id · date · interaction_count` — enforce N coaching calls/day server-side.
Invisible to the user unless they hit the cap.

## 6. Per-request context assembly

The backend builds a compact, mostly-stable prompt from the tables above on
every call:

```
[SYSTEM]
You are a strength coach for this athlete. Honor their voice and ALWAYS
respect their constraints.

IDENTITY: 45yo, advanced, "runs into the fire," wants intensity and to lift
  more. Tone: direct, no hand-holding.
CONSTRAINTS (never violate): left shoulder impingement — avoid barbell OHP
  and behind-neck pressing.
GOALS: primary — squat 295→315 by fall.
RECENT: last session (3 days ago) squat 5x5 @ 275, felt strong, RPE 8.

[USER]
"What should I do for legs today?"
```

A few hundred tokens, mostly stable between calls → **prompt-caches cheaply**,
**enforces safety**, and **is fully legible** because every line traces to an
editable row.

### Interaction modes
- **Conversational coaching** — free text, streamed token-by-token for a live feel.
- **Structured generation** (a plan, a weekly split) — use the model's
  structured-output / tool-use to return clean JSON the UI renders natively.

## 7. Data placement

- **On-device (SwiftData):** history, in-progress logging, cached profile —
  fast, offline at the gym, strong privacy.
- **Server (Supabase):** authoritative profile / constraints / goals (so
  context assembly and caps run where the key lives), usage counters,
  interaction log.

The **constraints layer is server-authoritative** so a safety rule is never
skipped because a device was offline.

## 8. Cost & pricing

Per substantive interaction (≈3k input + 1k output tokens):

| Model tier | ~per interaction | ~per active user / month |
|---|---|---|
| Haiku-class | ~$0.005 | $0.30–1.00 |
| Sonnet-class | ~$0.02–0.03 | $2–5 |

Mitigations: **prompt caching** (stable context is cached), **model routing**
(Haiku for routine, Sonnet when reasoning matters), **daily usage caps**.

**Pricing model:** freemium → **3-month free trial → $2.99/mo auto-renewing
subscription.** Recurring (not lifetime), because the app has a recurring
per-user cost — lifetime pricing loses money on the most engaged users. Digital
subscriptions must use Apple In-App Purchase (StoreKit); **RevenueCat** is the
standard library to manage it.

## 9. App Store notes

- **Guideline 4.2 (minimum functionality):** thin API wrappers get rejected.
  The onboarding, editable memory, logging, and coaching experience are what
  make this a real app — and they're also the differentiation.
- **Sign in with Apple** for auth.
- **Privacy:** keep as much on-device as possible to simplify privacy labels and
  reinforce the personal ethos.

## 10. Build phasing

1. **Skeleton:** SwiftUI shell + Sign in with Apple + backend that proxies one
   prompt to the LLM and streams a reply. Proves the whole pipe.
2. **Onboarding → context store:** the editable-memory feature (the core bet).
3. **Workout logging + coaching loop** using that context.
4. **Usage caps + polish + TestFlight** (real-device testing at the gym).
5. **Optional:** bring-your-own-key setting; paid tier via RevenueCat.

## 11. Environment note

iOS builds require Xcode on macOS. Cloud/web Claude Code sessions can write
logic, backend code, and docs, but compiling, running the simulator, device
testing, and App Store submission happen in Xcode on a Mac. GitHub is the bridge
between local Xcode work and cloud sessions.
