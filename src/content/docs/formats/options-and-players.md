---
title: Options and Players (*.tcf, gms.dat)
---

Theme Park World saves its options and its players as small binary files in a `save` folder in the directory the game runs from. Every one of them is a version number followed by a fixed list of fields, written straight out of the game's memory: there are no names, tags, lengths or checksums, and a field is only known by where it sits.

The game reads and writes each file with the same routine, handed a flag that says which way the bytes go, so the field list is the same in both directions. Some of its reads carry a name - `"VideoCard"`, `"MovieVolume"` - but the names are only labels for the game's debug log, not part of the file.

All values are little-endian.

## Config.tcf

`save\Config.tcf` holds the options that belong to the machine rather than to a player. The game reads it as it starts, before anything else is set up, and writes it when the options screen's tick is clicked and whenever a player is saved (on *Select New Player* and on quitting).

The game writes the name as `Config.tcf`. The `safemode.bat` shipped beside the game (`md save`, then `copy safemode.tcf save\config.tcf`) spells it in lower case, which on Windows is the same file. `safemode.tcf` is a ready-made copy of this file.

| Offset | Size    | Description                                           |
| ------ | ------- | ----------------------------------------------------- |
| `0x00` | 4 bytes | Version - `1`                                         |
| `0x04` | 4 bytes | Rendering - `1` for the 3D card, `0` for software      |
| `0x08` | 4 bytes | Screen resolution - `0` 512 x 384, `1` 640 x 480, `2` 800 x 600 |
| `0x0C` | 4 bytes | Graphics quality - `0` low, `1` medium, `2` high      |
| `0x10` | 4 bytes | Video card - `0` primary, `1` secondary               |
| `0x14` | 1 byte  | Movie sound - `0` off, `1` on                         |
| `0x15` | 3 bytes | Padding (whatever was in memory; zero in practice)    |
| `0x18` | 4 bytes | Movie volume - `0` to `100`                           |
| `0x1C` | 4 bytes | Audio quality - `0` to `100`                          |

A version of `0` leaves every option at its default. The game checks nothing else about the version: any other value is read the same way.

The shipped `safemode.tcf` reads as software rendering, 640 x 480, low graphics, the primary video card, movie sound on at `100`, and audio quality `32`.

The sound effects, music and speech volumes, and the switches for the advisor, tutorial, pop-up help, confirmations, scrolling, right-click cancel and rotation, are not in this file - they belong to the player, in their `gms.dat`. The movie sound and volume are in both, and the player's copy is used once a player is chosen.

**Defaults** (with no file, or a version of `0`): 3D card rendering, 640 x 480, medium graphics, the primary video card, and movie sound on at the volume `data\sound.sam` gives as `DefaultVolume.MOVIE`, with the audio quality it gives as `SoundInfo.DEFAULTQUALITY`.

> The defaults for the sound options come from values that are zero in the executable and filled in at run time. `sound.sam`'s `DefaultVolume` and `SoundInfo` rows are the only ones that match them, by name; the code that reads them into place has not been traced.

## Players

Each player is a folder, `save\users\<slot><name>`: the slot's number, `1` to `4`, run straight into the player's name - `save\users\1Alexah`. There is no list of players anywhere else. As the game starts it creates `save`, `save\users` and `save\online` if they are missing, and takes every folder in `save\users` whose name is a digit from `1` to `4` followed by at least one character; a later folder for the same slot replaces an earlier one. A player's name is only ever in their folder's name.

Inside are `gms.dat`, a folder for each theme (`jungle`, `hallow`, `fantasy`, `space`) where that player's parks are saved, and the player's online mail files (`addrbook.dat`, `outbox.dat`). The game creates any missing theme folder as it starts. An *Instant Action* player is given a copy of each theme's `data\levels\<theme>\easymode.TPWI` in their theme folder when they are made; only `jungle` ships one.

Deleting a player removes their whole folder, skipping any name that starts with a dot.

## gms.dat

`gms.dat` holds a player's progress and their own options. It is written when the player is made, when a new *Full Simulation* player is given their first golden key, after every park save, and when the player is saved on *Select New Player* or on quitting. On those last two the game writes `Config.tcf` straight after it.

| Size     | Description                                                     |
| -------- | --------------------------------------------------------------- |
| 4 bytes  | Version - `12`                                                  |
| 4 bytes  | Player-wide record tickets, one byte for each of four records   |
| 2 bytes  | Two more player-wide tickets                                    |
| 4 bytes  | Golden tickets spent on rides                                   |
| 4 bytes  | Golden keys given outright                                      |
| 1 byte   | Mode - `1` *Instant Action*, `0` *Full Simulation*              |
| 1 byte   | Swearing filtered - `1` for a new player                        |
| 1 byte   | First park still to come - `1` for a new player, cleared when a park starts |
| 4 bytes  | Park count                                                      |
| varies   | Park records (see below), one after another                     |
| 39 bytes | The player's options (see below)                                |
| 4 bytes  | Ride count                                                      |
| 2 bytes  | Ride id, for each ride bought with golden tickets               |

A brand-new player's file is 68 bytes: everything zero except the mode, the swearing filter and the first-park flag, with no parks and no rides, and the options as they stood when the player was made.

**Park record**

| Size     | Description                                                                 |
| -------- | --------------------------------------------------------------------------- |
| 4 bytes  | Name length                                                                 |
| varies   | Name - the theme's folder name, single-byte characters, no terminator       |
| 6 bytes  | The park's six golden tickets, one byte each                                |
| 20 bytes | Four times: 1 byte - this park holds that player-wide record; 4 bytes - the record's value (meaningless while the byte is `0`) |
| 132 bytes | 33 times: 2 bytes of the park's chosen name, first half, then 2 bytes of the second half (UTF-16, each half ended by a `0`) |
| 1 byte   | The park has been renamed                                                   |
| 1 byte   | All research completed                                                      |

Records are only made once a player is picked - one for each theme - and the game writes them sorted by the bytes of their names. Two records with the same name stop the file loading at the second.

**Options**

| Size    | Description                                              |
| ------- | -------------------------------------------------------- |
| 1 byte  | Sound effects on                                         |
| 3 bytes | Padding                                                  |
| 4 bytes | Sound effects volume - `0` to `100`                      |
| 8 bytes | Music on, padding, volume                                |
| 8 bytes | Speech on, padding, volume                               |
| 8 bytes | Movie sound on, padding, volume                          |
| 1 byte  | Advisor on                                               |
| 1 byte  | Tutorial on                                              |
| 1 byte  | Pop-up help on                                           |
| 1 byte  | Confirmations on                                         |
| 1 byte  | Scroll with the right button (rather than push scrolling) |
| 1 byte  | Right mouse button cancels                               |
| 1 byte  | Rotate 90 degrees at a time (rather than smoothly)       |

These are the options as they stand whenever the file is written, and they replace the current options when the player is picked. The movie settings are also in `Config.tcf`; the player's copy wins.

**Golden keys.** The game counts a player's golden tickets: every nonzero player-wide ticket byte, plus every nonzero ticket byte of each park record whose theme has a `global.sam`. The player holds a key for every three tickets, plus the keys given outright. A new *Full Simulation* player is given one key outright when they first leave the player slots; an *Instant Action* player is given none, and can enter every park.

**Loading.** The game refuses only a file whose version is below `12`. A file that ends early, or has two parks with one name, stops loading where it fails. The game never checks for that, so the player keeps whatever was read and the rest stays at its defaults. A missing `gms.dat` still leaves the player in their slot.

## Open questions

- What "tcf" stands for. No expansion appears anywhere in the executable.
- `dialog.tcf`, read from the game's own directory rather than `save`, is not an options file: it holds developer settings (a debug flag word, a starting theme, a texture cache size), needs a version of `2` or higher, and the game never writes it.
