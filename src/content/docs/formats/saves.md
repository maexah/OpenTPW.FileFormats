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
