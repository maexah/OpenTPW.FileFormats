---
title: Models (*.md2)
---

Theme Park World's models are Bullfrog's own `.md2` format. It has nothing to do with Quake II's
format of the same extension. MD2 files store 3D models: static mesh geometry (rides, terrain, props),
and separately, the keyframe animations that pose a model's meshes over time (a gate's doors swinging,
a dinosaur's head turning). Both kinds share one file format and the same `.md2` extension. Which
kind a given file is has to be worked out from its contents, not its name:

| Kind            | How to tell                                                      | Files in the game |
| --------------- | ---------------------------------------------------------------- | ----------------- |
| **Static mesh** | the mesh table offset at 0x70 is non-zero, and 0x98 is zero      | 850               |
| **Animation**   | 0x70, 0x50 and 0x54 are all zero, and 0x98 is not                | 1,279             |

An animation file carries no geometry at all (no vertex positions, no texture names). It poses the
nodes of a separate base model. Parsing one as a mesh is what runs off the end of the file. The two
tests agree on every file except `wr_tunnelm.md2`, an animation of an older version that has no
animation block either (see [Three versions](#three-versions-and-the-loader-takes-one)).

An animation file sits beside the mesh it animates, and its name is the base model's name followed by
a role letter and an optional number. `droid.MD2` is a mesh, and `droidc.MD2`, `droidi.MD2`,
`droidm1.MD2` and `droidm2.MD2` animate it. This holds for 1,275 of the 1,279 animation files in the
game. The four exceptions do not follow the role naming: `TROUGH_SCALE.MD2` and `TROUGH_ROTATE.MD2`
in the Halloween `rides/c_hade.wad`, `BAT_FLY.MD2` in `rides/c_scat.wad`, and `globe\plane_anim.MD2`
in `lobby.wad`.

**The number is optional**, and leaving it out is not another way of spelling `M1`. It changes which
file the engine loads. `fountainM.MD2` is the Round Fountain's entire animation, and 197 of the
game's base models ship theirs only this way. See
[Which animation file a model takes](#which-animation-file-a-model-takes).

> This page reflects an ongoing reverse-engineering effort. See **Open questions** at the end
> of each section for what isn't nailed down yet. Every offset and rule stated as fact here has
> been checked against the game's full model data, not inferred from one or two examples. The
> game ships 2,129 `.md2` files. 2,118 are inside the game's `.wad` archives. The other 11 are
> loose: the two arrows in `data/generic/dynamic/` (below), and three UI meshes (`bankrupt`,
> `congrats`, `paused`), each copied identically into three language folders. Counts on this page
> are over the archives unless a sentence says otherwise.
>
> Items marked **(engine-confirmed)** were also checked against the original game's own
> pointer-relocation routine (`FUN_0046d6d0`). That routine walks a freshly loaded `.md2` and turns
> every stored file offset into an absolute pointer. Whatever it relocates is a pointer and whatever
> it skips is not. The counts and strides it loops with are the record sizes. So those items are
> direct statements of the format, not inferences from the data. Where an item names another
> function or address, it was checked against that code instead.

## Shared header

Every `.md2` file, mesh or animation, opens with the same header shape. Most of it is still
unidentified. The table below lists only the fields that have been identified. Everything else is a
gap of unknown content, not a claim that nothing is there.

| Offset | Size     | Description                                                                          |
| ------ | -------- | ------------------------------------------------------------------------------------- |
| 0x00   | 4 bytes  | Magic number: `46 5D D1 1C` (little-endian `0x1CD15D46`), in every file                   |
| 0x04   | 4 bytes  | **Format version**. The engine loads only `0xDD`. See [Three versions](#three-versions-and-the-loader-takes-one) (engine-confirmed) |
| 0x08   | 4 bytes  | **Animation format version**. Must be exactly `0xCB`, or the animation block is discarded (engine-confirmed) |
| 0x14   | 4 bytes  | Save time, as a Unix timestamp. Read this way, every version-`0xDD` file dates from 16 April 1999 to 20 January 2000 |
| 0x18   | 20 bytes | The name the file was saved under, null-padded ASCII. It matches the file's own name in 2,031 of the 2,125 version-`0xDD` files. The other 94 carry an earlier name (`speaker2.MD2` says `speaker.MD2`) |
| 0x30   | 4 bytes  | Flags. Bit 0 gates the whole mesh/material fixup pass (engine-confirmed). It is set in every static mesh and in no animation file. Bit `0x4` makes the model's animation add to its authored pose instead of replacing it: a rotation key is composed with the node's rotation, and a position, path or raw-vertex-frame sample is added to what the node already holds (`FUN_00471860`, at `0x00471d53` and `0x0047205d`). It is set in 25 static meshes, all coaster parts: every `StdPylon` and `LoopPylon` (value `0xD`), and `candy_c`'s `ch_track`, `megacost`'s `cart` and `shocker`'s `CAR` and `RAIL` (value `0x5`). No model with a path sets it. Bit `0x20` is stored in 880 of the 2,129 files, all version `0xDD` and none a static mesh (the whole census: 0 in 91, 1 in 529, 3 in 5, 4 in 7, 5 in 4, 8 in 11, 9 in 288, 13 in 21, 16 in 292, 25 in 1, 32 in 878, 34 in 2), and `FUN_00470b30` sets it at load when the clip's word at `+0x12` is non-zero (whose header it writes is not re-checked). The stored values all fit in the low byte; the engine sets higher bits at load |
| 0x36   | 2 bytes  | Frame/texture count                                                                   |
| 0x40   | 2 bytes  | **Path count**, the number of records in the table at 0xAC (engine-confirmed). See [Paths](#paths) |
| 0x42   | 2 bytes  | **Total node count**. See **Target resolution** under Animation (engine-confirmed)   |
| 0x44   | 2 bytes  | Mesh count. The meshes are the *first* 0x44 of the 0x42 nodes                        |
| 0x46   | 2 bytes  | The node the first node-lookup record names. See **Node lookup ids**                  |
| 0x48   | 2 bytes  | Count for the node-lookup table at 0x7C (engine-confirmed)                            |
| 0x4C   | 4 bytes  | Pointer, purpose unknown (engine-confirmed pointer). Zero in every file in the game   |
| 0x50   | 4 bytes  | Frame table offset (see **Textures**, below)                                          |
| 0x54   | 4 bytes  | Frame data table offset (see **Textures**, below)                                     |
| 0x58 - 0x68 | 4 bytes each | Five pointers, purposes unknown (engine-confirmed pointers). Set in every version-`0xDD` static mesh |
| 0x6C   | 4 bytes  | **Heightfield block**. Set only in the five terrain models. See [Heightfield](#heightfield) (engine-confirmed pointer) |
| 0x70   | 4 bytes  | Mesh table offset. **0 marks this file as animation data**, see **Animation** below  |
| 0x74   | 4 bytes  | Transform-only node table: `(0x42 - 0x44)` records of 88 bytes. See **Node hierarchy** |
| 0x78   | 4 bytes  | **Root node**: the one node with no parent (engine-confirmed pointer). The engine's pose walk starts from it (`FUN_0044ab30`). In all 847 version-`0xDD` static meshes (the archives' 838 and 9 loose) it names the single parentless node, node 0 in all but 60; zero in all 1,278 version-`0xDD` animation files |
| 0x7C   | 4 bytes  | Node-lookup table: 0x48 records, 20 bytes each (engine-confirmed)                     |
| 0x80   | 24 bytes | The model's box: six floats, read as min x, y, z then max x, y, z. Of the 173 models with an [.hmp](/formats/hmp/) of the same name, 167 have min ≤ max on every axis. The engine copies the `.hmp`'s own box over this as it loads it (`FUN_00451640`); the two match in 53 of the 173. See Open questions |
| 0x98   | 4 bytes  | Animation data block offset. Animation files only, see **Animation** below           |
| 0xAC   | 4 bytes  | Path table: 0x40 records, 16 bytes each, or 0 when the model has none (engine-confirmed). See [Paths](#paths) |

The engine relocates pointers at 0x4C, 0x50, 0x54, 0x58, 0x5C, 0x60, 0x64, 0x68, 0x6C, 0x70,
0x74, 0x78, 0x7C, 0x98 and 0xAC. That whole run is a pointer block, even where the purpose of an
individual entry is still unknown. At load time the engine also writes its own base pointer to 0x9C
and the raw allocation to 0xB0, so those two are runtime scratch rather than file content.

The engine's loader is always asked for one kind or the other. Asked for a mesh, it refuses a file
whose 0x98 is set. Asked for an animation, it refuses one whose 0x98 is zero (engine-confirmed).

### Three versions, and the loader takes one

Measured over every `.md2` in the game, loose and in every archive (2,129 files):

| Words at `0x04` / `0x08` | Files |
| ------------------------ | ----- |
| `0xDD` / `0xCB` | 2,125 |
| `0xCF` / `0xC9` | 2: `wr_tunnel.md2` (a mesh) and `wr_tunnelm.md2` (its animation), in the Jungle's `rides/wateride.wad` |
| `0x18` / `0x17` | 2: `garrow.MD2` and `rarrow.MD2`, loose in `data/generic/dynamic/` |

> **The engine loads only `0xDD`.** Its one model reader refuses anything above `0xDD`. It refuses
> anything below too, unless the caller sets a flag asking for the older layout. Neither of the
> reader's two callers ever sets it: the item loader passes 0, and the other caller passes 2 or 6.
> So the four odd files are shipped data the game never draws.

`garrow.MD2` and `rarrow.MD2` are a green and a red block arrow. Each is a single node named `Line01`
with 14 vertices and 24 triangles: 5 units wide at the head, 8 long pointing +Z, and 2 thick.
The two files differ only in their name at 0x18, their texture (`green.tga` against `red.tga`) and
three header words at 0x0C, 0x10 and 0x14. The word at 0x14 is a save time of 9 September 1998,
seven months before the earliest release model. Their layout is **not** the one this page describes.
They use 32-byte vertex records, 24-byte face records and a 136-byte node, reached through unaligned
pointers from 0x5A. A reader for the `0xDD` layout that meets one takes the dword at 0x70
(`0x05F90000`) as a mesh table pointer and runs off the end of the file.

`wr_tunnel.md2` is the one version-`0xCF` mesh, and it breaks the `0xDD` layout in the ways listed
under **Node hierarchy** below.

### Open questions

- The meaning of the 0x0C and 0x10 fields, and of the five pointers between 0x58 and 0x68.
- The box at 0x80 has not been measured beyond the 173 models with an `.hmp` beside them. In 6 of those, min is
  above max on some axis, so a reading other than min then max is not ruled out for every model.
- The `0xCF` and `0x18` layouts, beyond what is said above about the arrows.

## Static meshes

A static mesh file has a non-zero mesh table offset at 0x70. Everything the mesh needs
(vertices, UVs, faces, materials, texture names) is reachable from the header fields above plus
the per-mesh table this section describes.

### Node hierarchy

**A model is a tree of nodes, not a flat list of meshes**, and a node's transform is relative to
its parent. A reader that ignores the tree leaves child meshes piled at the model origin.

The ushort at 0x42 is the total node count and the ushort at 0x44 the mesh count. The meshes are
the first `0x44` nodes, in the 160-byte records at 0x70. The rest are **transform-only
nodes** in 88-byte records at 0x74. The engine indexes them with one rule (engine-confirmed):

```c
if (node < meshCount)  ptr = meshTable(0x70) + node * 0xA0;
else                   ptr = nodeTable(0x74) + (node - meshCount) * 0x58;
```

Both record kinds begin with the same header, which is what makes that work:

| Offset | Size     | Description                                                          |
| ------ | -------- | ---------------------------------------------------------------------- |
| 0x00   | 4 bytes  | Flags. See [The flag word](#the-flag-word) below                       |
| 0x04   | 4 bytes  | **Parent** node (a file offset; 0 for a root)                          |
| 0x08   | 4 bytes  | **Next sibling** node                                                  |
| 0x0C   | 4 bytes  | **First child** node                                                   |
| 0x10   | 64 bytes | This node's transform, relative to its parent                          |
| 0x50   | 2 bytes  | This record's **own index** in the combined node list (0-based)        |
| 0x52   | 2 bytes  | **Path index**, read only on a path node (flag `0x8`). See [Paths](#paths). Zero in all but 3 of the 7,520 version-`0xDD` records (`wr_tunnel.md2`'s hold float bits; see below) |
| 0x54   | 4 bytes  | Offset of the node's null-terminated ASCII name (engine-confirmed)     |

The three links are stored as file offsets into whichever of the two tables the target lives in.
A reader converts an offset back to a node index by testing which table's range it falls in.

#### The flag word

| Bit          | Meaning |
| ------------ | ------- |
| `0x200`      | **Transform-only node.** It marks a place rather than holding geometry. The engine skips material processing for any node that has it set. It is exact: all 2,606 transform-only nodes in the game's 839 models have it set, and no mesh does |
| `0x8`        | **Path node.** Set on the 27 transform-only nodes that name a path, and on nothing else. See [Paths](#paths) |
| `0x2`        | On a mesh record, turns depth writes off for that mesh (`0x005817ba`). Set only on the five meshes named `heightfield` in the terrain models, all of which are empty |
| `0x10`       | **Hidden.** Set only at runtime; no node in the shipped data carries it |
| `0x20`       | Prune this node's children from the draw walk. Set only at runtime |
| `0x80000000` | Locks the hidden flag against the clip-start hide lists (`FUN_00472310`, `FUN_00472d70`, `FUN_004726d0`; see [Starting a clip](#starting-a-clip)). A [visibility](#visibility-bit-0x20000) track still sets and clears `0x10` on a locked node: `FUN_00471860` does not test the bit. Set only at runtime |

`0x10` and `0x20` are separate, and it matters. The draw walk tests `0x10` **after** computing the
node's matrix and then carries straight on into the children, so **a hidden node does not hide what
hangs off it**. Only `0x20` prunes a subtree. When the engine loads an item's model
(`FUN_004629d0`) it sets `0x80000010` on every transform-only node, which hides it and locks it
against the hide lists. It also sets `0x20` on any node whose children are all childless
transform-only nodes.

Only a handful of flag words occur in the shipped data. In the version-`0xDD` models the mesh records
carry `0x1` (4,578), `0x401` (331) or `0x2` (5), and the transform-only records carry `0x200` (2,545),
`0x600` (28), `0x208` (27) or `0x300` (6). `wr_tunnel.md2`'s ten meshes carry `0x1` (7) and `0x81` (3).
What `0x1`, `0x100` and `0x400` select is not known.

#### Node names

**Every node carries a name**, mesh or transform-only, through the same 0x54 field. The names live in
one blob of null-terminated ASCII strings packed end to end in node order, and **no header field
points at that blob**. The per-record offset is the only way into it. The engine relocates exactly
four words per 88-byte node record: 0x04, 0x08, 0x0C and 0x54 (engine-confirmed). 0x54 is the
same field it relocates as the mesh name when it walks the 160-byte records.

The name is the only place the file says what a node is *for*. The lookup table below gives a node a
number and a capability flag, but never a meaning. Transform-only nodes carry names such as
`sound node`, `ant_emitter`, `1stperson`, `camera`, `entrance` and `destroy`. A park's gate marks
where its sound belongs with a node called `sound node`, and the Space lobby island's antenna
carries `ant_emitter`.

Across the game's 839 static models, 7,520 of the 7,530 node records resolve 0x54 to a terminated
ASCII string, and the index word at 0x50 equals the record's own position in 7,521 of them. Every
exception is in `wr_tunnel.md2`. Six names are legitimately **empty**, so a zero-length name is not
a parse failure. The longest name is 25 characters (`StackedTrackOutgoingDummy`), none contains a
non-ASCII byte, and five end in a trailing space. So compare names trimmed and without regard to
case.

> **One model will defeat a trusting reader.** `wr_tunnel.md2`, in `levels\jungle\rides\wateride.wad`,
> is the version-`0xCF` mesh the engine refuses to load. Read with the `0xDD` layout, it has
> `0x42 == 0x44 == 10` with its mesh and node tables at the *same* offset. Its 0x50/0x54 words read
> as float bit patterns rather than an index and a pointer: one name offset is 0 and another is
> `0x3F800000`. Its node-lookup pointer at 0x7C reads `0xC21FF929`. Check the version first, or
> bounds-check the record and the name offset instead of trusting them. It is the only file in the
> archives that needs it.

`Jun_isle.MD2` shows why the tree matters. Its three palm trees hang off two dummy nodes plus the
island mesh:

```
 0 Island     parent -            firstChild node[25]
25 l_tree1    parent mesh[0]      firstChild mesh[1]     local (-11.92, 8.99, 8.40)
    1 Box77   parent node[25]     sibling   mesh[2]      local (  0.00, 0.00, 0.00)
    2 Box78   parent node[25]                            local (  0.00, 0.00, 0.00)
26 l_tree2    parent mesh[0]      firstChild mesh[7]     local (  2.28, 14.88, 8.81)
    7 Box69   parent node[26]     sibling   mesh[8]      local ( -1.15, -1.83, 0.00)
    8 Box70   parent node[26]                            local (  0.00,  3.66, 0.00)
```

Two of the trees store their trunk and leaves at a local `(0, 0, 0)` and `(0, 3.66, 0)`. The
leaves are 3.66 units above *their trunk*, and mean nothing until `l_tree1`/`l_tree2` supply the
position. Read flat, those four meshes land on top of each other at the model origin.

Animation targets index this same node list, which is why a target can legitimately point past
the mesh count: it is naming a transform-only node. Rotating such a node should carry its whole
subtree.

> **Don't decompose these transforms into translation/rotation/scale.** Some nodes are *sheared*:
> their axes are not perpendicular, and a TRS cannot represent that, so the shear is silently
> dropped. How many there are depends entirely on how square you insist they be, so each count
> below comes with its test. "Out of square" is the largest `|cos|` between any two of a node's
> three basis axes. Of the game's 7,530 node transforms, **172 are more than 1e-4 out of square**
> and 120 are more than 0.01. **45 are skewed far enough that .NET's `Matrix4x4.Decompose` gives
> up on them outright**, the mildest of those being 0.11 out of square. `Jun_isle`'s tallest palm
> trunk is one of the 45: its axes are 0.43 out of square, and decomposing skewed it more than 5
> units out of place, into the dinosaur that stands next to it. Keep the 4x4 and multiply it.
>
> Ten further records have a *collapsed* axis rather than a sheared one: a basis vector of zero
> length. All ten are `wr_tunnel.md2`'s nodes 0-9, the older-version model noted above. They are
> degenerate rather than skewed, and worth excluding from any shear count.

**The root's own transform** (the node header `0x78` names) is not always identity. Of the 541 readable static
meshes under `data/levels` (`wr_tunnel.md2` is truncated), it is identity in 305, a rigid turn or offset in 236, and
scaled in none; Lost Kingdom's Jungle Spray and Laughing Hyenas stand theirs turned 90° about y, the Jungle Spray's
also offset (−0.0735, 0, 0.0959). The engine keeps the root's turn when it places a thing and replaces its offset with
where the thing stands (OpenTPW's `docs/exe/ride-operation.md`, "How long a leg lasts, and where its ends are").

### Node lookup ids

Some nodes can also be found by number. The ushort at 0x48 is a record count, the uint at 0x7C the
offset of a table of that many 20-byte records, and the ushort at 0x46 the node the first record
belongs to. Record `r` names node `0x46 + r`.

| Offset | Size     | Description                                                              |
| ------ | -------- | -------------------------------------------------------------------------- |
| 0x00   | 4 bytes  | Flags                                                                      |
| 0x04   | 4 bytes  | Id                                                                         |
| 0x08   | 4 bytes  | Unknown. Non-zero in 128 of the game's 2,452 records                       |
| 0x0C   | 4 bytes  | Pointer (engine-confirmed) to the record's **face anchor**, below. Set on exactly the 248 records whose flag word carries `0x40`, zero on the rest |
| 0x10   | 4 bytes  | Pointer (engine-confirmed). Zero in every record                           |

The face anchor places the node on a face of its parent mesh (engine-confirmed):

| Offset | Size    | Description |
| ------ | ------- | ----------- |
| 0x00   | 2 bytes | The face, an index into the parent mesh's faces |
| 0x02   | 2 bytes | How far back from this record a turn is kept, which the engine applies with flag `0x40000`; not decoded further |
| 0x04   | 4 bytes | `u`, a float: the lerp from the face's first corner toward its second |
| 0x08   | 4 bytes | `v`, a float: the lerp from that point toward the third corner |
| 0x0C   | 4 bytes | A float: how far the node stands off the face, along the face's normal |

345 of the 838 version-`0xDD` models in the archives carry a table. Every record stays inside its
model's node count, and ids run from 0 to 99.

The engine's lookup (engine-confirmed) walks the table for a record whose id matches and whose
flag word shares a bit with a mask the caller passes, and returns the first such record's index (how it treats an
unknown mask is OpenTPW's `docs/exe/audio.md`, "Node lookup is by id AND a capability flag"). The caller
adds `0x46` to get the node. The flag words vary widely across the game (`0xB1`, `0x111` and
`0x811` are the most common), so the table is a general way of naming nodes. The masks callers pass
are `0x200` for a sound emitter, `0x100` for a particle emitter, `0x400` for a costume piece, `0x800`
for walk nodes, `0x80` for heads and `0x20000` for lights (the light instructions). `0x211` marks the gates' `sound node`, and `0x111` most
particle emitters. The costume code asks for bit `0x400`, and dresses a character by setting and
clearing a node's hidden flag, bit `0x10` of the node's own flag word.

The advisor (`Advisor.MD2` in `data\global\advisor.wad`) shows it best. Every one of his 14
records has `0x400`, and the ids are costume pieces. His antennae are 19 and 20, which most
costumes hide. His right hand is 21, which costume 14 (his spatula) hides. His hats and bow tie
are 5 to 13. None of those nodes is hidden in the file (the hidden bit is never set in any node the
game ships), so which pieces show is entirely the code's decision.

#### Which records have a position

A ride script asks for a node's position by id and flag (a walk node `0x800`, a head `0x80`, a sound
`0x200`, a particle emitter `0x100`) and gets that node's position as last posed, in the world
(engine-confirmed). Only a record whose flags carry `0x10` or `0x20` has a position at all. A node
with children is posed whenever its model is. A node with none, which every walk and head node in the
game is, is posed only when:

- its record's flags meet `0x580f00`, which takes in `0x100`, `0x200`, `0x400`, `0x800`, `0x80000`,
  `0x100000` and `0x400000` but not `0x80`;
- its item's description sets `Info.DoHeadProcessing` (see [the item descriptions](/formats/sam/));
- the ride view is on it (a `0x1000` record);
- or something is attached to it.

Otherwise it keeps where it was last posed, the origin if it never was: the Jungle Spray's `camera`
record (flags `0x1031`) read the origin in the running game, with no ride view on it. Every walk and
head record the shipped scripts name is posed (OpenTPW's `docs/exe/ride-operation.md`, "How long a
leg lasts, and where its ends are").

The pointer at 0x0C is read only for a record whose flags meet `0x40040`, and only while its parent
mesh carries runtime flag `0x200000`, which a vertex-morph clip sets on the mesh it animates: the
position is then taken from a face of that mesh, as the morph has posed it. The point is the face's
first corner lerped toward its second by `u`, that lerped toward the third by `v`, then the face's
normal (from the table at mesh 0x64, which the morph routine does not write) times the offset. So a rider's head
on Mumbo's tentacle moves with the tentacle. At rest the rule lands on the node's own place: measured
over all 248 anchors in the game's 2,073 readable models, 247 within 0.05 units; the Squark's `Head04`
is 0.26 off.

#### Open questions

- The word at 0x08, the turn the face anchor steps back to, and what the flag bits this page does
  not name select.

### Paths

A model can carry one or more **paths**: the routes that vehicles and ride cars follow. The ushort at
0x40 is the number of paths, and the uint at **0xAC** the offset of an array of that many 16-byte
records, or 0 when the model has none (engine-confirmed). Each record is four little-endian uints:

| Offset | Type   | Meaning |
| ------ | ------ | ------- |
| +0x00  | `uint` | Type. See below |
| +0x04  | `uint` | Number of points |
| +0x08  | `uint` | Offset of the points (engine-confirmed pointer) |
| +0x0C  | `uint` | Always 0. The engine relocates it as a pointer when it is set |

The points are 12-byte `XYZ` triples of `float`. They are in the space of the path's own node, not
the model's: a point becomes the local position of the node that follows the path, and that node
hangs directly off the path node (see [Path progress](#path-progress-bit-0x200)). No model with a path
sets header bit `0x4`, which would add the point to the node's position instead.

The array is not self-delimiting, so a reader must take the count from 0x40. The bytes after the
haunted house's fourth record are vertex floats, and read as a fifth record they give a point count
of 1,112,011,916. In animation files 0xAC is always 0. 82 of the clips of path-carrying models repeat
the model's count at 0x40 anyway, so 0x40 in an animation file says nothing.

#### Path nodes

Each path is named by exactly one **path node**: a transform-only node with flag bit `0x8`, whose
ushort at +0x52 is the index of its path. There are 27 in the game, one per path, and every one has
the flag word `0x208`. The haunted house's four are `Kart_path01` to `Kart_path04` and hold `0, 1, 2,
3`. In every other model the one path node holds 0, as does every other node of a version-`0xDD`
model. +0x52 cannot tell
"path 0" from "no path" on its own; the `0x8` bit is what marks a path node.

#### Type

The type word's bits choose the curve, and the engine picks its sampler on them (engine-confirmed,
`0x00471f77`-`0x00471fc7` and on to `0x00472037`):

| Bit   | Meaning |
| ----- | ------- |
| `0x2` | Cubic Bézier: segments of four points, three new points per segment |
| `0x8` | Straight lines between the points |
| `0x1` | **Closed**: the last segment runs back to point 0 |

With neither `0x2` nor `0x8` the engine has a Catmull-Rom sampler, but no path uses it. The observed
types are `2` (open Bézier), `3` (closed Bézier) and `8` (open straight lines).

The engine counts a Bézier path's segments as `points / 3`, and a straight path's as `points`, less
one unless the path is closed. The Bézier sampler wraps every point index modulo the point count.
The straight-line sampler does not wrap, and no straight path is closed. So a closed Bézier path's
point count is a multiple of three, not `3n + 1`, and its final segment ends on point 0: the
haunted house's 48 points make 16 segments. An open Bézier path stores a multiple of three as well,
but the engine never samples its last two points. The bus's 45 points give 14 segments, over points
0 to 42. On the bus, the ferry and the Wonder Land go-karts those two unused points repeat point
`n-3` and then point 0, which would be a straight closing segment. The seaplane's are ordinary
points.

#### Where the paths are

Measured across all of the game's models, twenty-four carry a path table:

| Model | Path node | Paths | Type | Points |
|---|---|---|---|---|
| `Bus.MD2` (all four themes) | `Line02` | 1 | 2 | 45 |
| `FERRY.MD2` (all four themes) | `FERRYSPLINE` | 1 | 2 | 33 |
| `Seaplane.MD2` (all four themes) | `Flightpath` | 1 | 2 | 33 |
| `gokarts.MD2` (Wonder Land) | `beespline01` | 1 | 2 | 102 |
| `gokarts.md2` (Halloween) | `h_line` | 1 | 3 | 51 |
| `haunt.MD2` | `Kart_path01` - `Kart_path04` | 4 | 3 | 48 each |
| `bigapple.MD2` | `Line01` | 1 | 3 | 39 |
| `ratrace.MD2` (Halloween) | `r_spline` | 1 | 3 | 12 |
| `wateride.MD2` (Space) | `sw_spline1` | 1 | 3 | 12 |
| `slide.MD2` | `sb_spline` | 1 | 8 | 12 |
| `Advisor_HAL.MD2` | `h_line` | 1 | 3 | 51 |
| `Jun_isle`, `Fan_isle`, `Hal_isle`, `Spa_isle` | `Spline path` | 1 | 3 | 12 |

The haunted house's four paths share an identical bounding box. They are four carts on one circuit,
offset in phase rather than following different routes: the four paths start a quarter of the way
round from one another.

The four lobby islands' path spans X -29.81 to 29.81 and Z -39.75 to 39.75, with Y exactly 0 at
every point. It is a flat loop in its path node's space.

Eight of the 27 paths are followed by no clip in the game: `bigapple`, `ratrace`, the Space
`wateride`, the Wonder Land go-karts, and the four lobby islands.

### Heightfield

The pointer at header 0x6C is set in exactly five files: the four parks' `terrain.wad` `base.MD2` and
the lobby's `terrain\Base.MD2`. It points at the heightfield block the park's ground is built from.
The block has a 0x30-byte header:

| Offset | Size    | Description |
| ------ | ------- | ----------- |
| +0x04  | 2 bytes | Vertex count |
| +0x06  | 2 bytes | Cell count |
| +0x10  | 4 bytes | Cell size X, a float (10.0 in all five) |
| +0x14  | 4 bytes | Cell size Y, a float (10.0 in all five) |
| +0x18  | 4 bytes | Cells across |
| +0x1C  | 4 bytes | Cells down |
| +0x20  | 4 bytes | Low end of the heights' authored range, a float: a whole number within 1.0 of the lowest height (jungle -10.0 against -10.0, fantasy -9.0 against -10.0, hallow 0.0, space -10.0, the lobby 0.0) |
| +0x24  | 4 bytes | High end of that range: a whole number within 1.0 of the highest height (jungle 60.0 against 60.16, fantasy 20.0, hallow 21.0, space 28.0, the lobby 0.0). The engine lights the ground with 1 / (`+0x24` - `+0x20`), or 1/16 when the two are equal (`FUN_0056e3f0`) |
| +0x28  | 4 bytes | Offset of the heights: vertex-count floats, rows of `cells across + 1` (engine-confirmed pointer) |
| +0x2C  | 4 bytes | Offset of the cell records: cell-count 4-byte records, indexed `y * across + x` (engine-confirmed pointer) |

In all five, the vertex count is `(across + 1) * (down + 1)` and the cell count `across * down`. Each
park is 96 by 85 cells (8,342 heights, 8,160 cell records), and the lobby is 111 by 110. Each of
these files also carries a mesh named `heightfield` with no vertices and no faces, whose flag word is
`0x2`.

### Textures

The frame table (offset at 0x50) is an array of *frame count* (0x36) 8-byte entries, immediately
followed by that many 20-byte null-padded ASCII texture filenames: one string per entry, in the
same order. A material's `FrameOffset` (see **Materials** below) is a byte offset that lands
inside this 8-byte-entry array. Dividing `(FrameOffset - frameTableOffset)` by 8 gives that
material's index into it.

The frame data table (offset at 0x54) is a second, parallel array of *frame count* 16-byte
records, indexed the same way:

| Offset  | Size    | Description                                                             |
| ------- | ------- | ------------------------------------------------------------------------ |
| 0x00    | 4 bytes | Flags. Bit 0 is tested by the engine (engine-confirmed)                  |
| 0x04    | 4 bytes | Zeroed at load time, so runtime scratch rather than file data (engine-confirmed) |
| 0x08    | 2 bytes | Padding                                                                  |
| 0x0A    | 2 bytes | Set to 1 by the engine under one load option (engine-confirmed)          |
| 0x0C    | 4 bytes | Offset of this frame's 20-byte texture filename string (engine-confirmed pointer) |

In practice this points at the very same filename strings that follow the frame table. The
two tables describe the same textures, reached two different ways. A material resolves its
texture name via the frame data table (`FrameNameOff`), not by reading the frame table's strings
directly.

A material's `FrameOffset` of exactly 0 is a sentinel meaning "no texture", not a real offset.
Real offsets always start at the frame table's own offset.

A texture name that resolves to nothing is not an error. The engine searches two directories for
each name and logs `Could not load texture '%s' from '%s' or '%s'`. It then assigns the frame the
**first entry of the texture cache**, which the cache fills at startup with
`Data\Generic\defaulttexture\NotFound.tga`, a 64x64 brown noise tile (engine-confirmed). The
shipped data does rely on this. `Hal_isle.MD2`'s sign mesh names a `signgrab` texture that exists
nowhere in the game, and its frame draws as that brown tile rather than as anything the artists
authored.

The 8-byte frame table entries also begin with a **flags byte**. The engine tests bits `0x40`
and `0x80` of byte 0 and propagates them into the model-wide flags at header 0x30
(engine-confirmed). When it loads an item's model (`FUN_004629d0`), it sets bit `0x1000` of header
0x30 if either bit is set.

That byte is the same one a material reads as its flags word. A material's `FrameOffset` points
at this entry, so the two are the same bytes reached from either direction. See
**Material flags** below for what is known of the individual bits.

#### Open questions

- The remaining 7 bytes of each frame table entry, and the frame data table's own flags field at
  0x00 beyond its bit 0 being tested.

### Mesh table

The mesh table (offset at 0x70) is an array of *mesh count* (0x44) fixed-size 160-byte records,
one per mesh:

| Offset | Size     | Description                                                              |
| ------ | -------- | ------------------------------------------------------------------------- |
| 0x00   | 4 bytes  | **Flags**. See [The flag word](#the-flag-word)                            |
| 0x04   | 12 bytes | Parent, next sibling and first child links. See **Node hierarchy** (engine-confirmed pointers) |
| 0x10   | 64 bytes | 4x4 transform matrix (16 floats, row-major): this mesh's placement       |
| 0x50   | 2 bytes  | This record's **own index** in the combined node list (0-based)            |
| 0x52   | 2 bytes  | Path index. Zero on every mesh of a version-`0xDD` model, since only transform-only nodes are path nodes |
| 0x54   | 4 bytes  | Offset of this mesh's null-terminated ASCII name. See **Node hierarchy**  |
| 0x58   | 2 bytes  | Vertex count                                                               |
| 0x5A   | 2 bytes  | Material count                                                             |
| 0x5C   | 2 bytes  | Face count                                                                 |
| 0x5E   | 2 bytes  | Vertex order length (see **Vertex order**, below)                         |
| 0x60   | 4 bytes  | Vertex data offset                                                         |
| 0x64   | 4 bytes  | Face normals offset: three floats each, indexed by each face's first word (see **Faces**; engine-confirmed) |
| 0x68   | 4 bytes  | UV data offset                                                             |
| 0x6C   | 4 bytes  | Material table offset                                                     |
| 0x70   | 4 bytes  | Face data offset                                                           |
| 0x74   | 4 bytes  | Unknown. Not a pointer: the engine does not relocate it                    |
| 0x78   | 12 bytes | Bounding box minimum (3 floats). See **Vertex data**                      |
| 0x84   | 12 bytes | Bounding box maximum (3 floats). See **Vertex data**                      |
| 0x90   | 4 bytes  | Unknown                                                                    |
| 0x94   | 4 bytes  | Vertex order table offset (see **Vertex order**, below)                   |
| 0x98   | 8 bytes  | Unknown                                                                    |

This mesh's bounding box (0x78/0x84) belongs to this mesh alone. It is **not** the box animation files
quantise vertex positions into: each morph track carries a box of its own. See **Vertex morph**
below.

#### Open questions

- The remaining unknown fields (0x74, 0x90, 0x98) and the pointer at 0x64. None has been narrowed
  down beyond "not used by anything this project's renderer needs".
- What mesh flag bits `0x1` and `0x400` select.

### Vertex data

Vertex positions are **not** stored as one `(x, y, z)` triple per vertex. They're grouped into
batches of up to 4 vertices. Within a batch they are stored axis-major: four X floats, then four Y
floats, then four Z floats. The vertex count is padded up to the next multiple of 4 for this
purpose (trailing padding vertices are read but discarded).

So for vertex count 5, the layout at the vertex data offset is:

```
X0 X1 X2 X3   X4 X? X? X?      (two batches of 4 X floats each - second batch mostly padding)
Y0 Y1 Y2 Y3   Y4 Y? Y? Y?
Z0 Z1 Z2 Z3   Z4 Z? Z? Z?
```

This is the *raw* vertex list, indexed by source vertex index. It is not the order the mesh is
actually drawn in. See **Vertex order**, next.

UV coordinates (at the UV data offset) use the same batches-of-4, axis-major layout. The count that
matters here is the **vertex order length** (0x5E in the mesh record), not the raw vertex count.
UVs are stored per final, reordered vertex slot, not per source vertex.

### Vertex order

The vertex order table (offset at 0x94, length 0x5E) is an array of *vertex order length*
`ushort`s. Reading it produces the mesh's real, final vertex list: entry `i` of this table is
the source vertex index (into the raw vertex data above) that should become vertex `i` of the
drawn mesh.

This reordering exists to group vertices contiguously by material. Each material record (see
below) gives a `StartIndex`/`EndIndex` range, and a final vertex `i` uses whichever material's
range contains `i`. So the vertex order table isn't just a remap. It makes each material's vertices
sit in one unbroken run, which is what lets a material be described by a single index range
instead of a vertex list of its own.

Face indices (see **Faces**, below) refer to this final, reordered vertex list, not the raw
vertex data.

#### Open questions

- Whether the vertex order table is ever used for anything beyond material grouping (e.g.
  triangle strips or fans). Not observed in the game's data so far.

### Faces

The face table (offset given in the mesh record) is *face count* records of 8 bytes each:

| Size    | Description                                          |
| ------- | ----------------------------------------------------- |
| 2 bytes | Low 15 bits: the face's normal, an index into the face normals at mesh 0x64. Top bit unknown |
| 2 bytes | Vertex index A (into the reordered vertex list)        |
| 2 bytes | Vertex index B                                         |
| 2 bytes | Vertex index C                                         |

Each record is one triangle. Winding order needs to be reversed from how it's stored to render
correctly with a standard right-handed culling convention.

#### Open questions

- The top bit of each face record's leading `ushort`.

### Materials

**How many.** Measured over every readable `.md2` (2,117 of them, 4,914 meshes; `wr_tunnel.md2` in jungle's
`wateride.wad` does not parse): a mesh names **at most 31** materials (the jungle Coaster3 preview's `GROUND`), and
49 name more than 16, among them the jungle Hot Pot's floor `jbb_floor` with 25, whose faces use materials 16 to 24.
A face's material index is the vertex's texture index, so a reader has to keep every one.

The material table (offset given in the mesh record, *material count* entries) uses 16-byte
records:

| Size    | Description                                                          |
| ------- | ---------------------------------------------------------------------- |
| 4 bytes | Frame offset. See **Textures**, above (0 = no texture)                 |
| 2 bytes | Unknown                                                                 |
| 2 bytes | Unknown                                                                 |
| 2 bytes | Start index: first reordered vertex index using this material          |
| 2 bytes | End index: last reordered vertex index using this material (inclusive) |
| 4 bytes | Unknown                                                                 |

A material with a non-zero frame offset also has a 4-byte flags word, read directly from that
frame offset in the file. The frame offset does double duty: it locates both the frame table entry
for the texture name, and a flags value sitting at that same byte offset.

### Material flags

Because that word is read *at* the frame offset, it is the first four bytes of the 8-byte frame
table entry described under **Textures**. Its low byte is the same "flags byte" whose `0x40`
and `0x80` the engine propagates into the model-wide flags at header 0x30.

That placement means **the word belongs to the texture as used by this model**, not to the
material alone. Two materials in the same model naming the same texture necessarily share it.
Two *different* models naming the same texture need not, and 62 of the 2,990 textures referenced
across the game are flagged one way by one model and another way by another.

Measured across the 12,951 material uses in the 838 material-bearing mesh files in the archives:

| Bit    | Uses  | What is known                                                                   |
| ------ | ----- | --------------------------------------------------------------------------------- |
| `0x01` | all   | Set on every material in the game; carries no information                        |
| `0x02` | 2,621 | Marks a material meant to be drawn see-through. See below                         |
| `0x10` | 3,849 | Unknown. Independent of `0x20`                                                    |
| `0x20` | 3,886 | Unknown. Independent of `0x10`                                                    |
| `0x40` | 159   | Propagated into the model-wide flags at header 0x30 (engine-confirmed)            |
| `0x80` | 0     | Also propagated into header 0x30 (engine-confirmed), but no shipped material sets it |

The word never exceeds `0x73` anywhere in the game. Only the low byte is ever used, and just 15
distinct values occur across all 12,951 uses. `0x10` and `0x20` are **not** a pair: they occur
alone 1,209 and 1,246 times respectively, and together 2,640 times.

#### Bit 0x02 - drawn see-through

This is the only bit whose meaning is established, and it is established from the data rather
than from the engine. Nothing in the decompile reads it back where a render state is chosen.

It correlates with the texture declaring an alpha channel, but **only in one direction**. Of the
12,773 material uses whose texture could be resolved:

|                          | texture has alpha | texture has none |
| ------------------------ | ----------------- | ---------------- |
| **bit set**              | 2,247             | 335              |
| **bit clear**            | 1,780             | 8,411            |

So when the bit is set the texture has an alpha channel 87.0% of the time, but when a texture has
an alpha channel the bit is set only 55.8% of the time. It is not a restatement of the texture's
format. It reads as an authoring decision, "draw this one see-through", and a great deal of the
game's 32-bit art is deliberately drawn opaque. The largest groups carrying alpha without the bit
are ground and path tiles: `m_grass1`, `m_grass2`, `jfl_cnr2`, `jpa_que1`.

Compare against the texture header's **alpha-channel byte and not its bit depth**. The two are
different fields and they disagree: `sen_ant1` is stored 32-bit but declares no alpha channel.

> The percentages above carry a small caveat: 100 of the 3,385 distinct `.wct` names in the game
> appear in more than one WAD with different alpha-channel bytes, so a name-keyed lookup cannot
> be exact for those.

In the lobby (the one scene OpenTPW currently renders) the correlation is far tighter. Of the
232 material uses whose texture ships in `lobby.wad`, 224 agree. Six of the eight that disagree
are textures carrying alpha and drawn opaque anyway. The other two set the bit over a texture
with no alpha channel at all, one of them the lobby's own sea surface. Treating the bit as "draw
see-through" is what makes the islands' shoreline ripple rings and the Space island's antenna
cone render correctly.

What the bit does **not** distinguish is cut-out art from genuinely blended art. Most of what
carries it in the lobby is cut-out foliage (palm fronds, grass blades, bushes, the bats and
butterflies), which wants its alpha *tested*. Only the ripple rings, the sea and the antenna cone
are true gradients. A renderer acting on this bit has to serve both.

#### Open questions

- The two unidentified 2-byte fields in the material record, and the 4-byte field at its end.
- What `0x10` and `0x20` select, and what `0x40` means beyond being propagated to header 0x30.
- Whether anything in the format distinguishes an alpha-tested cut-out material from a blended
  one, or whether the original engine drew both the same way.

### Normals

The game's model lighter (`FUN_00574660`) takes one normal per vertex-order entry from the table at
mesh 0x64, twelve bytes each: the same table a face's first word indexes. How many entries each
file holds has not been counted across the models. OpenTPW does not read them for lighting yet: it
accumulates each triangle's face normal onto its three vertices and re-normalizes.

## Animation

An animation file has a mesh table offset of **exactly 0** at 0x70, the same field that's a
real pointer for static meshes. In place of geometry it carries a list of **tracks**. Each track
poses one node of the base model (identified by the shared filename prefix) over a range of
authoring frames.

A track is not one kind of animation. It is a bundle of independent **channels**. A fountain's
water mesh morphs its vertices, scrolls its texture and carries a timing scalar, all on the same
track. Each channel kind owns its own slot in the track descriptor. That is what makes the
format safe to read incrementally: a channel you don't understand costs nothing, because its
data lives in a slot you simply don't read.

Of the game's 1,279 animation files, 1,189 have tracks, and every one of those carries at least one
channel documented below:

| What a file's tracks carry                                                      | Files |
| ------------------------------------------------------------------------------- | ----- |
| rotation, vertex morph or UV animation                                          | 1,151 |
| none of those, but position, visibility, scale or path progress (6 carry only path progress) | 30 |
| only raw vertex frames (bit `0x4000`)                                           | 8     |
| a track count of zero                                                           | 89    |
| no animation block (`wr_tunnelm.md2`, the version-`0xCF` file)                  | 1     |

### Locating the tracks

The uint at 0x98 points at a 72-byte **animation block**. That pointer is valid in 1,278 of the
1,279 animation files.

| Offset (from block start) | Size    | Description                                       |
| -------------------------- | ------- | --------------------------------------------------- |
| 0x04                        | 4 bytes | First frame of the animation, an integer. 0 in every block in the game |
| 0x08                        | 4 bytes | Last frame of the animation, an integer             |
| 0x10                        | 2 bytes | Sum of every track's rotation keyframe count (a cross-check, not needed to parse) |
| 0x12                        | 2 bytes | Track count                                          |
| 0x1A                        | 2 bytes | Hide-list entry count. See [The hide list](#the-hide-list) |
| 0x2C                        | 4 bytes | Track table offset                                   |
| 0x38                        | 4 bytes | Hide-list offset (engine-confirmed pointer)          |

The engine relocates five more pointers in the block: 0x20, a table of 16-byte records counted by
the ushort at 0x0C, with a pointer at +0x08 and +0x0C of each; 0x24; 0x28; 0x30, a table of 8-byte
records counted by the ushort at 0x18, with a pointer at +0x04 of each; and 0x3C. What they hold is
not decoded.

The track table offset is trustworthy on its own terms. That matters because there's nothing
else to check it against. The table is exactly `trackCount * 64` bytes and **ends exactly where
the animation block begins**: `tableOffset + trackCount * 64 == blockOffset`. **That identity
holds for all 1,189 files with tracks**, so no file with tracks is rejected as unreadable. A reader
should apply the identity and treat a file that fails it as carrying no readable animation, rather
than reading a table that isn't one. A mislocated table
produces confidently wrong animation, not an obvious failure.

### Track descriptors

Each track is a 64-byte descriptor:

| Offset (from descriptor start) | Size    | Description                                          |
| -------------------------------- | ------- | ------------------------------------------------------ |
| 0x00                               | 4 bytes | This track's own index (0-based, sequential; a validity check) |
| 0x04                               | 4 bytes | Channel flags. See below                               |
| 0x0C                               | 4 bytes | A frame value, close to but not always the track's last keyframe |
| 0x10                               | 2 bytes | Rotation keyframe count (channel `0x8` only)          |
| 0x12                               | 2 bytes | Scale keyframe count (channel `0x80` only; zero on every other track) |
| 0x14                               | 2 bytes | **Target node**. See **Target resolution** below       |
| 0x16                               | 2 bytes | Entry count for channel `0x20000`; otherwise unknown, and **not** part of the target |
| 0x18                               | 4 bytes | Position record (channel `0x1`)                        |
| 0x1C                               | 4 bytes | Rotation keyframes (channel `0x8`)                     |
| 0x20                               | 4 bytes | Scale keyframes (channel `0x80`)                       |
| 0x24                               | 4 bytes | Path progress record (channel `0x200`)                 |
| 0x28                               | 4 bytes | Vertex morph descriptor (channel `0x1000`)             |
| 0x2C                               | 4 bytes | UV animation descriptor (channel `0x10000`)            |
| 0x30                               | 4 bytes | Visibility entries (channel `0x20000`)                 |
| 0x34                               | 4 bytes | Easing curve table (rotation only, engine-confirmed)   |

The flag word at 0x04 says which channels the track carries, and every channel bit owns exactly one
slot. Counted across every track in every animation file in the game:

| Flag bit    | Owns slot                     | Channel       | Tracks | Decoded? |
| ----------- | ----------------------------- | ------------- | ------ | -------- |
| `0x00008`   | count at +0x10, data at +0x1C | Rotation      | 3039   | Yes      |
| `0x01000`   | +0x28                         | Vertex morph  | 1766   | Yes      |
| `0x10000`   | +0x2C                         | UV animation  | 690    | Yes      |
| `0x20000`   | count at +0x16, data at +0x30 | Visibility    | 2536   | Yes      |
| `0x00001`   | +0x18                         | Position      | 1216   | Yes      |
| `0x80`+`0x100` | count at +0x12, data at +0x20 | Scale      | 644    | Yes      |
| `0x00200`   | +0x24                         | Path progress | 71     | Yes      |

Every one of those correspondences is **exact**. Across all 1,279 files, not one track sets a
bit without filling its slot or fills a slot without setting the bit.

Other bits own no slot. `0x4000` changes what the `+0x28` slot points at (below). `0x400` and `0x800`
belong to path progress (see [Path progress](#path-progress-bit-0x200)).
`0x10` (2,077 tracks) appears only alongside rotation. `0x20` (25), `0x40` (937), `0x2000` (342) and
`0x8000` (29) are undecoded.

There is an eighth pointer, at **+0x34, that no flag bit owns**. It is set on 1768 tracks and
every one of them is a rotation track (out of 3039), so it is an optional extra *for rotation*
rather than a channel in its own right. **It is the easing curve table.** See
[The easing curve](#the-easing-curve) under Rotation below.

> **This page used to call that an observation rather than a decode**, on the grounds that "the
> records are not a fixed length and about a third are not monotonic". The first half was the
> mistake, and it was what kept the field undecoded: a record is exactly **eight bytes**, indexed
> by an id carried on each rotation keyframe. The ramp quoted here
> (`32, 66, 105, 141, 176, 208, 233, 249` in `Advisorm1`) is that file's **curve 0**, which the
> old reading happened to land on at the right stride. The second half is true and is not a
> problem: about 22% of curve records really do fall as well as rise, deliberately.

> **Bit `0x4000` is a modifier, not a channel.** It makes the `+0x28` slot point at a different
> structure, and the engine branches on it *before* reading any morph table. Thirty tracks in
> the game set it, always alongside `0x1000`. Reading those as vertex morph follows offsets into
> the wrong structure, so a reader must exclude them from morph. See
> [Raw vertex frames](#raw-vertex-frames-bit-0x4000).

> **The target at 0x14 is a ushort, not a uint.** 0x16 holds an unrelated value and is nonzero
> on 595 of the game's rotation tracks, so reading 32 bits there produces a garbage node index.
> That index is large enough that a range check discards a track that was perfectly good.

### Target resolution

The target indexes the base model's **node** list: all `0x42` nodes, not only the first `0x44` that
are meshes, resolved by the rule under [Node hierarchy](#node-hierarchy) (engine-confirmed). For
models with no extra hierarchy that list is simply the mesh list. `Jun_gateM1.MD2` has two rotation
tracks targeting nodes 0 and 1, which are `Jun_gate.MD2`'s two door meshes. `Advisor.MD2` has 29
nodes and 25 meshes, which is why its animations reach node 28.

Take every clip whose base model sits in the same archive. **6,537 of those clips' 6,541 targets fall
inside the node count**, against only 5,465 inside the mesh count. So a target above the mesh count
is a real node the model simply has no geometry for, not a misread. A reader with no representation
for non-mesh nodes should skip those tracks. The four targets past the node count are all in
`Pmegacostm.MD2`, a file the engine never opens (see
[Which animation file a model takes](#which-animation-file-a-model-takes)).

For morph tracks there is a reliable second test, described below: the channel count has to match the
target mesh.

### Rotation (bit 0x8)

Turns a whole node about its own origin. The keyframe count is the **ushort** at descriptor
0x10 and the data offset the uint at 0x1C. Each keyframe is 20 bytes:

| Size    | Description                                              |
| ------- | ----------------------------------------------------------|
| 2 bytes | Frame index                                                |
| 2 bytes | **Easing curve id**. `0xFFFF` means none; see [The easing curve](#the-easing-curve) |
| 4 bytes | Quaternion X                                               |
| 4 bytes | Quaternion Y                                               |
| 4 bytes | Quaternion Z                                               |
| 4 bytes | Quaternion W                                               |

Frame indices strictly ascend within a track, and every one of the 3039 rotation tracks in the
game decodes to a unit quaternion (within 0.01) at every keyframe. A reader should validate that
pair of properties before trusting a track.

Model space is Y-up, so a quarter turn about Y is a door swinging. `Jun_gateM1` takes the gate's
two doors from identity to a quarter turn, and `Jun_gateM2` is exactly the inverse: open, then
shut.

#### A rotation key is the orientation inside the parent

A rotation key is the orientation the node should hold, **not** a turn to add to the one it was
authored with. It replaces the node's authored rotation (`FUN_0046fbb0`). So the authored rotation
has to come back out before the keyed one goes in.

The exception is a model whose header word at 0x30 has bit `0x4` set. There the engine composes the
key with the node's authored rotation instead of replacing it (`FUN_00470d10`, chosen by the test at
`0x00471d53`). Only the 25 coaster parts listed under [Shared header](#shared-header) set it, and
their clips carry 51 rotation tracks.

**Which authored rotation is the trap.** It is the node's own transform, before its parents'
transforms are applied. It is not where the node ends up in the model. The two are the same only
when every ancestor of the node carries no rotation. **Every gate in the game is exactly that**, so
gates are the one family of models that cannot tell the difference: all 28 gate rotation tracks, in
the parks and the lobby, see the same matrix both ways. Across the game, **1,393 of the 3,035
rotation tracks whose target resolves** aim at a node where the two differ. 51 of the 3,035, and 3 of
the 1,393, are on the coaster parts with bit `0x4`.

The Jungle Spray, which does not set the bit, settles it. Its three animal heads hang off a bench that is itself turned a quarter
turn. So each head is square *within the bench* while standing at a quarter turn in the model, and
every clip keys them square. The proof is the construction clip, which by definition ends on the
finished object. It ends with the fence keyed at 90°, the guns at 180°, one puddle at 245° and the
bench square. Those are exactly those meshes' local orientations, and every one of them disagrees
with the mesh's world orientation.

#### The easing curve

The second ushort of a keyframe is **not a flag word**. It is an index into a table of curves at
track descriptor **+0x34**, and `0xFFFF` is the only sentinel. `0` is *curve number nought*, the
commonest id in the game, which is exactly why the field looks like a flag that is only ever set
or clear.

Where a key names a curve, the engine does not blend evenly between that key and the next. It
bends the fraction first, at `0x00471c83`-`0x00471d32`, immediately before the rotation sampler
at `0x00474490`.

**A curve is eight bytes**, and those bytes are the *inner* points of a ramp whose ends are
implied: nought before the first byte, one after the last. So the ramp is ten points and **nine
straight segments**:

```
0 -> curve[0] -> curve[1] -> ... -> curve[7] -> 1
```

To evaluate it, given the even fraction `t` between the two keys:

1. Multiply `t` by the segment count and **truncate toward zero** to pick a segment `s`.
2. Take that segment's endpoints: `(0, curve[0])` when `s` is 0, `(curve[s-1], curve[s])` for
   `s` of 1 to 7, and `(curve[7], 1)` for `s` of 8 or more.
3. Scale each byte by `1/255` and interpolate across the segment with what is left of the
   truncation.

Both constants are worth quoting exactly:

| Address      | Value                   | Purpose                                      |
| ------------ | ----------------------- | -------------------------------------------- |
| `0x006febe4` | **8.999995231628418**   | Segment count, deliberately just under nine  |
| `0x006febec` | **0.003921568859368563** | Byte scale, exactly `1/255`                 |

> **The segment count is not nine, and the shortfall is deliberate.** Because the engine
> truncates, an exact 9 would send `t == 1` into a tenth segment that has no upper point to
> reach, and the blend would answer `curve[7]/255` instead of 1. That is a jump *backwards* on the
> very last frame. Just under nine keeps `t == 1` inside segment 8, where it lands on 1.

**The id governing a segment is the one on the key being blended out of**: the lower of the two.

Measured over the 1166 clips under `levels/` whose track table validates:

| Measurement                                        | Count             |
| -------------------------------------------------- | ----------------- |
| Clips with at least one eased rotation key          | **457 of 1166**   |
| Rotation tracks carrying a +0x34 table              | 1759 of 3022      |
| Rotation tracks with no table (every key `0xFFFF`)  | 1263              |
| Eased keys whose id equals their own index          | 9358 of 12,428    |
| Eased keys whose id does **not**                    | **3070**          |
| Curve records that are non-monotonic                | 2797 of 12,428    |
| Curve records whose last byte is 255                | 112 of 12,428     |

Three things follow from those numbers. A reader can get each one wrong and still produce
plausible-looking output:

- **The table is indexed by the id, never walked alongside the keys.** Deriving a curve from the
  key's own position is right about four times in five and quietly wrong the rest. Ids reach
  **100**, far past any key count.
- **The last key of a track never names a curve.** All 1759 eased tracks end on `0xFFFF`, with no
  exceptions: there is no segment beginning at the final key.
- **A curve need not climb.** 2797 records dip, so the pose genuinely travels back the way it came
  partway through a blend before going on. That is an author's overshoot written down, and it
  should be reproduced rather than sorted. The implied 1 at the end matters for the same reason:
  only 112 records ever reach 255, so almost every curve is still climbing when it enters the
  ninth segment.

### Vertex morph (bit 0x1000)

Reshapes a mesh vertex by vertex. The slot at descriptor 0x28 points at a morph descriptor:

| Offset  | Size     | Description                                                          |
| ------- | -------- | ---------------------------------------------------------------------- |
| 0x00    | 1 byte   | Flags: `0x01` in 1,338 of the game's 1,736 readable descriptors, `0x03` in the other 398 |
| 0x02    | 2 bytes  | Record count                                                           |
| 0x0C    | 4 bytes  | Record table offset                                                    |
| 0x10    | 4 bytes  | Pointer (engine-confirmed). Zero in every descriptor                   |
| 0x14    | 12 bytes | Centre of this track's quantisation box (3 floats). See **Value decoding** |
| 0x20    | 12 bytes | Step of this track's quantisation box (3 floats). See **Value decoding**   |

Each track has its **own** descriptor and its own channel space, so one animation morphs as many
meshes as it has morph tracks. 752 animation files carry readable morph tracks, 360 of them
more than one: `ratraceM1.MD2` morphs four meshes at once. There are 1766 morph tracks in all,
30 of which set `0x4000` and are not morph descriptors at all.

> Because each morph track is self-contained, a reader that only looks at a fixed header offset
> finds just the first one. Three of `ratrace`'s four morphing meshes have 64 vertices each, so
> matching a track to a mesh by vertex count cannot tell them apart either. Only the target index
> can.

Each record in the table is 20 bytes:

| Size    | Description                                                          |
| ------- | ---------------------------------------------------------------------- |
| 2 bytes | Keyframe count (`a`)                                                    |
| 2 bytes | Channel count (`b`)                                                     |
| 4 bytes | Offset of a `b`-entry array of channel IDs (`ushort` each)              |
| 4 bytes | Offset of an `a`-entry array of ascending frame indices (`ushort` each) |
| 4 bytes | Offset of an `a * b`-entry array of packed values                       |
| 4 bytes | Unknown                                                                 |

The channel ID and frame index arrays are each padded up to a 4-byte boundary. So the gap between
the channel IDs and the frame indices is `2*b` or `2*b + 2` bytes, and the gap between the frame
indices and the values is `2*a` or `2*a + 2`. A reader has to accept either.

The value array is **entry-major**: the value for keyframe `e` of channel slot `k` sits at
`valueOffset + (e * b + k) * 4`. All channels of keyframe 0 come first, then all channels of
keyframe 1; they are not grouped by channel.

A morph track has exactly **one channel per vertex of its target mesh, plus two** trailing
channels that aren't vertices. Every one of the 1,734 morph tracks whose base model sits in the same
archive satisfies that exactly, whichever flag byte its descriptor carries. That makes
`channelCount == targetMesh.vertexCount + 2` a good validity test as well as a description.

The two trailing channels are the **corners of the box the vertices span** at each keyframe,
minimum then maximum, in the same packed form as the vertices. The engine reads them as the
animation's bounding box (engine-confirmed). In all 1,734 tracks, they sit on the corners of the
first keyframe's vertices to within three quantisation steps.

Channels the animation doesn't actually move are still present, in a record holding a single
keyframe of that channel's rest value. So a full mesh pose is always reconstructible by sampling
every channel.

**Value decoding.** Each 4-byte value is a vertex position quantised into three signed 10-bit
fields: X in bits 0-9, Y in 10-19, Z in 20-29 (bits 30-31 unused). Each field is a signed value
from -512 to 511. It is multiplied by that axis of the descriptor's **step** and added to that axis
of its **centre** (engine-confirmed, `FUN_004714a0`):

```
component(raw, shift, centre, step):
    field = signed_10_bit((raw >> shift) & 0x3FF)
    return centre + field * step
```

> **The box is the track's own, not the target mesh's.** It is usually close to the mesh's
> bounding box, which makes decoding into the mesh's box look almost right. But in 1,683 of the
> 1,734 tracks it is not the same box. The other 51 carry their mesh's own box, to within a
> hundredth of a quantisation step on every axis: among them the advisors' hands and eyes, the
> security camera's `cam01`, the jungle speakers' ferns, the Space speakers' petals and `supbogM`'s
> bushes. Compared against the mesh's rest vertices, a track's first keyframe lands on them to
> within one and a half quantisation steps in 676 tracks when decoded with the track's own centre
> and step, and in 119 when decoded into the mesh's bounding box: 105 both ways, 571 only with the
> track's box, and 14 only with the mesh's. (Plenty of animations don't start at the rest pose, so
> neither count reaches 1,734.)
>
> The 14 are all the Space Hoverbot's jet, `bs_jet`, one track in each of 14 clips, and the mesh's
> box matches them only by coincidence. Each clip's first keyframe puts six of the jet's 12 vertices
> 15 units below where they rest, on the bottom edge of the track's box (field -511). Decoded into
> the mesh's box, that same field lands on the mesh's own minimum, which is where those vertices
> rest.
>
> It shows most in the advisor's `Advisorm14`, where he rises onto the screen. Its antennae are
> quantised into a box nearly twice as tall as in his other clips. Read into the mesh's bounding
> box, they come out at a little over half their height and jump back when his next clip begins.

#### Raw vertex frames (bit 0x4000)

When a morph track also sets `0x4000`, the slot at +0x28 points at a 16-byte record instead of a morph
descriptor:

| Offset | Size    | Description |
| ------ | ------- | ----------- |
| 0x00   | 4 bytes | Offset of an index table: one uint per vertex (engine-confirmed pointer) |
| 0x04   | 4 bytes | Vertices per frame |
| 0x08   | 4 bytes | Offset of the positions: whole frames of 12-byte float `XYZ` vertices (engine-confirmed pointer) |
| 0x0C   | 4 bytes | Zero in all 30 |

The engine reads vertex `i` of frame `f` at `positions[f * verticesPerFrame + index[i]]`, over the
target mesh's vertex count. In all 28 of the 30 tracks whose base model resolves, the vertices per
frame equal the target mesh's vertex count. In all 30, the index table sits directly before the
positions. These tracks belong to the jungle speakers' and the Space droid's clips, among others.
Read as a morph descriptor, `droidm2.MD2` appears to aim 3,561 channels at a 16-vertex mesh.

### UV animation (bit 0x10000)

Slides texture coordinates. This is how the game animates water. The slot at descriptor 0x2C points
at a 20-byte descriptor:

| Offset  | Size    | Description                                |
| ------- | ------- | -------------------------------------------- |
| 0x00    | 4 bytes | Entry count `n`                              |
| 0x04    | 4 bytes | Offset of the index table                    |
| 0x08    | 4 bytes | Total key count `c`                          |
| 0x0C    | 4 bytes | Offset of the frame table                    |
| 0x10    | 4 bytes | Offset of the value table                    |

The three tables are contiguous, in the order index, value, frame, and exactly sized by those
two counts:

- **Index table**, `n * 4` bytes: per entry, a ushort first key and a ushort key count. The runs are
  packed end to end in entry order.
- **Value table**, `c * 8` bytes: per key, a `(u, v)` pair of floats.
- **Frame table**, `c * 2` bytes: per key, a ushort frame number.

The engine's sampler (`FUN_004745c0`, engine-confirmed) finds the two keys the current frame falls
between and interpolates the `(u, v)` pair across them. It writes entry `e` to UV slot `e` of the
target mesh, in the batches-of-4 layout under **Vertex data**. **An entry is one UV slot.** Entries
are not grouped into components, and no field names which UV an entry writes; its position in the
table does. In all 685 UV tracks whose target mesh resolves, `n` equals the mesh's **vertex order
length** (the UV count, not the vertex count). A channel moves a subset of its mesh by giving the
other entries keys that do not change.

All 690 UV channels in the game satisfy every one of those invariants. The per-entry key counts sum
to the stated total, both `indexTable + 4n == valueTable` and `valueTable + 8c == frameTable` hold
without exception, and every entry's frames strictly ascend from a first key at frame 0.

Of the 34,696 entries in the game, 30,305 have two keys. The other 4,391, with as many as 106 keys
(`Mbuggyb.MD2`), fall in 289 of the 690 tracks. A reader that takes each entry as a start and an
end value misreads those. The jungle Round Fountain's `fountainm.md2` shows it: its water mesh is 44
entries of five keys, whose `v` climbs 1.973, 2.534, 3.139, 3.789 and 3.873 over frames 0, 16, 32,
48 and 50.

`Jun_isleM1.MD2` is a clear two-key example. It scrolls all 128 of `Post Ripples01`'s UVs by
`(-1, -1)` over 100 frames. It also scrolls 104 of the `Island` mesh's 298 UVs by the same delta,
and holds the other 194 still. That is the shoreline foam lapping the beach, with the rest of the
island held still.

UV keys carry their own frames, but a clip's length is not worked out from its keys. It is the span
the animation block declares (see [Sequencing](#sequencing)). `Fan_isleM1` and `Hal_isleM1` scroll
water and do nothing else. Work an animation's length out from its rotation and morph keyframes
alone, and those files span zero frames, so their water never moves. 98 of the game's 1151 files with
rotation, morph or UV tracks are in that position.

### Position (bit 0x1)

The slot at +0x18 points at a 16-byte record:

| Offset | Size    | Description                                           |
| ------ | ------- | ------------------------------------------------------- |
| 0x00   | 4 bytes | Type: which curve joins the points, see below           |
| 0x04   | 2 bytes | Point count                                             |
| 0x06   | 2 bytes | Key count                                               |
| 0x08   | 4 bytes | Offset of the points: point count x 3 floats            |
| 0x0C   | 4 bytes | Offset of the keys: key count x 4 bytes                 |

A key is a ushort frame followed by a ushort that is zero in all 8,703 keys in the game. A point
is a position **relative to the node's parent**, and replaces the node's authored position, the
same way a rotation keyframe replaces its orientation (`0x0047208b`). On a model whose header word
at 0x30 has bit `0x4` set, the point is added to the authored position instead (`0x0047205d`); the
coaster parts' clips carry 43 position tracks. Before a track's first key the engine leaves the node
where it is.

The type's bits choose the curve, and the engine picks its sampler on exactly these bits
(engine-confirmed):

| Type   | Bit     | Tracks | Points per track     | Curve                                               |
| ------ | ------- | ------ | -------------------- | ----------------------------------------------------- |
| `0x12` | `0x2`   | 785    | 3 x (keys - 1) + 1   | Cubic Bezier, four points per segment               |
| `0x18` | `0x8`   | 431    | one per key          | Straight lines between keys                          |

Every one of the game's 1,216 position records has exactly the point count its type calls for.
The engine also has a sampler for a type with neither bit (a Catmull-Rom spline), but no file in
the game uses it. These are the same samplers, chosen on the same bits, that the engine uses for
[paths](#paths).

For a Bezier track, segment `s` runs from key `s` to key `s + 1` using points `3s` to `3s + 3`. The
last point of one segment is the first of the next. With `t` the fraction of the way between
the two keys' frames:

```
p = (1-t)^3 P0 + 3(1-t)^2 t P1 + 3(1-t) t^2 P2 + t^3 P3
```

The engine evaluates the same polynomial in power-basis form, with the constants 1, 3, -3 and -6.

Key frames never go backwards, though 4 tracks repeat a frame, which makes a zero-length segment.

The advisor is the clearest example. In `Advisorm14` his body's track (type `0x12`, keys at frames
0, 10, 20 and 30) raises it from (0, 0.21, -73.2) to its resting (0, 0.21, -21.2), and `Advisorm15`
drops it back to -74.4. His head does the same. That is him popping up from below the bottom of the
screen at the start of a line and ducking back out of view at the end.

### Scale (bit 0x80)

`0x80` always comes with `0x100`, and the two together are a scale channel (engine-confirmed). The
ushort at descriptor +0x12 is the key count, and the slot at +0x20 points at that many 16-byte keys:

| Size    | Description |
| ------- | ----------- |
| 2 bytes | Frame |
| 2 bytes | Zero in every key in the game |
| 4 bytes | Scale X |
| 4 bytes | Scale Y |
| 4 bytes | Scale Z |

The engine finds the two keys the current frame falls between, blends the three scales linearly, and
sets the lengths of the node's three basis axes to them (`0x00471d8d`, then `FUN_0046f910`). All 644
scale tracks in the game have strictly ascending frames. The values run from 0.00015 to 14.53, and
most sit near 1. The advisor's squash and stretch is one user: `Advisor_FANm17` keys his body at
(1.243, 0.838, 0.838) and then (1, 1, 1).

### Path progress (bit 0x200)

How far along its [path](#paths) a node has travelled, one float per frame. The slot at +0x24 points
at a 16-byte record, followed by the values:

| Offset | Size    | Description |
| ------ | ------- | ----------- |
| 0x00   | 4 bytes | Start frame, an integer. 0 on all 71 tracks |
| 0x04   | 4 bytes | Value count. On all 71 it is the clip's last frame plus one |
| 0x08   | 4 bytes | Zero on all 71 |
| 0x0C   | 4 bytes | Offset of the values (engine-confirmed pointer). On all 71 it is the record's own offset plus 0x10, but nothing in the format requires that |

The engine applies the channel only when the target node's parent is a **path node** (flag `0x8`).
It follows the path whose index that parent holds at +0x52 (engine-confirmed). All 71 tracks in the
game target a direct child of a path node. The haunted house's four carts `_f_kart01` to
`_f_kart04` hang off `Kart_path01` to `Kart_path04`. It samples the values only while the frame is
between the start and the start plus the count, and places the node on the path as described under
[Paths](#paths).

The value is a **percentage of the path**, not a distance. The engine adds 1000 and takes the result
modulo 100 (`0x006febf0` holds -1000.0, `0x006febf8` holds 100.0), so values below 0 and above 100
both wrap. It then multiplies by the path's segment count and by 0.01. Between two frames it blends
the values, and when they differ by more than 50 (`0x006fece0`) it blends the short way round the
loop.

`0x400` and `0x800` occur only on path-progress tracks. `0x400`, on 62 of the 71 tracks, turns the
node to face along the path. The engine samples the curve's direction as well as its position
(`0x004720aa`). `0x800` (12 tracks, all also `0x400`) is undecoded.

The values do not always rise. 41 tracks only rise, 4 only fall, 5 hold one value throughout, and
21 do both:

- Space's slide runs exactly 0 to 100 across 121 frames, in both `slidec` and `slidem`.
- A journey can be split across clips. The bus's `Busm2` runs 42.435 to 55.997, `Busm3` 55.996 to
  99.835 and `Busm1` 99.835 to 142.437: consecutive legs of one lap of 100, beginning part-way
  round. The bus's path is open, so where the value passes 100 the bus moves from the path's last
  sampled point to its first. Both are off the park.
- Each of the haunted house's `E`, `M` and `S` clips carries every cart once round. Three carts run
  from 99.99 to 199.99 and one from 0.004 to 100.004, which is the same place on the loop.
- The ferry runs its route backwards: `FerryM1` 99.98 down to 43.84, `FerryM2` 43.83 to 33.90,
  `FerryM3` 33.90 to -0.017. It breaks off twice, for 23 and 22 frames. `FerryM1` frames 152-174
  read 103.87 to 104.01 between 77.92 and 75.61. `FerryM3` frames 355-376 read 6.97 to 7.15 between
  11.72 and 10.13. `FerryM3` also reads 9.999 for frames 496-498. All four themes' ferries are the
  same.

### Visibility (bit 0x20000)

The ushort at +0x16 is this channel's entry count, and the slot at +0x30 points at that many
**signed 16-bit** entries. Each is a frame number whose sign says what happens from that frame on.
The engine walks the entries **from the end** and takes the first whose absolute value is at or
before the current frame (engine-confirmed):

- entry `>= 1`: show the node (clear its hidden flag, bit `0x10`)
- entry `< 1`: hide it. So an entry of `0` hides; there is no "+0"
- no entry qualifies: **leave the flag alone**. It is not defaulted to visible

All 2,536 visibility tracks in the game list their entries in order of frame.

This is how the advisor blinks. In `Advisorm10` his eyes carry `-40, 44, -180, 184, ...` and his
closed eyelids `0, 40, -44, 180, -184, ...`. The eyelids start hidden, and for four frames at 40,
and again at 180, his eyes are swapped for them.

### The hide list

An animation also carries a **hide list** of its own, separate from the visibility channel: a count
at block +0x1A and that many ushort node indices at block +0x38. 874 of the game's 1,278 animation
blocks carry one, 13,159 entries in all. Entries resolve through the same node rule as a track
target.

### Starting a clip

Starting a clip is not a blanket reset (engine-confirmed, `FUN_00472f60`). The engine first clears
`0x10` on every node the *outgoing* clip names (`FUN_00472310`): every entry in its hide list, and
every node one of its tracks targets. Then it hides every node in the *incoming* clip's hide list
(`FUN_00472d70`). Nodes carrying `0x80000000` are left alone by both steps, with one exception: when
the engine's animation object for the model carries flag `0x8` (at its +0x04), `FUN_00472310` first
clears `0x10` on every node of the model, locked or not.

Unless the caller suppresses it, `FUN_00472310` also puts back the rest pose of whatever the
outgoing clip moved, from the model as loaded. It skips any channel the incoming clip also carries on
the same node. It copies the node's transform where the outgoing track carried position, rotation,
scale or path progress (`0x289`), the vertices for vertex morph, and the UVs for UV animation. It has
no case for visibility, so visibility is never put back.

### Which animation file a model takes

The letter before the number is an **animation role**, not part of the model's name. The engine
keeps twelve of them, in the order `C D I L S M E U W B R O` (table at `0x006fe6bc`), and a model
ships a file only for the roles it uses. The jungle's security camera ships `camerac`, `cameras`,
`cameram` and `camerae`. The jungle's `bouncy.wad` holds `bouncyb`, `bouncyc`, `bouncyi` and
`bouncyr` (four roles of one clip each) alongside `bouncym1` and `bouncym2`, which are two clips of
one role. `junspray.wad` holds `JunsprayM1` through `M6` in a single role.

```
C=0   D=1   I=2   L=3   S=4   M=5   E=6   U=7   W=8   B=9   R=10   O=11
```

> The code assigns the letters **no meaning**, only ordinals. Role 0 is special because the build
> path hard-codes it, which makes "C = construct" safe. `I` for idle, `M` for motion and the rest are
> reasonable guesses and nothing more.

For each role, the loader at `FUN_00461f10` probes two filename forms, in this order:

1. `<stem><letter><n>.md2`, from `n = 1` upwards, stopping at the first gap. Format string
   `'%s%s%c%d.md2'` at `0x004623b3`. No shipped archive has a gap.
2. `<stem><letter>.md2`, with no number at all. Format string `'%s%s%c.md2'` at `0x004623df`,
   reached **only where the numbered run found nothing**.

So the unnumbered form is not a fallback for a missing file. It is the other way the same role is
shipped, and it is the commoner one. **197 of the 445 base models that carry any role at all ship
their `M` role like this and no other**, counted across all 312 archives by that rule. The terrain
is one: `terrain.wad` holds `base.md2` and `basem.md2` and nothing numbered, and the engine does
reach it. `FUN_004504c0` builds `'%s\Terrain'` and asks for the stem `"Base"`, falling back to
`"TestBase"`.

> **The "only where the numbered run found nothing" condition is load-bearing, and two archives in
> the game demonstrate it.** `jungle/mamfount` ships `mamfountm.md2` *and* `mamfountm1.md2` *and*
> `mamfountm2.md2`. The engine loads the two numbered ones and never opens the bare file. Space's
> `rides/megacost.wad` does the same in its second model set (below): `Pmegacostm.MD2` beside
> `PMEGACOSTM1.MD2`. Four of the bare file's targets lie past its model's node count. Suppose a loader
> listed the archive instead, or took the bare file in addition. It would find more entries than the
> engine finds, and every index into that role would be off by one from there on.

Not every such file animates anything. Of those 197, **160 carry a rotation, morph or UV track and
37 are empty**: structurally valid, declaring a frame span, but carrying no track of any kind.
None of the 197 carries the position and visibility channels alone.

**A leading `P` is a prefix, not a suffix.** When its caller asks for it, the item loader loads a
whole second model-and-animation set named `p<stem>` through the same probe, using the format string
`"p%s"` at `0x0074d368`, and keeps it apart from the first. `PJunspray.MD2` and `PJunsprayM.MD2` are
one such set. 39 archives ship one, all of them rides, sideshows and upgrades. What the second set
is for is not known.

#### Role 0 decides what a built item looks like

The engine plays role 0 once, when the player builds the thing, and never again. A park loaded from
a save restores the animation state each object had settled into, rather than replaying its
construction. **So the last frame of the `C` clip is the appearance of a finished item**, and that is
the only place some things are ever put away.

The Jungle's Belly Bounce is the clear case. It arrives as an egg. Its construction clip switches
the egg off at frame 94, the same frame the dinosaur inside is switched on. It drops the shell at
131, and brings in the fence, the posts and the two sign boards at frames 114 to 144. Anything that
draws the model without posing it shows a ride standing inside an unbroken egg.

Across all four themes, 966 construction-clip visibility tracks end with their node **shown** and 56
end **hidden**. The hidden ones fall into two groups. Some are things the building throws away:
the Belly Bounce's egg and shell, a witch's frog, the monkey ride's crate and its shards, puffs of
smoke, and the jungle speakers' ferns. The others are parts a ride shows only while it runs: beams
and flashes, water, the go-karts' bees and bats, the Orbiter's ships and the sideshows' targets.

### Sequencing

Nothing in the format says how a model's `M1`, `M2`, ... animations are ordered, when they
should play, or whether they loop. A gate's "doors open" and "doors close" are just two files, and
nothing in the `.md2` data tells them apart. In the original game this is driven externally,
by a `TRIGANIM` instruction in that ride's [compiled script](/formats/rsse/). See the gate
example on the [RSS](/formats/rss/) page.

The engine plays a model's clips on **animation channels**, each an independent 0x38-byte player, so
several clips of one item run at once. That is how a sideshow works three lanes at a time: the
Jungle Spray's script drives channels 0, 1 and 2. The number of channels is not in the file; the
code that builds the model chooses it. Nothing in the engine knows what a "lane" is. That lives in
the item's own [ride script](/formats/rsse/).

Keyframe numbers are frames at **30 per second**. The engine advances a playing animation by the
elapsed milliseconds times 0.03. Frame numbers set a speed, not a rate to draw at: the samplers
take a fractional frame, so poses between keys are interpolated. A clip's length is the span its
block declares, `(block+0x08 - block+0x04)` frames. The engine turns it into milliseconds by
multiplying by the float 33.33333206 at `0x006fec08` and truncating (engine-confirmed). That is the
nearest float to 1000/30, not the exact value.

### Open questions

- What the engine does with a zero-length position segment, which 4 tracks have.
- The frame-ish value at descriptor +0x0C, and the unknown 4 bytes ending each morph record.
- What bit `0x02` of a morph descriptor's flag byte selects. It does not change the channel count.
- What track bits `0x10`, `0x20`, `0x40`, `0x100`, `0x800`, `0x2000` and `0x8000` select.
- The five undecoded pointers in the animation block (0x20, 0x24, 0x28, 0x30, 0x3C) and the word at
  its start.
- What the ferry's two breaks in path progress do, and the three frames of 9.999 at the end
  of `FerryM3`.
- What bit `0x1` of a position record's type would do. No position record sets it, and no path sets
  `0x10`, which every position record carries.
- What drives the eight paths that no clip follows, the lobby islands' among them.
- What the second, `p`-prefixed model set is for.
