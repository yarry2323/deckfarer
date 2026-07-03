# Deckfarer — Design Spec

*Working title: **Deckfarer**. Rename on theme selection — it's just a folder name and a few doc headers.*

A web-based, **desktop-first** roguelike deck-builder wrapped in an Infocom-style prose skin. **Deck-building and progression are the core experience**; the world/theme is the stage they play out on, not the point. Every capability is a card, one resolver settles every challenge, and structure/mechanics draw from *Slay the Spire*.

**How to use this doc:** source of truth for intent. Section 2 (Resolver) and Sections 11–12 (architecture + data model) are what code is built against first. Section 7 (Run Structure) is central given the deck-building priority. Section 14 lists what's still genuinely open — don't invent answers to those. Art is tracked in `ART_ASSETS.md`; nothing here blocks on art.

---

## 1. The Spine (non-negotiable pillars)

1. **Cards are verbs.** A card is a *capability*, not an abstract attack. Inventory and deck are the same thing: a crowbar is `[Pry]`; a known name is `[Leverage: Name]`.
2. **Verbs are earned.** Cards enter your deck by finding, learning, surviving, or being given them. Your deck *is* what your character knows and carries.
3. **One engine resolves everything.** A vault, a wary guard, and a swordfight are one object — a Challenge with Requirements — in different costumes.
4. **Class is a lens, not a silo.** Class sets starting deck and resonance, doesn't wall off cards.
5. **One persistent, evolving deck (per run).** You curate a single deck across a run; context decides which cards are *useful*, never which are *present*.

---

## 2. The Resolver — the one shared operation

```
play_card(card, challenge, target_requirement) -> StateDelta[]
```

**Challenge** — any obstacle. `requirements: Requirement[]`, `state` (type-specific), `on_resolve` / `on_fail`.
**Requirement** — `type`: `suit_threshold` | `keyed`; `suit`+`threshold`, or `key` (e.g. `leverage:sergeant_name`); `hidden: bool` (fogged until a `[Read]`).
**Card** — `id, name, suit (Might|Wits|Charm|Tech), value, cost, tags[], one_shot, effects[]`.
**StateDelta** — universal output; state applies it: `reduce_hp` · `add_block` · `shift_footing` · `apply_status` · `unlock_exit` · `add_card` · `add_junk` · `remove_card` · `reveal_requirement` …

A Strike lowering enemy HP and a Leverage card raising your Footing against a guard are the **same call** — different Requirement, different Delta, identical machinery.

---

## 3. Suits

| Suit | Meaning | Feel |
|------|---------|------|
| **Might** | force | heavy, loud |
| **Wits** | cunning | sharp, cold |
| **Charm** | influence | warm, social |
| **Tech** | systems | precise, electric |

---

## 4. Cards

**Sources:** starting deck (class) · common world pool · class-exclusive pool · world cards (found/learned) · junk (from consequences).

**Notable cards**
- `[Read]` — reveals a Challenge's hidden Requirements/hooks. Weak in combat, master key in non-combat. A build bet.
- `[Leverage: X]` — one-shot `keyed` card minted at runtime from a world fact. Social capital as ammo.
- **Deployables** — persist on the table and fire each turn (Fixer).

**Junk / consequence cards:** `Heat` / `Fracture` / `Wound` (self-inflicted by big Might swings) · `Bleeding` (from wounds) · `Loose Thread` (from lies; an NPC can play it against you to cost you **Footing**).

---

## 5. Classes (3 at launch)

Each: distinct starting deck, pool resonance, and **one signature mechanic no one else has.**

### The Cipher — TEST CLASS (Charm/Wits)
**Leverage.** Reading people mints one-shot `[Leverage: X]` cards to bypass/collapse encounters. Heavy `[Read]`/draw, thin damage. Fantasy: win before the strike pile appears. Stresses the resolver hardest — build it first.

### The Enforcer (Might)
**Scaling + self-punishment.** Might plays boost later Might cards, but big swings shuffle junk in. Fantasy: kick the door down.

### The Fixer (Tech)
**Deployables.** Gadgets stay on the table and fire each turn. Slow start, snowball finish.

---

## 6. Encounter Model — two loops, one resolver

Only three variables differ; resolver is identical.

| | Combat | Non-combat (social & fixed) |
|---|---|---|
| **Card access** | Draw a hand (~5, shuffled) — RNG tension | Full relevant deck revealed — logic/leverage puzzle, no RNG |
| **What you're driving** | Enemy **HP** down to 0 (+ telegraphed intent) | Your **Footing** up to a success threshold |
| **Hidden state** | — | **Hooks** under fog (requirements that swing Footing hard; lift fog with `[Read]`) |
| **Failure** | Damage / junk | Footing hits 0 → you're out |

### Footing — the non-combat resource
The single resource for **all** non-combat challenges (a wary guard *and* a jammed lock — the term is deliberately physical so it fits both).
- **Fixed starting amount** per challenge.
- **Failed / off-target plays lower it; good plays raise it.** Cards may explicitly add Footing; hitting a revealed hook swings it hard.
- **Hits 0 → you're out:** social **converts to combat** (initiative rolls, draw a hand); a fixed challenge locks you out with a consequence (junk, alarm, closed route).
- **Thresholds gate outcome quality.** A minimum bar passes; a higher bar passes *well* — bonus card, minted `[Leverage]`, a shortcut, extra intel. Built-in push-your-luck: bank the win, or press for the great outcome and risk a slip.

**Symmetry:** combat drives *their* HP to zero; non-combat drives *your* Footing to a threshold. Same resolver, mirrored direction.

### Economy (decided)
- **Combat:** **3 energy per turn** (StS gold standard) funds card `cost`.
- **Non-combat:** no separate spend currency — plays are rationed by their effect on **Footing** (bad plays cost you position). Card `cost` may still gate especially strong plays.

---

## 7. Run Structure — the core loop

A **roguelike run**: a sequence of encounters toward a stated goal ("Save the McGuffin"), pursued by whatever means the deck allows.

**Acts.** A run is one or more Acts. Each Act = ~10–15 encounters. **We build Act I first**, expand later.

**Fixed bookends + shuffled middle.** Each Act opens on an **authored** beat (establishes the goal) and ends on an **authored climax/boss** tied to it. Encounters between are drawn from a **pool**, each carrying **self-contained framing prose** that never assumes what came before. Replayable *and* reads as a unified story.

**Light branching.** At each step the player **picks 1 of 2** next encounters. This choice *is* a deck-building act — a Cipher steers toward social nodes, an Enforcer toward brawls.

**Rest sites — the main stage.** At fixed points (~every 4–5 encounters, ~2–3 per Act) the player chooses among levers:
- **Heal**
- **Remove a card** (purge a `Loose Thread` / `Wound`) — *prominent*; deck-thinning is the core deck-building verb
- **Upgrade a card**
- (maybe) **Add a card**

**Death = roguelike permadeath.** You die, the run ends, you start over. No reload-to-checkpoint.

**Save = single suspend slot.** One save snapshots the run mid-flight so the player can quit and resume later. Not a reload point: consumed/invalidated on death (no save-scumming). Resume must be exact — persist RNG state so the shuffled middle and drawn hands continue identically.

**Meta-progression (between-run unlocks): DEFERRED.** Get one great Act I run working; bolt on later.

---

## 8. Determinism split

Randomness in the fight (drawn hands = tension/replay); determinism at the locked door (full access on logic/social puzzles — RNG there feels awful).

---

## 9. Progression

**The deck is the primary progression vector** — you grow by changing your deck (gaining cards, curating out junk at rest sites), not a separate XP/level tree at launch.

---

## 10. Presentation — hybrid, desktop-first

Prose story window on top (Infocom soul); visible card hand + small HUD beneath (deck/discard counts, energy/**Footing** pips, enemy/NPC intent). Card *state* is shown, not narrated. **Not designed for mobile** — assume desktop/web layout. Three art tiers (chrome → procedural → illustrated); build with stand-ins, defer illustration. See `ART_ASSETS.md`.

---

## 11. Architecture guidance for Claude Code

1. **Build the resolver headless first** — pure logic + data, plain-text I/O, no UI. The engine must not know whether it's rendered as prose or cards.
2. **Everything is data** — encounters, cards, challenges, enemies, NPCs, and the Act I pool are data files fed to the resolver. Authoring content never touches engine code.
3. **Hang the UI on top** — card HUD binds to engine state; DOM reflects emitted deltas.
4. **One code path for both loops** — combat and non-combat differ only by Section-6 variables + draw-vs-reveal + HP-down-vs-Footing-up.
5. **Serialize the whole run for suspend-save** — Player + current Act/encounter index + path taken + deck + HP + **RNG state**. Invalidate on death.

---

## 12. Data model sketch (refine as needed)

```
Act         { id, opener_encounter, boss_encounter, pool[], rest_points[] }
Encounter   { id, kind(combat|social|fixed), framing_prose, challenge, rewards[] }
Node        { id, prose, branch_options[2], state_flags[] }
Card        { id, name, suit, value, cost, tags[], one_shot, effects[] }
Challenge   { id, type, requirements[], state, outcome_bands[], on_resolve[], on_fail[] }
  // combat state: { enemy }
  // non-combat state: { footing, footing_max, hooks[] }   hooks = hidden requirements
Requirement { type, suit?, threshold?, key?, hidden }
Enemy       { id, name, hp, intents[] }
NPC         { id, name, start_footing, hooks[] }
RestSite    { id, options[heal|remove|upgrade|add] }
Player      { deck[], hp, energy, statuses[], run_flags[] }
RunState    { act, encounter_index, path[], rng_state, player }   // = the suspend-save
```

`outcome_bands` map Footing thresholds → outcome quality (pass / pass-well + bonus).

---

## 13. Act I build target (first milestone)

- **1 fixed opener** (establishes the McGuffin goal) + **1 fixed boss** (tied to it).
- **~8–12 pooled middle encounters**, mixing combat and non-combat so the Cipher shines.
- **Light branching** (pick 1 of 2) threading opener → middle → boss.
- **2–3 rest sites**; rest options incl. prominent **remove-a-card**.
- **Card pool deep enough that two runs build differently.**
- **Cipher fully playable**; Enforcer + Fixer may be stubs.
- **Single suspend-save**; permadeath invalidates it; resume exact (RNG persisted).
- Desktop/web layout; **3 energy/turn** combat; **Footing** on non-combat.

---

## 14. Still open (decide as we build)

- **Keyword/status vocabulary** — our own Vulnerable/Weak/Block-style set for both loops.
- **Card reward mechanics** — how pooled encounters offer new cards (pick-1-of-N after clearing?).
- **Enemy intent & NPC hook logic** — how enemies choose intents; how NPCs deploy hooks / play your `Loose Thread`.
- **Theme/setting** — the world is ours to build; genre not yet chosen.
