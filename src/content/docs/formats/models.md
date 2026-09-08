---
title: Models (*.md2)
---

MD2 files store 3D models: static mesh geometry (rides, terrain, props), and separately, the
keyframe animations that pose a model's meshes over time (a gate's doors swinging, a dinosaur's
head turning). Both kinds share one file format and the same `.md2` extension - which kind a
given file is has to be determined from its contents, not its name.

An animation file sits beside the mesh it animates, with an `M1`, `M2`, ... suffix on the base
model's filename - `droid.MD2` is a mesh, `droidM1.MD2`/`droidM2.MD2` are animations of it. This
holds for 1274 of the 1279 animation files in the game; the five exceptions are `SCALE.MD2`,
`ROTATE.MD2`, `FLY.MD2`, `anim.MD2` and `scatM1.md2`, which don't sit next to an identifiable
base model.

> This page reflects an ongoing reverse-engineering effort - see **Open questions** at the end
> of each section for what isn't nailed down yet. Every offset and rule stated as fact here has
> been checked against the game's full model data (over 2,300 files), not inferred from one or
> two examples.

## Shared header

Every `.md2` file, mesh or animation, opens with the same header shape. Most of it is still
unidentified - the table below lists only the fields this project's parser actually depends on;
everything else is a gap of unknown content, not a claim that nothing is there.

| Offset | Size     | Description                                                                          |
| ------ | -------- | ------------------------------------------------------------------------------------- |
| 0x00   | 4 bytes  | Magic number - `46 5D D1 1C` (little-endian `0x1CD15D46`)                             |
| 0x04   | 4 bytes  | Constant - always `0xDD`                                                              |
| 0x08   | 4 bytes  | Constant - always `0xCB`                                                              |
| 0x0C   | 4 bytes  | Unknown - varies per file                                                             |
| 0x36   | 2 bytes  | Frame/texture count                                                                   |
| 0x44   | 2 bytes  | Mesh count                                                                            |
| 0x50   | 4 bytes  | Frame table offset (see **Textures**, below)                                          |
| 0x54   | 4 bytes  | Frame data table offset (see **Textures**, below)                                     |
| 0x70   | 4 bytes  | Mesh table offset - **0 marks this file as animation data**, see **Animation** below  |
| 0x80   | -        | Start of the model's overall bounding box (not yet parsed - see Open questions)       |
| 0x98   | 4 bytes  | Animation data block offset - animation files only, see **Animation** below           |

Everything from 0x10 to 0x36, 0x38 to 0x44, and 0x46 to 0x50 is an unidentified gap. There is
almost certainly more structure in 0x80 onward that this project doesn't yet read for static
meshes either.

### Open questions

- The meaning of the 0x0C field, and everything in the unidentified gaps above.
- The exact shape of the bounding box at 0x80 (min/max as two vectors, one vector plus extents,
  etc.) - its presence is inferred only from animation files never containing a float triple
  that reproduces it, not from having parsed it directly.

## Static meshes

A static mesh file has a non-zero mesh table offset at 0x70. Everything the mesh needs -
vertices, UVs, faces, materials, texture names - is reachable from the header fields above plus
the per-mesh table this section describes.

### Textures

The frame table (offset at 0x50) is an array of *frame count* (0x36) 8-byte entries, immediately
followed by that many 20-byte null-padded ASCII texture filenames - one string per entry, in the
same order. A material's `FrameOffset` (see **Materials** below) is a byte offset that lands
inside this 8-byte-entry array; dividing `(FrameOffset - frameTableOffset)` by 8 gives that
material's index into it.

The frame data table (offset at 0x54) is a second, parallel array of *frame count* 16-byte
records, indexed the same way:

| Size    | Description                                                    |
| ------- | --------------------------------------------------------------- |
| 4 bytes | Value - unknown                                                 |
| 4 bytes | Always 0                                                         |
| 2 bytes | Padding                                                          |
| 2 bytes | Always 1                                                         |
| 4 bytes | Offset of this frame's 20-byte texture filename string           |

In practice this points at the very same filename strings that follow the frame table - the
two tables describe the same textures, reached two different ways. A material resolves its
texture name via the frame data table (`FrameNameOff`), not by reading the frame table's strings
directly.

A material's `FrameOffset` of exactly 0 is a sentinel meaning "no texture", not a real offset -
real offsets always start at the frame table's own offset.

#### Open questions

- The 8 bytes of each frame table entry, and the 4-byte "Value" field of each frame data record.

### Mesh table

The mesh table (offset at 0x70) is an array of *mesh count* (0x44) fixed-size 160-byte records,
one per mesh:

| Offset | Size     | Description                                                              |
| ------ | -------- | ------------------------------------------------------------------------- |
| 0x00   | 16 bytes | Unknown                                                                    |
| 0x10   | 64 bytes | 4x4 transform matrix (16 floats, row-major) - this mesh's placement       |
| 0x50   | 4 bytes  | Unknown                                                                    |
| 0x54   | 4 bytes  | Offset of this mesh's null-terminated ASCII name                          |
| 0x58   | 2 bytes  | Vertex count                                                               |
| 0x5A   | 2 bytes  | Material count                                                             |
| 0x5C   | 2 bytes  | Face count                                                                 |
| 0x5E   | 2 bytes  | Vertex order length (see **Vertex order**, below)                         |
| 0x60   | 4 bytes  | Vertex data offset                                                         |
| 0x64   | 4 bytes  | Unknown                                                                    |
| 0x68   | 4 bytes  | UV data offset                                                             |
| 0x6C   | 4 bytes  | Material table offset                                                     |
| 0x70   | 4 bytes  | Face data offset                                                           |
| 0x74   | 4 bytes  | Unknown                                                                    |
| 0x78   | 12 bytes | Bounding box minimum (3 floats) - see **Vertex data**                     |
| 0x84   | 12 bytes | Bounding box maximum (3 floats) - see **Vertex data**                     |
| 0x90   | 4 bytes  | Unknown                                                                    |
| 0x94   | 4 bytes  | Vertex order table offset (see **Vertex order**, below)                   |
| 0x98   | 8 bytes  | Unknown                                                                    |

This mesh's bounding box (0x78/0x84) matters beyond just culling: it's also the box that
animation files quantise vertex positions into - see **Animation** below.

#### Open questions

- The four unknown fields (0x00, 0x50, 0x64, 0x74, 0x90, 0x98) - none have been narrowed down
  beyond "not used by anything this project's renderer needs".

### Vertex data

Vertex positions are **not** stored as one `(x, y, z)` triple per vertex. They're grouped into
batches of up to 4 vertices, and within a batch, stored axis-major: four X floats, then four Y
floats, then four Z floats. The vertex count is padded up to the next multiple of 4 for this
purpose (trailing padding vertices are read but discarded).

So for vertex count 5, the layout at the vertex data offset is:

```
X0 X1 X2 X3   X4 X? X? X?      (two batches of 4 X floats each - second batch mostly padding)
Y0 Y1 Y2 Y3   Y4 Y? Y? Y?
Z0 Z1 Z2 Z3   Z4 Z? Z? Z?
```

This is the *raw* vertex list, indexed by source vertex index. It is not the order the mesh is
actually drawn in - see **Vertex order**, next.

UV coordinates (at the UV data offset) use the same batches-of-4, axis-major layout, but the
count that matters here is the **vertex order length** (0x5E in the mesh record), not the raw
vertex count - UVs are stored per final, reordered vertex slot, not per source vertex.

### Vertex order

The vertex order table (offset at 0x94, length 0x5E) is an array of *vertex order length*
`ushort`s. Reading it produces the mesh's real, final vertex list: entry `i` of this table is
the source vertex index (into the raw vertex data above) that should become vertex `i` of the
drawn mesh.

This reordering exists to group vertices contiguously by material: each material record (see
below) gives a `StartIndex`/`EndIndex` range, and a final vertex `i` uses whichever material's
range contains `i`. So the vertex order table isn't just a remap - it's what makes each
material's vertices sit in one unbroken run, which is what lets a material be described by a
single index range instead of a vertex list of its own.

Face indices (see **Faces**, below) refer to this final, reordered vertex list - not the raw
vertex data.

#### Open questions

- Whether the vertex order table is ever used for anything beyond material grouping (e.g.
  triangle strips or fans) - not observed in the game's data so far.

### Faces

The face table (offset given in the mesh record) is *face count* records of 8 bytes each:

| Size    | Description                                          |
| ------- | ----------------------------------------------------- |
| 2 bytes | Unknown                                                |
| 2 bytes | Vertex index A (into the reordered vertex list)        |
| 2 bytes | Vertex index B                                         |
| 2 bytes | Vertex index C                                         |

Each record is one triangle. Winding order needs to be reversed from how it's stored to render
correctly with a standard right-handed culling convention.

#### Open questions

- The leading unknown `ushort` of each face record.

### Materials

The material table (offset given in the mesh record, *material count* entries) uses 16-byte
records:

| Size    | Description                                                          |
| ------- | ---------------------------------------------------------------------- |
| 4 bytes | Frame offset - see **Textures**, above (0 = no texture)                |
| 2 bytes | Unknown                                                                 |
| 2 bytes | Unknown                                                                 |
| 2 bytes | Start index - first reordered vertex index using this material         |
| 2 bytes | End index - last reordered vertex index using this material (inclusive)|
| 4 bytes | Unknown                                                                 |

A material with a non-zero frame offset also has a 4-byte flags word, read directly from that
frame offset in the file (i.e. the frame offset does double duty: it locates both the frame
table entry for the texture name, and a flags value sitting at that same byte offset).

#### Open questions

- The two unidentified 2-byte fields, and the meaning of the flags word.

### Normals

Vertex normals aren't stored in the file at all - they're computed by accumulating each
triangle's face normal onto its three vertices and re-normalizing, the usual smooth-shading
approach. There's no format detail here; it's purely a rendering choice.

## Animation

An animation file has a mesh table offset of **exactly 0** at 0x70 - the same field that's a
real pointer for static meshes. Animation files carry no vertex positions, texture names, or
mesh geometry of their own. Instead they carry a list of **tracks**, each posing one node of the
base model (identified by the shared filename prefix) over a range of authoring frames.

A track is not one kind of animation. It is a bundle of independent **channels** - a fountain's
water mesh morphs its vertices, scrolls its texture and carries a timing scalar all on the same
track - and each channel kind owns its own slot in the track descriptor. That is what makes the
format safe to read incrementally: a channel you don't understand costs nothing, because its
data lives in a slot you simply don't read.

Of the game's 1279 animation files, 1164 (91%) carry at least one of the three channel kinds
documented below.

### Locating the tracks

The uint at 0x98 points at a 72-byte **animation block**. That pointer is valid in 1278 of the
1279 animation files.

| Offset (from block start) | Size    | Description                                       |
| -------------------------- | ------- | --------------------------------------------------- |
| 0x08                        | 4 bytes | Last frame of the animation                         |
| 0x10                        | 2 bytes | Sum of every track's rotation keyframe count (a cross-check, not needed to parse) |
| 0x12                        | 2 bytes | Track count                                          |
| 0x2C                        | 4 bytes | Track table offset                                   |

The track table offset is trustworthy on its own terms, which matters because there's nothing
else to check it against: the table is exactly `trackCount * 64` bytes and **ends exactly where
the animation block begins**, i.e. `tableOffset + trackCount * 64 == blockOffset`. That identity
holds for 1189 of the 1279 files. A reader should apply it and treat a file that fails as
carrying no readable animation, rather than reading a table that isn't one.

### Track descriptors

Each track is a 64-byte descriptor:

| Offset (from descriptor start) | Size    | Description                                          |
| -------------------------------- | ------- | ------------------------------------------------------ |
| 0x00                               | 4 bytes | This track's own index (0-based, sequential - a validity check) |
| 0x04                               | 4 bytes | Channel flags - see below                              |
| 0x0C                               | 4 bytes | A frame value, close to but not always the track's last keyframe |
| 0x10                               | 2 bytes | Rotation keyframe count (channel `0x8` only)          |
| 0x14                               | 2 bytes | **Target node** - see **Target resolution** below      |
| 0x16                               | 2 bytes | Unknown, but **not** part of the target                |
| 0x18                               | 4 bytes | Data pointer for channel `0x1` (undecoded)             |
| 0x1C                               | 4 bytes | Rotation keyframes (channel `0x8`)                     |
| 0x28                               | 4 bytes | Vertex morph descriptor (channel `0x1000`)             |
| 0x2C                               | 4 bytes | UV animation descriptor (channel `0x10000`)            |
| 0x30                               | 4 bytes | Timing scalar, 16.16 fixed point (channel `0x20000`)   |

The flag word at 0x04 says which channels the track carries, and every bit owns exactly one
slot. Counted across every track in every animation file in the game:

| Flag bit | Owns slot | Channel                    | Decoded? |
| -------- | ---------- | --------------------------- | -------- |
| 0x00008   | count at +0x10, data at +0x1C | Rotation      | Yes      |
| 0x01000   | +0x28      | Vertex morph                | Yes      |
| 0x10000   | +0x2C      | UV animation                | Yes      |
| 0x20000   | +0x30      | A 16.16 fixed-point scalar  | Partly - the encoding is clear, the meaning isn't |
| 0x00001   | +0x18      | Unidentified                | No       |

The correspondence is exact: across all 1279 files, no track sets one of those bits without
filling its slot, or fills a slot without setting the bit.

> **The target at 0x14 is a ushort, not a uint.** 0x16 holds an unrelated value and is nonzero
> on 595 of the game's rotation tracks, so reading 32 bits there produces a garbage node index -
> large enough that a range check discards a track that was perfectly good.

### Target resolution

The target indexes the base model's **node** list, which for models with no extra hierarchy is
simply its mesh list. `Jun_gateM1.MD2` has two rotation tracks targeting nodes 0 and 1, which
are `Jun_gate.MD2`'s two door meshes.

Some models have more nodes than meshes and index past the mesh list - `Advisor.MD2` has 25
meshes but its animations reach node 28 - and a handful land *within* the mesh list on a mesh
that clearly isn't the intended one (`droidm2.MD2` names a 16-vertex mesh while carrying 3561
morph channels). A range check alone is therefore not enough. For morph tracks there is a
reliable second test, described below; without one, a reader should prefer skipping a track to
applying it to the wrong mesh.

### Rotation (bit 0x8)

Turns a whole node about its own origin. The keyframe count is the **ushort** at descriptor
0x10 and the data offset the uint at 0x1C. Each keyframe is 20 bytes:

| Size    | Description                                              |
| ------- | ----------------------------------------------------------|
| 2 bytes | Frame index                                                |
| 2 bytes | Flags - only `0x0000` and `0xFFFF` are observed; meaning unconfirmed |
| 4 bytes | Quaternion X                                               |
| 4 bytes | Quaternion Y                                               |
| 4 bytes | Quaternion Z                                               |
| 4 bytes | Quaternion W                                               |

Frame indices strictly ascend within a track, and every one of the 3039 rotation tracks in the
game decodes to a unit quaternion (within 0.01) at every keyframe. That pair of properties is
what a reader should validate before trusting a track.

Model space is Y-up, so a quarter turn about Y is a door swinging. `Jun_gateM1` takes the gate's
two doors from identity to a quarter turn, and `Jun_gateM2` is exactly the inverse - open, then
shut.

### Vertex morph (bit 0x1000)

Reshapes a mesh vertex by vertex. The slot at descriptor 0x28 points at a 16-byte descriptor:

| Offset  | Size    | Description          |
| ------- | ------- | ---------------------- |
| 0x02    | 2 bytes | Record count           |
| 0x0C    | 4 bytes | Record table offset    |

Each track has its **own** descriptor and its own channel space, so one animation morphs as many
meshes as it has morph tracks. 768 animation files carry morph tracks, 368 of them more than
one, 1766 tracks in total - `ratraceM1.MD2` morphs four meshes at once.

> Because each morph track is self-contained, a reader that only looks at a fixed header offset
> finds just the first one. Three of `ratrace`'s four morphing meshes have 64 vertices each, so
> matching a track to a mesh by vertex count cannot tell them apart either - the target index is
> the only thing that can.

Each record in the table is 20 bytes:

| Size    | Description                                                          |
| ------- | ---------------------------------------------------------------------- |
| 2 bytes | Keyframe count (`a`)                                                    |
| 2 bytes | Channel count (`b`)                                                     |
| 4 bytes | Offset of a `b`-entry array of channel IDs (`ushort` each)              |
| 4 bytes | Offset of an `a`-entry array of ascending frame indices (`ushort` each) |
| 4 bytes | Offset of an `a * b`-entry array of packed values                       |
| 4 bytes | Unknown                                                                 |

The channel ID and frame index arrays are each padded up to a 4-byte boundary, so the gap
between the channel IDs and the frame indices is `2*b` or `2*b + 2` bytes, and between the frame
indices and the values `2*a` or `2*a + 2`. A reader has to accept either.

The value array is **entry-major**: the value for keyframe `e` of channel slot `k` sits at
`valueOffset + (e * b + k) * 4` - all channels of keyframe 0 first, then all channels of
keyframe 1, not grouped by channel.

A morph track has exactly **one channel per vertex of its target mesh, plus two** trailing
channels that aren't vertices. 457 of the 465 tracks whose base model resolves satisfy that
exactly, and the 8 that don't are the mistargeted ones described under **Target resolution** -
which makes `channelCount == targetMesh.vertexCount + 2` a good validity test as well as a
description.

Channels the animation doesn't actually move are still present, in a record holding a single
keyframe of that channel's rest value, so a full mesh pose is always reconstructible by sampling
every channel.

**Value decoding.** Each 4-byte value is a vertex position quantised into three signed 10-bit
fields - X in bits 0-9, Y in 10-19, Z in 20-29 (bits 30-31 unused). Each field is a signed value
from -512 to 511 mapped linearly onto the *target mesh's* bounding box (from its mesh table
record): -512 is that axis's box minimum, +511 its maximum.

```
component(raw, shift, min, max):
    field = signed_10_bit((raw >> shift) & 0x3FF)
    centre = (min + max) / 2
    return centre + field * (max - min) / 1023
```

Verified to R² >= 0.999997 per axis (max error ~0.028 units, exactly the quantisation step) by
decoding every channel's rest keyframe and comparing against a known mesh's actual vertices.

### UV animation (bit 0x10000)

Slides texture coordinates - this is how the game animates water. The slot at descriptor 0x2C
points at a 20-byte descriptor:

| Offset  | Size    | Description                                |
| ------- | ------- | -------------------------------------------- |
| 0x00    | 4 bytes | Entry count `n`                              |
| 0x04    | 4 bytes | Offset of the index table                    |
| 0x08    | 4 bytes | Total component count `c`                    |
| 0x0C    | 4 bytes | Offset of the duration table                 |
| 0x10    | 4 bytes | Offset of the value table                    |

The three tables are contiguous, in the order index, value, duration, and exactly sized by those
two counts:

- **Index table**, `n * 4` bytes: per entry, a ushort first component and a ushort component
  count. UV components are two per coordinate, so an entry covering a whole UV is `(2i, 2)`.
- **Value table**, `c * 8` bytes: per entry, its component count of start floats followed by the
  same number of end floats. So a 2-component entry is `(u_start, v_start, u_end, v_end)`.
- **Duration table**, `n * 4` bytes: per entry, a ushort pair whose **high** half is the end
  frame.

All 690 UV channels in the game satisfy every one of those invariants - the per-entry component
counts sum to the stated total, and both `indexTable + 4n == valueTable` and
`valueTable + 8c == durationTable` hold without exception.

A channel that animates every UV of its mesh has the identity index table `(2i, 2)` with `n`
equal to the mesh's **vertex order length** (the UV count, not the vertex count). One that
animates a subset names the components it wants - `arcadeM1`'s `screen` mesh animates 6 entries
out of 53 UVs.

`Jun_isleM1.MD2` is a clear worked example: it scrolls all 128 of `Post Ripples01`'s UVs by
`(-1, -1)` over 100 frames, and 104 of the `Island` mesh's 298 UVs by the same delta - the
shoreline foam lapping the beach, with the rest of the island held still.

### Sequencing

Nothing in the format says how a model's `M1`, `M2`, ... animations are ordered, when they
should play, or whether they loop. A gate's "doors open" and "doors close" are just two files,
indistinguishable by anything in the `.md2` data. In the original game this is driven externally,
by a `TRIGANIM` instruction in that ride's [compiled script](/formats/rsse) - see the gate
example on the [RSS](/formats/rss) page.

### Open questions

- The channel at flag `0x1` (data pointer at descriptor +0x18) - location known, contents not.
- The channel at flag `0x20000` is a single 16.16 fixed-point scalar at +0x30 rather than a
  pointer, so it's a per-track constant rather than a keyframed channel. Values are small
  fractions (0.01 - 0.15); what they scale is unknown.
- The frame-ish value at descriptor +0x0C, and the unknown 4 bytes ending each morph record.
- What the two extra channels beyond a morph track's vertex count represent.
- Why a few models' target indices don't land on the mesh the data clearly belongs to - i.e.
  what the node list actually contains for models that have more nodes than meshes.
- The 90 animation files that fail the track table identity, and the 25 that pass it but carry
  only the undecoded channel kinds.
