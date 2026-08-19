**English** · [Deutsch](de/01-binary.md) · [한국어](ko/01-binary.md)

---

# 01 - The binary

## PE facts

`[VERIFIED]` From the 2017-era retail `GRW.exe`:

| Fact | Value |
|---|---|
| Architecture | x86-64 |
| Preferred image base | `0x0000000140000000` |
| SizeOfImage | `0x1633A000` (about 369 MB) |
| Entry point RVA | `0x162AC020`, inside a 97-byte `.tls` section |
| Linker | MSVC 14.22 |
| **ASLR (`DYNAMIC_BASE`)** | **OFF** (`DllCharacteristics = 0x8120`) |
| **CFG (`GUARD_CF`)** | **OFF** (`GuardCFFunctionTable = 0`) |
| DEP (`NX_COMPAT`) | ON |
| `HIGH_ENTROPY_VA` | ON |

ASLR being off means RVAs and the loaded VA differ by a constant you can rely
on. CFG being off means an indirect-call target does not need to be registered,
which matters if you ever redirect one.

## The section table lies

`[VERIFIED]` Section names on this binary do not describe their contents. This
is the single most important practical fact in this file.

| Name | VA | VSize | Flags | What is actually in it |
|---|---|---|---|---|
| `.edata` | `0x00001000` | `0x0384E800` | CODE, EXEC, R | **jump thunks** (see below), entropy 1.097 |
| `.link` | `0x03850000` | `0x0083A000` | INIT_DATA, R | **the real read-only data and strings** |
| `.text1` | `0x0408A000` | `0x00D1A000` | CODE, R, W | **all 1,462 RTTI descriptors** |
| `.pdata` | `0x04DA4000` | `0x0038B000` | INIT_DATA, R | exception unwind data (this one is honest) |
| `.sbss` | `0x0517C000` | `0x111266CC` | CODE, EXEC, R, W | **the real `.text`.** Every export RVA lands here |
| `.code` | `0x162A3000` | `0x00007FE5` | CODE, EXEC, R | stub region; holds the `dxgi.dll` / `d3d11.dll` / `XINPUT1_3.dll` literals |
| `.tls` | `0x162AC000` | `0x00000061` | CODE, EXEC, R, W | contains the entry point |
| `.rdata` | `0x162B2000` | `0x000872E8` | INIT_DATA, R | resources |

**Consequence: never filter a signature scan by section name.** A scanner that
politely restricts itself to `.text` finds nothing at all, and a scanner that
skips `.edata` as "unidentified data" misses the entire thunk table.

On the 2026-08 update build the reflection descriptors live in a section named
`.arch` (`0x040FF000..0x04E38000`).

### The 2026-08 build: the full map, and four ways to waste a session

`[VERIFIED, 2026-08 build, TimeDateStamp 0x6A75F2F4]` That build is 411 MB
across **17 sections**, and its names lie in different ways from the 2017 one,
so do not carry the table above across. Measured makeup:

| Section | Raw | Exec | `0xCC` | `00` | Entropy | What it really is |
|---|---|---|---|---|---|---|
| `.link` | 324,949,504 | X | 0.3% | 4.0% | 7.10 | **THE ENGINE.** Dense real code |
| `.reloc` | 59,354,112 | X | **91.3%** | 0.5% | 1.02 | **TRAP.** Thunk stubs in `int3` padding. Executable, looks like code, is not |
| `.rsrc` | 11,070,976 | - | 0.1% | 64.5% | 2.79 | **INVERSE TRAP.** Two thirds zeros, and holds the reflection metadata |
| `.xtext` | 8,762,880 | - | 0.1% | 28.1% | 5.52 | non-executable despite the name |
| `.pdata` | 3,794,432 | - | 0.3% | 1.1% | **7.44** | unwind tables. High entropy is normal here, not encryption |
| `.tls` | 2,542,592 | - | 0.1% | 10.6% | 6.17 | normal |
| `.idata` | 553,984 | - | 0.3% | 10.7% | 7.36 | imports |
| `.edata` | 131,584 | - | 0.0% | **100.0%** | 0.00 | **entirely zero in the file** |
| `.xtls`, `.rdata`, `.trace` | tiny | X | | | | executable, a few KB each |
| `.xcode`, `.sdata`, `.srdata`, `.text`, `.data`, `.data1` | tiny | | | | | KB-scale |

**1. Measuring `.reloc` and concluding about the engine.** This one actually
happened, and it produced a confident, completely inverted strategic
conclusion: that the binary is "90% `0xCC` filler with hundreds of thousands of
`E9` thunks into an encrypted blob", therefore static analysis is dead for most
of the engine and there is no point continuing.

Every number in that sentence was correctly measured. They were measured on
`.reloc`, which **is** 91.3% `0xCC` and **is** the thunk stub table. `.link`, the
actual engine, is **0.3%** padding and is not encrypted. The lesson generalises
past this binary: **check which region a statistic describes before acting on
it**, because a true measurement of the wrong region reads exactly like a true
measurement of the right one.

**2. Dismissing `.rsrc` because it is mostly zeros.** It is 64.5% zero *and* it
is where Anvil keeps its reflection descriptors. The `EngineOptions` flag table
and the 44-entry HumanIK effector table both live there. A tool that searches
only `.rdata` and `.data` for tables misses them completely. See
[08-reflection.md](08-reflection.md).

**3. Searching the file for runtime-only data.** Several sections have virtual
sizes far above their raw size, and `.edata` is 100% zero on disk. Worked
example: the input-context manager global at `0x144D884E8` falls outside every
raw range, so a file scan for its contents finds zeros and proves nothing. It is
populated only at runtime. See [12-input.md](12-input.md).

**4. Chasing `.pdata`'s entropy.** 7.44 looks like encryption and is packed
unwind records. Not executable, not interesting.

## Denuvo, EasyAntiCheat, and what is actually analysable

`[VERIFIED, 2026-08 build]` Two protections are commonly assumed on this title.
They are in different states, and conflating them costs time in both directions.

**EasyAntiCheat is gone.** On the 2026-08 build the `EasyAntiCheat\` directory
is empty, no EAC file exists anywhere in the install, and a 1 Hz process watch
that had fired within 20 to 32 seconds on fourteen prior sessions never fired
across a 78 second session that reached in-world play. Recorded as a dated
observation of that build, not as a claim about the title in general: an
earlier build shipped it and a later one may again.

**Denuvo has not gone.** The same build still carries the full protector section
layout: 411,273,208 bytes, 17 sections, `.link` at 325 MB raw plus `.xtext`,
`.xcode`, `.xtls` and `.srdata`. Anti-debug behaviour must still be assumed, and
a string census of the retail image confirms `m_enableDebuggerDetection` exists.
Treat attaching a debugger as something the process may notice.

**But the mutation is local, not pervasive**, and this is the part that matters
for anyone who writes the binary off. Engine code inside `.link` disassembles
linearly with no gaps: roughly 750 bytes of registration code decoded cleanly in
one sitting. The protector shape does appear, for instance three decode gaps
immediately after a `ret` at `0x14D161AAD`, but it is localised around specific
paths rather than spread across the engine.

**So static analysis of engine code is viable on this build. It is the RUNTIME
that is hostile to instrumentation**, which is the opposite of the intuition
most people bring to a Denuvo binary. Prefer static derivation, and treat
runtime attachment as the expensive option rather than the default.

## Code is plaintext on disk

`[VERIFIED]` Disassembling an export directly out of the file on disk produces
valid instructions:

```
export ??1GraphicLibFacade@scimitar@@UEAA@XZ, RVA 0x0A8A10A0, section .sbss

0x000000014A8A10A0  48 8d 05 89 91 0a f9    lea rax, [rip - 0x6f56e77]
0x000000014A8A10A7  48 89 01                mov qword ptr [rcx], rax
0x000000014A8A10AA  c3                      ret
```

22 instructions decoded, 0 invalid. **Static AOB signature derivation against
the file on disk is therefore valid**, and you never need a running process to
find a function. The namespace in that export name, `scimitar`, is the engine's
internal name and shows up throughout.

## Jump thunks: the hooking surface

`[VERIFIED]` The engine does not call its own functions directly. Calls go
through a 5-byte relative jump living alone in a 16-byte `int3`-padded slot:

```
0x01347280:  E9 5B 4E 1C 0B  CC CC CC CC CC CC CC CC CC CC CC
0x0135F720:  E9 BB 50 28 0B  CC CC CC CC CC CC CC CC CC CC CC
0x01349DF0:  E9 2B 6D 1C 0B  CC CC CC CC CC CC CC CC CC CC CC
0x013329A0:  E9 3B 84 1A 0B  CC CC CC CC CC CC CC CC CC CC CC
```

so the shape is `caller -> E9 rel32 (thunk, .edata) -> real function (.sbss)`.

This is the best hooking surface on the target, and it is better than a
prologue detour for reasons specific to this binary:

- A 14-byte `FF 25 00 00 00 00` (`jmp qword ptr [rip+0]`) plus an 8-byte
  absolute address **fits entirely inside the thunk's own 16-byte slot**. You
  redirect a call by writing 14 bytes into a table of stubs and touching no real
  code at all.
- Your replacement is entered by a `jmp` from the caller's `call`, so it sees
  the identical registers, the identical stack layout and the identical return
  address. It is an ordinary function with the same signature. **No trampoline,
  no stolen instructions, no length disassembler.**
- `[VERIFIED]` It sidesteps a real trap: both camera maths functions begin with
  **rsp-relative** instructions (`mov rax, rsp`, `mov [rsp+8], rbx`). A
  copied-prologue detour would execute those at the wrong stack depth and
  corrupt the frame. With a thunk rewrite the question never arises.
- Restoring the original 16 bytes makes the change exactly reversible in-process.

Some slots are not `E9` thunks but virtual-dispatch stubs of the shape
`mov rax,[rcx] / jmp qword ptr [rax+disp]`, 10 bytes plus `int3` padding. Those
are hookable the same way, but there is no "original function" to verify against
or return to, so verify the exact expected byte sequence instead and re-implement
the dispatch yourself.

### Observing a function whose prototype you do not know

Many interesting functions here have prototypes you cannot confidently
reconstruct (one camera function takes 21 arguments). The robust approach is an
assembly stub that saves every register that can carry an argument under the
Microsoft x64 ABI, calls a recorder with a pointer to the saved block, restores
everything, and **tail-jumps** to the real function. Because it jumps rather
than calls, stack-passed arguments and the return address are untouched and the
real function cannot tell the difference.

One alignment detail, because getting it wrong produces a bug that stays latent
for weeks: entry via `jmp` from a thunk leaves `rsp` congruent to 8 mod 16. The
ABI wants `rsp` congruent to 0 mod 16 immediately before a `call`. So the stub's
own frame reservation must itself be 8 mod 16 (for example `0xB8`, not `0xB0`).
Reserve the wrong amount and everything works until the recorder spills xmm
registers with `movaps`, at which point the first aligned store faults.

## Runtime-resolved graphics and input APIs

`[VERIFIED]` `d3d11.dll`, `dxgi.dll`, `XINPUT1_3.dll` and `DINPUT8.dll` are
**not** statically imported. They are resolved at runtime by name, and the
literals sit in `.code`:

| String | RVA |
|---|---|
| `CreateDXGIFactory1` | `0x162A8458` |
| `dxgi.dll` | `0x162A846B` |
| `D3D11CreateDevice` | `0x162A84B9` |
| `d3d11.dll` | `0x162A84CB` |
| `XINPUT1_3.dll` | `0x162A897E` |
| `DINPUT8.dll` | `0x162AADCF` |

`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs` has no entry
for `dxgi`, `d3d11` or `xinput`, so the application directory is searched first.

`[VERIFIED]` XInput specifically is resolved by hand (`GetModuleHandle` +
`GetProcAddress`) into a fixed data global and called through that global,
**bypassing the real import table entirely**. If you are looking for the import
slot to patch, there isn't one; the data global is the only pointer, and its RVA
moves on every recompile.

`opengl32.dll` is statically imported, but only for `glGetString`,
`wglCreateContext`, `wglDeleteContext` and `wglMakeCurrent`, i.e. GPU
identification, not rendering.

## Middleware present in the image

| Middleware | Evidence |
|---|---|
| **Autodesk HumanIK** | Statically linked, names stripped. No DLL anywhere in the install tree. See [03-skeleton.md](03-skeleton.md). |
| **Havok** | Physics and the pooled placement subsystem. Named from its own profiling-timer string literals such as `TtWorldCastRay`. |
| **NVIDIA Ansel** | Exactly three symbols imported from `anselsdk64.dll`: `setConfiguration`, `updateCamera`, `isAnselAvailable`. |
| **Tobii EyeX** | 37 symbols from `tobii.eyex.client.dll`, including `tobii_head_pose_subscribe` and `tobii_gaze_point_subscribe`. |
| **NVIDIA TurfEffects, GFSDK SSAO, Volumetric Lighting** | Separate DLLs in the install root. |

Havok naming itself in profiling strings is worth remembering as a technique:
`HK_TIMER_BEGIN` writes a literal into the profiling stream *from inside the
function it names*, so a string cross-reference lands you directly in the right
function body.

## Coordinate conventions

`[VERIFIED]` These are the engine's **own stated basis vectors**, not an
inference. The Ansel SDK requires a title to declare its coordinate convention,
so hooking `ansel::setConfiguration` and dumping the struct the game passes in
gets you the answer from the engine's mouth:

```
right   = (+1, 0, 0)   = +X
up      = ( 0, 0,+1)   = +Z
forward = ( 0,+1, 0)   = +Y
```

| Property | Value |
|---|---|
| Up axis | **+Z** |
| Forward axis | **+Y** |
| Right axis | **+X** |
| Handedness | **RIGHT-HANDED** |
| World scale | `metersInWorldUnit = 1.0`, so **1 world unit = 1 metre** |

Handedness derivation: `right x up = X x Z = -Y`, and `(right x up) . forward
= -1.0`. For a basis `(right, up, forward)`, `right x up = +forward` means
left-handed and `-forward` means right-handed.

The declared struct, partially identified:

```
+0x000  right   (1,0,0)
+0x00C  up      (0,0,1)
+0x018  forward (0,1,0)
+0x024  1.0        metersInWorldUnit
+0x028  45.0       [INFERRED] a speed or angle limit
+0x02C  1   (int)
+0x030  8   (int)
+0x034  1.0
+0x038  01 01 01 01  [INFERRED] four bools
```

`[INFERRED]` Those four consecutive `01` bytes are very likely
`isCameraOffcenteredProjectionSupported`, `isCameraRotationSupported`,
`isCameraTranslationSupported` and `isCameraFovSupported`, all true. If that
reading is right, the game has declared to Ansel that it supports off-centre
projection, rotation, translation and FOV override.

**+Z up with +Y forward matches later Anvil titles**, so basis-conversion code
written for Odyssey or Valhalla transfers unchanged. The handedness is worth
double-checking against whatever reference you are porting from: at least one
well-known Anvil VR codebase asserts left-handed GLM while its actual projection
maths uses the right-handed form. The maths is right; the assertion is about the
GLM build configuration, not about the engine.

## The engine task graph

`[VERIFIED]` A contiguous block of 867 named engine tasks exists in the string
data. This is a free map of the frame.

Frame tasks:

| RVA | Task |
|---|---|
| `0x0394A4B0` | `BeginFrame` |
| `0x0394A4C0` | `Engine::BeginFrame` |
| `0x0394A4D8` | `BeginGraphicFrame` |
| `0x0394A548` | `GraphicFrame` |
| `0x0394A570` | `EndGraphicFrame` |
| `0x0394A5A0` | `BeginEngineFrame` |
| `0x0394AC88` | `EndEngineFrame` |
| `0x0394ACD8` | `EndFrame` |
| `0x03A29338` | `ViewPreRender` |

Camera tasks:

| RVA | Task |
|---|---|
| `0x0394A9D8` | `UpdateCamera` |
| `0x0394A9E8` | `Ai::UpdateCamera` |
| `0x0394AD80` | `UpdateActionAfterCameraTask` |
| `0x03964DF0` | `WorldView` |
| `0x03964E30` | `CameraTransform` |

Animation and skeleton tasks, in execution order:

| RVA | Task |
|---|---|
| `0x03980110` | `SkeletonGatherComponents` |
| `0x03980168` | `SkeletonUpdate` |
| `0x03980178` | `SkeletonUpdateInteractions` |
| `0x0397FF10` | `AnimUpdateBones` |
| `0x0397FD18` | `AnimUpdateBonesAfterCamera` |
| `0x0397FC78` | `SkelUpdateBeginAnimateAfterCamera` |
| `0x0397FCA0` | `Anim::UpdateEndAnimateAfterCamera` |
| `0x03980408` | `SkeletonPostUpdate` |
| `0x039803C8` | `Anim::SkeletonPostUpdate` |
| `0x039801B0` | `AtomAfterSkeleton` |

The `AfterCamera` variants are the interesting ones: the engine explicitly
distinguishes animation that runs before the camera update from animation that
runs after it.

`[VERIFIED]` **Frame ordering that matters:** skeleton work runs in the
component update stages inside `Engine::EngineLoop::Step`, **after** the graphic
phase. So `SkeletonPostUpdate` output in frame N is consumed by the renderer in
frame N+1. There is no same-frame post-palette write window.

## RTTI

`[VERIFIED]` 1,462 MSVC RTTI type descriptors are recoverable and demangle
cleanly, living in the misleadingly-named `.text1`. They cover engine
infrastructure (the `scimitar` namespace) rather than gameplay classes.

Gameplay class names are **not** in the image as plaintext at all. See
[08-reflection.md](08-reflection.md) for the CRC32 route, which is the only way
to get them.
