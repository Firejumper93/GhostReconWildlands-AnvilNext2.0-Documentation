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

`[VERIFIED]` The bullet is `cBallisticProjectileComponent`, size `0x180`,
class hash `0x09BFE10E`. The spawn function allocates and fills it, and the
field names come from the engine's own reflection data:

```
movaps xmm1, [owner+0x150]  ->  [proj+0x50]   m_vBulletShootOrigin
movaps xmm0, [owner+0x140]  ->  [proj+0x100]  m_vBulletSimulationDirection
mov    [proj+0x20], owner                     back-pointer
```

So **the shot direction is not computed at spawn**; it is read from
`[owner+0x140]`.

`[VERIFIED NEGATIVE]`, and this is a genuinely useful negative for anyone trying
to redirect fire: the visible projectile is **not what resolves the hit**. Three
separate interventions were each proven to execute on a live field, with
counters showing them applying, and impacts did not move: relocating the
projectile's own position fields, rewriting the spawn direction at
`owner+0x140`, and overriding the per-shot aim reader. The decision is made
somewhere upstream of all three.

The remaining lead, unresolved: a specific Havok `castRay` caller bursts
immediately before every projectile spawn and never after.

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
