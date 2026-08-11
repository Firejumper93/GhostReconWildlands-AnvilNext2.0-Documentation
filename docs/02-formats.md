**English** · [Deutsch](de/02-formats.md) · [한국어](ko/02-formats.md)

---

# 02 - Archive and data-file formats

Everything here is `[VERIFIED]` against the shipped Wildlands archives, at the
scale stated in each claim. The container layout was cross-checked against
Blacksmith (MIT); format constants were corroborated against a decompilation of
AnvilToolkit, read for facts only.

Both a reader and a **byte-identical writer** were built from these notes, so
the format is verified in the strongest sense available: a no-op rebuild of all
21 forges in a 62 GB install reproduces every file byte for byte, and a modified
forge loads in the shipped game.

## The `.forge` container

```
header:   "scimitar" (8 bytes), Unknown1 u8, FileVersionIdentifier i32 (= 27),
          OffsetToDataHeader u64

@OffsetToDataHeader (DataHeader1):
          NumOfEntries i32, Unknown1 i32[4], Unknown2 i64,
          MaxFilesForThisIndex i32 (= 5000), Unknown3 i32, OffsetToData i64

@OffsetToData (DataHeader2, one per index section):
          IndexCount i32, Unknown1 i32,
          OffsetToIndexTable i64, OffsetToNextDataSection i64 (-1 = last),
          IndexStart i32, IndexEnd i32,
          OffsetToNameTable i64, Unknown2 i64

@OffsetToIndexTable:  IndexCount x {
          OffsetToRawData i64, FileDataID i64, RawDataSize i32 }      (20 bytes)

@OffsetToNameTable:   IndexCount x {
          RawDataSize i32, Unknown1 i64, Unknown2 i32,
          ResourceIdentifier u32, Unknown3 i32[2], NextFileCount i32,
          PreviousFileCount i32, Unknown4 i32, Timestamp i32,
          Name char[128], Unknown5 i32[5] }                           (192 bytes)
```

Hard-won details:

- `[VERIFIED]` **The name field starts at record offset 44, not 40.** The
  44-byte prelude plus `char[128]` plus `i32[5]` is exactly the 192-byte stride.
  Reading the name at offset 40 leaks the printable bytes of the `Timestamp`
  field into every name. That is where the bogus entry names `XGlobalMetaFile`
  (base forges, timestamp `0x5888C31C`) and `JijGlobalMetaFile` (patch_01
  forges, timestamp `0x6A692399`) come from, if you have seen them in other
  tools. **Timestamp is at 40.**
- `[VERIFIED]` The `i64` at name-record offset 4 is **not** the FileDataID; it
  does not match the index table's value on live records. The index table is the
  authority on file ids.
- `[VERIFIED, 270,004 / 270,004 entries across a 62 GB install]` The name
  record's `RawDataSize` always equals the index record's `RawDataSize`, so a
  repack must patch both or the archive is inconsistent.
- `[VERIFIED]` **Entry names are not unique.** 5,602 duplicates in
  `DataPC_GRN_WorldMap.forge`, 335 in `DataPC.forge`, 0 in the small ones.
  FileDataIDs *are* unique within a forge. **Address an entry by file_id, never
  by name.**
- `[VERIFIED]` The header and both tables sit **below** the first payload. The
  file runs: header, index table, name table, a small `_Lost&Found` record,
  reserved space, then payloads. It is not a tail TOC.
- `[VERIFIED]` Payloads are stored **contiguously in index order**, with no gaps
  and no overlaps, on all 21 forges.
- `[VERIFIED]` Total file size is the end of the last payload rounded up to
  **`0x8000`**, padded with zeros. Exact on 21/21 forges. `0x4000` fits only
  10/21, so the alignment really is `0x8000`.
- `[VERIFIED]` Multi-section forges chain through `OffsetToNextDataSection`.
  `DataPC.forge` happens to be single-section with 29,640 entries, but the chain
  must be followed in general; at least one well-known reader only reads the
  first section.
- If you want a byte-identical repack, **do not sort entries on read.** File
  order has to survive.

## The payload: Anvil data-file block sets

Each entry's payload is one or more block sets:

```
per set: magic u64 = 0x1004FA9957FBAA33
         version i16 (= 1)
         compression u8
         maxBlockSize u16 (= 0x8000), maxBlockSize2 u16
         blockCount i32
         blockCount x { uncompressedSize u16, compressedSize u16 }
         blockCount x { checksum u32, data[compressedSize] }
```

- `[VERIFIED]` Block size is `0x8000` (32,768). The size fields are **u16**, not
  the i32 that some readers use for other Anvil titles; that dialect differs.
- `[VERIFIED]` A block whose `compressedSize == uncompressedSize` is stored raw.
- `[VERIFIED]` Compression bytes 0 and 1 are **both LZO1X**. (2 = LZO1A,
  5 = LZO1C, neither seen in Wildlands so far.)

### The compressor is LZO1X-999, not LZO1X-1

`[VERIFIED]` and this is the strongest single result in this file. Decompressing
and re-encoding **every entry** of `DataPC_GRN_TitleScreen_patch_01.forge` with
`lzo1x_999_compress` reproduces the shipped file **byte for byte**
(SHA-256 `1A7C3B8C...`). `lzo1x_1` produces valid streams about 15% larger.

So the toolchain's compressor is not merely compatible with the game's, it *is*
the game's. If you need a byte-identical repack of unchanged content, use
LZO1X-999.

### The per-chunk checksum

`[VERIFIED, 789 / 789 chunks across three forges]` It is **Adler-32 with both
accumulators initialised to ZERO**. Standard Adler-32 starts `a = 1`, which is
why an off-the-shelf implementation fails here:

```python
a = 0; b = 0
for byte in chunk_bytes:      # the STORED (possibly compressed) bytes
    a = (a + byte) % 65521
    b = (b + a)    % 65521
checksum = (b << 16) | a
```

This was the long-standing blocker on writing a valid archive, and it is
genuinely just that one initialisation difference.

### Set structure is load-bearing

`[VERIFIED]` Every stock payload uses compression byte 1 and carries a **small
leading block set** before the body set: 16 bytes for one material, 128 for a
cell datablock, 41,806 bytes in two blocks for a scene descriptor. That leading
set is the resource's header record and the engine reads it separately.

A payload repacked as a **single** set with compression byte 0 decompresses
correctly in an independent reader **but black-screens the game.** Verified on
the retail game. So a repack must preserve the original set structure and
compression byte, not merely produce a valid stream.

## The multi-resource container inside a payload

`[VERIFIED]` A forge entry's decompressed payload is usually not one resource.
It is a directory of them:

```
u16 count
count x { u64 file_id, u32 record_size, u16 flags }        (14 bytes, the TOC)
count x {
    u32 class_hash        # CRC32 of the class name
    u32 body_size
    u32 name_len
    char name[name_len]
    u8  gap[...]          # normally the single NUL terminating the name
    u8  body[body_size]   # begins with u64 file_id, u32 class_hash again
}
```

- `record_size` covers the **whole** record (12 + name_len + gap + body_size),
  so the body is located as `record_start + record_size - body_size`. Derive it
  that way rather than assuming a single NUL: at least one real record has
  `name_len == 0` and an 8-byte gap. Preserve the gap verbatim.
- `[VERIFIED]` Three independent cross-checks hold on every record: the TOC id
  equals the id inside the body, the outer class hash equals the hash inside the
  body, and the TOC `record_size` equals the walked record length. Enforce all
  three and a payload of a different shape fails loudly instead of silently.
- `[VERIFIED]` `class_hash` is plain **CRC32 of the class name**:
  `crc32("Skeleton") == 0x24AECB7C`. 49 of the 50 distinct class hashes seen
  across 6,563 resources resolve to real names this way (Animation, Mesh,
  Material, Skeleton, Entity, and so on).
- `[VERIFIED]` Coverage: 473 of 479 payloads across three whole forges plus a
  400-entry sample of `DataPC.forge` parse as containers, and all 473 rebuild
  byte-identically. The 6 exceptions are 3 `GlobalMetaFile` entries (a different
  structure) and 3 `PrefetchingFileInfos` (see below).

### `PrefetchingFileInfos` and `GlobalMetaFile`

`[VERIFIED]` Every forge ends with a `PrefetchingFileInfos` entry that carries
the data-file magic but **not** the block-set layout; its block count parses as
nonsense. It is the standing decompression holdout.

Usefully, its payload contains the FileDataIDs of the real entries and **no
sizes and no offsets**, so a repack that reflows offsets and changes payload
sizes does not invalidate it. Passing it through verbatim is safe.
`GlobalMetaFile` is not a data file at all (no magic) and likewise passes
through.

## Base forge vs patch forge: which copy the game reads

`[VERIFIED]` This decides where an edit has to go, and getting it wrong makes a
test silently inconclusive rather than failing.

`DataPC.forge` and `DataPC_patch_01.forge` both carry an entry named
`GR_PLAYER_Template` with the **same entry file_id**, but they are not the same
container: 1,838,722 bytes and 1,241 resources in the base, 4,916,946 bytes and
2,238 resources in the patch. By resource file_id, 1,230 are in both, 11 are
base-only, 1,008 are patch-only, and of the 1,230 shared, **277 differ in
bytes**.

The base-only set includes the 100-bone player body rig, and the game plainly
still has a player body. So the engine is not taking the newest container
wholesale: **it resolves resources by resource file_id across forges, and a
patch supersedes only the ids it actually contains.**

Practical rules:

- Editing a resource present in **both** forges means editing the patch copy (or
  both), or the change is masked and your test proves nothing.
- Editing a **base-only** resource is unambiguous: the base is the only copy.
- The small forges (GhostRoom, TitleScreen and their patches) contain no
  Skeleton resources at all, so a skeleton edit cannot be tested against them.

## What a writer can and cannot do

Established by building one:

- `[VERIFIED]` A no-op rebuild is **byte-identical (SHA-256) on all 21 forges**
  of a 62 GB install, up to a 21 GB / 63,351-entry archive.
- `[VERIFIED, in the retail game]` A rebuilt forge with **our** payload bytes,
  **our** block layout, **our** checksums and **reflowed offsets** loads
  correctly, provided the set structure and compression byte are preserved.
  The failed single-set attempt doubles as the positive control: it black-screens,
  which proves the engine really does read that archive and a later clean boot is
  not a false pass.
- LZO output is not canonical, so a byte-identical repack of a **changed** entry
  is not achievable and is not a sensible goal. Copy unchanged entries' payload
  bytes verbatim instead of re-encoding them.
- **Replace entries only.** Nothing has verified that the game tolerates a
  changed entry count, so adding, removing or reordering entries is unproven.

Two standing cautions for anyone shipping this kind of edit: Steam file
verification reverts it, and any game patch invalidates it.
