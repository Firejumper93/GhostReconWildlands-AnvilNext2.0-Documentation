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
| `+0x10` | packed; **byte offset = packed >> 18** |

That last line is the payoff. Once you can name a property, the same record
hands you its **byte offset inside the object**, so you get a full annotated
struct layout for free.

(Note the anchoring on this build is +4 from an older convention; check yours.)

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
