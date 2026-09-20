---
title: Models (*.MD2)
---

Theme Park World's models are Bullfrog's own `.MD2`, unrelated to Quake II's format of the same
extension. A model file holds meshes, a node tree, and — on twenty-four of the game's models — one
or more **paths**: the routes vehicles and ride cars follow.

This page currently documents the path table only. The rest of the format is read by
`ModelFile.cs` in the OpenTPW source, whose class comments carry the field-by-field detail.

### Paths

The `uint` at file offset **`0xac`** is the offset of an array of 16-byte records, or `0` when the
model has no paths. Each record is four little-endian `uint`s:

| Offset | Type | Meaning |
|---|---|---|
| `+0x00` | `uint` | Sampler type — see below |
| `+0x04` | `uint` | Number of points |
| `+0x08` | `uint` | Offset of the points |
| `+0x0c` | `uint` | Always `0` |

The points are 12-byte `XYZ` triples of `float`, in the model's own space.

#### How many records there are

**Nothing in the file states the number of paths.** Every `u16` in `0x90..0xc0` was measured against
the known counts and none of them is a count: `0xb8` reads `1` for the haunted house, which has
four paths, and `49` for the bus, which has one.

Instead the paths are indexed **by node**. A node that follows a path carries its index in the
`u16` at **`+0x52`** of its node record, immediately before that record's name pointer. The
haunted house's four nodes are named `Kart_path01` … `Kart_path04` and hold `0, 1, 2, 3`.

So the number of records is `1 + max(node +0x52)`, with a floor of one — a floor rather than a
maximum because three models (`Bus.MD2`, `FERRY.MD2` and `gokarts.MD2`) have a path that **no node
names at all**, leaving every `+0x52` at zero.

This matters, because the array is not self-delimiting. The bytes after the haunted house's fourth
record are vertex floats, and read as a record they give a point count of 1,112,011,916.

Note that `+0x52` does not distinguish "follows path 0" from "follows no path"; both read `0`.

#### Sampler type

The type word uses the same two bits as an animation file's position channel:

| Bit | Meaning | Models |
|---|---|---|
| `0x2` | Cubic Bézier control points | every vehicle and ride path |
| `0x8` | Plain waypoints | `slide.MD2` only |

Observed values are `2`, `3` and `8`.

A Bézier path's point count is a **multiple of three, not `3n+1`**, because the loop is closed —
the final segment's end point is point `0` again. The bus's 45 points are 15 closed cubic segments,
the ferry's and seaplane's 33 are 11 each, and the haunted house's 48 are 16.

#### Where the paths are

Measured across all of the game's models, twenty-four carry a path table:

| Model | Paths | Type | Points |
|---|---|---|---|
| `Bus.MD2` (all four themes) | 1 | 2 | 45 |
| `FERRY.MD2` (all four themes) | 1 | 2 | 33 |
| `Seaplane.MD2` (all four themes) | 1 | 2 | 33 |
| `haunt.MD2` | 4 | 3 | 48 each |
| `gokarts.MD2` / `gokarts.md2` | 1 | 2 / 3 | 102 / 51 |
| `bigapple.MD2` | 1 | 3 | 39 |
| `ratrace.MD2`, `wateride.MD2` | 1 | 3 | 12 |
| `slide.MD2` | 1 | 8 | 12 |
| `Advisor_HAL.MD2` | 1 | 3 | 51 |
| `Jun_isle`, `Fan_isle`, `Hal_isle`, `Spa_isle` | 1 | 3 | 12 |

The haunted house's four paths share an identical bounding box: four carts on one circuit, offset
in phase rather than following different routes.

The four lobby islands' path is named `Spline path` and spans X −29.8…29.8, Y exactly 0, Z
−39.8…39.8 — a flat loop at the island's own origin, not a camera track.

### Progress along a path

A path is driven by a **per-frame scalar**, stored in an animation track that sets channel bit
`0x200` (see the animation channel notes). The scalar is a **percentage of the path**, and values
outside `0…100` wrap: the bus's three clips run 42.435 → 55.997 → 99.835 → 142.460, one full lap
of 100.025 beginning part-way round, and the haunted house's run 0.004 → 199.995, which is two laps.

The scalar does not always increase. The ferry traverses its path backwards, its clips descending
99.983 → 43.844 and 33.903 → −0.017; of the game's 71 `0x200` tracks, 46 ascend and the rest descend.
