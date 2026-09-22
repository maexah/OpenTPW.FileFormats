---
title: Models and Animation (*.md2)
---

Despite the extension, these are not Quake II models. Every `.md2` in the game starts with the magic
`0x1CD15D46`, and all but four carry the version words `0xDD` and `0xCB` at `0x04` and `0x08` (see
[Three versions, and the loader takes one](#three-versions-and-the-loader-takes-one)). They come in one of
two kinds:

| Kind | How to tell | Count in the game |
| ---- | ----------- | ----------------- |
| **Static mesh** | the mesh table pointer at `0x70` is non-zero | 1,122 (47%) |
| **Animation** | that pointer, and those at `0x50` and `0x54`, are all zero | 1,279 (53%) |

An animation file carries no geometry at all — no vertex positions, no texture names — and poses the
nodes of a separate base model whose name is a prefix of its own. Parsing one as a mesh is what runs
off the end of the file.

## Three versions, and the loader takes one

Measured over every `.md2` in the game, loose and inside every `.wad` — 2,129 files:

| Words at `0x04` / `0x08` | Files |
| ------------------------ | ----- |
| `0xDD` / `0xCB` | 2,125 |
| `0xCF` / `0xC9` | 2 — `wr_tunnel.md2` and `wr_tunnelm.md2`, in the Jungle's `rides/wateride.wad` |
| `0x18` / `0x17` | 2 — `garrow.MD2` and `rarrow.MD2`, loose in `data/generic/dynamic/` |

> **The engine loads only `0xDD`.** Its one model reader refuses anything above `0xDD`, and anything
> below unless the caller asks for the older layout — which no caller in the shipped executable does.
> So the four odd files are shipped data the game never draws.

`garrow.MD2` and `rarrow.MD2` are a green and a red block arrow, each a single node named `Line01`
with 14 vertices and 24 triangles: 5 units wide at the head, 8 long pointing +Z, 2 thick. They differ
only in their name, their texture (`green.tga` against `red.tga`) and three header words, one a save
time dated September 1998 — thirteen months before the release models. Their layout is **not** the
one this page describes: 32-byte vertex records, 24-byte face records and a 136-byte node, reached
through unaligned pointers from `0x5A`. A reader for the `0xDD` layout that meets one will take the
dword at `0x70` (`0x05F90000`) as a mesh table pointer and run off the end of the file.

## A model is a tree of nodes, not a list of meshes

The ushort at `0x42` is the total node count and the one at `0x44` the mesh count. **The meshes are
the first `meshCount` nodes**, in 160-byte records at the pointer in `0x70`; the remainder are
transform-only nodes in 88-byte records at the pointer in `0x74`. The engine indexes them exactly
that way:

```
node < meshCount ? meshTable(0x70) + node * 0xA0
                 : nodeTable(0x74) + (node - meshCount) * 0x58
```

Both record kinds begin with the same header — a flag word at `+0x00`, then three links stored as
**file offsets** (parent `+0x04`, next sibling `+0x08`, first child `+0x0C`), the node's own
transform at `+0x10`, and a name pointer at `+0x54`.

A node's transform is relative to its parent, so a reader that ignores the tree leaves children piled
at the model origin.

### The flag word

| Bit | Meaning |
| --- | ------- |
| `0x200` | transform-only node — it marks a place rather than occupying one |
| `0x10` | **hidden**. Set at runtime only; nothing in the shipped data carries it |
| `0x20` | prune this node's children from the walk |
| `0x80000000` | protect this node from animation visibility (hide-and-lock) |

`0x10` and `0x20` are separate and it matters: the node walk tests `0x10` **after** computing the
node's matrix and then carries straight on into the children, so **a hidden node does not hide what
hangs off it**. Only `0x20` prunes a subtree.

Transform-only nodes are named like any other, and the names are the only place the file says what a
node is *for* — `sound node`, `ant_emitter`, `1stperson`, `camera`, `entrance`, `destroy`.

## An item's clips are twelve groups

A buildable item does not ship "an animation". The loader probes for clips using a **12-entry suffix
table**, in this order:

```
C=0   D=1   I=2   L=3   S=4   M=5   E=6   U=7   W=8   B=9   R=10   O=11
```

The name is `<prefix><stem><suffix><n>.md2` with **`n` counting from 1** until a probe fails, and if
no numbered file exists it falls back to the unnumbered `<prefix><stem><suffix>.md2`. So the jungle's
`bouncy.wad` holds `bouncyb`, `bouncyc`, `bouncyi` and `bouncyr` — four groups of one — alongside
`bouncym1` and `bouncym2`, which are two clips of one group; and `junspray.wad` holds `JunsprayM1`
through `M6` in a single group.

> The code assigns the letters **no meaning** — only ordinals. Group 0 is special because the build
> path hard-codes it, which makes "C = construct" safe; `I` for idle, `M` for motion and the rest are
> reasonable guesses and nothing more.

A leading `P` is a **prefix**, not a suffix: the engine loads a whole second model-and-animation set
named `p<stem>`, kept separately from the first. What it is for is not known.

An item has a count of animation **channels**, each an independent 0x38-byte record, so several clips
of an item run at once — which is how a sideshow works three lanes at a time. Nothing in the engine
knows what a "lane" is; that lives in the item's own [ride script](/formats/rsse/).

### Group 0 decides what a built item looks like

The engine plays group 0 once, when the player builds the thing, and never again — a park loaded from
a save restores the animation state each object had settled into rather than replaying its
construction. **So the last frame of the `C` clip is the appearance of a finished item**, and that is
the only place some things are ever put away.

The Jungle's Belly Bounce is the clear case. It arrives as an egg, and its construction clip switches
the egg off at frame 94 — the same frame the dinosaur inside is switched on — drops the shell at 131,
and raises the fence, the posts and the two sign boards over the frames after that. Anything that
draws the model without posing it shows a ride standing inside an unbroken egg.

Across all four themes, 966 construction-clip visibility tracks end with their mesh **shown** and 57
end **hidden**, and all 57 are things the building threw away: eggs and shells, a witch's frog, a
monkey's crate and its shards, puffs of smoke, and the beams and flashes a ride shows only while it
is running.

## Tracks

An animation is a list of **tracks**, each posing one node. The uint at `0x98` points at a 72-byte
block; in it, the uint at `+0x08` is the last frame, the ushort at `+0x12` the track count and the
uint at `+0x2C` the track table's offset. The table is `count * 64` bytes and **ends exactly where
that block begins** — an identity worth asserting, because a mislocated table produces confidently
wrong animation rather than an obvious failure.

Each 64-byte descriptor carries its own index at `+0x00`, a flag word at `+0x04` saying which
channels it has, and **the target node as a ushort at `+0x14`** (`+0x16` is a separate value, so a
32-bit read there yields a garbage index). A track can carry several channels at once.

| Bit | Channel | Slot |
| --- | ------- | ---- |
| `0x00008` | rotation | count (ushort) at `+0x10`, keys at `+0x1C` |
| `0x01000` | vertex morph | descriptor at `+0x28` (bit `0x4000` changes what it points at) |
| `0x10000` | UV animation | descriptor at `+0x2C` |
| `0x20000` | visibility | count (ushort) at `+0x16`, entries at `+0x30` |
| `0x00001` | position | record at `+0x18` |

Frame numbers are frames at **30 a second**, and are a speed rather than a rate to draw at: the
samplers take a fractional frame, so poses between keys are interpolated.

### A rotation key is the orientation *inside the parent*

A rotation keyframe is 20 bytes — ushort frame, ushort flags, then a float quaternion — and it is the
orientation the node should hold, **not** a turn to add to the one it was authored with. The
authored rotation has to come back out before the keyed one goes in.

**Which authored rotation is the trap.** It is the node's own transform, before its parents' — not
where the node ends up in the model. The two are only the same when the node hangs off a root that
carries no rotation, and **every gate in the game is exactly that**, which makes gates the one family
of models that cannot tell the difference. Across the game, **1,078 of 2,592 rotation tracks** target
a node where the two disagree.

The Jungle Spray settles it. Its three animal heads hang off a bench that is itself turned a quarter
turn, so each head is square *within the bench* while standing at a quarter turn in the model — and
every clip keys them square. The proof is the construction clip, which by definition ends on the
finished object: it ends with the fence keyed at 90°, the guns at 180°, one puddle at 245° and the
bench square, which are those meshes' local orientations exactly and disagree with every one of their
world orientations.

### Visibility

A count at `+0x16` and that many **signed shorts** at `+0x30`. Each is a frame number whose sign says
what happens from that frame on. To resolve it, walk the entries **from the end** and take the first
whose absolute value is at or before the current frame:

- entry `< 1` — **hide** the node (so an entry of `0` hides; there is no "+0")
- entry `>= 1` — show it
- no entry qualifies — **leave the flag alone**. It is not defaulted to visible

Starting a clip is not a blanket reset. The engine first clears `0x10` on every node the *outgoing*
clip hid — both those in its hide-list and those its tracks target — and then applies the *incoming*
clip's own initially-hidden set. Nodes carrying `0x80000000` survive both.

> An animation also carries a **hide-list** of its own, at `anim+0x38` with its count at `anim+0x1a`,
> applied when the clip starts. It is separate from the visibility channel.

### Vertex morph

The descriptor at `+0x28` gives a record count at `+0x02`, a record table at `+0x0C`, and two sets of
three floats at `+0x14` and `+0x20` — the centre and step of the box **this track's** positions are
quantised into, which is the track's own box and not its mesh's bounding box. A keyframe value packs
three signed 10-bit fields (X in bits 0–9, Y in 10–19, Z in 20–29); each is multiplied by the step
and added to the centre.

A morph track has one channel per vertex of its target mesh **plus two**, and those two trailing
channels are the corners of the box the vertices span at that keyframe, minimum then maximum — the
engine reads them as the animation's bounding box.

Each record is `ushort keyframes`, `ushort channels`, then pointers to the channel ids, the frame
indices and an entry-major block of `keyframes * channels` four-byte values. Both arrays are padded
to a 4-byte boundary, so the gaps between those pointers are `2*n` or `2*n+2`.

### Position

The record at `+0x18` is a uint type, a ushort point count, a ushort key count, and pointers to the
points and the keys. The type picks the sampler:

- `0x2` — a cubic Bézier through segments of four points, three new points per key, so the point
  count is always `3 * (keys - 1) + 1`
- `0x8` — straight lines, one point per key

A point **replaces** where the node sits relative to its parent, the same way a rotation key replaces
its orientation.
