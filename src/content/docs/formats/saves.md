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
| `0x08` | 4 bytes | Cursor into the sprite's animation program |
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
