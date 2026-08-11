**English** · [Deutsch](de/07-rendering.md) · [한국어](ko/07-rendering.md)

---

# 07 - How geometry is actually placed and drawn

Everything in this file was recovered from **the game's own shipped shaders**,
disassembled with the `d3dcompiler_47.dll` that ships beside the game. That is
worth stating because it is a much stronger source than inference from code: the
shader tells you exactly what the GPU reads.

One practical note on finding them. Material shader DXBC lives in the `.forge`
data, **not** in the shader container DLL. That container holds post-process,
terrain and water shaders only. A scan of it for skinned vertex shaders comes
back with nothing, which reads as "this engine does not do vertex skinning" and
is simply the wrong container.

## The three placement paths

### Path A: GPU-instanced rigid (props, vegetation)

Structured SRV at **`t15`, stride 56**. Bytes `0x00..0x2F` are a `float3x4`
world matrix; `0x30..0x37` are extra per-instance data, `[UNKNOWN]`.

Gated by the geometry option `UseInstancing` (bit 13 of the shader key).

### Path B: non-instanced rigid

**This is the weapon's dominant path.** The vertex position is object space and
goes straight to clip:

```
dp4 o0.x, r0.xyzw, cb4[0].xyzw     ; object-to-CLIP
...
dp3 r1.x, normal_obj, cb4[4].xyz   ; object-to-WORLD 3x3
```

No `t15`, no instance buffer, no bone palette. **All placement lives in the
vertex-stage constant buffer at slot `b4`**, declared `CB4[20]` = 320 bytes:

| Offset | Contents |
|---|---|
| `+0x00..0x3F` | object-to-clip 4x4 (rows are coefficient vectors) |
| `+0x40, +0x50, +0x60` | object-to-world 3x3 (`.xyz`; the `.w` lanes are `[UNKNOWN]`) |
| `+0x130` (`cb4[19]`) | `.y` uv/detail scale, `.z` instance base index (Path A only) |

`[VERIFIED]` The same `b4` slot serves both paths: instanced draws get pure
ViewProjection in `[0..3]`, single-object draws get World*ViewProjection. That
gives you a free runtime cross-check of your matrix convention on any prop draw.

### Path C: vertex-shader skinning

Bone palette is a structured SRV at **`t6`, stride 48, `float3x4`**. Indices are
used raw with no base offset, and because the skinned position goes straight
into `cb4[0..3]` with no world matrix, **the palette contents are WORLD space**.

`t8` is the previous-frame palette, used when `UseMotionVector` is set.

### The `t3` vs `t6` trap

`[VERIFIED, correction]` There is a second stride-48 palette at **`t3`**, and it
is the **compute pre-skinning** path (`PSComputeSkinningPositions4Bones_cs` /
`8Bones_cs`, output UAV `u0` at stride 48).

The **vertex-shader** skinning path used by weapon and character materials binds
its palette at **`t6`**. Same 48-byte `float3x4` layout, different slot.

**A draw-time probe that looks only at `t3` sees nothing on a character or
weapon draw.** That is a clean negative that looks exactly like "the palette is
not here", and it cost two independent investigations before the shader listing
settled it.

## Shader permutation counts, which tell you the architecture

`[VERIFIED]`

| Material | VS permutations | Rigid | Skinned | Instanced |
|---|---|---|---|---|
| `SHD_Weapon_InGame` | 142 | **122** | 20 | **0** |
| `SHD_Basic` (generic props) | 357 | | 36 | **192** |

The weapon material is compiled with `UseInstancing` **off throughout**. Zero of
142 permutations take the instanced path, so for weapons Path A is never taken.

## The shader key, decoded

`[VERIFIED]` From the engine's own shader-permutation debug printer, anchored by
its format strings `ShaderConfig: %s` and `Vertex Format: %s`:

- **ShaderConfig enum**: 0 = DepthOnly, 1 = GBuffer, 2 = Forward.
- **Geometry-option flag names**, 14 entries = bits 8..21 of the shader key
  dword: `Is2Sided`, `Opaque`, `UseLowShaderLOD`, `UseTransparencyWithAlphaTest`,
  `IsLayer`, `IsParticle2D`, `IsParticleDepthFade`, `UseMotionVector`,
  `UseClustering`, `UseCharacterDecals`, `UseEnvInfluence`, `UseDissolve`,
  `UseClutterDissolve`, **`UseInstancing`**.
- **Vertex format names**: `Skinning4Bones`, `Skinning8Bones`, `+Fat` variants,
  `StaticFat`, `StaticQuantizedPosition` (plus `Fat` / `NoColor` / `NoColorFat`),
  `Position3f_*`, `BulkInstance`, `FakeMesh`, `Particle2D`,
  `TerrainMeshFormat`.

`[VERIFIED]` Render traversal tasks: `Graphic::Render::SplitInit`, `SplitNodes`,
`SplitEnd`, `ClearNodeBuckets`, all registered in one contiguous block. Among
the pass names is **`PlayerCharactersPass`**: the player and his gear are drawn
in a **dedicated pass**, separate from other characters.

## The GPU bone palette producer

`[VERIFIED]` `BuildBoneMatrixPalette` at `0x0DBCE0C0`, thunk `0x013A7AB0`, with
a unique image-wide signature containing no rel32 or rip operands:

```
49 89 E3 49 89 5B 10 49 89 73 18 57 48 81 EC 90 00 00 00 41 0F 29 73 E8 48 89 D6 0F B7 51 1A
```

```c
BuildBoneMatrixPalette(Rig* rcx, F3x4** pLocal, F3x4** pOut)
```

Phase 1 normalises each quaternion (`rsqrtps` plus one Newton step) and converts
`{quat, translation}` in place to a `float3x4` **with the translation in the `.w`
lanes**. Phase 2 concatenates down the hierarchy:

```
O[0] = L[0]
O[i].row = O[parent].row.xyz * L[i].rows + (O[parent].row & (0,0,0,1))
```

Index arithmetic is `lea edx,[rax+rax*2]; shl eax,4`, i.e. x48.

### This path never reads the Pose

`[VERIFIED]` and this is important if you are hoping a CPU pose write shows up
on screen. A byte-level scan of **every function in the chain** for the
displacements `+0x178`, `+0x238`, `+0x8C` and for `shl reg,5` returns **zero in
all four**.

Its source is a **separate stride-48 `{quat, translation}` array filled by
animation-clip sampling**, parent-relative, which is the opposite space to the
model-space pose buffer. World placement is folded in afterwards by
post-multiplying every entry by a per-instance 4x4.

All 74 sites image-wide that test the pose-space flag were enumerated, and none
is a bulk 48-byte-stride loop.

**So the CPU skeleton and the GPU palette are fed from different sources.** This
is the concrete form of a rule worth internalising on any engine: writing the
GPU palette moves what you see and changes nothing about what the game thinks;
writing the CPU skeleton before the final pose read changes reach, collision and
muzzle position. They are not the same thing and a change that moves the picture
but not the bullets is not a fix.

### The GPU handoff

`[VERIFIED]` It is a **Map/Unmap into a double-buffered dynamic buffer**, not
`UpdateSubresource` and not a UAV dispatch. The map function has **38 call
sites** and is the engine's universal dynamic-buffer allocator; frame parity
comes from a global toggled with `^ 1`.

`[UNKNOWN]`, stated honestly: whether `0x0DBCE0C0` is the path that draws the
**player and his weapon**. Two facts suggest it is the instanced/crowd path: the
chain is gated on `u16 [rig+0x5C] >= 2` (at least two instances), and its output
is FP16 (12 halves, 24 bytes per bone) at a per-instance stride, **not** the
float32 stride-48 buffer the compute shader reads. Four independent heuristic
sweeps of all 378 MB of executable sections found no second `float3x4` producer,
but heuristic sweeps are not proof.

The decider is cheap: hook the thunk log-only and count calls with only the
player on screen. Zero calls means this is the crowd path and the hero producer
is still open.

## A last-resort visual write

If the authoritative route fails, the guaranteed-visual one is to rewrite
`cb4[0..3]` (object-to-clip) and `cb4[4..6]` (object-to-world 3x3) of the
vertex-stage constant buffer **at the moment of the draw**, inside a D3D11
detour. It is provably the last place the transform exists before the GPU sees
it, it needs no knowledge of the game-side object graph, and nothing can stomp
it because the GPU consumes it on that draw. Set
`newObjectToClip = desiredWorld * ViewProjection`, and ViewProjection is already
available from the camera's matrix block.

For skinned geometry the equivalent is rewriting the `t6` structured buffer
contents for that draw.

**Caveat, plainly: this is visual only.** It moves pixels and nothing else.

### A cheap runtime discriminator this enables

Inside a D3D11 detour, on a bounded sample window: read
`VSGetShaderResources(6,1)` and `VSGetConstantBuffers(4,1)`. A draw with a
non-null `t6` is skinned; a draw with a null `t6` is rigid and its entire
placement is in the 320-byte `b4` buffer.

## Two false friends

`[VERIFIED]` `sg_BulkSkinBuffer_vs` is a two-instruction passthrough. "Skin"
there means subsurface **skin shading**, not skinning.

`[VERIFIED]` `0x0DBDEDD0` reads the pose record layout **and** writes 48-byte
records at stride `0x30`, which makes it look exactly like the palette producer.
It is not: its layout is `{float3 T, float3 A, float3 B, float3 AxB}` with the
translation at floats 0..2, **not** in the `.w` lanes.

## Where hooking draws is done

`[VERIFIED]` Vtable patching is the wrong layer for D3D11 draws on this target,
because **every device context has its own heap vtable**. Patch one and you have
patched one context. Code detours inside `d3d11.dll` itself cover all of them.

## What remains unknown here

- `[UNKNOWN]` The CPU function that fills `cb4` was never found. Map-idiom
  scans, structural scans and string anchors all dead-ended.
- `[VERIFIED NEGATIVE]` The renderer's D3D layer carries **no strings**. The
  D3D11 error strings that do exist in the image belong to NVIDIA TurfEffects,
  not the main renderer, so they cannot anchor it.
- `[VERIFIED NEGATIVE]` A per-object "hidden" flag bit verified for the head
  attachment family shows **no sign of generalising** to weapons: an image-wide
  scan for the corresponding test instruction returns exactly one hit, inside a
  heap allocator. No pointer path from a weapon skeleton instance to a render
  node was found.
