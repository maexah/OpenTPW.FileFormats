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

**A value is the first word after the key, and the rest of the line is prose.** The shipped files
routinely write an explanation after the value with no comment marker at all - `Info.IsChoosable 1 People
CAN use this` - so a reader that takes the remainder of the line as the value gets a sentence where it
expected a number.

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
| `UsageInfo.ThirstEffect`, `UsageInfo.HungerEffect` | How much of each need using it takes away |
| `UsageInfo.MinCapacity`, `MaxCapacity`, `MinDuration`, `MaxDuration` | Bounds on how it may be operated |
| `Upgrades[n].InitCapacity`, `InitDuration`, … | Each upgrade level's settings |

**Two of these are the text-file side of bits the saved park carries.** `Info.IsChoosable` matches the
catalogue object's `mFlags` bit `0x4`, and `UsageInfo.ProvidesRelief` matches bit `0x1` - checked object
for object against Lost Kingdom's save, where the two sources agree on all fourteen objects: six may be
visited and three are toilets. `Upgrades[0].InitCapacity` and `InitDuration` likewise match the
`mOperatingCapacity` and `mOperatingDuration` that save records for a placed item.

**An item can be overridden for easy mode too.** Beside `Bouncy.sam` sits `Easy_Bouncy.sam`, changing
`Upgrades[i].WearRate` and `CostOfResearch` - the same `Easy_` prefix convention the theme-level
`Easy_Standard.sam` uses.
