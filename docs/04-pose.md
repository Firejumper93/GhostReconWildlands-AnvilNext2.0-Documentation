**English** · [Deutsch](de/04-pose.md) · [한국어](ko/04-pose.md)

---

# 04 - The Pose object and the real bone layout

This is the part of the engine that decides where a character's bones actually
are, and it is where an earlier confident answer turned out to be wrong. Both
versions are kept below, because the wrong one is a plausible reading of the
init code and someone else will reach it too.

## The chain from a skeleton to an animated bone

`[VERIFIED]`, three independent ways: the layout arithmetic inside three
separate leaf accessors.

```
Skeleton (0xCA0 bytes)
  +0x010 -> owning ENTITY
  +0x120    a full {float4 T, float4 Q} transform (a stale mirror, see below)
  +0x1F0    the skeleton lock
  +0x220 -> Rig descriptor      shared per skeleton CLASS, holds no transforms
  +0x228 -> rig instance
  +0x230 -> Pose A              the animation output
  +0x238 -> Pose B              THE FINAL POSE (aliased at +0x240 and +0x260)
  +0x268, +0x270 -> Poses C, D
  +0xC7C    name-map version stamp
  +0xC90    bit 7 = pose dirty

Pose (0x190 bytes)
  +0x000 float4   root translation
  +0x010 float4   root quaternion   (its w ALSO carries a uniform scale)
  +0x08C dword    flags
  +0x090/+0x0A0   a SECOND transform pair, with a companion dword at +0x160
  +0x170 ptr      the Rig (same object as skel+0x220)
  +0x178 ptr      THE BONE TRANSFORM BUFFER

Bone buffer, stride 0x20 per node index i
  buf + 0x20*i + 0x00   float4 translation
  buf + 0x20*i + 0x10   float4 quaternion
```

## The accessors, which are the layout ground truth

`[VERIFIED]` Each is reachable through a single jump thunk:

| What | Retail impl | Retail thunk | Update impl | Update thunk | Contract |
|---|---|---|---|---|---|
| `Rig::BoneIndexFromNameHash` | `0x0A85F0F0` | `0x00CF90F0` | | | `u16(Rig* rcx, u32 crc32 edx)`, `0xFFFF` on miss |
| `Pose::GetBoneRecord` | `0x0DC3F730` | `0x018BA4F0` | `0x0F1A51C0` | `0x018F00E0` | returns `buf + idx*0x20` |
| `Pose::GetBoneTranslation` | `0x0DC3F9C0` | `0x018BA500` | | | `void(Pose*, u16 idx, float4* out r8)` |
| `Pose::GetBoneRotation` | `0x0DC3FCF0` | `0x018BA550` | | | `void(Pose*, u16 idx, float4* out r8)` |
| `Pose::Bind` (allocates the buffer) | `0x0DC4DAA0` | | `0x0F1B55C0` | `0x018F3460` | `mov [rdi+0x178], rax` |
| `Pose::SetRootTransform` | | | `0x0F203FA0` | `0x01904570` | |
| `PoseCopy` (bones only) | | | `0x0F1CC7A0` | `0x018FA690` | |
| `Skeleton::PoseRefresh` | | | `0x0F091720` | `0x018A09A0` | |
| `Skeleton::PublishAttachments` | | | `0x0F090D00` | `0x018A03B0` | |
| `Skeleton::MarkPoseDirty` | | | `0x0F080AA0` | `0x018996B0` | |
| `SkeletonPostUpdate` | `0x0DA1A990` | `0x01865A10` | `0x0F085A50` | `0x0189B430` | `rcx` = skeleton |

`GetBoneRecord` is four instructions long and settles the stride argument
completely:

```
movzx eax, dx
shl   rax, 5           ; x 0x20
add   rax, [rcx+0x178]
ret
```

`GetBoneRotation` reads `+0x10` of the record and, when the flag below is clear,
composes it with the root quaternion at `[pose+0x10]`. That is what proves the
record is `{translation at +0x00, quaternion at +0x10}` rather than the other way
round.

Unique image-wide byte signatures (retail):

```
GetBoneRecord       0F B7 C2 48 C1 E0 05 48 03 81 78 01 00 00 C3
GetBoneTranslation  40 53 48 83 EC 30 48 8B 81 78 01 00 00
Pose::Bind          48 89 5C 24 08 57 48 83 EC 20 8B 81 8C 00 00 00
SkeletonPostUpdate  F6 81 92 0C 00 00 01 48 89 CF
```

## Space: model, not world

`[VERIFIED]` The pose at `skel+0x238` has flag bit 26 explicitly **cleared** at
creation, so its buffer is in **MODEL space**:

```
world position = rootQ applied to boneT, plus rootT
world rotation = rootQ composed with boneQ
```

The engine does exactly this itself. **Normalise the root quaternion before
using it as a rotation**, because its `w` carries a uniform scale; skip that and
the scale silently multiplies your bone offsets.

### `[pose+0x8C]` flag bits

From `Pose::Bind`:

| Bit | Mask | Meaning |
|---|---|---|
| 26 | `0x04000000` | bone buffer is already in world space. Cleared for Pose B at construction, which is why held weapons read clear |
| 27 | `0x08000000` | bones changed since last copy (set after the solver writes, cleared by `PoseCopy`) |
| 28 | `0x10000000` | use `[rig+0x36]` instead of `[rig+0x34]` for the bone count |
| 24, 25, 29, 31 | | sticky, preserved across `Bind` (mask `0xAF000000`); 25 and 31 are `[UNKNOWN]` |
| 0..23 | | never used, `Bind` masks them off |

Honour bit 26 even though it is always clear in practice. A future path handing
you a world-space buffer would otherwise be double-transformed, and that is a
subtle, hard-to-see error rather than an obvious one.

## The engine's own worked example

`[VERIFIED]` At retail `0x0A85B172` the engine loads `edx = 0x89B93A80`
(CRC32 of `LeftForeArm`), puts the rig in `rcx`, calls the name lookup, and
passes the returned index to `GetBoneTranslation`. It then repeats this for
`LeftHand`, `RightForeArm` and `RightHand`.

That is the canonical recipe, from the engine itself: **hash the name, resolve
the index against the rig, index the pose buffer.**

For repeated reads there is a better template at `0x0DA15AE0`: a cached bone
handle `{u32 nameHash +8, u16 nodeIdx +0xC, u16 stamp +0xE}`, re-resolved only
when the stamp differs from `[skel+0xC7C]`, with a pose refresh when
`[skel+0xC90] & 0x80`.

## The per-frame chain

`[VERIFIED]` `SkeletonPostUpdate(skeleton)` calls, in order: a preamble, the
work function (which ends by copying Pose A into Pose B), a cull, another step,
a function that sets the root transform on `+0x238` and copies again, and two
more.

`[VERIFIED]` **Writing `[[skel+0x238]+0x178] + idx*0x20` is a CPU-skeleton write
and it is the authoritative side.** That buffer is exactly what the three
accessors return, and they are called from 34+ sites across animation, gameplay
and graphics.

`[VERIFIED]` **Writing `[skel+0x230]` (Pose A) is pointless**: it is copied over
`+0x238` again in the same update.

`[VERIFIED]` **The pose root is not state, it is a derived per-frame copy.**
`Pose::SetRootTransform` is the only writer of `[pose+0x00]` / `[pose+0x10]`
image-wide (single-hit signature
`48 83 EC 28 0F 28 02 0F 29 01 0F 28 4A 10 0F 29 49 10 45 84 C0`), and it is fed
from the owner entity's transform node, `[[skel+0x10]+0x18]`, whose translation
is the **fourth row** at `node+0x30`. It is never read back the other way.

`[VERIFIED]` `PoseCopy` copies the **bone buffer only** and does not touch the
root.

## Stale mirrors: three fields that look authoritative and are not

`[VERIFIED]` `Skeleton+0x120`, `Skeleton+0x250` and `owner+0x050` are all stale
mirrors of the same upstream entity position. `Skeleton+0x120` is consulted only
by a reset/teleport path that does not run per frame.

**There is no code path from any of them to the renderer, so writing them is
invisible by construction.** If you observe that one of them is bit-identical to
the pose root, that is shared provenance, not evidence that it feeds the root.

This cost real time to establish, twice. Do not repeat it.

## The write window, and the lock

If you intend to change a bone and have the engine use your value, the window is
narrow and specific:

- **Too early** (before the animation solver) and your write is overwritten.
- **Too late** (after the consumer) and it has no effect.
- For attached objects, the right instant is at the **entry to
  `Skeleton::PublishAttachments`**, which is where the engine composes each
  attached object's world transform from the pose and then places it.

`[VERIFIED]` The whole sequence runs under the skeleton lock (`skel+0x1F0`) on
an engine worker thread. An unsynchronised high-frequency write from another
thread races it and can tear a `movaps` pair, so this is not somewhere to be
casual.

`[VERIFIED]` A pose-root write **cannot survive for an attached object**: the
parent's attachment publish marks the child's pose dirty through the
transform-changed notifier and then calls `PoseRefresh` on it inline seven
instructions later, re-deriving both roots.

## The correction, kept deliberately

An earlier reading of the rig's fifth init pass concluded that the pose buffer
held **two 4x4 matrices per node at stride 0x80**. That reading treated the
init code's "unit" as 4 bytes.

**It is wrong.** The unit is 1 byte, and the stride is `0x20` as
`{float4 T, float4 Q}`. `GetBoneRecord`'s `shl rax, 5` is unambiguous.

Two follow-on corrections from the same mistake:

1. The per-character pose buffer pointer is **not** in an instance-pool slot. It
   is `Pose+0x178`.
2. A runtime observation of "an animated 4x4 matrix at rig+0x430 with a copy at
   +0x470" does not fit a `0x20` stride and must not be carried forward as
   evidence of anything. It was a pose buffer allocated near the rig in the
   heap, not the rig itself.
