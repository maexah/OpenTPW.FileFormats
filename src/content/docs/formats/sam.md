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

## Item descriptions, and where an item's real values live

Every buildable item carries one, which the file itself calls a "Theme Park 2 Ride Description File"
whether it describes a ride, a shop, a sideshow or a litter bin.

> **An item's own file is an OVERRIDE, not a whole description — and it lives INSIDE the item's `.wad`.**
> Each folder carries a category default (`rides/Rides.sam`, `shops/Shops.sam`, `sideshow/SideShow.sam`,
> `features/Features.sam`) holding the values for everything of that kind, and an item states only what
> differs. Those category files sit loose on disk; **the per-item overrides do not**, so a grep of the
> installed game folder sees the defaults and never the values actually in force. Read on its own,
> `Bouncy.sam` does not say it is a ride, does not say people may use it, and does not say it has a queue.

That trap is worth stating plainly because it changes answers rather than hiding them. Jungle's
`shops/Shops.sam` declares a price of 10 and a cost of goods of 5; the Drinks Shop's own file, inside
`coconut.wad`, declares **30** and **20**. The Jungle Spray's category says a cost of goods of 30 and its
own file says **50** — and since the engine derives what winning is worth from cost against price, the
category value alone inverts the sign of the result.

### `UsageInfo` keys, with the values Lost Kingdom ships

Verified against `coconut.wad`, `junspray.wad` and the two category files.

| Key | Drinks Shop (1203) | Jungle Spray (1303) | Category default | Notes |
|---|---|---|---|---|
| `InitPricePerUse` | 30 | 20 | 10 both | What a guest is charged |
| `InitCostOfGoods` | 20 | 50 | 5 shops, 30 sideshow | For a sideshow this is the **prize paid to a winner** |
| `InitChanceOfLoosing` | *(absent)* | 75 | 70, sideshow only | The game's own spelling. **Absent means never loses** |
| `ThirstEffect` | 40 | *(absent)* | 5 shops | Deducted |
| `HungerEffect` | 0 | *(absent)* | 5 shops | Deducted. Explicitly nought — it is a drink |
| `VomitEffect` | 10 | *(absent)* | 5 shops | Added |
| `HappinessEffect` | 5 | *(absent)* | 5 shops | Added |
| `LitterEffect` | 50 | *(absent)* | 5 shops | Added |
| `SpecialIngredient` | 3 | *(absent)* | 0 | `0` none, `1` Fat, `2` Salt, `3` Ice, `4` Sugar |
| `AppearanceEffect` | *(absent)* | *(absent)* | 0 | `1` Balloon, `2` Costume |
| `ShopType` | 4 | *(absent)* | — | |
| `ExcitementLevel` | *(absent)* | 35 | 30 sideshow | |
| `RideHandlesSprite` | 1 | 1 | 0 rides, 0 shops | "If the script handles the person sprite" |
| `NumSimultAnims` | *(absent)* | 3 | 1 | How many animations it may run at once |
| `EntryCellStandPosX` / `Y` | *(absent)* | 0.5 / 0.1 | 0.5 / 0.5 | Sub-cell position, fractions of a cell |
| `ExitCellAppearPosX` / `Y` | *(absent)* | 0.5 / 0.1 | 0.5 / 0.5 | |
| `MinCapacity` / `MaxCapacity` | *(absent)* | *(absent)* | 0/0 shops, 1/3 sideshow | The sideshow file's own comment says "Always zero for sideshows" and is **stale** |

`Info.WhichUIType` is `0` rides, `1` shops, `2` sideshows, `3` features, by the comment beside the key.
`Upgrades[0].InitCapacity` is declared **twice** in `Coconut.sam`, both times 1, overriding the category's 10.

> **`QueueSizeInCells` appears in no `.sam` anywhere.** It is a save-record field, and the engine
> recomputes it by walking the map rather than reading it — so an item declaring no queue is not an item
> that cannot be queued for. See [saves](/formats/saves/) for the record.

### `Info.Shape` is a block, not a value

`Info.Shape` is followed by a row of dashes, an ASCII picture of the item's footprint, and another row of
dashes. A parser reading "the single word after the key" gets `---` and leaves the picture as junk.
`Info.Hoarding` takes the same block form. **The footprint is the grid's bounding box, not the cells
drawn in it**: `4x4rock` draws fourteen stars inside a four-by-four box, and `ground` draws none at all,
yet every one of Jungle's seventy items has a `.hmp` whose length is width times height.

Each item's easy-mode override sits beside it as `Easy_<name>.sam`. For every shop and sideshow in Jungle
that file is a single line, `# empty`.
