# Poke Showcase Dex: A Solo PM's Notes from Building an Android App with Claude Code

## TL;DR

**Poke Showcase Dex** is an Android app that helps Pokémon GO players decide which Pokémon to enter into "Showcase" events. PSD opens an overlay over Pokemon go to capture Pokémon's detail screen, the app reads its stats via on-device OCR, and instantly returns a calculated score range, and a ranking against the player's other scans.

I wrote a PRD, used Claude Code for 100% of the implementation — architecture, Kotlin/Compose UI, OCR tuning, a Python data pipeline, and Play Store launch. This doc is a summary of my process and my lessons learned.

---

## The Problem
In Pokémon GO, players leave a selected pokemon at a spot and the largest one wins.
There are some pains with this contest:

1. The largest pokemon is determined by a hidden score, only visible when entering the contest. You have no way of knowing which ones to keep in your limited storage.
2. **Slots are scarce and sticky.** You can only have 5 Pokémon entered across all showcases at once, with no way to withdraw. A bad entry locks up a slot for the entire event.
3. **Winning has a cost.** First-place winners are barred from re-entering that species' showcase for the next two events — so a player who wins with their *only* good candidate has no backup ready.
4. **Limited storage.** There's a finite amount of Pokémon that you can have with upgrades requiring real money. Holding onto low value hurts the bottom line.

Net effect: players enter blind, waste slots on uncompetitive Pokémon, and get caught flat-footed after a win. The app's job is to move the "is this any good?" decision *before* the slot commitment, using nothing but a screenshot.

---

## The Player
I'm building this for the core to hardcore player. These players spend hours in the game, min-maxing where possible. The end goal of winning 100 Showcases is a rare Pikachu variant.

---

## Competition
- Existing tools exist but require manual entry on a websites, which takes time.
- CalcyIV - another overlay app that will provide the estimate range however it does not provide sorting or management of showcases.
- Pokemon Go itself provides this feature during showcases for the selected types. Should they ever release this as persistent feature then this app has no use.

---

## Success
Primary: This tool is a learning experience for me, success is measured in a 0-1 launch in the store.
Secondary: User Engagement - steady growth of users

---

## What I Built
Everything runs **fully offline**
- **Screen scraping details** — A floating overlay captures details directly from the game. On-device OCR extracts the Pokémon's details.  Summary info is displayed to the player in the overlay. 
- **Evaluation screen** — Shows the calculated score, a position gauge against min/max possible score, and ranking.
- **Management** — Easily manage saved information

---

## How I Built It — Tools & Process

| Layer | Tooling |
|---|---|
| Spec | A living PRD, problem statement, goals, non-goals, screen-by-screen requirements with acceptance criteria, success metrics, and a resolved-decisions log. Built with Claude Chat, going back and forth on scope, decisions, and reasoning. |
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
| Account login / reading game memory | Hard line — the app only ever sees what the user explicitly shares |

Every time a "wouldn't it be cool if..." idea came up mid-build (and they did), I checked it against this table first. Most died there, which kept Claude Code's implementation work focused instead of sprawling.

### Phased delivery, PM-defined

The PRD laid out three phases — **Core → Polish → Launch** — with a P0/P1/P2 requirements split and explicit acceptance criteria per requirement (e.g., "OCR parse success rate ≥ 85% across the 20 most common showcase Pokémon"). That gave Claude Code a concrete "done" definition for each unit of work, and gave me a concrete basis for saying "not yet" to scope creep.

---

## Challenges

### 1. OCR tuning

The hardest problems involve defining OCR interpretation. Values aren't consistent, plenty of "if-then" flows, and recognizing non-character values.

### 2. Real usage reshaped the data model mid-build

Early on, the PRD specified a simple "scan history — last 20 scans." Once I started actually using the app, that wasn't the right shape: I didn't want a rolling log, I wanted a **roster I curate**, ranked by my best candidates per type. So:

- "Scan history" was cut entirely in favor of a manual "save" action.
- Later, I added the ability to log the **actual score** the game returned after entering a showcase — and rankings switched to use that real value over the estimate once it's known.
- The original LOW/MED/HIGH "win chance" badge — a P1 feature I'd specced as the casual-user headline — got removed once the type-comparison view existed, because it was strictly less informative and nobody (including me) looked at it.

None of these were planned in the PRD. All of them came from using the thing I was building.

---

## Lessons Learned (for other PMs trying this)

- **Write the PRD like the AI is your engineering team — because it is.** Goals, non-goals, acceptance criteria, and a resolved-decisions log gave Claude Code a stable spec to build against and gave me a fast "is this in scope?" check throughout.
- **Ship to yourself first, then let real usage rewrite the spec.** The biggest product changes (saved roster instead of scan log, actual-score tracking, dropping the win-chance badge) all came from a few days of dogfooding — not from more upfront planning.
- **For "fuzzy" inputs (OCR, ML, anything probabilistic), the product requirement *is* "what happens on failure," not just "what happens on success."** Define the failure UX before the success UX gets built.
- **File issues against yourself.** Even solo, a lightweight issue → branch → PR loop keeps "things I noticed while using it" from turning into untracked scope.
- **Meeting Google Play store test uers launch requirements** wasn't planned for. Needing to find within my circle the 'first users' rather than releasing and just relying on SEO.

---
