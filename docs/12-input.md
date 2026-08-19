**English** · Deutsch (translation pending) · 한국어 (translation pending)

---

# 12 - Input contexts, and the state discriminator problem

A mod on this engine repeatedly needs to answer one question: **what is the
player doing right now?** On foot, driving, riding as a passenger, flying a
helicopter, piloting the drone, holding binoculars, sitting in a menu. Almost
every behavioural change wants to be conditional on that, and a camera
modification wants it badly, because the correct camera for a person on foot is
the wrong camera for someone in a drone.

The engine exposes no scripting layer, no console, and no state query. So the
state has to be found.

This file documents what was established about the engine's own named input
contexts: the complete list, how to reach the object that owns them without a
hardcoded address, and an **unresolved conflict** about what the cached values
actually are. The conflict is left in rather than resolved on paper, because
both readings came from real evidence and the wrong one is reachable from the
same starting point.

**Build.** Everything below is pinned to the retail build with
`TimeDateStamp 0x6A75F2F4`, `SizeOfImage 0x18B09000`. Addresses are given as
VAs at the default image base `0x140000000`, with the RVA where it matters.
Per the accuracy notes in the README, prefer the byte signature over any
address here.

---

## The twelve contexts

`[VERIFIED]` A registrar caches twelve named input-context values into one
object at **8-byte stride, `+0x0EB8` through `+0x0F10`**, in registration
order.

| Offset | Name |
|---|---|
| `+0x0EB8` | `Empty` |
| `+0x0EC0` | `OnFoot` |
| `+0x0EC8` | `FastTuning` |
| `+0x0ED0` | `Binoculars` |
| `+0x0ED8` | `VehiclePassenger` |
| `+0x0EE0` | `Vehicle` |
| `+0x0EE8` | `Helicopter` |
| `+0x0EF0` | `Airplane` |
| `+0x0EF8` | `Drone` |
| `+0x0F00` | `SquadTactics` |
| `+0x0F08` | `Menu` |
| `+0x0F10` | `PopUp` |

**How each row was established.** The registration block spans RVA
`0x0E0FCB96` to about `0x0E0FCE7D` and is a repeated three-instruction triple.
Each offset was read from a decoded `mov [rdi+off], rax`, and each name from the
decoded rip-relative target of the `lea` in the same triple. So the offset and
the name come from the same instruction group rather than from two lists lined
up by eye.

Two of the strings, `Vehicle` and `Drone`, are linker-pooled outside the block
at `0x14393AF38` and `0x14393AF30`. The gaps they leave behind are zero
padding, confirmed by raw byte dump. They are **not** missing entries, and a
walker that treats a zero as a terminator will stop early and report ten
contexts.

The independent cross-check is the constructor described below, which stores to
`+0xEE0`, `+0xEE8`, `+0xEF0` and `+0xEF8` in table order, matching `Vehicle`,
`Helicopter`, `Airplane`, `Drone` in this table exactly.

**Why the list itself is useful even without anything else here.** Whatever
field ends up holding the active state must hold one of these twelve values. So
the discriminator can be found by matching against known values rather than by
guessing what a numeric id means, which is a much shorter search.

---

## Reaching the owner with no hardcoded address

`[VERIFIED]` The registrar does not own the contexts. It **asks** for them.

It loads a singleton pointer from a global, lazy-initialises it when null, then
calls **vtable slot `+0x28`** once per context name to get that context's value,
storing each result into the table above.

The global is at VA `0x144D884E8` (RVA `0x04D884E8`). A byte dump of
`0x144D88480..0x144D8850F` is all zeroes **in the file**, so this lives in
`.bss` and is populated at runtime. For an in-process mod that is the easy case:
it is directly readable once the game is running, with no ownership walk needed
to find it.

### The signature

This 27-byte pattern matches **exactly once** in the whole image:

```
48 8B 0D ?? ?? ?? ?? 48 8B 01 48 8D 15 ?? ?? ?? ?? FF 50 28 48 89 87 B8 0E 00 00
```

Decoding as:

```
mov  rcx, [rip + X]        ; the manager global
mov  rax, [rcx]            ; its vtable
lea  rdx, [rip + "Empty"]  ; first context name
call [rax + 0x28]          ; name -> value
mov  [rdi + 0xEB8], rax    ; store into the table
```

From a match at address `A`, recover the global with:

```
global = A + 7 + *(int32 *)(A + 3)
```

which reproduces `0x144D884E8` on this build. The 10-byte tail
`FF 50 28 48 89 87 B8 0E 00 00` is unique on its own as well, so there is a
shorter fallback anchor if the longer one ever breaks.

This matters more than the address does. A signature survives a rebuild that
moves every function; the address does not. Both retail builds examined for this
project differ in every function address.

---

## What a cached value actually is: UNRESOLVED

`[UNKNOWN]` Two readings were reached independently from the same object, both
backed by real evidence, and they cannot both be correct. This section exists so
that nobody spends a week rediscovering the fork.

### Reading A: pointers to engine objects

`[VERIFIED]` The cached values are **pointers to `0xD0`-byte engine objects**,
not integer ids. The engine dereferences them:

```
mov r14, [rbp + 0xEE8]
lea rbx, [r14 + 0xA0]
```

Supporting layout, all from disassembly:

- **`0xD0` allocation per context**, read off a `mov ecx, 0xD0` in the
  constructor loop at `0x14C6ECF20`, which allocates one object per slot and
  stores them in table order.
- **`+0x12`** is a u16 item count and **`+0xA0`** is a lock. The iterator at
  `0x14F815981` locks `+0xA0`, reads the count at `+0x12`, and walks items
  downward.
- Manager **`vtable+0x18`** returns an int32 that is cached at the global
  `0x144D882A4` and compared against its previous value to decide whether to
  rebuild. It is a **count, not a state**, which is worth knowing because it
  looks like a state at a glance.

The manager singleton is confirmed three separate ways: statically from the
registrar, at runtime by the signature above, and again from an unrelated
rebuild loop at `0x14E117D90` that loads the same global.

### Reading B: Steam Input action sets

`[INFERRED]` The object is **Steam's input interface rather than an engine
manager**, and the twelve names are **Steam Input action sets**. Three supports:

1. A runtime dump read the object's `+0x000` vtable pointer as
   `0x00007FF8985B0A70`. In that session GRW.exe was based at
   `0x00007FF7CC8D0000` with `SizeOfImage 0x18B09000`, so the image ends at
   `0x00007FF7E53D9000`. **That vtable is outside the game's image**, so it
   belongs to some other loaded module, and the string `SteamController003` at
   `+0x020` names a candidate.
2. `ISteamInput` (renamed from `ISteamController`, which is why that version
   string reads the way it does) exposes
   `GetActionSetHandle(const char *pszActionSetName)`. The registrar's shape is
   exactly that call: `lea rdx, [rip + name]`, `call [rax+0x28]`, store the
   result.
3. `OnFoot`, `Vehicle`, `VehiclePassenger`, `Helicopter`, `Airplane`, `Drone`,
   `Binoculars`, `SquadTactics`, `Menu`, `PopUp` is how an action set list gets
   named. An engine-internal enum rarely carries `FastTuning` and `PopUp` in the
   same list as `Airplane`.

### Why they conflict, and which is currently ahead

`ISteamInput::GetActionSetHandle` returns an **opaque `uint64`**. Steam would
not own an engine iterator, a per-object lock, or a rebuild counter. So Reading
A's evidence is not something Reading B can absorb.

Equally, an engine manager's vtable should live **inside** the game image, and
the one that was dumped did not.

**Reading A has the stronger evidence class** (disassembly of the engine's own
dereference and constructor) **and is currently treated as ahead.** Reading B is
inference from a runtime dump. One of the two is looking at the wrong object.

### The decisive test, which has not been run

One log line, no play session required:

> Resolve the manager, read `*(void **)obj`, and identify which **module** that
> address falls in by walking the loaded module list, rather than comparing
> against GRW.exe alone.

If it resolves inside `steam_api64.dll`, Reading B is right and what was dumped
is not the engine manager. If it resolves inside GRW.exe, Reading B is wrong,
the Steam strings nearby are incidental neighbouring data, and Reading A stands
on its own.

Run that before committing to either route, because they lead into different
modules and only one of those exists to be searched.

**Cheap thing to check first.** Steam Input may be entirely inert with mouse and
keyboard and no Steam-configured controller. If no controller handle exists,
Steam's action sets are not what drives the engine's state, and Reading B is the
wrong road regardless of what the vtable says.

---

## Still open: which context is active

`[UNKNOWN]` Everything above concerns the **cache** of all twelve contexts.
Which one is **currently active** is not established here.

Two routes, and the trap that sits in front of both:

- **Diff the object across labelled states.** Dump it while on foot, then in a
  vehicle, then in the drone, and look for the field that changes. The trap: if
  Reading B is right, a Steam flat-API shim is unlikely to hold the game's
  current action set as a plain field, so **the diff can come back empty and
  mean nothing**. An empty diff reads as "no discriminator here" when it
  actually means "wrong place to look". That failure shape has already cost this
  project once.
- **Hook the transition instead of hunting for a field.** Something has to
  activate a context when the player's state changes. That is a call on a vtable
  that already resolves reliably, so the transition becomes directly observable
  rather than inferred from a stored value. This is the better route if the
  first one comes back empty, and arguably the better route to start with.

---

## Carrying this to other Anvil titles

Treat this as extrapolation and see
[11-other-anvil-titles.md](11-other-anvil-titles.md) for the general caveats.

The part most likely to travel is not the offsets and not the signature. It is
the **shape**: a registrar that resolves a fixed list of named input contexts
once, by string, through a vtable call, and caches the results at a contiguous
stride. If a sibling title has that shape, the same three-instruction triple is
what to search for, and the names it passes will tell you that engine's state
vocabulary without any further work.

If Reading B turns out to be correct, the finding generalises considerably
further than Anvil, because Steam Input action sets are not engine-specific at
all.
