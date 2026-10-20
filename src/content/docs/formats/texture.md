---
title: Texture (*.wct)
---

> This page is a work in progress.

**Header**

| Size    | Description                              |
| ------- | ---------------------------------------- |
| 1 byte  | Compression type: 0x12: zlib, 0x13: lzss |
| 1 byte  | The graded-alpha flag: over an alpha chunk, graded rather than cut out (below) |
| 1 byte  | bit count: 32 when there is an alpha chunk, 24 when not |
| 1 byte  | version                                  |
| 2 bytes | width                                    |
| 2 bytes | height                                   |
| 2 bytes | Y channel quantization scale             |
| 2 bytes | Cr channel quantization scale (offset 10) |
| 2 bytes | Cb channel quantization scale (offset 12) |
| 2 bytes | A channel quantization scale             |
| 4 bytes | block 0 size                             |
| 4 bytes | block 1 size                             |
| 4 bytes | unknown                                  |

**A shadow's silhouette.** `data\generic\shadow\alphkid.wct`, a loose file: type `0x13`, header byte 1 = 1 (graded), 32
bits, 64 x 64. Its colour is white throughout (252 and 253 on every channel, 2,048 texels each) and its alpha a
child's silhouette, a head over a body: 2,433 texels 0, 1,412 at 16 or more (rows 2 to 62, columns 13 to 47), the most
255. The game lays it on the ground under every person at the medium detail tier ([SAM](../sam/), `SPRITESHADOW`).

**Measured across the game's files** (OpenTPW Q353; every `.wct` member of the wads under `data/`, 8,375 of
them, dumped by name with `wadcat --dump`; the 41 loose `.wct` files on disk are not among them):

| Version | Files |
|---|---|
| 1 | 540 |
| 2 | 6,650 |
| 3 | 47 |
| 4 | 1,138 |

The two chroma scales are unequal in 2,089 files: (30, 20) in 1,609, (44, 33) in 453, (10, 2) in 10, and a handful
of others, offset 10 first. **Offset 10 is Cr's scale and offset 12 is Cb's**: the original hands them to its decoder
as Y, offset 12, offset 10, and decodes the second channel (the one turned into blue) with the first of those (OpenTPW
`docs/exe/textures.md`, Q354). Read the other way round, `ui.wad`'s `textures\ridestatbar.wct` (32 x 32, version 4,
zlib, no alpha; Y 6, offset 10 = 10, offset 12 = 2) decodes olive to yellow; read this way, red to green, as the
original draws it. Of `ui.wad`'s 921 `.wct`, the 473 whose two scales are equal decode the same either way.

**The bit count says whether there is an alpha chunk; the byte before it does not always** (OpenTPW Q355; every `.wct`
member of the 312 wads, dumped one folder a wad, 8,375, and the 21 loose files' names: 8,396). The bit count is 32 in every file with an alpha
chunk (block 1 size over 0) and 24 in every file without one. The byte at offset 1 disagrees in 237: 236 have it 0
with a 32-bit count and an alpha chunk (3 of type `0x12`: `stexture\s_top.wct`, `smallguage.wct` twice; 233 of type `0x13`),
and `stexture\b_NEWwin.wct` has it 1 with a 24-bit count and no alpha chunk. The original reads the alpha chunk by the bit
count, and takes the byte at offset 1 as the choice between a graded alpha (set) and a cut-out one (0) for the
image it draws (OpenTPW `docs/exe/textures.md`, "The loader"; Q358): the 236 draw cut out. No header scale has its top
bit set.

| Type | Colour chunk | Files |
|---|---|---|
| `0x12` | starts `BILZ` (zlib) | 986 |
| `0x12` | LZSS | 3 (the 8 x 8 `stexture` copies of `Bowler_Btie.wct`, `Fan_stalk.wct`, `s_disc.wct`) |
| `0x13` | LZSS | 7,407 |

## Decoding Image

1. Get aligned width / height

```cpp
uint32_t GetAlignedSize(uint32_t size)
{
 uint32_t value = (size - 1);
 uint32_t n = 1;
 for (; (value >>= 1) != 0;){
  ++n;
 }
 if (n < 3){
  n = 3;
 }
 return 1 << n;
}
```

2. Calc max size (largest of aligned width / height)
3. Set up quantization values

```cpp
float float8 = 1.0f - ((float)header.YChannelQuantizationScale * -0.5f);
float floatA = 1.0f - ((float)header.CbChannelQuantizationScale * -0.25f); // offset 12, the second channel's
float floatC = 1.0f - ((float)header.CrChannelQuantizationScale * -0.25f); // offset 10, the third channel's
float floatE = (float)header.AChannelQuantizationScale + 1.0f;
```

4. Decompress

LZSS:

```pseudocode
result := []
while true:
  isMatch := GetBits( 1 )
  if isMatch == 0:                  # a clear bit is the literal
    literalValue := GetBits( 8 )
    push literalValue to result
  else:
    offset := GetBits( 12 )
    if offset == 0:                 # tested before the length is read
      break
    length := GetBits( 7 ) + 1
    for 0..length:
      push result[ current - offset ] to result
```

Bits are taken least significant first from consecutive little-endian 16-bit words. A **clear** bit
introduces a literal byte and a **set** bit a back-reference; 7,406 of the 7,407 `0x13` files decode that
way (upstream look 2026-10-19, `verify-images.md` L4). A copy may run past the bytes written so far, which
repeats the last `offset` bytes.

5. Decode channels using quantization scales
Cb/Cr use size / 2

Type `0x12` (the original's zlib codec, OpenTPW `docs/exe/textures.md`): the decompressed colour chunk is the values
of Y's `n x n` plane (`n` the larger aligned side), then Cb's `n/2 x n/2`, then Cr's, row by row; the alpha chunk is
the alpha's `n x n`. A value is a signed byte, or the byte `0x80` and then a little-endian int16. Each row's first
half is its smooth values and its second its detail, and each column's the same, for an inverse plain D4.

6. Convert YCbCr values to RGB
