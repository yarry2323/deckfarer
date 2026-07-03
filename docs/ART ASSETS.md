# Deckfarer — Art Asset Tracker

Living document. Every visual the game will eventually need, what it's standing in as *now*, and what it needs to become *later*. Update the Status column as assets move from stand-in → generated → final.

**Golden rule during development:** nothing blocks on art. Every asset below has a stand-in (typography + color + CSS/SVG) so the game is fully playable before a single illustration exists. Tier 3 art is decoration layered onto a working game, never a dependency.

---

## The three tiers (recap)

| Tier | What | When | Cost |
|------|------|------|------|
| **1 — UI chrome** | Card frames, hand layout, HUD, intent icons, pips | Build now | Nearly free (HTML/CSS) |
| **2 — Procedural** | Per-suit color themes, SVG suit icons, CSS card "art" | Add for readability | Cheap |
| **3 — Illustrated** | Card paintings, enemy art, room art, portraits | Defer to end | Expensive — quarantined |

---

## Style Bible (fill this in *before* generating any Tier 3 art)

This is what keeps LLM-generated art from looking like it came from five different games. Lock these first, paste them into every generation prompt.

- **Overall vibe:** _(e.g. "grimy retro-future noir, Infocom-meets-cyberpunk")_ — TBD
- **Rendering style:** _(painterly / flat vector / pixel / ink)_ — TBD
- **Card aspect ratio:** _(lock this early — e.g. 5:7 portrait)_ — TBD
- **Frame proportions:** art window vs. text box vs. cost/suit corner — TBD
- **Lighting/mood:** consistent light direction & contrast — TBD
- **Palette per suit** (locks color identity across the whole game):

| Suit | Meaning | Color (TBD) | Feel |
|------|---------|-------------|------|
| Might | force | — | heavy, hot |
| Wits | cunning | — | cool, sharp |
| Charm | influence | — | warm, soft |
| Tech | systems | — | electric, clean |

- **Per-class accent** (Enforcer / Cipher / Fixer each get an identity color + emblem) — TBD

---

## Asset inventory

### 1. Card frames & chrome — Tier 1
| Asset | Stand-in | Status | Notes |
|-------|----------|--------|-------|
| Card frame template (per suit) | Colored CSS border + suit color | ☐ stand-in | 4 variants, one per suit |
| Card back | Solid color + class emblem placeholder | ☐ stand-in | Per-class back is a nice-to-have |
| Cost / value corner | Text glyph | ☐ stand-in | |
| Hand layout container | Flex/fan CSS | ☐ stand-in | |
| Deck / discard counters | Number + label | ☐ stand-in | |
| Resource pips | Colored dots | ☐ stand-in | |
| Enemy intent icons | Text label ("winds up: 6") → then icon | ☐ stand-in | attack / block / buff / **social** — social intent is unique to us |
| Status/junk badges | Text tag | ☐ stand-in | Bleeding, Heat, Fracture, Wound, Loose Thread |

### 2. Suit iconography — Tier 2
| Asset | Stand-in | Status | Notes |
|-------|----------|--------|-------|
| Might icon | Letter/glyph | ☐ | SVG, single-color, tintable |
| Wits icon | Letter/glyph | ☐ | |
| Charm icon | Letter/glyph | ☐ | |
| Tech icon | Letter/glyph | ☐ | |

### 3. Class identity — Tier 2/3
| Asset | Stand-in | Status | Notes |
|-------|----------|--------|-------|
| Enforcer emblem + portrait | Color block + name | ☐ | Might scaling / self-inflicted junk |
| Cipher emblem + portrait | Color block + name | ☐ | **Test class.** Leverage mechanic |
| Fixer emblem + portrait | Color block + name | ☐ | Deployables |

### 4. Individual card art — Tier 3 (the big pile — defer)
Track per-card as authored. Categories, not a full list yet:
| Category | Stand-in | Status | Notes |
|----------|----------|--------|-------|
| Starting-deck cards (per class) | Typographic card | ☐ | Authored alongside class decks |
| Common world pool | Typographic card | ☐ | Shared — [Pry], [Read], etc. |
| Class-exclusive pool | Typographic card | ☐ | |
| Leverage cards ([Leverage: X]) | Typographic card | ☐ | Minted at runtime — art may be a shared template + dynamic label |
| Deployables (Fixer) | Typographic card | ☐ | Need an "on-table/active" visual state too |
| Junk/status cards | Typographic card | ☐ | Heat, Fracture, Wound, Bleeding, Loose Thread |

### 5. Encounters — Tier 3 (defer)
| Asset | Stand-in | Status | Notes |
|-------|----------|--------|-------|
| Enemy art (combat) | Name + intent text | ☐ | |
| NPC portraits (social) | Name + text | ☐ | Social encounters are first-class — these matter as much as enemies |

### 6. Rooms / locations — Tier 3 (defer longest)
| Asset | Stand-in | Status | Notes |
|-------|----------|--------|-------|
| Room / location art | Prose only (Infocom-style) | ☐ | Prose *is* the room until very late; art is pure garnish |

---

## Generation workflow (for later)
1. Lock the Style Bible above first — no art before that.
2. Generate one **test card per suit** and one enemy; check they read as the same game.
3. Generate in batches by suit/class so palette stays consistent within a set.
4. Keep source prompts in a `prompts/` folder next to assets so any image can be regenerated on-style.
5. Name files to match card/enemy IDs in the data files → drop-in replacement, no code changes.
