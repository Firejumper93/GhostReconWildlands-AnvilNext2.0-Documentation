**English** · [Deutsch](de/06-weapons.md) · [한국어](ko/06-weapons.md)

---

# 06 - Weapons and the attachment system

RVAs in this file are from the **2026-08 update build** unless marked retail.
The 2026 body of `SkeletonPostUpdate` is instruction-for-instruction identical
to the retail build's, so the structural notes transfer and only addresses moved.

## A weapon is many objects, not one

`[VERIFIED, from the shipped shaders]` Weapons are authored as **separate part
meshes**, one `.Mesh` asset each:

```
W_ASR_AK-12_body_LOD0
W_ASR_AK-12_Barrel_LOD0
W_ASR_AK-12_Stock_LOD0
W_ASR_AK-12_StockFolded_LOD0
W_ASR_AK-12_Magazine_LOD0
W_ASR_AK-12_FlashHider_LOD0
W_ASR_AK-12_Ironsights_LOD0
```

`[VERIFIED]` This is corroborated independently by the component layout:
`cWeaponAttachmentHolder` has **one slot per part**.

Practical consequence, and it explains a class of bug that looks mysterious:
because each part is its own drawn object with its own transform, **culling or
hiding the weapon body does not cull its attachments.** A suppressor floating in
mid-air with no gun attached is the expected outcome of hiding the body, not a
glitch.

`[INFERRED, strong]` The 18-to-21-bone weapon rig therefore exists to **position
the swappable parts**, not to deform one mesh. Part count plus attachment points
plus muzzle, eject and sight sockets lands exactly in that range, and the
Gunsmith customisation system requires per-part placement.

## The held-weapon attachment chain, per frame

`[VERIFIED, static]` End to end, for a weapon held in the hand:

```
character SkeletonPostUpdate            0x0F085A50  (thunk 0x0189B430)
  -> Skeleton::PoseRefresh(character)   0x0F091720  (thunk 0x018A09A0)
     -> Pose::SetRootTransform(Pose A)  0x0F203FA0  (thunk 0x01904570)
     -> Pose::SetRootTransform(Pose B)  same fn, source = OWNER ENTITY node 4x4
     -> Skeleton::PublishAttachments    0x0F090D00  (thunk 0x018A03B0)
        -> compose the hand-bone world transform from character Pose B
           (bone record + rootT/rootQ when flags bit 26 is clear)
        -> TransformNode::SetWorldTransform(weapon node)
                                        0x0EAB1C60  (thunk 0x017E1770)
           -> vtable notifier -> Entity::OnTransformChanged  0x0C426AC0
              -> Skeleton::MarkPoseDirty(weapon)  0x0F080AA0 (thunk 0x018996B0)
        -> Skeleton::PoseRefresh(weapon skeleton), inline, 7 instructions later
           -> re-derives BOTH weapon pose roots, clears the dirty bit
```

Read that inline `PoseRefresh` carefully, because it closes a whole line of
enquiry: **a pose-root write cannot survive for a held weapon.** The parent's
publish marks the child dirty and then immediately re-derives its roots.

## `TransformNode::SetWorldTransform`

**impl `0x0EAB1C60`, thunk `0x017E1770`.**

```c
__fastcall(TransformNode* rcx, const float4x4* rdx, bool r8b, bool r9b)
```

Layout of the matrix in `rdx`: rows at `+0x00`, `+0x10`, `+0x20` are the
rotation basis, and **the translation is the fourth row at `+0x30`.**

`[VERIFIED]` This is the source of truth for an attached object's placement: the
weapon's pose root is regenerated *from* it, and the transform-changed
notification propagates *from* it.

Three properties make it a good interception point rather than a compromise:

- `[VERIFIED]` The attachment publish calls it **only when the new matrix
  differs** from the node's current rows (there is a `cmpeqps` comparison block
  immediately before). An injected pose always differs, so an intercept always
  gets the last word.
- `[VERIFIED]` It is the **last writer** of the weapon's placement in the frame.
- `[VERIFIED]` The thunk is a standard 5-byte jmp in an `int3`-padded slot.

`[VERIFIED]` It has **9 rel32 call sites**, so it is a general placement API and
not weapon-private. Gating on `rcx` is mandatory; without a gate you are moving
everything in the world that uses transform nodes.

`[UNKNOWN]`, stated plainly: moving the transform node proves **placement and
ownership**, not the GPU upload path. Whether it is also visually sufficient in
every case depends on how that object's geometry is drawn (see
[07-rendering.md](07-rendering.md)).

## The alternative write target: the node's own local matrix

`[VERIFIED]` `PublishAttachments` computes `world = nodeLocal x boneWorld` and
commits with the fourth argument set, which skips the local recompute. So a
value written into the weapon's own `TransformNode+0x00` local matrix is
consumed every frame.

This touches only the 64 bytes that object owns: no shared pose buffer, no
skinning risk, and it sidesteps the question of which gun-root bone is the real
one. Two knobs should read as expected first (`[node+0x48] == 0` meaning
inherit, and `[node+0x88] == null`). Do not put scale in the local matrix; it is
orthonormalised away.

## The attach API

`[VERIFIED]` **`AttachEntityToBoneByNameHash`**, thunk `0x0085DBB0` to body
`0x094BF1A0`, 17 call sites:

```c
(rcx = child entity, rdx = parent Entity, r8d = CRC32(bone name))
```

It resolves the parent's SkeletonComponent, does a FindBone by hash, **refuses
if the rig lacks that bone**, calls `SetParent`, then registers the attachment
through a shim that supplies a default local offset.

`[VERIFIED]` `BoneHandle` is 16 bytes:
`{owner ptr +0x00, int32 index = -1 at +0x08, u16 slotNum = 0xFFFF at +0x0C}`.

`[VERIFIED]` **The hand-selection call site** is at `0x067EA0EB` inside function
`0x067EA090`: a selector at `[rcx+0x40]` picks 0 to `Prop_RightHand`
(`0x53135E44`) or 1 to `Prop_LeftHand` (`0x85562B5C`), then calls the attach API.

A second, data-driven specification at constructor `0x13ADBFB0` names **both
ends** of the join: the character bone (`Prop_RightHand` / `Prop_LeftHand`) and
the weapon-side bone **`wb-ref-anim`** (`0x8CDA0E3F`). The `wb-` prefix is the
weapon rig's own namespace: `wb-gunroot`, `wb-ref-anim`, `wb-LightRoot`.

### Bone lookup by name hash

`[VERIFIED]` The `FindBone` equivalent is thunk `0x0188D690` to `0x0EF071F0`:

```c
(rcx = skeleton/model object, rdx = out BoneHandle, r8d = CRC32(name))
```

It writes `[rdx+8] = 0xFFFFFFFF` and `[rdx+0xC] = 0xFFFF` (the invalid-bone
sentinel) before resolving. **295 direct call sites**, which makes it one of the
best-connected functions in the engine and a good place to learn what the game
looks up and when.

### The gun-root chooser

`[VERIFIED]` `0x080F8F10..0x080F9022` resolves **`FakeGunRoot_Gameplay`**
(`0x08B4DDD5`) first and falls back to **`Fake_gunroot`** (`0x826846F3`),
returning the winning hash. This is what decides what a weapon component's
`m_GunRootBone` becomes.

The distinction is real and it matters: one of those two bones is the visual
fallback and one is the gameplay mount, and driving the wrong one moves nothing
at all. This is worth measuring rather than assuming.

## Component layout

`[VERIFIED, from the reflection tables]` `GR_cWeaponComponent`,
hash `0x1B15F6FA`, descriptor `0x04AF60E0`, size `0x260`:

| Offset | Property | Type |
|---|---|---|
| `+0xA8` | `m_AssociatedEssence` | `WeaponEssence` |
| **`+0xB0`** | **`m_GunRootBone`** | `BoneHandle` |
| `+0xC0` | `m_MuzzleShootAnchor` | `BoneHandle` |
| `+0xD0` | `m_AimingPointAnchor` | `BoneHandle` |
| **`+0xE0`** | **`m_AttachmentHolder`** | `cWeaponAttachmentHolder` |
| `+0x100` | `m_SpreadSubComponent` | `sSpreadWeaponSubComponent` |
| `+0x158` | | `sBallisticControllerSubComponent` |
| `+0x17C` | `m_eWeaponCategory` | `EWeaponCategory` |

`[VERIFIED]` Those four offsets are **unchanged across the 2026-08 update**,
which is a useful data point on how stable this engine's data layout is compared
with its code addresses.

### The shot has two sources, and one of them is the camera

`[VERIFIED]` `GR_DBWeaponConstants` carries a hybrid shoot system:

| Offset | Property |
|---|---|
| `+0x13C` | `v_bEnableHybridShootSystem` |
| `+0x140` | `v_fMinAngleToCancelFocusPoint` |
| `+0x144` | **`v_fMinAngleToShootFromCamera`** |
| `+0x148` | `d_bDisplayHybridShootSystemDebug` |

This is a global constants object, not a per-weapon setting, and it is a bool
plus two angle thresholds rather than an enum. That combination is why it is
easy to miss: a search shaped around finding a per-weapon fire-source enum, of
the kind a comparable engine ships, will not find it however thorough it is.

`d_bDisplayHybridShootSystemDebug` implies the engine has a debug visualiser for
this system, which would be the fastest way to see it working.

`[UNKNOWN]` What the other source is when the angle test fails, what the angle is
measured between, and where the switch is consumed.

### The weapon skeleton definition

`[VERIFIED]` A single shared database entry named `weapon_skeleton`, of type
`GR_DBSkeletonDefinitionWeapon`, names the bones the weapon accessors resolve.
It holds both the bone IDs and their **authored name strings**:

| Offset | Contents |
|---|---|
| `+0x20` / `+0x28` | `m_iRootBoneID` / `v_cRootBoneName` |
| `+0x40` | `m_iWeaponSilencerID` |
| `+0x60` / `+0x68` | `m_iWeaponMuzzleID` / `v_cWeaponMuzzleName` |
| `+0x70` / `+0x78` | `m_iWeaponAimingPointID` / `v_cWeaponAimingPointName` |
| `+0xC0` / `+0xC8` | `m_iMainHandRootBoneID` / its name string |
| `+0xD0` / `+0xD8` | `m_iSecondHandRootBoneID` / its name string |

Because the names are authored strings rather than hashes, reading this object
answers "which bone is the sight" and "which bone is the grip" directly, with no
CRC cracking and no collision risk.

`[VERIFIED]` Two consequences worth knowing before building on the anchors.
**The unsilenced muzzle anchor and the aiming point resolve to the same bone**,
so a direction built from one to the other is a zero vector. And the silenced
muzzle bone exists on **3 of 3,607** weapon assets, all miniguns, so on a
silenced weapon that lookup cannot succeed.

`[VERIFIED]` The aiming point is **the sight, not the bore**. On one assault
rifle the bore sits at `z = -0.0628`, the iron sight line at `z = -0.0024`, and
a fitted optic's aiming point at `z = +0.0377`, so the sight is about 6 cm over
bore. That offset is a per-weapon authored number, readable offline from the
weapon's own data file.

### The anchors are bone handles, not vectors. This trips people up.

`[VERIFIED]` `m_GunRootBone`, `m_MuzzleShootAnchor` and `m_AimingPointAnchor`
sit at `+0xB0`, `+0xC0` and `+0xD0`, a sixteen-byte stride, and they are
**`BoneHandle`**, not embedded `float4` vectors. The sixteen-byte spacing makes
them look like three vectors and they are not.

Name and type both come from the descriptor's own fixed fields, nothing by
adjacency: `crc32("m_MuzzleShootAnchor") == 0x13EF175F` and
`crc32("BoneHandle") == 0xC11EA419`, both exact, in one record region.

Four independent corroborations, because this is the kind of claim that is
expensive to get wrong:

1. 21 other properties across the game carry the same type and are named things
   like `m_GunRootBone` and `m_hFrontWheelSteering`.
2. The engine has a **separate kind for `float4`** (`kind=0x0D`, 397 records).
   These are `kind=0x16`.
3. A sibling field is named `bHadMuzzleShootAnchorCachedWithSilencer`, which
   only makes sense for a handle that is resolved and cached.
4. Disassembly shows a sixteen-byte object of this type initialised as
   `{qword=-1, int32=-1, u16=0xFFFF}`. A `float4` is cleared with `xorps`, not
   with three integer stores of `-1`.

**Practical consequence: writing sixteen bytes at `weapon+0xC0` repoints the
handle at a different bone. It does not move the muzzle.** If you want to move
where a shot starts or where it is aimed, you move the bone, or you change which
bone the handle names. `+0xE0` `m_AttachmentHolder` is an ordinary eight-byte
pointer, which brackets the run and confirms where the handles stop.

`[VERIFIED]` **There is no general "anchor system" to reverse engineer.** Ten
properties in the entire game have `Anchor` in the name, no class is named
`*Anchor*`, and no enum contains the word. Five of the six spatial ones are
literally `BoneHandle`; the sixth is a raw bone ordinal. **"Anchor" is this
engine's naming convention for a bone reference.** So the geometry of a shot is
bone driven, on the same surface as everything else in `03-skeleton.md`.

`[UNKNOWN]` The internal layout of `BoneHandle`'s three fields, and the routine
that resolves one to a world transform.

`GR_cSniperAimingLaserComponent` is worth a look for anyone chasing this: it is
the only component in the game holding both a `BoneHandle`
(`m_RightEyeAnchor +0x70`) and a full world-space segment (`m_vTrailStart
+0xB0`, `m_vTrailFXDirection +0xC0`, `m_vTrailEnd +0xD0`), so it is a worked
example of "bone anchor in, segment out". Like `TrailFX`, it is a visual.

`cWeaponAttachmentHolder`, hash `0xECFEF6C0`, size `0xD8`:

| Offset | Slot |
|---|---|
| `+0x98` | `m_ScopeSlot` |
| `+0xA0` | `m_BulletSlot` |
| `+0xA8` | `m_MuzzleSlot` |
| `+0xB0` | `m_TriggerSlot` |
| `+0xB8` | `m_BarrelSlot` |
| `+0xC0` | `m_UnderBarrelSlot` |
| `+0xC8` | `m_RailSlot` |
| `+0xD0` | `m_MagazineSlot` |

All are `GR_SingleSlot`, whose `m_Stuff +0x40` points at the occupying
`GR_EquipmentEssence`.

`GR_cInventoryHolder`: `m_PrimaryWeapon +0xF0`, `m_SecondaryWeapon +0xF8`,
`m_ThirdWeapon +0x100`, `m_CurrentHandledCategory +0x128`.

`UniquePropEssence`: `m_CurrentBoneHandle +0x70`, `m_iAttachmentBone +0x80`,
`v_cAttachmentBone +0x88`, `m_Incarnation +0x90`. `[INFERRED]`
`m_iAttachmentBone` is a `BipedBoneID` ordinal and `v_cAttachmentBone` is its
authored string; a throwable-attachment definition class makes that pairing
explicit with `m_iBoneID +0x00` / `v_cBoneName +0x08`.

`[UNKNOWN]` The per-frame function that reads `m_AttachmentHolder` was not
located.

## Projectiles

> **This section was wrong for nine days, and it is worth reading for the
> mistake as much as for the facts.** An earlier revision stated that the bullet
> is `cBallisticProjectileComponent`, size `0x180`, with `m_vBulletShootOrigin`
> at `+0x50` and `m_vBulletSimulationDirection` at `+0x100`. The class name was
> correctly recovered and then attached to the wrong memory layout. See
> `09-methodology.md`, "The closure check", for the one-line test that catches
> this and for how the error was manufactured.

`[VERIFIED]` **Those fields belong to `TrailFX : ManagedObject`, size `0x180`,
42 properties. It is the cosmetic bullet trail.**

| Offset | Property |
|---|---|
| `+0x28` | `m_fMuzzleVelocity` |
| `+0x40` | `m_vShootPositionOrigin` |
| `+0x50` | `m_vBulletShootOrigin` |
| `+0x68` | `m_ShootEmitterAsWeapon` (`GR_cWeaponComponent*`) |
| `+0x98..+0xA4` | `m_fTotalDistance`, `m_fCurrentDistance`, `m_fLastDistanceTravelled`, `m_fInstantSpeed` |
| `+0xA8..+0xAB` | `m_bIsEndReq`, `m_bBulletSimulationCompleted`, `m_bBulletTrailVisible`, `m_bSmokeTrailVisible` |
| `+0xB0` | `m_vTrailFXPosition` |
| `+0xC0` | `m_vLastTrailFXPosition` |
| `+0xD0` | `m_vTrailFXDirection` |
| `+0xF0` | `m_vBulletSimulationPosition` |
| `+0x100` | `m_vBulletSimulationDirection` |
| `+0x118` | `m_eTrailState` |
| `+0x124` | `m_fCurrentBulletScale` |
| `+0x130` | `m_vRefDirection` |
| `+0x170` | `m_bFollowBulletSim` |

`[VERIFIED]` **The real `cBallisticProjectileComponent` is `0xB0` bytes**, class
hash `0x09BFE10E`, descriptor `0x04AECD90`, base `cAIComponentWithSubComponent`,
16 reflected records (7 own, 9 inherited). An offset of `+0x100` cannot exist on
it. Its own properties are `m_ProjectileDBEntry +0x60`, `m_Actor +0x68`,
`m_bRequestDespawn +0x70`, `m_bAddedInDangerMortarZone +0xA8`, and two unnamed.

**It carries no bullet vector at all**: no direction, no origin, no target
point, no segment end, no range and no damage float. Across the whole 21,286
record reflection surface, the only reflected bullet-origin vector in the game
is `TrailFX +0x50`. Honest bound: roughly 110 of the projectile's bytes are
unreflected, so this is a negative about **authored data**, not about runtime
memory.

`[VERIFIED]` The spawn function is real and its disassembly was never in
question. It allocates `0x180` bytes, which is `TrailFX`'s size:

```
movaps xmm1, [owner+0x150]  ->  [trail+0x50]   m_vBulletShootOrigin
movaps xmm0, [owner+0x140]  ->  [trail+0x100]  m_vBulletSimulationDirection
mov    [trail+0x20], owner                     back-pointer
```

`[owner+0x140]` and `[owner+0x150]` are a unit direction and an origin on the
**owner**, and that part stands: the bearing is not computed at this spawn, it
is copied into the trail from something upstream.

### Why this matters more than the correction

`[VERIFIED NEGATIVE]` Three separate interventions were each proven to execute
on a live field, with counters showing them applying, and impacts did not move:
relocating the trail's position fields, rewriting `owner+0x140` at spawn, and
overriding the per-shot aim reader. One of them produced the tell, reported by a
tester as **the tracer following the new line while the damage stayed on the old
one**.

With the class corrected, all three collapse into one explanation: **they were
correct measurements of a cosmetic object.** `m_bFollowBulletSim`, and the
per-tick overwrite of `m_vBulletSimulationDirection` that defeated every attempt
to write it, are exactly how a trail that follows a simulation it does not own
behaves.

The general lesson, and the reason this section is written at length: **"the
write applied and nothing happened" is evidence about the object, not
necessarily about the mechanism.** Establish the object's identity before
concluding the mechanism is wrong.

`[UNKNOWN]` Where damage is actually resolved. Two hypotheses are now retired:
the visible projectile (this section), and a "sliced ray" reading that turned
out to fuse two unrelated classes (`09-methodology.md`). A specific Havok
`castRay` caller bursting immediately before every spawn and never after remains
unexplained; the RVA recorded for it is build-specific and must be re-derived
per build.

## Havok

`[VERIFIED]` `hknpWorld::castRay` was located by cross-referencing Havok's own
monitor-timer string literal `TtWorldCastRay`, which `HK_TIMER_BEGIN` writes
into the profiling stream from **inside** the function it names. Signature
`castRay(hknpWorld* rcx, RayInput* rdx, Collector* r8)`. Only **eight call
sites** exist in the whole image.

There is also a `TtCastRay` body that three call sites reach **directly**,
bypassing the `hknpWorld::castRay` wrapper via a runtime-built virtual table. A
hook on the wrapper alone is blind to those three, which is exactly the sort of
thing that produces a confident false negative.

The pooled placement subsystem that sits behind the bone-gather consumer is
Havok's, not the renderer's.
