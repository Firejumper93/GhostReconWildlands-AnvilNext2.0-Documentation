# Ghost Recon Wildlands / AnvilNext 2.0 engine documentation

Reverse-engineering notes on Ubisoft's AnvilNext 2.0 as shipped in *Tom Clancy's
Ghost Recon Wildlands* (2017, x86-64, Direct3D 11).

Everything here was derived first-hand from the shipped executable, the shipped
shaders and the shipped data archives, then verified. It is published because
the engine is otherwise undocumented, because every one of these facts cost real
time to establish, and because the negative results in particular are the kind of
thing nobody ever writes down.

Wildlands is a useful reference point for the Anvil family. It shipped in March
2017, early in the AnvilNext 2.0 era and before the run of Assassin's Creed
titles most Anvil tooling targets, so it shows what the engine looked like
before those changes while still being recognisably the same engine.

---

## Accuracy disclaimer, up front

**This is best-effort reverse engineering. It is not documentation from Ubisoft,
it has not been reviewed by anyone with access to the source, and parts of it
are may not be 100% accurate, but this is an ongoing research project.**

Read it in that spirit:

- **Findings were verified to the standard stated in each claim, and no
  further.** "Verified" here means a specific artifact backs it up. It does not
  mean it was independently reproduced by someone else, or that it holds in
  cases nobody tested.
- **This engine has already proven capable of producing confident wrong
  answers.** Several conclusions in these files replaced an earlier version that
  was plausible, internally consistent, and false. Where that happened the
  correction is kept in the text rather than edited out, precisely because the
  wrong reading was reasonable and someone else will reach it too.
- **Addresses go stale.** RVAs are per-build. A game update moves all of them,
  and two retail builds of this game already differ in every function address.
- **Some of it is inference.** `[INFERRED]` means exactly that: reasoned from
  verified facts, load-bearing, and not proven.
- **File 11 is the least reliable and says so at the top.** Cross-title claims
  are extrapolation from a single title.
- **Absence of a finding is not evidence.** An `[UNKNOWN]` usually means nobody
  looked hard enough yet, not that the answer is unknowable.

If you build on this, re-derive anything load-bearing against your own binary
before you depend on it, and prefer the byte signatures over the addresses,
which is what they are for.

**Corrections are genuinely welcome.** Being wrong in public and fixed beats
being wrong in private. Open an issue with what you measured and how.

---

## How to read this

Every claim carries a confidence tag, and they mean exactly what they say:

| Tag | Meaning |
|---|---|
| `[VERIFIED]` | Backed by a specific artifact: a disassembly, a byte pattern, a shader listing, a round-trip test, or a runtime measurement. Pointable-at evidence. |
| `[INFERRED]` | A reasoned conclusion from verified facts. Plausible, load-bearing, not proven. |
| `[UNKNOWN]` | Open. Recorded so nobody assumes it was answered. |
| `[VERIFIED NEGATIVE]` | Something that is definitively **not** true, or an approach that definitively does not work. |

The negatives are not filler. On a closed engine, knowing an approach is dead is
worth as much as knowing one works, and there is a whole file of them.

**Addresses are RVAs and are build-specific.** Two builds appear here and are
always labelled: the 2017-era retail build, and the 2026-08 update. Where a
structure survived the recompile but moved, both addresses are given.

---

## Contents

All notes live in [`docs/`](docs/).

| File | What is in it |
|---|---|
| [01-binary.md](docs/01-binary.md) | PE layout, the scrambled section table, the jump-thunk call architecture, coordinate system, the engine task graph |
| [02-formats.md](docs/02-formats.md) | The `.forge` container, the Anvil data-file block format, the LZO variant, the checksum, the multi-resource container, base-vs-patch resolution |
| [03-skeleton.md](docs/03-skeleton.md) | The `.Skeleton` asset format, the runtime rig descriptor, CRC32 bone naming, the 143-entry biped bone enum, HumanIK |
| [04-pose.md](docs/04-pose.md) | The Pose object, the bone transform buffer, model vs world space, the per-frame skeleton chain, where a bone write survives |
| [05-camera.md](docs/05-camera.md) | The camera struct, the nine matrices, the authoritative transform, the six projection functions |
| [06-weapons.md](docs/06-weapons.md) | The attachment chain end to end, `TransformNode::SetWorldTransform`, weapon component layout, projectiles, Havok |
| [07-rendering.md](docs/07-rendering.md) | The three geometry placement paths, the GPU bone palette, shader permutation keys, register slots and strides |
| [08-reflection.md](docs/08-reflection.md) | The class/property reflection tables, and recovering engine names from them by CRC32 |
| [09-methodology.md](docs/09-methodology.md) | The techniques that worked, in the order they are worth trying |
| [10-negatives.md](docs/10-negatives.md) | Everything definitively **not** true, and every approach that definitively fails |
| [11-other-anvil-titles.md](docs/11-other-anvil-titles.md) | **Extrapolation.** What should carry to other Anvil / AnvilNext 2.0 games, what will not, a porting checklist |

---

## How to use this

### If you want to read the archives

Start with [02-formats.md](docs/02-formats.md). It is a complete specification
for reading and, importantly, **writing** a `.forge`: container layout, the
block-set payload format, the exact LZO variant, the non-standard checksum, and
the multi-resource directory inside a payload.

The non-obvious parts, which were learned by breaking the game, are the writer
constraints: preserve the original block-set structure and compression byte,
replace entries only, address entries by file id rather than name, and pad the
tail to `0x8000`.

### If you want to work with characters or animation

[03-skeleton.md](docs/03-skeleton.md) has a byte-complete `.Skeleton` format
with zero unexplained bytes, and the bone names resolve to the standard Autodesk
HumanIK convention. [04-pose.md](docs/04-pose.md) has the runtime side: how to
get from a skeleton pointer to an animated bone's world position, and which
buffer is authoritative versus which is a stale mirror.

### If you want to write a camera mod

[05-camera.md](docs/05-camera.md) has the authoritative camera transform, what
happens when you write it (tested), what happens when you write the obvious
wrong one (also tested), and the one visible artifact it produces.

### If you want to hook something

[01-binary.md](docs/01-binary.md) describes the jump-thunk architecture, which
is the cleanest interception surface on this binary and better than a prologue
detour for reasons specific to it. [09-methodology.md](docs/09-methodology.md)
has the verification discipline that keeps a game update producing a clean
refusal instead of a crash.

### If you want to save a week

Read [10-negatives.md](docs/10-negatives.md) first. It is the shortest path to
not repeating work that has already been done and failed.

### If you are working on a different Anvil title

[11-other-anvil-titles.md](docs/11-other-anvil-titles.md), with its own warning
attached. It has the handful of direct cross-title comparisons that were
actually made, a porting checklist, and an honest split between what should
carry and what definitely will not.

---

## Ongoing projects

This documentation is a byproduct. It exists because these projects needed the
answers.

### GRW-XR, a native OpenXR VR conversion (in development, alpha releases public)

A native VR mod for Wildlands: real OpenXR, stereo rendering on the game's own
D3D11 device, head-driven camera, Touch controllers, and a first-person camera
anchored to the character's animated head bone.

Status at the time of writing: playable in alpha, with a headset-confirmed
result that as far as this research could establish **has no prior art on this
engine family**: the in-game weapon follows the motion controller 1:1 in
position and rotation, driven by a bone write at the engine's own attachment
publish. The honest gap, documented rather than glossed: bullets still follow the
camera rather than the gun. That work is ongoing.

**Releases, installers and issue tracker:
<https://github.com/Firejumper93/GhostReconWildlandsVR>**

### GRW-FP, a flat-screen first-person mod (in development, unreleased)

The non-VR sibling. The same head-bone camera anchor for ordinary flat-screen
play, with the game's own mouse look keeping full authority over aiming, so
ballistics, the crosshair and every aim-anchored UI element behave exactly as
shipped. It moves the camera position and nothing else.

It exists because that restriction makes it a much smaller problem than the VR
conversion: almost all of the VR work is rotation work, and none of it is needed
for flat first person.

### Archive and asset tooling (private, findings published here)

A reader and byte-identical writer for the `.forge` container and the Anvil data
files, built to answer questions the runtime could not. Its results are what
[02-formats.md](docs/02-formats.md) documents: a no-op rebuild reproduces all 21
archives of a 62 GB install byte for byte, and a rebuilt archive with re-encoded
payloads and reflowed offsets loads correctly in the retail game.

The tooling itself is not published. The format knowledge is, in full, so anyone
can build their own.

---

## Scope, and what is deliberately not here

This is engine architecture documentation. It is not a modding cheat sheet and
it is not a toolkit.

Deliberately excluded, and not available on request:

- **Anything about the copy protection.** No anti-tamper analysis, no trigger
  locations, no bypass discussion. The work this came from ran inside the
  normally-protected process without touching that machinery, and the notes on
  it stay private.
- **Anything anti-cheat adjacent.** No accuracy, spread, damage or aim
  manipulation. The projects this came from are strictly single-player, and
  publishing that class of address serves cheating and nothing else.
- **Third-party derived material.** Notes taken from other people's
  permission-restricted mods, decompiled tools, or cheat tables are not
  republished here, whatever their licence status.
- **Any game content.** No binaries, no extracted assets, no archive dumps, no
  shader blobs. Readers are expected to own the game and extract their own.

---

## Provenance and method

The work was done for the VR conversion above, over roughly thirty working
sessions. Static analysis ran offline against a copy of the executable; runtime
facts were established by hooking, measuring, and then re-testing, under the
standing rule that an in-headset or on-screen outcome always outranks a log line
or a plausible-looking piece of reasoning.

A large fraction of what is written here exists because an earlier confident
version of it turned out to be wrong. Those corrections are kept in the text
rather than edited out, because the wrong readings were reasonable and someone
else will reach them too.

---

## Links and support

- **The VR mod** (releases, installer, issues):
  <https://github.com/Firejumper93/GhostReconWildlandsVR>
- **Buy me a coffee**: <https://buymeacoffee.com/firejumper93>

Support is entirely optional and never required. Everything here and everything
in the VR mod is free, and stays free. If this documentation saved you a week of
staring at a disassembler, that is already the point of publishing it.

---

## Licence

Notes and prose: **CC BY 4.0**. Attribute and use freely, including in
commercial work.

No game code, game data or third-party code is included or redistributed. Ghost
Recon, Wildlands, AnvilNext and Anvil are trademarks of Ubisoft Entertainment.
This is unofficial documentation produced by interoperability analysis, and is
not affiliated with, endorsed by, or supported by Ubisoft.
