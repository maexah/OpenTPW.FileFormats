---
title: Archives (*.wad)
---

The Bullfrog WAD file format contains compressed (or uncompressed) data for rides, textures, UI elements, and more.

The data within these files is typically compressed using [RefPack](http://wiki.niotso.org/RefPack) (see below).

### File Format

**File header**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | Magic number - "DWFB"                                                          |
| 4 bytes            | Version (`2` in all 312 installed archives)                                                                        |
| 64 bytes           | Padding                                                                        |
| 4 bytes            | File count                                                                     |
| 4 bytes            | Filename string-block offset (absolute)                                                               |
| 4 bytes            | Filename string-block length                                                               |
| 4 bytes            | Padding                                                                        |

**For each file**

| Size               | Description                                                                    |
|--------------------|--------------------------------------------------------------------------------|
| 4 bytes            | Unknown                                                                        |
| 4 bytes            | Filename offset                                                                |
| 4 bytes            | Filename length, including terminal NUL                                                                |
| 4 bytes            | Data offset                                                                    |
| 4 bytes            | File length                                                                    |
| 4 bytes            | Compression type - "4" for [RefPack](http://wiki.niotso.org/RefPack), "0" for uncompressed|
| 4 bytes            | Decompressed size for compression 4; zero for uncompressed entries                                                              |
| 12 bytes           | Padding                                                                        |

**Directories**

Occasionally, a file's filename may begin with the name of a directory separated using the `\` character. This should
be treated as entering a directory, meaning that file and any further files all belong to that folder.

For example:

| File #    | Filename                  |
|-----------|---------------------------|
| 0         | hello.sam                 |
| 1         | hello.RSE                 |
| 2         | textures\\hello.wct       |
| 3         | hello2.wct                |
| 4         | stexture\\hello.wct       |
| 5         | hello2.wct                |

Here:
- Files 0 and 1 both belong to the root of the archive
- Files 2 and 3 exist in the `textures` directory
- Files 4 and 5 belong to the `stexture` directory.
 
Despite file 3 and file 5 both being called `hello2.wct`, they do not overlap - they belong in different directories - and therefore they do not conflict. 

### Corpus validation (2026-10-03)

Observed across all 312 WADs in the installed Theme Park World data: 13,392 entries,
6,809 RefPack-compressed and 6,583 uncompressed (80 of the latter empty). An independent
Python byte survey and OpenTPW's decoder both traversed the complete corpus.

The fixed header is 88 bytes; the entry table immediately follows, 40 bytes per entry.
Header offsets `0x4c` and `0x50` name the **filename string block**, not that entry table.
Every filename span lies inside the block and ends in NUL. In this corpus the string block
ends at EOF and its length is the sum of filename lengths. Offsets are absolute bytes from
the archive start. The first entry dword is nonzero in every entry; its meaning remains unknown.

Every compressed span starts `10 FB`, followed by a **three-byte big-endian decompressed
length**, equal to the enclosing WAD entry's decompressed size. Every stream terminates
with a RefPack stop command, with no trailing bytes, and expands to that length. Backreferences
may overlap their output: copying must allow each newly produced byte to be read by a later
byte of the same command. The survey counted 107,365 overlapping matches.

These are observations of this installed corpus, not guarantees about other editions or
unseen RefPack header variants. OpenTPW accepts compression types 0 and 4 and this `10 FB`
variant. It rejects truncated spans/commands, invalid backreferences, inconsistent expanded
lengths, missing stop commands and trailing compressed bytes. It does not interpret unknown
header or entry words, nor require the uncompressed size field to equal the stored length.

The independent Python decoder and OpenTPW produced the same decoded-corpus SHA256:
`6cbd6c2291cdd2731b19898fc4535ce195ac62cc2b17e18ecc0d76f140ab1077`.
For reproduction: make one UTF-8 record per entry: lowercase relative WAD path (from `data`,
`/` separators), NUL, lowercase internal path (`/` separators), NUL, lowercase SHA256 hex of
the decoded bytes, LF. Sort records ordinally, concatenate and SHA256 the result. This hash
identifies the surveyed corpus; other editions need not have the same hash.
