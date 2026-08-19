**English** · Deutsch (translation pending) · 한국어 (translation pending)

---

# 12 - Input contexts, and reading the player's active state

A mod on this engine repeatedly needs to answer one question: **what is the
player doing right now?** On foot, driving, riding as a passenger, flying a
helicopter, piloting the drone, holding binoculars, sitting in a menu. Almost
every behavioural change wants to be conditional on that, and a camera
modification wants it badly, because the correct camera for a person on foot is
the wrong camera for someone flying a drone.

The engine exposes no scripting layer, no console, and no state query. So the
state has to be found.

It was found. **The active input context is a plain `int32` at a fixed offset in
a singleton**, and reading it costs one dereference chain and no engine call.
Getting there took an unusually instructive wrong turn, which is documented here
rather than tidied away, because the wrong turn is reachable from exactly the
same starting point and someone else will take it.

**Build.** Pinned to the retail build with `TimeDateStamp 0x6A75F2F4`,
`SizeOfImage 0x18B09000`. Addresses are VAs at the default image base
`0x140000000`. Prefer the byte signatures over the addresses, which is what they
are for.

---

## The short version

```
active_context_index = *(int32 *)(InputContextDispatcher + 0x9C0)
```

`Drone == 8`. The index addresses the twelve-slot table below as
`base 0xEB8 + idx*8`.

Everything after this is how that was established, what the neighbouring objects
are, and what is still genuinely unknown.

---

## Three objects, not one

`[VERIFIED]` This is the single fact that unlocks the rest, and the one whose
absence made two independent analyses appear to contradict each other for a
while. **Three distinct objects are involved**, reached by different routes, and
two of them are easy to mistake for each other because their contents overlap in
the same offset range.

| Object | Reached via | Vtable | Lives |
|---|---|---|---|
| **Steam input manager** | global `0x144D884E8` | `0x00007FF8985B0A70` at runtime | **outside GRW.exe** |
| **InputContextOwner**, size `0x10B0` | constructed, one per local player | `0x143AB2360` | inside GRW.exe |
| **InputContextDispatcher**, size `0x9D0`, singleton | `*(*(0x144D84E98) + 0x258)` | `0x143AB23F0` | inside GRW.exe |

Note the two globals: `0x144D884E8` is the Steam manager, `0x144D84E98` is the
dispatcher root. They differ by more than a typo, they were derived
independently, and confusing them wastes a session.

The `InputContextOwner` holds the twelve cached context values. The
`InputContextDispatcher` holds a vector of owners at `+0x9B0` and **the active
index at `+0x9C0`**.

---

## The twelve contexts

`[VERIFIED]` A registrar caches twelve named input-context values into the
`InputContextOwner` at **8-byte stride, `+0x0EB8` through `+0x0F10`**, in
registration order.

| Index | Offset | Name |
|---|---|---|
| 0 | `+0x0EB8` | `Empty` |
| 1 | `+0x0EC0` | `OnFoot` |
| 2 | `+0x0EC8` | `FastTuning` |
| 3 | `+0x0ED0` | `Binoculars` |
| 4 | `+0x0ED8` | `VehiclePassenger` |
| 5 | `+0x0EE0` | `Vehicle` |
| 6 | `+0x0EE8` | `Helicopter` |
| 7 | `+0x0EF0` | `Airplane` |
| **8** | `+0x0EF8` | **`Drone`** |
| 9 | `+0x0F00` | `SquadTactics` |
| 10 | `+0x0F08` | `Menu` |
| 11 | `+0x0F10` | `PopUp` |

**How each row was established.** The registration block spans RVA
`0x0E0FCB96` to about `0x0E0FCE7D` and is a repeated three-instruction triple.
Each offset was read from a decoded `mov [rdi+off], rax`, and each name from the
decoded rip-relative target of the `lea` in the same triple. Offset and name
come from the same instruction group rather than from two lists lined up by eye.

**The index column is not an assumption.** The engine literally addresses
`base 0xEB8 + idx*8` with a caller-supplied index, as
`mov r8, [rbx+rsi*8+0xEB8]`, at `0x14E0E92AF` and `0x14E0EDC5F`. The arithmetic
self-proves against a known value: the dispatcher's constructor initialises the
active index to `0x0A`, and `0xEB8 + 10*8 = 0xF08`, which is the `Menu` slot.
Starting in the menu is exactly right. By the same rule `Drone` is
`0xEB8 + 8*8 = 0xEF8`.

**A trap in the table.** Two of the strings, `Vehicle` and `Drone`, are
linker-pooled outside the block at `0x14393AF38` and `0x14393AF30`. The gaps
they leave behind are zero padding, confirmed by raw byte dump. They are **not**
missing entries, and a walker that treats a zero as a terminator will stop early
and report ten contexts.

---

## Reading the active context

`[VERIFIED, statically]` The active context is an `int32` index at
`InputContextDispatcher + 0x9C0`. Three independent confirmations of that
field's meaning:

- **Setter** at `0x14E12E060` (thunk `0x1417056F0`): early-outs on
  `cmp [rcx+0x9C0], r14d`, loops the owners calling manager `vtable+0x30` with
  `[owner + r14*8 + 0xEB8]`, then commits `mov [rsi+0x9C0], r14d`.
- **Constructor** at `0x14E0FC0EF`: `mov [rdi+0x9C0], 0xA`.
- **Readers** at `0x14E0E9340` and `0x14E0EDDC0`: `mov edx, [rbx+0x9C0]` fed
  straight into the indexed lookup to resolve an action binding.

Two signatures, each matching exactly once in the image:

```
constructor : C7 87 C0 09 00 00 0A 00 00 00
setter entry: 48 89 E0 56 41 56 48 83 EC 58 4C 63 F2 48 89 CE 44 39 B1 C0 09 00 00
```

**Honest limit.** Every claim in this section is verified against the binary and
**not** against the running game. The live confirmation step, logging the value
and watching it read 8 as a drone launches, had not been completed at the time
of writing. Treat the offset as static-verified and confirm it yourself before
depending on it.

### Negatives closed at the same time

`[VERIFIED NEGATIVE]` Worth recording so nobody repeats the search:

- **There is no hardcoded "switch to Drone" site.** Switching is data-driven
  through a name-to-index map at `0x149161972`.
- **There is no "active" flag inside the per-context object.** One integer names
  the winner, which makes the question moot.
- Of the nine `Drone` string references, seven are dead ends: three
  Denuvo-mutated argument stubs, a static property-bag entry, a
  bitmask-to-name mapper, and two report builders.
- The global `0x144D882A4` is a **local-player count**, not a state.

---

## What the manager is, and the wrong turn worth documenting

`[VERIFIED, in game]` The registrar does not own the contexts. It **asks** for
them, and what it asks is **not part of the game at all**.

It loads a singleton from the global `0x144D884E8`, lazy-initialises it when
null, then calls **vtable slot `+0x28`** once per context name. That call is
**`ISteamInput::GetActionSetHandle(const char *)`**. The twelve names are Steam
Input action sets.

The evidence chain, in the order it actually happened:

1. A runtime dump read the object's `+0x000` vtable pointer as
   `0x00007FF8985B0A70`. In that session GRW.exe was based at
   `0x00007FF7CC8D0000` with `SizeOfImage 0x18B09000`, so the image ends at
   `0x00007FF7E53D9000`. **That vtable is outside the game's image.** The string
   `SteamController003` sits at `+0x020`, naming the module. (`ISteamInput` was
   renamed from `ISteamController`, which is why that version string reads the
   way it does.)
2. This was initially only `[INFERRED]`, and it was disputed, because separate
   disassembly showed values in that offset range being **dereferenced as
   pointers** to `0xD0`-byte engine objects with a lock at `+0xA0` and a u16
   count at `+0x12`. An opaque Steam handle is not a pointer, so both readings
   could not describe the same slot.
3. **A crash settled it.** A mod called `vtable+0x28` with `"Drone"`, received
   **8**, stored it as an object address, and dereferenced it. The result was
   three `0xC0000005` access violations reading address `0x18`, which is
   `8 + 0x10`. One log line carried the whole answer:
   `ctx: Drone context object at 0x000000000008`.

A value of `8` is not an address. `vtable+0x28` returns an opaque handle.

**And then the two numbers met.** The Steam handle for `Drone` is **8**. The
static index for `Drone` derived from the addressing arithmetic is also **8**.
Two entirely independent derivations, one static and one live, produced the same
number. That is corroboration rather than coincidence.

> **Method note, offered because it paid for itself.** The crash was only
> legible because the mod carried a loud vectored exception handler that printed
> "the fault is inside our own DLL" with the faulting address. An invisible
> wrong assumption became a one-line answer. The standing objection that such a
> handler is a frame-time risk is unchanged, and so is its value.

The cheaper test that would have reached the same place, and is still the right
first move on any sibling title: resolve the manager, read `*(void **)obj`, and
name the **module** that address falls in by walking the loaded module list
rather than comparing against the main image alone.

**Note this does not gate the active-context read.** The dispatcher chain never
touches the Steam manager. Anyone who wants the active context can read
`dispatcher + 0x9C0` today without resolving anything about Steam.

---

## Still genuinely unknown

`[UNKNOWN]` **What the twelve cached values at `+0xEB8..+0xF10` actually are.**

The registrar fills them with Steam action-set handles, which are opaque
`uint64`s. Yet disassembly plainly shows values in that same offset range being
dereferenced as pointers to `0xD0`-byte objects, with an iterator at
`0x14F815981` that locks `+0xA0` and walks a u16 count at `+0x12`, and a
constructor at `0x14C6ECF20` allocating `0xD0` per slot in table order.

The likeliest resolution is that **two different classes share the
`0xEB8..0xF10` offset range**, which is supported by writes across `0xEA8..0xF90`
at `0x14B4EEF24` and `0x14D7818BA` that belong to a different class. Not proven.

Practical consequence: **do not assume a value read from that range is a
pointer, and do not assume it is a handle.** Check which object you are holding
first. The access violation described above is precisely what the wrong
assumption costs.

---

## Neighbouring findings

`[VERIFIED]` **The registrar continues into an action-name table.** Past the
twelve contexts, the same registrar registers action names with the identical
three-instruction shape, at a separate offset range from `+0x0B98` onward
(`nitro` lands at `+0x0E70`): `accelerate`, `brake`, `throttleUp`,
`throttleDown`, `descend`, `ascend`, `nitro`, `interrogate`, `interact`,
`swapShoulderView`, `textChat`, `pushToTalk`, `openTacmap`, `openLoadout`.

Shape alone cannot separate contexts from actions. The offsets can: contexts are
contiguous at 8-byte stride, actions are not. `throttleUp`, `throttleDown`,
`ascend` and `descend` give an aircraft-state check named actions to test
against, and `swapShoulderView` is a camera action.

`[VERIFIED]` **A `DroneContextType` / `isStart` property bag exists.** At
`0x1525E4152` a repeating string-map pattern builds a bag whose resolved
literals are, in order, `"Drone"`, then `type` to `DroneContextType`, then
`isStart` to `true`. So the engine has an explicit named notion of a drone
context **starting**, with a boolean separating start from end.

`[UNKNOWN]` whether that is a runtime event emission a mod could observe, or
initialisation of a static description table. The surrounding code is
string-construct, map-insert, string-destroy per entry, which reads more like
table building than a hot path. Do not assume the former.

---

## Carrying this to other Anvil titles

Treat as extrapolation, and see
[11-other-anvil-titles.md](11-other-anvil-titles.md) for the general caveats.

The offsets will not travel. Two things plausibly will:

- **The registrar shape**: a fixed list of named contexts resolved once, by
  string, through a vtable call, cached at a contiguous stride. The names it
  passes tell you that engine's state vocabulary with no further work.
- **The separation of concerns**: a per-player owner holding the cache, and a
  singleton dispatcher holding the active index. If a sibling title has the
  first, look for the second rather than hunting for a flag inside the contexts.

The Steam Input part generalises furthest of all, because Steam Input action
sets are not engine-specific in the slightest. If a title routes its input
contexts through Steam, the same `GetActionSetHandle` shape is what to look for,
and the same trap applies: the return value is a handle, not a pointer.
