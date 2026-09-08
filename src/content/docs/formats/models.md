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

Nothing in the format says how a model's animations are sequenced or when they should play - in
particular, a gate's "doors open" and "doors close" animations are just two more `M`-suffixed
files, indistinguishable from each other by anything in the `.md2` data. In the original game
this is almost certainly driven externally, by a `TRIGANIM` instruction in that ride's
[compiled script](/formats/rsse) - see the `.closed`/gate example on the
[RSS](/formats/rss) page, which triggers an animation by index in response to game state.

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
mesh geometry of their own; instead they carry one or more **tracks**, each of which poses part
of the base model (identified by the shared filename prefix) over a range of authoring frames.

Across all 1279 animation files in the game, two distinct track kinds have been decoded, reached
via two different, independently-located tables within the same file:

| Track kind        | Files          | Poses                                              |
| ------------------ | -------------- | --------------------------------------------------- |
| Vertex (morph)      | 282 (22%)      | Reshapes a single mesh, vertex by vertex             |
| Rotation             | 686 (54%)      | Turns one or more whole meshes about their own origin |
| *(undecoded)*        | 311 (24%)      | Loads with no tracks - see Open questions            |

No file in the game's data has been found to use both kinds together, though nothing in the
format rules it out - each kind's table is located independently of the other.

### Vertex (morph) animation

This kind reshapes exactly one mesh: a "channel" exists for every vertex of the mesh it
targets, plus two trailing channels whose purpose isn't identified, so the channel count is
always (target mesh's vertex count) + 2. That's how a consumer identifies which mesh an
animation drives - by matching this count against each of the base model's meshes.

The track table is located via two header fields:

| Offset | Size    | Description                        |
| ------ | ------- | ------------------------------------ |
| 0xBA   | 2 bytes | Track record count                   |
| 0xC4   | 4 bytes | Track record table offset            |

Some animation files put unrelated data in these two slots - notably the ones with an
undecoded track kind. A reader must validate the records below rather than trust this table
unconditionally, and must fail soft (skip the file) rather than throw when validation fails.

Each record is 20 bytes:

| Size    | Description                                                          |
| ------- | ---------------------------------------------------------------------- |
| 2 bytes | Keyframe count (`a`)                                                    |
| 2 bytes | Channel count (`b`)                                                     |
| 4 bytes | Offset of a `b`-entry array of channel IDs (`ushort` each)              |
| 4 bytes | Offset of an `a`-entry array of ascending frame indices (`ushort` each) |
| 4 bytes | Offset of an `a * b`-entry array of packed values (see below)          |
| 4 bytes | Unknown                                                                 |

The channel ID and frame index arrays are each padded up to a 4-byte boundary, so the gap
between the channel ID array and the frame index array is `2*b` or `2*b + 2` bytes, and the gap
between the frame index array and the value array is `2*a` or `2*a + 2` bytes - a reader has to
accept either, not just the unpadded size.

The value array is **entry-major**: the value for keyframe `e` of the record's channel slot `k`
sits at `valueOffset + (e * b + k) * 4` - i.e. all channels for keyframe 0 first, then all
channels for keyframe 1, and so on, not grouped by channel.

Channels the animation doesn't actually move are still present, in their own record holding a
single keyframe of that channel's rest value - so a full mesh pose can always be reconstructed
by sampling every channel, moving or not.

**Value decoding.** Each 4-byte value is a vertex position, quantised into three signed 10-bit
fields packed into the 32 bits - X in bits 0-9, Y in bits 10-19, Z in bits 20-29 (bits 30-31
unused). Each field is a signed value from -512 to 511, mapped linearly onto the *owning mesh's*
bounding box (from its mesh table record - see **Mesh table**, above): -512 maps to that axis's
box minimum, +511 to its maximum.

```
component(raw, shift, min, max):
    field = signed_10_bit((raw >> shift) & 0x3FF)
    centre = (min + max) / 2
    return centre + field * (max - min) / 1023
```

Verified against real game data to R² ≥ 0.999997 per axis (max error ~0.028 units, consistent
with exactly the quantisation step), by decoding every channel's rest keyframe and comparing
against a known mesh's actual vertex positions.

#### Open questions

- What the two trailing non-vertex channels represent.
- The meaning of the record's final 4-byte "unknown" field.
- How multiple `M`-suffixed animations of the same mesh are meant to be sequenced or blended -
  nothing in the file format addresses this (see the note on `TRIGANIM` at the top of this
  page).

### Rotation animation

This kind turns one or more whole meshes about their own origin - a gate's two doors, say,
independently swinging open. Unlike vertex animation, one file can drive several meshes at
once, each with its own track.

The rotation track table sits inside a 72-byte **animation data block**, whose offset is given
by the header field at 0x98 (valid - i.e. actually pointing at a well-formed block - in 1278 of
the game's 1279 animation files):

| Offset (from block start) | Size    | Description                                                      |
| -------------------------- | ------- | ------------------------------------------------------------------ |
| 0x08                        | 4 bytes | Last frame of the animation                                        |
| 0x10                        | 2 bytes | Sum of every track's keyframe count (a cross-check, not required for parsing) |
| 0x12                        | 2 bytes | Track count                                                         |
| 0x2C                        | 4 bytes | Track table offset                                                  |

The track table offset is trustworthy on its own terms: the table is exactly
`trackCount * 64` bytes long, and **ends exactly where the animation block begins** - i.e.
`tableOffset + trackCount * 64 == blockOffset`. That identity holds for 1189 of the 1279
animation files and is the check a reader should apply; files that fail it should be treated
as not carrying a (decodable) rotation track table, not as an error.

Each 64-byte **track descriptor** can carry more than one kind of channel, distinguished by bits
of a flags word - only the rotation channel (bit `0x8`) is decoded so far:

| Offset (from descriptor start) | Size    | Description                                              |
| -------------------------------- | ------- | ------------------------------------------------------------ |
| 0x00                               | 4 bytes | This track's own index (0-based, sequential - a validity check) |
| 0x04                               | 4 bytes | Flags - which channel kinds this track carries (see below)   |
| 0x10                               | 2 bytes | Rotation keyframe count (meaningful only when flag `0x8` is set) |
| 0x14                               | 4 bytes | Target index - see **Target resolution**, below              |
| 0x1C                               | 4 bytes | Rotation keyframe data offset (meaningful only when flag `0x8` is set) |

Counted across every track in every animation file in the game, the flags word's bits and the
data-pointer slot each one owns:

| Flag bit | Data pointer (from descriptor start) | Tracks in the game | Decoded? |
| -------- | --------------------------------------- | ------------------- | -------- |
| 0x00008   | Count at +0x10, data at +0x1C            | 3,039                | Yes - rotation, documented above |
| 0x00001   | Data at +0x18                            | 3,039                | No       |
| 0x01000   | Data at +0x28                            | 1,058                | No       |
| 0x10000   | Data at +0x2C                            | 704                  | No       |
| 0x20000   | Data at +0x30                            | 2,178                | No       |

A track's flags aren't exclusive - one track can carry several channel kinds at once, each
sitting in its own slot of the same 64-byte descriptor. A reader only needs to look at the bit
for the channel kind it understands and can leave the others alone.

**Rotation keyframes** are 20 bytes each, an array of *rotation keyframe count* entries at the
descriptor's data offset:

| Size    | Description                                              |
| ------- | ----------------------------------------------------------|
| 2 bytes | Frame index                                                |
| 2 bytes | Flags - observed values are only `0x0000` or `0xFFFF`; meaning unconfirmed |
| 4 bytes | Quaternion X                                               |
| 4 bytes | Quaternion Y                                               |
| 4 bytes | Quaternion Z                                               |
| 4 bytes | Quaternion W                                               |

Frame indices strictly ascend within a track. Every one of the 3,039 rotation tracks in the
game's data decodes to a unit quaternion (length within 0.01 of 1) at every keyframe - that,
plus strictly-ascending frames, is what a reader should validate before trusting a track.

**Target resolution.** The 4-byte target index at descriptor +0x14 addresses a node in the base
model, and for models whose only nodes are their meshes, that's simply the mesh index - two
rotation tracks with target indices 0 and 1 in `Jun_gateM1.MD2`/`Jun_gateM2.MD2` turn
`Jun_gate.MD2`'s two door meshes from identity to a quarter turn about the model's up axis and
back, i.e. the gate swinging open and shut.

Models with additional hierarchy beyond their meshes index past the mesh list - `Advisor.MD2`
has 25 meshes but its animations reach target index 28. What those extra three nodes are (and
therefore what a target index in that range should actually move) is unresolved; a reader
should treat an out-of-range target index as "skip this track" rather than guess.

#### Open questions

- Data pointer meaning for flag bits `0x1`, `0x1000`, `0x10000` and `0x20000` - locations are
  known (see the flags table above), record layout is not.
- The three extra hierarchy node(s) referenced by target indices beyond a model's mesh count.
- The meaning of the 20-byte keyframe's flags field.
- The remaining ~24% of animation files that validate against neither this table nor the vertex
  animation table above - what track kind(s) they actually contain hasn't been identified.
