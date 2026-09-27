# Lore — Product Specification

> Name: **Lore** · Domain: **tellmylore.com** (see [§14 Naming](#14-naming-candidates))
> Status: Brainstorm → Draft spec · Last updated: 2026-09-27
> Platforms: iOS + Android (cross-platform)

---

## Table of Contents

1. [Vision](#1-vision)
2. [Product Principles](#2-product-principles)
3. [Competitive Landscape & Our Wedge](#3-competitive-landscape--our-wedge)
4. [Core Data Model](#4-core-data-model)
5. [Pillar A — Capture](#5-pillar-a--capture)
6. [Pillar B — Understand (Entities)](#6-pillar-b--understand-entities)
7. [Pillar C — Relive (The Saga & Memories)](#7-pillar-c--relive-the-saga--memories)
8. [Pillar D — Reflect](#8-pillar-d--reflect)
9. [Pillar E — Ask](#9-pillar-e--ask)
10. [Pillar F — Share & Connect](#10-pillar-f--share--connect)
11. [Pillar G — Care (Nudges & Safety)](#11-pillar-g--care-nudges--safety)
12. [Onboarding & The First 30 Days](#12-onboarding--the-first-30-days)
13. [Privacy & Security Architecture](#13-privacy--security-architecture)
14. [Naming Candidates](#14-naming-candidates)
15. [Roadmap](#15-roadmap)
16. [Open Questions](#16-open-questions)

---

## 1. Vision

An AI journal where you **just talk about your day**, and the app quietly turns it into a living map of your life — the people you love, the places you go, the things you do, and how they made you feel.

Traditional journaling fails because it's effortful: blank page, manual tagging, no payoff. Life flips it:

- **Capture is effortless** — voice-first, one gesture, ramble freely, summarize a whole day in one go.
- **Organization is automatic** — AI extracts people, places, activities and mood. Zero manual tagging.
- **The payoff is emotional** — relive moments (nostalgia) and understand yourself (insight) through one shared graph viewed through two lenses.

**One-line pitch:** *Tell it your day. It remembers your life.*

---

## 2. Product Principles

1. **Sharp core loop over long feature list.** "More features than Apple" is explicitly *not* the goal. Capture → Understand → Relive must feel flawless before anything else ships.
2. **Zero setup, instant AI.** Works out of the box (managed backend). BYOK is an optional power-user mode, never a requirement.
3. **The user's words are sacred.** The original entry is never chopped, hidden or rewritten. AI annotates; it never replaces.
4. **Correction is a feature.** Every AI inference is visible and fixable in one tap. Trust comes from fast, satisfying correction.
5. **No guilt, no scores.** No broken streaks, no rating days, no punishment for gaps. Life isn't a game you can lose.
6. **Personal, never generic.** Every nudge, prompt and insight references the user's own life.
7. **Humane by default.** Some memories hurt. The app must know how to stay quiet (Quiet people/places, grief-aware resurfacing).
8. **Privacy is engineered, not promised.** Encryption, minimization and access controls are built in; marketing claims must be literally true.
9. **Delight in the details.** Polished UI, satisfying micro-interactions, beautiful artifacts people *want* to share.

---

## 3. Competitive Landscape & Our Wedge

| App | Strength | Gap we exploit |
|---|---|---|
| **Memex** | AI person/place/event cards, open source | Requires user's own LLM key; no working AI out of the box; "functional, not polished" UI |
| **Diarium** | Auto-pulls photos/calendar/fitness; map + timeline | Map is GPS-only (no place extraction from text); attachments not inline |
| **Journey** | WhatsApp journaling, on-device AI reflections | UI widely called outdated/clunky |
| **Apple Journal** | Deep OS integration | iOS-only; shallow "moments", no real people/places graph |
| **Rosebud** | AI reflective journaling | Memory-reliability complaints; aggressive usage limits |
| **Day One** | Gold standard for classic journaling | No AI insight layer |
| **Time Atlas** | Automatic GPS/movement journal, map + timeline, "Recall" | Zero understanding of text/voice; pure location logging |
| **Lifelog: Tracker Daily Journal** | Broad life tracking (meals, mood, habits), strong downloads | Manual tags only; breadth over depth; no entity extraction |

**Our wedge — nobody combines all three:**
1. AI that works instantly with zero setup
2. Genuine person/place/activity extraction from narrative as the **core browsing experience**
3. Polished, delightful, well-designed UI

**Watch closely:** Time Atlas (closest on map/timeline).

---

## 4. Core Data Model

### 4.1 The three levels

| Level | What it is | Role |
|---|---|---|
| **Entry** | What the user said/typed, in their own words | **Capture unit** — source of truth, owned by the user |
| **Moment** | One thing that happened: who · where · what · feeling · when | **Atomic unit** — every link, stat, pin and citation flows through it |
| **Day** | All moments whose `occurred_at` falls on a calendar date | **Display unit** — one node on the Saga |

**Decision: Moment is the atomic unit.** A user can summarize their whole day in a single entry at night; the AI splits it into moments.

### 4.2 Why moments (vs. entries)

Entity-on-entry breaks as soon as an entry covers more than one event:
- *"Coffee with Priya at Blue Tokai, dinner with Mom at home"* → entry-level tagging links Mom to Blue Tokai. The graph is silently wrong.
- Mood averages out across unrelated events; per-person sentiment becomes meaningless.
- Multi-place entries can't produce a clean map pin.
- Ask can only cite whole entries; stats and rhythms miscount.

Moments give accurate person × place × activity × time × mood links, focused person/place pages, per-moment mood, clean pins, precise citations and correct rhythms.

### 4.3 Span-anchored moments

```
Entry (immutable original audio/text + editable transcript)
  │
  ├── Moment A → span [0–54]   "coffee with Priya at Blue Tokai"
  │     people: [Priya] · place: Blue Tokai · activity: Food › Coffee
  │     mood: 😊 · occurred_at: 2026-09-26 10:00
  │
  ├── Moment B → span [55–98]  "dinner with mom"          🔒 user-corrected
  │
  └── Reflection → span [99–160] "feeling good about the new role"
        mood: 😌 · topic: Work
```

- Every moment points back to the exact words it came from → cards and Ask answers can highlight the source sentence.
- The entry is always viewable in full ("Read full entry").

### 4.4 Handling the hard cases

| Problem | Solution |
|---|---|
| AI splits wrongly (1→2 or 2→1) | Split shown in "What I caught" card; one-tap merge/split |
| User's narrative feels chopped | Moments are an *annotation layer*; the entry is never replaced |
| Re-extraction after an edit wipes corrections | User corrections stored as **locked overrides**; re-extraction only touches unlocked moments and changed spans |
| Pure reflection, no event | **Reflection** moment type (mood + topic, no place/person required) |
| Same moment logged twice (morning "meeting Priya later" / evening "met Priya") | Future-tense statements become **Intentions/Follow-ups**, not moments; dedupe prompt "Same coffee with Priya?" |
| Said now, happened earlier ("yesterday…") | Each moment has its own `occurred_at`, independent of entry `captured_at` |

### 4.5 Entities

| Entity | Key fields |
|---|---|
| **Person** | name, aliases/nicknames, photo, relationship tag, circles, remembered facts, quiet flag, first/last mention |
| **Place** | name, user label ("Home", "Nani's house"), coordinates, type, cover photo, city/country, quiet flag |
| **Activity** | fixed top-level category + emergent sub-activity |
| **Chapter** | title, date range (open-ended allowed), cover, theme/world style, auto-suggested flag |
| **Follow-up** | intention text, expected date, linked person/place, resolved flag |
| **Fact** | subject (person/place), statement, source moment, date learned |

### 4.6 Activity taxonomy (hybrid)

**Fixed top level (~12)** — stable for stats, colors and icons:
Food · Work · Fitness · Travel · Social · Family · Learning · Health · Leisure · Errands · Milestone · Reflection

**Emergent sub-activities** — AI names them as they appear ("Badminton", "Biryani run", "Sunday call with Mom"), personal to each user.

### 4.7 Extraction engine

- **Multi-turn tool-calling agent**, runs **async after instant save** (never blocks the user).
- Tools: `search_existing_people`, `create_person`, `search_existing_places`, `create_place`, `extract_moment`, `record_fact`, `create_followup`, `link_photo_to_person` (later).
- Loop: read entry → resolve each mention against existing entities (search → link or create) → split into moments with spans → extract facts/follow-ups → done.
- **Safeguards:** hard cap of ~4–5 tool-call turns before forcing a final answer; ambiguity surfaced to the user rather than guessed silently.
- Entity resolution ("is this the same Priya?") is the part that genuinely needs agentic judgment.

---

## 5. Pillar A — Capture

> Goal: telling it should feel lighter than *not* telling it.

### 5.1 Entry points (one gesture away)
- Home-screen widget (tap = start recording)
- Lock-screen widget / Control Center
- iPhone Action Button
- Apple Watch / Wear OS complication
- Share sheet (share a photo/link into Life)
- Siri / Google Assistant shortcut ("Hey Siri, tell Life…")

### 5.2 Voice-first UX
- Recording starts instantly — no loading screen.
- Live waveform + live transcript.
- Pause / resume; walk-and-talk friendly; long rambles welcome.
- "Add more to today" — append to the day later.
- Natural date references: "yesterday", "last Saturday", "this morning" → backdated moments.
- **Code-mixed language support** (Tanglish, Hinglish, etc.) and correct Indian names/places. A key differentiator.

### 5.3 Other capture modes
- Text entry (equal citizen, not a fallback).
- Photo + one-line caption.
- Offline capture, queued for sync.

### 5.4 Helping when stuck
- **Personal context prompts** from the user's data: *"You mentioned the launch was today — how did it go?"*, *"First weekend in the new flat?"*
- **Evening "day sketch"** (opt-in photos/location): *"Looks like you were at the office, then near Indiranagar at 8pm. Want to tell the story?"* — the app logs, the user adds meaning.

### 5.5 After capture — "What I caught" card
Appears seconds after save (extraction is async):

> 👤 Priya · 👤 Arjun · ☕ Coffee · 📍 Blue Tokai · 😊

- Tap any chip to fix; merge/split moments.
- Asks about ambiguity **once** ("Same Priya as college Priya?"), then remembers.
- Correcting should feel satisfying, like Google Photos' "same person?" flow.

### 5.6 Editing
- Edit transcript freely; original audio/text preserved.
- **"Tidy my ramble"** — AI cleans up a readable version; original kept.

---

## 6. Pillar B — Understand (Entities)

### 6.1 People
- **Profiles:** photo, aliases ("Amma" = "Mom" = "Lakshmi"), relationship tag.
- **Circles:** auto-grouped by co-occurrence (college gang, work, family), user-editable.
- **Remembered facts:** *"Arjun started at Zoho"*, *"Priya's dad had surgery in May"*, *"Karthik is vegetarian"* — each linked to its source moment, editable/deletable.
- **"Catching up with…" prep:** before a meetup (calendar or on demand) — *"Last saw Arjun 6 weeks ago. He was job hunting; his sister's wedding was coming up."*
- **Person page:**
  - Shared moments on a mini-saga
  - Places you go together, activities you share
  - First mention, last seen
  - "How your time together feels" — a mood strip across moments (**never** a verdict on the person)
  - Remembered facts
- **Management:** merge / split people, one-time disambiguation, Quiet flag.
- **v2+:** CV face-matching; v1 is manual "tap to link photo to person".

### 6.2 Places
- Extracted **from narrative text**, with GPS as a disambiguation hint ("the usual café").
- Auto-detect Home / Work; user labels ("Nani's house").
- **Map:** pins colored by activity; heatmap mode.
- **Place page:** visit count, first visit, who you go with, photos, cover image.
- **Travel map:** unlockable cities & countries (scratch-map vibe) — every new city is a small celebration.

### 6.3 Activities
- Activity page: frequency, rhythm, with whom, where, trend over time.

### 6.4 Showing the links — "Orbits"
The full knowledge graph is never shown (it becomes a hairball). Instead, **ego-centric orbits**:
- Open *Priya* → places and people orbit her, sized by co-occurrence.
- Tap an orbit to narrow: *"Priya × Blue Tokai — 14 coffees since 2024."*
- Every link reads as a small story, carrying sentiment, easy to understand.

### 6.5 Sentiment
- Mood inferred per moment; one-tap override.
- Aggregated as distributions ("mostly warm"), never labels on people.

---

## 7. Pillar C — Relive (The Saga & Memories)

### 7.1 The Saga — one node per day

A Candy Crush-style winding path through your life. Must be **easy to read at a glance, fun to traverse, and guilt-free about gaps.**

#### Day node anatomy (max 3 encodings)

| Encoding | Meaning |
|---|---|
| **Size** | Richness of the day (moments, photos) |
| **Color** (+ shape/emoji for accessibility) | Dominant mood |
| **Center** | Day cover photo, else top activity icon |

Everything else (date, people, details) appears **on tap or scrub** — never cluttering the path.

#### Path elements

| Element | Behavior |
|---|---|
| **Today node** | Glowing frontier with a "you are here" token; fills/grows with a satisfying animation when a memory is added |
| **Quiet days** | Empty days collapse into small stepping stones (`· · · 4 quiet days`). No red, no broken chain. Tap → gentle backfill: "Anything from Tuesday?" |
| **Landmark days** | Bigger, special nodes (first day at job, new city, birthdays, milestones, user-starred). Only nodes labeled on the path |
| **Chapters = worlds** | Each chapter has its own scenery palette; a **chapter gate** (signpost/arch) marks transitions |
| **Trips = islands** | Days away from home city branch onto a side island/bridge that rejoins the main path |
| **Time as scenery** | Month signposts; seasons shift the environment; weather auto-tag adds rain/sun touches |

#### Interactions

| Gesture | Result |
|---|---|
| Tap node | Bottom-sheet day peek: cover, auto-title, moment chips, faces, mood |
| Swipe sheet left/right | Previous / next day |
| Long-press node | Quick-add memory to that day |
| Pinch out | Semantic zoom: days → weeks → months → **chapter world map** |
| Side scrubber | Fast date jump (Google Photos-style) |
| **Lens** (person/place/activity) | Path lights up only on matching days; others dim — "my journey with Priya" |
| Ask citation tap | Camera flies to that day node |

#### Deliberately not borrowed from Candy Crush
- ❌ No stars or scores on days
- ❌ No locked levels, no punishment for gaps
- ✅ Optional list / calendar view for people who prefer it

#### Engineering notes
- Virtualized rendering (thousands of days).
- Reduced-motion mode.
- Layered, themeable scenery art so one illustration system serves all chapter worlds.

### 7.2 Day page
Auto-title (*"Rainy Saturday, biryani with the gang"*), cover, weather, moments, photos, full entries.

### 7.3 On This Day
Resurfaces past days — respects Quiet people/places, favors happy and landmark memories.

### 7.4 Map replay
Animated route of a trip or chapter across the map.

### 7.5 Time capsules
Write to your future self; unlocks on a chosen date. (*Open a letter from your first day at the new job, one year later.*)

### 7.6 Photos
- Cover images for chapters, people, places, days.
- Photos attached per moment.
- **On-device camera-roll suggestions** (matched by time + location, never uploaded unless attached): *"3 photos from this afternoon near Blue Tokai — attach?"*

---

## 8. Pillar D — Reflect

> Insight without judgment.

### 8.1 Chapters
- A new chapter of life: relocating, a new job, getting married.
- Manual creation **or auto-suggested** from data (new place cluster, new people cluster, milestone mention): *"Looks like a new chapter started around March — name it?"*
- Each chapter = a **world** on the Saga (theme, cover, gate).
- **Chapter recap** on close: *214 moments · 38 people · top place · happiest week*.

### 8.2 Rhythms (positive streaks)
Detected from **life events**, not app usage:
- *"Badminton every Wednesday — 7 weeks 🏸"*
- *"Called Mom every Sunday for 6 weeks"*
- Show **longest run** and **"welcome back"** — never a broken chain.
- Optional glowing trail segments on the Saga.

### 8.3 Gentle correlations
- *"Your days with badminton tend to be your brightest."*
- *"Sundays with family are consistently warm."*
- Always phrased as observations, never advice.

### 8.4 Weekly card (Sunday)
*"This week: 3 new places, lots of Priya, a tough Wednesday, a lovely Saturday."*

### 8.5 Year in Life
The year's saga zoomed out; top people, places, landmark days, mood seasons. The most emotional artifact the app produces.

---

## 9. Pillar E — Ask

Graph-RAG agent over the user's own data.

| Type | Example | Answer |
|---|---|---|
| Recall | "Have I ever been to Paradise Biryani?" | "Yes — 3 times, most recently with Arjun and Karthik in March. You loved it." + moment cards |
| Count | "How often did I go to the gym in August?" | Number + rhythm view |
| People prep | "What's going on with Priya lately?" | Recent facts + moments, cited |
| Reflective | "What made me happiest this month?" | Top moments by mood, cited |

**Rules**
- Every claim has a **tappable citation** (deep link to the exact sentence / day on the Saga).
- **"I don't see that in your journal"** is a valid, preferred answer over invention.
- Voice questions supported; suggested questions generated from the user's data.
- Tools are **read-only**, strictly scoped to the user (see §13).

---

## 10. Pillar F — Share & Connect

### 10.1 Postcards — sharing a moment ✉️

> A simple, unique, interactive, emotion-causing way to share a moment with someone.

**The flow (sender):** On any moment or day → **Send as Postcard** → optionally add a note and/or a **short voice clip in your own voice** → choose recipient → send via any messenger (WhatsApp, iMessage, etc.). Three taps.

**The experience (recipient)** — opens in the browser, **no app required**:

1. **"Remember this?"** — a sealed postcard with only a teaser: *the date and a blurred hint*. Builds anticipation.
2. **The reveal** — tap/hold to open: the photo **develops like a Polaroid**, the place and date fade in, then the sender's note appears.
3. **Their voice** — if attached, the sender's voice clip plays: *"Remember when we got lost looking for this place?"*
4. **"Add your side"** — the recipient can reply with **their own memory** of that moment (voice or text). It flows back to the sender and attaches to the original moment as *"Priya remembers…"*. This is the emotional loop — a shared memory becomes two-sided.
5. **React** — one tap: 🥹 🤗 😂 ❤️ — delivered back to the sender.

**Variants**
| Variant | Description |
|---|---|
| **Moment Postcard** | Single moment (default) |
| **Day Postcard** | A whole day as a mini-saga strip |
| **"Our Story"** | All moments with one person as a short, swipeable mini-saga reel — *"You & Priya: 47 moments since 2019."* Perfect for birthdays, anniversaries, farewells |
| **Scheduled Postcard** | Send on a future date — *"Send this to Priya on our friendship anniversary."* |

**Why it's unique:** most apps share a flat image. Postcards are a *moment of reveal* plus a *reply with your side*, which turns sharing into reconnecting.

**Privacy rules for Postcards**
- Explicit, per-share; recipient sees only the curated card (no other moments, no names of third parties unless the sender includes them).
- Revocable at any time; optional expiry; `noindex`; unguessable links.
- **End-to-end encrypted link:** the Postcard is encrypted with a per-share key placed in the URL **fragment** (`#key`), which browsers never send to the server. Our server stores only ciphertext. (Trade-off: link previews in messengers show a generic teaser — which fits the "Remember this?" reveal anyway.)
- Replies ("Add your side") are stored encrypted under the sender's account; the sender can accept/hide them.

### 10.2 Linked moments (app-to-app)
When both people use the app: each keeps their own private entry and can **optionally link** them. *"Priya's side of this day"* becomes viewable if she chooses to share. A Postcard reply can convert into a linked moment when the recipient installs the app.

### 10.3 Share cards (images)
Beautiful exportable image of a day/place/rhythm (names hidden by default) for stories and status.

---

## 11. Pillar G — Care (Nudges & Safety)

### 11.1 Nudge rules
- **Max one per day.** Always personal, never generic. Fully user-controlled per type.
- Push payloads contain **no journal content** (see §13).

| Nudge | Example |
|---|---|
| Capture | At the user's habitual time: *"Anything from today?"* |
| Open loop / follow-up | *"How did the interview go?"* |
| Memory | *"2 years ago today: Goa ✨"* |
| Place anniversary | *"A year since your first day in Bangalore."* |
| Relationship (opt-in) | *"It's been a while since you mentioned Arjun."* |
| Rhythm (positive) | *"Wednesday — badminton day? 🏸"* |

### 11.2 Quiet people & places
Mark a person or place as **Quiet** (an ex, someone who passed away, a falling-out): they stay in your history but **never** appear in resurfacing, nudges, On This Day or recaps. Grief-aware by design.

### 11.3 Private entries
Per-entry toggle: never sent to AI, never extracted, never resurfaced.

### 11.4 Sensitive moments
If an entry signals real distress, respond gently with support resources — never alarming, never preachy. Requires careful design and review.

---

## 12. Onboarding & The First 30 Days

| When | Magic moment |
|---|---|
| **Onboarding** | Voice prompt: *"Tell me about the people closest to you."* → people graph seeded before the first entry. Set home city. Optional import (Day One / Journey exports, photo library) — extraction runs retroactively for a cold-start advantage |
| **First entry** | "What I caught" card within seconds → *"it understood me"* |
| **Day 3** | First person page with 3 moments — the graph visibly forming |
| **Day 7** | First Weekly card; the first stretch of Saga path |
| **Day 30** | Orbits form; first Rhythm detected; On This Day active (from imports) |

---

## 13. Privacy & Security Architecture

**Decision: Full cloud, encrypted at rest — implemented with strict best practices.**

### 13.1 Honest threat model
- **Protects against:** DB dumps, backup leaks, stolen disks, insider snooping, vendor exposure.
- **Does not make us technically unable to read data** — the server must decrypt to run AI. Therefore the promise is backed by engineering controls, and public claims are precise:
  > *Encrypted. Never used for training. No human access. Delete anytime.*
- Stronger "even we can't read it" claim is reserved for confidential computing (§13.10).

### 13.2 Encryption

| Layer | Practice |
|---|---|
| In transit | TLS 1.3 only, HSTS, certificate pinning (public-key pins + backup pins) |
| Disk/DB | Managed encryption at rest — baseline only |
| **Application-level** | Entry text, transcripts, moments, entity names, facts, notes, embeddings encrypted **before** reaching the DB (AES-256-GCM via Google Tink / AWS Encryption SDK — never hand-rolled) |
| **Envelope encryption** | Per-user Data Encryption Key (DEK) wrapped by a KMS/HSM Key Encryption Key; only API + extraction service roles may call decrypt |
| **Crypto-shredding** | Account deletion destroys the user's DEK → data unreadable everywhere, including backups |
| Rotation | Scheduled KEK rotation; re-wrap DEKs |
| Encrypted lookups | Blind indexes (per-user HMAC) for exact matches; per-user isolated vector index; **embeddings treated as sensitive** (text can be partially reconstructed from them) |
| Decrypt scope | Decrypt in-memory only at call time; never persisted in plaintext |

### 13.3 Media
- Private buckets, per-user keys, short-lived signed URLs.
- **Raw audio deleted after transcription by default** (opt-in "keep my voice notes").
- EXIF stripped from stored photos; location stored as a separate encrypted field.

### 13.4 AI processing
- LLM providers only with **zero-data-retention + no-training** contracts.
- **Data minimization:** send only the entry + candidate entity names, never full history.
- No logging of prompts/completions.
- Private entries never enter the pipeline.
- **Hard scoping:** `user_id` comes from the server-side session, **never from the model**. Postgres **row-level security** as defense in depth.
- Ask tools are **read-only** with no outbound/network capability → prompt injection inside entries has nowhere to exfiltrate.

### 13.5 BYOK (optional mode)
- Keys routed through the backend relay (same code path as managed; only the credential source swaps).
- Stored with envelope encryption; decrypted in-memory only at call time.
- User-facing revoke/rotate.
- Never present in logs, crash reports or analytics.

### 13.6 Logging, analytics, crash reporting
- **Allow-listed structured logging** — content fields cannot be logged by construction.
- Crash reports scrubbed (e.g. Sentry `beforeSend` strips bodies/breadcrumbs).
- Analytics = event names only, no content.
- **No session-replay or screen-capture SDKs. Ever.**

### 13.7 Human access
- No production content access by default.
- Break-glass: approval + time-boxed + fully audited.
- Alerts on anomalous KMS decrypt volume (CloudTrail / Cloud Audit Logs).
- Separation of duties between infra and data roles.

### 13.8 Auth & device
- Sign in with Apple / Google; passkeys.
- Short-lived access tokens; rotating refresh tokens; device list + remote sign-out.
- Tokens in Keychain / Android Keystore.
- Encrypted local cache (iOS Data Protection *Complete*; SQLCipher on Android); minimal cache.
- Biometric / PIN app lock; blur in app switcher.
- **Push notifications carry no content** (they transit Apple/Google); details load in-app.

### 13.9 User controls & compliance
- Full export (JSON + media) anytime.
- Real deletion within a stated window (crypto-shred).
- Plain-language "What we store & who processes it" page.
- Explicit consent for AI processing.
- Designed for **India DPDP Act 2023** and **GDPR** from day one (journals contain health, relationship and emotional data).
- Pen test before launch → bug bounty → SOC 2 later.

### 13.10 Future hardening (v2+)
Run extraction & Ask inside **confidential computing** (AWS Nitro Enclaves / GCP Confidential VMs) to credibly claim that even engineers cannot read content.

---

## 14. Naming Candidates

**Decision: Lore**, primary domain **tellmylore.com** (voice-first CTA; ~$10/yr). Optional backups: loremoments.com (possible Postcard share-link domain), tellmylore.app.
*To do: trademark search for "Lore" (IP India / USPTO, classes 9 & 42) and App Store / Play Store name check.*

Candidates considered:

| Name | Why it works |
|---|---|
| **Lore** | "Your personal lore." One syllable, mythic, fits Chapters & Saga |
| **Trove** | A treasure trove of memories; warm, ownable |
| **Ode** | An ode to your days; tiny, poetic |
| **Yaad** | Hindi/Urdu for "memory"; distinctive, resonant for Indian users |
| **Pebble** | Days as stepping stones on the path (note: former smartwatch brand) |
| **Saga** | Directly names the core UI; likely crowded |
| **Kept** | "Everything you've kept." Simple and emotional |
| **Remi** | From *remember*; friendly, name-like |
| **Trail** | The path you've walked; maps to the Saga |
| **Unfold** | Your life unfolding chapter by chapter |

---

## 15. Roadmap

### v1 — The Core Loop
- Voice/text capture with instant save; all quick-capture entry points (widget, lock screen, shortcut)
- Code-mixed transcription
- Async extraction agent → span-anchored moments
- "What I caught" card with one-tap corrections, locked overrides
- People / Place / Activity pages; aliases, relationship tags, manual photo linking
- Basic Saga (day nodes, quiet stones, day peek sheet) + map
- Manual photos & covers
- Private entries, Quiet people/places, app lock
- Full security baseline (§13) — **non-negotiable for v1**
- Export & delete

### v1.5 — It Starts Giving Back
- Chapters (manual) + chapter worlds on the Saga
- Remembered facts & follow-ups
- Nudges: capture, open-loop, On This Day
- Ask (cited answers)
- **Moment Postcards** (reveal, voice clip, reactions, "Add your side")
- Weekly card
- Import from Day One / Journey

### v2 — Delight
- Full Saga: landmarks, trip islands, lenses, semantic zoom, seasonal scenery
- Rhythms & gentle correlations
- Auto-suggested Chapters + chapter recaps; Year in Life
- Orbits visualization
- Camera-roll suggestions; evening day sketch
- "Our Story" & scheduled Postcards; linked moments
- Travel map, map replay, time capsules
- "Catching up with…" prep

### v2+
- CV face matching
- Confidential computing
- Watch apps

---

## 16. Open Questions

1. **Saga direction:** today at the top (scroll down into the past) or bottom (Candy Crush-style climbing up)?
2. **Visual identity:** illustration style for chapter worlds, design tokens, typography, color system for moods.
3. **Moment granularity:** where does one moment end and the next begin? (e.g. "walked to the café and met Priya" — one or two?)
4. **Facts about people:** how much the user sees/controls; retention; sensitivity of third-party information.
5. **Postcard replies:** should replies from non-users be moderated/filterable before attaching?
6. **Distress detection:** exact policy, thresholds, regional resources, review process.
7. **Calendar integration:** in scope for v1.5 (people prep, day sketch) or later?
8. **Brand direction** for Lore (logo, voice, visual identity).
