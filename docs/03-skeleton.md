**English** · [Deutsch](de/03-skeleton.md) · [한국어](ko/03-skeleton.md)

---

# 03 - Skeletons, rigs and bone naming

## Bone identity is CRC32, everywhere

`[VERIFIED]` There are **no ASCII bone names** in a `.Skeleton` payload. A
printable-string scan of the player body rig finds none. Bone identity is a
plain **CRC32 of the bone name**, and the same CRC32 is used for class names in
the resource container and for property names in the reflection tables.

This is the single most useful fact about the engine's data model, because it
means you can go from a guessed name to the engine's own identifier with one
hash, and you can crack unknown hashes with a rainbow table.

Verified constants for the player body rig (CRC32 of the HumanIK name):

| Bone | CRC32 | Asset bone index |
|---|---|---|
| `Reference` | `0x2C52CBB0` | 0 |
| `Hips` | `0xDED10611` | 1 |
| `Head` | `0x07C159A2` | 45 |
| `Neck1` | `0xB05FD12B` | |
| `LeftShoulder` | `0x2D4660A8` | |
| `LeftArm` | `0xEB830ADA` | 8 |
| `LeftForeArm` | `0x89B93A80` | |
| `LeftHand` | `0xB675F36C` | 12 |
| `RightHand` | `0x75F94D30` | 58 |

Class hashes work the same way: `crc32("Skeleton") == 0x24AECB7C`,
`crc32("Bone") == 0x95741049`, `crc32("cBallisticProjectileComponent") ==
0x09BFE10E`.

## The `.Skeleton` asset format

`[VERIFIED]` The player body rig parses **byte-complete to EOF with 0 bytes
unexplained**, which is the standard this format was held to.

File framing is `u64 object ID` then `u32 class hash`.

Bone record layout:

```
pre-byte
u64  ID              (0xF80000XX)
u32  class hash      (0x95741049 = "Bone")
u32  Name            CRC32 of the bone name
ptr  Parent          ObjectPtr
ptr  Mirror          ObjectPtr
vec4 GlobalPos
quat GlobalRot
vec4 LocalPos
quat LocalRot
u8   prio
i32  MirroringType
     modifier list
     deps list
i32  WrinkleCategory
f32  WrinkleFactor
u16  Index
u16  ChildrenCount
```

`[VERIFIED]` The Wildlands **ObjectPtr dialect**: tag `02` means a `u64` ID
follows, tag `03` means null with no payload. Getting this wrong desynchronises
the whole parse, so it is worth checking first.

`[VERIFIED]` Structural invariants that hold across the whole rig and make good
parser assertions: the stored `Index` equals the array position, and
`ChildrenCount` equals the subtree size, for all 100 bones.

### The player body rig, concretely

`[VERIFIED]` `GR_PCF_Skeleton_Average.Skeleton`, 14,006 bytes:

- **100 bones**, single root (bone 0, `Reference`), hierarchy depth 12
- Bind pose is Z-up, **1.77 m tall**, root pelvis at z = 0.964
- 86 of 100 bone names resolved. The 14 unresolved are prop and attachment
  helpers, two of which provably end in `LeftWristTarget` / `RightWristTarget`
- `SkeletonKey == SkeletonHierarchyKey == 0x3121DFFF` for this rig, which makes
  a usable scan anchor for finding the player skeleton in process memory

`[VERIFIED]` **The bone names are the Autodesk HumanIK convention**, verbatim:
`Reference`, `Hips`, the Spine chain, `Neck`, `Neck1`, `Head`,
`LeftShoulder`/`Arm`/`ForeArm`/`Hand` with full finger chains, mirrored on the
right, plus helper prefixes `O_` and `L_` (for example the `O_01LeftForeArm`
twist chain, `O_LeftElbow`, `L_01LeftArm`).

That matters beyond naming: it means the runtime rig is a real HumanIK
character, so HumanIK's own semantics apply to it.

### IK data in the asset

`[VERIFIED]` `IkChainDescriptor` in the asset is:

```
u32   ContactBoneID          (a bone NAME hash)
u32   EndEffectorBoneID
u32   IkChainStartBoneID
vec3  ContactOffset
bool  InvertIkResolution
ptr   MirrorIkChain
u8    IkChainSize
```

`[VERIFIED]` **`IKData` carries no payload in the Wildlands era**, and the
player rig's `IkChainsDefinitions` / `IKData` body is null. **The effector table
is runtime-only**, consistent with HumanIK being fed at runtime rather than
authored per asset. Do not go looking for a baked effector list; there isn't one.

### Where the assets live

`[VERIFIED]` `GR_PLAYER_Template` decompresses (1.8 MB to 5.27 MB) into ~1,240
typed objects including **143 `.Skeleton` instances** in the base forge alone:
body, beard, hair, backpack and vest attach-point skeletons. Notable members are
`GR_PCF_Skeleton_Average` (the body rig), `Child_Skeleton`, and per-head
skeletons.

Animation payloads likewise carry no ASCII bone names. A scene descriptor from
one small forge yields ~2,800 typed objects of which 2,177 are `.Animation`,
these *with* readable Ubisoft names (`sb_` for soldier/player class, `civ_` and
`rbl_` for civilian and rebel).

## The runtime rig descriptor

`[VERIFIED, from disassembly of its constructor and init passes]` The rig object
is a **shared, refcounted, `0xF8`-byte skeleton DESCRIPTOR**, cached by dword ID
in a global manager. Characters with the same skeleton datablock **share one rig
instance**: the rig identifies the skeleton *class*, not the individual
character. It holds no transforms at all.

Key fields (retail-build offsets):

| Offset | Contents |
|---|---|
| `+0x08` | refcount |
| `+0x20` | pointer to a `0x60`-byte instance-pool helper |
| `+0x28` | vector of 8-byte channel records `{u32 nameHash, u16 pose-buffer offset, u16 type/index bits}` |
| `+0x34` | total pose-buffer size |
| `+0x3C` | dword rig ID |
| `+0x42` | layer count |
| **`+0x50`** | **sorted `{u32 CRC32 bone-name hash, u16 node index}` NAME MAP** |
| **`+0x5A`** | **name map entry count** |
| `+0x68` | node-to-record remap |
| `+0x80` | node records, 16 bytes each: parent word at `+0`, group byte at `+9`, flags byte at `+0xA` (bit 3 = HIK-marked, bits 4-6 LOD) |
| **`+0x8A`** | **bone count** |
| `+0x98` | hash-sort permutation |
| `+0xA4` | HIK-marked node list |
| `+0xD0` | `CRITICAL_SECTION` |

`[VERIFIED]` The name map at `+0x50` is sorted by hash, so a binary search over
it resolves a bone name hash to a node index without calling engine code. The
engine's own lookup is `Rig::BoneIndexFromNameHash`, retail impl `0x0A85F0F0`
via thunk `0x00CF90F0`, signature `u16(Rig* rcx, u32 crc32 edx)`, returning
`0xFFFF` on a miss.

### A node index is not stable for a character, and must not be cached

`[VERIFIED, 2026-08 update build]` **Resolve the hash to a node index every time
you need it. Do not cache the result, not even against a fixed skeleton
pointer.**

Observed on the player character, one second apart, from a 1 Hz re-resolve that
logs only on change. Same skeleton pointer both times, different node index for
`CRC32("Head")`:

```
[11:09:35.759] player BODY skeleton 0x021A3D8FD110, Head bone node 60 (1 skeleton(s) on this entity)
[11:09:36.784] player BODY skeleton 0x021A3D8FD110, Head bone node 59 (1 skeleton(s) on this entity)
```

The lookup path was skeleton pointer, then the rig pointer read out of that
skeleton, then a name-map search on that rig. Since the skeleton pointer did not
change, either the skeleton was rebound to a different rig, or the map it was
searched against changed underneath it. Both are consistent with the rig
identifying the skeleton **class** rather than the individual character, as
described above.

`[UNKNOWN]` Which of the two it was, and what triggers it. The sample was taken
seconds after the character spawned into the world, so character assembly, a
loadout or gear change, and an LOD transition are all plausible and none is
established. The node records carry LOD bits at `+0xA` and there is a per-bone
LOD override list at `[skel+0x2C8]`, so an LOD-driven remap is the obvious
candidate to test first.

The practical consequence stands regardless of mechanism, and it is the reason
this is worth writing down: code that resolves `Head` once at spawn and caches
the index will silently read **the wrong bone** afterwards. It will not error,
because the stale index is still a valid index into the bone buffer. Anything
built on a cached node index needs a plausibility check on the resulting
transform, or it will drive a limb where it meant to drive a head.

The instance-pool helper is `0x60` bytes with a 64-slot pool (one `0x1808`
allocation of 64 x `0x60`), free-list head at `+0x50`, active-list sentinel at
`+0x58`, slots linked by next at `slot+0x48` and prev at `slot+0x40`. So up to
**64 live character instances per rig class**.

`[VERIFIED]` A global HIK manager singleton holds a hash map keyed by rig ID,
with the rig pointer at `node+0x10`. Enumerating it yields **every live rig**.

`[VERIFIED]` Skeleton-side identity chain: `[skel+0x30]` reaches a guard block
`{encrypted qword +0, key dword +0xC, PLAIN datablock ptr at +0x10}` where the
datablock pointer is readable without decoding anything. `[skel+0x2C8]` is a
sorted `{u32 hash, u32 LOD}` per-bone override list.

## `Skeleton::BipedBoneID`: the attachment-point enum

`[VERIFIED, 2026-08 update build]` Table at RVA `0x046F5080`, **143 entries,
stride 8**, each entry `{u32 CRC32(bone name), u32 CRC32(BIPEDBONE_* tag)}` with
the index as the ordinal. 132 of 143 names recovered.

This is the engine's own attachment-slot enumeration, and it is what the
"attachment slots" concept in later Anvil titles maps onto. Slot numbers from
other titles do **not** transfer.

| idx | hash | name |
|---|---|---|
| 6 | `0x07C159A2` | `Head` |
| 15 | `0xB675F36C` | `LeftHand` |
| 38 | `0x75F94D30` | `RightHand` |
| **72** | `0x3FB256E5` | **`RightHand_Weapon_Ref`** |
| **73** | `0xA9611103` | **`LeftHand_Weapon_Ref`** |
| 115 | `0x826846F3` | `Fake_gunroot` |
| 116 | `0x08B4DDD5` | `FakeGunRoot_Gameplay` |
| 117-120 | | `FakeGunRoot_LeftTrigger` / `_LeftCannon` / `_RightTrigger` / `_RightCannon` |
| 121-124 | `0x53135E44` .. | `Prop_RightHand`, `Prop_RightHand2..4` |
| 125-128 | `0x85562B5C` .. | `Prop_LeftHand`, `Prop_LeftHand2..4` |
| 130-131 | | `FakeGunRoot_SecondHand`, `_SecondHand_Gameplay` |
| 132-136 | `0x7ECBAF84` .. | `Holster_Hips` / `_Back` / `_Chest` / `_LeftUpLeg` / `_RightUpLeg` |
| 137-138 | `0x48674E6D`, `0x776266E1` | `Backpack_Gun_AttachPoint_Primary` / `_Secondary` |

Note the pair at 115/116: there is a **visual** gun root and a separate
**gameplay** gun root, and they are distinct bones. Which one the engine picks
is decided by a chooser function that resolves `FakeGunRoot_Gameplay` first and
falls back to `Fake_gunroot`.

`[VERIFIED]` Also at `0x03AD1230`: a four-dword quad
`{LeftHand, RightHand, LeftHand_Weapon_Ref, RightHand_Weapon_Ref}` immediately
followed by the string `"BipedIkParamsRoot"`. That is the IK params
hand/weapon-reference set.

The weapon rig has its own bone prefix, `wb-`: `wb-gunroot`, `wb-ref-anim`
(`0x8CDA0E3F`), `wb-LightRoot`.

## HumanIK: present as data only

`[VERIFIED]` HumanIK is statically linked with **names stripped**. No HumanIK
DLL exists anywhere in the install tree, and the data tags are in the executable:

| RVA (retail) | Tag |
|---|---|
| `0x03A81BF8` | `HIKCHARACTER000` |
| `0x03A81C63` | `HIKSTATE0000000` |
| `0x03A81CD8` | `HIKEFFECTOR0000` |
| `0x03A81CF0` | `HIKPROPERTY0000` |
| `0x03A81D08` | `HIKDATABLOCK000` |

The **effector name table** is contiguous (retail `0x03C7FE30..0x03C80238`;
update build `0x03CD8370..0x03CD8790`, 44 entries) and is effectively the
effector enum: `HipsEffector`, `LeftWristEffector`, `RightWristEffector`,
`HeadEffector`, `LeftHandEffector`, `RightHandEffector`, `ChestOriginEffector`,
`ChestEndEffector`, plus every finger, plus a matching `*Tip` marker table.

The **property table** is verbatim Autodesk HumanIK 4.x property names:
`ReachActorLeftShoulder`, `ReachActorRightShoulder`, `SnSReachLeftWrist`,
`SnSReachRightWrist`, `SnSReachHead`, `ParamRealisticArmSolving`, and so on.

`[VERIFIED]` These arrays are referenced by **absolute stored pointers**
(relocated), so they are wired into runtime tables rather than being dead data.

`[VERIFIED NEGATIVE]` **No HumanIK public API symbol name exists anywhere in the
image.** `HIKSetEffectorStateTQSfv`, `HIKSolveForEffectorSet`,
`HIKSetNodeStateTQSfv`, `HIKCharacterCreate` and `HIKEffectorSetStateCreate` all
return zero hits across the whole 405 MB string set. The only HIK matches are
those five data tags plus packed-blob noise.

So the solver is reachable only by shape or through the effector name tables.
Whether its effector interface can be driven from outside is **`[UNKNOWN]`**,
and the name tables are the anchor to work from if you try.
