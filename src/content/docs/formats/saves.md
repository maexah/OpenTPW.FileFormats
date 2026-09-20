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

| Size      | Description                  |
| --------- | ---------------------------- |
| 4 bytes   | Magic number - `F4 01 00 00` |
| 823 bytes | Copyright Notice             |
| 711 bytes | Padding                      |

**File info**

| Size    | Description                                     |
| ------- | ----------------------------------------------- |
| 4 bytes | File type - `00 01 22 19`                       |
| 1 byte  | File version - `85`                             |
| 1 byte  | Online flag - `00` for offline, `01` for online |
| 2 bytes | Padding                                         |

**Data (compressed using ZLIB)**

| Size     | Description           |
| -------- | --------------------- |
| 4 bytes  | Magic number - `BILZ` |
| 4 bytes  | Unknown               |
| 4 bytes  | Compressed length     |
| 16 bytes | Unknown               |

The ZLIB stream begins after this point, and continues to the end of the file.

## The World block

The decompressed payload opens with the World block, and that block opens with a header of
fixed-width fields written in a fixed order. There is no table of contents and no field names in the
file: the names below are the game's own, recovered from the executable, where each field is
announced to a logging call that the release build compiles away.

| Size | Field |
| ---- | ----- |
| 4 | `version` |
| 2 | `mArrivalVehicle_Size1` |
| 2 | `mArrivalVehicle_Size2` |
| 2 | `mArrivalVehicle_Size3` |
| 2 | `mBankAccount` |
| 2 | `mCurrentArrivalVehicle` |
| 4 | `mGameTick` |
| 2 | `mMechanicHQ` |
| 2 | `mParkAnalyser` |
| 4 | `mParkClosed` |
| 4 | `mNumberOfVisitorsToDate` |
| 2 | `mParkGates` |
| 2 | `mTrafficLights` |
| 4 | `mRandomSeed` |
| 2 | `mResearchLab` |
| 2 | `mStaffHQ` |
| 2 | `mTagSystem` |
| 2 | `mUIMsgReceiver` |
| 2 | `mWeather` |
| 4 | `mWorldState` |
| 2 | `mFirstHandyman` |
| 2 | `mFirstMechanic` |
| 2 | `mFirstEntertainer` |
| 2 | `mFirstGuard` |
| 2 | `mFirstResearcher` |
| 2 | `mFirstObject` |

The sizes above total 64 bytes.

**The order in the file is not the order in memory**, and this is the trap to know about. The
engine's own field-listing pairs each name with the struct offset it reads into, and those offsets
run `mRandomSeed` `+0x1da708`, `mGameTick` `+0x1da70c`, `mParkClosed` `+0x1da710`,
`mNumberOfVisitorsToDate` `+0x1da714`, `mWeather` `+0x1da724`, `mBankAccount` `+0x1da726`,
`mParkGates` `+0x1da732`, `mWorldState` `+0x1da738` — a different order from the one written. Read
the file by the list above; do not sort by offset.

### The arrival vehicles

Most fields ending in an id hold a thing id, or nought where the park has none. The four arrival
fields are worth calling out because their names encode behaviour.

A park does not store a vehicle for each of the bus, the seaplane and the ferry. It stores one for
each **size of arriving crowd** — fewer than 36 people take the first, up to 60 the second, and more
than 60 the third — and the engine creates the vehicle thing the first time a crowd of that size
turns up, then caches its id in the matching slot. `mCurrentArrivalVehicle` holds whichever is on its
way, or nought when none is.

So the slots say what a park has *done*, not what it owns. Theme Park World's shipped Lost Kingdom
park holds a bus in `mArrivalVehicle_Size1` and nought in the other two, which is why its thing list
contains a bus and neither a ferry nor a seaplane: no crowd large enough for those has ever arrived
there.
