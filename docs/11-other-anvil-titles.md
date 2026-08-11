# 11 - Using this on other Anvil / AnvilNext 2.0 titles

> **Read this file more skeptically than the others.**
>
> Everything in files 01 to 10 was verified against *Ghost Recon Wildlands*
> specifically. **This file is largely extrapolation.** It is an informed guess
> at what carries across the engine family, written to save you time, not to
> save you from checking. Claims here are marked `[CROSS-TITLE, UNVERIFIED]`
> unless a specific cross-title observation backs them.
>
> Treat every address as wrong until you re-derive it, and treat every structure
> as a hypothesis until your own parser round-trips it.

## Where Wildlands sits in the family

Wildlands shipped in **March 2017**, which puts it early in the AnvilNext 2.0
era, before the run of Assassin's Creed titles that most Anvil tooling targets.
That makes it useful as a reference point in two directions: it shows what the
engine looked like before those titles changed it, and it is close enough to
them that a surprising amount is recognisably the same code.

Titles commonly identified as Anvil or AnvilNext 2.0 include Assassin's Creed
Unity, Syndicate, Origins, Odyssey, Valhalla and Mirage, For Honor, Steep, Ghost
Recon Wildlands and Breakpoint, Immortals Fenyx Rising, Riders Republic and
Skull and Bones. **That list is general knowledge, not a finding of this
research**, and the engine changed substantially across that span. Do not assume
a title is "the same engine" just because it appears on a list.

### Concrete cross-title observations from this work

These are the few places where a direct comparison was actually made, and they
are worth reading closely, because they show the pattern: **the code is often
literally the same, and the offsets are not.**

- `[VERIFIED]` **A projection function signature from Odyssey matched Wildlands
  byte for byte**, one match in the whole 369 MB image. Same function,
  unchanged, across a title and a year. But see the trap below.
- `[VERIFIED]` **Origins-era pseudocode for the camera's `worldMatrixOverride`
  block reproduces in Wildlands instruction for instruction, at a different
  struct offset**: `+0x2A0` here versus `+576` there. The surrounding code is
  identical, which is exactly what makes the wrong offset convincing.
- `[VERIFIED]` **`+Z` up and `+Y` forward match Odyssey and Valhalla exactly**,
  so basis-conversion code written for those titles transfers unchanged.
- `[VERIFIED]` **The "attachment slot" numbers from a Mirage-era guide do not
  transfer.** Wildlands has its own 143-entry `Skeleton::BipedBoneID` enum with
  its own ordinals.
- `[VERIFIED]` **One widely used community toolkit has no Wildlands support at
  all**: a byte scan of its main assembly finds Odyssey, Origins, Steep and
  Breakpoint strings and zero Wildlands hits. Tool coverage is per-title and
  advertised support is worth verifying rather than trusting.
- `[VERIFIED]` **Data-file block size fields are u16 in Wildlands** where at
  least one other title in the same family uses i32. Same format, different
  integer width, silent corruption if you guess.

### The trap that comes with a perfect signature match

`[VERIFIED]` The Odyssey projection signature that matched Wildlands byte for
byte points at a function the gameplay camera **does not call**. The engine has
six projection variants; the one with the famous signature is not the one on the
gameplay path here.

So a perfect cross-title signature match tells you **the function is present and
unchanged**. It does not tell you **the game calls it for the thing you care
about**. Those are different claims and conflating them costs a day.

## What is likely to transfer

`[CROSS-TITLE, UNVERIFIED]` unless noted. Ranked by how confident I would be.

### Very likely: identity and naming

**CRC32 is the universal identifier.** Bone names, class names, property names
and resource class hashes are all plain CRC32 of the name in Wildlands. This is
an engine-wide design decision, not a per-title one, and the same rainbow-table
approach should work anywhere in the family.

The two-source dictionary trick should also carry: a community hash dictionary
alone was **not** enough here, and hashing the image's own plaintext strings
plus token splits of them is what cracked the interesting names. Any title's
executable contains the words even when it does not contain the identifiers.

### Very likely: the reflection system

A runtime reflection system carrying class descriptors and property records as
static data, walked by a single registrar, is core engine architecture. If you
can find the registrar in another title, you get annotated struct layouts for
the whole game.

What to expect to differ: the section it lives in, the record stride, and the
bit-packing of the offset field (Wildlands: stride `0x38`, byte offset =
`packed >> 18`). Find one class whose layout you already know and use it to
calibrate the packing before trusting anything else.

### Very likely: HumanIK and Havok as middleware

HumanIK statically linked with names stripped, present only as data tags and
name tables, is a strong pattern. So is Havok naming its own functions through
profiling-timer string literals. Both give you anchors in a stringless binary.

### Likely: the container and compression stack

The `.forge` container, the block-set data-file layout, and the multi-resource
container inside a payload are the family's storage design.

Expect these to vary per title:

- **The forge `FileVersionIdentifier`** (27 in Wildlands).
- **The compression algorithm.** Wildlands is LZO1X; later titles moved to
  Oodle, and the compression byte enumerates several LZO variants. Do not
  hardcode.
- **Integer widths in the block header** (the u16 versus i32 split above).
- **The checksum.** Wildlands uses Adler-32 with both accumulators initialised
  to **zero** rather than `a = 1`. Whether that quirk is engine-wide or
  Wildlands-specific is `[UNKNOWN]`, and it is cheap to test: compute both over
  a known chunk and see which matches.

### Likely: the skeleton and pose architecture

A shared refcounted rig **descriptor** holding no transforms, a per-character
**Pose** object owning the bone buffer, a sorted hash-to-index name map, and
model-space bone records composed against a root transform. This is a coherent
design and unlikely to have been rewritten wholesale.

Expect the **offsets and the stride to differ**. Wildlands is
`{float4 T, float4 Q}` at stride `0x20`. Confirm yours from the accessor's own
arithmetic (`shl reg, N` gives you the stride directly) rather than from a
struct definition someone posted.

### Plausible but check first: the jump-thunk call architecture

Every engine call in Wildlands goes through a 5-byte `E9 rel32` alone in a
16-byte `int3`-padded slot. That is a build/link characteristic as much as an
engine one, so it may or may not hold for another title even on the same engine.

It is trivial to test: disassemble any call target and see whether you land on a
jmp. If you do, you have the same convenient interception surface; if you do
not, you need conventional techniques.

### Plausible: coordinate conventions

`+Z` up, `+Y` forward, right-handed, 1 unit = 1 metre. Verified to match Odyssey
and Valhalla.

If the title links NVIDIA Ansel, you can get this from the engine's own mouth
rather than inferring it: the Ansel SDK requires a title to declare its
coordinate convention, so intercepting the configuration call and dumping the
struct gives you the declared basis vectors and the world scale directly. That
technique is title-agnostic and takes an afternoon.

## What will definitely not transfer

- **Every RVA.** Two retail builds of *the same game* have every function at a
  different address. Across titles this is not even close.
- **Struct offsets, in general.** Though note the asymmetry observed within
  Wildlands across a nine-year update: **data layout survived where code
  addresses did not.** Four weapon-component property offsets re-derived
  identically while every function moved. So offsets are more portable than
  addresses, and still not portable enough to trust.
- **Enum ordinals.** The biped bone enum is per-title.
- **Asset naming conventions.** `GR_` prefixes, `wb-` weapon bone prefixes and
  `sb_`/`civ_`/`rbl_` animation prefixes are Ghost Recon specific.
- **Anything about a title's protection.** Out of scope here, varies per title
  and per release, and not discussed in this repository.

## A porting checklist

If you want to bring these findings to another title, this is the order that
would have saved the most time in hindsight.

1. **Identify the binary.** Read TimeDateStamp and SizeOfImage from the PE
   header and make them your build pin from the very start. Retrofitting a pin
   after a game update is painful; having one means an update produces a clean
   refusal instead of a crash.
2. **Check whether the section names lie.** Dump the section table and locate
   where the real code and real strings actually live. On Wildlands the code is
   in `.sbss` and the strings are in `.link`. Get this wrong and every scan you
   run afterwards is scoped to the wrong bytes.
3. **Check whether code is plaintext on disk.** If it is, all of your static
   work can happen offline against a copy, with no process involved.
4. **Test the thunk hypothesis.** Disassemble a few call targets and see whether
   they are jmp stubs in padded slots.
5. **Build the CRC32 rainbow table early**, from a community dictionary plus the
   image's own strings. Everything else gets easier once names resolve.
6. **Find the reflection registrar** and dump the class and property tables.
   This is the single biggest force multiplier.
7. **Parse the archives** and pull the material shaders out. Shipped shaders are
   an underused source: they tell you exactly which register slots and strides
   the GPU reads, with no runtime involvement and no capture tool.
8. **Only then** start runtime work, and only to answer specific binary
   questions that static analysis genuinely cannot.

## What this research could be a jumping-off point for

Offered as directions, not as claims that any of them is easy.

**Asset and archive tooling.** The container, block set, compression, checksum
and multi-resource layouts in [02-formats.md](02-formats.md) are a complete
specification for reading and, importantly, **writing** a `.forge`. The writer
constraints established there (preserve set structure and compression byte,
replace entries only, pad to `0x8000`) are the non-obvious part, and they were
learned by black-screening a game.

**Skeleton and animation tooling.** [03-skeleton.md](03-skeleton.md) has a
byte-complete `.Skeleton` format with zero unexplained bytes, and the bone names
resolve to the standard HumanIK convention. That is enough for a rig viewer, a
bind-pose editor, or a converter to a DCC format.

**Camera mods** (FOV, third-to-first person, free camera, cinematic tools). The
authoritative camera transform, what happens when you write it, and the one
visible artifact it produces are all in [05-camera.md](05-camera.md). The
finding that the engine rebuilds the transform every frame is what makes a
per-frame absolute write safe.

**VR conversions.** The camera work is the foundation; the coordinate
conventions and the 1-unit-equals-1-metre scale mean no calibration is needed.
The known artifact (culling decided upstream of the camera write) is documented
so it does not have to be rediscovered.

**Weapon and attachment mods.** [06-weapons.md](06-weapons.md) has the full
per-frame attachment chain and the identity of the last writer of an attached
object's placement. The finding that a weapon is many rigid part objects rather
than one skinned mesh reframes a whole class of problem.

**Render research.** [07-rendering.md](07-rendering.md) has the three placement
paths with exact register slots and strides, which is what you need before
hooking any draw.

**Any of the above on a different title**, using this as the map of what to look
for and, more usefully, [10-negatives.md](10-negatives.md) as the map of what
not to waste a week on.

## A closing caution

The most valuable single habit from this work: **prove your instrument produces
a positive on something you already know is there, in the same run, before you
believe a negative.** Nearly every expensive mistake documented in these files
was a probe looking in the wrong place and reporting an honest, meaningless
nothing.

That applies double when porting to a title you have not studied, because there
you have no intuition for which nothing is real.
