# Dialog + Trade + Modal Interactions Spec

> Morrowind/Daggerfall-style **modal** interactions — dialog, trade, services. World pauses while open. Topic-based, inventory-grid-based, or list-based depending on the mode. Gamepad/joystick-first navigation throughout. Touches [`specs/ui.md`](ui.md) (presentation), [`specs/events.md`](events.md) (triggers + outcomes), [`specs/gameplay.md`](gameplay.md) (quest + inventory + faction).

## Scope

A unified modal-interaction system that powers every interaction where the player engages a non-combat entity:

| Mode | Used by | UI shape |
| --- | --- | --- |
| **Dialog** | NPCs (greetings, quest-givers, faction officials) | Topic list + response text |
| **Trade** | Merchants | Two inventory grids + price + barter or buy/sell |
| **Services** | Trainers, repair, identify, enchant, healers | List of services + cost + buttons |
| **Container** (later) | Chests, corpses, shared storage | Two inventory grids + take/place |
| **Crafting station** | Forges, alchemy benches, kitchens | Recipe list + ingredient grid (per [`specs/gameplay.md`](gameplay.md) crafting) |

All share the **same modal-pause mechanic + the same gamepad navigation grammar**.

## The defining design choice — world pauses

When any modal interaction opens:

- **Time-of-day clock freezes**
- **NPC schedules pause**
- **All entity ticks paused** (no AI, no physics step beyond visual interpolation)
- **Voxel simulation paused** (water flow, growth, particle physics)
- **Save-game-style snapshot** — you can save mid-modal and load back into the same state
- **Music ducks** to a quieter "menu" layer (per [`specs/audio.md`](audio.md))
- **Input fully captured** — no movement; the modal owns input

This matches Morrowind, Daggerfall, Oblivion, Skyrim, and most CRPGs. It gives the player as much time as they need to read, weigh decisions, and check inventory without time pressure.

### Multiplayer considerations

In 4-player co-op (the engine's default), pausing the world for one player can't pause it for others. Two resolution options:

1. **Per-player modal that pauses only that player's clock** — others continue. NPC keeps schedule from the host's perspective. The dialog/trade NPC is "occupied" — other players see them rooted in place. Works for co-op.
2. **Co-op shared modal** — all players see the same dialog (one player drives, others watch). Better for narrative immersion but requires shared input contention.

Default: **Option 1 (per-player local pause)**. Option 2 is a per-quest opt-in via [`specs/scene.md`](scene.md) flag. Decision flagged in `gaps.md` if needed.

In singleplayer: world fully pauses (no ambiguity).

## Universal modal layout

ImGui-style mock — built using [`specs/ui.md`](ui.md) widgets:

```text
┌──────────────────────────────────────────────────────────────┐
│  [Portrait]   NPC Name              Disposition: 75/100      │  ← Header (universal)
├──────────────────────────────────────────────────────────────┤
│                                                              │
│         [ mode-specific body — dialog / trade / services]    │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  [A] select  [B] back  [LB/RB] switch mode  [Y] inventory    │  ← Footer (universal)
└──────────────────────────────────────────────────────────────┘
```

### Mode switching

For merchants, players often want to flip between dialog and trade quickly. Footer's `LB/RB` cycles available modes for the current NPC:

```text
Dialog ←→ Trade ←→ Services
```

Bumper buttons swap the body panel without closing the modal. Dialog state preserved while in trade; topic history maintained.

### Header info

Universal across modes:

- Portrait (auto-generated voxel-character render if no hand-drawn portrait)
- NPC name + (optional) profession
- Disposition meter (numeric or bars — gameplay decision per `specs/gameplay.md`)
- Service tags (Merchant, Trainer, Quest Giver — shown as small icons)

## Dialog mode — topic-based

Morrowind's UX, modernized for controller:

```text
│                                                              │
│  You greet the merchant who looks up from sorting            │
│  potions. "Welcome, traveler. Looking for [alchemy] or       │
│  perhaps news of the [strange storms]?"                      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Topics                                              │    │
│  │ ► alchemy                                            │    │
│  │   strange storms                                     │    │
│  │   the missing caravan                                │    │
│  │   trade                                              │    │
│  │   leave                                              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
```

### Topic model

Each NPC has a **topic graph** stored in TOML (per [`specs/data-schemas.md`](data-schemas.md) + [`specs/content-authoring.md`](content-authoring.md)):

```toml
[npc.alchemist_merra.topics.greeting]
response = "Welcome, traveler. Looking for [alchemy] or perhaps news of the [strange storms]?"
always_available = true

[npc.alchemist_merra.topics.alchemy]
display_name = "alchemy"
conditions = "faction_rep.guild >= 25 OR skill.alchemy >= 20"
response = "Ah, a fellow practitioner. I have some [rare ingredients]."
on_select.discover = ["rare ingredients"]

[npc.alchemist_merra.topics."strange storms"]
display_name = "strange storms"
conditions = "quest.weatherwitch.state == 'started' OR player.region == 'valdaran'"
response = "Aye, three nights ago the sky went black. Some say a [witch] in the high passes..."
on_select.discover = ["witch"]
on_select.quest_advance = { id = "weatherwitch", node = "learned_of_witch" }

[npc.alchemist_merra.topics.leave]
display_name = "leave"
always_available = true
closes_modal = true
```

### Topic visibility

- **Discovered** — topic shown in list. Set via `on_select.discover` calls + quest events.
- **Conditional** — `conditions:` block; hide if false.
- **Persistent across NPCs** — once discovered, a topic appears on any NPC who has a response. NPCs without a response fall back to a generic "I don't know anything about that."

### Compact mode — for simple NPCs (Stardew-style)

Not every NPC needs a topic graph. Most Stardew villagers, rogue-like NPCs, and minor characters have one greeting + maybe a single response. Authoring a topic list for them is overkill, and the empty UI feels awkward.

Compact mode collapses the dialog UI:

- No topic list rendered
- Just the response text + a single "[A] continue / leave" prompt
- Multiple greetings can be authored (one is picked at random per interaction)

Per-NPC opt-in via TOML:

```toml
[npc.villager_lily]
compact_mode = true                  # explicit opt-in

[npc.villager_lily.greetings]
default = [
    "Lovely weather today, isn't it?",
    "Did you see the sunset last night?",
    "I should get back to my chores.",
]
# rain-day variant
on_rain = ["Rain again? My garden's drowning."]
# Friendship-level variants picked automatically:
on_friendship_high = ["You're a good friend, you know that?"]
```

If `compact_mode` is not set, the system **auto-detects**: an NPC with no discovered topics (or only a single greeting + `leave`) shows the compact UI automatically. Setting `compact_mode = false` forces full topic UI even for sparse NPCs.

**Universal across target games:**

- Stardew villagers: compact (default for most)
- Daggerfall NPCs: full topic UI (vast majority)
- Atelier party members: full UI for story scenes; compact for ambient chitchat
- Rogue-like vendors: compact for non-shop NPCs; full for shopkeepers (they need trade mode)

Compact mode still uses the same modal-pause + gamepad nav grammar — only the body panel changes.

### Hyperlinks

Text inside `[brackets]` is a hyperlinkable topic. Controller flow:

- `X` toggles "link-mode" — bracketed words become focusable
- D-pad navigates between them; `A` follows
- Mouse: click bracketed word to follow

## Trade mode — Morrowind/Daggerfall-style

Two inventory grids, prices shown per item, with **buy/sell** as the primary flow + optional **barter** as a v1.x sub-mechanic.

### Primary: Buy/Sell layout

```text
│  Player Gold: 245    NPC Gold: 500                           │
│  ┌─────────────────────────┐  ┌─────────────────────────┐    │
│  │ Your Inventory          │  │ Merchant Inventory      │    │
│  │ ─────────────────────── │  │ ─────────────────────── │    │
│  │ ► Iron Sword     35g    │  │   Health Potion   25g   │    │
│  │   Bread (3)      6g     │  │   Steel Helm     120g   │    │
│  │   Leather Boots  18g    │  │ ► Rare Ingredient 80g   │    │
│  │   Lockpick (5)   15g    │  │   Restore Magicka 30g   │    │
│  └─────────────────────────┘  └─────────────────────────┘    │
│                                                              │
│  [A] sell/buy    [Y] inspect    [LB/RB] grid swap            │
```

- **Two grids side-by-side**: player's inventory + merchant's inventory
- **Per-item price** shown next to each item; price depends on disposition + barter skill (per `specs/gameplay.md` formulas)
- `A` button on a player-grid item = sell (gold flows from merchant to player; item flows the other way)
- `A` button on a merchant-grid item = buy
- `Y` button on focused item = inspect (full item description, stats, weight)
- `LB/RB` switches which grid has focus
- Quantity prompt for stackable items (Bread × 3 → "Sell how many? 1 / 2 / 3 / all")

### Price formula — disposition + barter skill + perks + faction

Per [`specs/gameplay.md`](gameplay.md) skill + perk systems. Four independent modifiers stack additively:

```text
disposition_term  = disposition × 0.005      // 0..0.5 from 0..100 disposition
barter_term       = barter_skill × 0.003     // 0..0.3 from 0..100 skill
faction_term      = faction_rep × 0.001      // 0..0.1 from 0..100 rep with this NPC's faction
perks_sell_bonus  = sum of active sell-side perks (see § Trade perks below)
perks_buy_bonus   = sum of active buy-side perks

sell_price = base_price × (0.5 + disposition_term + barter_term + faction_term + perks_sell_bonus)
             clamped to sell_cap × base_price   (default cap = 1.0×; perks can raise)

buy_price  = base_price × (1.5 - disposition_term - barter_term - faction_term - perks_buy_bonus)
             clamped to buy_floor × base_price  (default floor = 1.0×; perks can lower)
```

Both clamp to a 0.5–1.5× band by default. Numbers tunable per game (declared in `[engine_options]` of [`project.toml`](data-schemas.md)).

Refusals: NPCs refuse to buy items below disposition X; refuse stolen goods unless they're a fence; refuse weapons unless they're an arms dealer.

### Barter skill — XP grown by use

`barter_skill` is a tracked skill in the classless skill system ([`specs/gameplay.md`](gameplay.md) § Skills), per-character, range 0..100. It grows by use:

- Each completed sale or purchase grants a small XP increment to barter skill (scaled by transaction value to avoid grinding via 1-gold sales)
- Skill milestones (25 / 50 / 75 / 100) unlock perk tiers — see § Trade perks
- Skill level feeds the `barter_term` in the formula directly (no separate "check" on each transaction)

**No haggling mini-game.** No "offer slider" UI, no click-button-X-times-until-NPC-agrees. The whole pricing experience is: pick item → see its price → confirm. Price reflects the player's current disposition + barter skill + faction rep + perks at that moment.

### Trade perks — layered on top of the skill

Each game defines its own perks (TOML in `<project>/assets/data/perks/`). For a Daggerfall-style game, typical trade perks:

| Perk | Unlock | Effect |
| --- | --- | --- |
| **Apprentice Trader** | barter skill 25 | `perks_sell_bonus += 0.05`, `perks_buy_bonus += 0.05` |
| **Journeyman Trader** | barter skill 50 | additional `+0.05` to both terms (stacks) |
| **Master Trader** | barter skill 75 | additional `+0.05` to both; **raises sell_cap to 1.2×**, **lowers buy_floor to 0.8×** |
| **Grandmaster Trader** | barter skill 100 | sell_cap to 1.5×; buy_floor to 0.5× — full price-band override |
| **Silver Tongue** | quest-gated | bribery gives 2× disposition delta per gold spent |
| **Smooth Talker** | charisma 60 | favorable dialog topics give +2 disposition instead of +1 |
| **Fence's Favor** | faction: thieves' guild | merchants of opposing faction don't refuse stolen goods (still get worse prices) |
| **Connoisseur** | skill 50 + quest | reveals true item rarity in inspect view; immune to seller misrepresentation |

Perks are data-driven; the engine just sums `perks_sell_bonus` / `perks_buy_bonus` per active perk per character. Adding a new perk = adding a new TOML entry; no engine code change.

### Disposition manipulation — four ways to raise disposition

The `disposition_term` is set by the per-NPC disposition value (range 0..100). Players raise disposition through these gameplay paths — **none of which involve a mini-game**:

| Path | Mechanism | When |
| --- | --- | --- |
| **Bribery** | Give gold from inventory directly to the NPC | Always available; cost-effective at low disposition, diminishing returns at high. Affected by `Silver Tongue` perk |
| **Questing for the NPC** | Complete tasks assigned by them or aligned with their faction | Quests with `on_complete.disposition_delta = +N { npc_id }` events |
| **Favorable dialog choices** | Pick responses aligned with NPC's allegiance ("I support the current king" to a royalist NPC) | Dialog topics with `on_select.disposition_delta = +N` for matching factional or personal preferences. Affected by `Smooth Talker` perk |
| **Persuasion magic** | Charm / Calm / Command spells from the magic system (per [`specs/gameplay.md`](gameplay.md) magic) | Spell effects with `target_attribute = "disposition" + duration` — temporary boost while spell is active |

Per-NPC disposition is persistent across saves (per [`ARCHITECTURE.md`](../ARCHITECTURE.md) save model). The per-spell-duration boost from persuasion magic does NOT persist when the spell expires.

### Trade events for hooks

Per [`specs/events.md`](events.md), trade fires:

- `trade.completed { npc, player, items_in, items_out, gold_delta }` — quest hooks, achievement hooks
- `skill.gained { entity, skill_id = "barter", xp_delta }` — skill-progression UI
- `disposition_changed { npc, old, new, source = "bribery" | "quest" | "dialog" | "spell" }` — for quest gates that care about *how* the disposition changed

Implementation cost: disposition system already needs to exist for dialog gating + price calculation. The four manipulation paths are just additional fire-sites for the existing `disposition_changed` event. All ship in v1.0 (Phase 8 gameplay modules).

## Services mode

For trainers, repair, identify, enchant, healers:

```text
│  ┌─────────────────────────────────────────────────────┐     │
│  │  Services                                           │     │
│  │ ─────────────────────────────────────────────────── │     │
│  │ ► Repair all gear              (40 g)               │     │
│  │   Identify selected item        (15 g)              │     │
│  │   Train: Alchemy (50 → 51)      (85 g)              │     │
│  │   Train: Sneak (40 → 41)        (60 g)              │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  [A] purchase    [Y] inspect cost breakdown                  │
```

- List of services available from this NPC
- Cost shown per service (disposition-modified like trade)
- `A` = purchase; immediate effect (or follow-up modal for "which item to identify")
- Services list defined per-NPC in TOML

## Gamepad navigation grammar (universal)

Default bindings (rebindable per [`specs/accessibility.md`](accessibility.md)):

| Action | Button | Mouse + KB |
| --- | --- | --- |
| Open modal | A / interact | E / Click |
| Navigate within mode | D-pad up/down/left/right | Arrow keys |
| Select / confirm | A | Enter / Click |
| Back / cancel | B | Escape |
| Switch mode (Dialog ↔ Trade ↔ Services) | LB / RB | Tab / Shift-Tab |
| Inspect focused item / topic | Y | I |
| Inventory shortcut (from any modal) | back / select | Tab |
| Scroll text or list | RT / LT (analog) | Scroll wheel |
| Quantity prompt (during trade) | D-pad left/right | Arrow / type number |
| Skip text animation | A | Space |
| History (dialog) | menu / start | H |

### Focus rules

- Initial focus = first non-disabled item (greeting topic, first inventory item, top service)
- Focused item has visible ring + slight scale-up (controller affordance)
- B button always closes one level up; from top level closes modal entirely
- Wrapping: D-pad past the end wraps to top (toggle in settings; off by default)

## Save / load while modal

Per [`specs/save-ux.md`](save-ux.md): the modal's state is part of the save. Save mid-dialog → reload → resume the same modal with same NPC focus, same topic state. Treat the modal as a save-point-safe context.

Useful for:

- Long conversations players want to pause
- Mod testing (load a save at a known modal state)

## Disposition + faction integration

Per [`specs/gameplay.md`](gameplay.md):

- Modal opens at the NPC's current disposition toward the player
- Disposition affects: visible topics, response tones, trade prices, service availability, refusal thresholds
- Some dialog topics + completed trades modify disposition (positive: agreeable response, generous trade; negative: confrontational dialog, attempted barter below floor)
- Disposition changes emit events for quest hooks ([`specs/events.md`](events.md))

## Quest integration

Topics and trade actions can:

- Advance quest state (`on_select.quest_advance = { id, node }`)
- Discover quests (`on_select.quest_start = "weatherwitch"`)
- Be gated by quest state (`conditions = "quest.X.state == 'Y'"`)
- Hand over quest items (`on_select.give_item = "ancient_scroll"`)
- Receive rewards (`on_select.grant = { gold = 250, item = "amulet" }`)

Events fired (per [`specs/events.md`](events.md)):

- `modal.opened { npc, player, mode }`
- `modal.closed { npc, player, reason }`
- `dialog.topic_selected { npc, player, topic }`
- `trade.completed { npc, player, items_in, items_out, gold_delta }`
- `service.purchased { npc, player, service_id }`

## Localization

All player-visible text uses `tr(key)` per [`specs/localization.md`](localization.md). Topic display names, response text, service labels, item names, refusal messages — all keyed to `.po` files. TOML stores keys, not literals.

## Performance budget

- Modal opening: ~50 ms one-time cost (snapshot world state, build UI)
- Modal open: ~0.2 ms per-frame render budget (one UI panel)
- World fully paused while open — zero simulation cost
- Closing: ~50 ms (resume world; one-tick catch-up if needed)

No special perf concerns. Doesn't affect the 50–60 FPS target.

## Reference patterns

Per [`gap-references.md`](../gap-references.md) — modal pause + topic dialog + barter is a well-trodden CRPG pattern. Read for design + UX:

- Morrowind dialog system + barter UI ([UESP](https://en.uesp.net/wiki/Morrowind:Dialogue))
- Daggerfall dialog + buy/sell ([UESP](https://en.uesp.net/wiki/Daggerfall:Dialogue))
- Disco Elysium dialog (different model — branching wheel — for comparison, not adoption)
- Stardew Valley shop UI (simpler buy/sell, for the gamepad nav pattern)

Read for *information density* + *modal flow*. No verbatim port.

## Open decisions

- **Disposition surface** — show numeric value (Daggerfall-style) or inferred label only ("warm / neutral / cold")? Modern preference is inferred; Daggerfall fans expect numeric. Per-game `[engine_options]` toggle.
- **Barter mini-game** — **not in scope**. Disposition manipulation through bribery / questing / favorable dialog / persuasion magic ships in v1.0; no separate Morrowind-style haggling slider. See § Disposition manipulation above.
- **Co-op multi-modal** — per-player local pause vs shared modal. Default: per-player local. Per-quest opt-in to shared (e.g. for crucial story conversations).
- **Stolen-goods detection** — does the merchant recognize stolen items? Per-game gameplay decision. Mechanism (item flag + merchant flag check) is engine-supported.
- **Service-specific sub-modals** — "Identify which item?" pops another modal layer; need a max-depth limit (probably 2: primary modal + one sub-modal). Anything deeper = bad UX.

## Closes

Same status as before — this spec doesn't close a numbered `gaps.md` entry directly; it formalizes interaction design that was implicit in `specs/gameplay.md` (quest, NPC AI, inventory, crafting). Should be added to `gaps.md` next status update as a tracked spec under §2.1.

Touches:

- §3 #17 (Quest data model — uses dialog as the player-facing trigger surface)
- §3 #18 (NPC AI architecture — modal triggers when player interacts with an NPC; AI pauses cleanly)
- §3 #21 (Inventory model — trade uses inventory grids)
- §2.1.D ([`specs/ui.md`](ui.md)) — provides the widget building blocks

Sources:

- Morrowind dialog format: <https://en.uesp.net/wiki/Morrowind:Dialogue>
- Daggerfall barter: <https://en.uesp.net/wiki/Daggerfall:Bargaining>
- Game accessibility for modal dialog: <https://gameaccessibilityguidelines.com/> (subtitle + speaker-name + skip controls)
- Controller-friendly CRPG UI conventions: Pillars of Eternity / Divinity: Original Sin 2 (modern reference)
