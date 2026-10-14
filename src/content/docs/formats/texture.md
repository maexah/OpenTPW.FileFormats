---
title: Texture (*.wct)
---

> This page is a work in progress.

**Header**

| Size    | Description                              |
| ------- | ---------------------------------------- |
| 1 byte  | Compression type: 0x12: zlib, 0x13: lzss |
| 1 byte  | Has an alpha channel - 1 when the alpha chunk is non-empty |
| 1 byte  | bit count                                |
| 1 byte  | version                                  |
| 2 bytes | width                                    |
| 2 bytes | height                                   |
| 2 bytes | Y channel quantization scale             |
| 2 bytes | Cb channel quantization scale            |
| 2 bytes | Cr channel quantization scale            |
| 2 bytes | A channel quantization scale             |
| 4 bytes | block 0 size                             |
| 4 bytes | block 1 size                             |
| 4 bytes | unknown                                  |

**Measured across the game's files** (OpenTPW Q353; every `.wct` member of the wads under `data/`, 8,375 of
them, dumped by name with `wadcat --dump`; the 41 loose `.wct` files on disk are not among them):

| Version | Files |
|---|---|
| 1 | 540 |
| 2 | 6,650 |
| 3 | 47 |
| 4 | 1,138 |

The Cb and Cr scales are unequal in 2,089 files: (30, 20) in 1,609, (44, 33) in 453, (10, 2) in 10, and a handful
of others. `ui.wad`'s `textures\ridestatbar.wct` (32 x 32, version 4, zlib, no alpha) has Y 6, Cb 10, Cr 2, as
`happygrad.wct` and the `stexture\` copies of both do. **The decode below does not give that file's colours**: the
original draws its ramp red, orange, yellow, green ((255, 14, 0) at the second texel, measured on its screen), where
this page's decode gives (107, 104, 0) at the same brightness. Which step is wrong is not known; the unequal scales
alone are not it, being common.

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
float floatA = 1.0f - ((float)header.CbChannelQuantizationScale * -0.25f);
float floatC = 1.0f - ((float)header.CrChannelQuantizationScale * -0.25f);
float floatE = (float)header.AChannelQuantizationScale + 1.0f;
```

4. Decompress

LZSS:

```pseudocode
result := []
while true:
  isDelta := GetBit( 1 )
  if isDelta:
    literalValue := GetBits( 8 )
    push literalValue to result
  else:
    offset := GetBits( 12 )
    length := GetBits( 7 ) + 1
    
    if offset == 0:
      break
    for 0..length:
      push result[ current - offset ] to result
```

5. Decode channels using quantization scales
Cb/Cr use size / 2

6. Convert YCbCr values to RGB
