# Deckfarer — Build Order

A phased, checklistable plan for building the game in **small, individually verifiable chunks**. Work top to bottom. **Never advance to the next phase until the current phase's verification is green and committed.** Each phase ends at a **tag** that is a known-good rollback anchor.

Companion docs: `DESIGN_SPEC.md` (what to build), `ART_ASSETS.md` (deferred visuals).

---

## The rules (read once, follow always)

1. **One task = one small commit.** Every checkbox below is a commit's worth of work.
2. **One phase = one tag**, applied only after that phase's **Verify** block passes. That tag is your rollback point.
3. **Never advance on red.** If the Verify block fails, fix or roll back — do not build on top of a broken phase.
4. **Tests are permanent.** Every phase adds regression tests (mostly *golden transcripts*) that must stay green in *all* later phases. A later phase breaking an earlier test is a bug to fix now, not later.
5. **Definition of Done for every task:** code written · its test green · full suite green · committed.

---

## Verification backbone: golden transcripts

The engine is headless and deterministic, so verification is cheap and exact:

- **Seed the RNG.** Same seed → same shuffles, same draws, every time.
- **Script the inputs.** A fixed list of "play card X, target Y, end turn."
- **Snapshot the output.** Record the exact state log / plain-text transcript to a file.
- **Assert it never changes** unless you meant it to. A diff = a regression, caught instantly.

This is the same seeded-RNG mechanism that later powers exact save-resume (Phase 6). Build it once in Phase 0.

---

## Rollback strategy (git)

**Recommended: branch-per-phase.** Do each phase on a branch (`phase-2-resolver`); merge to `main` and tag only when its Verify block is green. `main` is therefore *always* playable/known-good.

**If something breaks, in order of least to most destructive:**
- Undo uncommitted mess: `git restore .`
- Undo last commit, keep the edits: `git reset --soft HEAD~1`
- Reverse one specific bad commit: `git revert <sha>`
- Roll a single data file back to a good state (the data-driven payoff): `git checkout <tag> -- path/to/file.json`
- Nuke back to the last good phase: `git reset --hard <tag>`  ⚠️ destroys uncommitted work

**Golden rule:** if `main` is tagged and green, you can always get back to a working game with one command.

---

## Phase 0 — Foundation & safety net
Goal: an empty project that can run one green test and is under version control.

- [ ] `git init`; first commit before any real code
- [ ] **Decide the stack.** *Recommended: TypeScript* — one language for headless engine *and* web UI makes the Phase-7 transition seamless. (Phases below are language-neutral; pick a test runner too, e.g. Vitest/Jest.)
- [ ] Test runner set up; a trivial `assert(true)` test runs green
- [ ] `docs/` holds `DESIGN_SPEC.md`, `ART_ASSETS.md`, this file
- [ ] Seeded-RNG utility (deterministic, injectable) + a test proving same seed → same sequence
- [ ] Golden-transcript harness scaffold (record-to-file + assert-against-file), proven on a dummy string

**Verify:** test command runs, all green · `git log` shows small commits · seeded RNG is reproducible.
**Tag:** `v0.0-foundation`

---

## Phase 1 — Data schema & fixtures
Goal: the game's nouns exist as types + a tiny hand-authored fixture set. No engine logic yet.

- [ ] Types for `Card, Requirement, Challenge, StateDelta, Enemy, NPC, Player, RunState` (per Spec §12)
- [ ] Fixture cards (~6): a Might strike, a Charm card, `[Read]`, `[Leverage: X]`, a `Wound` junk, one Tech card
- [ ] One fixture `Enemy` + one combat `Challenge`
- [ ] One non-combat `Challenge` with `footing/footing_max`, hidden `hooks[]`, and `outcome_bands[]`
- [ ] Loader + schema-validation for all fixtures

**Verify:** validation test passes · every fixture parses · malformed fixture is rejected with a clear error.
**Tag:** `v0.1-schema`

---

## Phase 2 — Headless resolver core (the crux)
Goal: the single shared operation from Spec §2, fully tested, no I/O.

- [ ] `play_card(card, challenge, target) -> StateDelta[]`
- [ ] Apply each `StateDelta` type to state (`reduce_hp`, `add_block`, `shift_footing`, `apply_status`, `add/remove_card`, `add_junk`, `reveal_requirement`, `unlock_exit`)
- [ ] Requirement checks: `suit_threshold` (accumulate) and `keyed` (e.g. `leverage:...`)
- [ ] `[Read]` → `reveal_requirement` on hidden hooks
- [ ] Reject invalid plays (not enough energy, wrong target) without corrupting state

**Verify:** unit test per delta type · unit test per requirement type · **golden test**: fixed challenge + fixed card sequence → exact `StateDelta[]` and exact final state.
**Tag:** `v0.2-resolver`  ← *heaviest test coverage of the whole project lives here.*

---

## Phase 3 — Combat loop, plain text
Goal: one full fight, terminal-only, no UI.

- [ ] Turn structure; **3 energy/turn**, reset each turn
- [ ] Draw a hand via seeded shuffle
- [ ] Enemy telegraphs intent, then acts
- [ ] Plays route through the Phase-2 resolver (no new resolution logic)
- [ ] Win at enemy HP 0 · lose at player HP 0
- [ ] Big Might swing shuffles junk into deck
- [ ] Plain-text renderer + scripted-input driver

**Verify:** golden transcript of a full **win**; golden transcript of a full **loss** · manual: play it in the terminal.
**Tag:** `v0.3-combat`

---

## Phase 4 — Non-combat / Footing loop (same resolver)
Goal: prove **one engine** handles talking and locks. *This is the thesis of the whole design.*

- [ ] Full relevant deck revealed (no draw)
- [ ] Footing meter: start value, min 0, max; up on good plays, down on bad
- [ ] Hooks hidden → revealed by `[Read]`; hitting a hook swings Footing hard
- [ ] `outcome_bands`: pass vs. pass-well (mint `[Leverage]` / bonus card / shortcut)
- [ ] Footing→0: **social converts to combat** (reuse Phase 3); **fixed challenge** locks out with a consequence (junk/alarm/closed route)
- [ ] An NPC can play your `Loose Thread` against you (−Footing)

**Verify (the key acceptance test):** golden transcripts for (a) Cipher talks past a guard → pass-well, mints Leverage; (b) blind Enforcer drains Footing → converts to combat; (c) a locked-door fixed challenge · assert the **same resolver code path** serves all three (no forked logic).
**Tag:** `v0.4-social`

---

## Phase 5 — Run structure & Act I skeleton
Goal: encounters strung into a short, branching, playable run.

- [ ] Act model: opener · pool · boss · rest_points
- [ ] Sequencer with **light branching** (pick 1 of 2)
- [ ] Rest sites: heal · **remove a card** (prominent) · upgrade · (maybe) add
- [ ] Permadeath: death ends the run
- [ ] Wire fixtures into a 6–8 encounter skeleton with per-encounter framing prose

**Verify:** golden transcript of a full run (opener → branch → rest → boss → win) · death mid-run ends cleanly · remove-a-card actually shrinks the deck (assert count) · manual playthrough.
**Tag:** `v0.5-act1-skeleton`

---

## Phase 6 — Suspend-save & resume
Goal: quit-and-resume that can't be exploited.

- [ ] Serialize `RunState` incl. `rng_state` to a save
- [ ] Load/resume from save
- [ ] Death invalidates/deletes the save (no reload-to-before-death)

**Verify:** golden test — run N steps, save, reload, continue → **byte-identical** to an uninterrupted run · after death, save is gone and a new run is forced.
**Tag:** `v0.6-save`

---

## Phase 7 — Web UI on top (desktop)
Goal: hang the hybrid interface on the *unchanged* engine.

- [ ] Prose story window (top)
- [ ] Card hand render + play interactions
- [ ] HUD: energy/Footing pips, deck/discard counts, enemy/NPC intent
- [ ] UI events → same engine API; UI only **reads** engine state + **sends** inputs
- [ ] Stand-in card frames per suit (CSS/SVG, no art) per `ART_ASSETS.md`

**Verify:** **all headless golden transcripts still pass unchanged** (proves the engine wasn't touched) · manual: play a full encounter in the browser.
⚠️ If this phase requires editing the engine, an earlier abstraction leaked — stop and fix it there.
**Tag:** `v0.7-ui`

---

## Phase 8 — Content & the other two classes
Goal: a real Act I with build variety.

- [ ] Flesh the Act I pool to 8–12 encounters (mix combat / social / fixed)
- [ ] Complete the **Cipher** deck + resonant pools (fully playable end to end)
- [ ] Stub **Enforcer** & **Fixer** starting decks
- [ ] More junk / Leverage / world cards for variety
- [ ] Light balance pass

**Verify:** two full runs demonstrably build *differently* (different acquisitions/paths) · Cipher run is completable start to finish · full regression suite green.
**Tag:** `v1.0-act1`

---

## Working with Claude Code, per task
Hand it **one checkbox at a time**: implement → run the suite → show the diff → commit. Resist "do the whole phase." The point of bite-sized commits is that when something breaks, the blast radius is one small change and one `git revert` away.
