---
title: Saves (*.tpws, *.ints, *.lays)
---

Theme Park World stores saves with three different extensions.

These extensions are:

- **TPWS**: Standard save
- **INTS**: Initial save (e.g. for *Instant Action* mode)
- **LAYS**: Online save (for uploading parks)

## File format

Offsets below are of an offline save; an online one carries extra data after the file info and is not
covered here.

**Header**

| Offset | Size | Description |
| --- | --- | --- |
| `0x000` | 4 bytes | Version. The park the game ships carries `400`; a saved park carries `500`. This is **not** a magic number - `F4 01 00 00` is simply `500` written little-endian |
| `0x004` | 1 byte | Padding |
| `0x005` | 824 bytes | Copyright notice, UTF-16 - so 412 characters, and reading it as single bytes gives every other byte as a NUL |
| `0x33D` | 711 bytes | Zeros |

**File info**

| Offset | Size | Description |
| --- | --- | --- |
| `0x604` | 4 bytes | File type - `00 01 22 19` |
| `0x608` | 1 byte | File version - `85` |
| `0x609` | 1 byte | Online flag - `00` for offline, `01` for online |
| `0x60A` | 3 bytes | Padding |

**Data (compressed using ZLIB)**

| Offset | Size | Description |
| --- | --- | --- |
| `0x60D` | 4 bytes | Tag - `BILZ` |
| `0x611` | 4 bytes | The size the payload inflates to |
| `0x615` | 4 bytes | The size of this whole block, its tag and header included - so `0x60D` plus this is the file's length |
| `0x619` | 16 bytes | Not identified; `15, 9, 0, 0` then zeros in the shipped park |

Neither of those two sizes is a compressed length. The ZLIB stream begins at `0x629` - the 28-byte
header counts the tag - and continues to the end of the file.

## Inside the payload

The inflated payload is a run of blocks, each closed by a four-character tag. The tags are written as
little-endian dwords, so **every one of them reads backwards in a byte dump**: searching an inflated
save for `WRLD` finds nothing and searching for `DLRW` finds it at once.

### The world block (`WRLD`)

The first block in the payload is the park itself, closed by the `DLRW` trailer. It opens with an
untagged recording - a flag, then a length, then that many bytes - so where the block's own header begins
is derived from that length rather than fixed. In the shipped park the length reads 1171, which puts the
header at `0x49B`.

The header is 26 fields written back to back with no padding. **The names below are the game's own.**
Each field is announced to a logging call that the release build compiles away, so the names never reach
the file - but they survive in the executable beside the address each one is read into, which is what
makes this list checkable rather than inferred.

| # | Size | Name | Notes |
| --- | --- | --- | --- |
| 0 | 4 bytes | *(version)* | The one field the logging call does not name |
| 1-3 | 2 bytes each | `mArrivalVehicle_Size1..3` | |
| 4 | 2 bytes | `mBankAccount` | A **handle**, not an amount - see below |
| 5 | 2 bytes | `mCurrentArrivalVehicle` | The bus; zero when none is due |
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

A handle is a thing id, compared against a thing's own id with `==`: `11` means "the thing whose id is
11", not "the eleventh thing".

`mParkClosed` reads the opposite way to its name. The command that opens and shuts a park writes `0` on
one branch and `1` on the other, then picks the word for its own message with
`mParkClosed == 0 ? "opened" : "closed"`. The world constructor writes `1` before anything is loaded, so
a park is born shut and a save holding `0` is one that was opened while it was being played.

`mBankAccount` is the field whose name misleads. It is two bytes, and the shipped park holds `8` in it -
no sort of balance. The executable reads it in exactly one place, and that reader is the weather thing's
accessor character for character with one offset changed: take the word, return `thingTable[id]`. So it
names a *thing*, and the thing it names is the one carrying the park's admission fee, balance, profit for
the year and loan table - which is why the state a guest is in while judging the admission fee reaches
the park's money through this very accessor.

`mWorldState` really is a value. The executable writes `1`, `2` and `4` into it and compares it against
`4` in six places, among them the game's own state machine and the build-a-park menu. The shipped park
holds `0`, which is not in that set - so either zero is a state nothing writes while a park is being
played, or it is what a park carries before it is first entered. Naming it either way would be a guess.

After the header come 150 object-control records, a pool of timers, the 128x128 map, and then the thing
list.

#### The map

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
reads the combinations no shipped park happens to contain.

The map sub-record is a **29-byte tile base** followed by a **23-byte litter block**. The track
sub-record repeats the same tile base field for field.

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
| 50 | 2 bytes | *(unnamed)* - the thing occupying the cell |

The serialiser really does announce `mLitterScript` twice, for two consecutive dwords, and the arithmetic
is what says so rather than the reading: `4+2+4+4+2+1+4+2` is exactly 23, and `29+23` is exactly the 52
that the cell walk measures from the other direction. Written once, every field after it would shift by
four and the record would close four bytes short of the next cell's status byte.

**`mStatusFlags` is the attribute map.** The byte at offset 45 holds the same value `base.map` carries
for that cell - checked across all 16,384 cells of the shipped park against a separate file, with its own
header, parsed by different code, and indexed `x * 128 + y` where the save's cells run `y * 128 + x`.
Every cell agrees. Agreement in aggregate would prove little; agreement cell by cell under *opposite*
indexing is not something a misaligned or transposed reading can produce. 1,495 cells are non-zero, over
exactly the eight values the attribute map uses: 0, 1, 3, 8, 17, 128, 144 and 148.

**The unnamed short at 50 is occupancy** - the id of the thing standing on the cell. Twenty-four cells of
the shipped park carry a value, and eleven of them are exactly its eleven placed catalogue objects, each
naming *itself* on the cell it stands on: the cell at (55,15) holds `23`, and object `23` stands at
(55,15), and so for all eleven. The remaining thirteen hold person ids, gathered on the approach to the
park gates at x 47-48 and at the staff's own positions. A guest waiting to be let in tests the cell
underfoot against their own id, which is this field read from the other side - though that test is made
against the cell's *runtime* record, which is `0x44` bytes where the file carries 52, so the two layouts
do not share offsets.

Nothing has been dropped in the shipped park: `mLitter`, `mLitterCollector`,
`mTimeMarkedForLitterCollection` and `mPylonIndex` are nought on every one of the 16,384 cells. That is a
fact about a save nobody has played rather than a gap in the reading.

#### Thing records

The thing list is a **linked list, not an array**. Each record opens with the id of the *next* thing and
then its model number, four bytes each, and a next of zero ends the list. The ids are not in order - the
shipped park runs 41, 40 … 29, then 15, then 28 - which is what a list with something spliced into it
looks like, and what a counter cannot be.

Every thing that has a place in the world then writes the same four 2-byte fields in this order: `mX`,
`mY`, `mMapChild`, `mMapParent`. So a thing's *own* fields begin **16 bytes into its record** - eight of
list head and eight of map base. `mX` and `mY` are in 256ths of a cell, and their high bytes are the
cell the thing stands on, which is how the engine reaches a cell without dividing.

**The offsets below are file offsets, and they are not the offsets a decompiler shows.** A thing is
written field by field in the order its reader asks for them, so a field's place in the record is the sum
of the sizes before it and bears no relation to where it sits in memory: `mAdmissionFee` is at `+0x118`
in the running game and at `+16` in the record. Taking the memory offsets and using them as file offsets
produces something that parses and is wrong.

#### A catalogue object (model 3)

Model 3 is everything a player buys and places - shops, rides, sideshows and scenery. The shipped park
holds **fourteen** of them: eleven placed, and three carrying the unplaced sentinel `128` in both
coordinates, because their positions live in their models rather than in the save. Its record is
**1,099 bytes**.

| Offset | Size | Name | Notes |
| --- | --- | --- | --- |
| 8 | 2 bytes | `mX` | 256ths of a cell, from the shared map base |
| 10 | 2 bytes | `mY` | |
| 16 | 4 bytes | `mAngle` | `0`, `90` or `270` in the shipped park |
| 20 | 2 bytes | `mId` | the item's `Info.Id`, from its own `.sam` |
| 22 | 32 bytes | eight `tv_t` dwords | packed and unpacked by a helper |
| 54 | 4 bytes | `MeshInstanceID` | |
| 58 | 2 bytes | `mFlags` | see below |
| 60 | 132 bytes | 33 pairs of `mNameA[`*i*`]`, `mNameB[`*i*`]` | 2 bytes each |
| 192 | 4 bytes | `mRideScriptHandle` | |
| 196 | 4 bytes | `mTrackRideHandle` | |
| 200 | 4 bytes | `mState` | |
| 204 | 2 bytes | `mTopLeft` | |
| 206 | 2 bytes | `mEntryPos` | the cell a visitor is sent to |
| 208 | 2 bytes | `mNext` | this object's link in the object list |
| 210 | 2 bytes | `mAssignedStaffMember` | |
| 212 | 2 bytes | `mBackOfQueue` | |
| 214 | 4 bytes | `mCanLoad` | |
| 218 | 2 bytes | `mExitPos` | |
| 220 | 2 bytes | `mFirstInQ` | |
| 222 | 4 bytes | `mIsTrackRideValid` | |
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
at 1090**, `mUpgradeBalloonSprite` at 1094 and `mUpgradeLevel` at 1098.

**A distinction worth keeping.** Those offsets all follow the *last* ring, so they hold for **any** split of
the 180 entries across the six rings - only the even split is a guess, and it is not one they depend on. The
two fields that sit *between* rings, `mNumCustomers` and `mNumWalkAways`, do depend on it, and are therefore
not safe to read on this evidence alone.

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

#### The economy thing (model 16)

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
to `mBalance` and to `mProfitThisYear`, and touches neither of the others.

#### A guest (model 1)

The sixth person model is the visitor, and its block begins at **+398**, after the same eight-byte list
head and 390-byte person base a staff member has. It is **135 bytes**, making the record `8 + 390 + 135`
= **533**.

| Offset | Size | Name | Shipped park |
| --- | --- | --- | --- |
| 398 | 4 bytes | `mArrivalDate` | |
| 402 | 4 bytes | `mArrivalIndex` | |
| 406 | 4 bytes | `mBalloonScript` | |
| 410 | 4 bytes | `mBeenAdmitted` | |
| 414 | 4 bytes | `mCash` | `684`, `510`, `654`, ... eleven different amounts across thirteen guests |
| 418 | 4 bytes | `mExitLevel` | `142`, `98`, `57`, ... |
| 422 | 4 bytes | *unnamed float* - happiness | |
| 426 | 4 bytes | *unnamed float* - hunger | `18`, `25`, `61`, ... |
| 430 | 4 bytes | `mLastPosX` | |
| 434 | 4 bytes | `mLastPosY` | |
| 438 | 4 bytes | *unnamed float* - litter | |
| 442 | 2 bytes | `mMajorDest` | thing handle - what they have chosen, or none |
| 444 | 4 bytes | `mNumRides` | |
| 448 | 4 bytes | `mNumShops` | |
| 452 | 4 bytes | `mNumSideshows` | |
| 456 | 4 bytes | `mNumSideshowsWon` | |
| 460 | 4 bytes | `mPaidAdmission` | |
| 464 | 4 bytes | `mParkOpeningWaitingTime` | |
| 468 | 1 byte | `mPersonType` | `5`, `7`, `2`, ... an index into `PeepTypes[0..7]` |
| 469 | 1 byte | `mPrankeryIndex` | |
| 470 | 16 bytes | `mPreviousRides[`*i*`]` and `mPreviousTemporaryRides[`*i*`]` | **interleaved in pairs**, four of each, 2 bytes apiece |
| 486 | 2 bytes | `mQNext` | `0` on all thirteen |
| 488 | 2 bytes | `mQPrev` | `0` on all thirteen |
| 490 | 4 bytes | `mQueueMoveDelay` | |
| 494 | 1 byte | `mQueuePos` | `0` on all thirteen |
| 495 | 4 bytes | `mRemainingBalloonLife` | |
| 499 | 2 bytes | `mSavedMajorDest` | |
| 501 | 4 bytes | `mSavedState` | `6` on all thirteen - a new guest is constructed deciding |
| 505 | 4 bytes | `mState` | `2`, `5`, `3` - heading for the gate, entering, waiting outside |
| 509 | 4 bytes | *unnamed float* - thirst | `36`, `13`, `12`, ... |
| 513 | 4 bytes | `mTimeOfLastSpotAnim` | |
| 517 | 4 bytes | `mTimeStartedIdling` | |
| 521 | 4 bytes | *unnamed float* - `mTiredness` | `0` throughout |
| 525 | 4 bytes | *unnamed float* - toilet | `13`, `15`, `24`, ... |
| 529 | 4 bytes | *unnamed float* - vomit | |

**The block closes on 533 exactly**, and that is what makes the offsets above worth trusting. They are not
measured one at a time: they are produced by walking the serialiser's own declared field sizes from +398,
and that single walk has to land on the record size - derived separately, by running the original's reader
and logging what it declared - or every offset in it is wrong together.

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
transposes one pair, the guest block's alphabetical order holds throughout.

#### A member of staff (models 4 to 8)

Five of the six person models are staff - **4** mechanic, **5** handyman, **6** entertainer, **7** guard,
**8** researcher - and all five share one block, because the original gives them one class. It begins at
**+398**, the same place a guest's own block begins: both follow the eight-byte list head and the
390-byte person base. It is **105 bytes**, and each kind then adds a few fields of its own.

| Offset | Size | Name | Shipped park (25, 26, 27, 28, 30) |
| --- | --- | --- | --- |
| 398 | 4 bytes | `mCurrentPayGrade` | `3`, `3`, `3`, `3`, `2` |
| 402 | 4 bytes | *unnamed float* - happiness | `89`, `89`, `92`, `91`, `97` |
| 406 | 4 bytes | `mJobsDone` | `0` on all five |
| 410 | 66 bytes | `mName[0..32]` | 33 shorts, not text |
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

### The sprite table (`TPCS`)

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
executable rather than stored in the save**. `SPSC`, despite its name, is the table of sprite *instances*
above and not the programs: the loader points every instance at the built-in array unconditionally and
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
