---
title: Saves (*.tpws, *.ints, *.lays)
---

Theme Park World stores saves with three different extensions.

These extensions are:

- **TPWS**: Standard save
- **INTS**: Initial save (e.g. for *Instant Action* mode)
- **LAYS**: Online save (for uploading parks)

## File format

**Header**

| Size      | Description                                            |
| --------- | ------------------------------------------------------ |
| 4 bytes   | Version - see below. Not a magic number                 |
| 1 byte    | Padding                                                |
| 824 bytes | Copyright notice, UTF-16 - 412 characters              |
| 711 bytes | Padding                                                |

> The first four bytes are a **version, not a magic number**. The only file of this shape the game
> ships — `data/levels/jungle/Easymode.TPWI` — carries **400** (`90 01 00 00`), so a reader that
> requires `F4 01 00 00` (500) rejects it. Both values should be accepted.

The copyright notice is UTF-16, so its 824 bytes are **412 characters**; reading it as one byte per
character yields every other byte as a null. It begins at offset 5, after a single pad byte.

These sizes close exactly on the offset the next section begins at: `4 + 1 + 824 + 711 = 1540`
(`0x604`), which is where the file type sits.

**File info**

| Size    | Description                                     |
| ------- | ----------------------------------------------- |
| 4 bytes | File type - `00 01 22 19`                       |
| 1 byte  | File version - `85`                             |
| 1 byte  | Online flag - `00` for offline, `01` for online |
| 2 bytes | Padding                                         |

**Data (compressed using ZLIB)**

| Size     | Description                                                  |
| -------- | ------------------------------------------------------------ |
| 4 bytes  | Magic number - `BILZ`                                        |
| 4 bytes  | Uncompressed size - what the payload inflates to             |
| 4 bytes  | Size of this block, including these 28 header bytes           |
| 16 bytes | Unknown                                                      |

> Neither of those two values is a compressed length. In the shipped Jungle park the first is
> **1,608,309** — the exact size the ZLIB stream inflates to — and the second measures the block from
> the `BILZ` tag to the end of the file. Both are worth checking on load: one confirms the block
> reaches the end of the file, the other that the payload inflated to the size it claimed.

The header is 28 bytes **including the `BILZ` tag**, which is an easy four bytes to lose. In the
shipped park the tag sits at `0x60D`, so the ZLIB stream begins at `0x629` and continues to the end of
the file.

## Inside the payload

> **The payload is a serialised memory image, not a portable format.** It stores live heap pointers
> verbatim — which is why two otherwise identical objects differ in three bytes of four — and nothing in
> it can be located by searching for a value. Every offset has to be reached by walking from the start.

It is a fixed sequence of module blocks. After each module's bytes the game reads a dword and **checks
it against that module's tag**, which is how it detects a module that failed to load the same number of
bytes it saved.

> **Tags are little-endian dwords, so they read backwards in a byte dump.** Searching the payload for
> `WRLD` finds nothing; searching for `DLRW` finds it immediately. Modules butt directly against one
> another with no length prefix or padding, so a reader finishes one module exactly on its tag and the
> next begins four bytes later. Being one byte out corrupts every module that follows.

An untagged **ActionRec** block comes first, and it is what gives the World block's start:

```text
u32  mLoadedPublishedPark
u32  recording_size
byte[recording_size]  recording      -> the block is 8 + recording_size bytes
```

In the shipped Jungle park that reads `0` and `1171`, so ActionRec occupies `0`–`1178` and the World
block begins at **`0x49B`**. Then, with the trailer offsets that park produces:

| Tag | Module | Trailer at | Block bytes |
| --- | ------ | ---------- | ----------- |
| `WRLD` | World | `0x16D1A6` | 1,495,462 |
| `SPSC` | Sprite scripts | `0x16E6F6` | 5,452 |
| `PART` | Particles | `0x1810D6` | 76,252 |
| `MESS` | Message centre | `0x1811E4` | 266 |
| `CLOK` | Clock | `0x1811F0` | 8 |
| `VANT` | Vanilla time | `0x1811F8` | 4 |
| `GSYS` | Game system | `0x181220` | 36 |
| `RSYS` | Ride system | `0x18569A` | 17,526 |
| `TRAK` | Track rides | `0x1856CA` | 44 |
| `FLYR` | Flying rides | `0x185892` | 452 |
| `RSSE` | RSSE scripts | `0x1882FE` | 10,856 |
| `KAME` | Camera | `0x18832A` | 40 |
| `COAS` | Coasters | `0x18833E` | 16 |
| `ADVS` | Advisor | `0x1884B6` | 372 |
| `SOUN` | Sound | `0x188A5F` | 1,445 |
| `CHTS` | Cheat | `0x188A65` | 2 |
| `ADSC` | Advisor scoring | `0x188A6D` | 4 |

A four-byte UI block follows, which has no tag.

## The World block

This is where a park's contents live. Its layout, with every boundary closing exactly on the next:

```text
0x00049B  26 header fields (below)                                 ->  0x0004DB
0x0004DB  mObjectControls[150], 32 bytes each                      ->  0x00179B
0x00179B  mNumObjectControls u32, mPreviousSearchKey u16           ->  0x0017A1
0x0017A1  32 records of 20 bytes                                   ->  0x001A21
          mType 4, mName 4, mPayGrade 1, mSubType 1, mValid 1,
          mOnPointer 1, mTimeSig 4, mTimeoutTime 4
0x001A21  arrival and clock fields, 76 bytes                       ->  0x001A6D
0x001A6D  16,384 map cells (below)                                 ->  0x152431
0x152431  u32 Used Thing Head                                      ->  0x152435
0x152435  the thing list (below)                                   ->  0x16D1A6
0x16D1A6  the trailer, stored as DLRW
```

The header is 26 fields in this order, and the names are the game's own — they are passed to a logging
call that the release build compiles away, so they never reach the file and cannot be searched for:

```text
version 4 | mArrivalVehicle_Size1/2/3 2 each | mBankAccount 2 |
mCurrentArrivalVehicle 2 | mGameTick 4 | mMechanicHQ 2 | mParkAnalyser 2 |
mParkClosed 4 | mNumberOfVisitorsToDate 4 | mParkGates 2 | mTrafficLights 2 |
mRandomSeed 4 | mResearchLab 2 | mStaffHQ 2 | mTagSystem 2 | mUIMsgReceiver 2 |
mWeather 2 | mWorldState 4 | mFirstHandyman 2 | mFirstMechanic 2 |
mFirstEntertainer 2 | mFirstGuard 2 | mFirstResearcher 2 | mFirstObject 2
```

`mParkGates`, `mTrafficLights` and the `mFirst*` fields are **handles, not indices** — the game compares
them against a thing's own id with `==`, so `mParkGates` of 11 means "the thing whose id is 11", not
"the eleventh thing".

### The map: 16,384 gated cells

Each cell begins with a status byte whose low three bits say which sub-records follow it. A cell that is
entirely default writes the byte and nothing else, which is where the block's variable length comes from
— and why no fixed stride will ever walk it.

| Bit | Sub-record | Bytes |
| --- | ---------- | ----- |
| `1` | Map cell | 52 |
| `2` | Track cell | 31 |
| `4` | Effects cell | 10 |

So a cell is `1 + the bits it sets`. In the Jungle park only two combinations occur: `3` on 16,134 cells
(84 bytes) and `7` on the other 250 (94 bytes), summing to 1,378,756 — the region exactly. An
implementation that only needs what is *in* the park can measure each cell and skip it.

> **Check the stride cell by cell, not just in total.** Measured this way, all 16,384 cells land on the
> next cell's status byte every single time. That is a sharper check than the block's own `DLRW` trailer,
> which a pair of compensating errors would still reach.

**The map cell**, 52 bytes. The first 29 are a tile base that the track cell repeats field for field; the
23 after them are litter and pylon bookkeeping:

```text
+0   u8   mDirection
+1   u16  mFlags
+3   u32  mMeshInstance
+7   u8   mNeighbours
+8   u16  mOverlapCounter
+10  u16  mParentID
+12  u8[12] mTileData
+24  u32  mType
+28  u8   mHoardingNeighbours        -- end of the 29-byte tile base
+29  u32  mLitter          | +33 u16 mLitterCollector | +35 u32 mLitterScript
+39  u32  mLitterScript    | +43 u16 mPylonIndex      | +45 u8  mStatusFlags
+46  u32  mTimeMarkedForLitterCollection                | +50 u16 mWho
```

The **track cell** is that same 29-byte base followed by `u16 mSegmentNumber`, giving 31.

> **Cells are indexed `y * 128 + x`** — the opposite way round from the attribute map in `base.map`,
> which is `x * 128 + y`. Getting it backwards still produces something that looks like a map, so it is
> worth pinning: only the y-major reading reproduces `base.map`'s own bus road, ticket booths and
> entrance column.

`mNeighbours` and `mDirection` are **stored, not computed**, so a renderer does not have to derive a
neighbour mask from the cells around it. They share one compass: over the Jungle park's path cells
`mDirection` only ever reads 0, 1, 4, 16 or 64 — the four cardinals of an
`N NE E SE S SW W NW` layout, and nothing between them.

`mTileData` is **three dwords**, not one opaque run:

| Dword | Meaning | Values in the Jungle park |
| ----- | ------- | ------------------------- |
| 0 | Tile set | `1` on all 78 path cells, `2` on all 4 queue cells, `0` on the other 16,302 |
| 1 | Tile index within that set | 2–20 on path cells, addressing the theme's [`PathTex` table](/formats/tct/) |
| 2 | Rotation, degrees | `0`, `90`, `180` or `270` — on every one of the 16,384 cells |

The index really does address the theme's `.tct`, confirmed by cross-referencing every index the shipped
park uses against the shape that cell's neighbours make: straights come out degree 2 and collinear,
corners degree 2 and bent, T-junctions degree 3, and the park's one crossroads degree 4 with a mask of
exactly N+E+S+W. The [Texture Correspondence Table](/formats/tct/) page carries that table in full,
including the two edge tiles whose masks show this to be an **area** tile set rather than one-cell-wide
lines — a walkway can be more than one cell across, and Jungle's entrance avenue is two.

> Two independent things support that split, and a wrong one would have to produce both by accident: the
> first dword divides the map exactly along the boundary the theme's `.tct` draws between its `PathTex`
> and `QueueTex` sections, and the third is a quarter turn everywhere and never an arbitrary angle. **So a
> park's paths carry the tile they draw and the turn it takes** — neither has to be inferred from
> neighbours.

`mType` says what a cell is. Counts across the Jungle park:

| `mType` | Cells | What |
| ------- | ----- | ---- |
| 7 | 9,077 | |
| 0 | 6,875 | |
| 2 | 240 | |
| 1 | **78** | **path** — drawing them gives a connected loop with an avenue down to the park entrance |
| 30 | 66 | exactly the count of `base.map` cells carrying `0x80`, so the fixed approach every park inherits |
| 4 | **35** | **the body of a built thing's footprint** |
| 9 | **8** | **a footprint cell the thing is used from** |
| 3 | **4** | **queue** |
| 10 | **1** | **the far end of a footprint** |

Types 7, 0 and 2 are still recorded as observed rather than named.

### 4, 9 and 10 are one thing: a built thing's footprint

Together those three are **44 cells, and that is exactly the footprints of the park's eleven placed
objects** — nothing left over and nothing missing. Grouping the 44 into connected components gives seven
groups whose bounding boxes are the items' own sizes:

| Group | Cells | What stands there |
| ----- | ----- | ----------------- |
| (57,15) 3×5 | 13 | the 2×2 staff room at (58,15) and the 3×3 fountain at (57,17) |
| (51,23) 3×4 | 12 | the Belly Bounce |
| (51,30) 3×3 | 9 | the Jungle Spray |
| (43,29) 2×3 | 5 | the 1×1 litter bin and the 2×2 drinks shop |
| (55,15) 1×3 | 3 | the three 1×1 toilets, stacked |
| (40,29), (55,29) | 1 each | the two security cameras |

`9` sits where a thing is used — the single cell of each toilet, the drinks shop's counter, the litter
bin, the staff room's door, and the end of the Belly Bounce its queue arrives at. `10` appears once, on
that ride's far end. Those two readings are inferred from where they sit; what is *measured* is that all
three types belong to a footprint rather than to the ground.

> **This matters to anyone drawing a park.** Every one of the 44 carries a real ground texture index in
> `base.MD2` — not one is the "something covers this" index 0 — and yet each is already covered, because
> every item this park is built with opens its model with a flat floor plate exactly as wide as its
> footprint: `J_WC` under a toilet, `wf_floor` under the fountain, `js_base` under the staff room,
> `cn_floor01` under the drinks shop, `jb_floor` under the Belly Bounce, `jc_base` under a camera. Draw
> the ground there as well and the two fight for the same depth. The same goes for the four queue cells,
> where every piece of queue brings its own base.

### Where a built thing stands on its cells

An object's record gives an anchor cell and an angle (see the thing list below). Its footprint is placed
by turning it about the **middle of that anchor cell** — not about the middle of the footprint — and the
stored angle turns the *opposite* way to a positive rotation about the up axis.

Both halves are forced by the shipped park, because the map cells state the answer independently of the
object records. The staff room is anchored at (58,16) at 90° and its footprint is marked (58,15)–(59,16);
the fountain is anchored at (57,19) at 90° and marked (57,17)–(59,19). Turning about the footprint's
middle, or turning the other way, puts each somewhere the save does not mark — a positive turn lands the
fountain on (55,19)–(57,21).

> **Do not read the executable's `0x168 - angle` as confirming this.** That constant is at the *queue's*
> call site (`FUN_005229e0`), and what it says is that a piece of queue turns the **opposite** way to a
> built object — it is the difference between the two, not a convention they share. The marked footprint
> cells are the evidence for an object's turn; the queue's is the art on `queend`, described in the
> [Texture Correspondence Table](/formats/tct/) page.

Those two are the only rotated items in the park whose footprint is bigger than one cell, so 180° and
270° follow the rule the two 90s establish rather than being measured in their own right.

### The thing list

A singly linked list of everything in the park — people, objects and the park's singleton managers.
`Used Thing Head` gives the first thing's id, and **each record's leading dword is the id of the record
that follows**, not its own.

> That is the trap in this block. Read as the thing's own id it is off by one everywhere and still looks
> plausible. Two things prove it is a next-pointer: the last record's is `0`, a terminator no thing could
> have as an id, and the sequence is not monotonic — it runs 41, 40 … 29, then 15, then 28, which is what
> a list with something spliced into it looks like. Read correctly, the Jungle park's chain is an exact
> permutation of 1–42.

Every thing opens with the same 16 bytes:

```text
+0   u32  Used Thing Next     the NEXT thing's id
+4   u32  thingmodel
+8   u16  mX                  in 1/256 of a cell
+10  u16  mY
+12  u16  mMapChild
+14  u16  mMapParent
```

After that they diverge completely and the sizes are wildly uneven, so the stream cannot be strided —
each model has to be recognised. In the Jungle park: model 1 is a guest (533 bytes), 3 a placeable
catalogue object (1,099), 4–8 the five kinds of staff (509–513), 9 the strike system (103), 10 a bare map
object (18), and 11–19 the singleton managers (16 to 73,544).

A **model 3** is what a park is furnished with, and it continues:

```text
+16  i32  mAngle    degrees
+20  u16  mId       the item's Info.Id, from its own SAM file
```

`mX` and `mY` are in 256ths of a cell, so the cell is `value >> 8`. A thing with no place on the map
stores `128` in both, which is *half a cell* rather than an obvious sentinel — anything treating it as a
position puts those objects at the origin. The Jungle park's gate, traffic lights and bus are all stored
that way, because their positions live in their models rather than in the save.

> **Make the trailer an assertion.** Every record length in the block feeds one running offset, so a
> reader that ends exactly on `DLRW` had all of them right, and one that is a single byte out cannot. It
> is the same end-to-end check the container already allows: the block length reaching the end of the
> file, and the payload inflating to its declared size.
