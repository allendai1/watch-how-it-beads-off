# +1 Waterproof Every Second — Design & Build Handoff

A Roblox "+1" game based on the TikTok trend **"watch how it beads off"**, where people stand under a shower in their jacket to show how waterproof it is and the water beads off.

This doc captures the design decisions made so far and a build plan. Numbers marked *(tune)* are placeholders for balancing.

---

## 1. Concept

- Every player's **Waterproof** stat goes up **+1 every second** (times multipliers).
- The map is a track of **waterfall gates**. Each gate covers a doorway and has a **requirement** number on a sign above it.
- When a player walks into a waterfall, the water **parts around them**. The width of the gap scales with their Waterproof stat compared to the gate's requirement.
- If the gap is at least as wide as the doorway, they walk through dry while beads fly off their jacket. If not, they slow down, soak and get **shoved back out**.
- The bead reveal is the core reward, and it's the trend itself, so it should be satisfying and easy to clip.

## 2. Core loop

1. Spawn at the start of World 1 wearing the basic game jacket. The stat is already ticking.
2. Walk to the next waterfall gate. Pass it or get pushed back.
3. While waiting for the stat to grow: buy jackets, hatch duck pets, stand in shower booths (AFK multiplier), redeem codes, claim rewards.
4. Clear gates → earn **Wins**. Clearing the last gate of a world unlocks **Rebirth**.
5. Rebirth: Waterproof resets to 0 → permanent multiplier + next jacket tier/bead effect.
6. Enough rebirths → portal to the next world (bigger numbers, more dramatic water).

### First-time experience (critical)
- First gate requirement is **10** *(tune)* and it's a few seconds' walk from spawn, with a glowing arrow on the ground pointing to it.
- The player should reach it at around 6–7 so they **fail once** ("Too weak! 7 / 10"), then **pass** seconds later with a celebration and "+1 Win".
- Early gates are close together with small requirements so players win repeatedly in the first 5 minutes.

## 3. The waterfall gate mechanic

### Player experience
| State | What happens |
|---|---|
| Under requirement | Walk speed drops sharply (walking into a current). A small gap parts at the shoulders. The jacket darkens as the soak meter fills. Doorway barrier stays solid. When soak hits 100% (about 1.5s *(tune)*), a splash shoves them a few studs back out in front of the waterfall with a "Too weak! X / Y" popup. |
| ≥80% of requirement | Same as above, but the gap is visibly close to the door frame, so the player can see they're almost there. |
| At/above requirement | No slowdown, no push. Water parts at least as wide as the door, beads fly off the shoulders and sleeves, and they walk through. |
| Way above requirement | The gap opens wider than the door. Cosmetic only, but it's the flex moment for other players to see. |

- The gap is calculated as `doorWidth * (waterproof / requirement)`, capped at a max width.
- The door frame should be clearly visible behind the water so players can compare the gap with the opening.
- Bead intensity (particle rate) scales with how far over the requirement the player is.
- First clear of a gate: brief camera move to a front-facing shot as the player walks out of the water with beads streaming off. Keep it short and skippable.

### How it works in Roblox

**Waterfall visuals (client)**
- Each waterfall is a row of about 20–30 vertical **Beams**, each between a top Attachment (cliff edge) and a bottom Attachment (floor).
- Water texture with `TextureSpeed` set so it scrolls down. A mist ParticleEmitter at the base and a looping water Sound.
- **Parting:** each client, every frame (`RenderStepped`), for waterfalls near the camera (~150 studs):
  - Find all players inside that gate's `WaterZone` part.
  - For each, read their `Waterproof` attribute, compute gap half-width.
  - For each strip, in the waterfall's local X axis, if it falls inside any player's gap, move its bottom Attachment sideways out of the gap and increase `CurveSize0`/`CurveSize1` so it bends around the body. Use the largest push if multiple players overlap.
  - Lerp toward targets so strips ease in and out. Strips return to straight when nobody is in the water.
- Every client renders this for **every** player, so everyone sees everyone's gaps.

**Passing the doorway (client + server)**
- Each doorway has an invisible collider Part (`Barrier`, `CanCollide = true`).
- Character physics is owned by the player's client, so **the client sets `Barrier.CanCollide = false` locally** when `Waterproof >= Requirement`. The character walks through, while the barrier stays solid for everyone else.
- When under the requirement and inside the `WaterZone`: the client lowers `Humanoid.WalkSpeed`, fills the soak meter, and on full soak applies a short backward shove (set `HumanoidRootPart.AssemblyLinearVelocity`). This is client-side so it feels instant.
- **Server validation:** the server tracks each player's `HighestGateCleared`. It periodically checks whether a player's position is past a gate's barrier plane:
  - Past gate N and qualifies → mark cleared (award wins if first clear), update checkpoint.
  - Past gate N and **does not** qualify and hasn't cleared it → teleport back to the front of that gate.
- Player-to-player collisions **off** (collision groups), so players can't block doorways or push each other.

**Jacket (client visuals, server assignment)**
- Every player gets a **game-owned jacket** (Accessory or MeshPart welded to the torso) on spawn, because avatar clothing can't be recolored reliably.
- Soak: tween the jacket color darker as soak rises. Drip particles under the character while soaked.
- Beads: ParticleEmitters (droplet texture) on shoulder and sleeve attachments, enabled while clearing water, rate scaled by `BeadQuality`.
- Jacket tiers swap mesh/color/bead effect.

### Gate model structure
Each gate is a Model in a known folder (tagged `Gate` in code at startup) with attributes:
- `Requirement` (number), `GateIndex` (number), `World` (number)

Children:
- `Strips/` — Attachments and Beams for the waterfall
- `WaterZone` — invisible, non-colliding Part in front of / around the doorway (who's in the water)
- `Barrier` — invisible colliding Part in the doorway
- `Sign` — SurfaceGui showing the requirement (formatted)
- `Checkpoint` — spawn/teleport point on the far side
- `PushbackPoint` — safe spot in front of the waterfall

One server system and one client controller handle every gate; adding a gate = copy the model, change the attributes.

## 4. Systems overview

### Launch
- **Waterproof stat:** server-authoritative, +1/s × (rebirth mult × jacket mult × pet mult × booth mult × gamepass mult × boosts). Stored as player attribute (replicates) and in leaderstats.
- **Gates:** World 1 with about 10–15 gates plus a boss waterfall. Requirements on an exponential curve, e.g. 10, 25, 60, 150, 400, … *(tune)*.
- **Wins & rebirth:** wins from clearing gates; rebirth unlocked at end of world, resets Waterproof and `HighestGateCleared`, grants multiplier and jacket tier.
- **Jacket shop:** a few tiers bought with wins; each adds a gain multiplier and a better bead effect.
- **Shower booths (AFK zones):** stand in a stall to multiply gains. The player plays an idle pose under running water with beads rolling off. Tiers: Basic Shower x2 (free), Rain Room x5 (after first rebirth), Storm Chamber x10 *(tune)*, plus a **VIP Shower** gamepass booth placed visibly. Show an "AFK for N min, +X waterproof" counter.
- **Duck pets:** "like water off a duck's back." 2–3 eggs bought with wins; rubber duck → mallard → golden → mythic storm duck. Equip best few for multipliers. (First thing to cut to an early update if time is short.)
- **Codes:** redeemable for boosts/wins; one redemption per code per player.
- **Lobby leaderboards (3D):** top Waterproof, most rebirths, most wins. (Can slip to first update.)
- **Gamepasses:** 2x Waterproof, VIP Shower, Auto-Rebirth. **Starter pack** developer product.
- **Big number formatting:** 1.2K, 3.4M, 5.6B… everywhere from day one.
- **Data saving:** stats, rebirths, wins, jackets, pets, redeemed codes, gamepass-granted state. Use a session-locked save system.
- **Mobile-first UI:** large buttons, readable numbers.

### First updates (weeks 1–2 after launch)
- World 2 (new gates, new boss waterfall) + new eggs/ducks
- Playtime gifts and daily login rewards
- Potions (2x Waterproof / 2x Wins, timed) and developer products: skip gate, waterproof bundles, egg packs
- Weather events (periodic server-wide storm with boosted gains)
- "Make it rain" server-wide purchase (boost for everyone, buyer's name announced)
- Friend boost (multiplier for playing with friends in the same server)
- Achievements/badges
- Auto-hatch gamepass

### Later
- Worlds 3–5+
- Pet trading
- Spin wheel
- Pet index and merging duplicates
- Limited-time / holiday events
- Design-your-own jacket

## 5. World and gate themes

Gates reuse a few shapes with escalating intensity: **curtain** (waterfall), **sideways jet** (hydrant/hose), **overhead dump** (tipping bucket). All check the same stat. Each world ends with a boss gate.

- **Backyard:** garden hose, sprinkler, overflowing kiddie pool, rain gutter, water balloon cannon
- **House:** sink tap, shower, overflowing bathtub, burst pipe, washing machine flood
- **City:** fire hydrant, car wash, fountain, fire truck hose, flash flood
- **Waterpark:** slide splashdown, giant tipping bucket, wave pool, water cannons
- **Wild:** rainstorm, river rapids, geyser, real waterfall, monsoon
- **Ocean:** crashing waves, whale spout, hurricane, storm surge, tsunami
- **Mega/fantasy (later):** dam spillway, cloud kingdom, sea god, space comet

Variety mostly comes from scale, flow speed, particle density, color and door width. A heavier version of the same gate is a valid "new" gate.

## 6. Code architecture

Follow the installed Roblox plugin conventions (`roblox-architecture`, `roblox-security`, `roblox-datastores`, `roblox-performance`, `luau-strict-typing`): Rojo layout, `--!strict`, exactly one server Script and one client LocalScript, everything else ModuleScripts, two-phase `init`/`start` bootstrap, remotes created server-side from a single list.

```text
src/
├── shared/
│   ├── Types.luau          player data, gate data, pet/jacket defs
│   ├── Config.luau         gate requirement curve, multipliers, booth tiers,
│   │                       soak timing, walk speeds, shove force, prices
│   ├── Remotes.luau        remote name list
│   └── Util/
│       ├── NumberFormat.luau   1.2K / 3.4M / 5.6B
│       └── GateMath.luau       gap width, bead quality (shared formula)
├── server/
│   ├── init.server.luau
│   └── Systems/
│       ├── PlayerData.luau     load/save, session locking
│       ├── WaterproofStat.luau +1/s ticker, multiplier stack
│       ├── Gates.luau          gate registry, position validation, wins, checkpoints
│       ├── Rebirth.luau
│       ├── ShowerBooths.luau   zone membership → booth multiplier
│       ├── JacketShop.luau     purchase validation, equip, jacket assignment
│       ├── Pets.luau           eggs, hatching, equip
│       ├── Codes.luau
│       ├── Monetization.luau   gamepasses, developer products, receipts
│       └── Leaderboards.luau
└── client/
    ├── init.client.luau
    └── Controllers/
        ├── WaterfallFX.luau    strip parting, mist, sound
        ├── GatePassage.luau    local barrier toggle, slowdown, soak, shove
        ├── JacketFX.luau       soak tint, drips, bead particles
        ├── FirstClearCamera.luau
        ├── HUD.luau            stat, multipliers, popups
        ├── ShopUI.luau
        ├── PetsUI.luau
        └── Onboarding.luau     guide arrow, first-gate prompts
```

Key attributes on the player (server-set, replicated): `Waterproof`, `Multiplier`, `HighestGateCleared`, `Rebirths`, `JacketTier`, `Soak` (optional; soak can be purely client-side).

### Security notes
- The server owns all stat, win, rebirth, purchase and pet logic. The client never sends its own stat.
- The local barrier toggle is a UX convenience only. Server position validation is the real check.
- Validate and rate-limit every remote (shop, hatch, equip, codes, rebirth).
- Gamepass/product effects granted only via server-side ownership checks and `ProcessReceipt`.

### Performance notes
- Only animate waterfalls within ~150 studs. StreamingEnabled on; handle gate models streaming in/out (CollectionService added/removed).
- Bead/drip particles are client-only; pool or toggle emitters instead of creating instances per droplet.
- Avoid per-frame allocations in the parting loop.

## 7. Suggested build order

1. Project scaffold (Rojo, strict Luau, bootstrap, Remotes, Config, NumberFormat).
2. PlayerData + WaterproofStat (ticker, attribute, leaderstats, saving).
3. **One complete gate:** model structure, WaterfallFX parting, GatePassage (barrier toggle, slowdown, soak, shove), server Gates validation. Get this feeling great before building more gates.
4. JacketFX (game jacket, soak tint, beads) and FirstClearCamera.
5. World 1 layout: 10–15 gates + boss, checkpoints, onboarding arrow, first-gate tuning (fail once, win once).
6. Wins, Rebirth, JacketShop.
7. ShowerBooths (incl. VIP).
8. Monetization (gamepasses, starter pack).
9. Pets (ducks), Codes, Leaderboards.
10. Mobile UI pass, number formatting everywhere, balancing pass on Config.
11. QA: multi-client playtest (gates parting for other players, exploit check on gate skipping), data-loss testing.

## 8. Open questions

- Wins per gate: only on first clear each rebirth cycle, or every time? (Leaning first clear per cycle.)
- Exact requirement curve and rebirth multiplier formula. Needs a balancing pass so the first rebirth lands around 15–20 minutes.
- How to handle Roblox's ~20-minute idle disconnect for AFK booth players. Research what comparable games do.
- Jacket as Accessory vs. welded MeshPart (and how it sits on R15 vs. different body types).
- Should pushback ever send players further than the front of the current gate? (Current plan: no, just a few studs.)
- Final game title, icon and thumbnail (e.g. a character under a waterfall with beads flying off).
- Ship-speed target: the trend is the hook, so aim for a playable launch within a few weeks and cut pets/leaderboards to the first update if needed.
