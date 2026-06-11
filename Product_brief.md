# Poke Showcase Dex: A Solo PM's Notes from Building an Android App with Claude Code

## TL;DR

**Poke Showcase Dex** is an Android app that helps Pokémon GO players decide which Pokémon to enter into "Showcase" events. PSD opens an overlay over Pokemon go to capture Pokémon's detail screen, the app reads its stats via on-device OCR, and instantly returns a calculated score range, and a ranking against the player's other scans.

I wrote the PRD, made every product and prioritization call, and paired with Claude Code for 100% of the implementation — architecture, Kotlin/Compose UI, OCR tuning, a Python data pipeline, and Play Store launch. This doc is a summary of my process.

---

## The Problem
In Pokémon GO, players leave a selected pokemon at a spot and the largest one wins.
There are some pains with this contest:

1. The largest pokemon is determined by a hidden score, only visible when entering the contest. You have no way of knowing which ones to keep in your limited storage.
2. **Slots are scarce and sticky.** You can only have 5 Pokémon entered across all showcases at once, with no way to withdraw. A bad entry locks up a slot for the entire event.
3. **Winning has a cost.** First-place winners are barred from re-entering that species' showcase for the next two events — so a player who wins with their *only* good candidate has no backup ready.

Net effect: players enter blind, waste slots on uncompetitive Pokémon, and get caught flat-footed after a win. The app's job is to move the "is this any good?" decision *before* the slot commitment, using nothing but a screenshot.

---

## What I Built

Two core screens, mapped directly to the user's decision flow — **input, then result**:

- **Select screen** — Import a screenshot (or use a floating overlay to capture directly from the game). On-device OCR extracts the Pokémon's name, height, weight, and XXL status. Individual Attack/Defense/HP IV dropdowns let the player enter values exactly how they'd read them off PokeGenie. Every OCR'd field is editable inline — nothing is silently trusted.
- **Evaluation screen** — Shows the calculated score range, a position gauge against that species' min/max possible score, a per-type comparison (where does this score rank against *every* Pokémon eligible for, say, a Fire-type showcase?), and species/type rankings against the player's own saved scans. Players can later log the *actual* score the game gave them, and the app uses that real value for ranking once it's known.

Everything runs **fully offline** — OCR, scoring, and the Pokémon database are all on-device. No accounts, no servers, no analytics SDKs.

---

## How I Built It — Tools & Process

| Layer | Tooling |
|---|---|
| Spec | A living PRD, problem statement, goals, non-goals, screen-by-screen requirements with acceptance criteria, success metrics, and a resolved-decisions log |
| Engineering | [Claude Code](https://claude.com/claude-code) (Claude Sonnet 4.6) — sole implementer, working from the PRD and from GitHub issues I filed as I dogfooded |
| App | Kotlin, Jetpack Compose, Room (local DB), Android MediaProjection (screen-capture overlay) |
| OCR | Google ML Kit Text Recognition (on-device) + a custom pixel-analysis reader for the IV bar charts (more on this below) |
| Data pipeline | A small set of Python scripts that parse the public PokeMiners "Game Master" dump into a bundled JSON species database (height/weight/type/forms for every showcase-eligible Pokémon, including regional forms) |
| Process | GitHub PRs per feature branch, issues for bugs/polish found during dogfooding, conventional commits |

### The PRD did most of the prioritization work for me

The PRD's **Non-Goals** table ended up being just as load-bearing as the goals:

| Non-Goal | Why it mattered |
|---|---|
| IV calculation | PokeGenie already owns this — don't rebuild it |
| Real-time overlay automation | Screenshot-based flow is simpler, safer, and keeps the app firmly on the right side of Niantic's ToS |
| Account login / reading game memory | Hard line — the app only ever sees what the user explicitly shares |

Every time a "wouldn't it be cool if..." idea came up mid-build (and they did), I checked it against this table first. Most died there, which kept Claude Code's implementation work focused instead of sprawling.

### Phased delivery, PM-defined

The PRD laid out three phases — **Core → Polish → Launch** — with a P0/P1/P2 requirements split and explicit acceptance criteria per requirement (e.g., "OCR parse success rate ≥ 85% across the 20 most common showcase Pokémon"). That gave Claude Code a concrete "done" definition for each unit of work, and gave me a concrete basis for saying "not yet" to scope creep.

---

## Challenges

### 1. OCR is a product decision, not just an engineering one

The hardest technical problem — reading IV values off the in-game appraisal screen's bar charts — turned into a product conversation fast. There's no text to OCR for IVs; it's three horizontal bar charts whose *fill length* encodes a 0–15 value. We ended up with a pixel-by-pixel brightness/saturation classifier that distinguishes "filled bar" from "empty track" from "background."

The PM call here wasn't the algorithm — it was deciding **what happens when OCR is wrong**. Per the PRD: *"If any field fails to parse, it shows as an empty editable input — no silent defaults."* That single requirement shaped the whole Select screen and avoided a much worse outcome: a confidently-wrong score that erodes trust on the very first use.

### 2. Real usage reshaped the data model mid-build

Early on, the PRD specified a simple "scan history — last 20 scans." Once I started actually using the app, that wasn't the right shape: I didn't want a rolling log, I wanted a **roster I curate**, ranked by my best candidates per type. So:

- "Scan history" was cut entirely in favor of a manual "save" action.
- Later, I added the ability to log the **actual score** the game returned after entering a showcase — and rankings switched to use that real value over the estimate once it's known.
- The original LOW/MED/HIGH "win chance" badge — a P1 feature I'd specced as the casual-user headline — got removed once the type-comparison view existed, because it was strictly less informative and nobody (including me) looked at it.

None of these were planned in the PRD. All of them came from a week of actually using the thing I was building — the single highest-leverage thing I did as the PM on this project.

### 3. Pre-launch hardening is its own mini-project

"Polish" and "Launch" in the PRD's phase plan turned out to hide a real checklist: release signing config, stripping debug logging from release builds (so OCR'd scan data can't leak into logs), disabling Android Auto Backup so local history can't leave the device (this directly backs a privacy-policy claim), trimming permissions to the minimum the app actually uses, and excluding tablets from the Play Store listing since the UI isn't tablet-tuned. None of this is visible to a user in a demo, and all of it is required to ship responsibly. Budgeting real time for it (it became its own PR) kept it from being a last-minute scramble.

### 4. Being the entire team

Solo means PM, designer, QA, and "stakeholder sign-off" are all the same person. The thing that made this tractable: filing real GitHub issues against myself during dogfooding sessions (narrow overlay was covering gameplay, evaluation screen needed a back button, etc.) instead of trying to fix things in the moment. That turned "I noticed something annoying" into a backlog Claude Code could pick up cleanly, with traceable commits referencing each issue.

---

## Lessons Learned (for other PMs trying this)

1. **Write the PRD like the AI is your engineering team — because it is.** Goals, non-goals, acceptance criteria, and a resolved-decisions log gave Claude Code a stable spec to build against and gave me a fast "is this in scope?" check throughout.
2. **Non-goals are the highest-leverage section you'll write.** They're what stop a solo project from drifting into "rebuild PokeGenie."
3. **Ship to yourself first, then let real usage rewrite the spec.** The biggest product changes (saved roster instead of scan log, actual-score tracking, dropping the win-chance badge) all came from a few days of dogfooding — not from more upfront planning.
4. **For "fuzzy" inputs (OCR, ML, anything probabilistic), the product requirement *is* "what happens on failure," not just "what happens on success."** Define the failure UX before the success UX gets built.
5. **Treat compliance/privacy/security work as a real backlog item with its own PR**, not a pre-submission afterthought — especially for anything touching screen capture, overlays, or local data.
6. **File issues against yourself.** Even solo, a lightweight issue → branch → PR loop keeps "things I noticed while using it" from turning into untracked scope.

---

## Key User Problems (Recap)

- *"I can't tell if this Pokémon is worth a showcase slot before I commit it."* → Score range + position gauge, computed from a screenshot.
- *"I don't know how I stack up against the field, not just my own roster."* → Per-type comparison against every eligible Pokémon.
- *"I won, and now my best entry is locked out for two events — who's my backup?"* → Saved roster with species/type rankings, refined by real scores once known.
- *"I don't trust an automated number I can't verify."* → Every input is shown, editable, and auditable on the result screen.

---

## What's Next

- **Pokémon database completeness** — base stats and forms are generated from the public PokeMiners Game Master dump via a small Python pipeline; this needs to stay current as Niantic adds species/forms.
- **Manual species override** — a searchable dropdown for when OCR misreads a name.
- **Crash reporting** and broader real-device OCR validation (the IV bar reader's color thresholds were tuned on a limited set of devices/screens).
- **iOS** — the screenshot-first design was deliberately chosen to make a port feasible; not yet scoped.
- **Crowdsourced leaderboard data** — replacing theoretical score ranges with real win-rate percentiles, if there's ever a backend to support it.

---

## Get in Touch

If you're a PM, founder, or hiring manager and want to talk about this project (or about building product with AI-assisted engineering more broadly), reach out: **laggerBP@gmail.com**
