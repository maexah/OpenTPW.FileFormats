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

### `FLYINGMESH(dir, model, count, x, y, z, speed)`

A swarm of animated models flying around the island. Only two parks use it:

| Park | Flyers |
| --- | --- |
| jungle | 10 `bfly_PINK` and 10 `bfly_YELL`, speed 1.5 |
| hallow | 50 `bat`, speed 2.5 |

**The last float is speed, not scale**, which is the obvious reading and the wrong one. It is
stored at `+0x28` on each flyer, and the per-flyer update multiplies it into the step:

```c
position += direction * flyer[0x28] * delta
```

Nothing in `FLYINGMESH` scales the model. Flying meshes are drawn at the size they were authored.

The speed is in the same per-tick units as `SPINSPEED` and the camera lag - the delta the game
multiplies by is counted in ticks of 25fps rather than in seconds - so 1.5 is 37.5 units a second
and 2.5 is 62.5.

**The three volume floats are the box's full size, not its half-extents.** Every random point the
original picks is

```c
coord = centre + extent * (random01 - 0.5)
```

from the constants `1/2^30` at `0x00702adc` and `0.5` at `0x00702ae0`. The centre is the island's
own position, unshifted (the offset constant at `0x00702d40` is zero). They are `200.0, 100.0,
200.0` in every use - 200 wide, 100 tall, 200 deep in the original's Y-up axes - so there is no
per-park variation.

The count is scaled by a detail percentage and clamped, so there is always at least one flyer and
never more than the script asked for. At full detail it is the script's number.

#### Flight

The swarm's tick builds flyers up over several frames rather than all at once: it makes one, then
keeps going with a seven-in-eight chance, so fifty bats take about six frames.

Each flyer's update (`FUN_005d9b50`) is the whole behaviour:

```c
wanted    = normalise( target - position )
direction = normalise( direction + (wanted - direction) * delta * 0.1 )
position += direction * speed * delta

if ( distanceSquared( position, target ) < 500 )
    target = randomPointInBox()
```

So: pick a random point in the box, lag toward it (never turning sharply), and on arrival - within
about 22 units, from the hard-coded `500.0` compared squared - pick another. There is no wander,
no boundary avoidance and no flocking. The box is respected because every destination is inside
it, not because anything pushes back at the edges.

Orientation is built with the flight direction as forward and a sideways axis of
`normalise(dz, 0, -dx)` - note the zero in the up slot, at `0x00702a38` - so a flyer pitches with
its climb and dive but never banks into a turn.

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

The bolt runs from ground level to 500 units up. Its base lands within fifty units of the island
(`random01 * 100 - 50`, from `0x00702c94`) and its top leans by up to ten more
(`random01 * 20 - 10`, from `0x00702c8c`). A strike also raises a flag the renderer flashes on,
and plays a thunder sound.

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
- Whether the flying-mesh `500.0` and the lightning bolt's 500-unit height being the same number
  is meaningful or a coincidence.
