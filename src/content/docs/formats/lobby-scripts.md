---
title: Lobby scripts (*.txt)
---

The lobby - the island-selection screen the game opens on - is configured by plain-text files in
`lobby.wad`. There is one for the lobby itself and one per park:

| File | Describes |
| --- | --- |
| `lobby.txt` | The camera, the globe, and where each island sits |
| `jungle.txt` | Lost Kingdom |
| `fantasy.txt` | Wonder Land |
| `hallow.txt` | Halloween World |
| `space.txt` | Space Zone |

They are short. `hallow.txt` in full:

```
ISLAND(2,"data\lobby\terrain","hal_isle","hal_gate","Halloween",180.0,38.0)
FLYINGMESH("data\lobby\terrain","bat",50,200.0,100.0,200.0,2.5)
LIGHTNING(63)
RAINY(1)
SKYCOLOUR(4,44,12)
```

## The vocabulary is closed

The parser (`FUN_005e3210`) splits the file on newlines and `strncmp`s each line against a table
of twelve keywords, which sit contiguously in the executable at `0x00774ce0`:

```
ISLANDCAMERAPOSITION(   GLOBERADIUSIN(   GLOBERADIUSOUT(   VERTICALOFFSET(
SPINRADIUS(   SPINSPEED(   ISLANDFOV(   SKYCOLOUR(   RAINY(   LIGHTNING(
FLYINGMESH(   ISLAND(
```

That table is the entire vocabulary - there is nothing else a lobby script can say. Anything else
the lobby does is a model animation or is hard-coded. Worth stating outright, because the shipped
scripts use only seven of the twelve and it is otherwise tempting to assume the missing effects
are configured somewhere else.

Note that the keywords are matched **including** the opening bracket. `ISLAND` alone would also
match `ISLANDFOV` and `ISLANDCAMERAPOSITION`.

Having found a keyword the parser scans forward to `(` and then reads field by field, stopping
each field at `,`, `"` or `)` as appropriate. Quoted fields are paths and names; everything else
is a number.

## Per-park directives

### `ISLAND(index, dir, isle, gate, name, heading, cameraHeight)`

The park itself. `index` is its place in the lobby's running order, `isle` and `gate` are `.md2`
names under `dir`, `heading` is where the island sits around the lobby globe in degrees, and
`cameraHeight` is how far above the island the camera looks (12.5 for the jungle against 38 for
hallow).

`name` is the park name, but only the jungle's is what the player is shown - the other three are
`"Fantasy"`, `"Halloween"` and `"Space"` where the game displays Wonder Land, Halloween World and
Space Zone. The displayed names are not in the shipped data at all.

### `FLYINGMESH(dir, model, count, x, y, z, scale)`

A swarm of animated models flying around the island. The parser passes the count, the model, the
island's own position, the three floats as a volume to wander, the scale, and a hard-coded
`500.0` to the spawner.

Only two parks use it:

| Park | Flyers |
| --- | --- |
| jungle | 10 `bfly_PINK` and 10 `bfly_YELL`, scale 1.5 |
| hallow | 50 `bat`, scale 2.5 |

The three volume floats are `200.0, 100.0, 200.0` in every use, so there is no per-park variation
in them, and they are in the original's globe space - islands there sit on a sphere of radius 475.

### `SKYCOLOUR(r, g, b)`

Three 0-255 components, stored as 16.16 fixed point. All four parks set one: jungle
`243,203,191`, fantasy `5,170,255`, hallow `4,44,12`, space `125,8,8`.

### `RAINY(n)`

A rain level. Only hallow sets it, to 1.

### `LIGHTNING(n)`

**A probability mask, not a count or a period.** Each frame the lobby tick draws a random number
and strikes when

```c
(n & random) == 1
```

A strike therefore needs bit 0 of `n` set and every other bit of the masked value clear, so the
chance per frame is `1 / 2^(bits set in n)` - and **never**, if `n` is even. Only hallow sets it,
to 63, which is six bits: one frame in sixty-four, or about one strike every two and a half
seconds at the 25fps the rest of the game's data assumes.

The bolt runs from ground level to 500 units up, its top offset from its base by a small random
lean, somewhere near the island. A strike also raises a flag the renderer flashes on, and plays a
thunder sound.

## Lobby-wide directives

These appear only in `lobby.txt`:

| Directive | Shipped value | Meaning |
| --- | --- | --- |
| `ISLANDCAMERAPOSITION(i, x, y)` | ten entries | Where island `i` sits. Only the first four are used. |
| `SPINRADIUS(n)` | 70 | The camera's distance from the island it is orbiting |
| `SPINSPEED(n)` | 0.02 | Orbit speed, per frame |
| `VERTICALOFFSET(n)` | 20 | How far above the island the camera orbits |
| `GLOBERADIUSOUT(n)` | 475 | Camera distance from the globe, zoomed out |
| `GLOBERADIUSIN(n)` | 200 | Camera distance from the globe, zoomed in |
| `ISLANDFOV(n)` | 100 | Field of view. Not degrees - see below. |

Selecting a park in the original spins the globe until that island's `ISLAND()` heading faces the
camera, then pulls in from `GLOBERADIUSOUT` to `GLOBERADIUSIN`.

## How the weather is scoped

Rain, lightning and sky colour are **not** properties of a place. The lobby tick works out which
island is nearest the camera and then, every frame, copies that island's `SKYCOLOUR` into the
renderer's sky colour, its `RAINY` value into a single global rain level, and rolls its
`LIGHTNING` mask. There is one sky, one rain system and one bolt for the whole lobby; selecting a
park is what changes them.

## Open questions

- `ISLANDFOV(100)` is not a vertical field of view in degrees - read that way the islands become
  specks in a bowed horizon. It means something else, or reaches the projection another way.
- `SPINSPEED(0.02)` read as radians per frame is half a radian a second at 25fps, which is much
  brisker than the lobby appears to turn.
- The float globals the lightning bolt picks its ground position and lean from.
- What the hard-coded `500.0` passed to the flying-mesh spawner is (the same constant is the
  bolt's height).
