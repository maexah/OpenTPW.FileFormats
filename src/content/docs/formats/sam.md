---
title: Settings and Modifiers (*.sam)
---

SAM files are plain-text files that specify values for different parts of the game. These files can be found both within archives and within the base directories for Theme Park World.  

A typical SAM file looks like this:

```text
# Sell 30 Drinks in 60 days
Challenges[1].Type                  3
Challenges[1].FollowupType          0
Challenges[1].TargetTime            60
Challenges[1].TargetVal             30
Challenges[1].TargetObj             0
Challenges[1].TargetObj2            0
Challenges[1].TargetStaffType       0
Challenges[1].Prize                 5000
Challenges[1].CheckAtEndOnly        0
Challenges[1].Independent           1
```

## Format

SAM files follow the format of `key <whitespace> value`.

Comments are preceded with a pound symbol (`#`) and continue until the end of the line.

Strings are surrounded with double quotes (`"`) and are used for various properties, i.e. the ride's name.

## Where they are found

- **Loose**, under `data/` — `high.sam`, `med.sam`, `low.sam`, `sound.sam`, `Challenges.sam`.
- **Per theme**, under `data/levels/<theme>/` — `Standard.sam` (the balance file), `global.sam` (what
  the lobby needs before the theme is loaded) and `Easy_Standard.sam`.
- **Per item**, inside the item's own WAD and named after it — `gates.wad` holds `Gates.sam`, alongside
  an `Easy_Gates.sam` which is frequently just `# Empty`.
- **Per category**, in each content folder — `features/Features.sam`, `rides/Rides.sam`, `shops/Shops.sam`,
  `sideshow/SideShow.sam`, `upgrades/Upgrades.sam`. These hold defaults for a whole category rather than
  a list of its items.

## The theme balance file is layered, not replaced

`data/levels/Standard.sam` is read first and the theme's own `data/levels/<theme>/Standard.sam` is read
**over the top of it, key by key**.

> This matters: the global file names 362 keys and Jungle's names 94, so a loader that reads only the
> theme's file silently loses the 286 keys it never mentions — every peep constant, and all the staff
> and ride economics. Nothing appears missing, because they are merely absent.

`Easy_Standard.sam` is a third pass. In the Jungle it overrides four `LoanInfo[n].Lendername` values and
introduces nothing, so reading it is an assertion that the game is in easy mode.

## Identifying an item

Every item's SAM carries an `Info.Id`, and the number is banded by category:

| Band   | Category    |
| ------ | ----------- |
| `11xx` | Rides       |
| `12xx` | Shops       |
| `13xx` | Sideshows   |
| `14xx` | Features    |
| `15xx` | Upgrades    |
| `16xx` | Fixed items |

The Jungle's fixed items are `1600` Bus, `1601` Gates, `1602` Seaplane, `1603` Lights, `1604` Ferry and
`1605` End. A theme's `global.sam` names its gate by that number, as `ParkName.GateObjectId`.

> WAD members are RefPack-compressed, so searching a `.wad` for `Info.Id` with a plain text search
> returns fragments of the surrounding compression tokens rather than the value. Decompress the member
> first — see [Archives](/formats/wad/).

## Fixed items

A handful of items are placed by the game rather than by the player, and are marked by two keys:

| Key                    | Meaning                                                                 |
| ---------------------- | ----------------------------------------------------------------------- |
| `Info.WhichUIType 4`   | Not shown in the build interface                                        |
| `Info.DontApplyOffset` | `1` — the item's animation plays relative to world `(0,0)`, not its own position |

Those models are authored with **world coordinates already in their node transforms**, so they are
loaded at the origin rather than placed. A model's bounding boxes will not show this: they are
node-local and describe only how large each mesh is.

An item may also override its own footprint and where that footprint sits:

| Key                                  | Jungle gate |
| ------------------------------------ | ----------- |
| `Info.EngineMapOffsetOverrideX`      | `45`        |
| `Info.EngineMapOffsetOverrideY`      | `16`        |
| `Info.EngineFootprintWidthOverride`  | `6`         |
| `Info.EngineFootprintHeightOverride` | `3`         |

The cell count those give agrees with the item's [footprint file](/formats/hmp/).

## Info.Shape is a block, not a value

Most keys are `key <whitespace> value`, but a few are followed by a fenced block: a line of three
dashes, the content, and another line of three dashes. `Info.Shape` is one, and `Info.Hoarding` another.

```text
Info.Shape
---
*S*
***
***
*2*
---
```

A parser that reads a value as the single word after the key returns `---` for these and leaves the
picture behind as unparsed junk, which is easy not to notice.

The picture is the item's footprint, drawn top-down. **The footprint is the box the picture is drawn in
— its widest row by its number of rows — not the number of marks inside it**; see the
[footprint file](/formats/hmp/), which agrees with that reading on every item and with the other on very
few.

### What each character means

Each character is a cell **kind**, looked up in a nineteen-row table inside the executable. The ways in
and out are laid out like a numeric keypad: `8 6 2 4` are entrances facing north, east, south and west,
and `N E S W` the exits facing the same. **So `2` is the entrance and `S` is an exit** — not a second
entrance, and not the entrance itself.

| Character | Kind | Facing | Meaning |
| --------- | ---- | ------ | ------- |
| `.` | 0 | — | an empty cell inside the box |
| `*` | 4 | — | the body of the item |
| `Q` | 3 | — | a queue cell |
| `@` | 1 | — | a path cell |
| `+` | 11 | — | track |
| `#` | 16 | — | track |
| `D` | 23 | — | track station |
| `>` `<` | 23 | east, west | track station, facing |
| `8` `6` `2` `4` | 9 | north, east, south, west | **entrance** |
| `O` | 9 | north | entrance |
| `N` `E` `S` `W` | 10 | north, east, south, west | **exit** |
| `X` | 10 | north | exit |

"Facing" is a compass bit: north `0x01` is −y, east `0x04` +x, south `0x10` +y and west `0x40` −x.
**Because the rows are flipped (below), +y runs UP the picture as drawn**, so south points toward its
top. Read that way, a facing is the direction a guest travels through the cell: the Belly Bounce's `2`
sits on its bottom edge and is walked into upward, and its `S` sits on its top edge and is walked out of
upward.

What the shipped items actually use, across all 274 shape blocks in the four themes: `*`, `.`, `+`, `<`,
`>`, `2`, `S`, `N` and `E`. **Every entrance is a `2`**, 137 of them, never more than one per item. The
72 exits are 44 `S`, 26 `N` and 2 `E`, again never more than one per item, and every item with an exit
also has an entrance. No shipped picture contains a space, a tab, a blank line or a character outside
the table.

### How the picture is read

> **The rows are read upside down.** Once the closing dashes are reached, the reader swaps the first row
> with the last, the second with the second-last, and so on. Row 0 of the stored grid is the **last**
> row drawn. This matters for every item whose entrance is not on its middle row: the Jungle's Staff
> Room (`**` over `*2`) enters on row 0, not row 1.

- A space is skipped: it is not a cell and takes no column.
- A blank line is still a row, with no cells in it.
- Any character not in the table makes the engine refuse the whole picture.
- A row may hold at most 20 cells, and the picture at most 20 rows.

The entrance is the **first** kind-9 cell and the exit the first kind-10 cell, searching **column by
column** from the left and each column from row 0. Positions are the column and row in the flipped grid.
Two fallbacks follow:

- **No exit:** the exit is put on the entrance cell and takes the entrance's facing. This is the common
  case: ten of the Jungle park's eleven placed objects have their exit on their entrance.
- **No entrance:** both are put at column 0, row 0, facing north and south respectively, even if the
  picture has an exit character.

Every placed object in the Jungle's shipped park has the entry and exit cells this reading predicts,
once each is turned by its placement angle — all fourteen of them.

## An item ships only its own art

An item's WAD carries `textures/` and `stexture/` folders, but they hold only the art unique to that
item. Everything else comes from the theme's shared archives, which sit beside the items in
`data/levels/<theme>/`:

| Archive | Pairs with | Contents |
| ------- | ---------- | -------- |
| `sharetex.wad` | `textures/` | full-size shared textures |
| `ssharete.wad` | `stexture/` | the same set at lower detail |

All four themes ship both, with 116 members each. The effect is easy to underestimate: the eleven
objects the Jungle's shipped park places name **40 textures that are in none of their own WADs**, so a
loader that looks only beside the model finds no art for most of every item's surfaces. Resolve a
material by looking in the item's own folder first and the theme's shared archive second.

## The particle effects an item gives off

Four keys each name a particle effect by number. The category files set them, with the same values in
all four themes:

| Key                          | Rides | Sideshows | Shops | Features | Upgrades |
| ---------------------------- | ----- | --------- | ----- | -------- | -------- |
| `Info.CreateParticleEffect`  | 79    | 80        | 81    | 82       | 80       |
| `Info.DestroyParticleEffect` | 75    | 76        | 77    | 78       | 76       |
| `Info.RepairParticleEffect`  | 51    | —         | —     | —        | —        |
| `Info.UpgradeParticleEffect` | 89    | —         | —     | —        | —        |

An item's own file overrides `Create` and `Destroy` together in **69 of the 138 feature archives**, and
in no ride, shop, sideshow or upgrade archive. **24 of the 69 set both to 0**, which is no effect at
all: the bus, the ferry, the seaplane, the gates, the traffic lights and `end`, in every theme. Of the
rest, 36 take the shops' pair (81/77) and 5 the rides' (79/75). Three mix the shops' and the features'
values - the Jungle's Large Tree and Staff Room declare 82/77, its Lava Fountain 81/78 - and its Mammoth
Fountain restates the features' own 82/78. So an item's effect is its own value where it declares one and
its category's otherwise; a reader that takes only the category gives the Jungle's Round Fountain 78
where it declares 77.

## Where the approach is placed

A theme's `Standard.sam` also states the shape of the playable map and the cells the fixed approach
occupies. Jungle's values:

| Key                                  | Value   |
| ------------------------------------ | ------- |
| `MapInfo.HeightfieldXStart` / `YStart` | `0`, `0` |
| `MapInfo.HeightfieldWidth` / `Height` | `95`, `84` — so 96×85 cells |
| `MapInfo.FixedItemOriginX` / `OriginY` | `48`, `17` — "the origin from which fixed items (buses, gates) are placed" |
| `FixedItemInfo.EntranceAPos` / `BPos` | `(47,17)`, `(48,17)` |
| `FixedItemInfo.TicketBoothAPos` / `BPos` | `(47,13)`, `(48,13)` |
| `FixedItemInfo.CrossingParkSideAPos` / `BPos` | `(47,9)`, `(48,9)` |
| `FixedItemInfo.CrossingBSSideAPos` / `BPos` | `(47,5)`, `(48,5)` |
| `FixedItemInfo.BusStopAPos` / `BPos` | `(42,5)`, `(53,5)` |

A cell is 10 world units, so a cell boundary lies at ten times the cell number.
