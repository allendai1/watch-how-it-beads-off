# +1 Waterproof Every Second

Roblox "+1" game based on the TikTok trend "watch how it beads off". Players' Waterproof stat rises +1/s; waterfall gates part around them, and they pass when the gap is wider than the doorway.

**Before any task, read `docs/DESIGN.md`.** It is the source of truth for mechanics, scope, scaling and build order. If a request conflicts with it, say so before writing code.

## How we work
- **Plan first.** For any new system or multi-file change, propose a plan and wait for approval before writing code.
- **One system per task.** Stay within the system being worked on. Don't refactor unrelated code without asking.
- **Small fixes stay small.** For a bug, explain the cause before changing anything. No large rewrites to fix small problems.
- **Every system ships with a Studio checklist**: the exact Instances, names, tags and attributes the code expects, so the human can build or verify them in Studio.
- **After finishing a system**, run the `roblox-dev:roblox-reviewer` agent on the changed files and fix what it flags.
- **Keep `docs/DESIGN.md` current.** When a design decision changes or an open question gets answered, update the doc in the same task.
- The human tests in Studio and reports results. Ask for Output logs if a bug report doesn't include them.

## Project conventions
- Follow the installed `roblox-dev` plugin skills: architecture, security, datastores, performance, strict typing, UI, testing.
- Rojo layout: `src/shared`, `src/server`, `src/client`. Exactly one server Script (`init.server.luau`) and one client LocalScript (`init.client.luau`); everything else is a ModuleScript with `init()`/`start()`.
- `--!strict` in every file. Shared types live in `src/shared/Types.luau`.
- Remotes are created on the server from the list in `src/shared/Remotes.luau`. Never `Instance.new("RemoteEvent")` anywhere else.
- No `_G` or `shared`. Require modules explicitly.
- World objects (gates, booths) are found via a known folder or CollectionService tag, with per-instance data in Attributes and behavior in one manager module.

## Balancing
- **All gameplay numbers live in `src/shared/Config.luau`**: requirement curve, multipliers, prices, booth tiers, soak time, walk speeds, shove force, rebirth formula. No magic numbers in system code.
- When asked to tune how something feels, change Config values, not logic.
- Display every player-facing number through `Util/NumberFormat`.

## Security (non-negotiable)
- The server owns Waterproof, wins, rebirths, purchases, pets, codes and gate progress. The client never sends its own stat or claims a gate clear.
- The client-side barrier toggle and shove are UX only. Server position validation (`HighestGateCleared`) is the real check.
- Validate types and rate-limit every remote handler.
- Gamepass and product effects are granted only after server-side ownership checks or `ProcessReceipt`.

## Data
- Player data uses session locking, autosave, `BindToClose` and a schema version. Any change to saved data needs a migration.
- Never change the saved data shape without saying so explicitly in the plan.

## Performance
- Waterfall parting and bead/soak effects run client-side only, and only for gates within ~150 studs.
- No per-frame allocations in `RenderStepped` loops. Disconnect connections on cleanup.
- StreamingEnabled is on; handle tagged instances streaming in and out.

## Testing
- Pure logic (`GateMath`, requirement curve, multiplier stack, NumberFormat) lives in `src/shared/Util` with unit tests runnable outside Studio.
- Before marking a system done: lint, format and type-check pass.
