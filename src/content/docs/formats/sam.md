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

**A key takes one word per field it names, and the rest of the line is prose.** The shipped files
routinely write an explanation after the value with no comment marker at all - `Info.IsChoosable 1 People
CAN use this` - so a reader that takes the remainder of the line as the value gets a sentence where it
expected a number.

Most keys name a single field and so take a single word. **But a key may name several fields, separated
by dots, and then it takes that many values in order:**

```
PeepTypes[0].PreferredExcitement.StartingCash.BoredomThreshold 80 300 40
```

That is three settings rather than one - `PreferredExcitement` 80, `StartingCash` 300 and
`BoredomThreshold` 40. A reader that takes only the first word after the key gets the first field right
and silently drops the others, which is what makes the peep constants look as though they are missing
from a file that states them plainly.

## An item's description is an override, not a whole description

Buildable items - rides, shops, sideshows and features - each carry a `.sam` inside their own `.wad`, and
the file calls itself a "Theme Park 2 Ride Description File" whether it describes a roller coaster or a
litter bin. **But an item's own file states only what differs from its kind.** Each folder carries the
defaults for everything in it:

| Folder | Category file | `Info.WhichUIType` |
| --- | --- | --- |
| `rides` | `Rides.sam` | 0 |
| `shops` | `Shops.sam` | 1 |
| `sideshow` | `SideShow.sam` | 2 |
| `features` | `Features.sam` | 3 |

Read on its own, the jungle's `Bouncy.sam` never says that Belly Bounce is a ride, that people may choose
it, or that it has a queue. All three come from `Rides.sam`. **A reader that opens only the item file gets
those wrong silently**, because the keys are absent rather than contradicted - and the case that proves the
layering matters is the toilet, which is a *feature*, a category whose default reads `Info.IsChoosable 0`
with the comment "People CANNOT choose to use most features in their decision making". `Toilet.sam`
overrides it back to `1`, "People CAN use this".

That in turn means a parser must distinguish **"the item did not say"** from **"the item said nought"**.
Nought is a real answer for every one of these keys, so a value defaulted to zero on absence would silently
suppress the category's.

### The keys a guest's decision is made of

| Key | Meaning |
| --- | --- |
| `Info.WhichUIType` | Which kind - see the table above |
| `Info.IsChoosable` | Whether a guest may be sent here at all |
| `Info.HasQueue` | Whether people queue for it |
| `Info.AttractionValue` | What it adds to the park's draw |
| `Info.NewAttractionDecayTime` | How long it counts as new |
| `UsageInfo.ProvidesRelief` | Set for toilets - "set to 1 for toilets" |
| `UsageInfo.ISIndoors` | Shelter from the rain (the game's own spelling) |
| `UsageInfo.ExcitementLevel` | How exciting it is |
| `UsageInfo.ThirstEffect`, `UsageInfo.HungerEffect` | How much of each need using it **takes away** |
| `UsageInfo.VomitEffect`, `UsageInfo.HappinessEffect`, `UsageInfo.LitterEffect` | How much of each it **adds** |
| `UsageInfo.MinCapacity`, `MaxCapacity`, `MinDuration`, `MaxDuration` | Bounds the engine clamps to - see below |
| `UsageInfo.EntryCellStandPosX`, `…PosY` | Where in its entry cell a guest stands to use it |
| `UsageInfo.ExitCellAppearPosX`, `…PosY` | Where in its exit cell a guest reappears afterwards |
| `Upgrades[n].InitCapacity`, `InitDuration`, … | Each upgrade level's settings |

**Two of these are the text-file side of bits the saved park carries.** `Info.IsChoosable` matches the
catalogue object's `mFlags` bit `0x4`, and `UsageInfo.ProvidesRelief` matches bit `0x1` - checked object
for object against Lost Kingdom's save, where the two sources agree on all fourteen objects: six may be
visited and three are toilets. `Upgrades[0].InitCapacity` and `InitDuration` likewise match the
`mOperatingCapacity` and `mOperatingDuration` that save records for a placed item.

**The five `*Effect` keys are one block, and two of them run the other way.** They are applied together
when a guest finishes using something, each to one of that guest's meters - thirst, hunger, sickness,
happiness and the litter they are carrying - and the files say so themselves in the prose after each
value: "how much thirst to deduct", "how much vomit to add". So thirst and hunger are subtracted while
sickness, happiness and litter are added, and a reader that treats all five alike gets three of them
backwards. The block belongs to shops: `Shops.sam` declares all five at `5` and each of the eight shops
overrides them, while `Rides.sam` and `SideShow.sam` declare none at all - so a ride reading nought here
is the category default showing through rather than a value of its own.

**`MinCapacity` and `MaxCapacity` are enforced, not advisory.** When a ride is opened the engine takes the
capacity it wants, clamps it between these two, writes it into the ride script's own capacity variable and
stores the same number as the save's `mOperatingCapacity` - so the figure a saved park carries is already
the clamped answer, and applying the bounds to it again would apply them twice. `MinDuration` and
`MaxDuration` bound the duration the same way.

**The stand and appear positions are sub-cell offsets**, not cells. A catalogue object records which cell
it is entered from and which it is left by; these four values say whereabouts *within* those cells a guest
should be placed, and the engine range-checks them, complaining "Dodgy X exit point in SAM file" when they
fall outside what it expects.

**An item can be overridden for easy mode too.** Beside `Bouncy.sam` sits `Easy_Bouncy.sam`, changing
`Upgrades[i].WearRate` and `CostOfResearch` - the same `Easy_` prefix convention the theme-level
`Easy_Standard.sam` uses.
