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
expected a number. The game's own parser takes as many values as the key names fields and never looks at
the rest of the line, so `PeepInfo.ExitLevel  120  starting value for the ExitLevel counter, in SECONDS`
reads `120` and stops there.

Most keys name a single field and so take a single word. **But a key is a group, an optional `[n]`
subscript, and then one or more field names separated by dots, and a key that names several fields takes
that many values in order:**

```text
PeepTypes[0].PreferredExcitement.StartingCash.BoredomThreshold	80	 300	40
```

That is three settings rather than one - `PreferredExcitement` 80, `StartingCash` 300 and
`BoredomThreshold` 40. The game's parser collects the dotted names into a table of its own, stops at
**sixteen** - the executable carries the string `Too many fields have been specified` for the seventeenth -
and then runs its value loop exactly as many times as there are names, naming the offending field if one
of them will not resolve. A reader that takes only the first word after the key gets the first field right
and silently drops the others, which is what makes the peep constants look as though they are missing
from a file that states them plainly: every name after the first simply appears not to exist, and every
caller quietly receives its fallback instead. Sixteen lines in the whole game are written this way, all of
them `PeepTypes`, and between them they hold the starting money and the boredom threshold for all eight
kinds of guest.

**Only the two global files set a `PeepTypes` row.** Of the 56 loose `.sam` files under `data/`, and the 344
inside the game's `.wad` files, only the two global balance files name `PeepTypes[n]`, each rows 0 to 7
(`Standard.sam` lines 56-63, `Online_Standard.sam` lines 53-60), and a park reads one or the other (see
"The theme balance file is layered", below), so every shipped park has eight kinds of guest. A guest is
given a kind at random when it is made, and the kind selects its starting money, its boredom threshold and
the excitement it prefers in a ride. The two files differ: `Online_Standard.sam`'s row 6 prefers 65
excitement where `Standard.sam`'s prefers 45, its row 3's boredom threshold is 20 where `Standard.sam`'s is
35, and its starting cash runs 400 to 1000 where `Standard.sam`'s runs 300 to 750. How the game counts the
rows it was given is the executable's business (OpenTPW's `docs/exe/ride-operation.md`, "What a thing is
worth to a guest").

## Where they are found

- **Loose, under `data/`** - `Challenges.sam`, `high.sam`, `med.sam`, `low.sam`, `sound.sam`, `Online.sam`,
  `onlineoverride.sam`, `_Resolution.sam` and `Advisor/Advisor.sam`.
- **Global balance, under `data/levels/`** - `Standard.sam` and `Online_Standard.sam`.
- **Per theme, under `data/levels/<theme>/`** - `Standard.sam` (the theme's balance file),
  `Online_Standard.sam`, `global.sam` (what the lobby needs before the theme is loaded), `Hoardings.sam` and
  `EndHoardings.sam`; the Jungle alone also ships `Easy_Standard.sam`.
- **Per category, in each content folder** - `features/Features.sam`, `rides/Rides.sam`, `shops/Shops.sam`,
  `sideshow/SideShow.sam`, `upgrades/Upgrades.sam`, and `rides/Online_Rides.sam` beside `Rides.sam`. These
  hold defaults for a whole category rather than a list of its items.
- **Per item, inside the item's own WAD and named after it** - `gates.wad` holds `Gates.sam`. Beside it the
  Jungle's items may carry an `Easy_<name>.sam` (`Easy_Gates.sam` is just `# Empty`); the four themes' tour
  and water rides carry an `Online_<name>.sam` (eight in all, each setting only `Upgrades[n].WearRate`); and
  the twelve coaster WADs, three per theme, carry a `coaster.sam` describing the track itself
  (`sCoasterType`, `sTrainType`, `asCarTypes`, `asTextureData` and the cross-section tables).

That is 56 loose files and 344 inside WADs: 274 item descriptions, 50 `Easy_` files, 8 `Online_` files and
12 `coaster.sam`.

**A `coaster.sam`'s tables.** All twelve carry `asCarTypes`, `asTextureData`, `asCrossSections`,
`asCrossSectionSelects`, `asCrossSectionEdges1`, `asCrossSectionPoints1` and `asPylonControls`; eight also carry
`asCrossSectionEdges2` (`b_drip`, `candy_c`, `c_hade`, `coasta`, `c_scat`, `coaster1`, `minecart`, `megacost`), two
`asCrossSectionPoints2` (`coaster1`, `megacost`) and one `Points3` and `Edges3` (`coaster1`). The schema is not decoded
from the executable; what the files show:

- `asCarTypes[n]` names the car: `pcMeshFilename` (Chak Atak's, `coaster1`, is `CrocCar`, with `uiCarTypeUniqueID`
  1188 and `fYOffset` 0.5). The car's `M1`..`M3` files beside it are its animation channels, as any model's are.
- `asCrossSectionPoints1[n]` are the track's profile, each `fX`, `fY`, `fU` and a normal `fNx`, `fNy`. Chak Atak's
  water channel: (-4, 2.5) U 1 N (-1, 1); (-3, 2) U 0.1 N (0, 1); (3, 2) U 0.9 N (0, 1); (4, 2.5) U 1 N (1, 1);
  (0, -1.5) U 0 N (0, -1); then points 1 and 2 again.
- `asCrossSectionEdges1[n]` join the points, each with a `usTexture` into `asTextureData` (a name and an
  `fScrollRate`). Chak Atak's edge 1 joins points 5 and 6 with texture 1, `water2.tga` scrolling at 1.2; its other
  edges use texture 0, `trak_sec3.tga`. Some edges join the same two points both ways.
- `asCrossSections[n]` pick a point block and an edge block (`usCSPointBlockIndex`, `usCSEdgeBlockIndex`,
  `usTrackEdge`, `usBlendToCSPointBlockIndex`, `fBlendLength`); `asCrossSectionSelects[n]` choose one by the
  track's pitch (`usCrossSectionIndex`, `fPitchRangeMin`, `fPitchRangeMax`, `usBlendFrom/ToCrossSectionIndex`).
- `asPylonControls[n]` name the supports: `pcMeshFilename` `StdPylon` (`uiPylonTypeUniqueID` 1186), with a radius,
  an incline multiplier, the most pylons stacked, a least and greatest height and three costs.

**The track's own textures are in a `GTexture` folder.** The twelve coaster WADs, and no other, ship one: 42 members,
a `qickload.txt` in each (`coaster1`'s lists `Trak_sec2.wct` and `Trak_sec3.wct`). `coaster.sam` names them `.tga`
where the folder holds `.wct`. The engine names the folder itself (OpenTPW's `docs/exe/park.md`, "Buildable items: the per-item archive").

The 50 `Easy_` files are all the Jungle's, one in each of 50 of its 70 item WADs. Twelve rides' set
`Upgrades[0..2].WearRate` (3, 2, 1; 1, 1, 1 for `minecart` and `wateride`) and `Upgrades[1..2].CostOfResearch`
`0`, and `minecart`'s also `Research.Group` `2`; the other 38 hold comments only. The 20 WADs without one are
`coaster1`, `coaster3`, `incagod`, `tourride`, `volcano`, `giftshop`, `steak`, `arc2x3`, `5x5rck`, `5x5rck2`,
`lavspurt`, `lure`, `mamfount`, `speaker2`-`4`, `statue2` and the three upgrades. Their absence matters: in Instant
Action the game catalogues no item whose WAD lacks one (OpenTPW's `docs/exe/park-engine.md`, "How a key finds its
global").

## The theme balance file is layered, not replaced

`data/levels/Standard.sam` is read first and the theme's own `data/levels/<theme>/Standard.sam` is read
**over the top of it, key by key**. An online game reads `data/levels/Online_Standard.sam` and then the
theme's `Online_Standard.sam` instead, and an Instant Action game reads a third file over the first two,
the theme's `Easy_Standard.sam` (the executable's `FUN_005156a0`; OpenTPW's `docs/exe/park-engine.md`,
"Standard.sam is a generic Bullfrog balance file").

> This matters: the global file names 362 keys and Jungle's names 94, 76 of them also in the global file,
> so a loader that reads only the theme's file silently loses the 286 keys it never mentions — every peep
> constant, and all the staff and ride economics. Nothing appears missing, because they are merely absent.

For a theme without one the executable logs `There is no Easy_standard.sam file for this theme - This is
not critical`. The Jungle's `Easy_Standard.sam` names 68 keys, every one of them already in the global
file, and introduces no group of its own. It sets no `MapInfo`. It restates the value already in force for
31 of them and changes it for 37, which is what makes an Instant Action park easy:

| Keys | In force before it | `Easy_Standard.sam` |
| ---- | ------------------ | ------------------- |
| `BankAccountInfo.InitialCash` | `50000` | `100000` |
| `LoanInfo[0..7].APRInPercent` | `18` to `23` | `0` |
| `LoanInfo[1,3,4,6].Lendername` | the theme's `3`, `10`, `6`, `11` | the global file's `1`, `3`, `4`, `6` |
| `PeepInfo.AveragePriceMultiplier`, `ExpensivePriceMultiplier` | `1.25`, `2.0` | `1.5`, `2.5` |
| `Arrival.PointsPerVisitor` | `6` | `5` |
| `PerGradeStaffConsts[0..4].BaseWage` | `4`, `5`, `6`, `8`, `12` | `3`, `4`, `5`, `7`, `9` |
| `PerTypeStaffConsts[0..4].PayMultiplier` | `10`, `30`, `15`, `20`, `35` | `9`, `23`, `12`, `15`, `25` |
| `ResearcherConstsPerGrade[0..4].ResearchAbility` | `2` to `6` | `6`, `9`, `12`, `16`, `20` |
| `StaffPoolInfo.MaxResearchersInPark` | `10` | `6` |
| `ResearchCategories[3].Effort`, `[4].Effort` | `15`, `10` | `25`, `0` |
| `Costs.MapCell`, `KartTrackCell`, `WaterTrackCell` | `100`, `400`, `500` | `10`, `200`, `250` |

### The balance file is where the simulation's constants live

The global file is not merely bigger than a theme's; it is the only place most of the simulation is
tuned. Its groups are `PeepInfo` (33 keys) and `PeepTypes[0..7]`, `StaffPoolInfo`, `Arrival`,
`AllStaffConstants`, `PerGradeStaffConsts[0..4]`, `PerTypeStaffConsts[0..4]`, the five `*ConstsPerGrade`
(mechanic, handyman, guard, entertainer and researcher), `BankAccountInfo`, `LoanInfo[0..7]`, `Research`,
`ResearchTech`, `ResearchCategories`, `Costs`, `Challenges`, `GoldenTicketGlobal` and `RegionFX[0..7]`,
alongside the `MapInfo`, `FixedItemInfo`, `ThemeEngine`, `Seasons`, `Weather`, `WeatherEffects`,
`LightNormal`, `WaterStartPos`, `SkyObjects` (with `NumSkyObjects`) and `Sound` groups a theme does
override. A theme's file adds
three groups the global file lacks: `ThemeAdvisorCostumes`, `GoldenTicketLocal` and `ChallengesInThisLevel`.

**The golden-ticket thresholds.** `GoldenTicketLocal` is in the four themes' `Standard.sam` only; `GoldenTicketGlobal`
in the global file only; there is no `GoldenTicketSecret` group, and no `Easy_` or `Online_` file sets a ticket key.
How the game tests them is OpenTPW's `docs/exe/ride-operation.md`, "Golden tickets".

| `GoldenTicketLocal.` | jungle | hallow | fantasy | space |
|---|---|---|---|---|
| `Visitors` | 100 | 100 | 2500 | 3000 |
| `PeopleInPark` | 200 | 200 | 300 | 350 |
| `Happiness` | 75 | 75 | 80 | 85 |
| `AtLeastThisManyHappyPeople` | 150 | 150 | 150 | 150 |
| `ProfitYear` | 15000 | 15000 | 20000 | 30000 |
| `RecentVisitors` | 350 | 350 | 400 | 500 |
| `RecentVisitorMonths` | 6 | 6 | 6 | 6 |

`GoldenTicketGlobal`: `CoasterHeight` 105 (*"Gotta build this one on high ground. Jungle or fantasy it is"*),
`GokartExcitement` 90, `WaterLength` 50, `MinCellsOwned` 3000 (which the game never reads) and `MinCellsCovered` 2000.

**The challenges.** The global `Challenges` group holds `DaysAfterCompletedChallenge` 270,
`DaysAfterDeclinedChallenge` 270, `DaysUntilFirstChallenge` 540 (game days), `DeclinesToForfeit` 2 and
`ShortTimeLeftWarningAt` 20 (a percentage of the time left); no theme overrides them. A theme's
`ChallengesInThisLevel[n].ChallengeType` is, despite its name, a **1-based index into `Challenges.sam`'s
`Challenges[]`**, not a `Type`, up to 20 per level (the game copies them into its slots in this order, as
[saves](/formats/saves/), "The challenge manager (model 19)", measures):

| Theme | `ChallengesInThisLevel` |
|---|---|
| jungle | 1, 15, 6, 18, 12, 7, 8, 9 |
| hallow | 2, 21, 23, 24, 17, 4, 20, 22, 5 |
| fantasy | 28, 10, 26, 27, 11, 13, 30, 29, 31, 32 |
| space | 3, 6, 16, 25, 33, 34, 35, 14 |

Together they cover 1 to 35 but for 19, the *"NEVER CALL THIS CHALLANGE"* entry; 6 is listed twice. A chain
(jungle's 7, 8, 9) is listed whole, not by its head.

`Challenges.sam` itself (above) has 35 entries, each under a comment of EA's naming its goal, and so its `Type`:
1 fries, 2 burgers, 3 drinks, 4 ice creams, 5 gifts, 6 balloons, 7 costumes, 8 restaurant meals, 9 and 10 a
percentage of kids with balloons or costumes, 11 new visitors, 12 shop profit, 13 sideshow profit, 15 go-kart
crossroads, 16 rides researched, 17 and 20 peeps onto a named ride, 18 build a named coaster, 19 its loops, 21 a kart
track's sections, 24 toilet cleanliness, 25 all staff's happiness, 27 a handyman's happiness, 28 a ride upgraded to a
level, 29 average staff skill, 30 build a named item, 31 an upgrade or a named item, 33 a named upgrade. The executable
confirms 1-8, 12 and 13 by what it posts (6 counts balloons sold, not held); the rest are the comments' word.
`FollowupType` names the next entry's `Type`.

Many of the keys carry a trailing comment explaining themselves, for example
`PeepInfo.ToiletDesparate 100` — *"toilet level above which peep is 'desperate'"*.

> **These constants are not in the executable, and looking for them there is misleading.** The engine
> reads them from a block of memory that is **entirely zero in the image on disk**, filled at load by
> the balance parser — the binary names `BalanceLoader.cpp` in its error strings. Anyone reverse
> engineering the simulation will find the peep code reading constants that all appear to be `0`; they
> arrive from this file.

A theme barely touches any of this. Of the peep keys, a theme's `Standard.sam` overrides only
`PeepInfo.ExcitementToCostDivisor`, which Fantasy and Space raise from `4` to `5`; Jungle and Halloween
override none, and no theme's `Online_Standard.sam` names a peep key at all. Beyond the scenery groups,
its own three groups and that one peep key, a theme's `Standard.sam` sets only four
`LoanInfo[n].Lendername` values (three in Halloween) and, in Halloween and the Jungle, the five
`StaffPoolInfo.AvgGradeOf*` grades, all to `2`.

### RegionFX: what a thing does to the cells around it

Eight numbered effects say how a placed object, or a member of staff, changes the ground near it. They
are the route by which scenery reaches a guest at all:

| Key | Meaning |
| --- | ------- |
| `RegionFX[n].Radius` | How many cells out the effect reaches |
| `RegionFX[n].Happiness` | Added to the happiness of a guest standing there |
| `RegionFX[n].Illness` | Added to their illness |
| `RegionFX[n].Hunger` | Added to their hunger — a negative value *reduces* it, and the file says so in a comment (`// decreases hunger`) |
| `RegionFX[n].Security` | The cell's security level, which decides whether a guard is sent after a vandal |
| `RegionFX[n].Attraction` | How strongly the cell draws people |

The global file heads each of the eight with a comment naming what it is for: 0 Entertainer, 1 Clean
Toilet, 2 Vomit puddle, 3 Guard, 4 Security Camera, 5 Stink Bomb, 6 Dirty Toilet and 7 Fireworks.

The engine keeps a **ten-byte record for every map cell** — five shorts, in exactly the order above once
the radius is set aside — and stamps an effect into each cell within `Radius`, every value divided by
the **Manhattan distance plus one**. Placing a thing adds those amounts and removing it subtracts the
same ones, so a thing that *moves* does both in turn: staff carry their effect around the park with them.

> **Two instruments agree on the order, which is why it can be trusted.** The keys appear in the order
> the code reads the five shorts; and the fourth short is independently identified as the security level
> by the routine that sweeps all 16,384 cells to report what percentage of the park is covered, and by
> the one that prints `There's no security level on this cell` before setting a guard on a vandal.

Which of the eight a given thing stamps is chosen in code rather than named in the item's own file. No
theme overrides any of this, and `Online_Standard.sam` keeps a copy of its own with two values changed:
`RegionFX[2].Illness`, the vomit puddle's, is `2` where `Standard.sam` has `1`, and `RegionFX[5].Illness`,
the stink bomb's, is `5` where it has `3`.

## An item's description is an override, not a whole description

Every item - rides, shops, sideshows, features (the fixed items among them) and upgrades, 274 across the
four themes - carries a `.sam` inside its own `.wad`, and the file calls itself a "Theme Park 2 Ride
Description File" whether it describes a roller coaster or a litter bin. **But an item's own file states
only what differs from its kind.** Each folder carries the defaults for everything in it:

| Folder | Category file | `Info.WhichUIType` |
| --- | --- | --- |
| `rides` | `Rides.sam` | 0 |
| `shops` | `Shops.sam` | 1 |
| `sideshow` | `SideShow.sam` | 2 |
| `features` | `Features.sam` | 3 |
| `upgrades` | `Upgrades.sam` | 0 |

The comment beside the key names the four kinds - "0=rides, 1=shops, 2=sideshows, 3=features" - and a
fifth value, `4`, "Not to be shown in UI", is set by items themselves (see [Fixed items](#fixed-items)).

> **The category files sit loose on disk; the per-item overrides do not.** They live inside the item's
> `.wad`, so a grep of the installed game folder sees the defaults and never the values actually in force;
> [Two of Lost Kingdom's items, key by key](#two-of-lost-kingdoms-items-key-by-key), below, sets the two
> side by side. The difference is not cosmetic: a sideshow's excitement is worked out from its cost of
> goods less its price,
> `20 + trunc( 0.08 × chance of winning × √clamp( CostOfGoods − PricePerUse, 0, 100 ) )` (OpenTPW's
> `docs/exe/ride-operation.md`, "What a thing is worth to a guest"), so at its starting price of 20 the
> category's cost of goods would give the Jungle Spray √10 where its own gives √30.

Read on its own, the jungle's `Bouncy.sam` never says that Belly Bounce is a ride, that people may choose
it, or that it has a queue. All three come from `Rides.sam`. **A reader that opens only the item file gets
those wrong silently**, because the keys are absent rather than contradicted - and the case that proves the
layering matters is the toilet, which is a *feature*, a category whose default reads `Info.IsChoosable 0`
with the comment "People CANNOT choose to use most features in their decision making". `Toilet.sam`
overrides it back to `1`, "People CAN use this".

That in turn means a parser must distinguish **"the item did not say"** from **"the item said nought"**.
Nought is a real answer for every one of these keys, so a value defaulted to zero on absence would silently
suppress the category's.

The game reads an item's files in order, each over the last: its folder's category file; in an online game
that folder's `Online_<Category>.sam` (only `rides` has one, setting every `WearRate` to `0`); the item's own
`.sam`; then `Online_<name>.sam` in an online game or `Easy_<name>.sam` in Instant Action. A key that no file
sets reads nought, or its lower bound where the executable bounds it, so `Info.NewAttractionDecayTime` is
`1` and `UsageInfo.ExciteFactor` `50` where no file names them (OpenTPW's `docs/exe/park-engine.md`, "How a
key finds its global").

**A line the executable cannot take is not read past: the game quits naming the file.** Every key must be one the
executable's table names, spelt in its case. A whole number is an optional `-` and digits, a float digits with at most
one `.`. A bounded key must lie in `[lo, hi)` - the flags `[0, 2)`, `Info.WhichUIType` `[0, 5)`,
`Info.NewAttractionDecayTime` `[1, 1000)`, `UsageInfo.ExcitementLevel` `[0, 101)`, `UsageInfo.ExciteFactor`
`[50, 200)`, `Research.Group` `[0, 255)` and the rest in that table - and a key that takes nought and up (`Info.Id`,
the costs, prices, capacities, speeds and durations, `Upgrades[i].WearRate`) must not be negative. The effects
(`UsageInfo.ThirstEffect` and its four siblings) and `Bumper.BumperType` may be. None of the Jungle's 128 item
descriptions breaks a rule. The `Coaster.sam` beside three coasters' own file is not an item description: it names
the track's textures, `asTextureData[i].pcTextureFilename` (OpenTPW's `docs/exe/park-engine.md`, "How a key finds
its global").

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
| `UsageInfo.CannotRide` | Whether it cannot be ridden by walking onto its entrance in first person: 0 in every theme's `Rides.sam` and 1 in its `Shops.sam`, `SideShow.sam` and `Features.sam`; `Upgrades.sam` leaves it unset, and none of the 274 items, the 11 upgrades included, overrides it |
| `UsageInfo.ExcitementLevel` | How exciting it is: every `Rides.sam` declares `60` and every `SideShow.sam` `30`, the latter beside the unmarked note "// Do not set to zero. Ben's fault." |
| `UsageInfo.GoldenTicketCost` | What a golden ticket costs to ride it. Nine rides of the 274 `.wad` files set it: `fantasy`'s `dragfly` 4 and `twetours` 2, `hallow`'s `spider` 3 and `tourride` 1, `jungle`'s `tourride` (Jurassic Tours) 1 and `volcano` (Eruption) 3, and `space`'s `scitour` 2, `spawheel` 5 and `station` 3. Of the category files only `hallow/rides/Rides.sam` declares it, at `0` |
| `Bumper.BumperType` | Which bumper vehicle the ride makes. Twelve rides set it, each to a different value from `-1` to `-14` (`-2` and `-3` are unused): every theme's go-karts and water ride, and its bumper ride (`bumper`, or `fantasy`'s `bbugs`); the jungle's Hot Pot `-1`, Dino Karts `-4` and Splish Splash `-5`; the other three bumper rides are hallow's `-6`, fantasy's `bbugs` `-11` and space's `-14`. Every other item reads `0`, which the four `Rides.sam` declare. None of the four bumper rides sets `Upgrades[0].InitSpeed`, so each inherits its `Rides.sam`'s 60, which the game makes the cars' performance (OpenTPW's `docs/exe/park.md`, "How a bumper ride's cars move") |
| `Bumper.NorthXAdjust` .. `Bumper.WestYAdjust` | Eight integers, an x and a y for each of the four turns, which the placer adds to the anchor cell (with a fixed offset of its own per turn) to find a bumper ride's arena centre. 34 files across the four themes set them: all 12 rides with a `BumperType`, and 22 other rides (coasters, track rides and a few more) whose `BumperType` is `0`, declared or inherited; the comment beside `NorthXAdjust` reads "some constants to line up the Track sections with the rest of the co-ordinate system". The Hot Pot's, `-2 1`, `1 3`, `3 0`, `0 -2`, put the centre on the middle cell of its 5 × 5 footprint at every turn (OpenTPW's `docs/exe/park.md`, "Where a bumper ride's cars float") |
| `SupplementalMeshes[n].FileName` | The extra models a ride loads beside its own, by index: what a bumper ride's cars, a go-kart ride's karts (`fantasy`'s and `space`'s list a `gk_shadow` first), a water ride's ring and a tour ride's craft are made of. The jungle's: Dino Karts `gk_blue`, `gk_green`, `gk_orange`, `gk_purple`; Splish Splash `wr_ring`; Jurassic Tours `Bird`. The Hot Pot's are `b_wake.md2` and `b_car.md2`, and its template takes the car from index 1; the other three themes' bumper rides name their car at index 0 (`halcar.md2`, `spacecar.md2`, `fantasy`'s `bbugscar.md2`) |
| `UsageInfo.ThirstEffect`, `UsageInfo.HungerEffect` | How much of each need using it **takes away** |
| `UsageInfo.VomitEffect`, `UsageInfo.HappinessEffect`, `UsageInfo.LitterEffect` | How much of each it **adds** |
| `UsageInfo.MinCapacity`, `MaxCapacity`, `MinDuration`, `MaxDuration` | Bounds the engine clamps to - see below |
| `UsageInfo.EntryCellStandPosX`, `…PosY` | Where in its entry cell a guest stands to use it |
| `UsageInfo.ExitCellAppearPosX`, `…PosY` | Where in its exit cell a guest reappears afterwards |
| `UsageInfo.InitCostOfGoods` | What one use costs the park to provide - and a sideshow's prize |
| `UsageInfo.InitChanceOfLoosing` | A sideshow's chance, in per cent, of not paying out (the game's own spelling); its chance of winning is 100 less. `SideShow.sam` declares `70` in `jungle` and `hallow` and `75` in `fantasy` and `space`, and 15 of the 17 sideshows set their own `75`; the Arcades of `hallow` and `space` take the category's. No shop, ride or feature file declares it, so theirs reads nought and they never lose |
| `UsageInfo.InitPricePerUse` | What a go costs a guest on one just built; the player sets it from then on. `Shops.sam` declares `10` ("sale price") in all four themes and `SideShow.sam` `10` (`hallow`'s `20`), and every one of the 49 shops and sideshows sets its own: the jungle's Drinks Shop `30`, Jungle Spray `20`, Gift Shop `75`. No ride or feature, nor `Rides.sam` or `Features.sam`, declares it |
| `UsageInfo.RipOffOK` | How far over what a thing is worth a guest will still pay, in per cent - see below |
| `UsageInfo.SpecialIngredient` | A shop's goods: "0 = none, 1 = Fat, 2 = Salt, 3 = Ice, 4 = Sugar" |
| `UsageInfo.AppearanceEffect` | What a shop changes about a guest's looks: "1 = Balloon, 2 = Costume" |
| `Upgrades[n].QueueWaitTimeConstant` | Each upgrade level's queue constant, **a float** (how the game parses one: OpenTPW's `docs/exe/park-engine.md`, "How a key finds its global"), which the longest queue a guest will join or stay in is scaled by. Every theme's `Rides.sam` declares `30`, `35` and `40`, and no other category file, nor any `Easy_` or `Online_` file, declares it. 68 of the 76 rides in the four themes set their own at all three levels; the other eight are the four Mystery rides and `fantasy`'s `ccride`, `dragfly` and `flamingo` and `space`'s `wateride`, which take the category's. All 217 declarations, the category files' twelve included, are whole numbers from `3` to `250`; `fantasy`'s `b_drip` states level 0 twice, `50` both times. In the jungle all 17 rides with a queue set their own: the Belly Bounce `130`, `135`, `145`, The Hot Pot `100`, `120`, `140`, the Sun God `3`, `4`, `4` |
| `Upgrades[n].InitCapacity`, `InitDuration`, … | Each upgrade level's settings: every theme's `Rides.sam` gives `InitSpeed` 60, 75 and 90 for the three levels and `InitDuration` 3 at each. Of the 76 rides in the four themes none sets its own `InitSpeed` at any level, and nine set `InitDuration`, one value at all three levels: the Belly Bounce and Bounce On Iggy `30`, The Hot Pot, Pumpkin Castle and Uforia `25`, Jurassic Tours, Tweety Tours, Flightmare Tours and Star Tours `40`; none of the jungle's 50 `Easy_` files sets either |

**Two of these are the text-file side of bits the saved park carries.** `Info.IsChoosable` matches the
catalogue object's `mFlags` bit `0x4`, and `UsageInfo.ProvidesRelief` matches bit `0x1` - checked object
for object against Lost Kingdom's save, where the two sources agree on all fourteen object records (the
eleven placed things and the fixed bus, gates and lights): six may be visited and three are toilets.
`Upgrades[0].InitCapacity` and `InitDuration` likewise match the `mOperatingCapacity` and
`mOperatingDuration` that save records for a placed item.

> **`QueueSizeInCells` appears in no `.sam` anywhere.** It is a save-record field, and the engine
> recomputes it by walking the map rather than reading it — so an item declaring no queue is not an item
> that cannot be queued for. See [saves](/formats/saves/) for the record.

**The five `*Effect` keys are one block, and two of them run the other way.** They are applied together
when a guest finishes using something, each to one of that guest's meters - thirst, hunger, sickness,
happiness and the litter they are carrying - and the files say so themselves in the prose after each
value: "how much thirst to deduct", "how much vomit to add". So thirst and hunger are subtracted while
sickness, happiness and litter are added, and a reader that treats all five alike gets three of them
backwards. The block belongs to shops: `Shops.sam` declares all five at `5` and each of the eight shops
in every theme overrides all five, while `Rides.sam`, `SideShow.sam` and `Features.sam` declare none at
all - so a ride reading nought here is the category default showing through rather than a value of its
own. The one item outside the shops that names any of them is `hallow`'s sideshow `ball`, with
`HappinessEffect` `20`.

**`UsageInfo.RipOffOK` is set by the category files alone.** `Shops.sam` declares `100` and `SideShow.sam`
`250`, the second annotated "%premium peeps willing to pay above 'average win'", identically in all four
themes; no item's own file overrides it (checked across the `.sam` inside every one of the 312 `.wad` files
the game ships), and `Rides.sam` and `Features.sam` do not declare it, so a ride reads nought. It is what
lifts a thing's worth to a guest above its cost of goods before the price is compared with it: without it
the jungle's Drinks Shop can be worth as little as 21 against its price of 30, with it no less than 42.

**`SpecialIngredient` and `AppearanceEffect` are shop keys.** `Shops.sam` declares both at `0`; among the
jungle's shops the Drinks Shop (`Coconut.sam`) sets ingredient `3`, ice, the Burger `1`, the Fries `2` and
the Ice Cream `4`, while the Balloon sets appearance `1` and the Costume Shop `2`. The Steak's value is `0`
beside a comment reading `FAT=1`, and the value is what is read. Nothing outside the `shops` folders declares
either.

**`MinCapacity` and `MaxCapacity` are enforced where they are set.** When a thing is built or opened the
engine takes the capacity it wants, clamps it between these two when they sum above nought, writes it into
the ride script's own capacity variable and stores the same number as the save's `mOperatingCapacity` - so
the figure a saved park carries is already the answer, and applying the bounds to it again would apply them
twice. Every `Rides.sam` declares `1` and `5`, and every `SideShow.sam` `1` and `3`, beside the comment
"Always zero for sideshows", which the values contradict. Every `Shops.sam` declares both `0` ("Always zero
for shops"), so a shop keeps its own `Upgrades[0].InitCapacity`, which all 32 shops set, to `1`, `5` or
`10`. `MinDuration` and `MaxDuration` bound the duration always, so a duration with neither declared is held
to nought. Eight items start away from their own values: four rides that inherit `Rides.sam`'s
`InitDuration` `3` under a `MinDuration` of their own (`fantasy`'s Jelly Bounce and Bumper Bugs and
`hallow`'s Brain Buster `10`, `space`'s Zero G `15`), and four sideshows whose capacity passes `MaxCapacity`
(the Arcades of `fantasy`, `hallow` and `space`, `5` against `SideShow.sam`'s `3`, and `space`'s Giant
Puzzle, the category's `3` against its own `1`).

**The stand and appear positions are sub-cell offsets**, not cells, in fractions of a cell: every theme's
`Rides.sam`, `Shops.sam`, `SideShow.sam` and `Features.sam` declares `0.5` for all four, and its
`Upgrades.sam` none. A catalogue object records which
cell it is entered from and which it is left by; these four values say whereabouts *within* those cells a
guest should be placed, and the engine range-checks them, complaining "Dodgy X exit point in SAM file" when
they fall outside what it expects.

**An item can be overridden for easy mode too.** Beside `Bouncy.sam` sits `Easy_Bouncy.sam`, changing
`Upgrades[i].WearRate` and `CostOfResearch` - the same `Easy_` prefix convention the theme-level
`Easy_Standard.sam` uses, and like it read only in Instant Action. Only the Jungle ships item `Easy_`
files, and not every Jungle item carries one: 20 of the jungle's 70 item folders ship none (the Gift Shop,
the Steak Restaurant, the Arcade, five rides, nine features and the three upgrades), and in Instant Action
an item without one is never catalogued at all (OpenTPW's `docs/exe/park-engine.md`, "How a key finds its
global"). Twelve rides' files set only `Upgrades[i].WearRate`, `CostOfResearch` and, in the minecart's,
`Research.Group`; the other 38, `Easy_mystery.sam` and every shop's and sideshow's among them, are a single
comment line, `# empty` or `# Empty`.

**A file can declare a key twice.** The jungle's `Arc2x3.sam` gives `Upgrades[0].InitCapacity` as `6` and then
`5`, and the later is the one the game keeps; four of its shops repeat `InitCapacity` (the Drinks Shop's
`Coconut.sam` gives `1` twice, over `Shops.sam`'s `10` and its comment "Always one for shops"), the Costume
Shop `SpecialIngredient` and Jurassic Tours both duration bounds, each time with the same value.

### `Info.DoHeadProcessing`: every node the model's lookup table names is kept posed

Set to `1` by six items, and by no category file: the five rides that carry their riders on the
model's head nodes (`hallow`'s `firepit`, `jungle`'s `totem` and `tvsim`, `space`'s `hoverbot` and
`tv_ride`) and `fantasy`'s `bugstv`. With it, the game keeps a position for every node its
[`.md2`](/formats/models/#which-records-have-a-position)'s lookup table gives one (flag `0x10` or
`0x20`, which every record of the six models carries), the heads included. Without it, a childless
node keeps one only under the other conditions that section lists, and the mask `0x580f00` there does
not take in the head flag `0x80` (engine-confirmed: the item loader passes the model loader
`0x400000` for it, and the model loader then marks every record).

### Two of Lost Kingdom's items, key by key

Verified against `coconut.wad`, `junspray.wad` and the Jungle's two category files.

| Key | Drinks Shop (1203) | Jungle Spray (1303) | Category default | Notes |
|---|---|---|---|---|
| `InitPricePerUse` | 30 | 20 | 10 both | What a guest is charged |
| `InitCostOfGoods` | 20 | 50 | 5 shops, 30 sideshow | For a sideshow this is the **prize paid to a winner** |
| `InitChanceOfLoosing` | *(absent)* | 75 | 70, sideshow only | Absent from both the shop and its category, so the Drinks Shop never loses |
| `ThirstEffect` | 40 | *(absent)* | 5 shops | Deducted |
| `HungerEffect` | 0 | *(absent)* | 5 shops | Deducted. Explicitly nought — it is a drink |
| `VomitEffect` | 10 | *(absent)* | 5 shops | Added |
| `HappinessEffect` | 5 | *(absent)* | 5 shops | Added |
| `LitterEffect` | 50 | *(absent)* | 5 shops | Added |
| `SpecialIngredient` | 3 | *(absent)* | 0 shops | `0` none, `1` Fat, `2` Salt, `3` Ice, `4` Sugar |
| `AppearanceEffect` | *(absent)* | *(absent)* | 0 shops | `1` Balloon, `2` Costume |
| `ShopType` | 4 | *(absent)* | — | Every shop sets its own; no category file declares it |
| `ExcitementLevel` | *(absent)* | 35 | 30 sideshow | |
| `RideHandlesSprite` | 1 | 1 | 0 rides, shops and sideshow | "If the script handles the person sprite". The other three themes' `SideShow.sam` declare 1 |
| `NumSimultAnims` | *(absent)* | 3 | 1 | How many animations it may run at once |
| `EntryCellStandPosX` / `Y` | *(absent)* | 0.5 / 0.1 | 0.5 / 0.5 | Sub-cell position, fractions of a cell |
| `ExitCellAppearPosX` / `Y` | *(absent)* | 0.5 / 0.1 | 0.5 / 0.5 | |
| `MinCapacity` / `MaxCapacity` | *(absent)* | *(absent)* | 0/0 shops, 1/3 sideshow | |

## Identifying an item

All but three of each theme's items carry an `Info.Id` of four digits: the theme, the category, then two
for the item. The theme digit is `1` for the Jungle, `2` Halloween, `3` Space and `4` Fantasy; the
category digit is the band:

| Band (Jungle) | Category    |
| ------------- | ----------- |
| `11xx`        | Rides       |
| `12xx`        | Shops       |
| `13xx`        | Sideshows   |
| `14xx`        | Features    |
| `15xx`        | Upgrades    |
| `16xx`        | Fixed items |

Three items carry the same small number in every theme instead: `100` the Mystery ride, `101` Buy Land
(`ground`) and `102` Clear Land (`groundc`).

The Jungle's fixed items are `1600` Bus, `1601` Gates, `1602` Seaplane, `1603` Lights, `1604` Ferry and
`1605` End. A theme's `global.sam` names its gate by that number, as `ParkName.GateObjectId`.

> WAD members are RefPack-compressed, so searching a `.wad` for `Info.Id` with a plain text search
> returns fragments of the surrounding compression tokens rather than the value. Decompress the member
> first — see [Archives](/formats/wad/).

## Fixed items

A handful of items are placed by the game rather than by the player, and are marked by two keys:

| Key                    | Meaning                                                                 |
| ---------------------- | ----------------------------------------------------------------------- |
| `Info.WhichUIType 4`   | Not shown in the build interface ("Not to be shown in UI")              |
| `Info.DontApplyOffset` | `1` — the item's animation plays relative to world `(0,0)`, not its own position |

`Info.DontApplyOffset` is set by the six fixed items alone, in every theme. `Info.WhichUIType 4` is not
theirs alone: the Mystery ride, Buy Land and Clear Land carry it too.

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

Only the four gates declare them. Halloween's gate declares the same four values; Fantasy's is 6 by 5,
and Space's 6 by 4 at `(45, 15)`. Every gate's own `Info.Shape` picture is a single `*`, so its footprint
comes from these keys alone. The cell count those give agrees with the item's
[footprint file](/formats/hmp/).

## Info.Shape is a block, not a value

Most keys are `key <whitespace> value`, but a few are followed by a fenced block: a line of three
dashes, the content, and another line of three dashes. `Info.Shape` is one, in all 274 items, and
`Info.Hoarding` another, in 129 of them; no loose `.sam` holds a block.

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
— its widest row by its number of rows — not the number of marks inside it.** Every shipped picture's rows
are the same width. The [footprint file](/formats/hmp/) agrees with the box on all 274 items, the gates'
through their overrides; counting the marks instead goes wrong on the 22 items whose pictures hold a `.`,
among them the Jungle's `4x4rock`, which draws fourteen stars inside a four-by-four box, and `ground`,
which draws none at all.

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

### The hoarding picture

`Info.Hoarding` uses a separate sixteen-character table. Every cell has kind zero; the bits select
panel edges. Each selected edge makes one panel, including internal edges if the picture asks for them.
The names below use the executable's compass bits; the same row reversal described below applies.

| Character | Edge bits | Edges |
| --------- | --------- | ----- |
| `.` | `0x00` | none |
| `^` | `0x01` | north |
| `_` | `0x10` | south |
| `[` | `0x40` | west |
| `]` | `0x04` | east |
| `J` | `0x14` | south, east |
| `F` | `0x41` | north, west |
| `7` | `0x05` | north, east |
| `L` | `0x50` | south, west |
| `=` | `0x11` | north, south |
| `H` | `0x44` | east, west |
| `C` | `0x51` | north, south, west |
| `U` | `0x54` | south, east, west |
| `n` | `0x45` | north, east, west |
| `3` | `0x15` | north, south, east |
| `O` | `0x55` | all four |

The Belly Bounce's block is:

```text
Info.Hoarding
---
F.7
[.]
[.]
L.J
---
```

Its twelve selected edges leave gaps; a rectangle would give the wrong outline. A fresh Q91b corpus
check (2026-10-03) reads all 274 item archives: 129 have hoardings, each matching its footprint's width
and depth. Their first model meshes have at least four source vertices, all finite after transformation.
The executable's mode-2 parser at `0x00402720` uses the glyph table at `0x007397b0`, skips literal spaces,
preserves blank rows and reverses the rows. The working grid is 20 by 20; malformed overlong input is
not a supported way to enlarge it. Corner fitting, terrain placement and animation are executable
behaviour, documented in OpenTPW's `docs/exe/ride-hoardings.md`.

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

Every object record in the Jungle's shipped park has the entry and exit cells this reading predicts,
once each is turned by its placement angle — all fourteen of them, the eleven placed things and the fixed
bus, gates and lights.

## An item ships only its own art

An item's WAD carries `textures/` and `stexture/` folders, but they hold only the art unique to that
item. Everything else comes from the theme's shared archives, which sit beside the items in
`data/levels/<theme>/`:

| Archive | Pairs with | Contents |
| ------- | ---------- | -------- |
| `sharetex.wad` | `textures/` | full-size shared textures |
| `ssharete.wad` | `stexture/` | the same set at lower detail |

All four themes ship both, the two of a pair holding members of the same names: 116 each in the Jungle,
110 in Space, 98 in Halloween and 81 in Fantasy, one of them a `qickload.txt`. The effect is easy to
underestimate: the eleven objects the Jungle's shipped park places name **40 textures that are in none of
their own WADs**, so a loader that looks only beside the model finds no art for most of every item's
surfaces. Resolve a material by looking in the item's own folder first and the theme's shared archive
second.

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

## `data\sound.sam`

Worth singling out because it holds the mix the game starts with, and because one of its values
is not what its own header comment implies.

The file says of itself that "all the values correspond to detail values at which they are
enabled" — that is, most `SoundInfo` keys are thresholds on the 0–100 sound-detail slider
(`QSOUND 15`, `EAX 10`, `GEOMETRY 20`, `MPEGRATE 35`, and so on).

`DefaultVolume` is not a threshold. It is the starting position of the four volume sliders, out
of 100:

| Key | Default |
| --- | --- |
| `DefaultVolume.SFX` | 75 |
| `DefaultVolume.MUSIC` | 60 |
| `DefaultVolume.SPEECH` | 75 |
| `DefaultVolume.MOVIE` | 100 |

Those four groups are also how the game attenuates itself while the advisor talks: it multiplies
the music and SFX group volumes by a percentage and leaves speech alone, restoring them when the
sample ends.

**Open question.** `SoundInfo.DUCKINGLEVEL` is `38`, and it is the only ducking-related key in the
file. The percentage the engine multiplies by lives in memory that is zeroed in the image and has
no writer the decompiler can see, so it is filled in from configuration at runtime — which makes
`DUCKINGLEVEL` the only candidate, and would mean the mix drops to 38% while the advisor speaks.
But that reading contradicts the file's own header comment, under which `38` would instead be the
detail level at which ducking switches on. Not yet resolved either way.

## `data\_Resolution.sam`

EA's resolution override, 424 bytes, CRLF line endings. It ships **inactive**: the game reads only
`Resolution.sam`, and the file's own comment says to rename it to activate it (along with "Modifying this file
is NOT recommended unless directed by an Electronic Arts Customer Service Representative"). It sets one key,
tab-separated:

```
Res.RESOLUTION	5
```

| Value | Resolution |
|---|---|
| 0 | the options screen's setting |
| 1 | 512 x 384 |
| 2 | 640 x 480 |
| 3 | 800 x 600 |
| 4 | 1024 x 768 |
| 5 | 1280 x 1024 (the shipped value) |
| 6 | 1600 x 1200 |
| 7 | 2048 x 1536 |

The table is the file's own comment, which spells the key `RESOULTION`. Checked in the running original (under
Proton): with the file shipped as it is, the game started at its default 640 x 480, not at 5; renamed to
`Resolution.sam` with value 4, it ran at 1024 x 768. The other values have not been tried.
