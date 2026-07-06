# Project DC-3 — Game Design Document v2 (Prototype Skeleton)

**Owner:** martian (dinoJenna) · **Version:** 2.0 · **Date:** 2026-07-06
**Scope:** This document defines **Phase 1 (Prototype)** only. Everything deferred lives in §11 Phase 2 Parking Lot. Nothing in the parking lot may be built, referenced, or partially implemented in the prototype.

---

## 0. One-Line Pitch

A 1v1 tactical card game where you battle a simple AI on a randomized Terrain-driven Arena, earning Momentum through aggression to reduce your opponent's Anima from 75 to 0.

---

## 1. Pinned Decisions (Single Source of Truth)

Every past contradiction is resolved here. If any other section ever disagrees with this table, **this table wins.**

| # | Topic | Pinned Rule |
|---|-------|-------------|
| D1 | Word "Vector" | A **Vector** is one of the 4 vertical attack lanes. Zones (Stormfront, Nexus, Topographic, Discard) are called **Zones**, never Vectors. |
| D2 | Discard | One **Discard Zone** per Battler. When a deck empties, its Discard is shuffled back in (max 1 Effect card reshuffled at a time). No permanent banishment in prototype. |
| D3 | Effect durations | Effects are one of three types: **Instant**, **Trap** (sits face-down until trigger), **Buff** (lasts 1 round). |
| D4 | Effect pricing | All Effects cost **7+ MP**. `fate` is raised from 5 → **7 MP**. |
| D5 | Rear Line | Rear Line Patrols **cannot attack and cannot be targeted**. It is a safe staging row. Front Line is active and vulnerable. |
| D6 | Blitz cost | Blitz costs a **flat 2 MP** (tunable constant `BLITZ_COST`). |
| D7 | Combat targeting | **Lane-based:** a Front Line Patrol may only clash with the enemy Front Line Patrol in the *same Vector (lane)*. Empty enemy Front slot in that lane = Blitz allowed. |
| D8 | Deck size | **20 cards:** 16 Patrols + 4 Effects. Max 2 copies of a Patrol; max 1 copy of any Effect. |
| D9 | Draw timing | Draw 1 card during the **End Epoch** (auto-draw on End Turn). Opening hand: 3. Hand max: 5 (if full, no draw). |
| D10 | Players | 1 human vs 1 **simple AI**. Moon side always goes first (coin toss decides who is Moon). |
| D11 | Tech | **Single-file** `index.html` (embedded CSS + JS). Assets in `/assets/`. No frameworks, no build step, no localStorage. |
| D12 | AP start | 75 for both Battlers. Constant `STARTING_AP`. |

---

## 2. Tone & Content Guardrails

- Combat is stylized **energy clash** — Patrols *dissipate* or are *banished to the Discard*, never killed, wounded, or shown suffering. No blood, gore, or cruelty in art, text, or sound.
- Overall feel: relaxing, engaging, hopeful competition. Losing is framed as an invitation to try again ("Test the fates").

---

## 3. Glossary (One Word = One Meaning)

| Term | Meaning |
|------|---------|
| **Arena** | The entire game board. Its background is the active Terrain. |
| **Battler** | A player (human or AI). |
| **AP — Anima Points** | Life force. Start 75. Reach 0 = lose. |
| **MP — Momentum Points** | Currency to play cards, move Patrols, and Blitz. |
| **VsP — Versus Power** | A Patrol's combined attack strength and durability. |
| **Patrol** | A combat unit card. |
| **Effect** | An Instant / Trap / Buff card played to the Nexus. |
| **Terrain** | A card defining the Arena background and a global rule. |
| **Vector** | One of 4 vertical lanes running across the Arena. |
| **Stormfront Zone** | A Battler's two Patrol rows (Front Line + Rear Line). |
| **Front Line** | Row nearest the opponent. Can attack; can be attacked. |
| **Rear Line** | Row nearest the Battler. Safe staging; cannot attack or be targeted. |
| **Nexus Zone** | Where a Battler's Effect cards sit. |
| **Topographic Zone** | Shared center slot holding the active Terrain card. |
| **Discard Zone** | Face-up pile of banished cards; reshuffles per D2. |
| **Blitz** | A direct attack on enemy AP through an empty enemy Front slot (same lane). |
| **Epoch** | One phase of a turn (there are four). |

---

## 4. Arena Layout

4 Vectors (lanes), read top to bottom from the human player's view:

```
            OPPONENT (AI)
 [Hand — hidden]        [Deck] [Discard]
 ┌─────┬─────┬─────┬─────┐
 │ R1  │ R2  │ R3  │ R4  │  ← AI Rear Line
 ├─────┼─────┼─────┼─────┤
 │ F1  │ F2  │ F3  │ F4  │  ← AI Front Line
 ├─────┴──┬──┴──┬──┴─────┤
 │ Nexus  │TERR.│ Nexus  │  ← AI Nexus | Topographic | Player Nexus
 ├─────┬──┴──┬──┴──┬─────┤
 │ F1  │ F2  │ F3  │ F4  │  ← Player Front Line
 ├─────┼─────┼─────┼─────┤
 │ R1  │ R2  │ R3  │ R4  │  ← Player Rear Line
 └─────┴─────┴─────┴─────┘
 [Hand — face up]       [Deck] [Discard]
```

**HUD (top to bottom, exactly this order):**
1. Opponent AP + MP
2. Current Terrain card name + its effect text
3. Player AP + MP
4. **End Turn** button
5. **New Match** button

AP displays as dual **Sun/Moon bars** (Dead Space-style depletion): orange sun for the Sun Battler, blue moon for the Moon Battler.

---

## 5. Resources & Core Numbers

All values live as named constants at the top of the JS — never hard-coded inline.

| Constant | Value | Notes |
|----------|-------|-------|
| `STARTING_AP` | 75 | Per Battler |
| `STARTING_MP` | 3 | Per Battler |
| `MP_PER_TURN` | +2 | Gained in MP Epoch |
| `MP_ON_KILL` | +1 | Per successful Patrol clash won (banked, paid next MP Epoch) |
| `MP_ON_BLITZ` | +2 | Bonus per successful Blitz (banked, paid next MP Epoch) |
| `BLITZ_COST` | 2 | Flat MP cost to declare a Blitz |
| `MOVE_TO_FRONT_COST` | 1 | MP per Patrol moved Rear → Front |
| `MOVE_TO_REAR_COST` | 0 | Front → Rear is free |
| `HAND_START` | 3 | Opening hand |
| `HAND_MAX` | 5 | No draw if hand is full |
| `DECK_SIZE` | 20 | 16 Patrols + 4 Effects |

---

## 6. Turn Structure (Four Epochs)

1. **MP Epoch** — Gain `MP_PER_TURN` plus any MP banked from last turn's kills/Blitzes.
2. **Main Epoch** — Any number of, in any order (MP permitting): play Patrols to Rear or Front Line; play Effects to Nexus; move Patrols between lines (per movement costs). One Patrol per slot.
3. **Combat Epoch** — Each Front Line Patrol may act once: **Clash** (enemy Front Patrol in same lane) or **Blitz** (that lane's enemy Front slot is empty; pay `BLITZ_COST`).
4. **End Epoch** — Resolve end-of-turn effects (expire Buffs, return `control`ed Patrols), draw 1 card (respect `HAND_MAX`, reshuffle per D2 if deck empty), pass turn.

---

## 7. Combat Resolution

**Clash (Patrol vs Patrol, same lane, Front vs Front):**
- Compare VsP. Lower VsP Patrol is banished to its Discard.
- Winner survives with `VsP = winner VsP − loser VsP`.
- If equal, **both** are banished.
- A surviving Patrol at 0 VsP is banished.

**Blitz (direct attack):**
- Legal only if the enemy Front slot in the attacker's lane is empty, and the active Terrain allows Blitz.
- Pay `BLITZ_COST`. Opponent AP −= attacker's current VsP.
- Trap Effects (e.g., `disruption`) may intercept — resolve Traps before damage.

**Win/Loss:** first Battler at 0 AP loses. Check after every AP change.

---

## 8. Card Database (Prototype Set)

### 8.1 Patrols (9 units)

| Card | Asset | Tags | MP Cost | VsP | Rarity |
|------|-------|------|---------|-----|--------|
| Elf | elf.png | humanoid, non-mars | 1 | 2 | Common |
| Astronaut | astronaut.png | humanoid, mars | 2 | 3 | Common |
| Orc | orc.png | humanoid, non-mars | 3 | 4 | Common |
| Martian | martian.png | non-humanoid, mars | 4 | 5 | Common |
| Mage | mage.jpg | humanoid, non-mars | 5 | 5 | Common |
| Flux | flux.jpg | non-humanoid, non-mars | 6 | 7 | Rare |
| Piercer | piercer.jpg | humanoid, non-mars | 8 | 9 | Rare |
| Stitcher | stitcher.jpg | non-humanoid, mars | 10 | 11 | Rare |
| Weaver | weaver.jpg | non-humanoid, mars | 12 | 14 | Legend |

*(Starter numbers for balance testing — all tunable in one data table.)*

### 8.2 Effects (6 cards)

| Card | Asset | MP | Type | Rules Text (authoritative) |
|------|-------|----|------|---------------------------|
| Fate | fate.jpg | 7 | Instant | Launch a Blitz from any lane, ignoring blockers and `BLITZ_COST`. Damage = average VsP of all your field Patrols × random multiplier (100% / 50% / 35% / 2%, equal odds). |
| Control | control.jpg | 7 | Instant | Take control of one enemy Front Line Patrol for one Clash or Blitz this turn. It returns to its owner in the End Epoch. |
| Siege | siege.jpg | 7 | Buff | Until your next turn, the enemy cannot Clash your Patrols or Blitz you. Max one use per match. |
| Time | time.jpg | 8 | Trap | Triggers when the enemy declares any attack: all enemy attacks this turn are cancelled. Next enemy turn, their strongest cancelled attack resolves against your highest-VsP Patrol; excess damage hits your AP. |
| Last Stand | last-stand.jpg | 11 | Trap | Triggers when your Patrol would be banished by a Clash: it strikes back with VsP equal to its attacker's. Both Patrols are banished. |
| Disruption | disruption.jpg | 12 | Trap | Triggers on an enemy Blitz: you take no damage, and half the Blitzing Patrol's VsP is dealt to the enemy's AP. While this Trap is set, you cannot play other Effects. |

### 8.3 Terrains (5 cards — one drawn at random per match)

| Card | Asset | Global Rule |
|------|-------|-------------|
| Colosseum | colosseum.jpg | All card MP costs −2 (minimum 1). |
| Vault | vault-terrain.png | Both Battlers gain +1 extra MP per turn. |
| Toxic | toxic-terrain.png | −1 VsP to all non-mars Patrols. |
| Fortress | fortress-terrain.jpg | Blitz is disabled for both Battlers. |
| Lava | lava-terrain.png | +3 VsP to non-humanoid Patrols; −2 VsP to humanoid Patrols. |

Terrain VsP modifiers apply while the Patrol is on the field; base VsP is stored separately so modifiers never permanently mutate a card.

### 8.4 Prototype Decks

Both Battlers use the same starter deck (mirror match) unless changed:
2× Elf, 2× Astronaut, 2× Orc, 2× Martian, 2× Mage, 2× Flux, 2× Piercer, 1× Stitcher, 1× Weaver (16 Patrols) + Fate, Control, Siege, Time (4 Effects).

---

## 9. UI, Audio & Flow Spec

**Match start modal:** "The Match begins, Anima Flows… Press Start to decide your fate" — Start button in a navy blue box with a baby blue outline.

**Coin toss animation:** coin with orange sun / blue moon faces flips; Moon Battler goes first.

**Terrain reveal:** Terrain card drawn, Arena background updates to match, effect text shown in HUD.

**Defeat modal (player loses):** matches start modal style — "Lose, test the fates more or give up?" — *test the fates* starts a new match; *give up* shows a farewell screen (no menu until post-prototype).

**Victory modal (player wins):** same style — "Victory, the Anima favors you! Test the fates again?" — *test the fates* starts a new match. Plays `defeat-win.mp3`.

**Audio:** background rotates `shuffled-fates.mp3` → `celestial-horizon.mp3` → `cards-of-fate.mp3` in a loop, with click-to-start fallback (browsers block autoplay).

**Missing-asset rule:** if any image/audio file is absent, render a styled placeholder card (name + stats on a colored frame by rarity) or skip audio silently. The game must never break over assets.

**Palette:** navy / baby blue primary (per existing modal spec); sun-orange and moon-blue as the two Battler accent colors.

---

## 10. Simple AI (Prototype)

Heuristic, no learning, fully deterministic given a seed (seedable RNG for reproducible bug reports):

1. **Main Epoch:** play the highest-VsP affordable Patrol to an empty Rear slot; move Patrols forward into lanes that enable a favorable Clash or a Blitz (respect movement cost); play `siege` if below 25 AP; set an affordable Trap if Nexus is empty.
2. **Combat Epoch:** for each Front Patrol — Clash if it wins or ties the trade; Blitz if the lane is open and MP allows; otherwise hold.
3. Never makes illegal moves; always ends turn within 2 seconds (no timer mechanic in prototype).

---

## 11. Phase 2 Parking Lot (Do NOT build in prototype)

3-player mode · Coffer trading Zone · turn timer + AP forfeit · Tallies currency, Vend marketplace, Essence · badges & Conspectus grimoire · deck classes & deck builder · matchmaking/online play · booster packs & monetization · permanent Banishment Zone · additional Terrains/expansions · sham (decoy) cards · player-mat milestone bonuses · Godot/mobile port.

---

## 12. Definition of Done (Prototype)

- [ ] Full match vs AI playable start → win/lose → rematch, with zero console errors
- [ ] All 12 Pinned Decisions verifiably true in play
- [ ] All 20 cards, 6 Effects, and 5 Terrains function per §8 rules text
- [ ] Reshuffle rule (D2) works when a deck empties
- [ ] HUD order, modals, coin toss, and audio rotation match §9
- [ ] Works in a desktop browser at 1280×720 and degrades gracefully on mobile width
