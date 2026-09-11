---
title: Particle library (*.plb)
---

Every particle effect in the game - fireworks, steam, the sparkles round the golden key in the lobby,
the glints round a button under the pointer - comes from one file, `data\Particle\Tp2.plb`. Beside it,
`data\Particle\par_lib.h` is a C header naming each effect (`P_EFFECT_KeySparkle` is 97) and each
effector (`E_EFFECT_KeyAttr` is 4).

An **effect** is an emitter's settings: where particles appear, how they move, what they look like and
how long everything lasts. An **effector** is a point that pulls particles towards it, or pushes them
away, for a while.

## Layout

| Offset | Size | Description |
| --- | --- | --- |
| `0x00` | 4 bytes | Effect count - `105` |
| `0x04` | 4 bytes | Effect record size - `320` |
| `0x08` | 320 bytes each | Effects |
| after the effects | 4 bytes | Effector count - `20` |
| | 4 bytes | Effector record size - `104` |
| | 104 bytes each | Effectors |
| | 4 bytes | `33`; kept by the loader (clamped to 10-500) but not seen used |
| | 4 bytes | `1024`; kept by the loader but not seen used |

All values are little-endian. The game refuses records of any other size.

## Units

Everything is a whole number.

- **Distances** are in particle space. Effects are started at a position sixteen times that size, so
  an effect started at 50000 across has its emitter at 3125.
- **Times** are ticks of the particle system, which the game steps every 31 milliseconds.
- **Angles** are 4096 to a turn, except a particle's own rotation, which is 65536 to a turn.
- **Drag** is in 1024ths of the velocity lost each tick.

## Effect record

Starting an effect copies its record whole into an emitter, and the running emitter is read at the same
offsets. The fields below are identified by what the emitter code does with each one. Offsets missing
from the table are only used while an effect runs (links, counts, the bounding box from `0x128`) or
were not identified.

| Offset | Size | Description |
| --- | --- | --- |
| `0x04` | 1 byte | Group, given to each particle so an effector can act on one group only |
| `0x05` | 1 byte | Drawn on the screen rather than in the world |
| `0x0E` | 2 bytes | Emission: `0` emits at the rates at `0x68`; `1` emits rings that widen every tick |
| `0x10` | 2 bytes | Effect started where the emitter ends, `-1` for none (an effector when `0xBB` is set) |
| `0x12` | 2 bytes | Up to this much is added to each part of the emitter's velocity when it starts |
| `0x14` | 12 bytes | Position, overwritten when the effect starts |
| `0x20` | 4 bytes | Emitter lifetime, in ticks |
| `0x24` | 2 bytes | Emitter drag |
| `0x26` | 2 bytes | Endless: the lifetime never counts down, so the effect runs until it is stopped |
| `0x28` | 2 bytes | Emitter gravity, taken off its upward velocity each tick |
| `0x2C` | 12 bytes | Emitter velocity |
| `0x38` | 12 bytes | Particle starting velocity |
| `0x44` | 12 bytes | Spawn area: how far from the emitter particles appear on each axis |
| `0x50` | 2 bytes | Up to this much either way is added to each part of a particle's velocity; a ring's growth per tick |
| `0x52` | 2 bytes | Depth an on-screen particle is drawn at |
| `0x54` | 4 bytes | Starting angle round the orbit at `0xC4` |
| `0x58` | 4 bytes | Colour mode: `0` by age; bit 1 picks a random colour every tick; anything else picks one at birth |
| `0x5C` | 4 bytes | Radial speed: when not 0, particles fly off in a random direction at up to this speed instead |
| `0x60` | 4 bytes | Burst: particles emitted at once when the effect starts |
| `0x64` | 2 bytes | Most particles alive at once |
| `0x68` | 4 x 1 byte | Emission rate for each quarter of the emitter's life, first quarter first: that many particles a tick, or when negative one every that many ticks |
| `0x6C` | 1 byte | Particles start with the emitter's velocity added |
| `0x6D` | 1 byte | Particles start at a random rotation |
| `0x70` | 4 bytes | Draw flags - see [Drawing](#drawing) |
| `0x74` | 2 bytes | Particle size at birth |
| `0x76` | 2 bytes | The emitter bounces off the ground |
| `0x78` | 4 bytes | Particle lifetime; each particle gets up to a quarter as long again |
| `0x7C` | 4 bytes | Particle drag |
| `0x80` | 4 bytes | Particle gravity |
| `0x84` | 12 bytes | Jitter: up to this much either way is added to a particle's velocity every tick |
| `0x94` | 2 bytes | Sprite set: the bank (the number over 16) and the set in it (the rest) - see [Sprites](../sprites/) |
| `0x96` | 2 bytes | Frames of the sprite set played over a particle's life; `0` draws plain squares |
| `0x98` | 4 bytes | Effect started wherever a particle dies, `-1` for none |
| `0x9C` | 4 bytes | Effectors that act on every group leave these particles alone |
| `0xA0` | 2 bytes | Particles appear only above the emitter within the spawn area |
| `0xA2` | 2 bytes | Drawing scale about the emitter, in 1000ths |
| `0xA6` | 2 bytes | Particle size at death; the size changes steadily between the two |
| `0xA8` | 2 bytes | Particles appear on the edge of the spawn area rather than inside it |
| `0xAA` | 1 byte | Particles end when the emitter does |
| `0xAC` | 4 bytes | Rotation of the spawn area about the upright axis |
| `0xB0` | 4 bytes | Scale for a velocity set on the running emitter, in 1024ths |
| `0xB4` | 4 bytes | Effect started along with this one, `-1` for none (an effector when `0xBA` is set) |
| `0xB8` | 2 bytes | The linked effect is kept at this emitter's position |
| `0xBA` | 1 byte | The linked effect is an effector |
| `0xBB` | 1 byte | The end effect at `0x10` is an effector |
| `0xBC` | 2 bytes | The linked effect ends when this one does |
| `0xBE` | 2 bytes | Least a particle turns each tick |
| `0xC0` | 1 byte | Not scaled by the particle density setting |
| `0xC1` | 1 byte | Refused while the system is holding such effects back |
| `0xC2` | 1 byte | The spawn area is a box; otherwise it is an upright ellipse |
| `0xC4` | 2 bytes | Orbit: particles appear this far out, round a circle the emitter turns |
| `0xC6` | 2 bytes | Most a particle turns each tick |
| `0xC8` | 4 bytes | How far round the orbit the emitter moves each tick |
| `0xD8` | 16 x 4 bytes | Colours, `0xAARRGGBB`. With colour mode 0 a particle is born with the last and dies with the first |
| `0x118` | 16 bytes | Name, NUL-terminated; what follows the NUL is often left over from another name |

The particle density setting is `GameOptions.PARTICLEDENSITY` in `low.sam`, `med.sam` and `high.sam`
(500, 1000 and 1500). Unless `0xC0` is set, it scales the rates, the most-alive count and the burst
by `density / 1024` when the effect starts.

## Effector record

| Offset | Size | Description |
| --- | --- | --- |
| `0x04` | 4 bytes | Group acted on, `-1` for every group |
| `0x0B` | 1 byte | Bounces off the ground |
| `0x0C` | 12 bytes | Position, overwritten when it starts |
| `0x18` | 12 bytes | Velocity |
| `0x24` | 4 bytes | Drag |
| `0x28` | 4 bytes | Gravity |
| `0x2C` | 4 bytes | Radius for mode 2 (`0` is taken as `1`) |
| `0x30` | 4 bytes | Mode: `2` also ends particles that come within the radius |
| `0x34` | 12 bytes | Scatter: the point acted from moves by up to this much either way, afresh every tick |
| `0x40` | 4 bytes | Strength: positive pulls particles in, negative pushes them away |
| `0x45` | 1 byte | Acts on on-screen effects |
| `0x46` | 2 bytes | Effect started where it ends, `-1` for none (an effector when `0x49` is set) |
| `0x49` | 1 byte | The end effect is an effector |
| `0x4A` | 2 bytes | Endless |
| `0x4C` | 4 bytes | Lifetime, in ticks |
| `0x50` | 16 bytes | Name |

Each tick an effector adds `strength * d / (((dx² + dy² + dz²) >> 8) + 10)` to each particle's velocity,
where `d` is the particle's distance from the scattered point, quartered, one axis at a time.

## Drawing

An on-screen effect is drawn on the same 2048x1536 screen the interface is laid out on. The game
converts a particle at `(x, z)` in particle space to `((x - 3125) * 16) / 49` across and
`(z * 16 - 37500) / 37` down, and scales both by 1/1024 into -1 to 1 of the screen. So particle space
runs 6250 across and about 4687 down, with the middle of the screen at 3125 across. Up and down in the
world, `y`, is not used.

A particle's size over 64 is its height in 2048ths of that -1 to 1. Its width is the height times the
sprite's width over height, measured in the wider across units, so on a 4:3 screen every sprite is
stretched by a third across.

The draw flags at `0x70` choose the blending:

| Flags | Blend |
| --- | --- |
| `0x4` and `0x2000` | Added to the screen, scaled by alpha (source alpha, one) |
| `0x4` only | Added to the screen (one, one) |
| neither | Blended by alpha |

The sprite is multiplied by the particle's colour, alpha included.
