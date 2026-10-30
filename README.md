# SimVM.Luau

An implementation of SimAntics targeting Luau, based on [FreeSO](https://github.com/riperiperi/FreeSO)'s source code.

SimVM.Luau ports the structure of FreeSO's SimAnticsVM to Luau as a standalone library, so any Luau project can embed it, fork it or build its own game on top of it.

It does not load EA / The Sims Online content. There is no BHAV/IFF bytecode interpreter and no original assets. Object behaviors are plain Luau modules supplied by the host game.

## Design rules

These rules apply to every contribution. If a change needs to break one, discuss it first.

1. **Pure Luau, no Roblox APIs.**
   No Instances, no services (`DataStoreService`, `RunService`...), no `task.wait`/`task.spawn`. The VM takes data in (object definitions, saved state, player commands) and emits events out (an avatar moved, an animation started, an object was created). The host game turns those events into parts, animations and network replication.

2. **Interactions are coroutines driven by the VM tick.**
   A behavior blocks on primitives such as `Sleep`, `Animate` or `RouteToSlot` by yielding. Only the VM's scheduler resumes threads, from `vm:Tick()`, never from a real-time clock. Thread state is never serialized: on load, entities resume from their saved attributes and start fresh actions.

3. **Content lives in the host, not here.**
   Object definitions (a fridge, a bed, a pet bowl) belong to the game. The VM only defines the primitive API those definitions call. Test objects live in `tests/`.

## Layout

```
src/
├─ init.luau                  -- public API
├─ Types.luau                 -- shared type definitions
├─ Core/
│  ├─ VM.luau                 -- owns entities, clock and event bus
│  ├─ Clock.luau              -- household/world clock, pause, speed (max 3x)
│  ├─ Scheduler.luau          -- spreads entity ticks across frames
│  ├─ Thread.luau             -- coroutine runner (FreeSO's VMThread)
│  └─ Queue.luau              -- interaction queue, priorities, interrupts
├─ Entities/
│  ├─ VMEntity.luau           -- base for everything on a lot
│  ├─ GameObject.luau         -- furniture and items
│  └─ Character/              -- motives, autonomy, routing, queue
│     ├─ Avatar.luau          -- controllable person (FreeSO's VMAvatar)
│     └─ Pet.luau             -- autonomous only, refuses direct commands
├─ Simulation/
│  ├─ Motives.luau            -- decay, thresholds, failure states
│  ├─ Autonomy.luau           -- advertisement scoring and cache, panic checks
│  ├─ Skills.luau
│  └─ Relationships.luau
├─ Routing/                   -- tile grid, pathfinding, object slots
├─ Primitives/                -- the API behaviors call (Animate, RouteToSlot...)
└─ Serialization/             -- (planned) snapshots to and from plain tables
tests/                        -- Lune specs and dummy objects
```

## Entity model

In SimAntics a person (an *avatar*, what the games call a Sim) is also an object. Avatars and furniture share one base, which is why autonomy can choose "Chat with Bob" the same way it chooses "Use fridge".

```
VMEntity          -- ID, position, attributes, own thread, slots, advertised interactions
├─ GameObject     -- catalog entry, price, footprint and placement, condition (dirty, broken, powered)
└─ Character      -- motives, autonomy, routing, interaction queue, animation
   ├─ Avatar      -- skills, personality, relationships, player-controllable
   └─ Pet         -- autonomous; motives visible, cannot be controlled
```

Rule of thumb: if a bed and an avatar both need it, it goes in `VMEntity`. If an avatar and a pet need it but a bed does not, it goes in `Character`.

## Simulation notes

- **Tick budget.** The VM does not tick every entity every frame. The scheduler splits entities into groups that tick on different frames.
- **Advertisement cache.** An object's advertisement scores are cached and recomputed only when that object's state changes.
- **Autonomy still runs for Characters with queued actions.** A failing motive must be able to interrupt the queue, which is what produces panic states (a bladder failure, collapsing from exhaustion). Queued Characters check motive thresholds every tick and run full advertisement scoring only when a threshold is crossed or the queue empties.

## Conventions

- Use `--!strict` at the top of every module.
- Document public functions with Moonwave-style doc comments:

  ```lua
  --[=[
      Applies motive decay for one tick.
      @param avatar Avatar -- the avatar being updated
      @param dt number -- sim-minutes elapsed
      @return boolean -- true if any motive failed
  ]=]
  ```
- Terminology: people in the simulation are **avatars** (`Avatar` in code), following FreeSO's `VMAvatar`. Keep game-specific names out of the VM; a host game can call them whatever it likes. Mention FreeSO names when pointing at the original source (for example, `Thread.luau` is FreeSO's `VMThread`).

## Credits

SimVM.Luau is based on the source code of [FreeSO](https://github.com/riperiperi/FreeSO), the open-source re-implementation of The Sims Online by riperiperi and the FreeSO contributors. Its SimAntics VM is the reference for this port's architecture and behavior.

FreeSO is licensed under MPL-2.0. Files ported from FreeSO stay under MPL-2.0 and must name the FreeSO file they are based on in a header comment, for example:

```lua
-- Based on FreeSO: <path of the original file>/VMThread.cs (MPL-2.0)
```

## License

[Mozilla Public License 2.0](LICENSE).

## Disclaimer

SimVM.Luau is an independent fan project. It is not affiliated with, endorsed by or sponsored by Electronic Arts Inc. or Maxis. The Sims and The Sims Online are trademarks of Electronic Arts Inc.

This project contains no code, bytecode or assets from Electronic Arts. It does not read or run original game files.
