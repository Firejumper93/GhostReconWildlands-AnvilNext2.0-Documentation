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

Coverage actually recovered: **4,491 class descriptors** and **24,642 property
records**, of which 2,564 classes and 12,813 records were **named**.

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
- The projectile component's field names, so
  `m_vBulletSimulationDirection` and `m_vBulletShootOrigin` are the engine's own
  names rather than our labels for two offsets.
- The class-hash-to-name mapping for the resource container: 49 of 50 distinct
  class hashes across 6,563 resources.

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
