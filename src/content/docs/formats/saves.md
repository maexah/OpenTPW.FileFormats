---
title: Saves (*.tpws, *.ints, *.lays)
---

Theme Park World stores saves with three different extensions.

These extensions are:

- **TPWS**: Standard save
- **INTS**: Initial save (e.g. for *Instant Action* mode)
- **LAYS**: Online save (for uploading parks)

## File format

Offsets below are of an offline save; an online one carries an author header after the flag at `0x609`
and is not covered here.

**Header**

The fields are the game's own reads, one after another: `FUN_00414d40` reads the version, and
`FUN_00416240` everything after it up to `BILZ`.

| Offset | Size | Description |
| --- | --- | --- |
| `0x000` | 4 bytes | Version. The park the game ships carries `400` (`90 01 00 00`); a saved park carries `500`, as all eight park files the original wrote in play do. This is **not** a magic number - `F4 01 00 00` is simply `500` written little-endian, and a reader that requires it rejects the shipped park. Accept both. The game refuses a version above 500 when `FUN_00414d40` is called in its mode 2, as two of its three callers do: "Trying to load future version savegame into earlier game - get a patch" |
| `0x004` | 1 byte | The language the game was running in when it wrote the file: `0` English (and any language not named here), `1` Spanish, `2` Italian, `3` Swedish, `4` German, `5` French, `6` Japanese (the writer, `FUN_00415f50`). The loader compares it with nothing and uses it as an index (× 20 into `0x78a460`) to pick the legal text the next field is checked against; `0` in all ten park files |
| `0x005` | `0x500` bytes | Legal text: a copyright notice in UTF-16, 824 bytes (412 characters, so reading it as single bytes gives every other byte as a NUL), then zeros to the end of the field. A load whose text fails `FUN_005f7e60`'s check stops with "The save game legal text has been jiggered with!" |
| `0x505` | `0x100` bytes | Validated by `FUN_0051ab60`, and its first 32 bytes are copied to `0x802080`; the writer fills the field with zeros and copies the same 32 bytes back in from `0x802080` (`FUN_00415f50`). Zero in all ten park files. What it holds is not settled |
| `0x605` | 4 bytes | Magic, read **big-endian** (through `ntohl`) and required to equal `0x01221985`, so stored `01 22 19 85` |
| `0x609` | 4 bytes | Online-header flag: any value but `0` means an author header follows it (read by `FUN_00418da0`: the author's name and e-mail, a park description and the date published); `0` means none does. `0` in all nine park files, and nothing pads it |

These close exactly on the compressed block: `4 + 1 + 0x500 + 0x100 + 4 + 4` is `0x60D`, where `BILZ` sits in
all nine park files. Bytes `0x600` to `0x60C` read `00 00 00 00 00 | 01 22 19 85 | 00 00 00 00`; a reading
that takes a four-byte file type `00 01 22 19` at `0x604`, a one-byte file version `85` at `0x608` and a
one-byte online flag at `0x609` covers the same bytes cut at different places, and the loader's reads are what
place the cuts.

**Data (compressed using ZLIB)**

| Offset | Size | Description |
| --- | --- | --- |
| `0x60D` | 4 bytes | Tag - `BILZ` |
| `0x611` | 4 bytes | The size the payload inflates to |
| `0x615` | 4 bytes | The size of this whole block, its tag and header included - so `0x60D` plus this is the file's length |
| `0x619` | 16 bytes | Four dwords, `15, 9, 0, 0`: the first two are the window bits and the memory level the stream was deflated with, the last two are written as nought (`0x00619280`). The same in all ten park files |

Neither of those two sizes is a compressed length. The ZLIB stream begins at `0x629` - the 28-byte
header counts the tag, which is an easy four bytes to lose - and continues to the end of the file. Both sizes
are worth checking on load: one confirms the block reaches the end of the file, the other that the payload
inflated to the size it claimed, 1,608,309 bytes in the shipped park.

The stream is plain zlib 1.1.3, deflated in one call at the default level with a 32K window (15 bits) and
memory level 9, so it opens `78 9C`. Inflating any of the ten park files and deflating the result again with
those settings (level 6, window bits 15, memory level 9, default strategy) gives the stored stream back byte
for byte; with memory level 8, zlib's own default, it does not in any of them. The ten are the shipped park,
the eight files Alexah's Full Simulation play wrote (three of them `restart.INTS`, which has this same
layout), and one Instant Action save written under Proton on 2026-10-08.

## Inside the stream: a chain of modules

The inflated payload is **not** one structure. It is a run of modules, each written by the subsystem
that owns it and each followed by a **four-character tag** the game checks on the way back in — so a
reader that lands exactly on the next tag has agreed with the game about every byte in between. The
check is how the game itself detects a module that did not load the same number of bytes it saved.

> The tags are compared as **dwords**, not as text, so they are stored little-endian and read
> **backwards** in a hex dump: `WRLD` appears as `DLRW`, `RSYS` as `SYSR`. Searching a dump for a tag
> the right way round finds nothing at all, which reads exactly like proof the module is absent. The
> magics that open three modules rather than close them - `TPCS` at the head of the sprite table, `LCTP` at
> the head of the particles and `RSSE` at the head of the script module, all below - read in a dump exactly
> as printed here.

The payload is also a **serialised memory image, not a portable format**. Live heap pointers are written
out verbatim - the sprite table's slot handles below are addresses from the session that saved it - so
apart from the tags nothing in it can be found by searching for a value, and every offset is reached by
walking from the start. Modules butt against one another with no padding: a reader finishes one module
exactly on its tag and the next begins four bytes later, and being one byte out corrupts every module
that follows. Only some modules carry a length of their own (the ride system and the track rides below),
so in general the chain has to be walked module by module.

The order is the game's own, and it is the same in all nine park files. The offsets are measured in
`data/levels/jungle/Easymode.TPWI` and are that file's rather than a general layout.

| Tag (as stored) | Read as | Module | The game's names | Tag at | Bytes since the previous tag |
| --- | --- | --- | --- | --- | --- |
| `DLRW` | `WRLD` | World — the map and everything standing in the park | `SAD_AI`, World | 1,495,462 (`0x16D1A6`) | 1,494,283, from `0x49B` |
| `CSPS` | `SPSC` | Sprite scripts | `SAD_SPRITE_SCRIPTS`, Scripts | 1,500,918 (`0x16E6F6`) | 5,452 |
| `TRAP` | `PART` | Particles | `SAD_PARTICLES`, Particles | 1,577,174 (`0x1810D6`) | 76,252 |
| `SSEM` | `MESS` | Message centre | `SAD_MESSAGE`, MsgCntr | 1,577,444 (`0x1811E4`) | 266 |
| `KOLC` | `CLOK` | Clock | `SAD_CLOCK`, Clock | 1,577,456 (`0x1811F0`) | 8 |
| `TNAV` | `VANT` | "Vanilla time" | `SAD_VANILLA_TIME`, VanillaTime | 1,577,464 (`0x1811F8`) | 4 |
| `SYSG` | `GSYS` | Game system | `SAD_GAMESYS`, GameSystem | 1,577,504 (`0x181220`) | 36 |
| `SYSR` | `RSYS` | Ride system | `SAD_RIDESYS`, Ridesystem | 1,595,034 (`0x18569A`) | 17,526 |
| `KART` | `TRAK` | Track rides | `SAD_TRACK`, TrackRides | 1,595,082 (`0x1856CA`) | 44 |
| `RYLF` | `FLYR` | Flying rides | `SAD_FLYERS`, FlyingRides | 1,595,538 (`0x185892`) | 452 |
| `ESSR` | `RSSE` | Ride scripts | `SAD_RSSE`, RSSE | 1,606,398 (`0x1882FE`) | 10,856 |
| `EMAK` | `KAME` | Camera | `SAD_CAMERA`, Camera | 1,606,442 (`0x18832A`) | 40 |
| `SAOC` | `COAS` | Coasters | `SAD_COASTERS`, Coasters | 1,606,462 (`0x18833E`) | 16 |
| `SVDA` | `ADVS` | Advisor | `SAD_ADV`, Advisor | 1,606,838 (`0x1884B6`) | 372 |
| `NUOS` | `SOUN` | Sound | `SAD_SOUND`, Sound | 1,608,287 (`0x188A5F`) | 1,445 |
| `STHC` | `CHTS` | Cheats | `SAD_CHEAT`, Cheat | 1,608,293 (`0x188A65`) | 2 |
| `CSDA` | `ADSC` | Advisor scoring | `SAD_ADV_SCORING`, AdvisorScoring | 1,608,301 (`0x188A6D`) | 4 |

The game's names are the two the loader (`FUN_00415270`) gives each module: the `SAD_` string its tag check
reports, and the name it logs as `<name>: loaded %d bytes`. The World module's check names `SAD_AI`. The
untagged UI block at the end is logged as `UI`.

Each tag *follows* the module it belongs to. The World module does not begin at the start of the stream
either: an untagged **action recording** (`GActionRec` in the game's save log) is written first, and its
serialiser (`FUN_00403780`) names its two dwords:

```text
u32  mLoadedPublishedPark
u32  recording_size
byte[recording_size]  recording      -> the block is 8 + recording_size bytes
```

In the shipped park those read `0` and `1171`, so the recording occupies bytes 0 to 1,178 and the World
module begins at **`0x49B`**; where World starts is always derived from that length rather than assumed.

After the last tag comes an untagged UI block. It is four bytes, nought, in seven of the nine park files -
in the shipped park it ends the stream on 1,608,309. In the two saves of Alexah's played jungle park its
first dword is `1` and 540 more bytes follow, among them the UTF-16 text of an advisor line about the
ticket price; that longer form is not decoded.

## The world block (`WRLD`)

The first block in the payload is the park itself, closed by the `DLRW` trailer. Its header begins after
the action recording, and the block runs, with every boundary closing exactly on the next:

```text
0x00049B  26 header fields (below), 64 bytes                       ->  0x0004DB
0x0004DB  mObjectControls[150], 32 bytes each                      ->  0x00179B
0x00179B  mNumObjectControls u32, mPreviousSearchKey u16           ->  0x0017A1
0x0017A1  the staff pool's 32 records of 20 bytes                  ->  0x001A21
          mType 4, mName 4, mPayGrade 1, mSubType 1, mValid 1,
          mOnPointer 1, mTimeSig 4, mTimeoutTime 4
0x001A21  arrival and clock fields, 76 bytes                       ->  0x001A6D
          the last 18 of them the arrival block (below), from 0x001A5B
0x001A6D  16,384 map cells (below)                                 ->  0x152431
0x152431  u32 Used Thing Head                                      ->  0x152435
0x152435  the thing list (below)                                   ->  0x16D1A6
0x16D1A6  the trailer, stored as DLRW
```

The offsets are the shipped park's. The recording's length varies from file to file - the eight played ones
start their header at `0x8F`, `0x6F4` or `0x1C73` - so a reader takes the rest relative to the header. The writer (`FUN_00516c80`) logs the size of each part under its own name, which
is what groups them: "World vars" is the header, `mControlManager` the object controls with their count and
key, `mMacroAI` the staff pool, the park clock and the arrival timer, `mMap` the cells, and "Thing Array"
the list.

The header is 26 fields written back to back with no padding, 64 bytes in all. **The names below are the
game's own.** Each field is announced to a logging call that the release build compiles away, so the names
never reach the file and cannot be searched for - but they survive in the executable beside the address each
one is read into, which is what makes this list checkable rather than inferred.

| # | Size | Name | Notes |
| --- | --- | --- | --- |
| 0 | 4 bytes | `version` | The writer always writes `2`, and all nine park files hold it |
| 1-3 | 2 bytes each | `mArrivalVehicle_Size1..3` | Handles - see below |
| 4 | 2 bytes | `mBankAccount` | A **handle**, not an amount - see below |
| 5 | 2 bytes | `mCurrentArrivalVehicle` | Handle - see below |
| 6 | 4 bytes | `mGameTick` | The park's own tick counter - `755` in the shipped park |
| 7 | 2 bytes | `mMechanicHQ` | Handle |
| 8 | 2 bytes | `mParkAnalyser` | Handle |
| 9 | 4 bytes | `mParkClosed` | **Zero means open** - `0` in the shipped park |
| 10 | 4 bytes | `mNumberOfVisitorsToDate` | Guests ever admitted - `0` in the shipped park |
| 11 | 2 bytes | `mParkGates` | Handle - thing `11` |
| 12 | 2 bytes | `mTrafficLights` | Handle - thing `12` |
| 13 | 4 bytes | `mRandomSeed` | |
| 14 | 2 bytes | `mResearchLab` | Handle |
| 15 | 2 bytes | `mStaffHQ` | Handle |
| 16 | 2 bytes | `mTagSystem` | Handle |
| 17 | 2 bytes | `mUIMsgReceiver` | Handle |
| 18 | 2 bytes | `mWeather` | Handle - the weather is a thing like any other |
| 19 | 4 bytes | `mWorldState` | A value, not a handle - see below |
| 20-24 | 2 bytes each | `mFirstHandyman`, `mFirstMechanic`, `mFirstEntertainer`, `mFirstGuard`, `mFirstResearcher` | Heads of the per-trade staff lists. **Guard comes before Researcher** |
| 25 | 2 bytes | `mFirstObject` | Head of the object list - thing `15` |

**The order in the file is not the order in memory**, and this is the trap to know about. The writer pairs
each name with the struct offset it is read into, and those offsets run `mRandomSeed` `+0x1da708`,
`mGameTick` `+0x1da70c`, `mParkClosed` `+0x1da710`, `mNumberOfVisitorsToDate` `+0x1da714`, `mWeather`
`+0x1da724`, `mBankAccount` `+0x1da726`, `mParkGates` `+0x1da732`, `mWorldState` `+0x1da738` — a different
order from the one written, down to the last pair: `mFirstResearcher` sits at `+0x1da742`, before
`mFirstGuard` at `+0x1da744`. Read the file by the list above; do not sort by offset.

A handle is a thing id, compared against a thing's own id with `==`: `11` means "the thing whose id is
11", not "the eleventh thing".

`mParkClosed` is `1` while the park is shut and `0` while it is open. The command that opens and shuts a park writes `0` on
one branch and `1` on the other, then picks the word for its own message with
`mParkClosed == 0 ? "opened" : "closed"`. The world constructor writes `1` before anything is loaded, so
a park is born shut and a save holding `0` is one that was opened while it was being played. The played
files agree: the three `restart.INTS` files and an autosave taken four ticks in hold `1`.

`mBankAccount` is the field whose name misleads. It is two bytes, and the shipped park holds `8` in it -
no sort of balance. The executable reads it in exactly one place, and that reader is the weather thing's
accessor character for character with one offset changed: take the word, return `thingTable[id]`. So it
names a *thing*, and the thing it names is the one carrying the park's admission fee, balance, profit for
the year and loan table - which is why the state a guest is in while judging the admission fee reaches
the park's money through this very accessor.

`mWorldState` really is a value. The executable writes `1`, `2` and `4` into it and compares it against
`4` in six places, among them the game's own state machine and the build-a-park menu. The shipped park
holds `0`, which is not in that set - and so does every one of the eight played park files, the two saves
of a jungle park played to `mGameTick` 19,004 and 19,007 among them. So zero is what a park carries while
it is played, at least as saved, and not only before it is first entered. What it means is not settled,
and naming it would be a guess.

### The arrival vehicles

The four arrival fields are worth calling out because their names encode behaviour. **Each slot belongs to
one kind of vehicle**: `mArrivalVehicle_Size1` the bus, `_Size2` the seaplane and `_Size3` the ferry
(`FUN_0051a2f0` caches them at `+0x1da72c`, `+0x1da72e` and `+0x1da730`). The size in the names is how the game
picks a kind when it is told the size of the arriving crowd: fewer than 36 people force the bus, 36 to 60 the
seaplane and more than 60 the ferry (OpenTPW's `docs/exe/park.md`, "Arrivals"). A call made with no size, which
the arrival timer makes when no vehicle is current, picks one of the three at random from `mRandomSeed`, and a
kind whose feature cannot be found falls back to the others; both cache by kind as well. However it was picked,
the engine creates the vehicle thing the first time that kind is sent for, then caches its id in that kind's
slot. `mCurrentArrivalVehicle` holds whichever is on its way, or nought when none is.

So the slots say what a park has *done*, not what it owns: an empty slot means that kind has never been sent
for, whatever the size of the crowds. The shipped Lost Kingdom park holds the bus, thing `15`, in
`mArrivalVehicle_Size1` and nought in the other two, which is why its thing list contains a bus and neither a
ferry nor a seaplane: only the bus has ever been sent for there. Alexah's played jungle park holds all three -
the bus (item 1600) as thing 38, the seaplane (1602) as 60 and the ferry (1604) as 73 - with the ferry
current; the played fantasy park holds the first two, the second current.

### The object controls

`mObjectControls` holds one 32-byte record per kind of item, keyed by the item's id, filling the first slots;
the rest of the 150 are zero. `mNumObjectControls` after the array counts the used ones - 50 in Lost Kingdom's
save, among them the eleven kinds of its fourteen objects (the eight kinds it has placed, plus the bus, the
gates and the traffic lights), so it is not a list of what stands in the park. `mPreviousSearchKey` reads 1100
there; the loader zeroes it after reading it. The first five fields below are settled against the item files
for all 50 records with no mismatch; the count is checked against that save's fourteen objects, and both of
the last two against the running game's memory (the game's reader of them is its arrivals' headcount):

| Offset | Type | Holds |
| --- | --- | --- |
| `0x00` | u16 | The item's `Info.Id` |
| `0x04` | i32 | What buying one costs, the item's `Upgrades[0].CostOfUpgrade` - Belly Bounce 500, Drinks Shop 650 |
| `0x0C` | i32 | The item's `UsageInfo.RipOffOK` - 100 for the six shops, 250 for the four sideshows (each from its folder's `Shops.sam` or `SideShow.sam`), 0 for the rest |
| `0x10` | u8 | Researched: 1 on 26 records, exactly the 26 whose item's file (with its `Easy_` layer) sets `Upgrades[0].CostOfResearch` 0. Among them the rides Belly Bounce, Crazy Ape, Rocky Racers and Aztec Mayhem, the shops Balloon, Burger and Drinks, and the sideshows Jungle Spray and Strength Bird; the other 24 read 0 |
| `0x14` | i32 | The upgrade tier researched, 0 to 2: 2 on 17 of the 26 researched records, 0 on every other record |
| `0x18` | i32 | How many of the item stand in the park. Lost Kingdom's save: 3 for the Small Toilet, 2 for the Security Camera, 1 for each of nine more kinds, 0 for the rest, 14 in all, its fourteen objects |
| `0x1C` | u32 | `mGameTick` when the first of the item was built, kept after the last is sold: 15 on nine records in Lost Kingdom's save, the Balloon Shop among them with a count of 0, and 604 for the bus. The game writes it only while it reads 0, so the gates and the traffic lights, made on tick 0, keep 0 |

Across the six park files read (Lost Kingdom's and five played saves of three themes) every record with a
count above 0 has a stamp, but for the gates and the lights. The other fields (`0x08` and the three bytes after `0x10`, all nought here) are not settled here. The game fills a
record from the item's own description, so a save carries whatever the item files said when the record was made.

### The staff pool

The people a park may hire: 32 records of 20 bytes, each read field by field by the game (`0x005070a0`), whose
names for the fields these are. A record with `mValid` 0 is an empty slot, and the rest of it means nothing.

| Offset | Type | Field | What it holds |
| --- | --- | --- | --- |
| `0x00` | i32 | `mType` | The kind: 0 handyman, 1 mechanic, 2 entertainer, 3 guard, 4 researcher |
| `0x04` | i32 | `mName` | A row, 0 to 34, of the kind's name table (`HANDYMAN_NAMES.str`, `MECHANIC_NAMES.str`, `ENTERTAINER_NAMES.str`, `GUARD_NAMES.str`, `RESEARCHER_NAMES.str`) |
| `0x08` | u8 | `mPayGrade` | The grade, 0 to 4 |
| `0x09` | u8 | `mSubType` | 0 on every handyman, guard and researcher measured, 0 or 1 on the mechanics, 0 to 2 on the entertainers. The game takes it as the costume |
| `0x0A` | u8 | `mValid` | 1 where the slot holds a candidate |
| `0x0B` | u8 | `mOnPointer` | 0 on every record measured. By the game's name for it, 1 while the candidate is in the player's hand; no file here shows that |
| `0x0C` | i32 | `mTimeSig` | The header's `mGameTick` as the candidate joined the pool |
| `0x10` | i32 | `mTimeoutTime` | How long the candidate waits to be hired, in fours of `mGameTick`. 120 to 179 on every record measured, which is the range the shipped balance gives (`StaffPoolInfo.StaffTimeoutTime` 120, and up to half of it more), not a limit of the format |

Lost Kingdom holds sixteen, in slots 0 to 11, 13, 18, 22 and 23: six joined on tick 361 and ten on 722, which is
also the pool's own `mTimeSig` (below), the tick it was last topped up on. The name is kept as a row, not as text,
so the same save shows different names under the two language folders where their tables differ: slot 3's mechanic,
row 12, is Rob O'Farrell by `English` and Alex Cullum by `american`.

Measured on the ten different park files on the machine this was written on (the shipped `Easymode.TPWI`, of which
a player's copy is the same byte for byte; that player's `restart.INTS`; and eight `.TPWS` and `.INTS` files written
by the game in three themes; one more than the nine the rest of this page counts, which leaves out that
`restart.INTS`): 172 occupied records, every one inside the ranges above, with grades 0 to 3 and a
`mTimeSig` never later than the file's `mGameTick`. In Lost Kingdom's file the sixteen empty slots are of two kinds:
eight still hold the fields of a candidate who has gone, with `mValid` 0, and eight were never used, their `mName`,
`mPayGrade` and `mTimeoutTime` the fill byte `0xCD` and the rest nought. The three `restart.INTS` files written as a Full Simulation park was created hold the opening pool, twenty-two
candidates all joined on tick 0.

### The arrival and clock fields

The 76 bytes after the staff pool's records are three blocks the game reads one after another: the end of
the staff pool (30 bytes, read by `0x00507850` after its records), the park clock (28 bytes, `0x004f7f30`)
and the arrival timer (18 bytes, `0x004cf050`). The names are the game's own, passed to its logging call as
each field is read:

| Offset | Size | Field | Lost Kingdom |
| --- | --- | --- | --- |
| `0x001A21` | 4, 1 | `mPeopleInCat[0]`, `mStopProducing[0]`, and so on for five pairs | 1, 0 in each pair |
| `0x001A3A` | 4 | `mTimeSig` - the staff pool's own, not the arrival timer's | 722 |
| `0x001A3E` | 1 | `mOpeningStaffPoolGenerated` | 1 |
| `0x001A3F` | 8 | `mFunnyTimeStart`, a Windows FILETIME | 2000-01-01 00:00:00 |
| `0x001A47` | 8 | `mSessionStart`, a Windows FILETIME | 1999-10-21 20:54:53.29 |
| `0x001A4F` | 4 | `mMonthAtLastUpdate` | 1 |
| `0x001A53` | 4 | `mDayAtLastUpdate` | 2 |
| `0x001A57` | 4 | `mFunnySecsPerRealSec` | 15000 |

#### The arrival block

The last 18 bytes are the state of the game's arrival timer:

| Offset | Size | Field | Lost Kingdom | Holds |
| --- | --- | --- | --- | --- |
| `0x001A5B` | 4 | `mArrivalRate` | 0 | Not settled; 0 in all nine park files |
| `0x001A5F` | 4 | `mTimeSig` | 661 | The `mGameTick` of the turn that found the last load of visitors all off |
| `0x001A63` | 4 | `mTargetVehicleCapacity` | 5 | Not settled; 5 in all nine park files, and 5 is also what a park starts with before any save is read |
| `0x001A67` | 4 | `mPeopleOnBus` | 0 | How many of the load in progress are still to get off |
| `0x001A6B` | 1 | `mOffloading` | 0 | 1 while a load is in progress |
| `0x001A6C` | 1 | `mGatesOpen` | 1 | Not settled; 1 in all nine park files, and 1 is also what a park starts with |

`mTimeSig` is measured against the header's `mGameTick` (755 in Lost Kingdom), and both are in the same unit, one
count per turn of the game's things. The game calls the next load once `mGameTick / 4` has passed `mTimeSig / 4`
(each rounded down) by more than `Arrival.TimeBetweenArrivals`. Both fields are saved, so a park loaded from this file
is 94 counts into its wait already (23 in the fours the comparison uses). The executable's side is in OpenTPW's
`docs/exe/park.md`, "Arrivals".

### The map

The map is 128x128 cells whatever size the park inside it is, and it is **the bulk of the block** - about
1.3MB of the shipped park's 1.5MB. It has no fixed stride. **Each cell opens with a status byte saying
which of three optional sub-records follow it**, one bit each:

| Bit | Sub-record | Size |
| --- | --- | --- |
| `0x1` | map | 52 bytes |
| `0x2` | track | 31 bytes |
| `0x4` | effects | 10 bytes |

A cell that is entirely default writes its status byte and nothing else, which is where the block's
variable length comes from. Only two combinations occur in the shipped park - `3` on 16,134 cells and `7`
on the other 250, coming to 84 and 94 bytes - but the bits add independently, so a walk that sums them
reads the combinations no shipped park happens to contain. The shipped park's cells total 1,378,756 bytes,
exactly the region from `0x1A6D` to `0x152431`, and an implementation that only needs what is *in* the park
can measure each cell and skip it.

> **Check the stride cell by cell, not just in total.** Measured this way, all 16,384 cells land on the
> next cell's status byte every single time. That is a sharper check than the block's own `DLRW` trailer,
> which a pair of compensating errors would still reach.

The map sub-record is a **29-byte tile base** followed by a **23-byte litter block**. The track
sub-record repeats the same tile base field for field and adds `mSegmentNumber`, two bytes, for its 31 -
`0xffff` on every cell of the shipped park. The effects sub-record's ten bytes are not described here.

| Offset | Size | Name |
| --- | --- | --- |
| 0 | 1 byte | `mDirection` |
| 1 | 2 bytes | `mFlags` |
| 3 | 4 bytes | `mMeshInstance` |
| 7 | 1 byte | `mNeighbours` |
| 8 | 2 bytes | `mOverlapCounter` |
| 10 | 2 bytes | `mParentID` |
| 12 | 12 bytes | `mTileData` - three dwords: set, index, angle |
| 24 | 4 bytes | `mType` |
| 28 | 1 byte | `mHoardingNeighbours` |
| 29 | 4 bytes | `mLitter` |
| 33 | 2 bytes | `mLitterCollector` |
| 35 | 4 bytes | `mLitterScript` |
| 39 | 4 bytes | `mLitterScript` **again** |
| 43 | 2 bytes | `mPylonIndex` |
| 45 | 1 byte | `mStatusFlags` |
| 46 | 4 bytes | `mTimeMarkedForLitterCollection` |
| 50 | 2 bytes | `mWho` - the first thing on the cell; the rest hang off it by each thing's `mMapChild` |

The serialiser really does announce `mLitterScript` twice, for two consecutive dwords, and the arithmetic
is what says so rather than the reading: `4+2+4+4+2+1+4+2` is exactly 23, and `29+23` is exactly the 52
that the cell walk measures from the other direction. Written once, every field after it would shift by
four and the record would close four bytes short of the next cell's status byte.

> **Cells are indexed `y * 128 + x`** — the opposite way round from the attribute map in `base.map`,
> which is `x * 128 + y`. Getting it backwards still produces something that looks like a map, so it is
> worth pinning: only the y-major reading reproduces `base.map`'s own bus road, ticket booths and
> entrance column.

**`mStatusFlags` is the attribute map.** The byte at offset 45 holds the same value `base.map` carries
for that cell - checked across all 16,384 cells of the shipped park against a separate file, with its own
header, parsed by different code, and indexed `x * 128 + y` where the save's cells run `y * 128 + x`.
Every cell agrees. Agreement in aggregate would prove little; agreement cell by cell under *opposite*
indexing is not something a misaligned or transposed reading can produce. 1,495 cells are non-zero, over
exactly the eight values the attribute map uses: 0, 1, 3, 8, 17, 128, 144 and 148.

**`mWho` at 50 is occupancy** - the id of the thing standing on the cell. Twenty-four cells of
the shipped park carry a value, and eleven of them are exactly its eleven placed catalogue objects, each
naming *itself* on the cell it stands on: the cell at (55,15) holds `23`, and object `23` stands at
(55,15), and so for all eleven. Twelve more hold person ids, gathered on the approach to the
park gates at x 47-48 and at the staff's own positions, and (0,0) holds `15`, one of the three
unplaced objects. A guest waiting to be let in tests the cell
underfoot against their own id, which is this field read from the other side - though that test is made
against the cell's *runtime* record, which is `0x44` bytes where the file carries 52, so the two layouts
do not share offsets.

Nothing has been dropped in the shipped park: `mLitter`, `mLitterCollector`,
`mTimeMarkedForLitterCollection` and `mPylonIndex` are nought on every one of the 16,384 cells. That is a
fact about a save nobody has played rather than a gap in the reading.

`mNeighbours` and `mDirection` are **stored, not computed**, so a renderer does not have to derive a
neighbour mask from the cells around it. They share one compass: over the shipped park's path cells
`mDirection` only ever reads 0, 1, 4, 16 or 64 — the four cardinals of an
`N NE E SE S SW W NW` layout, and nothing between them.

The two are nevertheless **read in completely different ways**, and a reader that treats them alike will
be wrong about one of them. `mNeighbours` is **bit-tested**; `mDirection` is compared for **equality**
against a single value and is never masked. The park bears that out: across all 16,384 cells
`mNeighbours` takes **38 distinct values with 82 cells carrying more than one bit**, while `mDirection`
takes **five and never carries two**. Because those five are `0`, `1`, `4`, `16` and `64`, a single value
and a one-bit mask are indistinguishable over exactly the data you have — so the difference has to come
from the code, not from the corpus.

Which bit is which side can be settled from the file alone, without trusting either reading. For every
path or queue cell, whether the neighbour on a given side is *also* a path or queue cell is knowable by
walking the map, so a candidate pairing can be scored against reality. Over the shipped park's 82 such
cells, pairing `0x01` with `-y`, `0x04` with `+x`, `0x10` with `+y` and `0x40` with `-x` reproduces real
adjacency **94.8%** of the time; the opposite pairing manages 75.3%, and loses on all four sides taken
separately.

:::note[The bit is read from the cell being entered]
The executable's step check numbers its directions by axis, fixed absolutely by its own boundary guards —
it refuses `x == 0` going 3, `y == 0` going 0, `x == 0x7f` going 1 and `y == 0x7f` going 2, so **0 is
`-y`, 1 is `+x`, 2 is `+y`, 3 is `-x`**. Asked about direction 0, which is `-y`, it consults bit `0x10` —
the `+y` bit. Read as a question about the cell being *left*, that is its far side, and that reading is
wrong: the disassembly loads the **destination** cell into `ECX` before the call, and the destination's
`+y` side is precisely the side facing the cell being left. The predicate reads the **near side of the
cell being stepped into**.

No measurement on this park can tell those two readings apart. The side bits of `mNeighbours` are
**symmetric across every pair of side-by-side cells** - all 32,512 pairs, 65,024 tests counted from each
cell - so scoring "the destination's facing bit" against "the source's facing bit" returns 100% either
way; the disassembly is what settles it, which is why the `this` pointer the decompiler drops matters so
much here. It also reports "blocked" when the bit is **clear**, so the byte says where a cell *connects*,
not where it is walled.
:::

`mOverlapCounter` is **how many more times a cell has been built over** — the engine bumps it when a
path is laid on a cell that is already path, and deleting the cell takes one off, removing it only once
the count falls below nought (a forced clear, or one with no step, removes it at once). In the shipped park
14 path cells carry it: eleven at 1 — (39,21), (39,28), (43,29), (44,28), (47,28), (48,20), (56,15),
(56,16), (56,17), (56,21) and (56,28) — and three at 2 — (47,21), (48,21) and (48,28). By their side bits
every one is a corner or a junction except (43,29), which is a straight from north to south. The queue cell
at (52,22) reads 1.

`mFlags` bit `0x40` **marks land outside the park**. It is set on 13,878 of the shipped park's 16,384
cells, all of `mType` 7, 0, 2 or 30, and every path, queue and footprint cell is among the 2,506 without
it. The meaning is read from that split; nothing in the file names it. Bit `0x20` is the engine's
NOMODIFY, set on 18 of the 78 path cells.

In the **track** sub-record, `mNeighbours` is 0 on all 16,384 cells of the shipped park.

`mTileData` is **three dwords**, not one opaque run:

| Dword | Meaning | Values in the shipped park |
| ----- | ------- | ------------------------- |
| 0 | Tile set | `1` on all 78 path cells, `2` on all 4 queue cells, `0` on the other 16,302 |
| 1 | Tile index within that set | 2–20 on path cells, addressing the theme's [`PathTex` table](/formats/tct/) |
| 2 | Rotation, degrees | `0`, `90`, `180` or `270` — on every one of the 16,384 cells |

The index really does address the theme's `.tct`, confirmed by cross-referencing every index the shipped
park uses against the shape that cell's neighbours make: straights come out degree 2 and collinear,
corners degree 2 and bent, T-junctions degree 3, and the park's one crossroads degree 4 with a mask of
exactly N+E+S+W. The [Texture Correspondence Table](/formats/tct/) page carries that table in full,
including the two edge tiles whose masks show this to be an **area** tile set rather than one-cell-wide
lines — a walkway can be more than one cell across, and Lost Kingdom's entrance avenue is two.

> Two independent things support that split, and a wrong one would have to produce both by accident: the
> first dword divides the map exactly along the boundary the theme's `.tct` draws between its `PathTex`
> and `QueueTex` sections, and the third is a quarter turn everywhere and never an arbitrary angle. **So a
> park's paths carry the tile they draw and the turn it takes** — neither has to be inferred from
> neighbours.

`mType` says what a cell is. Counts across the shipped park:

| `mType` | Cells | What |
| ------- | ----- | ---- |
| 7 | 9,077 | |
| 0 | 6,875 | |
| 2 | 240 | |
| 1 | **78** | **path** — drawing them gives a connected loop with an avenue down to the park entrance |
| 30 | 66 | exactly the 66 cells whose `mStatusFlags` - and so `base.map` - carry `0x80`: the fixed approach every park inherits |
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

`9` sits where a thing is used: it is exactly the `mEntryPos` cell (see the catalogue object below) of the
eight objects somebody is sent to — the single cell of each toilet, the drinks shop's counter, the litter
bin, the staff room's door, the Jungle Spray, and the end of the Belly Bounce its queue arrives at. `10`
appears once, on that ride's far end, which is its `mExitPos`. The fountain and the two cameras, which
nobody is sent to, carry neither. Nothing in the file names the two types; what is *measured* is that all
three belong to a footprint rather than to the ground, and that `9` and `10` fall on exactly those cells.

> **This matters to anyone drawing a park.** Every one of the 44 carries a real ground texture index in
> `base.MD2` — not one is the "something covers this" index 0 — and yet each is already covered, because
> every item this park is built with opens its model with a flat floor plate exactly as wide as its
> footprint: `J_WC` under a toilet, `wf_floor` under the fountain, `js_base` under the staff room,
> `cn_floor01` under the drinks shop, `jb_floor` under the Belly Bounce, `jc_base` under a camera. Draw
> the ground there as well and the two fight for the same depth. The same goes for the four queue cells,
> where every piece of queue brings its own base.

### Where a built thing stands on its cells

An object's record gives an anchor cell and an angle (see the catalogue object below). Its footprint is
placed by turning it about the **middle of that anchor cell** — not about the middle of the footprint — and
the stored angle turns the *opposite* way to a positive rotation about the up axis.

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

### Thing records

The thing list is a **linked list, not an array**, of everything in the park - people, objects and the
park's singleton managers. `Used Thing Head`, the dword before it, gives the first thing's id. Each record
opens with the id of the *next* thing and then its model number, four bytes each, and a next of zero ends
the list:

```text
+0   u32  Used Thing Next     the NEXT thing's id
+4   u32  thingmodel
+8   u16  mX
+10  u16  mY
+12  u16  mMapChild
+14  u16  mMapParent
```

That leading dword is the trap in this block: read as the thing's own id it is off by one everywhere and
still looks plausible. The ids are not in order - the shipped park runs 41, 40 … 29, then 15, then 28 -
which is what a list with something spliced into it looks like, and what a counter cannot be; and the last
record's is `0`, a terminator no thing could have as an id. Read correctly, the shipped park's chain starts
at 42 and is an exact permutation of 1 to 42.

**Every thing opens with the same 16 bytes**: the list head, then the same four 2-byte map fields in the order
above (`FUN_0050b090`). The singleton managers write them too, and models 12 and 17 write nothing else. So a
thing's *own* fields begin **16 bytes into its record** - eight of list head and eight of map base. `mX` and
`mY` are in 256ths of a cell, and their high bytes are the cell the thing stands on, which is how the engine
reaches a cell without dividing. A thing with no place on the map stores `128` in both, which is *half a cell*
rather than an obvious sentinel - anything treating it as a position puts it at the origin, as the map itself
does. In the shipped park thirteen things are stored that way: things 1 to 10 (models 9 to 19) and the gates,
traffic lights and bus (11, 12 and 15), whose positions live in their models rather than in the save. All
thirteen hang off cell (0,0) through `mMapChild`: that cell's `mWho` holds 15, and the chain runs 15, 12, 11,
10 and on down to 1, whose `mMapChild` is 0. The same holds in all nine park files, the head of the chain
being whichever unplaced thing is in `mWho` at (0,0).

After that the models diverge completely and the sizes are wildly uneven, so the stream cannot be strided
- each model has to be recognised, and each has one fixed size: **1** a guest (533 bytes), **3** a
placeable catalogue object (1,099), **4 to 8** the five kinds of staff (511, 513, 509, 511 and 509), **9**
the strike system (103), **10** a bare map object (18), and **11 to 19** the singleton managers, from 16
to 73,544 (11: 5,846; 12: 16; 13: 73,544; 14: 4,190; 15: 99; 16: 300; 17: 16; 19: 937). The header's handles
name the managers, and name the same things in all nine park files: `mStaffHQ` thing 1 (model 9),
`mMechanicHQ` 2 (model 10), `mTagSystem` 4 (12), `mParkAnalyser` 5 (13), `mResearchLab` 6 (14), `mWeather`
7 (15), `mBankAccount` 8 (16) and `mUIMsgReceiver` 9 (17); things 3 (model 11) and 10 (model 19) are named
by no header field, though 10 is the challenge manager (below). There is no model 18 in a park file: the writer refuses it, "Should not be able to save
online persons". With those sizes, the walk closes exactly on `DLRW` in all nine park files.

**The offsets below are file offsets, and they are not the offsets a decompiler shows.** A thing is
written field by field in the order its reader asks for them, so a field's place in the record is the sum
of the sizes before it and bears no relation to where it sits in memory: `mAdmissionFee` is at `+0x118`
in the running game and at `+16` in the record, and a guest's `mState` is at `+0x220` and at `+505`. Taking
the memory offsets and using them as file offsets produces something that parses and is wrong.

> **Make the trailer an assertion.** Every record length in the block feeds one running offset, so a
> reader that ends exactly on `DLRW` had all of them right, and one that is a single byte out cannot. It
> is the same end-to-end check the container already allows: the block length reaching the end of the
> file, and the payload inflating to its declared size.

### A catalogue object (model 3)

Model 3 is everything a player buys and places - shops, rides, sideshows and scenery. The shipped park
holds **fourteen** of them: eleven placed, and three carrying the unplaced sentinel `128` in both
coordinates. Its record is **1,099 bytes**.

| Offset | Size | Name | Notes |
| --- | --- | --- | --- |
| 8 | 2 bytes | `mX` | from the shared map base |
| 10 | 2 bytes | `mY` | |
| 16 | 4 bytes | `mAngle` | degrees - `0`, `90` or `270` in the shipped park |
| 20 | 2 bytes | `mId` | the item's `Info.Id`, from its own `.sam` |
| 22 | 32 bytes | eight `tv[`*t*`]` dwords | when the object was built: year, month, day, day of the week, hour, minute, second, millisecond - not `SYSTEMTIME`'s order, which puts the day of the week third. It is a date on the **park's own calendar**, not the real one; the game writes it with `FileTimeToSystemTime` and reads it back with `SystemTimeToFileTime`, ignoring the day of the week; a stamp that will not convert loads as nought. The shipped park: the Belly Bounce and ten more read 2000-01-01 15:37:30, the bus 2000-01-27 05:10:00, the gates and the lights 2000-01-01 00:00:00 |
| 54 | 4 bytes | `MeshInstanceID` | the object's model instance; a coaster's record in the coasters module carries the same number (Temple Of Gloom in Alexah's played jungle park: 330) |
| 58 | 2 bytes | `mFlags` | see below |
| 60 | 132 bytes | 33 pairs of `mNameA[`*i*`]`, `mNameB[`*i*`]` | 2 bytes each |
| 192 | 4 bytes | `mRideScriptHandle` | the id of the object's running script in the ride scripts module |
| 196 | 4 bytes | `mTrackRideHandle` | nought, or for an item whose `Bumper.BumperType` is set, its slot in the track-rides module in the low byte and the `BumperType` above: `0xfffffc00` for a Dino Karts in slot 0. In all nine park files (the shipped park and eight played ones), non-zero on exactly the objects whose item has a `BumperType` |
| 200 | 4 bytes | `mState` | |
| 204 | 2 bytes | `mTopLeft` | |
| 206 | 2 bytes | `mEntryPos` | the cell a visitor is sent to |
| 208 | 2 bytes | `mNext` | this object's link in the object list |
| 210 | 2 bytes | `mAssignedStaffMember` | a handle: the member of staff sent to service it, `0` for none |
| 212 | 2 bytes | `mBackOfQueue` | |
| 214 | 4 bytes | `mCanLoad` | non-zero when the object will take anyone aboard |
| 218 | 2 bytes | `mExitPos` | packed like `mEntryPos` - the cell a guest is put down on when they leave |
| 220 | 2 bytes | `mFirstInQ` | |
| 222 | 4 bytes | `mIsTrackRideValid` | 1 on every object in the shipped park; 0 on Alexah's saved Temple Of Gloom, the one object in nine park files that reads 0 |
| 226 | 2 bytes | `mUpgradeParent` | |

After that come several ring buffers - each a `mCurrentEntry`, an `mNumEntries`, an `mWrappedAround` flag,
an `mTemp` and then `mNumEntries` entries of `mData[`*i*`]` - interleaved with `mNumCustomers` and
`mNumWalkAways`, and then a long tail of shop and ride fields: `mOperatingCapacity`,
`mOperatingDuration`, `mOperatingSpeed`, `mPersonBeingLoaded`, `mCostOfGoods`, `mQualityOfGoods`,
`mChanceOfWinning`, `mPricePerUse` (clamped to 0-500 as it is read), `mAmountOfSpecialIngredient`,
`mQueueSizeInCells`, `mRequestedService`, `mTimeMarkedForMaintenance`, `mTotalCosts`, `mTotalTakings`,
`mUpgradeBalloonSprite` and `mUpgradeLevel`.

**Those ring buffers are not empty, and the arithmetic is what says so.** An empty ring writes 13 bytes -
`mCurrentEntry` 4, `mNumEntries` 4, `mWrappedAround` 1, `mTemp` 4, then `mNumEntries` entries of 4 - and
laying the whole record out that way totals **379** against the **1,099** the record actually occupies. The
gap is exactly **720**, which is six rings of thirty entries at four bytes each. At thirty entries a ring is
133 bytes, and the record then closes on 1,099 **exactly** - the same kind of arithmetic that closes the map
cell on 52.

That puts `mOperatingCapacity` at 1034, `mOperatingDuration` at 1035, `mOperatingSpeed` at 1036,
`mPersonBeingLoaded` at 1040, `mCostOfGoods` at 1042, `mQualityOfGoods` at 1046, `mChanceOfWinning` at 1050,
**`mPricePerUse` at 1054**, `mAmountOfSpecialIngredient` at 1058, **`mQueueSizeInCells` at 1062**,
`mRequestedService` at 1078, `mTimeMarkedForMaintenance` at 1082, `mTotalCosts` at 1086, **`mTotalTakings`
at 1090**, `mUpgradeBalloonSprite` at 1094 and `mUpgradeLevel` at 1098, a byte: 0 on every object in the shipped park, and non-zero only
on the Belly Bounce in Alexah's played jungle park, at 1, beside speed 75, duration 30 and capacity 7, that level's starting values.

**The shop fields, measured across all nine park files** (the shipped Lost Kingdom and Alexah's eight):
`mQualityOfGoods` and `mAmountOfSpecialIngredient` hold 50 on every object in the shipped park and 0, 50 or 100 in
the played ones; `mCostOfGoods` is 20 on the Drinks Shop, 50 on the Jungle Spray and nought on every other shipped
object, and `mChanceOfWinning` 25 on the Jungle Spray and 100 on the rest; `mTotalCosts` is nought on every shipped
object. In the played parks every one of the seven sideshows holds a `mChanceOfWinning` of 55 to 58 (the Jungle
Spray 58). In a played park each shop's `mTotalCosts` is a whole number of its per-sale cost (the executable's
`FUN_004e1b40`, which reads the low byte of the two shop fields): a Drinks Shop at quality 0 and amount 100 books
10 a sale, 5,240 over 524 sales.

**The split is even, and the serialiser's order places every ring.** The writer (`FUN_004db7d0`) walks six
thirty-entry rings and the two counts in this order, each ring `mCurrentEntry` (4), `mNumEntries` (4, 30),
`mWrappedAround` (1), `mTemp` (4) and 30 `mData` (4 each), 133 bytes:

| File offset | Field | In the running game |
|---|---|---|
| 228 | a ring | today's costs, `+0xf8` |
| 361 | a ring | today's takings, `+0x70` |
| 494 | 4 bytes, `mNumCustomers` | every visit |
| 498 | a ring | today's customers, `+0x1a8` |
| 631 | 4 bytes, `mNumWalkAways` | |
| 635 | a ring | today's walk-aways, `+0x230` |
| 768 | a ring | today's served, `+0x2b8` |
| 901 | a ring | today's satisfaction, `+0x340` |

and 901 + 133 closes on `mOperatingCapacity` at 1034. The rings are the object's last thirty game days: what
each counts, and how the game rolls them, is OpenTPW's `docs/exe/ride-operation.md`, "The settle-up's
bookkeeping". Read this way, Alexah's two played jungle files agree with themselves on every object: each of the 46
rides, shops and toilets with customers has served equal to customers, today and over the thirty days, and a
sideshow has fewer (its winners: the Jungle Spray 30 of 43). In the shipped park every ring entry a reader reaches is
zero, as are both counts; every ring's `mNumEntries` is 30; most objects' rings have `mCurrentEntry` 1 and are wrapped,
things 11 and 12 are on 2, and thing 15 (item 1600, unplaced) has not wrapped (`mCurrentEntry` 5), its unreached
entries 6 to 29 holding `0xCDCDCDCD`, the build's fill for memory never written. All nine park files to hand (the
shipped one and Alexah's eight) hold 30 in every ring.

**Two of these offsets check all the others.** `mAngle` at 16 and `mId` at 20 fall out of laying the
serialiser's write order against the record - eight bytes of list head, then the map base's four shorts -
and they are exactly the two offsets an entirely separate reading, by emulating the loader, had already
produced. Since the arithmetic reproduces two known answers before it reaches any unknown one, the
running total behind `mFlags`, `mEntryPos` and `mNext` is carrying its own evidence.

**`mFlags` bit `0x4` is somewhere a guest may be *offered*.** The routine that walks the object list
looking for somewhere to send a guest tests exactly this before it will even score a candidate. It is not
"is a ride": the shipped park sets it on **six** objects, and the game's own catalogue names them - three
`Small Toilet`s, the `Drinks Shop`, the `Jungle Spray` sideshow and the `Belly Bounce` ride, one from each
of the three folders the game sorts its items into (`shops`, `sideshow`, `rides`). Choosing to visit a
toilet is a decision a guest makes like any other. What it excludes is the telling part: the object
carrying the rest-area bit is called `Staff Room`, and a guest has no business in one.

**`mFlags` bit `0x1` is a toilet and bit `0x2` a rest area.** Two searches in the executable read them,
one looking for the nearest toilet and one for the nearest rest area - and the second announces itself in
its own debug string, "Looking for rest area...". Both walk the object list from the header's
`mFirstObject`, follow `mNext`, and measure distance from the thing's cell. The shipped park has three
toilets and one rest area, and **the three toilets are three copies of one catalogue item standing in a
single column**, at (55,15), (55,16) and (55,17) - which is what says the bit is being read rather than
that some bit happens to be set.

**`mFlags` bit `0x40` holds litter and bit `0x80` is fireworks**, the item's `UsageInfo.HoldsLitter` and
`UsageInfo.IsFireworks`. A deciding guest carrying too much litter looks for the nearest object with `0x40`, and
one passing an object with `0x80` stops to watch it. In Lost Kingdom's `Easymode.TPWI` exactly one object carries
`0x40`, thing 17, the Litter Bin at (44,29), and none carries `0x80`; the other three themes' files were not
measured for these two bits.

**`mNext` is the object list's own link**, and walking it from `mFirstObject` reaches all fourteen objects
exactly once and stops on nought. Garbage does not terminate, so a chain that covers the list and ends
cleanly is itself the evidence for the offset.

**`mEntryPos` is a packed cell id - `y * 128 + x + 1`** - naming the cell people are sent to when they
want this object. It is the object's own cell or one beside it. The added one is the same packing the
staff patrol corners use, and the executable's own searches unpack it the same way: they build a cell as
`(byteAt7 * 0x80) + 1 + byteAt5` and then subtract one before splitting it with `& 0x7f` and `>> 7`.

**That one is easy to miss and plausibility will not catch it**, which is worth saying because it was
missed here first. Every object's entry is within two cells of it under either reading, so "it lands
beside the object" looks like confirmation of whichever decode is tried. What discriminates is
reachability: of the eleven placed objects five decode differently enough to matter, and all five are
walkable only with the one subtracted. The clearest is the rest area, whose entry unpacks to (58,15) - a
cell every member of staff can route to - where the plain reading gives (59,15), a cell with **no
connected edges at all**, which nothing could ever walk to.

**`mExitPos` is packed the same way**, and names the cell a guest is put down on when they leave rather
than the one they arrived at. The executable's point-fetching routine decodes both from the same code: a
non-zero argument selects the exit, nought the stand point, and each is returned as a fixed-point position
with the cell in the high byte and a sub-cell offset in the low one. Those offsets come from the item's
own `.sam` - the engine range-checks them and complains "Dodgy X exit point in SAM file" when they look
wrong - so they are data rather than anything computed.

**Here too the park will flatter a wrong decode, and worse than for `mEntryPos`.** `mExitPos` holds the
*same value* as `mEntryPos` on **ten of the eleven** placed objects, so counting how many are walkable
only with the one subtracted mostly restates `mEntryPos`'s answer through a field carrying the same
number - the two readings agree wherever the two fields do. Exactly one object separates them: the
`Belly Bounce`, entered from (52,23) and left from (52,26), three cells apart on opposite sides of it. Its
packed cell has sixteen connected edges where the plain reading gives (53,26), which has none. One object
is the whole of the evidence for this field, and it is worth knowing that rather than trusting a total.

**`mCanLoad` is non-zero when the object will take anyone aboard.** The filter deciding whether a guest
may be offered an object refuses on it, and so does the admission that follows. It is `1` on all fourteen
objects in the shipped park, so nothing there exercises the refusal. Closing an object - the park's door
shuts every one a guest may be offered, and each object's window has a door of its own - writes `0`, and
opening it writes `1`.

**`mRequestedService` is non-zero while a mechanic has been called to the object** - four bytes at 1078.
Calling one closes the object and writes `1`; a repair or a cancelled call writes `0`, and while it is set
the open check refuses the object, though leaving the track editor opens it anyway. It is `0` on all fourteen objects in the shipped park. Its offset rests on the
chain that closes on `mTotalTakings` at 1090 rather than on a value, since nought is also what its
neighbour after it holds.

**`mTimeMarkedForMaintenance` is the park clock when `mAssignedStaffMember` was last written** - four bytes at 1082,
a reading of `mGameTick`. The game stamps it as a handyman takes a toilet to clean or a mechanic a ride, and reads
it to forget the member: more than 100 ticks on, an assignment whose member has since taken other work is cleared,
and the stamp with it. A finished clean clears the member and the request and leaves the stamp. It is `0` on all
fourteen objects in the shipped park, where nobody is assigned to anything.

### The staff HQ (model 9): strikes and training budgets

Thing 1 in every park file, the one the header's `mStaffHQ` names, **103 bytes**. Its serialiser writes the
strike fields first and then hands a second writer the training block, which names its own two fields:

| Offset | Size | Name | Shipped park |
| --- | --- | --- | --- |
| 16 | 4 bytes | `mForceStrike` | `0` |
| 20 | 2 bytes | `mStaffMemberPickedUp` | `0` |
| 22 + 12*i* | 12 bytes each, *i* 0 to 4 | `mStrikeLevel[`*i*`]`: three dwords, the kind's strike level (0 to 4), whether it is on strike, and the game tick it was last looked at | `0`, `0`, `715`, on all five |
| 82 + 4*i* | 4 bytes each, *i* 0 to 4 | `mBudget[`*i*`]`, the monthly training budgets | `0` on all five |
| 102 | 1 byte | `mHaveEverTrained` | `0` |

The five records are in the order handymen, mechanics, entertainers, guards, researchers. `715` is the tick the
month first turned in the shipped park, 1 February 2000.

`16 + 4 + 2 + 5 × 12 + 5 × 4 + 1` is `103`, and in the shipped park the record ends exactly on the World
module's `DLRW`. The five budgets are, in order, the handymen's, the mechanics', the entertainers', the
guards' and the researchers' - models 5, 4, 6, 7 and 8, which is the order of the rows on the game's Staff
Training Budgets screen. What the game does with them each month is OpenTPW's `docs/exe/ride-operation.md`,
"The month's change".

### The park analyser (model 13)

Thing 5 in every park file, 73,544 bytes, named by the header's `mParkAnalyser`. Most of it is not decoded. Near
its end are **twenty rings of bytes, one per kind of opinion**, each 60 bytes: `mCurrentEntry` (4, from −1),
`mNumEntries` (4, 50), `mWrappedAround` (1), `mTemp` (1) and 50 `mData` (1 each), kind 0 first; after them the
record's last 76 bytes hold `mMisbehavingKids`, `mMisbehavingKidsThatGotAwayWithIt`, `mMisbehavingKidsEver`,
`mLifetimeVisitors`, `mLastStrike`, `mLastPuke`, `mLastPrank`, `mParkLastOpened`, `mLastObjectBuilt`,
`mLastStaffHired`, `mMostPaidForTicket`, `mLargestBusLoad` and `mLongestStay`, not measured one by one. In
`Easymode.TPWI` the rings start 72,268 bytes into the record (the only run of twenty such rings in the whole
stream): kind 0 holds eight samples of 50, the rest are empty, and `mLifetimeVisitors` reads 8. A played park
fills kinds 1 to 4, 6, 7, 9 to 13, 15, 16 and 18 too. What each kind is, and that only kind 0 is ever read back,
is OpenTPW's `docs/exe/ride-operation.md`, "The analyser's samples".

### The challenge manager (model 19)

Thing 10 in every park file, 937 bytes, named by no header field. The writer is the executable's `FUN_004d2320`, from
the World module's case `0x13` (OpenTPW's `docs/exe/ride-operation.md`, "Challenges", for what the game does with it).
Measured field by field in the nine park files above and a `restart.INTS` the original wrote under Proton (two more copies measured are the shipped `Easymode.TPWI`, byte for byte): the 16-byte
base thing, then **twenty 45-byte slots** from byte 16, then the manager's own 21 bytes.

| Slot offset | Size | Field |
|---|---|---|
| 0 | 4 | `mType`, the entry's `Type` (0: an empty slot) |
| 4 | 4 | `mTargetTime` |
| 8 | 4 | `mTargetVal` |
| 12 | 2 | `mTargetObj` |
| 14 | 2 | `mTargetObj2` |
| 16 | 4 | `mTargetStaffType` |
| 20 | 4 | `mPrize` |
| 24 | 4 | `mDoubleOrNothingFollowup`, the `.sam`'s `FollowupType` |
| 28 | 1 | `mCheckAtEndOnly` |
| 29 | 4 | `mChallengeOffered` |
| 33 | 4 | `mChallengeLost` |
| 37 | 4 | `mChallengeDeclined` |
| 41 | 1 | `mHaveWon` |
| 42 | 1 | `mIndependent` |
| 43 | 2 | `mThingIdForFollowup` |

Then `mCurrentChallenge` (4, at 916), `mNextChallengeEventTime` (8, at 920, an absolute time on the park's calendar),
`mSystemSwitchedOn` (1, 928), `mChallengeOn` (1, 929), `mWaitingForReply` (1, 930), `mValue` (4, 931) and
`mActualThing` (2, 935), ending at 937. The slots hold the theme's `ChallengesInThisLevel` entries in list order
([Settings and Modifiers](/formats/sam/)), field for field as `Challenges.sam` gives them. In every Instant Action file
the empty slots' unwritten fields are `0xCD` fill and `mCurrentChallenge` is −1 with the system off; in the Full
Simulation saves the empty slots are zero, and Alexah's jungle save has the system on, slot 0 (sell 30 drinks) won with
`mValue` 30, and its next event at 2002-05-05 19:22:30. `mChallengeOffered` and `mChallengeLost` are 0 in every slot of
every file, the won one too.

### The economy thing (model 16)

This is the thing `mBankAccount` names - thing `8` in the shipped park - and it is where a park's money
actually lives. Its record is **300 bytes**.

| Offset | Size | Name | Shipped park |
| --- | --- | --- | --- |
| 16 | 4 bytes | `mAdmissionFee` | `25` |
| 20 | 4 bytes | `mBalance` | `87987` |
| 24 | 4 bytes | `mBatchBalance` | `0` |
| 28 | 4 bytes | `mWithdrawalsEnabled` | `1` |
| 32 | 4 bytes | `mLastBalance` | `87787` |
| 36 | 4 bytes | `mTurnEnteredRed` | `0` |
| 40 | 4 bytes | `mProfitThisYear` | `-12013` |
| 44 + 32*i* | 32 bytes | `mLoans[`*i*`]` | eight slots, always written |

Each loan is eight 4-byte fields, in this order:

| Offset in loan | Name |
| --- | --- |
| 0 | `loan_available` |
| 4 | `amount_available` |
| 8 | `APR_in_percent` |
| 12 | `repayment_period_in_months` |
| 16 | `monthly_repayment` |
| 20 | `loan_bought` |
| 24 | `months_repaid` |
| 28 | `lenderNameIndex` |

`44 + 8 * 32` is `300`, which closes on the record size exactly. The in-memory struct agrees from the
other side: there the loan array runs from `+0x14` for `8 * 0x20` bytes and stops at `+0x114`, which is
precisely where the next named field, `mWithdrawalsEnabled`, sits.

**The check worth trusting is a different file.** The eight saved loans match `LoanInfo[0..7]` in
`data/levels/Standard.sam` field for field - amounts 100000, 50000, 25000, 10000, 18000, 30000, 80000,
65000 and periods 36, 36, 36, 36, 24, 30, 48, 30 - and every `monthly_repayment` is its own
`amount_available` divided by its own `repayment_period_in_months`, truncated, on all eight. A layout off
by one field, or by one loan's stride, could not reproduce sixteen unrelated numbers in order.

One reading falls out of that. Every saved `APR_in_percent` is **nought**, which matches
`jungle/Easy_Standard.sam` exactly and matches the global `Standard.sam` - whose rates are 20, 20, 20,
20, 23, 22, 18 and 21 - nowhere. The shipped park is an Instant Action park, and its own loans say so.

`mBalance` is the money and `mBatchBalance` is not a second copy of it: taking an admission fee adds it
to `mBalance` and to `mProfitThisYear`, and touches neither of the others. `mBatchBalance` is `0`, and
every `loan_bought` is `0`, in all nine park files.

### The person base

Every person - guest or staff - writes the same 390 bytes after its eight-byte list head, from the person
serialiser's write arm (`FUN_004f8b10`), which names each field and states its size. The list head and the
map base come first, as for every placed thing.

| Offset | Size | Name |
| --- | --- | --- |
| 8 | 2 bytes | `mX` |
| 10 | 2 bytes | `mY` |
| 12 | 2 bytes | `mMapChild` |
| 14 | 2 bytes | `mMapParent` |
| 16 | 4 bytes | `mSpriteScript` |
| 20 | 4 bytes | `mNextAnim` |
| 24 | 4 bytes | `mNextServiceInterval` |
| 28 | 2 bytes | `mAccurateDestX` |
| 30 | 2 bytes | `mAccurateDestY` |
| 32 | 2 bytes | `mAdjustorSpeed` |
| 34 | 2 bytes | `mBaseSpeed` |
| 36 | 1 byte | `mCount` |
| 37 | 4 bytes | `mESPSprite` - the sprite kind worn: 0 a child, 2 a costume, a member of staff their own |
| 41 | 2 bytes | `mLastRecordedMapId` |
| 43 | 177 bytes | the navigator - see below |
| 220 | 4 bytes | `mPreviousSpeed` |
| 224 | 4 bytes | `mPreviousX` |
| 228 | 4 bytes | `mPreviousY` |
| 232 | 4 bytes | `mStrandedTime` |
| 236 | 2 bytes | `mPurposeSpeed` |
| 238 | 4 bytes | `mSetDestSuccessfully` |
| 242 | 4 bytes | `mSpriteAngle` |
| 246 | 4 bytes | `mSpriteID` - the bank of that kind |
| 250 | 4 bytes | `mSpriteUnderRideCtrl` |
| 254 | 132 bytes | the event ring: `mActionHistIndex`, then 32 `mEventHistory[`*i*`]` of 4 |
| 386 | 4 bytes | `mLastThought` - the thought last set, 1 to 22, or 0 for none yet |
| 390 | 4 bytes | `mThoughtScript` - the thought bubble's slot in the sprite table, 0 with no bubble showing |
| 394 | 4 bytes | `mTimeBubbleShown` - the park's clock (`mGameTick`) when the last bubble was made |

It closes on 398, where each model's own block begins, and it lands on the two places read independently of
it: the navigator at 43 and `mSpriteAngle` at 242.

The three thought fields are measured on two played saves. Lost Kingdom at `mGameTick` 19,004, 339 guests: every
`mLastThought` is one of 0, 1, 2, 3, 4, 5, 6, 7, 9, 11, 14 and 16, no `mTimeBubbleShown` is above the park's clock,
and `mThoughtScript` is set on 21 guests, the oldest of whose bubbles was made at 18,989, 15 ticks before. Wonder
Land at 2,145, 53 guests: thoughts 0, 2, 3, 5, 6 and 9, and two bubbles, the older made at 2,132. `mStrandedTime` is
0 on every guest of both. In the shipped `Easymode.TPWI` all four are 0 on every guest.

**`mESPSprite` and `mSpriteID` are what the person's sprite is built again from**, and they equal that sprite's own kind
and bank (the `TPCS` record's `+0xac` and `+0xb0`) on all 18 people of the shipped park and every person holding a
sprite in Alexah's played parks; the rest hold no sprite (59 riding guests and 5 staff in the jungle park, 7 riding
guests in the fantasy park). A child's bank is fixed by the guest's
id: the game's generator seeded with the id word, one draw, `(r >> 2) %` the kid banks loaded - all 13 shipped children
over eight, and all 296 jungle and 53 fantasy children of the played parks over six, the count at medium and high detail
(see [sprites](../sprites/)). So a park saved with eight kid banks holds children on banks 6 and 7 (the shipped park's
guests 33, 35 and 29) that a game loading six brings down to 0 and 1. Alexah's jungle park has 43 guests in costume,
`mESPSprite` 2, bank 0, the theme's one costume.

Two parts of it are worth knowing about before the models' own fields. The navigator is the person's
**entire navigation state** - 177 bytes of steering and path data - so a person resumes the route they were
walking rather than re-planning on load. The event ring is a 32-entry ring of `[u16 type][u16 param]` with
its index, which the played parks bear out: the low word a small event number, the high word a thing id
where one applies. The ring is read only by a debug dump whose logger is an empty function in the release
build, so nothing a player sees comes from it (OpenTPW's `docs/exe/park-engine.md`).

Two fields are useful on their own. **`mSpriteScript` at +16** is the person's slot in the park's table of
sprites (`TPCS`, below): the shipped park's eighteen people hold exactly its eighteen live slots. **`mSpriteAngle`
at +242** (`0xf2`) is an 11-bit heading, `0` to `0x7ff`.

**The four speed fields are a person's walking speed** (OpenTPW's `docs/exe/ride-operation.md`, "Where a
WALKING peep is drawn"). `mBaseSpeed` (+34) is one of 60, 80, 100, 120 and 140; `mPurposeSpeed` (+236) is a hurry
of 0, 25 or 50; `mAdjustorSpeed` (+32) is an ice cream's few; and `mPreviousSpeed` (+220), a float, is the speed
the three sum to over a hundred, eased a quarter of the way each turn, which the navigator's `max_speed` is made
from (below). A person who has walked at one pace long enough holds the three summed over a hundred: fourteen of the
shipped park's eighteen, and 320 and 317 of the played saves' 392 (304 and 300 hold their base alone over a hundred,
the rest hurrying at 25). In the shipped park the hurry and the
ice cream are 0 on everyone; its guests' bases are 60 (four), 100 (four), 120 (three) and 140 (two), and three
guests at base 100 hold 1.0105584 and the guard at base 120 holds 1.2047515. The played saves' guests spread
74, 56, 70, 62 and 77 over the five bases.

**`mCount` is a guest's walking-turn count**: the walk to a chosen thing adds one each turn and, on the
twelfth, zeroes it and makes its minor decision. In the shipped park it is 0 on all eighteen people. In
Alexah's played parks it spans exactly 0 to 11 - a count reset at twelve never shows 12 - on every guest:
339 in each of two jungle saves (264 and 262 of them non-zero) and 53 in each of two fantasy saves (39
non-zero); the jungle saves' staff all hold 0.

#### The navigator - 177 bytes at +43

Every person carries one, staff included, because the person base reads it for all six models. Its fields
are written **alphabetically by name**, like the rest of the save, so each offset is the sum of the sizes
before it — and they close on exactly 177:

```text
+43  i32×2 force              +51  i32×2 formation_pos    +59  i32×2 local_xaxis
+67  i32×2 local_yaxis        +75  i32   mass             +79  i32   max_force
+83  i32   max_speed          +87  i32   mCantReachDest   +91  i32   nav_mode
+95  i32   path_buffer_count  +99  i32   path_count       +103 u8    path_finished
+104 i32   path_last_progress +108 i32   path_stuck_buffer
+112 i32   path_subpath_dist  +116 i32   path_tail_dist   +120 i32×2 path_target_pos
+128 i32   path_timestamp     +132 i32   path_total_count +136 i32   path_total_dist
+140 i32×2 position           +148 i32   radius
+152 5 × (i32×2 subpath_buffer[i] + i32 subpath_dist[i]) = 60 bytes
+212 i32×2 velocity                                              = 177
```

Three of those carry **no name in the binary** — `mass`, `position` and `velocity` above. Each is named
by four things agreeing: the alphabetical slot it has to occupy, how the steering loop uses it, the
Reynolds-steering vocabulary the block's other names come from, and the constructor, which writes `1.0`,
`0.2`, `0.4` and `0.2` to mass, radius, max force and max speed at exactly those places.

**Everything here is 16.16 fixed point**, where `65536` is one map cell. A position is therefore in
65536ths of a cell — 256 times finer than the `mX`/`mY` every thing carries — and the engine reaches
`mX`/`mY` from it by shifting right by eight.

> **How to check a reader of this block**, and it is a strong check: `position >> 8` must equal the `mX`
> and `mY` stored separately at `+8` and `+10`. Those were not used to place this block, so agreement is
> real evidence. Across the shipped park all eighteen people match, at eighteen distinct positions, and
> none matches at a base shifted by −8, −4, +4 or +8.

Some of it is measurable rather than merely readable. **`max_speed` and `max_force` are made from the person
base's `mPreviousSpeed` (+220)**: held to at most 2.0, times 13107.2 and 26214.4, each truncated and at least 655.
That is exact on all eighteen people of the shipped park and on all 392 of each of Alexah's two played Lost
Kingdom saves, so `max_force` is twice `max_speed` or one unit (a 65536th of a cell) more: exactly twice on nine
of the eighteen, one more on the other nine. `mass` is `1.0` and `radius` `0.2` of a cell on every one of them.

:::caution[`subpath_dist` is only partly meaningful]
When a route is set, only **`path_buffer_count − 1`** of the five distances are written — a one-waypoint
route writes none at all — so an unused slot keeps whatever the previous route left in it. In the shipped
park sixteen of the eighteen people are on one-waypoint routes: nine hold `0xCDCDCDCD`, uninitialised fill
saved verbatim, in the first distance slot, and seven a stale real distance. Only
`i < path_buffer_count − 1` means anything.
:::

Two details make the rest of the block legible. Waypoints are whole cells walked to at their **centre**,
stored as `cell × 65536 + 32768`. And every distance in it — the legs, the totals, the arrival tests — is
an **octagonal approximation**, `ax + ay − min(ax, ay) / 2` with the halving truncated, never a real
square root; the three legs the shipped park holds come back at exactly the distances it saved.

### A guest (model 1)

The sixth person model is the visitor, and its block begins at **+398**, after the same eight-byte list
head and 390-byte person base a staff member has. It is **135 bytes**, making the record `8 + 390 + 135`
= **533**.

| Offset | Size | Name | Shipped park |
| --- | --- | --- | --- |
| 398 | 4 bytes | `mArrivalDate` | `648` to `660`, one apart in id order: the world's `mGameTick` (755 in this file) as the guest was made |
| 402 | 4 bytes | `mArrivalIndex` | |
| 406 | 4 bytes | `mBalloonScript` | `0` on all thirteen: the one-based slot of the guest's balloon in the `TPCS` sprite table, or none (below) |
| 410 | 4 bytes | `mBeenAdmitted` | |
| 414 | 4 bytes | `mCash` | `684`, `510`, `654`, ... twelve different amounts across thirteen guests |
| 418 | 4 bytes | `mExitLevel` | `142`, `98`, `57`, ... |
| 422 | 4 bytes | *unnamed float* - happiness | `50` on all thirteen, what a new guest is constructed with |
| 426 | 4 bytes | *unnamed float* - hunger | `18`, `25`, `61`, ... |
| 430 | 4 bytes | `mLastPosX` | a float in world units, ten to a cell: where the held balloon goes next frame, across |
| 434 | 4 bytes | `mLastPosY` | the same, down |
| 438 | 4 bytes | *unnamed float* - litter carried | |
| 442 | 2 bytes | `mMajorDest` | thing handle - what they have chosen, or none |
| 444 | 4 bytes | `mNumRides` | `0` on all thirteen shipped guests; 287 of the 339 in Alexah's played jungle file hold some of these four |
| 448 | 4 bytes | `mNumShops` | `0` on all thirteen |
| 452 | 4 bytes | `mNumSideshows` | `0` on all thirteen |
| 456 | 4 bytes | `mNumSideshowsWon` | `0` on all thirteen; never above `mNumSideshows` in any file to hand |
| 460 | 4 bytes | `mPaidAdmission` | |
| 464 | 4 bytes | `mParkOpeningWaitingTime` | |
| 468 | 1 byte | `mPersonType` | `5`, `7`, `2`, ... an index into `PeepTypes[0..7]` |
| 469 | 1 byte | `mPrankeryIndex` | |
| 470 | 16 bytes | `mPreviousRides[`*i*`]` and `mPreviousTemporaryRides[`*i*`]` | **interleaved in pairs**, four of each, 2 bytes apiece, thing handles newest first; `0` on all thirteen |
| 486 | 2 bytes | `mQNext` | `0` on all thirteen |
| 488 | 2 bytes | `mQPrev` | `0` on all thirteen |
| 490 | 4 bytes | `mQueueMoveDelay` | |
| 494 | 1 byte | `mQueuePos` | `0` on all thirteen |
| 495 | 4 bytes | `mRemainingBalloonLife` | `0` on all thirteen: the needs turns the balloon has left, kept while it is not showing |
| 499 | 2 bytes | `mSavedMajorDest` | thing handle - where they were going when something on the way took them, to go on to after it, or none; `0` on all thirteen |
| 501 | 4 bytes | `mSavedState` | `6` on all thirteen - a new guest is constructed deciding |
| 505 | 4 bytes | `mState` | `2`, `5`, `3` - heading for the gate, entering, waiting outside |
| 509 | 4 bytes | *unnamed float* - thirst | `36`, `13`, `12`, ... |
| 513 | 4 bytes | `mTimeOfLastSpotAnim` | `0` on all thirteen: a reading of `mGameTick`, nought until the first spot animation |
| 517 | 4 bytes | `mTimeStartedIdling` | `0` on all thirteen: a reading of `mGameTick`, nought until the guest first queues or stands idle |
| 521 | 4 bytes | *unnamed float* - `mTiredness` | `0` throughout |
| 525 | 4 bytes | *unnamed float* - toilet | `13`, `15`, `24`, ... |
| 529 | 4 bytes | *unnamed float* - vomit, the meter the game's own log calls illness | |

**The block closes on 533 exactly**, and that is what makes the offsets above worth trusting. They are not
measured one at a time: they are produced by walking the serialiser's own declared field sizes from +398,
and that single walk has to land on the record size - derived separately, by running the original's reader
and logging what it declared - or every offset in it is wrong together.

**The shipped park cannot check the two histories; a played one can.** Every guest in `Easymode.TPWI` has both
empty, so a reader at the wrong offset would read the same noughts. Two saves of
one jungle park played in the original to `mGameTick` 19,004 and 19,007 - not files the game ships - each hold
1,060 non-zero entries across 339 guests, and every one is the thing handle of an object in the same save;
none follows a nought, which is what a newest-first list of four looks like. The same saves hold a non-zero
`mSavedMajorDest` on 51 and 50 of the 339 guests, 31 of them a toilet in each, and two fantasy saves on 6 of
53; every one names an object in its own save.

**The balloon is its own sprite, joined by the slot.** `mBalloonScript` is not a handle into anything else: it names
a slot of the same `TPCS` table the guest's own `mSpriteScript` names, one-based, `0` for none. The shipped park has no
balloons. The two played jungle saves hold 29 and 28, and two fantasy saves 11 each, every one a kind-10 record (`balloons`) of bank 0, frame 0, alpha
255, state 2, on script word 1650 at 1652, its set 0 to 3 (the colour, see [sprites](../sprites/)) and its position
equal to its guest's `mLastPosX` and `mLastPosY`; its height above the ground runs 1.00 to 1.30 for a guest standing
still and lower, down to -1.2, for one walking. `mRemainingBalloonLife` is non-zero on 35 guests of each: the six and seven more
are guests riding (state 16), whose balloon is not showing, and the six lowest lives (2, 10, 11, 18, 21, 22) are one
lower in the later of the two saves. `mLastPosX` and `mLastPosY` sit between hunger and litter by the same alphabetical
rule as everything else here.

The two queue links deserve a note, because finding them turned entirely on the **name**. Sweeping the
executable's serialised field names for `InQ`, `mNext`, `Queue` and `mPrev` finds no per-person queue link
at all, and it is tempting to conclude the queue is rebuilt at load time. It is not: the field is `mQNext`,
with `mQPrev` beside it, and no one of those four guesses reaches either spelling. Reading the guest
serialiser's whole field list is what finds them, and the original's own diagnostic confirms the pair -
*"Person %d is in queue for object %d (next %d, prev %d) but doesn't think he is"*. A queue is therefore
**doubly linked through the guests themselves**, headed by the object's `mFirstInQ`, and its length is
found by walking it. Mind that `mFirstInQ` is a **thing handle** while `mBackOfQueue`, two bytes before it,
is a **packed cell id**: they are adjacent and they are not the same kind of number.

The unnamed floats are placed the way the staff block's are - this block is written in alphabetical order,
so an unnamed field's name is pinned by where it sorts. That is also what names the one at 521: it falls
between `mTimeStartedIdling` and `mToilet`, which leaves `mTiredness`. Unlike the staff block, whose order
transposes one pair, the guest block's alphabetical order holds throughout. It is also why the last float
is vomit and cannot be a field called illness: `mIllness` would sort between `mHunger` at 426 and
`mLastPosX` at 430, which are adjacent, and 529 sorts after `mToilet`. Illness is the log's and the balance
file's word for what that meter measures (`PeepInfo.DecisionVarIllnessWeight`, `RegionFX[i].Illness`), while
each item's `UsageInfo.VomitEffect` - "How much vomit to add" - names it the other way.

The six needs - happiness, hunger, litter, thirst, toilet and vomit - are floats the engine clamps to
`0..100`. Five are named by the engine's own logging: it prints `thirst`, `hunger`, `toilet` and `illness`
while scoring which ride a guest will choose, and prints `Litter gone up by %d, is now %d` over the litter
field.

> **A corroboration worth repeating, because it is the reason to trust the naming.** The balance file
> `data/levels/Standard.sam` has `PeepInfo.DecisionVar…Weight` keys that run **Dist, Queue, Excitement,
> Thirst, Hunger, Toilet, Illness** — the same seven terms in the same order that the scoring code
> multiplies. A text file written by the developers agrees with the disassembly about which float is which.

`mState` is one of 22 behaviour states, and `mSavedState` is the one to return to after a one-off
animation. In the shipped park every guest's `mState` reads 2, 3 or 5, and that matches where the same
guests stand on the map.

> **How to check a reader of this block.** Four things in the shipped park are specific rather than
> merely in range, and a map that is even one byte out fails all of them: happiness and `mSavedState` each
> hold the single value the table gives on every guest, every need is a whole number, and each
> guest's `mCash` falls within `PeepInfo.StartingCashVarPc` (15%) of `PeepTypes[mPersonType].StartingCash`
> from the balance file. Note that a plain range test on the floats is nearly useless here: a small
> integer read as a float is a denormal of about `1e-43`, which passes any `0..100` check, so shifting
> the base by a few bytes still appears to work.

### A member of staff (models 4 to 8)

Five of the six person models are staff - **4** mechanic, **5** handyman, **6** entertainer, **7** guard,
**8** researcher - and all five share one block, because the original gives them one class. It begins at
**+398**, the same place a guest's own block begins: both follow the eight-byte list head and the
390-byte person base. It is **105 bytes**, and each kind then adds a few fields of its own.

| Offset | Size | Name | Shipped park (25, 26, 27, 28, 30) |
| --- | --- | --- | --- |
| 398 | 4 bytes | `mCurrentPayGrade` | `3`, `3`, `3`, `3`, `2` |
| 402 | 4 bytes | *unnamed float* - happiness | `89`, `89`, `92`, `91`, `97` |
| 406 | 4 bytes | `mJobsDone` | `0` on all five |
| 410 | 66 bytes | `mName[0..32]` | The name as text: 33 characters of 16 bits, ended by a nought. `Duke Mighten`, `Mike Cooper`, `Pierre Hintze`, `Mike Man`, `Nicholas Ricks` |
| 476 | 2 bytes | `mPatrolRegionBL` | packed cell id |
| 478 | 2 bytes | `mPatrolRegionTR` | packed cell id |
| 480 | 1 byte | `mPercentageThroughGrade` | `0` on all five |
| 481 | 2 bytes | `mRestArea` | `0` on all five |
| 483 | 4 bytes | `mState` | `1`, `1`, `1`, `0`, `1` |
| 487 | 4 bytes | `mTimeStartedIdling` | `0`, `0`, `712`, `752`, `0` |
| 491 | 8 bytes | `mTimeHired` | |
| 499 | 4 bytes | *unnamed float* - tiredness | `75`, `75`, `82`, `79`, `93` |

Then, at **+503**, whatever the kind adds:

| Model | Fields, in order | Bytes | Record |
| --- | --- | --- | --- |
| 4 mechanic | `mDurationOfRepair` 4, `mObjectToRepair` 2, `mNext` 2 | 8 | 511 |
| 5 handyman | `mTargetLitterCell` 2, `mTimeStartedCleaning` 4, `mToiletToClean` 2, `mNext` 2 | 10 | 513 |
| 6 entertainer | `mTimeStartedEntertaining` 4, `mNext` 2 | 6 | 509 |
| 7 guard | `mPerp` 2, `mProsecutionTimestamp` 4, `mNext` 2 | 8 | 511 |
| 8 researcher | `mTimeStartedResearching` 4, `mNext` 2 | 6 | 509 |

**The sizes close five ways at once, which is the check worth trusting here.** `8 + 390 + 105` is `503`,
and the five record sizes leave exactly `8`, `10`, `6`, `8` and `6` over it - which is precisely what each
kind's own serialiser declares. The record sizes were derived by a completely different route (running the
original's reader and logging what it declared), so the two never shared a step.

Two of the block's fields carry no name in the binary, and they are placed the way the navigator's
unnamed fields were: the block is written in alphabetical order, so an unnamed field's name is pinned by
where it sorts. The one at 402 falls between `mCurrentPayGrade` and `mJobsDone`, and the one at 499 after
`mTimeHired`. What the code does with them agrees - the resting handler recovers the first by
`HappinessRecuperationRate` and the second by `RecuperationRate`, both indexed by the pay grade, and the
"too tired to work" test reads the second against `AllStaffConstants.RestLevel`.

**The alphabetical rule is not quite a rule here, though, and that is worth knowing rather than relying
on.** `mTimeStartedIdling` is written *before* `mTimeHired`, which sorts the other way. The order above is
the serialiser's own, because the serialiser is what the file follows.

**What the stamps count.** `mTimeStartedIdling` is a reading of the World header's `mGameTick`, the park's own
clock, which goes up once a thing sweep (248 ms). The save writes that clock and every member of staff in one pass,
so a stamp is never ahead of it: the shipped park's `712` and `752` sit just under its `755`. **Nought is not "since
the start"**: the game writes `0` whenever a member of staff stops to idle from anything but a walk, and that wait is
over on the next sweep. Three of the shipped five carry it, and a walk does not clear a stamp - the entertainer is
saved walking with `712`, left from their last idle. Of the kinds' own fields, `mTimeStartedCleaning`,
`mTimeStartedEntertaining` and `mTimeStartedResearching` are readings of the same clock, taken as the job began;
**`mDurationOfRepair` and `mProsecutionTimestamp` are not readings at all**, whatever the second's name says. Each
holds the sweeps left, loaded from its kind's `WorkDuration` (the mechanic's scaled by what the ride needs) and
taken down by one a sweep.

A patrol region is a rectangle, stored as two **packed cell ids** - `y * 128 + 1 + x`, the same one-based
packing the destination setter takes - naming the bottom-left and top-right corners. Nought means no area
at all. Unpacking the shipped park's gives places that mean something: the entertainer patrols `(47,18)`
to `(48,25)`, the two columns running south from the gateway cells at `(47,17)` and `(48,17)`, and the
researcher's is `(0,0)` to `(127,127)`, the whole map.

**One trap is worth spelling out, because three different numberings of these five kinds are in use.**
The thing model runs mechanic 4 to researcher 8, as above. The sprite folders in `esprites.wad` run
entertainers 4, handymen 5, mechanics 6, guards 7, researchers 8. And the balance file's
`PerTypeStaffConsts` runs handyman 0, mechanic 1, entertainer 2, guard 3, researcher 4. So the mechanic is
model 4, wears sprite type 6, and is paid as type 1; crossing any two of them reads the wrong constants
while still looking entirely plausible.

## The sprite table (`TPCS`)

Directly after the world block's `DLRW` trailer sits the table of the park's sprites - the guests and
staff walking about. It is written as:

| Size | Description |
| --- | --- |
| 4 bytes | Tag - `TPCS` |
| 4 bytes | The size of one record - `0x118`, 280 bytes |
| 4 bytes | How many slots the table has |
| 4 bytes x slots | One handle per slot; zero where the slot is empty |
| 280 bytes x live | One record per **non-zero** handle, in slot order |

Slot 0 is never used. The park the game ships has 100 slots of which 18 are live, and those 18 are
exactly its people: thirteen from the `kids` banks and one each from `entertainers`, `handymen`,
`mechanics`, `guards` and `researchers`.

A record is a runtime structure written out whole, so most of it is bookkeeping. The fields that can
be named from the code that fills them are:

| Offset | Size | Description |
| --- | --- | --- |
| `0x08` | 4 bytes | How far into that program it has got. Because showing a frame is the only thing that ends a turn, a saved value always rests just past a frame instruction |
| `0x0C` | 4 bytes | **Which** animation program - the index of its first instruction, in the same array `0x08` counts into. A jump inside a program moves `0x08` and leaves this alone, so it names the program the sprite was *started* on rather than where it has reached |
| `0x14` | 4 bytes | Where that program starts; re-pointed on load, so the stored value is meaningless |
| `0x18` | 4 bytes | State |
| `0x7C` | 4 bytes | When this sprite is next due to step |
| `0x80` | 4 bytes | How long between steps |
| `0x88` | 4 bytes | Where it stands across the map, a float, in world units - ten to a map cell |
| `0x8C` | 4 bytes | How far above the ground, a float. `0` on every person in the shipped park; the game looks up the land underneath and adds it as it draws |
| `0x90` | 4 bytes | Where it stands down the map, a float, in the same units |
| `0xA0` | 4 bytes | Alpha - `255` throughout the shipped park |
| `0xA4` | 4 bytes | Scale across, a float - `1.0` throughout |
| `0xA8` | 4 bytes | Scale down, a float - `1.0` throughout |
| `0xAC` | 4 bytes | Which kind of sprite this is - an index into the table of fourteen in [Sprites](/formats/sprites/) |
| `0xB0` | 4 bytes | Which bank of that kind |
| `0xB4` | 4 bytes | Two numbers in one: the low four bits are the **set**, and everything above them is how far past its kind's first bank this sprite's bank sits. The game takes it apart exactly that way before it looks a picture up |
| `0xB8` | 4 bytes | Which frame of that set |
| `0xC0` | 4 bytes | Which of eight ways round it was last drawn facing |

The three floats at `0x88`, `0x8C` and `0x90` are where the sprite stands, in world units at ten to a
map cell.

An earlier version of this page said the record carried **no position at all**. That was wrong, and
wrong for a reason worth keeping: the scan that went looking for one swept for values shaped like map
cells, which run to the tens, while these are world units and run to the hundreds. It reported nothing
because nothing it could see was there. A negative result is only ever as wide as the encoding it
assumed.

The thing that owns the sprite knows where it is too, as `mX` and `mY` in 256ths of a cell, and the two
agree: across all eighteen of the shipped park's people the two readings differ by less than a third of
a world unit. The thing is the better source of the two, because it is what the park saved rather than
where the runtime last drew.

### The animation programs

`0x0C` and `0x08` are indices into an array of animation programs, and **that array is compiled into the
executable rather than stored in the save**. `CSPS` - the trailer closing the `TPCS` block, and despite
its name - marks the table of sprite *instances* above and not the programs: the loader points every instance at the built-in array unconditionally and
reads no bytecode from the file at all, which is also why `0x14` is meaningless on disk.

The array holds 83 programs back to back. Each word in it is either an opcode or an operand of the one
before it, so it can only be read by walking it from the start - and walking it with the wrong operand
widths lands on a word that is not an opcode, which is what makes the widths checkable rather than
assumed. Of its eighteen opcodes, a person's animation uses four:

| Opcode | Operands | What it does |
| --- | --- | --- |
| Set local | 2 | Writes a value into one of the instance's own words. Every person program opens with the same one, and nothing anywhere reads it back |
| Choose set | 1 | Writes `0xB4` - the set and the bank offset together, as one word. It does **not** end the turn |
| Show frame | 1 | Writes `0xB8` and **ends the turn**. 581 of the array's 863 instructions are this one |
| Jump | 1 | Moves `0x08`. It does **not** change `0x0C` |

So a person's program is always the same shape: choose a set, show some frames, jump. None of them
contains a loop or a branch. The jump is usually back to the program's own first instruction, which is
what makes a walk cycle; the one-shot animations instead jump into the standing program, so they play
once and settle.

Because choosing a set does not end a turn, a program's set and its first frame appear together - a
sprite is never seen for a turn wearing the set it had before. And because showing a frame does end one,
the shipped park's sixteen walkers are stopped at **seven different positions** of the same eight-frame
walk, which is what keeps a whole park from stepping in time with itself when it loads.

Programs are stepped by a system that runs **once every two of the game's 31ms ticks**, so every 62ms.
`0x7C` is when a sprite is next due and `0x80` is how long it waits between turns; the test is a strict
"is now past it", and the interval a sprite is created with is 62 - exactly one turn - so a sprite left
alone comes due every *other* turn, at 124ms. A walking person's interval is driven from the distance
they moved that step, doubled if they are **not** hurrying, and in practice that arithmetic only ever
yields nothing, one or two. The effect is therefore a doubling of the animation rate rather than a
continuous control of it. Nothing is written when it yields nothing, so "no distance moved" leaves the
interval alone rather than setting it to zero.

The ceiling of 250 that the same code applies cannot be reached from a walk at all: the distance is
squared as a 32-bit integer, and a step large enough to want an interval past 250 would overflow that
thousands of times over before the ceiling could apply.

## The particles module (`TRAP`)

The particle system as two memory images, between the sprite scripts' `CSPS` tag and the particles' own `TRAP`.
Its magic reads `LCTP` in a dump: it is the game's `PTCL`, stored little-endian like the tags. Written by `FUN_0051f680`, which logs its failure as
`PAR_SaveStatus`.

```text
char[4]  magic              "LCTP"
u32      enabled            1: particles were on, and the rest follows
u32      size               0x9e68
byte[0x9e68]  live system
u32      size               0x8b60
byte[0x8b60]  templates
u32      pool size          0: the pool is always written empty
```

That is 76,252 bytes, and it is so in all nine park files: each reads `LCTP`, 1, `0x9e68`, `0x8b60` and 0.

The **live system** is a `0x2c`-byte header, 120 emitters of `0x140` bytes, 20 effectors of `0x68` bytes and
`0x1c` bytes more. Of those last, the first five words are list heads: the used emitters, the used effectors,
the free emitters, the free effectors and the free particles. A sixth word and the 16 bytes after it are nought
in all nine files. The header's `+0x0c` is the particle count, 2048 in all nine. The used emitters are a chain through
each emitter's `+0xd0`: four in `Easymode.TPWI` (two `Button`, `Bubbles` in emitter slot 20, `WaterFall` in emitter slot 0).

The **templates** are the effect library: 105 effects of `0x140` bytes and 20 effectors of `0x68`, the
[particle library](../particles/)'s records. In all nine files the 105 effects equal `data/Particle/Tp2.plb`
byte for byte, but for slots 101 and 102 in the four jungle files, which hold the jungle's two `.emt` effects,
`Smoke` and `BeamUp`. The 20 effectors equal the library's byte for byte in all nine files.

The particles themselves (`0x34` bytes each) are never saved: the pool size is always 0, and a load starts
each emitter with no particles.

## The message centre module (`SSEM`)

Which things hear which message: one listener set for each of the game's 29 message types, between the
particles' `TRAP` and the message centre's own `SSEM`.

```text
u32  set count                 29
per message type, 0 to 28:
  u32  count
  u16  thing id  x count       ascending
```

In the shipped park that is 266 bytes: `4 + 29 × 4` for the counts and 73 ids. Each set is written in
ascending id, and a load replaces whatever the running park's constructors had registered with these sets
as they stand, so a set is the file's word on who is told, and in what order. Set `0xc`, the month's change,
is things 1, 4, 5 and 8 and then every member of staff in all nine park files: `1, 4, 5, 8, 25, 26, 27, 28,
30` in the shipped park, and `1, 4, 5, 8` in the four park files with no staff. Set `0xb`, the day's change, is
the fourteen catalogue objects and thing 10; set `0xd`, the year's change, is thing 8, the economy thing,
alone. What each message does is OpenTPW's `docs/exe/ride-operation.md`, "The month's change".

## The clock module (`KOLC`)

Two dwords, between the message centre's `SSEM` tag and the clock's own `KOLC`, so the second tag sits exactly
twelve bytes after the first:

| Size    | Description                                                                                   |
| ------- | --------------------------------------------------------------------------------------------- |
| 4 bytes | The game clock's reading at the save, in milliseconds - the clock the ride scripts and the animation channels read |
| 4 bytes | A second stopwatch's reading, which an animation channel carrying flag `0x40` reads instead    |

The game makes both clocks read these values again when it loads a park, so every other reading of the clock in the
file - a script's deadlines, a channel's time stamps - keeps its distance from the save's own moment. In
`Easymode.TPWI` they are `0x06D13894` (114,374,804) and `0x06D8DF7E` (114,876,286); in a played jungle park,
`0x00498D70` (4,820,336, about 80 minutes) and `0x0049A676` (4,826,742).

## The game system module (`SYSG`)

Nine dwords, between the "vanilla time" `TNAV` tag and the game system's own `SYSG`, so the module is 36 bytes in
all nine park files.

| Dword | Description |
| ----- | ----------- |
| 0 to 4 | Five values of the game system. A load reads all five; it then overwrites the first three from dwords 6 to 8 |
| 5 | Not a value: the writer fills it from a stack slot it never sets. It reads 2 in all eight played files, and `0x0075DBC8` in `Easymode.TPWI`. A load skips it |
| 6 to 8 | The first three values again, each written less a clock reading the game keeps (equal to the clock module's first dword in both files compared); a load adds its own reading back |

In `Easymode.TPWI` dwords 6 to 8 are 2, -29 and -215; in a played jungle park, 20, -11 and -11.

## The ride script module (`ESSR`)

Every script that was running when the park was saved is stored here, program counter and all — which
is what lets a loaded park resume instead of starting over. It begins immediately after the `RYLF` tag
closing the module before it, and unlike the tags its own magic is stored **forwards**.

| Size     | Description                                                     |
| -------- | --------------------------------------------------------------- |
| 4 bytes  | Magic — `RSSE` (`52 53 53 45`)                                  |
| 4 bytes  | Header length                                                   |
| n bytes  | Header — the subsystem's own globals, not per-script: see below |
| 20 bytes | Five dwords the loader reads and discards                       |
| 4 bytes  | Script count — **14** in the shipped park                       |
| 4 bytes  | Bytes per script struct — **244** in the shipped park           |

The header is the script subsystem's five global dwords, which the game reads straight back over its own: a flag that
the subsystem is set up, the tick counter behind every script's one-in-eight turn, the handle the next new script will
be given, the script count, and a list pointer from the saving session that means nothing on loading. The shipped
park's read 1, 6,055, 16, 14 and a pointer. So a loaded park goes on counting ticks, and numbering new scripts, where
the saved one left off.

Then one record per script, **newest first**: the saving session's list order, handles 15 down to 1 in the shipped
park, 5 absent. The game puts each at the head of its list as it reads it, so a loaded park walks them oldest first.
Each is a struct followed by a run of length-prefixed blocks, and a record
**must** end on the literal guard `OBJ ` — the game refuses the load without it, logging
`RSSE: Load Fail - Object list missing`, which is what makes a walk of this module self-checking.

| Size     | Description                                                             |
| -------- | ----------------------------------------------------------------------- |
| 244      | The script struct — see below                                           |
| 4 + n    | The script body, as words                                               |
| 4 + n    | **Block 1 — the stack**, one dword per slot; empty with no `#setstack`  |
| 4 + n    | **Block 2 — the variables**, one dword per slot                         |
| 4 + n    | **Block 3 — the string blob**, byte for byte the `.RSE` file's          |
| 4 + n    | Blocks 4 and 5                                                          |
| 4 + n×32 | **The walk slots** — a count, and that many 32-byte slots (below)       |
| 4 + n    | **The head table** — one dword per head slot (below)                    |
| 4 + n    | The script's directory, the string the game's loader keeps at `+0x38`   |
| 4 bytes  | Guard — `OBJ `                                                          |
| 4 bytes  | Object count                                                            |
| 4 bytes  | Bytes per object                                                        |
| n        | The script's own object list                                            |

Fourteen fields of the struct are established:

| Offset | Dword | Description                                                                     |
| ------ | ----- | ------------------------------------------------------------------------------- |
| `0x08` | 2     | The script handle — the number an object record's `mRideScriptHandle` also holds |
| `0x3c` | 15    | The program counter, in **words** from the start of the body                     |
| `0x40` | 16    | The call index — the stack slot the next `JSR` writes, counting down              |
| `0x44` | 17    | The heap index — how many values `HUSH` has pushed, counting up                   |
| `0x48` | 18    | The result register, which the conditional branches test                         |
| `0x50` | 20    | The body length in words, which the counter is bounds-checked against            |
| `0x54` | 21    | The stack size in dwords — block 1's length over four                             |
| `0xa0` | 40    | The deadline a `WAIT`, `WAITABS`, `WAITANIM` or `WAITANIM_CH` is sitting on, a clock reading; nought for none |
| `0xa4` | 41    | The deadline the last trigger armed, a clock reading, which `WAIT4ANIM` waits on. It is not cleared when it passes, only by a passed `WAIT4ANIM`, a `WAITANIM`'s first visit (or a `WAITANIM_CH`'s), a `LOOPANIM` that starts a loop or a `LOOPANIM_CH`, so a save can hold one long past; nought for none |
| `0xa8` | 42    | The looping key: `(entry << 16) + role` of the last `LOOPANIM`, or `0xffff` from creation, a `TRIGANIM`, `TRIGANIMSPEED`, `TRIGWAITANIM` or `WAITANIM`; the `_CH` forms leave it alone |
| `0xbc` | 47    | `TRIGWAITANIM`'s mark (shared with `TRIGWAITANIM_CH`): the role it is waiting for, plus one; nought when not waiting |
| `0xc4` | 49    | The deadline the last `SETTIMER` set, a clock reading, not scaled by the speed word; nought until one runs |
| `0xe0` | 56    | A coaster script's ride handle, which its `COAST 8` stores; nought before that   |
| `0xe4` | 57 (low word) | A 16-bit play rate in thousandths: 1000, or a `TRIGANIMSPEED`'s rate. The word above it, `0xe6`, is a separate field; it reads `0xffff` in every script of the shipped park |

**How those were pinned rather than guessed.** Dword 20 equals the length of the body block that
follows it for all fourteen scripts, which fixes the struct's alignment and with it every offset in the
table; every one of the fourteen counters then lands on an exact instruction boundary, and on a
`BRANCH`, `BRANCH_Z`, `TEST` or `WAIT` — what a settled script waits on. Block 2 is the variable array
because slot 2 is `VAR_CAPACITY` and slot 3 `VAR_DURATION`, and both agree with the **object records**
in the World module, a wholly separate part of the file: 5 and 30 for the ride, and 1, 1, 1 and 3 for
the three toilets and the sideshow.

**The stack and its two indices.** Block 1 is 12 bytes for the one script that declares a stack, the
Belly Bounce's `Bouncy.RSE` (`#setstack 3`), and empty for the other thirteen, and dword 21 is its
length over four in all fourteen. `JSR` fills the stack down from its top slot and `HUSH` fills it up
from slot 0 (the [Instruction Set](/vm/instructions/)). The Belly Bounce was saved with dword 16 at 2,
the top slot, so nothing was pushed, and dword 17 at 0; the thirteen others read -1 and 0. **Only the
slots above the call index are live.** The Belly Bounce's block reads `CD CD CD CD` twice, slots never
written, then `18 00 00 20`: the return address of a call that had already returned, still there. Block
3 was compared byte for byte with each script's `.RSE` file. Dword 18 reads 3033 for the fountain, 1 for
the gate and 0 for the other twelve.

**The animation fields.** The game reads the whole struct back, so these come back as they were saved. The
shipped park holds `0xffff` at `0xa8` for ten of the fourteen, 5 for the fountain, the drinks kiosk and the traffic
lights, and 2 for the ride; `0xbc` and `0xc4` are nought for all fourteen and `0xe4` 1000. The deadlines are readings of the game
clock, which the save keeps in the clock module (`KOLC`) as the first of its two dwords, `0x06D13894` here, and which
the game puts back on loading, so a deadline less that reading is the time still to wait. `0xa4` is set only for the
sideshow, 180,933 ms in the past; `0xa0` for the two security cameras, 2,341 and 2,329 ms ahead, and for the ride, 63 ms
ahead. A played jungle park holds `0xc4` set in two scripts, 25,198 and 2,745 ms past, and in its autosave one of them
284 ms ahead.

**The head table** is the script's `ADDHEAD` slots (the [Instruction Set](/vm/instructions/)): one dword per
slot, the visitor whose head hangs on head node *slot + 1*, nought for a free slot. The game takes the slot count
from the block's length over four. A script gets one slot per head node of its thing's model, counted from id 1 in
the model's head space until an id is missing, so every script with a thing and a model carries the block, used or
not. The shipped park's fourteen are all empty. A played jungle park carries seven, each as long as its model's head
count: the Tom Tom Twister 40, Mumbo 5, the Sun God 32, the Crazy Ape 16, Rocky Racers 8, the Aztec Mayhem 27 and the
ferry 1. Five of the seven hold riders (the Crazy Ape six of its sixteen, and seven in the
autosave), at slots scattered as a random draw leaves them: the Twister's riders sit on slots 4, 24, 25 and 28.

**The walk slots** are the script's walk-on places (the `.RSE` header's walk count; the
[Instruction Set](/vm/instructions/)'s `WALKON` and `WALKOFF`), copied raw, 32 bytes each:

| Offset | Size | Description |
| ------ | ---- | ----------- |
| `0x00` | 2 | Walk node id |
| `0x02` | 2 | Head node id |
| `0x04` | 2 | Walk-off node id, from |
| `0x06` | 2 | Walk-off node id, to |
| `0x08` | 4 | Start, a reading of the game clock (`KOLC`) |
| `0x0c` | 4 | Due, the same clock |
| `0x10` | 4 | The walker's person handle |
| `0x14` | 2 | Facing, an octant 0-7 |
| `0x16` | 2 | Action |
| `0x18` | 2 | State: 0 free, 1 walking on, 2 on the ride, 3 walking off, 4 off |
| `0x1a` | 2 | Flags |
| `0x1c` | 4 | Not decoded |

A slot let go keeps everything but its state and handle, so due less start is the last leg walked, the walk off's (how
long one lasts: OpenTPW's `docs/exe/ride-operation.md`, "How long a leg lasts, and where its ends are"). The shipped
park's sideshow saves its three slots all nought. In Alexah's Lost Kingdom autosave the Jungle Spray's read 700 and
1,100, the Steak Shop's 600, the Inca God's 800; a slot on the ride saves its start at or just past its due,
restamped on arrival.

> The struct's speed word at `0xc0` (dword 48) reads 50 for thirteen of the fourteen and **60** for
> the one ride. That is not a misalignment: 50 is what the *loader* writes, and the game pushes an
> object's own operating speed over it — that ride's `mOperatingSpeed` is 60 in the same file.

## The ride system module (`SYSR`)

What every thing's **model** was doing when the park was saved. It is the other half of a park that
loads without rebuilding itself: the script module above stops the construction clip being replayed,
and this one stops everything standing frozen once it is.

| Size    | Description                                                      |
| ------- | ---------------------------------------------------------------- |
| 4 bytes | Module length in bytes, counting what follows this dword - **17,522** in the shipped park, which ends exactly on the `SYSR` tag |
| 4 bytes | Present record count — **161** in the shipped park                |
| 4 bytes | Free record count                                                |
| 4 bytes | High-water mark                                                  |
| n       | One record per slot                                              |

A slot that holds **nothing costs exactly one byte** — the tag alone — which is how a module with far
more slots than things stays small. A present record is:

| Offset   | Size     | Description                                                     |
| -------- | -------- | --------------------------------------------------------------- |
| `0x00`   | 1 byte   | Tag — `01` for present                                          |
| `0x01`   | 4 bytes  | The **item** id, not a thing id                                 |
| `0x1d`   | 4 bytes  | Packed model flags; low seven bits are the hoarding state below |
| `0x23`   | 4 bytes  | Hoarding progress, little-endian float (`0` retracted, `1` closing endpoint) |
| `0x2b`   | 2 bytes  | Node flag-word count                                            |
| `0x2d`   | 2 bytes  | The model's node-lookup record count                            |
| `0x2f`   | 4 bytes  | The lookup records' shared flags: `0x1` some record has a position, `0x4` something is attached, `0x8` some record's file flags carry `0x2` (walkable meshes); `0x2` is set by the routine that hides a model's head nodes. The Jungle Spray's reads `0x1`, a ridden Aztec Mayhem's `0x7` |
| `0x33`   | 4 bytes  | How many things are attached                                    |
| `0x37`   | n×8      | Per lookup record: its runtime flags, and the handle of what is attached to it (-1 for nothing; a record without a position starts at nought) |
| —        | n×4      | One flag word per node; bit `0x10` is hidden                    |
| —        | n×44     | The animation channels — see below                              |

Everything from `0x01` on is **unaligned**, because the one-byte tag leads.

**Hoarding flags:** `0x01` active, `0x02` raising, `0x04` lowering; `0x08` selects Closed,
`0x10` Hoarding (broken), `0x20` Condemn and `0x40` Upgrade. Bits above `0x40` belong to other
model state. The restore routine (`0x004647a0`) expands these bits to runtime model flags
`0x20` through `0x800`, and restores progress through `0x004547f0`. Progress alone does not say
whether the panels are active or moving. The closing endpoint does not guarantee equal full height
for every panel: the executable staggers their deformation.

The saved placed object's model handle selects **slot index plus one**, not the item id; repeated
instances of an item can have different hoarding states. Q91b checks this pairing for all eleven
placed objects in the shipped Jungle park. Its synthetic record checks the unaligned flag and
progress reads independently of normal zero-progress saves.


**A lookup record's runtime flags** say whether the game keeps its node's position
([Models](/formats/models/#which-records-have-a-position)): `0x1` it has one, `0x20` its node has no children, `0x8`
it is posed all the same, `0x2` something is attached to it, `0x4` the ride view is on it, `0x10` its node carries
`0x400`. The shipped park's Jungle Spray saves `0x29` on eleven records and `0x21` on its `camera`, which is posed only
while the ride view is on it; a played park's Aztec Mayhem saves `0x29` on 34 of its 39 records and `0x2b`, with a
handle, on the five heads carrying a rider: all 39 carry `0x8`.

**A channel is 11 dwords**, in the order the game copies them back onto the running channel:

| Dword | Description                                                                        |
| ----- | ----------------------------------------------------------------------------------- |
| 0     | Flags — `0x1` loop, `0x4` hold the last frame rather than count as busy             |
| 1     | The animation role, or **12** for "running nothing"                                 |
| 2     | Which clip of that role                                                             |
| 3     | Start stamp — a reading of the saved clock (`KOLC`): where the clip began           |
| 4     | Clip time — the same clock's reading at the channel's last advance, or for a held channel its start plus a whole clip |
| 5     | A third stamp on the same clock, which the shipped park saves equal to the clip time or the clock itself |
| 6     | Speed, a float                                                                      |
| 7     | The role queued to play next, or 12 for none                                        |
| 8     | Its clip                                                                            |
| 9     | The flags it was queued with — a caller's flags, and left behind when the queue empties |
| 10    | Its speed, a float                                                                  |

Dword 4 less dword 3 is how far into its clip the channel was: 1,376 ms for the Belly Bounce's saved loop, played at
1.1. In the shipped park no channel has a clip queued, though three running channels keep the loop flag of a queue that
has emptied; a played park saves some with a clip queued behind the running one.

> **The flag word is the engine's own field, not the one a caller passes when starting a clip**, and
> the two disagree where it matters. A caller's flags are `0x1` loop, `0x2` start at once, `0x4` do not
> lay the rest pose down, `0x8` do not apply the hide list. Stored on the channel, `0x1` and `0x8` mean
> the same, but `0x2` means **frozen at frame nought** and `0x4` means **held on the last frame** —
> states the engine has recorded, not requests. On loading, the game copies the word back as it stands
> and adds `0x10` to a held channel (OpenTPW's `docs/exe/ride-operation.md`). Feeding the stored word back
> in as caller flags therefore drops the held pose — in the shipped park that is **eleven of the fifteen**
> channels that hold a real role.

> **The module does not say how many channels a thing has**, and the walk cannot step over a record
> without knowing. The count is the item's own `NumSimultAnims` — the Jungle Spray runs three lanes and
> everything else one. This is not a detail: walked with one channel for everything, the cursor lands
> **5,786 bytes short** of the module's end; with the real counts it lands **exactly** on it across all
> 161 records, which is what makes the layout above trustworthy.

In the shipped park **15 of 163 channels hold a real role** and the other 148 hold the sentinel — most
records are scenery with nothing to animate. The fourteen placed things are saved as: Gates role 5
entry 1, Traffic Lights role 5 (looping), Belly Bounce role 2 (looping), Jungle Spray role 2 on all
three lanes, Coconut Kiosk role 5 (looping), Litter Bin role 0, both Security Cameras role 6, Staff
Room nothing, the three Small Toilets role 5, Fountain role 5 (looping) and the Bus role 5 entry 2.

> Because a record names an **item**, three Small Toilets are three records that read alike, and the
> module carries nothing that tells them apart. In this file the records of placed things happen to run
> in ascending script-handle order, which pairs them off — but that is an ordering that matches, not a
> decoded thing handle, and within one item id the choice is unobservable because those records are
> identical.

Everything in this section is measured from `Easymode.TPWI`, the one file of this shape that ships, so
treat it as what that file proves and no more.

## The track-rides module (`KART`)

Every ride whose item has a `Bumper.BumperType` - the karts, the water rides and the bumper arenas -
and the track the player has laid for it. It begins right after the `SYSR` tag and ends at the `KART`
tag, and it is a tree of chunks, each with a 12-byte header:

| Size    | Description                                                            |
| ------- | ---------------------------------------------------------------------- |
| 4 bytes | Chunk type                                                             |
| 4 bytes | The chunk's own size, the header included and its children not        |
| 4 bytes | The whole chunk's size, its children included                          |

Its children follow the chunk's own data. The module is one type-1 chunk of own size 12 whose whole size
is the module's length, and every other chunk is its child:

| Type | Size | Description                                                                         |
| ---- | ---- | ----------------------------------------------------------------------------------- |
| 2    | 32   | A stamp: the time and date of the build that wrote it                               |
| 3    | 52   | One ride: after the header its handle, x, y, orientation flags and item id, then five more dwords |
| 4    | 28   | One track section: after the header the ride's handle, the section type, x and y   |
| 5    | 208  | One car                                                                             |
| 9    | 24   | A record belonging to a car                                                         |
| 6    | 16   | Closes a ride                                                                       |

**A handle is the ride's slot in its low byte and the item's `BumperType` above it**, the same number the
object record's `mTrackRideHandle` holds: `0xfffffc00` for a Dino Karts (`BumperType` -4) in slot 0.
**Positions** are map cells times `0xc00`. **A section's type** is its low 16 bits - 5 to 8 a bend,
9 and 10 a straight, 11 a crossing, 12 a straight with an add-on, by the collision the game builds for
each - and bit 16 is set on the first two straights from the station. The sections are stored in
circuit order from the station, mostly two cells (`0x1800`) apart.

The shipped park's module is 44 bytes, the root and the stamp only (inflated 1,595,038 to the tag at
1,595,082). In Alexah's played jungle park it is 1,964 bytes: one Dino Karts, its 33 sections
(`9 9 9 12 12 9 5 10 10 8 11 9 7 10 6 9 5 10 10 8 9 12 9 9 7 10 6 8 5 9 7 10 6`), four cars and a close.
The played fantasy park holds one bumper arena with no sections. Walked as a tree, the module lands
exactly on its tag in all nine park files read (the shipped park and eight played ones).

## The coasters module (`SAOC`)

It begins right after the `EMAK` tag and ends at the `SAOC` tag, and opens with four dwords: the number
of coasters, then three counts of the game's coaster handles. A park with no coaster reads `0 0 0 1`,
which the shipped park does (inflated 1,606,446); Alexah's played jungle park, with one Temple Of Gloom,
reads `1 1 0 2`. Each coaster then opens with a 32-byte header, read from the game's own saver and loader and
measured in that park:

| Offset | Size    | Description                                                                              | Temple Of Gloom |
| ------ | ------- | ---------------------------------------------------------------------------------------- | --------------- |
| `0x00` | 4 bytes | Flags: bit 0 the circuit is closed, bit 1 a gap is open in it; bits 2 to 8 are the coaster's own flags | `0x101` |
| `0x04` | 4 bytes | Not described                                                                            |                 |
| `0x08` | 2 bytes | Its model, the coaster's type                                                          | 19              |
| `0x0a` | 2 bytes | Its model instance - the number its object record's `MeshInstanceID` holds, and how the game finds the coaster of an object | 330 |
| `0x0c` | 2 bytes | Its handle - the number its script's `+0xe0` holds in the ride scripts module             | 1               |
| `0x0e` | 8 bytes | Not described                                                                            |                 |
| `0x16` | 2 bytes | How many 34-byte (`0x22`) piece records follow the header                                 | 38              |
| `0x18` | 4 bytes | Not described                                                                            |                 |
| `0x1c` | 4 bytes | How many pairs of its sections intersect; the game offers no coaster with any            | 0               |

The pieces follow, and then the track and the trains, which are not described here: 510 bytes of them after
that coaster's pieces, so only the first coaster's header sits at a known place. A guest is offered a coaster
only with bit 0 set, bit 1 clear and no clash. The module holds no rating of the ride: the game works the ride's
excitement out again after a load.

## The cheats module (`STHC`)

Two bytes, between the sound's `NUOS` tag and the cheats' own `STHC`.

| Offset | Size   | Description | In `Easymode.TPWI` |
| ------ | ------ | ----------- | ------------------ |
| `0x00` | 1 byte | The cheats flag: 1 lets the park's cheat keys work | 1 |
| `0x01` | 1 byte | Not described: nothing but the reader, the writer and the constructor touches it | 1 |

The eight played park files carry `00 00`, the five Full Simulation saves among them. So a park loaded from
`Easymode.TPWI` starts with the cheat keys on.
