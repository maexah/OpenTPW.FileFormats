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

### A key may name several fields at once

A key is a **group**, an optional `[n]` subscript, and then **one or more field names separated by
dots** — and the line supplies one value for each name, in order:

```text
PeepTypes[0].PreferredExcitement.StartingCash.BoredomThreshold	80	 300	40
```

That line defines three settings, so `PeepTypes[0].StartingCash` is `300`. The game's parser collects
the dotted names into a table of its own, stops at **sixteen** — the executable carries the string
`Too many fields have been specified` for the seventeenth — and then runs its value loop exactly as many
times as there are names, naming the offending field if one of them will not resolve.

> A reader that takes the line as one key and one value keeps the `80`, drops the rest, and reports
> nothing at all. Every name after the first then simply appears not to exist, and every caller quietly
> receives its fallback instead. Sixteen lines in the whole game are written this way, all of them
> `PeepTypes`, and between them they hold the starting money and the boredom threshold for all eight
> kinds of guest.

It is also why the **unmarked prose** these files trail their values with does no harm: the loop takes
as many values as the key named fields and never looks at the rest of the line, so
`PeepInfo.ExitLevel  120  starting value for the ExitLevel counter` reads `120` and stops there.

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

### The balance file is where the simulation's constants live

The global file is not merely bigger than a theme's; it is the only place most of the simulation is
tuned. Its groups are `PeepInfo` (33 keys) and `PeepTypes[0..7]`, `StaffPoolInfo`, `Arrival`,
`AllStaffConstants`, `PerGradeStaffConsts[0..4]`, `PerTypeStaffConsts[0..4]`, the four
`*ConstsPerGrade`, `BankAccountInfo`, `LoanInfo[0..7]`, `Research`, `ResearchTech`,
`ResearchCategories`, `Costs`, `Challenges` and `GoldenTicketGlobal`, alongside the
`MapInfo`/`FixedItemInfo`/`ThemeEngine`/`Seasons`/`Weather` groups a theme does override.

Many of the keys carry a trailing comment explaining themselves, for example
`PeepInfo.ToiletDesparate 100` — *"toilet level above which peep is 'desperate'"*.

> **These constants are not in the executable, and looking for them there is misleading.** The engine
> reads them from a block of memory that is **entirely zero in the image on disk**, filled at load by
> the balance parser — the binary names `BalanceLoader.cpp` in its error strings. Anyone reverse
> engineering the simulation will find the peep code reading constants that all appear to be `0`; they
> arrive from this file.

`PeepTypes` is eight rows of `PreferredExcitement . StartingCash . BoredomThreshold`, and a peep is
given a type at random when it is made, which then selects its starting money and the kind of ride it
enjoys.

A theme barely touches any of this: across all four themes the only peep key overridden anywhere is
`PeepInfo.ExcitementToCostDivisor`, which Fantasy and Space raise from `4` to `5`. Jungle and Halloween
override none of it.

### RegionFX: what a thing does to the cells around it

Eight numbered effects say how a placed object, or a member of staff, changes the ground near it. They
are the route by which scenery reaches a guest at all:

| Key | Meaning |
| --- | ------- |
| `RegionFX[n].Radius` | How many cells out the effect reaches |
| `RegionFX[n].Happiness` | Added to the happiness of a guest standing there |
| `RegionFX[n].Illness` | Added to their illness |
| `RegionFX[n].Hunger` | Added to their hunger — a negative value *reduces* it, and the file says so in a comment |
| `RegionFX[n].Security` | The cell's security level, which decides whether a guard is sent after a vandal |
| `RegionFX[n].Attraction` | How strongly the cell draws people |

The engine keeps a **ten-byte record for every map cell** — five shorts, in exactly the order above once
the radius is set aside — and stamps an effect into each cell within `Radius`, every value divided by
the **Manhattan distance plus one**. Placing a thing adds those amounts and removing it subtracts the
same ones, so a thing that *moves* does both in turn: staff carry their effect around the park with them.

> **Two instruments agree on the order, which is why it can be trusted.** The keys appear in the order
> the code reads the five shorts; and the fourth short is independently identified as the security level
> by the routine that sweeps all 16,384 cells to report what percentage of the park is covered, and by
> the one that prints `There's no security level on this cell` before setting a guard on a vandal.

Which of the eight a given thing stamps is chosen in code rather than named in the item's own file. No
theme overrides any of this, and `Online_Standard.sam` keeps a copy of its own with one value changed.

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

The picture is the item's footprint, drawn top-down. `*` marks a cell, `2` the cell people enter by, and
`S` a cell with its own special meaning to the item. **The footprint is the box the picture is drawn in
— its widest row by its number of rows — not the number of marks inside it**; see the
[footprint file](/formats/hmp/), which agrees with that reading on every item and with the other on very
few.

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
