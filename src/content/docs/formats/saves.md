---
title: Saves (*.tpws, *.ints, *.lays)
---

Theme Park World stores saves with three different extensions.

These extensions are:

- **TPWS**: Standard save
- **INTS**: The park as the theme was first entered, `restart.INTS`, which the park menu's Restart Park loads; in either
  game mode, the same layout as a `.TPWS` (version `500`). The game writes it as a park is entered from the lobby
  when the player's theme folder has none, and never again while one is there (OpenTPW's `docs/exe/saves.md`,
  "Restart Park: restart.INTS"). In the reference player's folder, Lost Kingdom entered on its `easymode.TPWI`
  wrote one at `mGameTick` 755 with the file's balance, 87,987, and its 13 guests. `Refresh.INTS` is the same,
  written around a sound-quality change.
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

**The game does not need that stream back.** A file whose payload was deflated again by another zlib (.NET's
`ZLibStream`, 39,239 bytes of stream where the game's own was 37,210), behind a header giving the new block size
and the same four dwords, was listed and loaded by the game under Proton, the park running on from the saved tick
(2026-10-08). The loader inflates what is there to the length the header gives.

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

### The interface block: the aerial's tags

After the last tag comes an untagged block the loader logs as `UI`: the tags standing on the gadget's aerial,
the small pictures a said advisor line leaves there. It is a count and 540 bytes (`0x21c`) a tag. The count is
nought, so the block is four bytes, in the shipped park (where it ends the stream on 1,608,309) and in six
other files of the nine first read; it is 1 in the two saves of Alexah's played jungle park and in three files
the original wrote under Proton with a tag standing.

The writer (`FUN_0048ee50`) counts the records in use among the ten of a table at `0x7c26b0`, then writes one
record for each in use, in the table's order, from the properties of its control (the getter `FUN_0065e705`):

```text
u32        count                                0 to 10
per tag (0x21c bytes):
  0x000  u32        id        the tag's row in the executable's tag table (property 1)
  0x004  u16[256]   text      UTF-16, terminated; at most 255 characters
  0x204  i32        life      milliseconds, the WHOLE life and not what was left of it (property 7)
  0x208  u16        picture   the part of msgtag.md2 the tag shows (property 4), then two bytes of padding
  0x20c  u32        button 1  the message bar's first button type (property 5)
  0x210  u32        button 2  its second (property 6)
  0x214  u16        thing     the thing the line was about, nought for none (property 3), then two bytes of padding
  0x218  i32        item      that thing's item id, nought for none (property 8)
```

**The record is built on the writer's stack and only these fields are stored**, so the text's bytes after its
terminator and the two paddings are whatever stood there: in every record read the paddings are `6f 00` and
`41 00`, and between 195 and 231 of the bytes after the terminator are not nought. The loader reads none of
them.

The loader (`FUN_0048ef70`) takes every standing tag off, then hands each record to the tag list as a new tag
(`FUN_0048eb20`) with the record's id, text, life, picture, buttons, thing and item. The list starts the tag's
timers at the life, so **a loaded tag stands for its whole life again**.

The five records read:

| File | id | life | picture | buttons | thing | item | text |
|---|---|---|---|---|---|---|---|
| Alexah's jungle `autosave.TPWS` and `New Save.TPWS` | 94 | 30000 | 2 | 24, 0 | 0 | 0 | "Interestingly, some visitors believe your ticket price is cheap. Is there room for you to raise it?" |
| a park saved just after a sale (Proton) | 148 | 30000 | 1 | 4, 0 | 46 | 2984 | "A prankster is littering your park. Get your janitors to the scene to clear up the mess." |
| a park saved with a broken Hot Pot, and again after its repair (Proton) | 72 | 30000 | 1 | 3, 0 | 43 | 1140 | "Your The Hot Pot ride has broken down, and a mechanic is on his way to fix it." |

Each agrees with its id's row of the tag table in the life, the picture and the buttons, and item 1140 is The
Hot Pot. What item 2984 is, on a guest's tag, is not decoded.

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
| 13 | 4 bytes | `mRandomSeed` | The park's one random generator's state as it stood at the save - `0x3f416bba` in the shipped park. A load hands it to the generator as it is: the original's first read after loading a file is the file's dword |
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
the arrival manager makes for a guest waiting at the stop to go home when no vehicle is current, picks one of the
three at random: one step of the generator in `mRandomSeed`, then `% 3` unsigned, nought the bus, and a
kind whose feature cannot be found falls back to the others; both cache by kind as well. However it was picked,
the engine creates the vehicle thing the first time that kind is sent for, then caches its id in that kind's
slot. `mCurrentArrivalVehicle` holds whichever is on its way, or nought when none is, and goes with that
vehicle's saved script: the game asks the named thing's script how far round it has got. A park written with the
bus driving in, its script as it ran and this handle 15, loaded with the bus current and driving on; written
with the handle nought and a guest waiting at the stop to go home, it loaded with another vehicle sent for at
once (measured under Proton, 2026-10-08 and 2026-10-09).

**A vehicle made is an ordinary object on the corner cell.** In three files the original wrote after a ferry
or a seaplane was first sent for, its record reads `mX` and `mY` `0x0080` (the middle of cell (0,0)), angle
nought, its item (1602, 1604), the date it was sent for, flags nought, `mState` 3, `mTopLeft`, `mEntryPos` and
`mExitPos` all 1, `mCanLoad` 1, as the bus's does. Its name holds one line ("Seaplane", "Ferry"), and the words
between its characters and after its end are whatever the stack held, where the bus's are `0xcdcd`. It heads
the object list, and cell (0,0)'s chain: `mWho` there is the newest vehicle and the chain runs newest first
(ferry 32, seaplane 44, bus 15, then 12, 11 and down to 1 in one file). Its model's record is a fixed item's
("The ride system module"). The original loads such a file written by another program with the vehicle current
on its thing and makes none again; a file whose slot is nought has it make the vehicle afresh the first time
that kind is sent for (measured under Proton, 2026-10-13).

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
| `0x14` | i32 | The upgrade tier researched, 0 to 2: 2 on 17 of the 26 researched records, 0 on every other record. In a new Full Simulation park's `restart.INTS` (70 records) it is 2 on 13 of the 28 researched ones, those whose item has no upgrade to research, and 0 on the rest |
| `0x18` | i32 | How many of the item stand in the park. Lost Kingdom's save: 3 for the Small Toilet, 2 for the Security Camera, 1 for each of nine more kinds, 0 for the rest, 14 in all, its fourteen objects |
| `0x1C` | u32 | `mGameTick` when the first of the item was built, kept after the last is sold: 15 on nine records in Lost Kingdom's save, the Balloon Shop among them with a count of 0, and 604 for the bus. The game writes it only while it reads 0, so the gates and the traffic lights, made on tick 0, keep 0 |

Across the six park files read (Lost Kingdom's and five played saves of three themes) every record with a
count above 0 has a stamp, but for the gates and the lights. The three bytes after `0x10` are nought here. **`0x08` (u32) is 5 on a ride (but the three track rides, items 1500 to 1502), a shop or a sideshow and nought on a feature, a vehicle, a track ride and the other items**: on 23 of `Easymode.TPWI`'s 50 records (its 12 rides, 6 shops, 4 sideshows and item 100, a ride of the rides folder that is not a buy-list item), on 29 to 32 of the 66 to 70 of Alexah's played Full Simulation saves and on 31 of the 70 of every fresh park's `restart.INTS` (17 of its 20 rides, which leave out the three track rides, items 1500 to 1502, the 8 shops, the 5 sideshows and item 100). What makes it, and what reads it, is not decoded; a file with nought in all of them is loaded by the game.
**Buying one adds one to the count and stamps a record that reads 0; selling one takes one off and leaves the
stamp**: a park the game saved after a Crazy Ape was bought on tick 924 holds count 1 and stamp 924 on item 1101,
and saved again after the ape was sold, count 0 and stamp 924 (measured under Proton, 2026-10-08). The game fills a
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
candidates all joined on tick 0. A candidate whose time runs out has `mValid` cleared and nothing else
(`0x005084f0`), which is the first kind of empty slot, and a newcomer takes the lowest slot whose `mValid` is 0.

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
| `0x001A4F` | 4 | `mMonthAtLastUpdate`, nought-based | 1 (February) |
| `0x001A53` | 4 | `mDayAtLastUpdate`, the day of the month | 2 |
| `0x001A57` | 4 | `mFunnySecsPerRealSec` | 15000 |

**The month and the day are the date of the file's own `mGameTick`** (Lost Kingdom's 755 is 2 February 2000): the
park's date is `mFunnyTimeStart` plus `mGameTick` × `mFunnySecsPerRealSec` / 4 seconds, and the game stores the date
it last compared every time the tick moves. Measured in the 48 park files in the game's folders on this
project's machine (its save folder, a player's Full Simulation saves and the files kept from runs of the original):
41 hold their tick's date (22 at the shipped tick 755 with its pair, ten of them byte copies of the disc's
`Easymode.TPWI` and ten of one `restart.INTS`, and 19 at other ticks); the four `restart.INTS` written at tick 0 hold
month 0 and day -1 (`0xFFFFFFFF`), what the clock holds before its first update; the other three are files another
program wrote, put there to be loaded. A file whose day differs from its tick's makes the game send the day's change on
its first tick after the load, and one whose month differs the month's. The year is not saved. `mFunnyTimeStart` is
2000-01-01 and the rate 15000 in all 741 park files measured; `mSessionStart` is carried from the file a park was
loaded from (1999-10-21 20:54:53.29 in Lost Kingdom's), and a park made afresh takes the computer's local time as
the clock is made (2026-09-22 17:51:03.208 in one `restart.INTS`). OpenTPW's `docs/exe/saves.md`, "OpenTPW's writer,
the calendar's fields", has what the game does with them.

The five pairs are the park's staff as the pool last counted them, one pair a kind in the order of `mType` (handyman,
mechanic, entertainer, guard, researcher): `mPeopleInCat` is how many of the kind the park employed, and
`mStopProducing` is 1 where that had reached the kind's `StaffPoolInfo.Max<Kind>InPark`. The game counts them only as
it tops the pool up (`0x00508000`), so they are a reading taken at the pool's `mTimeSig`, not at the save: Lost
Kingdom's 1 in each is its five staff, one of a kind. An OpenTPW-written file with these pairs, the pool's records and
its `mTimeSig` taken from a running park was read by the game, which topped the pool up 361 counts after that
`mTimeSig` (OpenTPW's `docs/exe/saves.md`, "OpenTPW's writer, the staff pool and the arrival timer").

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

**The game's writer gives every cell its map and its track sub-record**: its test for a default one answers
"not default" for both, always, and only the effects sub-record is left out when its ten bytes are a default's.
So a file the game wrote opens every cell with `3` or `7`, 84 or 94 bytes, and the effects sub-record is where
the block's variable length comes from: `3` on 16,134 cells of the shipped park and `7` on the other 250;
15,132 and 1,252 in a played Full Simulation park. The reader takes each sub-record its bit names, so a walk
that sums the bits reads the six combinations the game never writes. The shipped park's cells total 1,378,756 bytes,
exactly the region from `0x1A6D` to `0x152431`, and an implementation that only needs what is *in* the park
can measure each cell and skip it.

> **Check the stride cell by cell, not just in total.** Measured this way, all 16,384 cells land on the
> next cell's status byte every single time. That is a sharper check than the block's own `DLRW` trailer,
> which a pair of compensating errors would still reach.

The map sub-record is a **29-byte tile base** followed by a **23-byte litter block**. The track
sub-record repeats the same tile base field for field and adds `mSegmentNumber`, two bytes, for its 31 -
`0xffff` on every cell of the shipped park.

**The effects sub-record is five signed 16-bit words**: happiness, illness, hunger, security and attraction,
the balance file's `RegionFX` keys in its own order. A cell's words are the sum of what the things round it
add. Each source adds one of the eight `RegionFX` effects of `data/levels/Standard.sam` over the square of that
effect's `Radius` round its own cell, held to the map, each word divided by `|dx| + |dy| + 1` and cut towards
nought:

| Source | Effect | Where |
| --- | --- | --- |
| An object with `0x1` in `mFlags` (a toilet) | 1, or 6 while its State of repair, cut to a byte, is under 25 | the object's cell |
| An object with `0x10` (it provides security) | 4 | the object's cell |
| An object with `0x80` (fireworks) | 7 | the object's cell |
| An entertainer (model 6) | 0 | the cell in their `mLastRecordedMapId`, the person record's `+41` |
| A guard (model 7) | 3 | the same field of theirs; a field of nought adds nothing |
| A cell whose `mLitter` is 7 | 2 | that cell |
| A cell whose `mLitter` is 8 | 5 | that cell |

So a camera's own cell reads security 20, the four beside it 10, and its eleven-by-eleven square's corners 1.
**Measured on every cell of thirteen park files the game wrote**, 212,992 cells: the ten bytes are exactly this
sum, and a cell has the sub-record exactly where one of its words is not nought. No file among them holds
fireworks or litter of kind 8, so those two rows are the executable's alone. A file whose cells do not hold an
object's effect is still loaded; the game takes the effect off when the object is sold all the same, and the
cells round it then read under nought.

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

**`mMeshInstance` at 3 is a queue cell's model.** It is a slot of the ride system module (`SYSR`, below) plus
one, kept as saved and used after a load: non-zero on exactly the queue cells (`mType` 3), 4 of 4 in the shipped park and 78 of 78 in
a played one, and nought on every other cell of both. `mHoardingNeighbours` at 28 is nought on every cell of
both.

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

`mFlags` bit `0x80` **is the cell's attribute `0x10`** (`mStatusFlags`, and `base.map`), carried over as the map
loads: the two agree on every one of the shipped park's 16,384 cells, 552 with both and none with one. Those 552 are
the approach before the gates, rows 0 to 16 of columns 28 to 67, `mType` 7 (490) and 30 (62), and every one carries
`0x40` as well. The game draws the ground of a cell with `0x40` and not `0x80` at half light, so this bit keeps the
approach lit, though no ground of it is drawn in the shipped park (the scenery covers it); OpenTPW's
`docs/exe/park-engine.md`, "The park view's colours", has what the executable does with both.

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

Type 2 is the 240 river cells, every one a heightfield hole (`0x1`), as are 853 of the type-7 cells and all 66 of type 30 (jungle only; fork watch 2026-10-25). Types 7 and 0 are still recorded as observed rather than named.

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

**The ground tile under a footprint is index 8 of set 0, turned 0, and bare ground's is index 55**: all 1,374
footprint cells and all 47,861 bare cells of seven park files. The game writes the first as it
builds a thing and the second as it sells one: the sixteen cells of a Crazy Ape went from `(0, 55, 0)` to
`(0, 8, 0)` when it was bought and back when it was sold, and the queue cell the game had laid before its
entrance, over a path, was left bare ground too (measured under Proton, 2026-10-08).

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
at 42 and is an exact permutation of 1 to 42. That holds in all ten park files: the highest id is the number
of things (526 of 526 in a played park), so the game gives a freed id out again. The loader hangs each thing on
the node its id names and then rebuilds its free list from every id the file did not use.

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

**A person stands in the chain of their own cell.** `mMapChild` and `mMapParent` with the cell's `mWho` are one
chain a cell: `mWho` names its head, each thing's `mMapChild` the next and `mMapParent` the one before. Every thing
of all ten park files is in exactly one chain, and every guest and member of staff in the chain of the cell their
`mX` and `mY` name, ahead of any catalogue object on it: 18 of 18, 65 of 65 and 392 of 392, riders and resting
staff with no sprite among them. The loader takes the chain as the file has it and enters nobody in a cell of its
own accord.

**The five staff lists run in the list's order.** The header's `mFirstHandyman`, `mFirstMechanic`,
`mFirstEntertainer`, `mFirstGuard` and `mFirstResearcher` each name the first thing of that model in the thing
list, and each member's `mNext` (the record's last two bytes) the next of the model after it, `0` on the last: so
in all ten files.

After that the models diverge completely and the sizes are wildly uneven, so the stream cannot be strided
- each model has to be recognised, and each has one fixed size: **1** a guest (533 bytes), **3** a
placeable catalogue object (1,099), **4 to 8** the five kinds of staff (511, 513, 509, 511 and 509), **9**
the strike system (103), **10** a bare map object (18), and **11 to 19** the singleton managers, from 16
to 73,544 (11: 5,846; 12: 16; 13: 73,544; 14: 4,190; 15: 99; 16: 300; 17: 16; 19: 937). The header's handles
name the managers, and name the same things in all nine park files: `mStaffHQ` thing 1 (model 9),
`mMechanicHQ` 2 (model 10), `mTagSystem` 4 (12), `mParkAnalyser` 5 (13), `mResearchLab` 6 (14), `mWeather`
7 (15), `mBankAccount` 8 (16) and `mUIMsgReceiver` 9 (17); things 3 (model 11) and 10 (model 19) are named
by no header field, though 3 is the advisor and 10 the challenge manager (both below). There is no model 18 in a park file: the writer refuses it, "Should not be able to save
online persons". **Model 10 is the mechanics' HQ**, eighteen bytes: the sixteen every thing begins with and
then `mNextObject`, two bytes, the id of the object the mechanics' search for a ride to fix last began on, or
`0`. Each search moves it one object down the object list, so it is `18` in the shipped Lost Kingdom park and
`17` in a park saved by the original a little later; a search that moves it off the list's end leaves `0`. With those sizes, the walk closes exactly on `DLRW` in all nine park files.

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
| 8 | 2 bytes | `mX` | from the shared map base: the anchor cell in the high byte, and `0x80`, the middle of the cell, in the low byte of every placed object in seven park files (349 of 349) |
| 10 | 2 bytes | `mY` | |
| 16 | 4 bytes | `mAngle` | degrees - `0`, `90` or `270` in the shipped park |
| 20 | 2 bytes | `mId` | the item's `Info.Id`, from its own `.sam` |
| 22 | 32 bytes | eight `tv[`*t*`]` dwords | when the object was built: year, month, day, day of the week, hour, minute, second, millisecond - not `SYSTEMTIME`'s order, which puts the day of the week third. It is a date on the **park's own calendar**, not the real one; the game writes it with `FileTimeToSystemTime` and reads it back with `SystemTimeToFileTime`, ignoring the day of the week; a stamp that will not convert loads as nought. The shipped park: the Belly Bounce and ten more read 2000-01-01 15:37:30, the bus 2000-01-27 05:10:00, the gates and the lights 2000-01-01 00:00:00 |
| 54 | 4 bytes | `MeshInstanceID` | the object's model instance; a coaster's record in the coasters module carries the same number (Temple Of Gloom in Alexah's played jungle park: 330) |
| 58 | 2 bytes | `mFlags` | see below |
| 60 | 132 bytes | 33 pairs of `mNameA[`*i*`]`, `mNameB[`*i*`]` | 2 bytes each: the two lines of the object's name, a character of each in turn - see below |
| 192 | 4 bytes | `mRideScriptHandle` | the id of the object's running script in the ride scripts module |
| 196 | 4 bytes | `mTrackRideHandle` | nought, or for an item whose `Bumper.BumperType` is set, its slot in the track-rides module in the low byte and the `BumperType` above: `0xfffffc00` for a Dino Karts in slot 0. In all nine park files (the shipped park and eight played ones), non-zero on exactly the objects whose item has a `BumperType` |
| 200 | 4 bytes | `mState` | nought on every object a guest may be offered (`mFlags` `0x4`) and 3 on every other, in seven park files (349 of 349); the game sets it so as it builds the object. **1 on a ride broken down**, with `mCanLoad` nought: the original's own file of a broken Hot Pot, and nought again with `mCanLoad` 1 in its file of the same pot mended. 2 (shut for an upgrade) and 4 (condemned) are in no file to hand |
| 204 | 2 bytes | `mTopLeft` | a packed cell: the anchor cell (`mX`, `mY`) moved by the item's `Info.MapOffsetX` and `Info.MapOffsetY`, the pair turned by the object's angle and a half turn more (see below). Two items of Lost Kingdom's declare an offset, the Huge Hollow Rock (item 1427) and the Dino Karts Tunnel (item 1501), each one row; on every other object it is the anchor cell. The object's model record carries this cell |
| 206 | 2 bytes | `mEntryPos` | the cell a visitor is sent to |
| 208 | 2 bytes | `mNext` | this object's link in the object list |
| 210 | 2 bytes | `mAssignedStaffMember` | a handle: the member of staff sent to service it, `0` for none |
| 212 | 2 bytes | `mBackOfQueue` | |
| 214 | 4 bytes | `mCanLoad` | non-zero when the object will take anyone aboard |
| 218 | 2 bytes | `mExitPos` | packed like `mEntryPos` - the cell a guest is put down on when they leave |
| 220 | 2 bytes | `mFirstInQ` | |
| 222 | 4 bytes | `mIsTrackRideValid` | 1 on every object in the shipped park; 0 on Alexah's saved Temple Of Gloom, the one object in nine park files that reads 0 |
| 226 | 2 bytes | `mUpgradeParent` | a track add-on's ride, by its thing id: the ride it was put down on (the original's builder's object at the placement); nought on every other object. Alexah's played Full Simulation Dino Karts Jump and Tunnel both read 289, the Dino Karts' id, and the original wrote 289 on a Tunnel and a Jump bought for it there (Q780) |

After that come several ring buffers - each a `mCurrentEntry`, an `mNumEntries`, an `mWrappedAround` flag,
an `mTemp` and then `mNumEntries` entries of `mData[`*i*`]` - interleaved with `mNumCustomers` and
`mNumWalkAways`, and then a long tail of shop and ride fields: `mOperatingCapacity`,
`mOperatingDuration`, `mOperatingSpeed`, `mPersonBeingLoaded`, `mCostOfGoods`, `mQualityOfGoods`,
`mChanceOfWinning`, `mPricePerUse` (clamped to 0-500 as it is read), `mAmountOfSpecialIngredient`,
`mQueueSizeInCells`, three floats (below), `mRequestedService`, `mTimeMarkedForMaintenance`, `mTotalCosts`,
`mTotalTakings`, `mUpgradeBalloonSprite` and `mUpgradeLevel`.

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

**Three floats follow `mQueueSizeInCells`**, at 1066, 1070 and 1074. The game starts all three at 100.0 as it
builds the object. The second and third are the remaining life and the state of repair (a played Crazy Ape reads
70.0 and 29.0); the first reads 100.0 on every object of seven park files (349 of 349) and what it is for is not
described here.

**The name is two lines.** `mNameA` is the first and `mNameB` the second, each up to 32 characters and a
terminator, stored a character of each in turn. They are rows of the running language's `OBJECT_NAMES.str`: the
game holds a table of item id, first row, second row for 281 items, and a ride's name is two rows (`Belly` and
`Bounce`, `Crazy` and `Ape`, `Temple` and `Of Gloom`) where every other item's is one, its second row the
table's empty row 1 (`Small Toilet`, `Drinks Shop`). The gates carry the park's name, `LOST` and `Kingdom`. A
save written under another language holds that language's rows (`Small bathroom`, `Trash Can` in a park saved by
the American release). In a record the game has just made, the bytes after each terminator are whatever the
memory held, not nought.

**A record the game has just made** (a Crazy Ape bought and the park saved 111 ticks on, measured under Proton,
2026-10-08) holds: the date on the park's calendar; `mFlags` `0xc`; `mState` 0; `mTopLeft` the anchor; its entry
and exit cells; `mNext` the object that headed the list before it, itself the new `mFirstObject` and the new head
of the thing list; `mBackOfQueue` the one queue cell the game lays before its entrance and
`mQueueSizeInCells` 1; `mCanLoad` 0, because a thing with a queue is built closed; `mIsTrackRideValid` 1; each
ring with `mNumEntries` 30, not wrapped, its unreached entries unwritten memory; the item's starting capacity,
duration and speed; quality and ingredient 50, chance 100; and the three floats 100.0. Its id joins listener set
`0xb`. Only its anchor cell names it in `mWho`; all sixteen cells of its footprint carry the anchor's packed id
as their parent.

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

**The whole of `mFlags` is nine keys of the item's description, a key a bit**, as the game sets them when it
builds the object:

| Bit | Key |
|---|---|
| `0x1` | `UsageInfo.ProvidesRelief` |
| `0x2` | `UsageInfo.ChillsYouOut` |
| `0x4` | `Info.IsChoosable` |
| `0x8` | `Info.HasQueue` |
| `0x10` | `UsageInfo.ProvidesSecurity` |
| `0x20` | `UsageInfo.RideHandlesSprite` |
| `0x40` | `UsageInfo.HoldsLitter` |
| `0x80` | `UsageInfo.IsFireworks` |
| `0x100` | `Info.RunsContinuously` |

Each key is the item's own where its file names it and its folder's where it does not. Measured on the 382
placed objects of thirteen park files (Lost Kingdom's shipped park, four the original wrote from it, and a
player's two saves each of Lost Kingdom and Wonderland; the rest of the thirteen hold no placed object): the
word is the nine keys' on all 378 whose item one of the four item folders holds, and nought on the four
others, two Dino Karts upgrades in each Lost Kingdom save, whose files set none of the nine. The bits set
among the 382: `0x1` 103, `0x2` 17, `0x4` 185, `0x8` 32, `0x10` 82, `0x20` 175, `0x40` 33, `0x100` 11, and
`0x80` on none. No higher bit is set on any.

**`mTopLeft` is the anchor cell moved by the item's `Info.MapOffsetX` and `Info.MapOffsetY`**, the pair turned
by the object's angle and a half turn more. For an offset of `(x, y)`: unturned, `x` columns and `y` rows
before the anchor; at 90 degrees, `y` columns before it and `x` rows after; at 180, `x` columns and `y` rows
after; at 270, `y` columns after and `x` rows before. In the same thirteen files the 378
objects above all hold the cell this gives. Two records have an item with an offset, the Huge Hollow Rock
(`Info.MapOffsetY 1`) unturned, a row above its anchor, in both Lost Kingdom saves; and the Dino Karts
Tunnel, an upgrade whose own file declares the same `Info.MapOffsetY 1`, stands at 90 degrees a column
before its anchor in both. No object with an offset stands at 180 or 270 degrees in these files.

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

**`mPersonBeingLoaded` is the guest the object has called forward** - two bytes at 1040, a handle. Where it is set
it names the head of the object's queue (`mFirstInQ`), a guest in state 11 or 13: on seven objects of Alexah's played
jungle park, and on the Belly Bounce of a park saved by the game with a guest stepping up to it. Nought on all
fourteen objects in the shipped park.

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

### The advisor (model 11)

Thing 3 in every park file, 5,846 bytes, named by no header field. The game names each field as it writes it:

| Offset | Size | Name | Description |
| ------ | ---- | ---- | ----------- |
| 0      | 16 bytes | | What every thing begins with |
| 16     | 4 bytes | `mTimeSig` | `0` in 21 park files |
| 20     | 4 bytes | `mSoundId` | `0` in 21 park files |
| 24 + 21*n*, *n* 0 to 7 | 21 bytes each | | The eight slots: the messages waiting to be said (below) |
| 192    | 4 bytes | `mNextAvailableQueuePos` | `0` in 21 park files |
| 196    | 4 bytes | `mLastActionStarted` | When his last line began, a reading of the clock module's second stopwatch, in milliseconds |
| 200    | 4 bytes | `mLastActionDuration` | That line's length and a second more, in milliseconds; 0 when a line was cut off |
| 204 + 16*i*, *i* 0 to 350 | 16 bytes each | `mHistory[`*i*`]` | The message records (below) |
| 5820   | 21 bytes | | One more message, laid out as a slot: a copy of the last message of a kind the game keeps aside as it is given (which kind is not described here) |
| 5841   | 1 byte | `mRequestedLastTutorial` | `CD` in the shipped file and files saved from it, 0 in five Full Simulation parks |
| 5842   | 4 bytes | `mMessagePlaying` | The message he last said |

**His wait.** He says nothing new until the clock module's second stopwatch (the second dword of `KOLC`) passes
`mLastActionStarted + mLastActionDuration`, so a file whose sum is above its own stopwatch was saved while he was
still inside the wait, and a load goes on waiting for the difference. Two of 21 park files were saved so (959 ms
and 28,196 ms left); in the other nineteen the wait was over by 134 ms to fifteen minutes. Five of the 21 hold a
duration of 0. `mMessagePlaying` is `0x11a` in the shipped file, and 0 in one Full Simulation park whose wait
was still running.

**A slot** is one message he has been given and has not said yet:

| Offset | Size    | Name | Description |
| ------ | ------- | ---- | ----------- |
| `0x0`  | 2 bytes | `mThingId` | The thing the message is about; 0 for none |
| `0x2`  | 4 bytes | `mSecondary` | That thing's item (1100 for a Belly Bounce); 0 for none |
| `0x6`  | 4 bytes | `mMessage` | The message's id |
| `0xa`  | 4 bytes | `mScore` | Its score: the highest-scoring taken slot over 25 is said next |
| `0xe`  | 1 byte  | `mValid` | 1 while the slot is taken. When the message is said only this byte is cleared, and the rest stays |
| `0xf`  | 4 bytes | `mResponseNo` | The line asked for; -1 to take the next in turn |
| `0x13` | 1 byte  | `mOverrideOnlyOnce` | 1 for a message that is said even if it was said before |
| `0x14` | 1 byte  | `mScoreOverridden` | 1 when the score was given with the message and not worked out from it |

A slot never used is `CD` in every byte but `mValid`. In 21 park files twenty hold exactly one taken slot, always
the same: message `0x80` ("the park is open") at a score of 20, in slot 2 or slot 0. Its score is under 25, so it
is never said and stays in every save made from the shipped park. `Easymode.TPWI`'s slots 0 and 1 hold messages
`0xbc` and `0x15a` at 1000, both already said.

**The message records.** From 204 (`0xcc`) the record holds 351
message records of sixteen bytes, one for each message the advisor can be given, in the order of their ids:

| Offset | Size    | Description                                                                              |
| ------ | ------- | ---------------------------------------------------------------------------------------- |
| `0x0`  | 4 bytes | The game tick he last said the message on; 0 if never                                    |
| `0x4`  | 4 bytes | The line he said, counted from 0; -1 before any                                          |
| `0x8`  | 1 byte  | 1 once the message has been said, 0 again when he is cut off saying it                   |
| `0x9`  | 3 bytes | Not written by the game: `CD CD CD` in the shipped file and files saved from it, 0 in five Full Simulation parks        |
| `0xc`  | 4 bytes | How many times he has been cut off saying it                                             |

A record he has never used is (0, -1, 0, 0). In eighteen park files, 6,318 records, the tick is never past the
file's own, the said byte is 0 or 1, the line -1 to 2 and the cuts 0 to 3. `Easymode.TPWI` holds six records
used: messages 0 (tick 1), `0xb9` (257), `0xbd` (55, cut off once), `0x11a` (605), `0x135` (89) and `0x143` (16).

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

A file the game saved in the middle of a strike (from its own menu, every kind's level 3) holds `mForceStrike` `1`,
`mStaffMemberPickedUp` `0`, and `3`, `1`, `17557` on all five: the flag is written as `1` and the stamp is the
tick of the month's change that called the strike. Five files the game saved from parks begun in Full Simulation
hold `0`, `0`, `0` on all five and `mForceStrike` `0`. (OpenTPW Q523; its `docs/exe/saves.md`, "OpenTPW's writer,
the strike records", says what the game does with them.)

`16 + 4 + 2 + 5 × 12 + 5 × 4 + 1` is `103`, and in the shipped park the record ends exactly on the World
module's `DLRW`. The five budgets are, in order, the handymen's, the mechanics', the entertainers', the
guards' and the researchers' - models 5, 4, 6, 7 and 8, which is the order of the rows on the game's Staff
Training Budgets screen. What the game does with them each month is OpenTPW's `docs/exe/ride-operation.md`,
"The month's change".

### The park analyser (model 13)

Thing 5 in every park file, 73,544 bytes, named by the header's `mParkAnalyser`, written by `FUN_004c5d70`. Near
its end are **twenty rings of bytes, one per kind of opinion**, each 60 bytes: `mCurrentEntry` (4, from −1),
`mNumEntries` (4, 50), `mWrappedAround` (1), `mTemp` (1) and 50 `mData` (1 each), kind 0 first; after them the
record's last 76 bytes hold `mMisbehavingKids`, `mMisbehavingKidsThatGotAwayWithIt`, `mMisbehavingKidsEver`,
`mLifetimeVisitors`, `mLastStrike`, `mLastPuke`, `mLastPrank`, `mParkLastOpened`, `mLastObjectBuilt`,
`mLastStaffHired`, `mMostPaidForTicket`, `mLargestBusLoad` and `mLongestStay` (the table below). In
`Easymode.TPWI` the rings start 72,268 bytes into the record (the only run of twenty such rings in the whole
stream): kind 0 holds eight samples of 50, the rest are empty, and `mLifetimeVisitors` reads 8. A played park
fills kinds 1 to 4, 6, 7, 9 to 13, 15, 16 and 18 too. What each kind is, and that only kind 0 is ever read back,
is OpenTPW's `docs/exe/ride-operation.md`, "The analyser's samples".

**Its first 62,598 bytes** are the 16 every thing opens with, a grid of 32 × 32 blocks, four small rings and six
kept answers. Measured in all 539 different park bodies on hand (787 files: the shipped `Easymode.TPWI`, Alexah's Full
Simulation saves, the parks the original wrote under Proton and OpenTPW's), every one as below:

| At | Size | Field | In every file |
|---|---|---|---|
| 16 | 4 × 5 | `mMapSquaresPerCell`, `mGridSizeX`, `mGridSizeY`, `mMapSizeX`, `mMapSizeY` | 4, 32, 32, 128, 128 |
| 36 | 61 × 1,024 | the blocks, row by row (`FUN_004c85b0`): `mStartX`, `mStartY`, `mSizeX`, `mSizeY` (4 bytes each), then nine kept answers of 5 bytes, `mVal` (4) and `mWorkedOut` (1) | block *k* is ((*k* mod 32) × 4, (*k* div 32) × 4, 4, 4); `mWorkedOut` 0 or 1 |
| 62,500 | 17 × 4 | four small rings, the recently researched of the buy list's four tabs (`FUN_004cdb80`): `mCurrentEntry` (4), `mNumEntries` (4, 3), `mWrappedAround` (1), `mTemp` (2) and three 2-byte `mData`, item ids | `mNumEntries` 3 |
| 62,568 | 5 × 6 | six kept answers, `mVal` (4) and `mWorkedOut` (1) | |
| 62,598 | | the sixteen rings of dwords, below | |

**A kept answer saves its value and its flag, never the tick it was worked out on** (`FUN_004cd9c0`): the stamp
before them in memory is not in the file. One never worked out holds `mVal` −999 and `mWorkedOut` 0. The six, in
order: the guests inside (`Easymode.TPWI` 0), the rides standing (1), the people whose `+0x220` is 3 in the chains of two
cells (−999, never worked out in any file), the objects with bit 0 of a byte at their `+0x32` (3), the park value (23977, the
2nd ring's month) and a sixth (−999; 53 in the same two Full Simulation files as the small rings below). What each does with them is OpenTPW's `docs/exe/hud.md`,
"The park value" and "The guests inside and the rides at a close".

**A block's nine** hold −999 when never worked out, the first two as a 4-byte float (`0xC479C000`), the other seven
as an int. Only the sixth, seventh and eighth are ever worked out in a file: the eighth in every block of all but
ten bodies (0 in 528,342 of them, 1 to 16 in the rest), the sixth and seventh together in the same 877 blocks. What
the nine count is not read.

**The four small rings** read −1, 3, 0, 0 and the allocator's `0xCDCD` (or 0) in their slots in 537 of the 539:
never pushed. In Alexah's played Full Simulation jungle (its `New Save.TPWS` and `autosave.TPWS`, the same in both)
they hold catalogue item numbers a hundred to a ring: 1102, 1104, 1108; 1208, 1202, 1206; 1304, 1309, 1307; 1435,
1407, 1434. They are the rides', the shops', the sideshows' and the features' rings of the recently researched, in
that order (the items' `Info.WhichUIType`): a finished research project pushes its item into its tab's ring, the entry
stepping on and wrapping to the first slot past the third, and the buy list stars a row whose item is one of a ring's
three newest. In Alexah's park the rides' ring stands on slot 2 and has wrapped (so 1108 is the newest, then 1104,
1102), the shops' on 1, wrapped, the sideshows' on 2, not wrapped (three pushed), the features' on 1, wrapped. What the
game does with them is OpenTPW's `docs/exe/park-engine.md`, "The lab at work", and `docs/exe/hud.md`, "What the buy list
actually filters on". A park OpenTPW wrote just after the stock Lost Kingdom park's first finish (`q702`) holds the rides'
ring at `mCurrentEntry` 0, unwrapped, 1150 (Dino Karts) in its first slot and the file's `0xCDCD` in the other two.

**Sixteen rings of dwords** lie 62,598 bytes into the record, one after another, 589 bytes each: `mCurrentEntry` (4,
−1 before the first month is closed), `mNumEntries` (4, 144), `mWrappedAround` (1), `mTemp` (4, the month so far) and
144 `mData` (4 each, signed). Measured in all 513 park files on hand: the shipped `Easymode.TPWI` (the only one, the
jungle's), Alexah's five Full Simulation saves and a dozen others the original wrote under Proton, and OpenTPW's. A slot never filled holds `0xCDCDCDCD`, the allocator's
fill. The first and the third to ninth are the month's money, the second the park's value, the tenth to thirteenth the guests inside, the visitors, the rides and the arrivals, the fourteenth their happiness, its months **floats** where every other ring's are dwords, the fifteenth the guests' stay, the sixteenth a score of the park:

| Ring | At | What its months are | `Easymode.TPWI` |
|---|---|---|---|
| 1st | 62,598 | the bank's balance as each month was closed; its `mTemp` is 0 in every file | one month closed, 88325 |
| 2nd | 63,187 | the park value as each month was closed; its `mTemp` is 0 in every file | one month closed, 23977 |
| 3rd | 63,776 | every withdrawal made | one month closed, 13175; 538 so far |
| 4th | 64,365 | the staff's wages and severances | 0; 538 so far |
| 5th | 64,954 | the staff's training | 0; 0 so far |
| 6th | 65,543 | every deposit | 1500; 200 so far |
| 7th | 66,132 | the gate's admission fees | 0; 200 so far |
| 8th | 66,721 | what the shops have charged | 0; 0 so far |
| 9th | 67,310 | what the sideshows have charged | 0; 0 so far |
| 10th | 67,899 | the guests inside the park as each month was closed; its `mTemp` is 0 in every file | one month closed, 0 |
| 11th | 68,488 | `mLifetimeVisitors` (73,480, below) as each month was closed; its `mTemp` is 0 in every file | one month closed, 0 |
| 12th | 69,077 | the objects whose item is a ride's kind (`Info.WhichUIType` 0, an upgrade's too) as each month was closed, whatever their state; its `mTemp` is 0 in every file | one month closed, 1 (the Belly Bounce) |
| 13th | 69,666 | the guests the arrival vehicles dropped off, one by one, in each month; `mTemp` the month's so far | one month closed, 13 (its thirteen guests); 0 so far |
| 14th | 70,255 | the mean happiness of the guests inside the park as each month was closed, a 4-byte float (0.0 with none inside or the park shut); its `mTemp` is 0 in every file | one month closed, 0.0 |
| 15th | 70,844 | the mean of the last fifty stays below as each month was closed, whole days truncated (0 with none); its `mTemp` is 0 in every file | one month closed, 0 |
| 16th | 71,433 | the park's score out of 100 as each month was closed, eleven counts each capped (the guests inside, the rides, shops, sideshows and features, the rides upgraded twice, each kind of staff); its `mTemp` is 0 in every file | one month closed, 18 |

**After the sixteen, at 72,022, a ring of the last fifty stays**, 213 bytes of the same shape: `mCurrentEntry` (4, −1
before the first), `mNumEntries` (4, 50), `mWrappedAround` (1), `mTemp` (4, 0 in every file) and 50 `mData` (4 each):
each one a guest's whole days in the park, put there as a guest who paid at the gate stands at the stop to go home. In
`Easymode.TPWI` it is empty. **After it, at 72,235, a ring of the last five loads the arrival vehicles were called
for**, 33 bytes of the same shape: `mCurrentEntry` (−1 before the first), `mNumEntries` (5), `mWrappedAround`, `mTemp`
(0 in every file) and five dwords, each a load's headcount as the load was called. A slot no load has reached holds
whatever the game's memory held: `0xcdcdcdcd` in `Easymode.TPWI`, whose one load is its 13, and 0 in the parks the game
made fresh. While the ring has not gone round, its loads sum to the guests the 13th ring counts dropped (its months and
its month so far): measured in 584 of the 599 such files on hand, the 15 others OpenTPW's own, written before it kept the
ring. Then, at 72,268, the twenty rings of bytes above.

**Every ring is closed at once, so all sixteen stand on one slot**, and the gate's, shops' and sideshows' are part of
the deposits: measured in 612 of the 616 park files on hand (the shipped one, Alexah's Full Simulation saves, every
file the original wrote under Proton and OpenTPW's), the sixteen `mCurrentEntry` are equal and in every closed month
and the month so far the 7th, 8th and 9th together are at most the 6th. The four that are not are OpenTPW's own,
written before it kept the 5th and 7th to 9th. **Each closed month of the 1st is the one before it plus that month's 6th
less its 3rd**, the bank's balance carried through the deposits and withdrawals: measured in 572 of the 573 files on hand (of
579) whose rings stand on one slot; the one that is not, a save of the original's, holds as its second month
the shipped file's month so far unchanged (6th 200, 3rd 538) where its balance fell 494 more: by every sign the month it
took over from a file OpenTPW wrote before it kept the 3rd and 6th, which wrote the balance and the file's rings. **The 2nd is a sum over the park as it stood at the close**: every cell inside the park by
its `mType` and its track record, every object not condemned by its item's upgrade costs and its State of repair (the
rule is OpenTPW's `docs/exe/hud.md`, "The park value"). From `Easymode.TPWI` alone it sums to 23977, its own month:
cells 19527, objects 4450; of the 581 files on hand 186 sum to their newest month, the rest worn or built on since.
**The 11th never falls and its newest month is at most `mLifetimeVisitors`**, in all 584 files on hand; in all 578
where it stands on the 7th's slot, the 7th's month so far is `mLifetimeVisitors` less the 11th's newest month, times one
admission fee (25 in 561 files, 20 in 4, none since in 13), never above `mMostPaidForTicket`: one visitor is counted for
each fee the gate takes. **The 12th's newest month is the file's things of type 3 whose item is a ride's kind** in all 188
of the 587 files on hand whose 2nd ring's newest month is their own park value (untouched since their close); **the 10th's
is at most the file's guests** in 586 (the one other closed on 30 and holds 27, its guests gone since). The 13th's month
so far is never below 0 in the 589 files on hand; the header's `mNumberOfVisitorsToDate` is no partner of it (the months
and the month so far sum to it in 24 of the 42 files whose 13th has moved). **Each closed month of the 14th times the
10th's month is a whole number**, the guests' happiness bytes summed: in 243 of the 270 closed months with guests inside
of the 593 files on hand (the 27 others three months of one Full Simulation park in nine copies), and every month with
none reads 0.0; every month lies from 0 to 255. **Each closed month of the 15th is the mean of the stays the ring of
fifty held at that close**: in all 34 of the 596 files on hand whose ring of stays holds any and has not wrapped, each
month is the truncated mean of its oldest k stays for some k (those that left after the close later in the ring); the
553 with no stay read 0, and the 9 whose ring has wrapped cannot be checked. The stays seen run from 18 to 587 days. **The 16th's newest month is the score of the
file's own park**, reckoned from the 10th's newest month and the file's things (`Easymode.TPWI`'s 18: one ride, one
shop, one sideshow, eight features and one of each kind of staff), in 180 of the 197 of the 598 files on hand whose 2nd
ring's newest month is their own park value; each of the 17 others has hired or lost staff since its close. Training is under way in 9 files, gate takings in 613, shops' and
sideshows' takings in 44.

The hire and training screens' account reads the 3rd, 4th and 6th rings' newest month; the finances screen graphs
the 1st to 4th and 6th to 9th, every money ring but training among them; the park status screen graphs the 10th
beside the 13th to 16th, the 13th its first series; nothing reads the 12th. What feeds them and when a month is closed is
OpenTPW's `docs/exe/hud.md`, "The mini-balance is a MONTHLY ACCOUNT"; what counts the visitors, and what reads them,
its "The lifetime visitors"; what counts the guests inside and the rides, its "The guests inside and the rides at a close";
what counts the arrivals, its "The arrivals"; how a stay is reckoned, its "The leavers' stays"; how the score is counted, its "The park score".

**The record's last 76 bytes**, measured in `Easymode.TPWI`, three jungle files the original wrote under Proton
and Alexah's five Full Simulation saves (jungle, fantasy and hallow):

| Offset | Size | Field | |
|---|---|---|---|
| 73,468 | 4 × 3 | `mMisbehavingKids`, `mMisbehavingKidsThatGotAwayWithIt`, `mMisbehavingKidsEver` | 0, 0, 0 in seven of the nine; 0, 0, 3 in the two saves of the longest-played park; the first two are this month's, the third is never read back (below) |
| 73,480 | 4 | `mLifetimeVisitors` | 8 in `Easymode.TPWI`, 759 in the longest-played park's two; one for each admission fee taken, so `Easymode.TPWI`'s 8 are its gate's 200 so far at 25 |
| 73,484 | 8 × 3 | `mLastStrike`, `mLastPuke`, `mLastPrank` | the calendar's start in all nine, and in all 612 park files on hand: the game never moves them from its constructor's |
| 73,508 | 8 | `mParkLastOpened` | 2000-01-01 15:37:30 in `Easymode.TPWI`, tick 15; the calendar at the newest request to open the park, later than the start in 609 of 612 files |
| 73,516 | 8 | `mLastObjectBuilt` | 2000-01-27 05:10:00 in `Easymode.TPWI`, tick 604 |
| 73,524 | 8 | `mLastStaffHired` | 2000-01-29 03:00:00 in `Easymode.TPWI`, tick 648 |
| 73,532 | 4 × 3 | `mMostPaidForTicket`, `mLargestBusLoad`, `mLongestStay` | 25, 0, 0 in `Easymode.TPWI`; 20, 0, 0 in four Full Simulation saves and 0, 0, 0 in the fifth, saved on tick 4; the first is the dearest admission fee taken |

**The three misbehaving kids' counts.** Of 608 park files on hand, 594 hold 0, 0, 0, the longest-played park's nine
copies 0, 0, 3, and one file the original wrote 1, 1, 1 (a kid who got away that month). The first two count this
month and are nought just after a month's close; the third is written but **never read back by the game**: its load
reads the first two and writes over the third, so a file's third counts only since that session's load (and holds
whatever the game's memory held at the load: nought in two loads measured, -999 in a third). What counts into them, and
what reads the first, is OpenTPW's `docs/exe/hud.md`, "The misbehaving kids".

The six stamps are times on the park's calendar, each a `FILETIME` as a member of staff's `mTimeHired` is (below):
the calendar's start, 2000-01-01, plus 3,750 seconds for each tick of `mGameTick` in the jungle's files. **`mLastStaffHired`
equals the newest `mTimeHired` among the file's staff in all eight files that hold staff**, and in the ninth, a park
saved on tick 4 with nobody hired, all six stamps are the calendar's start. What the game does with the stamp is
OpenTPW's `docs/exe/advisor-park.md`, "More rides than the mechanics can keep". **`mLastObjectBuilt` is the
build date of the newest object the park has had** (an object's own is the eight dwords at 22 of its record): in
ten of twelve files the original wrote it equals the newest standing object's, and in the two saves of the
longest-played jungle park it is a month newer than every one of the 124 objects standing, a thing built and since
sold. What writes it is OpenTPW's `docs/exe/advisor-park.md`, "The newest build".
**`mParkLastOpened` is the calendar at the newest request to open the park**, the park already open or not, and the
three stamps before it never move from the calendar's start; nothing the game does reads any of the four. What
writes them is OpenTPW's `docs/exe/hud.md`, "The analyser's other stamps".

### The research lab (model 14)

Thing 6 in `Easymode.TPWI`, 4,190 bytes, named by the header's `mResearchLab`. After the sixteen bytes every
thing's record begins with it holds **256 projects of 16 bytes**, used or not, then **five records of 13 bytes,
one per kind of research**, then thirteen bytes that close it. The names are the reader's own.

| Offset | Size | Field | |
|---|---|---|---|
| 16 | 256 × 16 | the projects | the first `mNumProjects` are used; the others hold whatever was there, 928 of 1,952 unused ones not all nought |
| +0 | 2 | `mObjId` | the catalogue id of the item the project researches |
| +2 | 4 | `mCat` | the kind of research: 0 a ride, 1 a shop, 2 a sideshow, 3 a feature, 4 any other |
| +6 | 1 | `mResearchGroup` | 0 to 4 |
| +7 | 1 | `mUpgradeLevel` | 0 to 2 |
| +8 | 4 | `mCost` | a float: the research points the project takes |
| +12 | 4 | `mResearchDone` | a float: the points put to it |
| 4,112 | 5 × 13 | the kinds, 0 to 4 | |
| +0 | 4 | `effort` | 100, 15, 30, 25 and 0 in `Easymode.TPWI`, which sum to 170 |
| +4 | 4 | `output` | 49, 6, 14, 11 and 0 there |
| +8 | 4 | `currentProject` | the index of the project the kind is working on, −1 for none: 1, 20, 22, 8 and −1 there |
| +12 | 1 | `highestGroupAvailable` | 1 in every kind there |
| 4,177 | 4 | `mNumProjects` | 24 in `Easymode.TPWI` |
| 4,181 | 4 | `mWorkLoad` | 85 there |
| 4,185 | 4 | `mMostExpensiveProject` | 3 there |
| 4,189 | 1 | `mLabActive` | 1 there |

**Measured in nine files**: `Easymode.TPWI`, the reference player's copy of it, two jungle files the original
wrote under Proton and Alexah's five Full Simulation saves (jungle, fantasy and hallow). The record is 4,190
bytes in each; `mNumProjects` is 24 in the four Instant Action files and 44, 44, 48, 60 and 60 in the Full
Simulation ones; `mLabActive` is 0 or 1; and every one of the 352 used projects has a kind from 0 to 4, a cost
above nought and points at or above nought.

**A project's kind follows its item**: all 87 of kind 0 are items of the rides' hundred of their theme (11xx in
the jungle), all 37 of kind 1 the shops' (12xx), all 20 of kind 2 the sideshows' (13xx) and all 141 of kind 3
the features' (14xx). The 67 of kind 4 are 54 of the rides' hundred, 44 at upgrade level 1 and 10 at level 2,
and 13 of the next hundred (15xx) at level 0; every project of kinds 0 to 3 is at level 0. **A project that is
done stays in the list**: in 80 of the 352 the points are at or over the cost, all in the Full Simulation saves.

**A park the game has just made** (the jungle `restart.INTS` of a new Full Simulation park, tick nought, one
written on Windows and one under Proton) holds 48 projects, none begun: 11 of kind 0, 5 of kind 1, 3 of kind 2,
20 of kind 3 and 9 of kind 4, in the order of the object controls. The kinds' `effort` is 100, 15, 30, 15 and
10, `output` 49, 6, 14, 6 and 4, `currentProject` 4, 41, 46, 17 and 0; `mWorkLoad` 85, `mMostExpensiveProject`
3, `mLabActive` 0. The two files' used projects, kinds and closing fields are the same byte for byte; they
differ only in the 208 unused projects, from byte 784 of the record on. The fantasy and hallow `restart.INTS`
hold 44 and 48 projects with the same efforts and outputs.

In `Easymode.TPWI` the 24 are 8 rides, 3 shops, 2 sideshows and 11 features, none done; three have points
(30 of 600, 4.5 of 450 and 9 of 700) and the features' current one, project 8, has 7.5 of 250. What the game
asks of the list is OpenTPW's `docs/exe/advisor-park.md`, "Every feature built, and more to research".

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

### The weather (model 15)

The thing `mWeather` names - thing `7` in every park file. Its record is **99 bytes**, read field by field after the
sixteen every thing opens with (`FUN_00512010`; the forecast's three fields by `FUN_005132a0`):

| Offset | Size | Name | Shipped park |
| --- | --- | --- | --- |
| 16 | 4 bytes | `mQuality` | `78` |
| 20 | 8 bytes | `mLastChanged` | a `FILETIME` |
| 28 | 8 bytes | `mLastForecasted` | a `FILETIME` |
| 36 | 8 bytes | the forecast's `mTime` | a `FILETIME` |
| 44 | 4 bytes | the forecast's `mQuality` | `78` |
| 48 | 1 byte | the forecast's `mValid` | `0` |
| 49 | 4 bytes | `mCurrentDrops` | `0` |
| 53 | 4 bytes | `mTargetDrops` | `0` |
| 57 | 1 byte | `mLightning` | `0` |
| 58 | 4 bytes | `mLightningCountdown` | `0` |
| 62 | 4 bytes | `mLightningNumThisPeriod` | `0` |
| 66 | 4 bytes | `mThunderCountdown` | `-755` |
| 70 | 12 bytes | `mLastLightningLocation` | `0xcdcdcdcd` three times |
| 82 | 4 bytes | `mRainSoundHandle` | `0` |
| 86 | 4 bytes | `mPType` | `0xcdcdcdcd` |
| 90 | 4 bytes | `mWindDir`, a float, degrees | `139.0` |
| 94 | 4 bytes | `mWindForce`, a float | `0.22` |
| 98 | 1 byte | `mOverridden` | `0` |

The sizes sum to 99, and the next thing's list head starts on byte 99 in all six park files that hold one (the shipped
Lost Kingdom park and five Full Simulation saves of three themes). The three stamps are the park calendar's (the shipped
park's 2 February 2000 09:05, 29 January and 2 February 08:12:30). `mWindForce` is `(100 - mQuality) / 100` in five of
the six (0.22, 0.27, 0.32); the Halloween World save holds 0.50 beside a quality of 86: the constructor's force, over a
quality it rolled and never applied (`FUN_00511f70` stores the roll at `0x00511fc4` and does not call
`FUN_00512f00`); its first forecast had not arrived. `mWindDir` is a whole number of degrees in each, 139, 24, 90 and 350. `0xcdcdcdcd` is the
debug heap's fill, written where nothing was ever stored: a park where no bolt has struck holds it in the location, and
`mPType`, which nothing reads, holds it, nought or another value. The executable's behaviour is OpenTPW's `docs/exe/weather.md`.

**The stamps are the park calendar's, on whole ticks.** Taken as ticks of the file's own clock (2000-01-01 plus a tick
of 3,750 seconds at the rate 15000; "The arrival and clock fields"), `mLastChanged` is a whole tick at or before the
file's `mGameTick` in 43 of the 48 park files in the original's folders on the machine measured (Q528, 741 files); the
other 5 hold nought, the constructor's: four saved at tick 0, and a Halloween World autosave on tick 4 whose first
forecast (made on tick 1) had not yet arrived; three of the 48 are files OpenTPW wrote, kept there to be loaded, and ten
are byte copies of the disc's park. The forecast's `mTime` is `mLastForecasted` plus exactly four days (92.16 ticks) in 44 of them, the other four, saved at tick 0, holding
noughts. The shipped park was changed on tick 746, forecast on 653 and due on 745.16. `mThunderCountdown` falls by one
every weather tick, a storm or not, so between storms it is the ticks since nought taken off: -755 on the shipped
park's tick 755, and so in 40 of the 48; in five of the original's own files it is the tick of a load or of a later
bolt, and in the three of OpenTPW's it is the stock -755 carried. `mLastLightningLocation` is written only as a bolt arms the thunder: (x, 300, z),
the bolt's top, in 3 of the 48, the debug fill in 38, and bytes nothing wrote in the other 7. `mRainSoundHandle` and
`mOverridden` are nought in all 48 (none was saved raining). A file OpenTPW writes holds its running weather in the same
layout (its handle always nought), and the original loads it and runs on from it.

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

**A file the game saved with a loan running** (OpenTPW Q524, `Q524ORIG-by-the-original.TPWS`): the original's own
save of a Lost Kingdom park that OpenTPW wrote, with loan 4 marked bought by a byte edit (no screen in either game
buys one) and its red stamp, last balance and months repaid made by OpenTPW. The original carried them through its
load and save unchanged; only `mBalance` and `mProfitThisYear` moved there, by admission fees. It shows the original's
layout, not the original's stamping. Loan 4 holds `loan_bought` `1` and `months_repaid` `6`, the slot's other six
fields as offered; `mTurnEnteredRed` is `17557`, the tick of OpenTPW's withdrawal that took the balance below nought,
and `mLastBalance` `-1188`, that withdrawal's balance, while `mBalance` had moved on to `-796`; `mAdmissionFee` `28`, `mProfitThisYear` `392`. The other seven loans are the offer as shipped.

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
| 28 | 2 bytes | `mAccurateDestX` - the navigator's `path_target_pos` in `mX`'s units (shifted right by eight): 18 of 18 in the shipped park, 392 of 392 in a played one |
| 30 | 2 bytes | `mAccurateDestY` |
| 32 | 2 bytes | `mAdjustorSpeed` |
| 34 | 2 bytes | `mBaseSpeed` |
| 36 | 1 byte | `mCount` |
| 37 | 4 bytes | `mESPSprite` - the sprite kind worn: 0 a child, 2 a costume, a member of staff their own |
| 41 | 2 bytes | `mLastRecordedMapId` - a packed cell id (`y × 128 + x + 1`): the cell the person was made on, and nought for an arrival, who is made on none. Only an entertainer's and a guard's is ever changed: it follows them, a step behind at most, and is the cell their region effect stands round (the map cells' effects sub-record). In thirteen park files every entertainer's and guard's is the cell they stand on or the one before, and no mechanic's, handyman's or researcher's is |
| 43 | 177 bytes | the navigator - see below |
| 220 | 4 bytes | `mPreviousSpeed` |
| 224 | 4 bytes | `mPreviousX` - where the person stood a turn before, in the navigator's fixed point, not a float |
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
| 402 | 4 bytes | `mArrivalIndex` | `0` on all thirteen, none yet through the gate: the guest's number among the park's visitors, the world's `mNumberOfVisitorsToDate` as they were counted in, at the gate for one made outside the park and as they were made for one made on a park cell. `1` to `13`, each once, in a save made with thirteen visitors |
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
| 476 | 2 bytes | `mPatrolRegionBL` | packed cell id: `2728`, `2728`, `2352`, `2728`, `1` |
| 478 | 2 bytes | `mPatrolRegionTR` | packed cell id: `3641`, `3641`, `3249`, `3641`, `16384` |
| 480 | 1 byte | `mPercentageThroughGrade` | `0` on all five |
| 481 | 2 bytes | `mRestArea` | `0` on all five |
| 483 | 4 bytes | `mState` | `1`, `1`, `1`, `0`, `1` |
| 487 | 4 bytes | `mTimeStartedIdling` | `0`, `0`, `712`, `752`, `0` |
| 491 | 8 bytes | `mTimeHired` | a `FILETIME` (hundreds of nanoseconds since 1601): `1.1.2000 15:37:30` on four and `29.1.2000 03:00:00` on the researcher (below) |
| 499 | 4 bytes | *unnamed float* - tiredness | `75`, `75`, `82`, `79`, `93` |

**The patrol corners** are two packed cell ids, `y * 128 + x + 1`, the bottom-left and the top-right of the
rectangle the member keeps to: `2728` and `3641` are (39,21) and (56,28). **`1` and `16384` are the whole map**, (0,0)
to (127,127), which the game's staff constructor writes for every hire. Measured: of the 130 members in the four
park files with staff that the game saved itself (eight files in all), 20 carry them and the rest are fenced in; none
carries `0` and `0`, and neither does a member of the shipped park.

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

**`mTimeHired` is the park's calendar as the member was made**: the calendar's start plus `mGameTick` x
`mFunnySecsPerRealSec` / 4 seconds, 3,750 seconds a tick in every shipped park, stored whole as a `FILETIME`. The
shipped four read tick 15 exactly and the researcher tick 648; the 53 members of Alexah's played jungle park read 53
different dates from 2.3.2000 to 25.2.2002, each a whole number of ticks, the newest hires the latest. The staff
window's days employed are counted from it.

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
taken down by one a sweep. `mObjectToRepair` is the id of the object the mechanic's last search found, `0` for none: it
is set while he walks to the ride and repairs it, and also left set when no route reached the ride. The object
names him back in its `mAssignedStaffMember`.

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

A handle is the address the record had in the session that saved it, and the loader only tests it against
nought: it allocates each live record afresh and stores the new address in the slot. A file with every live
handle set to `1` loads and its people walk (measured under Proton, 2026-10-08).

A new sprite takes the lowest empty slot from 1, and when none is empty the table grows by fifty; the loader makes
its table of the count the file gives. A sprite the game has just made holds its slot, state `1`, a loop stack with all its room (`20`), alpha
`255`, both scales `1.0`, its program and its place, and whatever the allocator left elsewhere: a file whose new
sprites are nought everywhere else loads and its people walk (measured under Proton, 2026-10-08, with a park
written by OpenTPW).

Slot 0 is never used. The park the game ships has 100 slots of which 18 are live, and those 18 are
exactly its people: thirteen from the `kids` banks and one each from `entertainers`, `handymen`,
`mechanics`, `guards` and `researchers`. People are not the only sprites: Alexah's played jungle park has 450
slots with 439 live in its save and 435 in its autosave, for 392 people in the thing list.

A record is a runtime structure written out whole, so most of it is bookkeeping. The fields that can
be named from the code that fills them are:

| Offset | Size | Description |
| --- | --- | --- |
| `0x04` | 4 bytes | The record's own slot in the table |
| `0x08` | 4 bytes | How far into that program it has got. Because showing a frame is the only thing that ends a turn, a saved value always rests just past a frame instruction |
| `0x0C` | 4 bytes | **Which** animation program - the index of its first instruction, in the same array `0x08` counts into. A jump inside a program moves `0x08` and leaves this alone, so it names the program the sprite was *started* on rather than where it has reached |
| `0x14` | 4 bytes | Where that program starts; re-pointed on load, so the stored value is meaningless |
| `0x18` | 4 bytes | State - `1` as a sprite is made, `2` once its program has run a turn, `4` once it has reached its end word; `2` on all but one of the 1,154 saved sprites to hand |
| `0x1C` | 4 bytes | How much room the loop stack has left: `20` with nothing pushed, one less for each loop the program is inside (below) |
| `0x20` | 4 bytes x 20 | The loop stack: where each loop the program is inside starts, a word of the array. The first pushed lies in the **last** place, `0x6C` |
| `0x78` | 4 bytes | How many loops the program is inside |
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
| `0xBC` | 4 bytes | The frames per direction of the set the sprite was **made** on, the third byte of that set in its bank's `.esp` (below) |
| `0xC0` | 4 bytes | Which of eight ways round it was last drawn facing |
| `0xC8` | 4 bytes | A bank's state group's third byte, on a sprite **made** on a state: the frame a state animation's later rounds start on. `0` on every person's sprite to hand |
| `0xCC` | 4 bytes | That group's fourth byte: how many turns more frame 0 is held between rounds. `0` on every person's sprite to hand |
| `0x114` | 4 bytes | Whether it is showing a frame: `1` once a frame has been shown, `0` as made and again at its end word |

The three floats at `0x88`, `0x8C` and `0x90` are where the sprite stands, in world units at ten to a
map cell.

**`0xBC` is set as the sprite is made and not when its picture changes.** A person's sprite is made on set 0 of
its bank and its animation programs change the set at `0xB4` afterwards, so a walking child on set 1 (eight frames a
direction) still holds set 0's `1`. Only a bank's own state animation writes it again, with that state group's set: an
entertainer that has performed holds its group's (`8` on the jungle's `SPR_DI`, `17` on its `SPR_EX`) and one
that has not holds set 0's. A balloon is made on its colour's set and holds `2`, whole and burst. A thought bubble is
made on bank 0's set 0 whatever its picture and holds `1`. All 1,053 live sprites of the six park files to hand that
hold any (the shipped jungle park, and two played saves each of jungle and fantasy) read exactly this against their
banks: `1` on every child, costume and handyman and on a mechanic of bank 1 (`SPR_OM`), `8` on every head and on a
mechanic of bank 0 (`SPR_FA`), `4` on every guard and researcher.

**A thought bubble is a record of its own, as a balloon is**, named by its person's `mThoughtScript`: kind 9
(`thoughts`), bank 0, `0xB4` the picture (0 to 15 the first bank's sets, 16 up the second's, so here the word does
run past fifteen), frame 0, 2.5 above the ground, on one of 22 programs of six words each from word 1462 and, once
it has shown, four words into it. The programs set pictures 0 to 15 in order, then 21, then 16 to 20, so picture 21
is on word 1558 and picture 18 on 1576. The played jungle saves hold 23 and 19, the fantasy saves 2 each; one of
the 46 is saved at its program's first word, just made.

**The loop stack.** A program's loop start pushes where the loop begins: it takes one off `0x1C` and stores the
word at `0x20` plus four times what is left, so the stack fills from its last place down, and it counts the loop in
`0x78`. An earlier version of this page called `0x1C` a timer; it is the room left. Of the 1,154 sprites in the
twelve park files to hand that the game itself wrote, 1,018 hold `20` and an empty stack, 122 hold `19` with word
1714 in the last place (on program 1704, whose count is at `0x74`, a second counter this page does not explain:
these are the riders' heads hung on rides, below) and
14 hold `19` with word 1730 there and `1` at `0x78` (on program 1760).

**A sprite's drawing flags, `0xC4`.** The word is the program's local 16, which the game's draw reads. The
constructor leaves it nought and the first word of every program a person runs sets it to `0x1200`, so it is
nought only on a sprite made and not yet run a turn: all 902 people's sprites of the ten files hold `0x1200` (a
rider's head holds `0x3000080`, below). A load reads it back. The game's own memory shows it: given a file whose
thirteen newest people's sprites held nought there, it held nought on all thirteen through the 5.6 s of its
loading and set each to `0x1200` within 0.9 s of its clock starting, as each program came round to its first
word; given the same park with `0x1200` written, it held `0x1200` from its first reading.

**A person's own sprite inside a loop.** Those 14 are entertainers performing, which is the one animation of a
person's that loops: a bank's state animation, program 1760, whose loop starts at word 1729 and shows the set's
frames one a turn. Counted again over ten files the game wrote (1,172 sprites), all 14 rest alike: `0x08` at
1732, just past the loop's frame instruction, `0x1C` at `19`, word 1730 at `0x6C`, `1` at `0x78`, nought at `0x70`
and `0x74`, state `2`, shown, `0xB4` set 4, `0xB8` the frame they were on (1 to 14), `0xBC` `8` or `17`, and
nought at `0xC8` and `0xCC`. One more is on program 1760 with an empty stack, at its first word: started again and not yet run. The
stack is needed to go on: the game's loop-end instruction does nothing but log when `0x78` is nought, so a record
resting at word 1732 with an empty stack falls out of the loop on its first turn and starts the round again at
frame 0. The game's own memory shows both: given a file with the stack it showed the saved frame 2 and then 3, 4,
5; given one with `20` at `0x1C` and nought at `0x78` it showed the saved frame 3 and then 0, 1, 2, with `19`,
1730 and `1` in the record from that turn on.

**A balloon let go is a record nobody names.** The game puts the balloon's own sprite on the program at word
1666 and clears its guest's `mBalloonScript`: the alpha is set to 250, then frame 1 (the burst) is shown and 20
taken off the alpha, turn after turn, in a loop that starts at word 1655. So one saved mid-burst is kind 10 on
program 1666, resting on word 1657 with `0x1C` at `19`, word 1655 at `0x6C`, `1` at `0x78`, state `2` and shown;
one just let go is on word 1666 with alpha 255 and an empty stack; one that has finished is state `4`, not shown,
alpha -10, and its slot is emptied on its next turn. None of the twelve files holds one, since a burst lasts
under two seconds. The layout is the game's own after it let balloons go, read from its memory, and the game
loaded a file written this way and burst the balloon on from the alpha the file held (2026-10-09).

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

### A live emitter and its handle

An emitter in use is its template's `0x140` bytes with the words below written over them as it is started
(`Particles_Spawn`, `0x00521e60`), and a handful more that run on afterwards.

| Offset | Size | What a start writes |
|---|---|---|
| `+0x00` | 2 | 1: in use |
| `+0x06` | 2 | Its first particle, -1 for none. The loader sets it to -1 in every emitter in use |
| `+0x08` | 2 | How many particles it has alive. The loader sets it to nought |
| `+0x0a` | 2 | The emitter's count: the live header's word at `+0x18`, which is then one more |
| `+0x0c` | 2 | The template it was started from, its slot in the effect library |
| `+0x14`, `+0x18`, `+0x1c` | 4 each | Where it stands: across, height and down, the park's units times 64 |
| `+0x2c`, `+0x30`, `+0x34` | 4 each | Its velocity: the template's, each plus one draw of the system's generator under the template's range at `+0x12` (no draw where the range is nought; a range of -1, the library's commonest, adds nothing) |
| `+0x38`, `+0x3c`, `+0x40` | 4 each | Its own velocity. A plain start leaves the template's. A start with a direction (`Particles_SpawnFull`, `0x00521930`, a ride script's type 2 record) writes each as the direction's component times 1024, cut to a whole number, times the template's speed at `+0xb0`, over 1024 toward nought; it makes no draw for `+0x2c`. The sweep writes the three again on every tick from the way the record's node points (`Particles_SetVelocity`, `0x0051fd90`) |
| `+0x60`, `+0x64`, `+0x68` to `+0x6b` | 4, 2, 1 each | The burst, the most particles and the four rates. Where the template's byte at `+0xc0` is nought each is the template's times the particle density over 1024, and never scaled down to nought |
| `+0x6f` | 1 | 1 where any of the three words at `+0x84` is not nought |
| `+0xab` | 1 | Nought; 1 on an emitter another one links and carries with it |
| `+0xb4` | 4 | Where the template links another effect: the handle of the emitter started for it |
| `+0xd0`, `+0xd2` | 2 each | The next and the previous emitter of its chain, -1 at an end |
| `+0xd4` | 4 | The life it started with, the template's `+0x20` |
| `+0x128` to `+0x13c` | 4 each | The box its particles have reached: `0x7fffffff` three times and `0x80000000` three times as it starts, where the byte at `+0x05` is nought, and six noughts where it is not |

The word at `+0x0a` makes a handle with the emitter's slot: a ride script's started effect
([the ride script module](#the-ride-script-module-essr)) keeps `count << 16 | slot`. In thirteen park files all 55 particle
records name an emitter in use whose count is the handle's high word; the Drinks Shop's bubbles are handle
`0x7a0014`, emitter slot 20 with a count of `0x7a`, started from template 58.

**Measured:** in eighteen park files, every one of the 61 emitters a script's record names is the bytes above
made from its template at its place, outside the words that run on: the two particle words, the life at `+0x20`
of an effect that has one (it counts down a tick, and -2 is an emitter killed, freed once it holds no
particle), the emitting countdown at `+0x66` and the box.

**A particle started with a direction** is aimed along the third row of its node's matrix, a unit long. Six of
the 61 are such records, and each holds the words the rule gives: the Jungle Spray's jet (template 37, speed 40)
on its node 2 is (0, 25, 30) with the Spray turned nought and (-30, 25, 0) turned 270, where the direction is
(-0.0113, 0.6414, 0.7672): -11 times 40 over 1024 is cut to nought, not down to -1. Its node 1 is (-34, 19, 0)
turned 270, and the Laughing Hyenas' node 5 (template 73, speed 31) is (0, 23, -19) turned 180.

**The place is in whole units.** An emitter a script's record names with a node is put where that node stands
on every sweep, each coordinate cut to a whole number first: all 61 hold multiples of 64. The
Drinks Shop's node stands at (440.15, 6.60, 312.02) and its bubbles at (440, 6, 312).

The live image's header holds what a start reads: the switch at `+0x08` that refuses an effect marked at `0xC1` (nought in every file),
the generator's seed at `+0x10`, the next emitter's count at `+0x18` (a word; the next effector's follows it) and
the particle density at `+0x20`: 500 in the shipped park and the nine files saved from it under Proton, 1500 in the eight played saves.
The game loads the whole header back, so a park runs at its file's density, not the options' graphics quality. The
chains' heads are 16-bit slots, -1 for an empty chain. An emitter is started in the slot at the head of the free
chain and put at the head of the used one.

Not every emitter in use is named by a script's record: 73 of the 128 in those files are not. A broken
bumper ride's car names its smoke itself ([a car's smoke](#the-track-rides-module-kart)). A script's one-shot
event keeps no record, so a park saved with a Loudspeaker standing holds the two puffs it had in the air
(template 36) with nothing naming them.

An item's own `.emt` files are templates, not emitters. They are read into the first unnamed template slots
as the park's catalogue is read, whatever stands in the park: a file saved with a Loudspeaker bought and
the file it was loaded from hold the same bytes in slots 101 and 102.

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
alone. Set `0x13`, the park door's, is things 3, 4, 5, 8 and 9 in the shipped park, and holds the analyser
(thing 5) in all 612 park files on hand; set `0x17`, the strike's, is thing 3, the advisor, alone, and holds the
analyser in none.

Four sets follow the thing list by model, in all ten park files: set `0xa` holds every guest (model 1), every
member of staff (models 4 to 8) and the one model-17 thing; `0xb` every catalogue object (model 3) and the
model-19 thing; `0xc` every member of staff and the things of models 9, 12, 13 and 16; `0x1b` every guard
(model 7) and the model-17 thing. The other sets hold only the singleton managers, the same in every file but
set `0x1c`, which holds the model-17 thing in the eight played files and nothing in the shipped park or the
Instant Action save. What each message does is OpenTPW's `docs/exe/ride-operation.md`, "The month's change".

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

## The vanilla time module (`TNAV`)

One dword, between the clock's `KOLC` tag and the module's own `TNAV`: a reading of the game's real-time clock
in milliseconds as the file was written (`FUN_00403220`, the clock `FUN_005f5f10`: the performance counter, or
`timeGetTime` without one). On loading the game keeps this reading less its own (`FUN_00403260`). In the eight played
files it is 12 ms to 28 s ahead of the clock module's second dword (`4,852,920` against `4,826,742` in a jungle
save); it is `114,938,044` in `Easymode.TPWI`.

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
the saved one left off. The fourth is not the count the records are walked by, which is the dword after the five
discarded ones: an Instant Action save written after a load holds 28 in the header over fourteen records (29 over
fifteen, with an arrival vehicle's script made since). A file
with the pointer set to `0xDEADBEEF` loads and runs (measured under Proton, 2026-10-08).

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
| 4 + n    | **Block 4 — limbo**, 8 bytes a slot: a visitor's handle (nought for a free slot) and the clock reading they are due back at |
| 4 + n    | **Block 5 — the bounce slots**, 16 bytes a slot: a visitor's handle, the node they are on, the clock reading they are due off at and the one they got on at |
| 4 + n×32 | **The walk slots** — a count, and that many 32-byte slots (below)       |
| 4 + n    | **The head table** — one dword per head slot (below)                    |
| 4 + n    | The script's directory, the string the game's loader keeps at `+0x38`   |
| 4 bytes  | Guard — `OBJ `                                                          |
| 4 bytes  | Object count                                                            |
| 4 bytes  | Bytes per object — 28                                                   |
| n        | **The object list**, the effects the script has started, newest first (below) |

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

Sixteen more, from the game's loader of a script (which makes the struct) and a count over the 374 records of six
park files:

| Offset | Description |
| ------ | ----------- |
| `0x00`, `0x04` | The list links, and `0x18` to `0x38` nine pointers to the blocks that follow, and `0xb0` the object list's head: addresses of the saving session, which the game replaces on loading |
| `0x0c`, `0x10`, `0x14` | Other scripts' handles, set on a few records in pairs (a ride's script and its companion's); nought otherwise |
| `0x4c` | The head slot count — the head table's length over four |
| `0x58`, `0x60` | The limbo slot count (block 4's length over eight) and how many are taken |
| `0x64` | The bounce slot count (block 5's length over sixteen) |
| `0x6c`, `0x6e` | Two words: how many are bouncing, and `BOUNCESETBASE`'s value (a Belly Bounce with six aboard reads `0x00080006`) |
| `0x70` | 1 on every record: the base node numbers are counted from |
| `0x74` | The name's offset in the string blob, `-1` until the script names itself |
| `0x7c` | The walk slot count |
| `0x88`, `0x8a` | Two words, 1000 each on every record |
| `0x8c` | The variable count — block 2's length over four |
| `0x90` | The string blob's length |
| `0x94` | A dword of the `.RSE` header, 50 on every record |
| `0x98` | `-1` on 369 of 374 records, nought on five; `-1` on a record the game has just made |
| `0xac` | 2 bytes: the id of the thing the script belongs to; nought for a script with none |
| `0xc0` | The speed word (below) |
| `0xc8` | **The model handle of the script's thing** — the object's `MeshInstanceID`, a slot of the ride system module plus one; nought for a script with no thing |

A record whose struct holds only what the loader sets (the handle, the counts, the call index at the stack size
less one, `0x70`, `0x74`, the two words at `0x88`, the looping key `0xffff`, the thing, the speed word 50, the
model handle, `-1` at `0xd8` and `0xffff03e8` at `0xe4`) with the counter at word 0 loads, and the script runs
from its start (measured under Proton, 2026-10-08).

**The records tie to the objects one to one.** Every object's `mRideScriptHandle` names a record that holds the
object's id at `0xac` and its `MeshInstanceID` at `0xc8`: 360 of 360 across the six files. Of the other fourteen
records six share a thing and its model with that thing's own script, and eight have no thing.

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
| `0x14` | 2 | Facing, one of eight ways, 0-7 (below) |
| `0x16` | 2 | Action |
| `0x18` | 2 | State: 0 free, 1 walking on, 2 on the ride, 3 walking off, 4 off |
| `0x1a` | 2 | Flags |
| `0x1c` | 4 | Not decoded; nought in every slot counted |

**The facing** is which of eight ways the walker is drawn. Along z is 0 and the ways count on towards x: for a
direction on the ground (x, z), `trunc( 10.5 − 4·atan2( z, x )/π ) mod 8`. A slot walking on (state 1) holds the way
from its walk node to its head node, and one walking off or off (3, 4) the way from its walk-off pair's first node to
its second, in the world, so it turns with the ride. A slot on the ride (state 2) keeps the walk's facing where its
action is 1, 2 or 4; under any other action (5 and 6, the sideshows') it holds the way its head node points, the third
row of the node's matrix. A Jungle Spray turned 0 holds 7, 0 and 1 walking on to its three lanes, 0 standing in one,
and 3, 4 and 5 walking off; turned 270 it holds 6 standing. All 100 slots that hold a walk in thirteen files of the
game's own follow it: 36 in use and 64 let go, which keep their walk off's.

A slot no walk has used is 32 nought bytes. A slot let go keeps everything but its state and handle, which are
nought: in thirteen files of the game's own, 281 slots are 181 never used, 36 in use and 64 let go, and each of the 64
holds its four nodes, its action, flags of 1, and a due past its start and at or before the file's clock. So due less
start is the last leg walked, the walk off's (how
long one lasts: OpenTPW's `docs/exe/ride-operation.md`, "How long a leg lasts, and where its ends are"). The shipped
park's sideshow saves its three slots all nought. In Alexah's Lost Kingdom autosave the Jungle Spray's read 700 and
1,100, the Steak Shop's 600, the Inca God's 800; a slot on the ride saves its start at or just past its due,
restamped on arrival.

**The object list** is what the script's `ADDOBJ`s have started and no `KILLOBJ` has stopped (the
[Instruction Set](/vm/instructions/)), each record copied raw from the game's linked list, 28 bytes, from the head
of the list: the newest first.

| Offset | Size | Description |
| ------ | ---- | ----------- |
| `0x00` | 4 | Next record, an address; the game replaces it on loading |
| `0x04` | 4 | Previous record, the same |
| `0x08` | 4 | Type: 1 and 2 a particle, 3 to 10 a sound |
| `0x0c` | 4 | The handle the particle or sound system gave for it; never nought |
| `0x10` | 4 | The model node asked for, by id; -1 for none |
| `0x14` | 4 | That node's lookup record in the model (the [Models](/formats/models/) page's node lookup ids), found among the particle nodes (`0x100`) for types 1 and 2 and the sound nodes (`0x200`) for the rest; -1 for none |
| `0x18` | 4 | The tag `KILLOBJ` matches |

The game links each record in at the head of the list as it reads it, so a loaded list stands the other way round
from its file. It starts nothing on loading: a record keeps its handle, and the same handle goes out in the next
save (the Drinks Shop's smoke is `0x7a0014` in the shipped Lost Kingdom park and in three files saved from it). The
effect's own id is not kept. Thirteen files of the game's own hold 140 records in 100 of their 440 scripts: a litter
bin one sound with no node (type 5, tag 10), a Gift Shop three smokes on nodes 3, 2 and 1 (records 0, 1 and 2), a
dirty Small Toilet two particles on node 1.

**A bounce slot** is taken from the first free one, and the node its rider is on is the slot's index plus the base
at `0x70`: slots 0 to 6 on nodes 1 to 7 in a played park. A rider's due is their start plus the ride's duration
(30,000 ms on a Belly Bounce at 30 seconds). **A slot let go keeps its node and its two readings**, with a handle of
nought: a Belly Bounce the game saved with three aboard holds two such slots, due 43 and 61 seconds before the
file's clock. **A slot never used is 16 nought bytes.** Counted in thirteen park files: seven scripts hold bounce
slots, 70 in all; 25 are in use, 37 are all nought, and all 8 let go hold a handle of nought, the node of their
slot, and a due 30,000 ms past its start and at or before the file's clock. The count at `0x6c` is the slots in
use in all seven. The rider is a
guest in state 16 whose `mMajorDest` is the script's thing, out of the queue (no links), standing where they got on,
with their sprite on animation program 66 and picture set 2 (nine of nine bounce riders in two files).

**A record the game has just made** (a Crazy Ape's, bought 111 ticks before the save, the ride closed for want
of a queue; measured under Proton, 2026-10-08) is first in the module under the header's next handle. Its struct
holds the fields the loader sets, its name's offset (0: the script had named itself), `-1` at `0x98`, its thing
and its thing's model handle, and the object's operating speed, 60, as the speed word. Its stack block is twenty
dwords of `0xFFFFFFFF` but for what has been pushed, its head table sixteen noughts, its object list empty, and
its directory `data\levels\jungle\Rides\monkey\`. A record written with the folder in lower case
(`rides`) loads.

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
| 4 bytes | Empty slot count — 7 there, so 168 slots                         |
| 4 bytes | The cursor: the slot the next model made will take. The lowest empty slot (90 there, slots 90 to 96 being the empty ones), or the slot count where none is empty, in all six park files read |
| n       | One record per slot, present and empty together                  |

A slot that holds **nothing costs exactly one byte** — the tag alone — which is how a module with far
more slots than things stays small. A present record is:

| Offset   | Size     | Description                                                     |
| -------- | -------- | --------------------------------------------------------------- |
| `0x00`   | 1 byte   | Tag — `01` for present                                          |
| `0x01`   | 4 bytes  | The **item** id, not a thing id                                 |
| `0x05`   | 4 bytes  | The cell it stands on, across: an object's `mTopLeft` cell; a fixed item's (flags `0x361`) is its description's `Info.EngineMapOffsetOverrideX` and `Y` (the gates' 45 and 16), or 48 and 17 where it sets none, as the lights and all three arrival vehicles read in every Lost Kingdom file, whose objects stand on cell (0,0) |
| `0x09`   | 4 bytes  | The cell, down                                                  |
| `0x0d`   | 4 bytes  | Its footprint in cells, across                                  |
| `0x11`   | 4 bytes  | Its footprint, down                                             |
| `0x15`   | 4 bytes  | The flags it was made with: `0x32F` a placed object, `0x361` a fixed one (gates, traffic lights, an arrival vehicle), `0x33A` a piece of queue |
| `0x19`   | 4 bytes  | **The script handle of the thing it is the model of** - the object's `mRideScriptHandle`; nought for a piece of queue and other scenery |
| `0x1d`   | 4 bytes  | Packed model flags; low seven bits are the hoarding state below |
| `0x21`   | 2 bytes  | Not described; nought on every object's record in six park files |
| `0x27`   | 4 bytes  | The angle in degrees: an object's `mAngle`; a piece of queue's is 360 less its cell's tile angle |
| `0x23`   | 4 bytes  | Hoarding progress, little-endian float (`0` retracted, `1` closing endpoint) |
| `0x2b`   | 2 bytes  | Node flag-word count                                            |
| `0x2d`   | 2 bytes  | The model's node-lookup record count                            |
| `0x2f`   | 4 bytes  | The lookup records' shared flags: `0x1` some record has a position, `0x4` something is attached, `0x8` some record's file flags carry `0x2` (walkable meshes); `0x2` is set by the routine that hides a model's head nodes. The Jungle Spray's reads `0x1`, a ridden Aztec Mayhem's `0x7` |
| `0x33`   | 4 bytes  | How many things are attached: the count of lookup records below that carry `0x2` |
| `0x37`   | n×8      | Per lookup record: its runtime flags, and the slot in the sprite table of the head attached to it (-1 for nothing; a record without a position starts at nought) |
| —        | n×4      | One flag word per node; bit `0x1` is hidden (the table below)   |
| —        | n×44     | The animation channels — see below                              |

Everything from `0x01` on is **unaligned**, because the one-byte tag leads.

**Hoarding flags:** `0x01` active, `0x02` raising, `0x04` lowering; `0x08` selects Closed,
`0x10` Hoarding (broken), `0x20` Condemn and `0x40` Upgrade. Bits above `0x40` belong to other
model state. The restore routine (`0x004647a0`) expands these bits to runtime model flags
`0x20` through `0x800`, and restores progress through `0x004547f0`. Progress alone does not say
whether the panels are active or moving. The closing endpoint does not guarantee equal full height
for every panel: the executable staggers their deformation.

`Easymode.TPWI` holds `0x08` on the Belly Bounce's record alone and nought on the others. A park the game loaded
and saved again holds `0x08` on the records of all six objects a guest may be offered (the ride, the sideshow, the
shop and the three toilets) and nought on the other eight, where the file it had loaded held nought on five of the
six (measured under Proton, 2026-10-08).

The saved placed object's model handle (`MeshInstanceID`) selects **slot index plus one**, not the item id; repeated
instances of an item can have different hoarding states. The record names the object back by the script handle at
`0x19`. Across six park files (the shipped Lost Kingdom park and five played saves of three themes) every one of
the 360 objects names a present slot whose record holds its item id, its script handle and its angle, and each of
the 337 placed ones its `mTopLeft` cell.

**Buying a thing takes the slot at the cursor and selling one empties its slot.** A park whose table read 161
present, 7 empty, cursor 90 was saved after a Crazy Ape was bought: 163, 5, 92, the ape in slot 90 (handle 91)
and the queue piece the game lays before its entrance in slot 91. Saved again after the ape was sold, with two
arrival vehicles made in between: 163, 5, 90, slots 90 and 91 empty again and the vehicles in 92 and 93. A made
record holds the placer's flags `0x32F`, the object's script handle, hoarding `0x9` at progress 1.0 (a thing
with a queue is built closed), and one channel on role 0, held (flags `0xC`) at speed 1.1 (measured under
Proton, 2026-10-08).

**A queue cell's `mMeshInstance` is a slot of this module too**, plus one. Its record's item id is 17000 plus the
cell's tile index (17001 to 17007), its cell is the cell, and its script handle is nought. A queue cell holding
nought there is drawn black by the game, piece and ground both, and still works as a queue (measured under Proton,
2026-10-08). The other records are named by a cell's track sub-record: most of them, items 173xx (143 of the
shipped park's 161), are the boundary's pieces, below; 171xx in a played park are named by cells of track type 11.

### A boundary piece's record

**The wall between the park and the land not bought is one model to an anchor**, a cell whose track sub-record's
`mType` is 25 (the three other cells of its two by two hold 12 and name it as their `mParentID`). The anchor's track
sub-record names its piece and says which it is:

| Track sub-record | Value |
|---|---|
| `mMeshInstance` | the piece's slot in this module, plus one |
| `mTileData` | set 5; index the piece's row of the game's hoarding table; angle its turn, 0, 90, 180 or 270 |
| `mType` | 25 |
| `mHoardingNeighbours` | which of the four cells two away are anchors too: `0x01` at y − 2, `0x04` at x + 2, `0x10` at y + 2, `0x40` at x − 2 (520 of 520 anchors in four parks; the gate's two ends name the one side they have) |
| `mDirection` | in the same bits, the way the boundary was laid leaving the anchor, or for the last anchor laid the way it came (the 107 anchors of each of 34 fresh Lost Kingdom parks, in the order of their pieces' slots) |
| `mOverlapCounter` | 1 on a corner, nought on a straight and on the ends: a clicked point of the laid boundary ends one run and starts the next, so its anchor is built over once (1,819 of 1,819 anchors in the seventeen fresh Lost Kingdom files, whose corners are the masks of two perpendicular sides but `0x11` and `0x44`) |
| `mFlags` | 1 on an anchor over water or hidden ground, 0 on every other in the files to hand: a straight's piece is then row 11 (`ho_fos4`), a corner's row 26 (`ho_fob3`; one, in Alexah's played jungle park, on land bought later); the fresh parks flag straights alone (one in Lost Kingdom, at (59,39)) |

and the piece's record holds:

| Offset | Value |
|---|---|
| `0x01` | 17300 plus the nearest row at or above the piece's that the game's table names: a row the table names no model for draws the row before's, whose item it is |
| `0x05`, `0x09` | the anchor, moved one on in x for a turn of 90, in x and y for 180, in y for 270 (by the angle at `0x27`: y for 90, x for 270) |
| `0x0d`, `0x11` | 2 and 2 |
| `0x15` | `0x33A` |
| `0x27` | 360 less the turn, 360 folded to 0 |

The items the table can make and the model each draws, from the theme's `hoarding.wad` (all four ship the same nine
names): 17300, 17302 and 17304 `ho_fos1` (17302 the straight, 17304 for rows 4 to 8), 17301 `ho_end1`, 17303 `ho_fob1`
(a corner), 17309, 17312 and 17315 (each for its row and the two after it) and 17318 `ho_fos4`, 17319 `ho_fob2` (a corner),
17320 and 17323 `ho_fos2` and `ho_fos3` (a straight's two variants, each for its row and the two after it), 17326
`ho_fob3`, 17327 `ho_fob4`. The files hold 17302, 17303, 17309, 17319, 17320, 17323 and 17326. **Measured on every park file to hand**, seven files (four distinct parks, three
themes: the shipped Lost Kingdom park, Alexah's Full Simulation saves of jungle, fantasy and Hallowe'en): 933 anchors (520 in the four distinct parks),
each naming a record of its own whose item, cell and angle are exactly these. The shipped park's 143: 90 `ho_fos1`,
19 `ho_fos2`, 13 `ho_fos3`, 12 `ho_fob1`, 8 `ho_fob2` and one `ho_fos4`. What the executable does with them is
OpenTPW's `docs/exe/park.md`, "The boundary hoarding's piece".

**A fresh park's pieces are slotted in the order the boundary was laid**: in each of 34 fresh Lost Kingdom parks the
original wrote (Alexah's `restart.INTS` and 33 made under Proton), slot 3 is the anchor at (51,17), three cells right of
the gate, and slot 109 the one at (43,17) on its left, the walk round the park between them; their 107 anchors' masks,
directions, flags and the rows and turns of their tile data are the same in all 34 but for which straights take the
variant rows 22 and 25, which differ from park to park.

### A kart or water track piece's record

**A kart or water ride's track is one model to a piece of two by two cells**, as the boundary's is. The piece's top-left
cell, its anchor, carries a track sub-record of `mType` 11 (the karts) or 16 (the water rides), the station's two pieces
13 or 18, and the three other cells of its two by two 12, each naming the anchor as its `mParentID` and holding its
overlap counter; in the file to hand 16 of the 99 also hold `mTileData` (0, 0x37, 0), and the rest nought. A cell whose
piece was taken back (Backspace, or a sale) holds type 0, no links, no direction, no parent and `mTileData` set 0, index
0x37, angle 0, the row the executable gives a cell of no track (`Track_ClearCell` then `Track_RetileCell`), and so do
its two by two's cells. The file to hand holds karts alone; that a water piece's other cells are 12 as well, not 17, is the
executable's (its laying stamps them 12 for both kits, and nothing in its track code was found writing 17), not a file's. An anchor's track sub-record:

| Track sub-record | Value |
|---|---|
| `mMeshInstance` | the piece's slot in this module, plus one; nought on a station's anchor and on a cell with an add-on |
| `mTileData` | set 3 (karts) or 4 (water); index the piece's row of the ride's kit; angle its turn, 0, 90, 180 or 270. A station's anchor holds 0, 0, 0 |
| `mParentID` | the packed cell of the ride the track is laid for (its catalogue object's anchor) |
| `mNeighbours` (`0x07`) | which of the four anchors two cells away the track runs on to: `0x01` at y − 2, `0x04` at x + 2, `0x10` at y + 2, `0x40` at x − 2 |
| `mDirection` | in the same bits, the way the last run laid over the cell left it |
| `mFlags` | `0x10` where the cell carries an add-on, whose model is not the kit's piece; the low four bits a hill (read from the executable; no hill in the file to hand) |
| `mOverlapCounter` (`0x08`) | how many runs were laid over the piece after its own (0 or 1 in the file to hand); the cells of its two by two hold the same |

and the piece's record holds item 17100 plus the nearest row at or above its own that the kit names (17200 on for the
water ride's kit), on the anchor moved one on in x for a turn of 90, in x and y for 180 and in y for 270, at the angle 360
less the turn: the boundary's rule. The rows, the models they draw from the theme's `rides\gokarts.wad` and
`rides\wateride.wad`, and the end caps each row hides are OpenTPW's `docs/exe/park.md`, "A track piece's model, price and
open end" ([Models](/formats/models/#end-caps)). **Measured** on the one park file to hand that holds such a track, Alexah's played
Full Simulation Lost Kingdom (two saves of it): 33 anchors, the station's two at turn 180 (`mDirection` `0x40`,
`mNeighbours` `0x44`) and 31 kart pieces, 28 of them with a record (15 of item 17102, 10 of 17103, 2 of 17119 and one of
17104), each on its anchor moved by its turn at 360 less it; the three others carry `mFlags` `0x10` and no record. The
water ride's records are read from the executable, not from a file.

### A queue piece's record

Every queue piece's record is the same bytes but for four fields, so a writer can make one from the cell alone.
In nine park files (the shipped Lost Kingdom park, five played saves of three themes, one of them with no queue, and three saved under Proton)
all 225 queue cells name a record that reads exactly:

| Offset | Size | Value |
|---|---|---|
| `0x00` | 1 | 1, the slot is present |
| `0x01` | 4 | 17000 plus the cell's tile index |
| `0x05`, `0x09` | 4, 4 | the cell's x and y |
| `0x0d`, `0x11` | 4, 4 | 1 and 1, one cell square |
| `0x15` | 4 | `0x33A`, the flags a queue piece is made with |
| `0x19` to `0x26` | 14 | nought: no script, no hoarding |
| `0x27` | 4 | 360 less the tile's angle, 360 folded to 0 |
| `0x2b` | 2 | the node flag words that follow: 2, or 4 for tile indices 5, 6 and 7 |
| `0x2d` to `0x36` | 10 | nought: no lookup records |
| `0x37` | 4 a word | the node flag words, all nought |
| then | 44 | one channel: role 12 (none) at its second dword and at its eighth, every other dword nought |

Tile index 1 is the dead end, the piece of a queue that has not reached a path (one record, in a park saved just
after a ride was bought); 2 a straight, 3 and 4 the bends, 5 the end at a path, 6 and 7 the bins. Index 0 names
the same model as index 1 in the game's table and is in no file.

**A cell that leaves the queue gives its slot up, and a cell tiled again takes another.** A park saved by OpenTPW
with two queue cells cleared, one tiled again as a bend and two laid, their records made as above in the lowest
empty slots, loaded in the game with the queue fenced along its new cells and bare grass where the old ones were
(measured under Proton, 2026-10-09). The same park saved with the two new cells naming no model drew them black;
the bend's cell, still naming the straight's record, drew a straight; and the cleared cell, still naming its old
record, kept its piece standing on bare ground.

**The game makes each model again from its record when it loads**, in the slot the record lies in, and then lays
the saved state over it. It compares the two counts at `0x2b` and `0x2d` with the fresh model's first: where either
differs it steps over both tables by the file's counts and keeps the fresh model's own. So a record may declare no
nodes and no lookup records. A file made that way, with one idle channel, loaded and its model stood and animated
(measured under Proton, 2026-10-08). Q91b checks this pairing for all eleven
placed objects in the shipped Jungle park. Its synthetic record checks the unaligned flag and
progress reads independently of normal zero-progress saves.


### The node flag words

A node's word holds eleven bits of the game's own flags for that node, packed. In the 45 park files to hand that
the game wrote (7,688 records of 95 items) ten of them appear:

| Bit | What it says |
|---|---|
| `0x1` | The node is hidden |
| `0x2` | Every child of the node is a transform-only node with no child of its own: a mesh that holds only markers, as the Crazy Ape's two arms hold its head and nose nodes |
| `0x4` | Not described; set in no record |
| `0x8` | The node's mesh has been morphed and its face normals are out of date; the game works them out again when it next poses the mesh |
| `0x20` | A clip bound to one of the model's channels has a track for this node |
| `0x40` | Set as the model's file is loaded, on one node of seven track and scenery items (62 words); held too by the first node of a track car's model and of its wake's, on all 24 measured ("A car's two model records", below), where what sets it is not traced; not described further |
| `0x80` | That track is a morph (its flags hold `0x1000`) |
| `0x100` | That track moves the mesh's texture coordinates (`0x10000`) |
| `0x800` | That track places, turns, scales or routes the node (any of `0x289`) |
| `0x200` | The node's [lookup record](/formats/models/#node-lookup-ids) has a position: its flags in the model's file meet `0x30` |
| `0x400` | The node is transform-only (its flags in the model's file hold `0x200`). A clip's hide list neither hides nor shows it; a visibility track still does |

**A word can be worked out from the model's file and the clips its channels have run.** A model just made holds
`0x400` and `0x1` on every transform-only node, and `0x200` and `0x2` where the rows above say. A clip bound to a
channel sets `0x20` on each node it has a track for, with `0x80`, `0x100` and `0x800` by the track's flags, and
`0x1` on each node of its [hide list](/formats/models/#the-hide-list) but a transform-only one. The next clip
bound to that channel first takes those off: the track bits of the old clip's tracks, and `0x1` of its tracks'
and its list's nodes, a transform-only node's apart. A clip that merely ends takes nothing off, so an idle
channel's last clip still marks its nodes: an idle Staff Room keeps `0xa2` on the mesh its build clip morphed. A
[visibility](/formats/models/#visibility-bit-0x20000) track sets and clears `0x1` as the clip plays, on a
transform-only node too, and there what it left stays after its clip is replaced: the Crazy Ape's `Dummy01`, the
parent of its arms, is shown by the build clip and reads `0x420` under it and `0xc20` under the ride clip.

Of the 870 records of an item a player can buy or a park places by name, 836 read exactly so, bit `0x8` apart.
In 30 of the others a transform-only node was shown or hidden by a visibility track of an earlier clip the record
does not name; in 4 a running clip's tracks are not marked at all, and in 2 an idle shop holds no build marks. A
Crazy Ape just bought reads `0 a0 a0 20 a0 a2 a2 a1 a1`, then `601` for its 24 markers, then `420`; one running
its ride clip `0 0 0 0 a0 a2 a2 1 1`, `601` 24 times, `c20`.

**A record written with no tables keeps its build clip's marks.** The game makes the model with its build clip
bound, so a record declaring no node words leaves those marks on it through every later clip: a Crazy Ape saved
running, with no words, read `0 a0 a0 20 a0 a2 a2 a1 a1`, `601` 24 times, `c21` in the game's memory after the
load and went on reading so, where the same file with its words read `0 0 0 0 a0 a2 a2 1 1 .. c20` (measured under
Proton, 2026-10-09). The two looked the same on screen from the saved camera; what the leftover marks change in the
picture is not known.

**The words follow from the item and what its channels play.** Grouped by item and by each channel's role, clip
and loop, frozen and held bits, the 7,688 records make 127 groups: in 113 of them every record's words are the
same, and in the other fourteen they differ in `0x8` alone, but for one word of item 1181's in `0x400`. On loading,
the game lays each word over the fresh model's node and does not work the track bits out again, so they are what
tell it which nodes the running clips move until a clip is next started.

### A head on a node

A rider's head showing on a ride (on a Crazy Ape's tentacle, in a coaster's car) is a sprite of its own in the
[sprite table](#the-sprite-table-tpcs), and the ride's model record names it: the lookup record of the node it
hangs on carries `0x2` in its flags and the sprite's slot beside them, the shared flags at `0x2f` carry `0x4`, and
`0x33` counts the heads. In four played parks (two saves each of a jungle and a fantasy one) 122 lookup records
carry `0x2`, on fifteen items, and every one names a live sprite that no person's record names; in every record
`0x33` is the count of them, and `0x4` is set exactly where there is one.

A head's sprite is kind 1 (a child's head, `0xB0` the child) or kind 3 (a costume's head, `0xB0` the costume), and
all 122 read alike elsewhere: `0x08` 1698, `0x0C` 1704, `0x10` 1696, state 2, `19` at `0x1C` with word 1714 in the
loop stack's last place, `1` at `0x74`, set 0, `8` at `0xBC`, `0x3000080` at `0xC4` and `1` at `0x114`. They are
every sprite of kinds 1 and 3 in the four files. `0x0C` is the word the sprite was made on and stays; `0x74` is
`1` while the loop count at `0x78` is nought, so the word in the stack is not a loop's start. Of the fields that
differ from one head to the next, only the slot at `0x04`, the kind and `0xB0` are the head's own. The place
(`0x88` to `0x90`), the scale (`0xA4`, `0xA8`) and the frame (`0xB8`) are nought, `1.0` and nought on every head
of a ride whose script keeps a head table, and are set on the heads of the rides that keep none; `0xD0` to
`0xDC`, `0xE4` to `0xEC`, `0xFC` and `0x100` repeat what the sprite was last drawn with, and `0xF8` and `0x110`
are pointers of the run that wrote the file. A file whose head sprites hold nought in all of those loads, and the
game shows the heads and takes them off as its own. The node's own flag word is the same with a head on it and
without. A ride's script keeps a table of the same riders by head node ([the ride script module](#the-ride-script-module-essr)),
in another order; some rides hang heads and keep no such table (a coaster's cars, the Aztec Mayhem).

**The Aztec Mayhem's heads are named by its script's walk slots ([the ride script module](#the-ride-script-module-essr))**: a slot whose action is 4 and
whose state is 2, a rider carried, has a head on the lookup record of its head node, the slot's second word looked
up among the model's head nodes. A played park's Mayhem holds five such slots, on head nodes 1 to 5, and its model
record holds `0x2b` and a sprite on records 5, 7, 8, 9 and 12, which the model's file gives those five ids; no
other slot in use in that file has action 4. The two must agree: loading a file with the slot carried and no head on
the record, the game takes one off the count at `0x33` as each such rider walks off, below nought, and the count
stays short from then on.

**A lookup record's runtime flags** say whether the game keeps its node's position
([Models](/formats/models/#which-records-have-a-position)): `0x1` it has one, `0x20` its node has no children, `0x8`
it is posed all the same, `0x2` something is attached to it, `0x4` the ride view is on it, `0x10` its node carries
`0x400`. The shipped park's Jungle Spray saves `0x29` on eleven records and `0x21` on its `camera`, which is posed only
while the ride view is on it; a played park's Aztec Mayhem saves `0x29` on 34 of its 39 records and `0x2b`, with a
handle, on the five heads carrying a rider: all 39 carry `0x8`.

Over the same 870 records, with nothing attached, every pair follows from the model's file: `0x1` where the
record's flags there meet `0x30`, `0x8` where they meet `0x580f00` or the item's description sets
`Info.DoHeadProcessing`, `0x10` where the node's own flags hold `0x400`, `0x20` where the node has no child; the
handle is `-1` for a record with a position and nought for one without. The shared flags at `0x2f` are `0x1`
where some record has a position, `0x8` where some record's flags hold `0x2`, and `0x2` on every record of a ride
with head nodes (lookup flag `0x80`), bought or shipped, but not on an arrival vehicle's (the ferry's four records).

**A channel is 11 dwords**, in the order the game copies them back onto the running channel:

| Dword | Description                                                                        |
| ----- | ----------------------------------------------------------------------------------- |
| 0     | Flags — `0x1` loop, `0x2` frozen on frame nought, `0x4` held on the last frame, `0x8` a clip of role 0 has run past its end on this channel |
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
> lay the rest pose down, `0x8` do not apply the hide list. Stored on the channel, `0x1` means
> the same, but `0x2` means **frozen at frame nought**, `0x4` means **held on the last frame** and `0x8`
> that **a clip of role 0 has run past its end** —
> states the engine has recorded, not requests. On loading, the game copies the word back as it stands
> and adds `0x10` to a held channel (OpenTPW's `docs/exe/ride-operation.md`). Feeding the stored word back
> in as caller flags therefore drops the held pose — in the shipped park that is **eleven of the fifteen**
> channels that hold a real role.

**Bit `0x8` marks a role 0 clip that has ended.** The game sets it as a channel on role 0 (an item's `C` clips,
the one a thing plays as it is built) runs past its clip's end, keeps it through the hold or the loop that
follows, and takes it off at the first advance of the channel on any other role. In thirteen park files the game
wrote (2,882 channels) the 12 channels held on a role 0 clip all carry it, and no channel on another role and no
idle channel does: a Litter Bin and a newly bought Crazy Ape read `0xc`. A writer sets it on a channel held or
looping on role 0 whose clip has ended, and on no other.

**Bit `0x10` comes and goes on a held channel.** The files hold a held channel as `0x4` or as `0x14` (`0xc` and
`0x1c` on role 0): 8 and 2 in `Easymode.TPWI`, 27 and 28 in a played jungle park, and the two with
`0x14` in the shipped park are the security cameras, whose clips ended less than a second before the save. A file
whose nine held channels all carried `0x10` was loaded by the game and saved by it four minutes on: the three still
held from before the load (the gates, a lane of the Jungle Spray, the litter bin) came back `0x4` or `0xc`, as did
two lanes whose clips ended 47 and 25 seconds before the save; a toilet's, ended 5 seconds before it, came back
`0x14`; and channels running a clip started after a hold carried `0x10` alone (measured under Proton, 2026-10-08).
So a held channel's `0x10` is not kept for good, and a reader must take both forms as held.

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

> Three Small Toilets are three records of one **item** id. What tells them apart is the script handle at
> `0x19` and the slot each lies in, which the toilet's own record names: scripts 11, 12 and 13 in slots 120, 121
> and 122 of this file.

The channel layout and the counts in this section are measured from `Easymode.TPWI`, the one file of this shape
that ships; the header, the fields at `0x05` to `0x19` and the angle are measured over the six park files named
above.

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
| 3    | 52   | One ride (below)                                                                    |
| 4    | 28   | One track section: after the header the ride's handle, the section type, x and y   |
| 5    | 208  | One car (below)                                                                     |
| 9    | 24   | One rider of the car before it: the ride's handle, the rider's thing id, the seat's id |
| 7    | 20   | One person who has come off and not yet been let go: the ride's handle, their thing id |
| 8    | 20   | One person waiting to board the next car: the ride's handle, their thing id        |
| 6    | 16   | Closes a ride: its handle                                                           |

A ride's chunks are together and in that order: the ride, its sections, each car followed by its riders,
the two lists, the close. The rides are in the order of their slots.

**A ride** (type 3), after the header:

| Offset | Size    | Description                                                                 |
| ------ | ------- | --------------------------------------------------------------------------- |
| 12     | 4 bytes | The handle                                                                  |
| 16     | 8 bytes | Where it was placed, x then y, in map cells times `0xc00`                   |
| 24     | 4 bytes | The orientation code: 0, 5, 6 and 1 for a turn of 0, 90, 180 and 270 degrees |
| 28     | 4 bytes | The item's id                                                               |
| 32     | 4 bytes | Performance, 0 to 100: the ride's operating speed                           |
| 36     | 4 bytes | The first of the item's supplemental meshes a car is drawn as               |
| 40     | 4 bytes | How many of them the cars cycle through                                     |
| 44     | 4 bytes | Duration: what a car's timer is set to, in ticks of 31 ms                   |
| 48     | 4 bytes | State: 0 closed, 1 loading, 2 running                                       |

**A car** (type 5), after the header:

| Offset | Size      | Description                                                               |
| ------ | --------- | ------------------------------------------------------------------------- |
| 12     | 4 bytes   | The ride's handle                                                         |
| 16     | 172 bytes | The car's record as the game held it in memory, its addresses with it    |
| 188    | 8 bytes   | The centre of the arena or section it was in, x then y                   |
| 196    | 4 bytes   | The handle of the ride whose buoy it steers at, nought with none         |
| 200    | 8 bytes   | That buoy's place, x then y                                              |

Of the car's record:

| Offset | Size    | Description                                                                              |
| ------ | ------- | ---------------------------------------------------------------------------------------- |
| `0x00` | 4 bytes | Flags: `0x1` in use, `0x4` chasing a car, `0x8` steering at a buoy, `0x20` unloading, `0x4000` driven, `0x80000` riders being seated, `0x400000` the ride's lead car, `0x1000000` bobs, `0x2000000` has a wake |
| `0x04` | 4 bytes | The supplemental mesh it is drawn as                                                     |
| `0x08` | 4 bytes | **The model table handle of its own model** (a slot of the ride system module, with a record there) |
| `0x0c` | 4 bytes | And of its wake's                                                                        |
| `0x10` | 4 bytes | The clip its model plays: 5 in a go, 12 for none                                         |
| `0x20` | 4 bytes | The handle of the sound it holds, of the session that saved it; nought with none        |
| `0x24` | 4 bytes | The lookup record of its model's emitter node of id 2, -1 before it is placed: 3 for a Hot Pot boat |
| `0x28` | 4 bytes | And of id 1: 2 for a Hot Pot boat                                                        |
| `0x2c` | 4 bytes | The handle of its smoke's emitter in [the particles module](#a-live-emitter-and-its-handle), `count << 16 \| slot`; -1 with none |
| `0x30` | 4 bytes | An address: its rider list                                                               |
| `0x34` | 8 bytes | Where it floats, x then y                                                                |
| `0x3c` | 8 bytes | Its velocity, a tick                                                                     |
| `0x44` | 8 bytes | Its velocity after the last tick's friction, before any bump                             |
| `0x4c` | 4 bytes | Its speed                                                                                |
| `0x50` | 4 bytes | The heading it steers along, in 512ths of a turn                                         |
| `0x54` | 4 bytes | The heading it is drawn at                                                               |
| `0x58` | 4 bytes | The heading the drawn one turns to: nought until the ride's first go, then as the last tick of a go left it; for a water ride's car the word is its spin instead |
| `0x5c` | 4 bytes | How far the drawn heading turned in the last tick                                        |
| `0x60` | 4 bytes | How far that turn changed in the last tick: nought for a bumper ride's car               |
| `0x64` | 4 bytes | Its radius                                                                               |
| `0x68` | 4 bytes | A bonus to its thrust: nought for a bumper ride's car                                    |
| `0x6c` | 8 bytes | The point it steers at                                                                   |
| `0x74` | 8 bytes | Its offset from its target                                                               |
| `0x7c` | 4 bytes | Its buoy's id, which is the buoy's place in its ride's ring; -1 with none                |
| `0x80` | 4 bytes | The pool index of the car it chases, -1 with none                                        |
| `0x84` | 4 bytes | Nought: the game reads and writes nothing here                                           |
| `0x88` | 4 bytes | The ticks left in its go                                                                 |
| `0x8c` | 4 bytes | Buoys, or ticks of a chase, left before it chooses a new target                          |
| `0x90` | 4 bytes | Ticks since it was launched: the phase of its bob                                        |
| `0x98` | 4 bytes | An address: its arena                                                                    |
| `0x9c` | 4 bytes | An address: its ride                                                                     |
| `0xa0` | 4 bytes | A float: the height it was last drawn at, the water's plus its bob for a boat            |
| `0xa4` | 4 bytes | Its stuck count: -1 for a bumper ride's car                                              |
| `0xa8` | 4 bytes | The ticks counted while stuck: nought for a bumper ride's car                            |

The words at `0x14`, `0x18` and `0x1c` are not decoded; they are nought in every car of the files looked at.
**The game's loader keeps every word but these**: the three addresses and `0x94` are made again, the arena
from where the car floats (or from the chunk's centre where nothing holds that point); `0x7c` is found again
from the chunk's buoy place, -1 where the chunk names no ride and nought where no buoy of the named ride lies
there; and `0x8c` is set to nought on a car that is chasing. **A rider** (type 9) goes on the car its ride
read last. The riders of a car and the people of each list are written head first and each is put back on
its list's head, so each list comes back turned round.

In a Hot Pot's go saved by the original, four boats and a rider in each, the first car reads flags
`0x340c009`, mesh 1, models 94 and 95, clip 5, at (`0x20b9f`, `0x12fad`) with velocity (-134, -94), headings
345 and 336, radius `0x300`, buoy 5, timer 319 of the ride's 750, and its chunk ends on the arena's centre
(`0x20a00`, `0x13200`), the ride's own handle and the buoy's place (`0x20187`, `0x12987`).

**A car's smoke.** While a bumper ride is broken, each car that was driven as it broke (flag `0x4000`) holds
at `0x2c` the handle of an emitter of template 2 in the particles module, started where its emitter node of id 2
stood. The car's step puts the emitter there again every track tick, each coordinate times 1024, cut to a
whole number and shifted down four bits, so **its place is in 64ths of a unit**, not the whole units of an
emitter a script's record names. In a Hot Pot broken in a go and saved by the original seven seconds later,
the four boats hold `0x1400001`, `0x1410008`, `0x1420005` and `0x143000c`: emitters 1, 8, 5 and 12 under the
counts 320 to 323, each template 2 with a life of nought, the first at (28330, 1949, 16855) for a node at
(442.6575, 30.4622, 263.3747). The pot's own two smokes, a script's records of template 16, were started
after them (324 and 325). A file saved after the ride was fixed holds -1 in each car and the four emitters
killed, a life of -2, until their last particle is gone. The ride's own word for broken is not in the file:
its script sets it again on its first turn after a load.

**A car's two model records.** The handles at `0x08` and `0x0c` each name a record of the ride system module,
made as the car is launched and written with it. In the three files of the original's with a Hot Pot's boats
out, twelve boats and twelve wakes:

| Field | The boat's | The wake's |
| ----- | ---------- | ---------- |
| Item | 1142: the ride's item (1140) plus one plus the car's mesh (1) | 1141: mesh nought |
| Cell and footprint | nought, and none | the same |
| Flags (`0x15`) | `0x101` | `0x101` |
| Script handle, angle | nought | nought |
| Node words | six: `0x40`, nought, then `0x601` four times; the second reads `0x820` while clip 5 plays | one: `0x40` |
| Lookup records | four: flags `0x21`, `0x21`, `0x29`, `0x29`, each handle -1 | none |
| Shared flags, count | 3 and nought; 7 and 1 with a rider's head | nought |
| Channel | at rest role 12 at speed 1.0, nothing queued (role 12), every stamp nought; in a go flags 1 (looped), role 5, clip 0, its three stamps, speed 1.0 | role 12 at speed nought |

Bit `0x40` of a node word is on the first node of every one of the 24 and on no other node of theirs.

**A boat's three stamps in a go** are readings of the saved clock (the clock module's). The start stamp is the
reading as the clip's present loop began: it steps by the clip's length, 534 ms, each loop. The time and the
no-pause stamp are the reading at the channel's last advance, in both files the clock module's own reading.
All four boats of a file hold the same three: 114598988, 114599017, 114599017 in one of the original's files
(29 ms into the loop) and 114633495, 114633597, 114633597 in another (102 ms). The original, loading the second,
holds the same start stamp on its first frame and 114634029 a loop later.

**A rider's head** is on the lookup record of their seat's node: seat 1 is record nought, which then reads
flags `0x23` and the slot of a head's sprite (kind 1, or 3 for a costume, on the rider's bank; "A head on a
node"). The rider's own person record names no sprite while they ride. The head's sprite record holds what its
last drawing left: its place at `0x88`, its picture of 56 at `0xb8` and a scale of 0.685 at `0xa4`.

**A handle is the ride's slot in its low byte and the item's `BumperType` above it**, the same number the
object record's `mTrackRideHandle` holds: `0xfffffc00` for a Dino Karts (`BumperType` -4) in slot 0.
**Positions** are map cells times `0xc00`, a section's its 2 × 2's top-left cell. **A section's type** is its
low 16 bits, by the sides its track joins (north is -y): 5 a bend joining north and east, 6 north and west, 7 south
and west, 8 east and south; 9 an east-west straight, 10 a north-south one; 11 a crossing; 12 and 13 an east-west and a
north-south straight with an add-on. Bit 16 is set on the first two, the station's straights, which have no buoy.
The sections are stored in circuit order from the station, mostly two cells (`0x1800`) apart. Measured in the running
original (OpenTPW's `docs/exe/park.md`, "The karts' and the water ride's track, buoys and race"): a Dino Karts
looped from its mouth at (34,25) round to (28,25) held `9 9 7 10 6 9 9 5 10 8`, the first two flagged, at (30,25),
(32,25), (34,25), (34,27), (34,29), (32,29), (30,29), (28,29), (28,27) and (28,25); a Splish Splash looped the same
way, the same ten.

The shipped park's module is 44 bytes, the root and the stamp only (inflated 1,595,038 to the tag at
1,595,082). In Alexah's played jungle park it is 1,964 bytes: one Dino Karts, its 33 sections
(`9 9 9 12 12 9 5 10 10 8 11 9 7 10 6 9 5 10 10 8 9 12 9 9 7 10 6 8 5 9 7 10 6`), four cars and a close.
The played fantasy park holds one bumper arena with no sections. Walked as a tree, the module lands
exactly on its tag in all nine park files read (the shipped park and eight played ones).

A Hot Pot (`BumperType` -1) bought in the shipped park and saved before its queue reaches a path is the
ride and its close alone, 112 bytes with the root and the stamp: `ffffff00 20400 12c00 0 474 3c 1 1 10fe 0`.
Saved open it holds four cars, each a model of item 1142 with a wake of item 1141 in the model table, and
a type 9 after each car with a rider.

## The flying rides module (`RYLF`)

Between the track rides' `KART` tag and the module's own `RYLF`, written by `FUN_0055de70`:

```text
char[4]  magic               "FLY_"
u32      set up              1 in all ten park files; nought and nothing more follows
u32      table size          400
byte[400]  a table of 100 dwords
u32 x 4  four values         268500, 13500, 106500, 13500 in Easymode.TPWI
char[4] x 5  "PAD_" five times
u32      record size         0x122c
byte[0x122c] per record      one for each non-zero dword of the table's last 99
```

With no record that is 452 bytes, as in eight of the ten park files; the two saves of Alexah's played jungle park
carry one record (5,104 bytes). What a record holds is not decoded.

## The camera module (`EMAK`)

Forty bytes between the ride scripts' `ESSR` tag and the camera's own `EMAK`. The names are the loader's
(`FUN_0042cec0` logs each):

| Size | Name | In `Easymode.TPWI` |
| --- | --- | --- |
| 4 bytes, float | `gf_CameraZoom` | 110 |
| 4 bytes, float | `gf_YRotation`, the camera's turn about the point it looks at, in radians | 0 |
| 4 bytes | `gui_CameraFlags`; the loader reads it and then sets the running flags to nought | 256 |
| 12 bytes, three floats | `gs_RequiredPOIPosition`, the point the camera looks at, ten units to a cell | 475, 0, 175.01 |
| 12 bytes, three floats | `gs_SavedPOIPosition` | 0, 0, 0 |
| 4 bytes, float | `gf_SavedYRotation` | 0 |

The flags read 256 in the four files saved from the menu and nought in the three autosaves and the three
`restart.INTS`. The saved point and rotation are nought in all ten, and the rotation is nought in all ten (`-0.0`
in two). The game turns the camera a quarter turn a key press, and on loading rounds the rotation to a whole
quarter turn when one of its options is set: a file holding -0.7854 read 0 after the load, and one holding
±1.5708 read as written (measured in the game, 2026-10-08).

## The coasters module (`SAOC`)

It begins right after the `EMAK` tag and ends at the `SAOC` tag, and opens with four dwords: the number
of coasters, then three counts of the game's coaster handles. A park with no coaster reads `0 0 0 1`,
which the shipped park does (inflated 1,606,446); Alexah's played jungle park, with one Temple Of Gloom,
reads `1 1 0 2`. Each coaster then opens with a 32-byte header, read from the game's own saver and loader and
measured in that park:

| Offset | Size    | Description                                                                              | Temple Of Gloom |
| ------ | ------- | ---------------------------------------------------------------------------------------- | --------------- |
| `0x00` | 4 bytes | Flags: bit 0 the circuit is closed, bit 1 a gap is open in it; bits 2 to 8 are the coaster's own flags | `0x101` |
| `0x04` | 4 bytes | Its state word                                                                           | `0x14`, open    |
| `0x08` | 2 bytes | Its model, the coaster's type                                                          | 19              |
| `0x0a` | 2 bytes | Its model instance - the number its object record's `MeshInstanceID` holds, and how the game finds the coaster of an object | 330 |
| `0x0c` | 2 bytes | Its handle - the number its script's `+0xe0` holds in the ride scripts module             | 1               |
| `0x0e` | 8 bytes | Not described                                                                            |                 |
| `0x16` | 2 bytes | How many 34-byte (`0x22`) piece records it holds                                         | 38              |
| `0x18` | 2 bytes | How many trains are running                                                              |                 |
| `0x1a` | 2 bytes | Its capacity                                                                             |                 |
| `0x1c` | 4 bytes | How many crossings it has: a crossing adds 1 to each side, so a section crossing its own coaster counts 2. The game offers no coaster with any | 0 |

A u32 count follows, and that many u16s, the coaster's third node's crossing partners; then the pieces, each
34 bytes and its own crossing partners after it (a u8 count in its last byte, then that many u16s); a partner is
a **position in the coaster's chain of nodes**, counted from the head (the first node is 0): the loader turns each into the node
itself after the stack pass (`FUN_0043e520`, `0x0043827b`), and it does not bound a count (a u32 here, a byte for a piece;
the running game holds eight). No park file on hand holds one (the two coasters are the Minecart, counts nought throughout)
(fork watch 2026-10-20, `verify-world.md` M1); then every
train slot up to the coaster's `uiMaxTrains`, 27 bytes each, used or not; then every car slot, 11 bytes each,
its two rider counts in bytes 4 and 5 and the riders' ids after it, 4 bytes each (the saver `FUN_00437300`, the
loader `FUN_00437b60`). In both of Alexah's played jungle parks the module is 1,850 bytes, and adds up: 16 + 32
+ 4 + 38 × 34 + 6 × 27 + 24 × 11 + 20 × 4, the Temple Of Gloom being the Minecart (six trains of four cars) with
20 riders on board. A guest is offered a coaster
only with bit 0 set, bit 1 clear and no clash. The module holds no rating of the ride: the game works the ride's
excitement out again after a load.

**A piece** is one of the coaster's pylons, in the order of its track from the station; the loader (`0x00437b60`) puts
each back as a node after the station's own (three where the coaster type's flag `0x20` is set, as it is on every type shipped; two without it; OpenTPW's `docs/exe/ride-operation.md`, "A coaster's highest track
point"). Measured on the Temple Of Gloom's 38 in Alexah's jungle `New Save`, every value against the running original's:

| Offset | Size    | Description                                                                                                | Temple Of Gloom's first |
| ------ | ------- | ---------------------------------------------------------------------------------------------------------- | ----------------------- |
| `0x00` | 4 bytes | Flags: `0x1` the yaw is kept, `0x2` the incline, `0x4` the bank (otherwise the game works them out again); `0x10` reads `0x1d` in the bank's place; `0x8` is not a node flag: it calls `FUN_00442cd0` (`0x00437de9`), which sets a global and no node field, and no piece in the parks on hand carries it; `0x40` to `0x800` go to the node's own flag word (`0x00437ed0`..`0x00437f58`) | `0x400` |
| `0x04` | 4 bytes | The pylon's model instance                                                                                 | 84 |
| `0x08` | 2 bytes | Its cell's x on the ground heightfield's grid (96 to a row, 10 units a cell)                                | 26 |
| `0x0a` | 2 bytes | Its cell's y                                                                                               | 35 |
| `0x0c` | 1 byte  | Its stack ordinal                                                                                          | 0 |
| `0x0d` | 4 bytes | Its loft, a float from 0 to 1, how far up its pylon's range it stands: the pylon model's loft clip at that share puts the pylon's knot between the type's lowest and highest height above the ground (10 to 100 for the Minecart, 15.56 to 95.56 for the Gorilla Thrilla, not `fMinHeight` to `fMaxHeight`, which only hold a height asked for) | 0.1 |
| `0x11` | 4 bytes | Its bank, radians                                                                                          | 0 |
| `0x15` | 4 bytes | Its incline, radians (positive is down)                                                                    | -0.7168 |
| `0x19` | 4 bytes | Its yaw, radians                                                                                           | 1.5708 |
| `0x1d` | 4 bytes | A loop pylon's rotation, read instead of the bank, with the incline nought, when the flags hold `0x10`        | 0 |
| `0x21` | 1 byte  | How many crossing partners follow, u16 each                                                                | 0 |

The floats are unaligned. The yaw, incline and bank are the values the game had worked out when it saved, so they equal
the running game's after a load, kept or not. None of the 38 is stacked, a loop pylon or kept but for the bank and incline
of three (flags `0x6`).

## The advisor, sound and advisor scoring modules (`SVDA`, `NUOS`, `CSDA`)

**The advisor** (`FUN_00599c30`), between `SAOC` and `SVDA`: the magic `ADV_`, the dword `0x168`, 360 bytes of
the advisor's state copied whole, then a length and that many bytes. The length is nought in nine of the ten park
files (372 bytes in all) and 88 in the played Halloween autosave. Neither part is decoded.

**Sound** (`FUN_0051c350`), between `SVDA` and `NUOS`: two dwords (1 and 1024), the dword `0x54`, 84 bytes, and
then, when both dwords are set, records written one by one through a callback (`0x006b5c10`): 1,349 bytes of them
in `Easymode.TPWI`, 666 to 5,186 over the 539 different park bodies on hand. They open with three dwords and a length L, and end L bytes after
that length began, at +12+L: a loader with no sound skips them by L less the 4 it has read (`FUN_0051c400`,
`0x0051c547`). The two dwords are 1 and 1024 and the length is right in all 539.
The two dwords are the sound's "initialised" word (`0x00802bd4`, which the loader reads into a local only) and its
"enabled" word (`0x00802bc8`, which the loader writes back; with the first nought it puts back the value from before the load).
The dword `0x54` is a length, and the 84 bytes are 21 little-endian words, the sound's own state at `0x00803aa0`: four
volumes (set at `+0x18` to `+0x24`), the music voice's handle (`+0x0`, `0x00803aa0`), the crowd's voice handle (`+0x4`, `0x00803aa4`, nought with no crowd voice; the voice list below holds the voice), another voice's handle (`+0xc`, `0x00803aac`), the three on-flags (set at `+0x38`, `+0x3c`, `+0x40`), the reverb preset (`+0x14`), the crowd level
(`+0x10`, 0 to 35), a u16 music dip at `+0x30` and a u16 at `+0x32`, and words nothing reads (`+0x34`, `+0x44` to `+0x50`).
In all 558 different park bodies on hand the volumes read 100, 75, 60 and 75, the three flags 1, 1 and 1, the dip 0, the
reverb preset 0 (3 in the Halloween Full Simulation autosave) and the word at `+0x32` 1 in most (0 in the restart and autosave files).
The loader reads that many bytes with no bound.

**The voice list.** The records are the sound manager's live voices (`SoundManager_WriteState`, `0x006ba0c0`, and its list writer
`0x006b6c30`; read back by `SoundManager_ReadState`, `0x006ba150`, and `0x006b6da0`). Every value is a little-endian
dword unless a size says otherwise.

| Size | Description |
| ---- | ----------- |
| 12 | Three floats the loader reads and drops. The second is 10,000.0 in 577 of the 580 different park bodies on hand (10,000.0 is also the height a paused park lifts the listener to; whether the word is the listener's height is not established, Q560's prediction of it missed on 3 files), 46.8 in two the original saved and nought in one |
| 4 | **L**, the length, which counts **itself**: the list ends L bytes after L began |
| 4 | The voice count |
| 4 | The category record's length: the bytes after this dword and the next |
| 4 | **N**, the category slots (19 in the shipped `Easymode.TPWI` and the files that carry its block unchanged, 16 in the files the original wrote here but one, 18 in a Full Simulation Halloween `restart.INTS`) |
| 4 × N | Noughts |
| N × (1 + n) | Each slot: a byte n, then n bytes of its file name with its nought (`SoundLibrary_WriteCategories`, `0x006c0dc0`). A voice names its category by its slot here. Lost Kingdom's: slot 0 an empty name, 1 to 5 `\data\global\sound\cat_ambient`, `cat_rides`, `cat_ui`, `cat_kids`, `cat_staff`, 6 `\data\global\speech\cat_speech`, 7 to 9 the level's `sound\cat_ambient`, `sound\cat_rides` and `speech\cat_speech`, 10 its `music\cat_music`; 11 to 18 empty slots (n nought). The slots are not fixed: by file and theme a category moves (Halloween's ambient is at 8, Fantasy's speech at 10), `cat_music` may be absent, and the lobby's `cat_globallobbysfx` and `cat_locallobbysfx` can take slot 9 or 13; a reader names a slot by its file name, not its number |
| 4 | The voice pool's blocks of 100 slots (1 in every file) |
| per voice | Its handle, its **type**, and its record (below) |

**The handle** is the voice's pool slot in its low 16 bits (below 100 × the blocks in every file) and a serial in its
high 16. The reader puts the voice back into the free slot of that number and gives it the whole saved handle
(`0x006b6efd`, `0x006b6fc3`), so a handle kept elsewhere in the file names the same voice after a load.

**The type** is the voice's class: 1 the music's class (effect flags `0x2`, `0x4`, `0x400`), 2 the class of flag
`0x10`, 3 a one-shot (no `0x4`), 4 the chain of any variation (`0x4` alone), 5 the plain class (`0x2` and `0x4`; a second class, vtable `0x0070a8d0`, also writes 5 and the reader rebuilds it as the plain one; when it is made is not settled), 6 the
held chain (`0x4` and `0x400`); a child is 7 (of a chain of any variation) or 9 (of a held chain). The class of flag
`0x100` writes 10, which the reader has no case for: its record would not be read, and the rest of the list would be
read wrong. So would a voice whose handle's number has no free slot (`0x006b6f04`); no file on hand has one. No file on hand holds a 10 or a 2; over the 580 bodies the voices are 2,304 one-shots, 1,955 plain,
1,193 of the music's class, 682 held chains (627 with a child) and 2 chains of any variation (each with a child).

**A voice's record** (`Voice_WriteRecord`, `0x006bccf0`; `Voice_ReadRecord`, `0x006bd0a0`):

| Offset | Size | Description |
| ------ | ---- | ----------- |
| `0x00` | 4 | `0x0012d591`, a mark the reader checks |
| `0x04` | 12 | The place: x, height, z as floats. A voice with no place (effect flag `0x200`) holds the whole words 0, 1, 0 |
| `0x10` | 4 | Nought in every file the original wrote; the shipped `Easymode.TPWI` holds small floats here (0.01 to 0.71). The reader drops it |
| `0x14` | 4 | The voice's own range, a float: nought for none (97 and 19 for the level's placed sounds) |
| `0x18` | 4 | The category's slot in the list above; nought for none found. The writer then writes the type and nothing of the record (`0x006bcd6d`), and the reader takes the next voice's handle for the mark, gives up on this record and reads the rest of the list from the wrong bytes (no file holds one) |
| `0x1c` | 4 | The effect's place in its category's effect table, from nought |
| `0x20` | 4 | The variation's place among the effect's, from nought |
| `0x24` | 4 | The sample's place among the variation's, from nought |
| `0x28` | 4 | The mark again |
| `0x2c` | 4 | 1 if the voice has controllers, else nought |
| | 24 | With controllers only: four key bytes, four value bytes, then the controller block's `+0xc`, `+8`, `+0x10` and `+0x14`, in that order: for a voice that follows the listener in a rectangle, its corners, x and z then x and z |
| | 4 | The mark again |
| | 1 | 1 if the voice follows the listener in a rectangle (voice flag `0x1000`) |
| | 4 | The handle again; the reader drops it. A child has none of its own, and this word holds whatever its memory held |
| | 4 | The child's type, nought for none; then the type again and the child's own record, laid out the same way |

The records stand oldest first: the writer walks the live list from its tail (the oldest voice) to its head, and the
reader links each voice at the head, so the live order comes back as it was. The reader stops every live voice first, then, with sound on, rebuilds each and sets it up as a new voice of its
effect, variation and sample would be, at its place and range.

**What the shipped park holds.** `Easymode.TPWI`'s ten voices, as the original puts them back on entering it (read in
its memory, same handles and classes): the jungle music's effect 2 (`0x0126005a`); the global ambient's 2 and 3, plain,
each following the listener in its rectangle (353, 31 to 450, -66 and 372, 21 to 567, -174); the global ambient's 8 and
7 at (530, 0, 513) and (511, 0, 664), one-shots with a range of 97, and the jungle ambient's 181, flat, range 19; the
gates' held chain of 177 at (450, 0, 160), key 18, with a child of its variation 8; the jungle rides' 107 at (585, 0,
185) and the global rides' 93 at (445, 0, 295), the handles of the Fountain's and the Litter Bin's object-list records (their scripts'
handles 14 and 7); and kids 91, the crowd, whose handle is also the sound block's word at `+0x4`.

**Who else keeps a voice's handle.** A script's object-list record (types 3 to 10, "The object list" above): over the
580 bodies 1,394 of 1,581 such records name a saved voice, and the other 187 name none (their voices had ended or were
left out). The sound block's word at `+0x4` (the crowd's voice) names a saved voice in all 569 bodies where it is
not nought; the weather's `mRainSoundHandle` is set in one, and names a saved voice. The block's `+0x0` (the music voice's) names a saved music-class voice in 59 of the 65 different lists on hand and is nought in 6; `+0xc` names a saved voice in 1 and is nought in 59, a handle with no voice in 5.

All 580 bodies parse so and end exactly at L; 31 different lists among them (fork watch 2026-10-20, `verify-world.md`
M2, for the length; Q560 for the records; the code is OpenTPW's `docs/exe/audio.md`, "A park file's voices"). OpenTPW
reads the list (`ParkVoices`), puts back its held chains and counts the other classes (Q612); its writer writes the loaded
file's list less each voice put back and let go since (`ParkVoices.Without`).

**Advisor scoring** (`FUN_0059c890`), between the cheats' `STHC` and `CSDA`: one dword, 277 in `Easymode.TPWI`,
nought in three of the played files and 113 to 258 in the rest. It is the advisor's poll cursor: the row of his
351-row message table that the next tick's poll starts from. Each tick scores the first polled row from there (155
of the rows are polled) and leaves the cursor on the row after, so it reads 0 to 351; a load reads it back
(`FUN_0059c860`). `Easymode.TPWI`'s 277 is the row after the last its makers' park scored on tick 754: with it, a
park begun from that file scores row 185 on tick 877. OpenTPW reads it at a load and writes its own.

## The cheats module (`STHC`)

Two bytes, between the sound's `NUOS` tag and the cheats' own `STHC`.

| Offset | Size   | Description | In `Easymode.TPWI` |
| ------ | ------ | ----------- | ------------------ |
| `0x00` | 1 byte | The cheats flag: 1 lets the park's cheat keys work | 1 |
| `0x01` | 1 byte | Not described: nothing but the reader, the writer and the constructor touches it | 1 |

The eight played park files carry `00 00`, the five Full Simulation saves among them. So a park loaded from
`Easymode.TPWI` starts with the flag set, **in the byte**: the game re-runs the object's constructor, which clears the flag,
behind three first-use guards (the load, a rename and the first key), so in a fresh launch the first key that reaches the cheat
check clears it and the cheat keys work from a later park of the same run (fork watch 2026-10-20, `verify-other-ui-audio.md`
O-G; read from the bytes, not run in the original).

## A park made fresh (`restart.INTS` at tick nought)

The game writes `restart.INTS` as a park is first entered, before its first tick. For a player with no park file the
park is a fresh world, so these files are the fresh world as the game writes it. Measured over seventeen Lost Kingdom
files the game wrote (one on Windows, sixteen under Proton) and the Fantasy and Halloween ones beside them; what makes
each value is OpenTPW's `docs/exe/saves.md`, "A park made fresh: its file at tick nought".

- **The container** is version 500, and the preamble after it is the shipped `Easymode.TPWI`'s byte for byte, legal
  text and all, in every theme. The payload is 1,560,670 to 1,561,505 bytes, against the shipped park's 1,608,309.
- **The modules' lengths** are the same in every Lost Kingdom file, but the sound's: the action recording 143 (length
  135), World 1,469,113, the sprite table 412, the particles 76,252, the message centre 196, the clock 8, the vanilla
  time 4, the game system 36, the ride system 11,311 (11,307 Fantasy, 11,319 Halloween), the track rides 44, the
  flyers 452, the ride scripts 1,176 (1,170, 1,152), the camera 40, the coasters 16, the advisor 372, the sound 1,017 to
  1,484, the cheats 2, the advisor scoring 4 and the interface block 4.
- **World.** Twelve things, ids 1 to 12, models 9, 10, 11, 12, 13, 14, 15, 16, 17, 19, 3 and 3; `mFirstObject` 12,
  the gates 11, the lights 12, the five staff heads nought, tick 0, closed 1. The clock fields hold month 0 and day -1,
  not the date of tick 0. The staff pool holds 22 candidates. Each manager's record is its constructor's: the analyser's
  1,024 blocks each `(x, y, 4, 4)` with nine pairs of `(-999, 0)` (the first two as floats), its four small rings
  `(-1, 3, ...)`, its six kept pairs `(-999, 0)`, its sixteen int rings opening `-1, 144` and its six stamps all
  `125911584000000000` (2000-01-01); the advisor's 351 records all fresh. In about half the files the bytes no constructor
  writes (the lab past its used projects, the advisor's records' unused bytes, its eight message slots and its kept slot, the challenge manager's `+0x17c` to
  `+0x393`, the two objects' day rings and name words) hold leftover memory, the same in each such file but for 7 bytes
  of the challenge manager's at `+0x28a`; in the others they are
  nought. The weather's location holds leftover memory in every file.
- **The sprite table** has 100 slots, none live: `TPCS`, `0x118`, 100 and 400 bytes of nought.
- **The particles** hold one emitter in use, slot 0 with the count 1 (handle `0x10000`): effect 20, `WaterFall`, at
  (532.3, 4.9, 526.1), raw 34068, 312, 33668 (Lost Kingdom; Fantasy and Halloween hold no emitter). The live header's generator seed is `0x18be873a` in every Lost Kingdom file (seed 1 in the others), its `+0x1c` `0x803b68` and `+0x24` `0x5d60c0` in every fresh file (a load keeps the live system's two and ignores these); the density is 1,500 on Windows and
  1,000 under Proton.
- **The message centre**'s sets: `0x01` and `0x02` thing 7; `0x05` 3; `0x07` 2, 4, 9; `0x08` 3, 9; `0x0a` 9; `0x0b`
  10, 11, 12; `0x0c` 1, 4, 5, 8; `0x0d` 8; `0x0e` 1, 3; `0x0f` 9; `0x11` 3, 5, 6, 10; `0x12` 9; `0x13` 3, 4, 5, 8, 9;
  `0x14` 4; `0x17` 3; `0x18` 4; `0x19` 3, 4; `0x1a` 10; `0x1b` 9; `0x1c` 9; the rest empty.
- **The game system**'s dwords 6 to 8 are 11,864 under Proton (11,863 and 11,866 once each) and 2,235 to 2,992 on Windows, and the readings stand 33 ms (once 34) before the clock's first under Proton, 3 ms on Windows; dwords 3 and 4 nought,
  dword 5 the stack's 2.
- **The ride system** holds 109 records: the gates', the lights' and 107 of the boundary's pieces, whose items vary
  from file to file.
- **The track rides** hold the stamp alone: `15:25:02` and `Mar 17 2000`, where the shipped park's reads `20:25:29` and
  `Oct 21 1999`.
- **The flyers** are set up with no flyer, the four values 190000, 70000, 140000 and 30000 in Lost Kingdom.
- **The ride scripts** hold two records, the gates' (handle 1) and the lights' (handle 2), with no object; the next handle is 3.
- **The economy thing**'s `mLastBalance` (its word at `+0x20`) holds the starting cash, 50,000, as `mBalance` does, and the loans' table is the balance's.
- **The camera**: zoom 110, rotation 0, flags 0, the point (475, 0, 175), the saved point and rotation nought.
- **The coasters** read `0 0 0 1`; **the advisor** is `ADV_`, `0x168`, the advisor's 360 bytes (words at `+0x40` to
  `+0xb4`, `+0x110` 5, `+0x12c` -1 and `+0x144` to `+0x158`) and the length nought; **the cheats** `00 00`; **the
  advisor scoring** nought (258 in the Halloween file); **the interface block** a count of nought.
- **The sound**'s voice list holds the land's five placed voices and whatever the game had playing as the park was
  entered.
