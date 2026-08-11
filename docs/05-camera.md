# 05 - The camera

All RVAs in this file are the 2017-era retail build unless stated.

## The one function that matters

**`UpdateCameraMatricesAndFrustum`**, impl RVA `0x0C5E47E0`, thunk `0x0135F720`.
The Anvil VR community generally calls this `on_calc_mvp`. It takes **21
arguments**, `rcx` = the camera, and runs about **2 calls per frame** for the
player camera (measured: 146,628 calls over 73,311 frames).

`[VERIFIED]` The argument mapping, confirmed at runtime over 55 full argument
snapshots across menu, on-foot, slow rotation and free-flying photo mode, with
**zero mismatches**:

| Arg | Value |
|---|---|
| 1 (`rcx`) | `cam` |
| 2 (`rdx`) | `cam + 0x290` |
| 3 (`r8`) | `cam + 0x314` |
| 4 (`r9`) | `cam + 0x334` |
| 5 to 11 | `cam + 0x324`, `cam`, `+0x360`, `+0x420`, `+0x4A0`, `+0x4E0`, `+0x520` |
| 12, 14, 15 | floats (measured `0.8`/`0.846`, `0.1`, `0.2`) |
| 13 | `cam + 0x79C` |
| 16 | **a pointer** outside the camera struct, target unidentified |
| 17 to 21 | `cam + 0x5A0`, `+0x560`, `+0x460`, `+0x5E0`, `+0x620` |

Note argument 6 is the camera base itself. That was the doubted part of the
static reading and it held up in 55 of 55 samples.

## The camera struct

| Offset | Contents |
|---|---|
| **`+0x000`** | **the camera's world transform, 64 bytes. THE authoritative one** |
| `+0x290` | int, camera mode / type enum |
| `+0x2A0` | `Matrix4x4*` `worldMatrixOverride` |
| `+0x2B0` | float |
| `+0x2BC` | float, base field of view |
| `+0x2C4`, `+0x2C8` | float, projection skew x and y |
| `+0x420 .. +0x620` | nine 4x4 matrices, contiguous at `0x40` spacing |
| `+0x748` | reached by a mode-change synchroniser at function entry |

### `worldMatrixOverride` is at `+0x2A0`

`[VERIFIED]` If you are porting from Origins-era pseudocode, that code reads
`*(Matrix4x4 **)(pCamera + 576)`. Wildlands reproduces the same block
**instruction for instruction** at a different offset:

```
0x0C5E4844  mov rax, [rsi + 0x2a0]     ; rsi = pCamera. The override pointer.
0x0C5E484B  mov rdx, [rbp + 0x4f]      ; = argument 6 = the camera base
0x0C5E484F  test rax, rax
0x0C5E4852  je  0x0C5E4872             ; if null, skip
            movaps x4                  ; copy 64 bytes
0x0C5E4872  lea rcx, [rbp - 0x39]
0x0C5E4876  call ...                   ; -> 0x0CC28A10, the view-matrix builder
```

So the override is at **`Camera + 0x2A0` (672), not `+576`**, and crucially the
block copies the override matrix **into `Camera+0x000`**. The same pattern
appears independently at `0x0C508FA7` on the same struct.

### The nine matrices at `+0x420 .. +0x620`

`[VERIFIED]`, identified from their runtime contents rather than guessed.
Matrices are row-major with translation in the fourth row, so **the engine
composes with row vectors** (`v * M`).

| Offset | Arg | Identity | Evidence |
|---|---|---|---|
| `+0x420` | 8 | **projection** | perspective shape, `[3][2] = 0.1` = near, `[2][3] = -1` |
| `+0x460` | 19 | projection variant | same x/y scales, `[2][2] = -1`, `[3][2] = -0.1`. `[INFERRED]` the far half of a near/far depth partition |
| `+0x4A0` | 9 | camera pose (world) matrix | orthonormal rows, translation row = camera world position |
| `+0x4E0` | 10 | inverse projection | numerically the exact inverse of `+0x420` |
| `+0x520` | 11 | inverse view-projection | equals `invProj * pose` row by row |
| `+0x560` | 18 | inverse view-projection, second variant | rows 0-1 identical to `+0x520`, rows 2-3 sign-flipped |
| `+0x5A0` | 17 | **view-projection** | column 3 = negated camera forward row |
| `+0x5E0` | 20 | copy of `+0x420` | identical every dump |
| `+0x620` | 21 | copy of `+0x4E0` | identical every dump |

`[INFERRED]` The two calls per frame are one per depth-partition half, matching
the `+0x420`/`+0x460` projection pair.

## Where to write, and where not to

This was settled by experiment, and the negative is as useful as the positive.

`[VERIFIED NEGATIVE]` **Writing `Camera+0x4A0` (the pose slot) does nothing.**
Composing a 15-degree yaw onto it at function entry, about 400 writes per
second, produced zero faults and **zero visual change**, and periodic dumps
showed the `+0x4A0`/`+0x5A0` pair clean and mutually consistent with no trace of
the injected yaw. `+0x4A0` is a **derived output** that the function rewrites
before use.

`[VERIFIED]` **Writing `Camera+0x000` works, and it is the injection point.**
The same 15-degree yaw composed onto the root transform at function entry
produced:

- a rendered view at a constant yaw offset from where the game aims, extremely
  visible in play
- **hip fire landing off-crosshair**: the game's aim stays authoritative while
  the rendered view moves, which is exactly the intended separation for a camera
  mod
- no accumulating spin, which proves the engine **rebuilds `Camera+0x000` from
  its own camera state every frame**, so a per-call compose yields a constant
  offset rather than a ratchet
- every derived matrix (view-projection, both inverses, the `+0x4A0` pose)
  consistently following the injected pose, with a measured root-to-derived yaw
  delta of 0.00 degrees

`[VERIFIED]` **The pose slot accepts fully general 6DOF orientation.** During
photo-mode roll the rows held arbitrary orthonormal bases and the renderer
followed. No gimbal constraint, no re-orthogonalisation against gravity.

### The one visible artifact

`[VERIFIED, observed]` **Vegetation culling mismatches at the view edge** on the
side the offset rotates toward. Vegetation visibility is decided by something
that consumes the camera *before* `on_calc_mvp` applies the rotation.

At small per-frame deltas this is invisible. If it ever matters, the fix is to
apply the pose write earlier in the frame, at the `BeginFrame` / `UpdateCamera`
task seam.

### Frame idempotency

If you write this slot, write an **absolute** value derived from a base captured
once per frame, not an incremental composition. There are two calls per frame
and it is not established whether the engine refreshes `Camera+0x000` before
each of them or once per frame; an absolute write makes the question irrelevant.

## Which camera is the player's

`[VERIFIED]` **The player's camera is the one with `mode == 0`** at
`Camera+0x290`.

`[VERIFIED]` **The engine presents only TWO camera objects, not six.** This was
tested across every state that could plausibly have swapped cameras: on foot,
vehicles, aiming, scopes, menus, photo mode. The discriminators inherited from
later Anvil titles could never have worked here, because they assume a camera
per state.

**Use the mode, not the pointer.** Pointer identity is not stable and is not the
discriminator.

Photo mode is worth knowing about as a research instrument: it flies the mode-0
camera freely, with roll, up to ~170 m from the character, and everything
derived follows. It is a free way to prove that a camera write is fully general
before you build anything on it. (On this title the free camera is the game's
own photo mode, bound to F10; it is not NVIDIA Ansel, despite Ansel being
linked in.)

## The projection family

`[VERIFIED]` There is not one projection function, there are **six**, dispatched
by a selector at `0x0C510B20` that computes fov from aspect:

| Thunk | Function RVA | Shape |
|---|---|---|
| `0x01347280` | `0x0C50C0E0` | `sub rsp,0x90`, takes a `pInputVector` |
| `0x01347460` | `0x0C50C2E0` | small, `sub rsp,0x40`, no input vector |
| `0x01347530` | **`0x0C50C420`** | large, `sub rsp,0x100`, extra pointer in `rdx` |
| `0x01347840` | `0x0C50C7E0` | |
| `0x01345720` | `0x0C5094D0` | |
| `0x01345800` | `0x0C509720` | |

`[VERIFIED, by measurement]` **The gameplay camera uses `0x0C50C420`, not
`0x0C50C0E0`.** `0x0C50C0E0` is the one that matches the well-known Odyssey
signature byte for byte, which makes it the obvious candidate and the wrong one.
This is worth stating plainly because the signature match is genuinely perfect:
it is the same function, it is just not the one the gameplay path calls.

Its fifth argument is the **vertical field of view in radians**, passed on the
caller's stack. Measured ranges: 0.78 to 0.87 in normal play, about 1.22
sprinting or in vehicles, 0.49 to 0.52 in plain ADS, about 0.17 through a 3x
magnifier, 0.33+ in menus, and exactly pi/2 (1.5708) for cubemap face captures
(sky and reflection probes). If you ever override this value, band-limit it:
widening the cubemap captures makes the sky slide against the world.

`0x0C50C0E0` is still the useful **anchor** for verifying you are looking at the
binary you think you are, because its signature is unique image-wide:

```
48 89 E0 53 48 81 EC 90 00 00 00 0F 29 70 E8 48 89 CB F3
```

## Supporting maths functions, named by their maths

| RVA | What it is | How it was identified |
|---|---|---|
| `0x0C4DADE0` | `UpdateFrustumPlanesFromVPMatrix` | Gribb-Hartmann plane extraction from a view-projection matrix, normalised and negated. 11 call sites, thunk `0x013329A0` |
| `0x0CC28A10` | view-matrix builder | copies four rows, multiplies one row by a sign constant, calls the matrix inverse. Camera world matrix in, view matrix out |
| `0x05FE48C0` | 4x4 matrix multiply | broadcast-shuffle, `mulps`, accumulate |
| `0x06B1DBB0` | 4x4 matrix inverse | transpose by `shufps`, cofactor expansion |
| `0x0C510B20` | projection-mode selector | computes fov from aspect, dispatches to the six variants |

Naming a function by what its arithmetic provably computes is far more robust
here than naming it by a string or a guess, because there are no gameplay
strings to go on. The frustum builder is a good example: nothing names it, but
nothing else computes `m[row][0] + m[row][3]` for four rows, normalises by the
xyz length, and negates.

## A method note that cost a session

**Do not key a diagnostic table on a continuously varying float.** An early
probe recorded one row per distinct argument value, and a float that changes
every frame consumed the entire table within seconds. Accumulate ranges
(min/max) instead of exact values, and always print whether the table saturated.
A capacity-limited probe that does not report its own limit produces confident
garbage: it looks like "these are the nearest objects" when it means "these are
the first objects that happened to fill the table".
