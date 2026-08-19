**English** · [Deutsch](de/08-reflection.md) · [한국어](ko/08-reflection.md)

---

# 08 - The reflection tables, and recovering engine names

This is the single highest-leverage technique on this engine, so it gets its own
file. It is what turns "an unknown dword at `+0xB0`" into
`m_GunRootBone : BoneHandle`.

## The problem

`[VERIFIED NEGATIVE]` **No plaintext gameplay class names exist in the image at
all.** Not one. The 1,462 recoverable MSVC RTTI descriptors cover engine
infrastructure and nothing else.

So the usual route (find the class name string, cross-reference it, land in the
constructor) does not exist here. The CRC32 route was not a shortcut, it was the
only route.

## The reflection tables

`[VERIFIED, 2026-08 update build]` Anvil carries a full runtime reflection
system as static data, in a section named `.arch`
(`0x040FF000..0x04E38000`). One registrar function walks all of it:
**RVA `0x017582D0`**.

Coverage actually recovered: **5,439 class descriptors** and **21,286 property
records**.

> **Corrected 2026-08-16.** An earlier revision reported 4,491 classes and
> 24,642 records. Both figures came from a walker with two defects, and the
> combination is worth describing because it fails in a way that looks like
> success:
>
> 1. **Classes with a NULL property table were dropped entirely**, hiding every
>    pure base class. Fewer classes than exist.
> 2. **The table walk ran past the end of each class's table into the next
>    class's records**, because tables are packed back to back and are not all
>    zero terminated. More records than exist, and about **3,200 property
>    attributions landed on the wrong class**.
>
> Defect 2 is the dangerous one. It does not produce garbage, it produces
> plausible properties on the wrong owner, and the resulting struct maps look
> entirely reasonable. It is the direct cause of the misattribution documented
> in `06-weapons.md`. Demonstrable case: `GR_ShootContactInfo`, size `0x50`, was
> credited with fields at `+0x180` and `+0x190` that belong to
> `SoundFXVehicleTrainCarriageLODSet`.
>
> If you write your own walker, bound every table by the owning class's record
> count and do not rely on a terminator. See `09-methodology.md`, "The closure
> check".
>
> **And even a correct walker sees only part of the picture. See the next
> section, which is the more important limitation.**

## Part of the reflection data does not exist on disk

`[VERIFIED, 2026-08-16]` **Anvil builds some of its property records at
RUNTIME**, by code writing them into the reflection section as immediate stores
(`mov [rip+disp], imm32`). Image-wide there are **35,427** reflection dwords
written that way, and **27,357 of them are zero on disk**.

That is on the order of 9,000 to 12,000 property records that no purely static
dumper can see, against roughly 21,000 that it can. **So a static dump of this
engine's reflection data is complete enough to be useful and not complete enough
to prove a negative.**

A worked example of the damage. `GR_cWeaponComponent` reads as 33 own properties
statically. It has **43**. The ten that only appear at runtime include
`v_vFirstPersonCameraOffset +0x190`, `v_fMuzzleVelocity +0x1A0`,
`v_fMaxRange +0x1A4`, `v_fOptimalRange +0x1A8`, `v_eShootMode +0x1BC` and
`v_fSpreadMinRadius +0x1C0`, which is most of the weapon's actual ballistics
tuning. A confident static reading of that class concluded it carried no range,
spread or velocity data at all.

**The practical rule: on this engine, "I dumped the tables and the property is
not there" is not evidence the property does not exist.** Either read the
immediate stores as well, or take the reading at runtime.

### The bug that hid more than the runtime writes did

`[VERIFIED, 2026-08-16]` Chasing the runtime records turned up a defect in the
dumper that was costing more than the runtime mechanism was. The record
validator rejected any record whose `+0x0C` field had a non-zero low half:

```c
if (kind & 0xFFFF) return false;    // wrong
```

**The low half of `+0x0C` is an ELEMENT COUNT for array properties**, proved by
two independent closure checks. That one line was silently deleting **3,879 real
records** image-wide: a single condition, written once and never revisited, was
quietly eating a fifth of the answer. If you write a walker, treat `+0x0C` as a kind in the high
half and a count in the low half, and do not reject on the count.

Record totals after both fixes, on the Last Rites build:

| Stage | Records |
|---|---|
| Original static walk | 21,286 |
| After the validator fix | 25,165 |
| Plus decoded runtime stores | **27,183 attributed, 6,110 orphan, 33,293 readable** |

`GR_cWeaponComponent` goes from 33 own properties to **55**, closing exactly
inside its `0x260` instance size.

Two corrections to constants published earlier here. The reflection span on this
build is **`0x040F8000..0x04E31000`**, of which only up to `0x04B86E00` is
file-backed, so 2.79 MB is never in the file at all. And there is **no single
registrar**: the RVA previously given is padding on this build, and the records
are filled by **13,421 separate straight-line blocks**.

### A sibling title confirms the layout, and adds something useful

`[VERIFIED, public source]` The `ACUFixes` project for Assassin's Creed Unity
independently documents this same record layout on a 2014 Anvil title: stride
`0x38`, `+0x04` name CRC, `+0x08` type CRC, offset = `packed >> 18`. Two titles
nine years apart, same structure.

It also documents **four accessor function pointers per record at
`+0x18/+0x20/+0x28/+0x30`**, and a plaintext `char* typeName` on the class
descriptor. This documentation elsewhere records, as a verified negative, that
records carry no getter pointer because those bytes are zero on disk. Both are
true: **those pointers are among the fields filled at runtime.** If they are
live in memory then every property in the game has a callable getter and setter,
which routes around the standing problem that descriptors have no code
references.

Property record layout, stride `0x38`:

| Offset | Contents |
|---|---|
| `+0x04` | CRC32 of the property name |
| `+0x08` | CRC32 of the type name |
| `+0x0C` | kind |
| `+0x10` | packed; byte offset is decoded from this, see the correction below |

That last line is the payoff. Once you can name a property, the same record
hands you its **byte offset inside the object**, so you get a full annotated
struct layout for free.

(Note the anchoring on this build is +4 from an older convention; check yours.)

> **Corrected 2026-08-19: `packed >> 18` is NOT the offset decode on this
> build.** That rule is documented above from `ACUFixes` on Assassin's Creed
> Unity, and it was repeated here on the assumption that it transferred. It does
> not.
>
> On this build the decode is **`(packed & 0xFFFF) >> 3`** on the dword at
> `record+0x10`.
>
> It was caught by a case where the wrong rule is not merely inaccurate but
> impossible: applied to two adjacent flags in the same table, `packed >> 18`
> yields **the same offset for both**, and two distinct members cannot share one
> offset. The correct rule was then verified against three properties whose
> names are known independently, reproducing member offsets `0x00`, `0x18` and
> `0x1C`. See the worked example immediately below.
>
> Take this as a warning about the cross-title section above it. The record
> **stride** and the **field roles** did transfer across nine years of Anvil.
> The **bit packing did not.** Re-derive any packing rule against your own
> binary using properties you can name independently, and prefer a test case
> where a wrong rule produces an impossible answer rather than a plausible one.

### Worked example: recovering flags that exist as no string at all

`[VERIFIED, 2026-08-19]` `DisplayIKDebug` and `DisableHumanIK` do not exist as
plaintext anywhere in the 411 MB image, in ASCII or UTF-16. They exist only as
CRC32 name hashes inside reflection property descriptors, which is this
chapter's whole argument compressed into one example.

Owning class `EngineOptions`, class hash `0xB4E69FA1`, base `ManagedObject`,
instance size `0x88`, descriptor at `0x144485100`. The full 30-flag table runs
`0x144483610..0x144483CA0`.

| Flag | CRC32 | Record | Member offset |
|---|---|---|---|
| `ShowHud` | `0xE0622D50` | `0x144483610` | `0x00` |
| `DisplayIKDebug` | `0x41A87E75` | `0x144483760` | `0x18` |
| `DisableHumanIK` | `0x79C7CDCD` | `0x144483798` | `0x1C` |

`zlib.crc32` of each name reproduces the hash exactly, and each hash is present
little-endian at the claimed record, so the identification is confirmed rather
than assumed. Four further IK flags fell out of the same table that nobody was
looking for: `DisplayDebugGroundIK`, `ActivateGroundIK`, `DisplaySkeletonStates`
and `DisableQuadrupedIK`.

**The caveat that matters before anyone acts on this.** Storage is an instance
member of a runtime-allocated object, so there is no static address, and
**whether flipping a flag draws anything is `[UNKNOWN]` and cannot be settled
statically.** Retail builds routinely keep reflection metadata while compiling
out the guarded draw path behind it. Gate any attempt on toggling `ShowHud` at
offset `0x00` first: it is cheap, self-verifying, and it proves the base pointer
and the offset convention before anyone trusts an IK write.

## The method: CRC32 rainbow cracking

Everything is hashed with plain CRC32, and CRC32 is not a cryptographic hash. So
build a rainbow table of `hash -> name` and look the answers up.

Where the candidate names come from, in order of how much they contributed:

1. **A community hash dictionary.** AnvilToolkit ships one (about 506,000
   names, Oodle-compressed inside its resources). Good coverage of bone names
   and asset names. **On its own it was not enough.**
2. **The image's own plaintext strings.** Hashing all 1.2 million plaintext
   strings already in the executable, plus token splits of them, added about
   490,000 entries. **This is what cracked the hand and weapon bone names.**
   The engine contains the words even when it does not contain the identifiers.
3. **Targeted guessing with positive controls.** Once you know the naming
   convention (`m_` prefix for members, `v_` for authored strings, `GR_` for
   Ghost Recon specific classes, `c` for classes, `s` for structs), you can
   generate and test candidates cheaply.

Combined table: **1,616,260 entries.**

### Which SKU you dumped changes what is plaintext

`[VERIFIED]` This one is easy to trip over and produces a false negative that
looks authoritative.

Anvil reflection names such as `BIPEDBONE_*`, `DisableHumanIK` and `IKData` are
**present as plaintext in the Ubisoft Store build** of this game and are
**absent as plaintext from the Steam build**, where the same identifiers exist
only as CRC32 hashes.

So a name list harvested from one SKU is a genuine dictionary of real engine
identifiers, and is **not** evidence about what the other SKU contains. If you
inherit a names file, find out which binary produced it before you cite it, and
carry that qualifier wherever you use it. The names are real either way; the
absence of the plaintext is a property of the build, not of the engine.

The practical use is the happy direction: a build that ships the strings is a
free dictionary for cracking the build that does not.

### Always carry positive controls

Every cracking run should reproduce facts you already know, in the same run. The
controls used here were `crc32("Head") == 0x07C159A2`,
`crc32("cBallisticProjectileComponent") == 0x09BFE10E`, and four weapon-component
offsets already established on the previous build re-deriving independently on
the new one.

Without controls, a cracking run that finds nothing is indistinguishable from a
broken script, and this is exactly the case where a broken script looks like a
clean negative.

## What it bought

Concrete examples, all from the tables:

- The entire `GR_cWeaponComponent` layout, with property names and offsets
  (see [06-weapons.md](06-weapons.md)).
- `cWeaponAttachmentHolder` with one named slot per weapon part, which
  independently corroborated the shader finding that a weapon is many objects.
- `Skeleton::BipedBoneID`, the 143-entry attachment-point enum, 132 names
  recovered (see [03-skeleton.md](03-skeleton.md)).
- `TrailFX`'s field names, so `m_vBulletSimulationDirection` and
  `m_vBulletShootOrigin` are the engine's own names rather than our labels for
  two offsets. (Read `06-weapons.md` before using them: they were attributed to
  the projectile for nine days and they are the bullet trail's.)
- The class-hash-to-name mapping for the resource container: 49 of 50 distinct
  class hashes across 6,563 resources.

## When the dictionary runs out: cracking compound names

`[VERIFIED, 2026-08-19]` The rainbow table above cracks what some dictionary
already contains. The hard corpus is the one where nothing does.

The archive's own class list is that corpus. `DataPC.forge` uses **770 distinct
classes**, of which community tooling names about **43**. The other 727 ship as
a hash with no name anywhere, in the archive or in the executable.

They are still recoverable, because Anvil identifies a class by the **zlib CRC32
of its ASCII name**, and CRC32 is not a one-way function in any useful sense. So
naming is a dictionary attack and the only real question is the dictionary.
Two independent proofs that the identity holds:

```
crc32("ShootingShapeGeneratorSpecification") == 0xD0471AC3   (a forge class id)
crc32("DisplayIKDebug")                      == 0x41A87E75   (a reflection flag)
```

### Why a plaintext sweep is not enough, and the reason points at the answer

Of ten class names known independently, **five occur as plaintext in the
executable and five do not**. The misses are not random. They are the
**compounds**, and they are built from tokens that *do* occur on their own:

```
Build + Table        Texture + Set        Engine + Options
```

The engine contains the words even when it does not contain the identifier. So
harvest CamelCase tokens from the image and **recombine** them, rather than
searching for whole names.

### Frequency ranking is a trap

The obvious optimisation is to rank the token vocabulary by frequency and keep
the top few thousand. It fails, and it fails specifically because **the tokens
you need are rare**. On this build, out of a 51,728-token vocabulary:

| Token | Frequency rank |
|---|---|
| `Specification` | 7,458 |
| `Shooting` | 21,638 |

Any sensible cutoff discards exactly what is required. A 3,000-token run scored
**2 of 5** on a known-answer self-test. The full-vocabulary run scored **5 of
5**. If you truncate the vocabulary you are optimising away the answer.

### CRC32 reversal is what makes full vocabulary affordable

Enumerating token pairs is `51,728^2`, about **2.7 billion** combinations, which
is hopeless to hash exhaustively per target.

It is also unnecessary. **zlib's CRC32 byte step is invertible**, because every
entry in its table has a distinct top byte. So given a target hash and a
candidate **tail** token, you can compute the **head** hash that would be
required, and answer it with a single dictionary lookup. Cost per target drops
from `vocab^2` to `vocab`.

Verified directly: unwinding `Table` from `crc32("BuildTable")` reproduces
`crc32("Build")` exactly.

### The honest limit, which decides how the output must be used

At full vocabulary the two-token space is around 2.7 billion candidates against
a **32-bit** hash. So for roughly **half** of random targets, *some* spurious
pair will collide with the right answer. A tool that reports the first match
would manufacture confident nonsense at scale, and it would look exactly like
success.

So the output must be a **shortlist, not an answer**. Collect every candidate
and rank them by corroboration:

| Signal | Weight |
|---|---|
| Token appears as plaintext in the image | +4 |
| Known Anvil suffix (`Descriptor`, `Specification`, `Template`, ...) | +2 |
| Common token | +1 |

Self-test on five independently known names: **5 of 5 recovered**, ranked first
in three cases and third or fourth in the other two, always inside a
six-candidate shortlist.

**Single-token hits are different and are trustworthy alone.** A 51,728-entry
space against a 32-bit hash makes a collision roughly a 1-in-84,000 event, so a
lone one-token match needs no corroboration.

### A sample of the result

These are **proven**, not ranked guesses: `crc32(NAME) == HASH` exactly for
every row. Resource counts are from the archive's own class census and are given
to show which classes actually carry the content.

| Class hash | Recovered name | Resources |
|---|---|---|
| `0xBC11BD33` | `MissionStepActionPack` | 1,443 |
| `0xC391C66C` | `GR_SpawnEntityDescriptor` | 1,439 |
| `0xBC31EF66` | `ConversationExchangeTemplate` | 1,429 |
| `0xB2C08E53` | `DBLootConfig` | 914 |
| `0x8A87C2F5` | `DBIntelDescriptor` | 825 |
| `0x769C0B3F` | `DBCampTag` | 809 |
| `0x0165AE97` | `XCurve` | 790 |
| `0xFAC21375` | `DBDriveAIAggressiveness` | 602 |
| `0xEAE55C42` | `DBGameContext` | 452 |
| `0x8C5BEF63` | `GR_SpawnNpcDescriptor` | 434 |
| `0x58B04F9E` | `DBSkillPassive` | 425 |
| `0xB992756B` | `DBInputActionMappingButton` | 339 |

The `DB` prefix turns out to be the single most productive naming convention in
the archive, which is itself a dictionary hint: once one `DB*` name is confirmed,
the prefix becomes a high-value head token for everything else.

Honest note on coverage: the **largest** unnamed class by population,
`0x36C9CB38` with 2,608 resources, is still unnamed. The long tail is long, and
the remaining names are the ones no dictionary and no recombination has reached.

### A recovered name is not a recovered meaning

`[VERIFIED, 2026-08-19]` This is the failure mode to plan for, and it cost real
time here.

`ShootingShapeGeneratorSpecification` is correctly named. The hash matches at
offset `0x08` of the extracted resource, so the identity is confirmed rather
than assumed. It reads exactly like the weapon spread table, and it was taken to
be so.

**It is not.** Strongest evidence first:

- **Exactly one exists.** All ten forges walked, 155,109 entries, 4,964,649
  resources, two hits and both are the same bytes. Not one per weapon, not one
  per anything.
- **No followable link.** All 253 file-id-shaped `u64` values inside it were
  looked up against every entry and resource id in all ten archives. **One of
  253** resolved, and it was the resource's own id.
- Of its 49 serialised objects, the eight "shape" units on a 649-byte stride are
  byte-identical across the seven comparable ones over 426 consecutive bytes, so
  that block carries no per-shape information at all.

Editing it could not tell a rifle from a pistol. The real per-weapon data was
found by searching resource **names** rather than classes, as 60 `Spread_*` and
102 `Recoil_*` entries under a different class entirely.

**So treat a recovered name as a lead, not a conclusion.** A suggestive
identifier is evidence about what a class was *called*, not about what it
*holds*, and this engine has at least one class whose name is a near-perfect
description of something it does not do.

## Enums, and how to dump every one of them

`[VERIFIED, 2026-08-16]` Enum descriptors have their own format, and once you
know its terminator you can enumerate **every enum in the game** by scanning for
a single marker dword. The body is a run of pairs:

```
{ int32 index, uint32 crc32(valueName) }   repeated
```

followed by a fixed terminator sequence:

```
crc32("Count")      crc32("Invalid")      crc32(TypeName)      crc32("ID_Enum")
```

`crc32("ID_Enum")` is the marker to scan for. Walking backwards from each
occurrence recovers the type name and every value name, subject to the usual
dictionary limits on cracking them.

Yield on the Last Rites build: **1,149 enums recovered with value names**, 488
type names resolved.

This matters because **a mode switch or a behaviour selector is, by definition,
an enum or a bool**. If you want to answer "does this engine expose a setting
for X", enumerating the whole enum surface answers it exhaustively rather than
by failure to find a string. Two worked examples, both negatives, are in
[10-negatives.md](10-negatives.md).

Honest bounds on the run that produced those figures: 31 of 1,180 descriptors
failed to parse, and 90 of 962 small enums have all value names unresolved.

## Four negatives that change how you navigate this engine

These are the ones that will save you the most time, because each is a technique
that works fine on other engines and does not work here.

1. `[VERIFIED NEGATIVE]` **Class descriptors and property tables have zero
   rip-relative code references.** Checked on five of them with a control
   passing in the same run. They are reached only through pointer slots walked
   by the registrar. So **"cross-reference the descriptor to find the code that
   reads field X" does not work.** Field-read code has to be found by offset
   pattern or by bone-hash constants instead.
2. `[VERIFIED NEGATIVE]` **Property records on this build carry no getter
   pointer.** Bytes `+0x14..+0x37` are zero on disk. An older note describing a
   getter thunk in the record does not transfer.
3. `[VERIFIED NEGATIVE]` **Method names are largely unrecoverable.** Only 11,283
   of 103,014 entries resolve, nearly all generic lifecycle methods. No
   `AttachTo`, no `GetAttachmentBone`, no `FindBone` name was recovered; neither
   dictionary contains engine method identifiers. Do not plan around getting
   method names.
4. `[UNKNOWN]` 11,124 hashes remain unresolved. The tail is long and the
   remaining names are the ones no dictionary contains.

## Other things that are hashed the same way

Once you know CRC32 is the universal identifier, several other structures open
up at once:

- **Bone names** in `.Skeleton` assets and in the runtime rig name map.
- **Class names** in the multi-resource container TOC.
- **Bone names in code**: the engine loads a literal CRC32 into `edx` and calls
  the bone lookup. So a search for a known bone hash as an immediate finds every
  site that touches that bone by name. This is how the gun-root chooser and the
  hand-selection call site were found.
- **The HumanIK node lists**, which are static CRC32 lists.

That last technique deserves emphasis. On an engine with no gameplay strings, a
**bone-hash immediate is the closest thing to a symbol** that exists in the code,
and there are only a few hundred of them. Enumerating every site that loads one
gives you a map of everything the engine does with named bones.
