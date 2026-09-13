---
title: Item Footprints (*.hmp)
---

Every buildable item ships an `.hmp` beside its model, inside the item's own WAD and named after it —
`gates.wad` holds `gates.hmp`, `toilet.wad` holds `toilet.hmp`. The file describes the patch of ground
the item occupies.

Its **size alone gives the number of cells the item covers**, which is often all an implementer needs:

```text
size = 48 + (27 * cells)
```

So a one-cell item is 75 bytes, and the park gate — six cells by three — is 534.

## Verified against every item

This was checked against all 71 item WADs the Jungle theme ships, across `features`, `rides`, `shops`,
`sideshow` and `upgrades`. **Every one fits the rule with no remainder**, giving cell counts of 1, 2, 4,
6, 8, 9, 12, 16, 18, 20 and 25 — all plausible footprint areas.

| Bytes | Cells | Items at this size (Jungle)                                  |
| ----- | ----- | ------------------------------------------------------------ |
| 75    | 1     | `1x1east`, `1x1rck1`, `1x1rck2`, `bus`, `bush`, `camera`, `end`, `ferry`, `lights`, `pelbin`, `seaplane`, `toilet`, … |
| 102   | 2     | `2x1log`                                                     |
| 156   | 4     | `2x2rck`, `bigpalm`, `burger`, `coconut`, `fries`, `icecream`, `staff`, `mystery`, … |
| 210   | 6     | `arc2x3`, `coaster1`, `squark`, `supbog`                     |
| 264   | 8     | `sign1`                                                      |
| 291   | 9     | `balloon`, `cost_shp`, `fountain`, `giftshop`, `hyenas`, `junspray`, `puzzle`, `steak`, `tourride` |
| 372   | 12    | `bouncy`, `gokarts`, `mammtunn`, `minecart`, `totem`         |
| 480   | 16    | `4x4rock`, `lookout`, `monkey`, `mumbo`, `porkpie`, `tvsim`, `volcano`, `wateride`, `watertun` |
| 534   | 18    | `gates`                                                      |
| 588   | 20    | `coaster3`, `incagod`                                        |
| 723   | 25    | `5x5rck`, `5x5rck2`, `bumper`, `spider`                      |

The item names are an independent check on the arithmetic, because several of them state their own
size: `1x1east`, `1x1rck1` and `1x1rck2` come out at one cell, `2x1log` at two, `2x2rck` at four,
`4x4rock` at sixteen, and `5x5rck` and `5x5rck2` at twenty-five.

The park gate is the other check. Its `.hmp` gives eighteen cells, and its `Gates.sam` — a file that
carries no footprint bytes at all — separately declares `Info.EngineFootprintWidthOverride 6` and
`Info.EngineFootprintHeightOverride 3`. Six by three is eighteen.

## Open questions

> The **contents** of the file are not decoded. Only the relationship between its size and the item's
> cell count is established here. The 48-byte header and the 27 bytes per cell have not been read.

- What the 48-byte header holds is unknown.
- What each cell's 27 bytes hold is unknown. The extension suggests a height map — items do deform the
  ground they sit on — but nothing here confirms that, and it should not be assumed.
- **Only the area is recoverable from the size, not the shape.** Twelve cells could be 4×3 or 6×2. The
  aspect comes from the item's [SAM file](/formats/sam/) instead: the `Info.Shape` ASCII grid, or the
  `Info.EngineFootprintWidthOverride` / `Info.EngineFootprintHeightOverride` keys where an item
  overrides its own footprint.
