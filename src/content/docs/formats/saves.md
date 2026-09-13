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

| Size      | Description                                            |
| --------- | ------------------------------------------------------ |
| 4 bytes   | Version - see below. Not a magic number                 |
| 1 byte    | Padding                                                |
| 824 bytes | Copyright notice, UTF-16 - 412 characters              |
| 711 bytes | Padding                                                |

> The first four bytes are a **version, not a magic number**. The only file of this shape the game
> ships — `data/levels/jungle/Easymode.TPWI` — carries **400** (`90 01 00 00`), so a reader that
> requires `F4 01 00 00` (500) rejects it. Both values should be accepted.

The copyright notice is UTF-16, so its 824 bytes are **412 characters**; reading it as one byte per
character yields every other byte as a null. It begins at offset 5, after a single pad byte.

These sizes close exactly on the offset the next section begins at: `4 + 1 + 824 + 711 = 1540`
(`0x604`), which is where the file type sits.

**File info**

| Size    | Description                                     |
| ------- | ----------------------------------------------- |
| 4 bytes | File type - `00 01 22 19`                       |
| 1 byte  | File version - `85`                             |
| 1 byte  | Online flag - `00` for offline, `01` for online |
| 2 bytes | Padding                                         |

**Data (compressed using ZLIB)**

| Size     | Description                                                  |
| -------- | ------------------------------------------------------------ |
| 4 bytes  | Magic number - `BILZ`                                        |
| 4 bytes  | Uncompressed size - what the payload inflates to             |
| 4 bytes  | Size of this block, including these 28 header bytes           |
| 16 bytes | Unknown                                                      |

> Neither of those two values is a compressed length. In the shipped Jungle park the first is
> **1,608,309** — the exact size the ZLIB stream inflates to — and the second measures the block from
> the `BILZ` tag to the end of the file. Both are worth checking on load: one confirms the block
> reaches the end of the file, the other that the payload inflated to the size it claimed.

The header is 28 bytes **including the `BILZ` tag**, which is an easy four bytes to lose. In the
shipped park the tag sits at `0x60D`, so the ZLIB stream begins at `0x629` and continues to the end of
the file.
