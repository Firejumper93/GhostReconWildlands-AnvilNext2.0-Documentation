# 10 - Verified negatives

Things that are definitively **not** true, and approaches that definitively do
not work. On a closed engine these are worth as much as the positive findings,
and they are the part nobody publishes.

Each entry says what was tried, what the evidence was, and what it means.

## Data model and structures

**`Skeleton+0x120`, `Skeleton+0x250` and `owner+0x050` do not feed the
renderer.** `[VERIFIED]` All three are stale mirrors of the same upstream entity
position. `Skeleton+0x120` is consulted only by a reset/teleport path that does
not run per frame. There is no code path from any of them to the renderer, so
writing them is invisible **by construction**, not by accident. A runtime
observation that one of them is bit-identical to the pose root is shared
provenance, not a causal link.

**Writing Pose A (`skel+0x230`) is pointless.** `[VERIFIED]` It is copied over
Pose B in the same update. Pose B at `skel+0x238` is the final pose.

**A pose-root write cannot survive for an attached object.** `[VERIFIED]` The
parent's attachment publish marks the child's pose dirty through the
transform-changed notifier and calls `PoseRefresh` on it inline seven
instructions later, re-deriving both roots.

**The pose stride is not `0x80` and the buffer is not two 4x4 matrices per
node.** `[VERIFIED, correction]` That reading comes from treating the rig init
code's "unit" as 4 bytes. The unit is 1 byte, the stride is `0x20`, and the
record is `{float4 translation, float4 quaternion}`. The accessor's `shl rax, 5`
is unambiguous.

**The per-character pose buffer pointer is not in an instance-pool slot.**
`[VERIFIED]` It is `Pose+0x178`.

**`IKData` carries no payload in the Wildlands era.** `[VERIFIED]` The player
rig's IK chain definitions body is null. The effector table is runtime-only, so
there is no baked effector list to find in the asset.

## Camera

**Writing `Camera+0x4A0` (the pose matrix slot) does nothing.** `[VERIFIED]`
About 400 writes per second of a 15-degree yaw produced zero faults and zero
visual change, and dumps showed no trace of the injected rotation. It is a
derived output that the camera function rewrites before use. `Camera+0x000` is
the authoritative transform.

**`worldMatrixOverride` is not at `+576`.** `[VERIFIED]` It is at
`Camera+0x2A0` (672). The `+576` figure comes from Origins-era pseudocode, and
the surrounding block is otherwise instruction-for-instruction identical, which
is exactly what makes the wrong offset convincing.

**The gameplay camera does not use the projection function that matches the
well-known Odyssey signature.** `[VERIFIED, by measurement]` The signature match
on `0x0C50C0E0` is byte-perfect and it is the wrong function; the gameplay path
calls `0x0C50C420`. This one is worth re-reading: a perfect signature match on
the right function name can still be the wrong call path.

**The engine does not present six camera objects.** `[VERIFIED]` It presents
**two**, across every state tested (on foot, vehicles, aiming, scopes, menus,
photo mode). Discriminators inherited from later Anvil titles assume a camera
per state and could never have worked here. Use `mode == 0` at `Camera+0x290`,
not pointer identity.

## Rendering

**The shader container DLL does not hold the material shaders.** `[VERIFIED]`
It holds post-process, terrain and water only. Scanning it for skinned vertex
shaders returns 0 of 548, which reads as "this engine has no vertex skinning"
and is simply the wrong container. Material DXBC lives in the `.forge` data.

**The bone palette is not at `t3` for character and weapon draws.**
`[VERIFIED, correction]` `t3` is the **compute pre-skinning** path. The
vertex-shader skinning path binds its palette at **`t6`** (plus `t8` for the
previous frame). Same 48-byte `float3x4` layout, different slot. A probe
watching only `t3` returns a clean, meaningless negative on every character
draw.

**The GPU bone palette producer never reads the Pose.** `[VERIFIED]` A
byte-level scan of every function in the chain for the displacements `+0x178`,
`+0x238`, `+0x8C` and for `shl reg,5` returns zero in all four. Its source is a
separate parent-relative stride-48 array filled by animation-clip sampling. The
CPU skeleton and the GPU palette are fed from different places.

**`sg_BulkSkinBuffer_vs` has nothing to do with skinning.** `[VERIFIED]` It is a
two-instruction passthrough; "skin" there means subsurface skin **shading**.

**`0x0DBDEDD0` is not the palette producer.** `[VERIFIED]` It reads the pose
record layout and writes 48-byte records at stride `0x30`, which looks exactly
right, but its layout is `{float3 T, float3 A, float3 B, float3 AxB}` with the
translation at floats 0..2 rather than in the `.w` lanes.

**The renderer's D3D layer carries no strings.** `[VERIFIED]` The D3D11 error
strings present in the image belong to NVIDIA TurfEffects, not the main
renderer, so they cannot anchor it. The CPU function that fills the placement
constant buffer was never found by any string, map-idiom or structural route.

**Vtable patching is the wrong layer for D3D11 draws.** `[VERIFIED]` Every
device context has its own heap vtable, so patching one patches one context.

**Per-object hidden flags do not generalise from heads to weapons.**
`[VERIFIED]` A flag bit verified for the head attachment family has an
image-wide scan for its test instruction returning exactly one hit, inside a
heap allocator. No pointer path from a weapon skeleton instance to a render node
was found.

## Reflection and naming

**Class descriptors and property tables have zero rip-relative code
references.** `[VERIFIED, five checked with a control passing in the same run]`
They are reached only through pointer slots walked by the registrar. Therefore
"cross-reference the descriptor to find the code that reads field X" **does not
work on this engine.**

**Property records carry no getter pointer on this build.** `[VERIFIED]` Bytes
`+0x14..+0x37` are zero on disk.

**Method names are largely unrecoverable.** `[VERIFIED]` 11,283 of 103,014
entries resolve, nearly all generic lifecycle. No `AttachTo`,
`GetAttachmentBone` or `FindBone` name exists in either dictionary.

**No plaintext gameplay class names exist in the image at all.** `[VERIFIED]`
The CRC32 route was not a shortcut, it was the only route.

**Debug format strings are not always anchorable.** `[VERIFIED]` One set of
render-state dump format strings has zero rip-relative and zero absolute
references because they are reached by pooled base+offset addressing. A control
in the same run did return its one known reference, so the negative is the
string's property, not the scanner's.

## HumanIK

**No HumanIK public API symbol name exists anywhere in the image.**
`[VERIFIED]` `HIKSetEffectorStateTQSfv`, `HIKSolveForEffectorSet`,
`HIKSetNodeStateTQSfv`, `HIKCharacterCreate` and `HIKEffectorSetStateCreate` all
return zero hits across the whole 405 MB string set. The library is statically
linked with names stripped. The only HIK matches are five data tags plus
packed-blob noise. The effector **name tables** are the only anchor.

## Weapons and projectiles

**The visible projectile is not what resolves the hit.** `[VERIFIED]` Three
independent interventions each executed on a live field, with counters proving
they applied, and impacts did not move: relocating the projectile's own position
fields, rewriting the spawn direction the spawn function reads, and overriding
the per-shot aim reader. The decision is upstream of all three.

**The pooled placement subsystem is Havok's**, not a renderer-side bone-gather
consumer. That line of enquiry is closed.

**A hook on the raycast wrapper is blind to three call sites** that reach the
raycast body directly through a runtime-built virtual table. A shot-window
census on the wrapper alone sees only ambient traffic.

## Archives

**Entry names in a `.forge` are not unique.** `[VERIFIED]` 5,602 duplicates in
one archive, 335 in another. FileDataIDs are unique. Address by file_id.

**The TOC is not at the tail.** `[VERIFIED]` Header and both tables sit below
the first payload, on all 21 archives in the install.

**The name field is not at record offset 40.** `[VERIFIED]` It is at 44.
Reading at 40 leaks the Timestamp field's printable bytes into every name, which
is where certain well-known bogus entry names come from.

**A valid decompressible payload is not necessarily a loadable one.**
`[VERIFIED, in the retail game]` A payload repacked as one block set with
compression byte 0 round-trips perfectly through an independent reader and
**black-screens the game**. The original set structure and compression byte have
to be preserved.

**The block size fields are u16, not i32.** `[VERIFIED]` The i32 dialect belongs
to a different Anvil title.

**The chunk checksum is not standard Adler-32.** `[VERIFIED, 789/789 chunks]`
It is Adler-32 with **both accumulators initialised to zero** rather than
`a = 1`.

**The shipped compressor is not LZO1X-1.** `[VERIFIED]` It is **LZO1X-999**,
proven by reproducing a shipped archive byte for byte.

**A patch forge does not supersede a base forge wholesale.** `[VERIFIED]`
Resolution is by **resource** file_id across forges, and a patch supersedes only
the ids it actually contains. A base-only resource is read from the base even
when the patch carries an entry of the same name and same entry file_id.

**One community toolkit has no Wildlands support compiled in at all.**
`[VERIFIED]` A byte scan of its main assembly (ASCII and UTF-16) finds Odyssey,
Origins, Steep and Breakpoint strings and **zero** Wildlands hits.

## Input

**`GetAsyncKeyState`'s pressed-since-last-call bit is unusable here.**
`[VERIFIED]` The game polls input every frame and consumes it, so a
once-a-second poll on that bit **never fires**. Two hotkeys registered zero
presses across an entire session. Read the key-is-down bit (`0x8000`) and do
your own edge detection.

## Frame ordering

**There is no same-frame window between the skeleton update and the render.**
`[VERIFIED]` Skeleton work runs in the component update stages after the graphic
phase, so `SkeletonPostUpdate` output in frame N is consumed by the renderer in
frame N+1.

## Tooling

**RenderDoc does not capture this title.** Four separate routes were tried and
none produced a capture. Reading the shipped shaders out of the archives turned
out to be strictly better anyway, since it yields exact register slots and
strides with no runtime involvement.

**Shipped shaders have RDEF stripped.** You get slots and strides, not names.
