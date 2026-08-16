**English** · [Deutsch](de/09-methodology.md) · [한국어](ko/09-methodology.md)

---

# 09 - Methodology: what actually worked

Techniques, in roughly the order they are worth trying on this engine. Most
generalise to other closed engines; a few are specific to Anvil.

## Order of attack

1. **Community format specs before disassembly.** For file formats, an existing
   open-source reader gets you most of a container layout in an afternoon, and
   the remaining part is where the real work is. Read it for facts, then verify
   every field against your own files, because dialects differ between titles in
   exactly the fields nobody documents.
2. **Static analysis on the file on disk.** Code is plaintext on disk here, so
   the whole executable can be analysed offline with no process and no debugger.
   Almost everything in this repository was derived that way.
3. **CRC32 name recovery** (see [08-reflection.md](08-reflection.md)). On this
   engine it is not one technique among several, it is the technique.
4. **Runtime measurement last**, and only to answer specific binary questions.

## Name functions by what their arithmetic computes

With no gameplay strings and no method names, the most reliable identification
is mathematical. A function that computes `m[row][0] + m[row][3]` for four rows,
normalises by the length of the xyz part and negates **is** Gribb-Hartmann
frustum plane extraction. Nothing else looks like that.

Likewise, a function whose entire body is `movzx / shl rax,5 / add rax,[rcx+X] /
ret` is an array accessor with stride `0x20`, and it establishes that stride
with more authority than any amount of surrounding context.

Middleware helps, because it names itself. Havok's `HK_TIMER_BEGIN` writes a
literal such as `TtWorldCastRay` into the profiling stream **from inside the
function it names**, so a string cross-reference lands directly in the right
body. HumanIK's data tags and effector name tables do the same job on the
animation side.

## Verify a target three ways before writing anything

The pattern that held up:

1. **A unique byte signature**, which must match **exactly once** image-wide. A
   signature that matches twice is not a signature.
2. **An expected-RVA cross-check** against a per-build address table.
3. **A byte verification at the moment of use**: for a jump thunk, it must be
   `E9 rel32` followed by `int3` padding, and its jump must resolve to the
   function you expect.

Any one failing means do nothing and log loudly. A wrong table can then refuse
to act, but it cannot act on the wrong bytes.

Never fall back to a hardcoded address from a different build. Some frameworks
return a stale fallback on a signature miss and the caller uses it unchecked,
which turns a game update into a crash instead of a clean refusal.

## The closure check: a recovered name is bound to a hash, not to a layout

This one cost roughly five test builds and nine days, and the test that catches
it is one line.

**A class name recovered by CRC32 is bound to a HASH. It is not bound to a
memory layout.** Those are two separate lookups, and the second can silently
attach a correctly recovered name to somebody else's property table.

> **Before attributing any offset to a name, print the whole chain, hash to
> descriptor to table to the exact `nprops` records. Then check: every claimed
> property offset must fit inside that class's own recorded instance size, and
> the number of records claimed must equal `nprops`. A layout that does not fit
> its own class size is a binding error, not a discovery.**

Applied to the case in [06-weapons.md](06-weapons.md): the class was recorded as
size `0xB0` while the claimed layout used `+0xF0`, `+0x100` and `+0x124`. Three
offsets past the end of the object. That is detectable in the time it takes to
read one line of a dump, and nobody read it for nine days, because the field
names were plausible, the offsets were real, and every value read off them
behaved exactly as the names suggested.

Three corollaries, each paid for separately.

**Corroboration must attach to the claim's subject, not to a passenger.** The
spawn function allocated `0x180` bytes, matching the recorded instance size, and
that was written down as independent confirmation of the class **identity**. It
only ever confirmed the **size**, which was not the disputed term. State
explicitly which part of a claim an independent measurement actually touches.

**Adjacency is not membership.** The same paragraph that misattributed the
projectile also fused two unrelated classes into one imagined "ray settings
class", because their property names appeared near each other in a dump. They
had different descriptors, different tables, different base classes and
different sizes, and the three floats filled a `0x20` byte struct completely, so
there was no room for the vectors claimed to sit alongside them at `+0x20` and
`+0x30`. Same defect, same paragraph, undetected for the same nine days.

**When a lookup returns nothing, suspect the method before inventing a property
of the target.** When the fabricated class had no findable property table, the
note written down was "its table has zero references because these descriptors
are built at runtime, so the static route is closed". The specific address named
was not a table at all, and both of the real tables were ordinary static ones the
same tool had already dumped, so as an explanation for THAT lookup it was
invented.

> **And then the correction over-reached, which is the more interesting half.**
> "Descriptors are built at runtime" was dismissed as a fabrication. It is not:
> this engine really does build part of its reflection data at runtime, by code
> writing property records into the reflection section as immediate stores.
> Image-wide there are 35,427 reflection dwords written that way, **27,357 of
> them zero on disk** and therefore invisible to any purely static dumper. The
> original note was wrong about the address and right about the mechanism.
>
> The lesson is symmetrical and worth more than either half: **a claim can be
> unsupported and true at the same time.** When you retract something, retract
> the part the evidence actually reaches. Deciding a mechanism is imaginary
> because the one address offered for it was wrong is the same error as
> believing the mechanism because the address looked plausible.

> **A method that cannot find a known answer in a binary where the answer IS
> known is not evidence about a binary where it is not. A failed control means
> the METHOD is wrong. It never licenses a theory about why the target is
> special.**

## Use `.pdata`, not backward disassembly

To find a function's start from an address inside it, look it up in the
exception unwind table rather than disassembling backwards. Backward
disassembly on x86-64 is guesswork; `.pdata` is a sorted table of exact function
bounds and is right by construction.

## Prove the instrument before believing a negative

The most expensive mistakes in this project were all the same shape: a probe
reported nothing, the nothing was believed, and the real answer was that the
probe was not looking where it thought.

Concrete instances, each of which cost a full test cycle:

- A palette probe watching shader resource slot `t3` when the path in use binds
  `t6`. Clean negative, twice, both meaningless.
- A skinned-shader scan run against the wrong shader container. Zero hits out of
  548, which read as "this engine has no vertex skinning".
- A raycast observer on a wrapper function that three call sites bypass through
  a runtime-built virtual table.
- A hotkey polled on `GetAsyncKeyState`'s pressed-since-last-call bit. **The
  game polls input every frame and consumes that bit**, so a once-a-second poll
  never fires. Two keys registered zero presses across an entire session. Read
  the key-is-down bit (`0x8000`) and do your own edge detection.

Before believing a negative, demonstrate the instrument produces a positive on
something you already know is there, **in the same run**, not a different one.

## Every gate gets its own counter

A counter sitting behind several early returns cannot answer "why did nothing
happen". One probe reported zero results, which was read as "the object never
matches" when an entirely different gate was rejecting everything.

Give every early return its own counter and print them all. It turns "I pressed
the key and nothing happened" from a second test cycle into one log line.

Related: **a capacity-limited probe that does not report its own limit produces
confident garbage.** One census filled 509 of 512 slots, said nothing about it,
and its ranking was read as "the nearest objects" when it meant "the nearest of
whatever was recorded first". Always print saturation.

## Sample faster than the effect you are measuring

A test that ramped a value by 180 degrees per sample produced data that looked
exactly like two systems fighting over one field. It was aliasing. If you are
sampling a value the engine also writes, sample faster than either of you
changes it, or you will invent a conflict that does not exist.

## Prefer absolute values over incremental ones

Wherever possible, compute an absolute value from a source you control rather
than composing a delta onto whatever is currently there.

Incremental composition ratchets whenever the engine does not refresh the field
as often as you assumed, and it forces you to answer questions like "does the
engine rebuild this once per frame or once per call", which are genuinely hard
to measure. An absolute value makes those questions irrelevant. Two independent
conversion projects on other engines reached the same conclusion after their
delta versions ratcheted.

## Work with the engine's own final commit, not against it

For placement, the reliable position is the engine's own last write of a value,
rather than racing it from outside. You are then not competing for ordering.

On this engine that means `TransformNode::SetWorldTransform` for attached-object
placement, and the entry to `Skeleton::PublishAttachments` for a bone that
attachments are composed from. Too early and the animation solver overwrites
you; too late and the consumer has already read. The same idea appears in other
conversion projects under names like "Permanent Change", and at least one
well-known mod abandoned its renderer and animation hooks in favour of it.

## State which side of the CPU/GPU divide a change affects

The GPU bone palette and the CPU skeleton are fed from **different sources** on
this engine (see [07-rendering.md](07-rendering.md)). Changing the palette
affects what you see and nothing the game believes. Changing the CPU skeleton
before the final pose read affects reach, collision and muzzle position.

Always say which one you touched. A change that moves the picture but not the
bullets is not a fix, and calling it one costs everyone the next day.

## One behavioural change per test build

With a play session as the only ground truth, two changes in one build means an
ambiguous result and both have to be retested. It feels slow and it is the
fastest option available.

Corollary: revert failed experiments out of the tree rather than leaving dormant
switches and fallback paths behind. A tree full of disabled probes makes the
next problem three times harder to find.

## Pin the build, fail closed

Two retail builds of this game ship the same engine with **every function at a
different RVA**. Detect the build from the loaded module's own PE header
(TimeDateStamp plus SizeOfImage), select a per-build address table from that,
and do nothing at all for a binary you have never analysed.

Note the asymmetry that makes this cheap: **data layout survived the recompile
where code addresses did not.** Four weapon-component property offsets
re-derived identically across the 2026-08 update while every function moved. So
per-build tables need addresses, not struct offsets.

## Tooling that earned its keep

- A signature scanner that reports **miss** and **ambiguous** as distinct
  outcomes. Zero matches usually means the game updated; more than one means the
  pattern is too short. Conflating them wastes a day.
- A data cross-reference scanner handling **both** rip-relative and absolute
  stored pointers. The HumanIK tables are referenced only by absolute relocated
  pointers, so a rip-relative-only scanner reports them as dead data.
- A `.pdata` lookup tool (function bounds from an interior address).
- An RTTI recovery and demangling pass, even though here it only covers engine
  infrastructure.
- A DXBC disassembler driven through the game's own shipped `d3dcompiler_47.dll`.
  The reflection section (RDEF) is stripped from shipped shaders, so you get
  slots and strides but not names.
- A CRC32 rainbow table built from a community dictionary **plus the image's own
  strings**. The second half is what cracked the hand and weapon bone names.

## What was tried and did not work

RenderDoc was pursued through four separate routes on this title and never
produced a capture. If a plan depends on a frame capture, have a second plan.

Reading the shipped shaders out of the archives turned out to be strictly better
anyway: it gives exact register slots and strides with no runtime involvement at
all, and it is what settled the placement-path question definitively.
