---
title: Strings (*.dat, *.str)
---

## Encoding

Strings are converted from Unicode to Bullfrog Multibyte format using two files: `MBToUni.dat` (converting from Multibyte to Unicode) and `UniToMB.dat` (converting from Unicode to Multibyte).
The contents of this file may differ depending on the language that is being used and depending on which characters are required.  The offset of each of these characters is then specified within a BFST file.

The index stored in the BFST file is **one-based**: a stored byte of `n` means `MBToUni` entry
`n - 1`. The game's own decoder subtracts the one (`0x006afe80`). Getting this wrong shifts every
character by one and still produces plausible-looking letters, so it is easy to miss. Decoding the
end of row 4 of `data\Language\English\TAG_SYSTEM.str` zero-based turns

```
Hi there! Welcome to Theme Park :)
```

into

```
Gh<TAB>sgdqd<US><TAB>Vdkbnld<TAB>sn<TAB>Sgdld<TAB>O`qj<TAB>/(
```

Every letter shifts down one, the space (index `0x0C`) lands on tab, and `!` on `U+001F`, a
control character that prints as nothing (shown here as `<US>`).

### File Format

This is the layout of `MBToUni.dat`. `UniToMB.dat` is laid out differently (below).

**Header**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | Magic number - `BFMU` (`42 46 4d 55`)                                          |
| 2 bytes            | Lead bytes - `0` in both shipped tables (see note)                             |
| 2 bytes            | Character count                                                                |

**For each character**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 2 bytes            | The character itself, as a Unicode code point                                  |

The header is 8 bytes: on the English `MBToUni.dat` the count at offset 6 reads 249 and
`8 + 249 * 2` is the file size exactly (248 and 504 bytes on the american one). The two shipped
tables differ: the american one has no `’` (`U+2019`), which is index `0x15` (entry `0x14`) in the
English one, so every character after it has an american index one lower than its English one.
Each language folder's strings have to be decoded through that folder's own table: the american
`THEMENAMES.str` row 0 read through the English table comes out `Mptu Ljohepn`.

**The note on lead bytes.** The game reads the first of these two bytes as the number of byte
values, counted down from `0xff`, that begin a two-byte character (`0x006afe80`); the loader
(`0x006ada90`) does not read the second. Both bytes are `0` in both shipped tables, so every
character in the shipped strings is one byte.

The loader does not bound a lookup by the character count: it runs the count's two bytes
through the formula for a two-byte character, as if the low byte were the lead byte and the high byte
the one after it, and keeps the result as the bound (`0x006ada90`). That gives 64005 for the English
table and 63750 for the american one, so no one-byte index is refused (`0x006afe80`).

### UniToMB.dat

`UniToMB.dat` has its own magic, `BFUM` (`42 46 55 4d`), and no count: after the magic, from
offset 4 to the end of the file, it is a binary search tree of code-point ranges. The game starts
at the node at offset 4 and walks down it (`0x006ad900`).

**Each node**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 2 bytes            | First code point in the range                                                  |
| 2 bytes            | Where the node for lower code points starts, counted from this node; `0` if none |
| 2 bytes            | Last code point in the range                                                   |
| 2 bytes            | Where the node for higher code points starts, counted from this node; `0` if none |
| 2 bytes each       | The multibyte code of each code point in the range, first to last              |

A code point in no range converts to `0`. A code is the same one-based index a BFST file stores;
one above `0xff` would be written as two bytes, low byte first (`0x006ada00`), and none in the
shipped files is.

The English file (526 bytes) has three nodes: `U+0009`-`U+00FF` at offset 4, with `U+0007` below
it and `U+2019` above it. The american one (516 bytes) has two: `U+0007` at offset 4 and
`U+0009`-`U+00FF` above it. Each is the exact inverse of its folder's `MBToUni.dat`: every
character there maps back to its one-based index, and nothing else is listed. The `09 00` at
offset 4 of the English file is its first node's first code point, `U+0009`; the american file
reads `07 00` there.

## Storage

The Bullfrog String file format (*.str) is used in order to store localized strings for in-game text.
These don't have any specific character encoding - they use the two aforementioned file formats to convert to and from 'Bullfrog Multi-byte' characters.

### File Format

**Header**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | Magic number - `BFST`                                                          |
| 4 bytes            | A number from 1000 to 1020, different in every table: its resource id (see note) |
| 4 bytes            | String count                                                                   |

**The note on the second word.** Measured across all 42 `.str` files the game ships, 21 tables in each of
`data/Language/English` and `data/Language/american`: every table has a value of its own, the 21 values run
1000 to 1020 with none repeated, and a table carries the same value, and the same string count, in both folders.
It is the table's own resource id: [`residx.dat`](/formats/fonts/#residxdat) in the same folder lists each of
the 21 tables under exactly that value, with type `3`. The loader (`0x006acec0`) compares the header's copy
with the id it looked the file up under and refuses the table when they differ.

| Value | Table | | Value | Table | | Value | Table |
|---|---|---|---|---|---|---|---|
| 1000 | `ERRORMSG` | | 1007 | `UIHELPTEXT` | | 1014 | `GUARD_NAMES` |
| 1001 | `INGREDIENT` | | 1008 | `CHAT_COMMANDS` | | 1015 | `RESEARCHER_NAMES` |
| 1002 | `TAG_SYSTEM` | | 1009 | `THEMENAMES` | | 1016 | `FEMALE_NAMES` |
| 1003 | `THOUGHTS` | | 1010 | `OBJECT_NAMES` | | 1017 | `ITEMTYPES` |
| 1004 | `KIDSTATES` | | 1011 | `HANDYMAN_NAMES` | | 1018 | `LOANNAMES` |
| 1005 | `STAFFSTATES` | | 1012 | `MECHANIC_NAMES` | | 1019 | `STAFF_TYPES` |
| 1006 | `UITEXT` | | 1013 | `ENTERTAINER_NAMES` | | 1020 | `KEYBOARD` |

**String Directory**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | String offset (from the end of the string count)                               |

One entry per string. In all 42 files every offset is a multiple of four, the first is the
directory's own length, and they rise in order.

**For each string (at offset)**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 1 byte             | Part type - `01`, text: every shipped string starts with one (see note on parts) |
| 3 bytes            | String length (see note)                                                       |
| *n* bytes          | Characters in BFMU format                                                      |
| 0-3 bytes          | Zero padding to a multiple of four                                             |
| 4 bytes            | `00 00 00 00`, the end of the string - unless further parts come first (see note on parts) |

**The note on string length.** Of the 4,730 records in the 42 shipped `.str` files, four are longer
than 255 characters, all in `UITEXT.str`: row 400 reads `01 06 01 00` (262) in English and `01 00 01 00`
(256) in american, and row 417 reads `01 67 02 00` (615) and `01 66 02 00` (614). Each decodes cleanly to
that full length and ends at the zero padding before the next record, so the length is at least two
bytes, little-endian. The fourth byte is `00` in every record; the game takes the length as all three
bytes, the part's word shifted right by eight (`0x006acb70`). A reader that takes only the first byte
cuts those four short: English row 400 comes out as `CHANGE`, the american one as nothing.

**The note on parts.** A string is a run of parts, each starting on a four-byte boundary with a
four-byte word: the part's type in the low byte and a value in the three bytes above it. The game's
interpreter for them is `0x006acb70`.

| Type | Value | What follows the word |
|---|---|---|
| `0` | `0` | Nothing: the string ends |
| `1` | The length *n* | *n* characters, then zero padding to a multiple of four |
| `2` | A parameter number | Nothing: the game inserts the value it was handed under that number |

A string of plain text is one text part and the end: four to seven zero bytes after its last
character. Of the 4,730 shipped strings, 4,256 are plain text. The other 474, 237 in each folder, in
`CHAT_COMMANDS`, `ERRORMSG`, `KIDSTATES`, `TAG_SYSTEM`, `UIHELPTEXT` and `UITEXT`, hold 868 parameter
parts, numbered 0 to 27; each of them starts and ends with a text part, which may be empty. A parameter
is found by its number, not its place: English `TAG_SYSTEM` row 185 is text ending
`…a golden ticket for getting `, parameter 1, ` visitors in the last `, parameter 2 and ` months.`, and
has no parameter 0.

The interpreter also takes types `3` to `6`, which hold settings handed to the next parameter part
(a type `3` part is followed by that many four-byte words); no shipped string uses them, and any other
type fails the string. The game's plain-string lookup (`0x006ace60`) returns a string only when it is a
single text part. A reader that takes only the first text part cuts a string at its first parameter:
English `ERRORMSG` row 10, `Error: Code `, parameter 0, ` - please refer to the readme.txt file for
details`, comes out as `Error: Code `.
