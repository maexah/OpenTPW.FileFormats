---
title: Instruction Set
---

## NOP

`NOP` - This performs no operation, essentially wasting 1 cycle.

### Operands

None

## CRIT_LOCK

`CRIT_LOCK` - Locks a ride, preventing visitors from accessing the ride until unlocked.

### Operands

None

## CRIT_UNLOCK

`CRIT_UNLOCK` - Releases the lock `CRIT_LOCK` took, and **gives up the rest of the script's turn**.

The second half is easy to miss and is load-bearing: a script whose main loop contains no `ENDSLICE`
yields only because its `CRIT_UNLOCK` does, so a reader that treats this as clearing a flag and nothing
else will have such a script spin until its instruction budget runs out.

### Operands

None

## COPY

`COPY <dest> <source>` - Copy a value from one variable to another.

`COPY <dest> <value>` - Set a variable's value.

### Operands

`<dest>` - The destination for the instruction's result.

`<value>` - The value to copy.

`<source>` - The source containing the value to copy.

## SETLV

`SETLV <unknown>` - Unknown

### Operands

Unknown

## SUB

`SUB <value> <value> <dest>` - Subtract one value from another.

`SUB <source> <value> <dest>` - Subtract a value from a variable.

`SUB <value> <source> <dest>` - Subtract variable's value from a value.

`SUB <source> <source> <dest>` - Subtract one variable from another.

Upon performing this calculation, the relevant flags will be set based on the calculation's result.

### Operands

`<dest>` - The destination for the instruction's result.

`<value>` - The value to subtract.

`<source>` - The source containing the value to subtract.

## ENDSLICE

`ENDSLICE` - Unknown

### Operands

Unknown

## GETTIME

`GETTIME <dest>` - Gets the time that the ride has been alive for.

### Operands

`<dest>` - The destination for the instruction's result.

## ADDOBJ

`ADDOBJ <type> <parameter> <id> <slot>` - Add an object of a specific type to the ride.

### Operands

`<type>` - The type of object to add.

`<parameter>` - Unknown

`<id>` - The ID of the object to add.

`<slot>` - The 'slot' within the current ride to add the new ride.

## ADDOBJ_EXT

`ADDOBJ_EXT <unknown1> <unknown2> <unknown3> <unknown4> <unknown5>` - Unknown

### Operands

Unknown

## KILLOBJ

`KILLOBJ <slot>` - Remove an object of a specific slot type.

### Operands

`<slot>` - The slot of the desired object.

## FADEOBJ

`FADEOBJ <slot>` - Fade out an object, and then remove it.

### Operands

`<slot>` - The slot of the desired object.

## SETOBJPARAM

`SETOBJPARAM <slot> <parameter> <value>` - Set a sub-object's parameters.

### Operands

`<slot>` - The slot of the desired object.

`<parameter>` - The parameter to change.

`<value>` - The desired value of the parameter.

## EVENT

`EVENT <type> <unknown> <event>` - Trigger an in-game event.

### Operands

`<type>` - The type of the event.

`<unknown>` - Unknown

`<event>` - The event?

## EVENT_EXT

`EVENT_EXT <unknown1> <unknown2> <unknown3> <unknown4>` - Unknown

### Operands

Unknown

## FLUSHANIM

`FLUSHANIM` - Stop all active animations

### Operands

None

## TRIGANIM

`TRIGANIM <role> <entry> <dest>` - Start a one-shot animation on channel 0, and answer how long it runs.

### Operands

`<role>` - Which of the model's twelve animation roles to play. A role is not a file: the twelve are lettered `C D I L S M E U W B R O`, and the letter names the clips sitting beside the model (`<stem><letter><n>.md2`). 12 means "no animation".

`<entry>` - Which clip within that role, numbered from 0.

`<dest>` - Where the length in milliseconds is written. Most shipped uses name a literal here, which leaves the answer in the result register instead.

The length is the clip's own duration less 300ms, floored at 300 with a **signed** comparison - so a model that has no such clip answers 300. The instruction also arms the deadline `WAIT4ANIM` waits on, and stamps the looping key with `0xFFFF`, which is why a `LOOPANIM` after a trigger always reads as a change. `TRIGANIM_CH` does the same on a chosen channel but does **not** stamp that key.

## WAITANIM

`WAITANIM <type> <unknown>` - Start playing an animation, and wait for it to end before continuing.

### Operands

`<type>` - The type of animation to play

`<unknown>` - Unknown

## LOOPANIM

`LOOPANIM <type> <unknown>` - Start playing an animation on loop

### Operands

`<type>` - The type of animation to play

`<unknown>` - Unknown

## TRIGWAITANIM

`TRIGWAITANIM <unknown1> <unknown2> <unknown3>` - Unknown

### Operands

Unknown

## GETANIM

`GETANIM <dest>` - Ask which animation role channel 0 is playing.

### Operands

`<dest>` - Where the answer is written.

This is `GETANIM_CH` with the channel fixed at 0; see there for what the answer means. **No shipped script uses it** - all 15 uses in the game are of the `_CH` form.

## TRIGANIMSPEED

`TRIGANIMSPEED <unknown1> <unknown2> <unknonw3> <unknown4>` - Unknown

### Operands

Unknown

## FLUSHANIM_CH

`FLUSHANIM_CH <unknown>` - Unknown

### Operands

Unknown

## TRIGANIM_CH

`TRIGANIM_CH <role> <entry> <dest> <channel>` - Start a one-shot animation on a chosen channel.

### Operands

`<role>` - As `TRIGANIM`.

`<entry>` - As `TRIGANIM`.

`<dest>` - As `TRIGANIM`.

`<channel>` - Which animation player to run it on, counted from 0. Resolved like any other value, so a variable may name it.

A model's channel count is not in the model - it is an argument to the model loader, taken for a placed thing from its item description's `UsageInfo.NumSimultAnims`. The shipped data agrees exactly: the Jungle Spray, Hyenas, Frushy, Squirtem and Marsmoon declare 3 and use channels up to 2, and the Totem declares 4 and uses up to 3.

This is instruction-for-instruction `TRIGANIM` apart from the channel, with **one** difference: `TRIGANIM` ends through a shared tail that writes both the `WAIT4ANIM` deadline and the looping key, while `TRIGANIM_CH` ends inline and writes only the deadline. It is the busiest of the family - 63 uses across 6 scripts.

## WAITANIM_CH

`WAITANIM_CH <unknown1> <unknown2> <unknown3>` - Unknown

### Operands

Unknown

## LOOPANIM_CH

`LOOPANIM_CH <role> <entry> <channel>` - Start an animation looping on a chosen channel.

### Operands

`<role>` - As `TRIGANIM`.

`<entry>` - As `TRIGANIM`.

`<channel>` - Which animation player to run it on. **Taken raw**: unlike `TRIGANIM_CH` and `GETANIM_CH`, this operand gets no variable tag test, so a variable arrives as its tagged word and names no channel at all.

It differs from `LOOPANIM` in two further ways: it has no "already looping" early exit, and it never writes the looping key. It does still clear the `WAIT4ANIM` deadline, because a loop never finishes.

Used **once** in the whole game: space's `Gates.RSE` opens with `LOOPANIM_CH 5, 2, 1` at word 2, before it tests anything, so that park's gate idles on a channel of its own from the moment it loads.

## TRIGWAITANIM_CH

`TRIGWAITANIM_CH <unknown1> <unknown2> <unknown3> <unknown4>` - Unknown

### Operands

Unknown

## GETANIM_CH

`GETANIM_CH <dest> <channel>` - Ask which animation role a channel is playing.

### Operands

`<dest>` - Where the answer is written. Note the order: the destination comes **first** here, the reverse of the triggers.

`<channel>` - Which animation player to ask about. Resolved like any other value.

The answer is the role the channel is running, **or -1 once the clip has finished**. The engine reads the player's flag word and overrides the role with -1 when it carries `0x4`, which is the bit set when a finished clip with nothing queued is parked on its last frame. An idle channel answers 12, the "no animation" sentinel.

Those three answers are what makes it usable as a "has it finished?" test, and the sideshows use it as exactly that: the Jungle Spray checks each occupied lane with `GETANIM_CH 0, <lane>` and branches away on a positive answer and on zero, so **-1 is the only answer that lets a rider off**.

## RAND

`RAND <dest> <max value>` - Generate a random number.

### Operands

`<dest>` - The destination for the instruction's result.

`<max value>` - The highest value to generate.

## JSR

`JSR <subroutine>` - Jump to a subroutine.

### Operands

`<subroutine>` - The destination subroutine to jump to.

## RETURN

`RETURN` - Return to the previous subroutine (after the last `JSR`).

### Operands

None

## BRANCH

`BRANCH <location>` - Branch to another location.

### Operands

`<location>` - The branch to execute.

## BRANCH_Z

`BRANCH_Z <location>` - Branch to another location if the "zero" flag has been set.

### Operands

`<location>` - The branch to execute.

## BRANCH_NZ

`BRANCH_NZ <location>` - Branch to another location if the "not zero" flag has been set.

### Operands

`<location>` - The branch to execute.

## BRANCH_NV

`BRANCH_NV <location>` - Branch to another location if the "negative value" flag has been set.

### Operands

`<location>` - The branch to execute.

## BRANCH_PV

`BRANCH_PV <location>` - Branch to another location if the "positive value" flag has been set.

### Operands

`<location>` - The branch to execute.

## DBGMSG

`DBGMSG <unknown>` - Unknown

### Operands

Unknown

## NAME

`NAME <value>` - Set the current script's name (likely used for debugging purposes).

### Operands

`<value>` - The new name for the script.

## TEST

`TEST <value>` - Set flags depending on the value given.

This instruction will modify all flags, which can then be used to execute different branches depending on `<value>`.

### Operands

`<value>` - The value to test.

## CMP

`CMP <value / variable> <value / variable>` - Compare two values (using a bitwise AND), and set any flags according to the result.

Upon performing this calculation, the relevant flags will be set based on the calculation's result.

### Operands

`<value / variable>` - A value or variable to compare.

## PUSH

`PUSH <value>` - Push a value to the stack.

### Operands

`<value>` - The value to push onto the stack.

## POP

`POP <dest>` - Pop a value from the stack into a variable.

### Operands

One - the destination the popped value is written into. A reader that treats `POP` as taking none will
swallow the following opcode word as data and decode the rest of the script wrongly.

## HUSH

`HUSH <unknown>` - Unknown

### Operands

`<unknown>` - Unknown

## HOP

`HOP <unknown>` - Unknown

### Operands

`<unknown>` - Unknown

## WAIT

`WAIT <time>` - Wait for a specified period of time.

### Operands

`<time>` - The length of time to wait for (cycles / slices / milliseconds?)

## WAITABS

`WAITABS <unknown>` - Unknown

### Operands

Unknown

## WAIT4ANIM

`WAIT4ANIM` - Wait for all of the currently playing animations to finish.

### Operands

None

## ADD

`ADD <source> <value>` - Add a value to the value of a variable.

Upon performing this calculation, the relevant flags will be set based on the calculation's result.

### Operands

`<source>` - The source containing the value to add, and the destination for the result.

`<value>` - The value to add.

## MULT

`MULT <unknown1> <unknown2> <unknown3>` - Multiply values (not used in any existing rides)

Upon performing this calculation, the relevant flags will be set based on the calculation's result.

### Operands

Unknown

## DIV

`DIV <value> <value> <dest>` - Divide one value by another.

`DIV <source> <value> <dest>` - Divide a variable by a value.

`DIV <value> <source> <dest>` - Divide a value by a variable's value.

`DIV <source> <source> <dest>` - Divide one variable by another.

Upon performing this calculation, the relevant flags will be set based on the calculation's result.

### Operands

`<dest>` - The destination for the instruction's result.

`<value>` - The value to divide.

`<source>` - The source containing the value to divide.

## MOD

`MOD <value> <value> <dest>` - Get the remainder of a division of one value from another.

`MOD <source> <value> <dest>` - Get the remainder of a division of a variable by a value.

`MOD <value> <source> <dest>` - Get the remainder of a division of a value by a variable's value.

`MOD <source> <source> <dest>` - Get the remainder of a division of one variable by another.

Upon performing this calculation, the relevant flags will be set based on the calculation's result.

### Operands

`<dest>` - The destination for the instruction's result.

`<value>` - The value to perform modulo on.

`<source>` - The source containing the value to perform modulo on.

## TURBO

`TURBO <value>` - Unknown

### Operands

`<value>` - Unknown [0..1]

## END

`END` - Unknown

### Operands

None

## TOUR

`TOUR <command> <params>` - Call a tour ride command (with parameter if applicable)

### Operands

`<command>` - The command to use

`<params>` - The parameter(s) to give to the command

#### Commands

- `1`: Set style of ride

- `4`: Let off visitor

- `3`: Let on visitor

- `5`: Number of cars

- `6`: Unused ("Unknown tour ride command")

- `7`: Unused ("Unknown tour ride command")

- `8`: Set time to fly for (in milliseconds)

- `9`: Set ride wear status
  
  - `1`: Ok
  
  - `2`: Worn
  
  - `3`: Breaking down
  
  - `4`: Condemned

- `10`: Check if there is a car in the station (sets flags)

- `11`: Activate launch / start flight

- `12`: Set whether ride uses runway type of approach (0/1)

- `14`: Set whether ride is open (0/1)

- `15`: Check if there is a broken car in the station (sets flags)

- `16`: Set whether the ride is running (0/1)

- `17`: Set size of car (1024 == 1)

## BUMP

`BUMP <command> <params>` - Set bumper ride properties, and set flags where appropriate.

### Operands

`<command>` - The ID of the property being set

`<params>` - The value to assign to the property

#### Commands

- `1` (`BUMP_PEEPON`): Add visitor to ride

- `2` (`BUMP_PEEPOFF`): Get visitor from ride

- `3` (`BUMP_STARTRACE`): Start the race

- `4` (`BUMP_LAUNCHCAR`): Launch car

- `5` (`BUMP_ISTRACKVALID`): Get whether the track is valid (sets flags)

- `6` (`BUMP_CLOSERIDE`): Close the ride

- `7` (`BUMP_OPENRIDE`): Open the ride

- `8` (`BUMP_SETBROKEN`): Set whether the ride is broken (0/1)

- `9` (Unnamed?): Set whether the ride is worn or not

- `10` (`BUMP_HALTRIDE`): Eject all visitors from the ride

- `11` (`BUMP_CARSONRIDE`): Get whether there are cars on the ride (0/1)

- `12`: Unknown

- `13`: Unknown

- `14` (`BUMP_SETLAPS`): Set the number of laps

- `15`: Unused ("Unknown bumper ride command")

- `16`: Unknown

- `17` (`BUMP_WATERCLOSED`): Set whether water is flowing (0 - flowing / 1 - not flowing)

## COAST

`COAST <id> <value>` - Set coaster ride properties, and set flags where appropriate.

### Operands

`<id>` - The ID of the property being set

`<value>` - The value to assign to the property

#### IDs

- `COAST_ADDPEEP` - Let on visitor

- `COAST_GETQUEUE` - Gets the number of people in the ride's queue

- `COAST_GETPEEP` - Gets the number of people on the ride

- `COAST_SETBROKE` - Set's the coaster's broken state

- `COAST_SETCLOSED` - Sets the coaster's open/closed state

- `COAST_SETCAPACITY` - Sets the coaster's capacity

- `COAST_SETWORN` - Sets the coaster's wear rating

- `COAST_INITIALISE` - Initialises the coaster

## ADDHEAD

`ADDHEAD <visitor ID>` - Add a visitor (head only) to the ride.

### Operands

`<visitor ID>` - The ID of the visitor to add.

## DELHEAD

`DELHEAD <visitor ID>` - Remove a visitor from the ride.

### Operands

`<visitor ID>` - The ID of the visitor to remove.

## LIMBO

`LIMBO <visitor ID> <unknown>` - Send a visitor into limbo.

### Operands

`<visitor ID>` - The ID of the visitor to send into limbo.

`<unknown>` - Unknown (possibly related to `LIMBOSPACE`, but may also be duration)

## UNLIMBO

`UNLIMBO <visitor ID>` - Remove a visitor from limbo.

### Operands

`<visitor ID>` - The ID of the visitor to remove from limbo.

## FORCEUNLIMBO

`FORCEUNLIMBO <visitor ID>` - Forcefully remove a visitor from limbo.

### Operands

`<visitor ID>` - The ID of the visitor to remove from limbo.

## INLIMBO

`INLIMBO <unknown>` - Unknown

### Operands

`<unknown>` - Unknown.

## LIMBOSPACE

`LIMBOSPACE <unknown>` - Unknown

### Operands

`<unknown>` - Unknown.

## SPAWNCHILD

`SPAWNCHILD <file name>` - Add a child script to the current script (max. 1 child)

### Operands

`<file name>` - The path to the desired script.

## SPAWNSOUND

`SPAWNSOUND <file name>` - Add a child script to the current script (max. 1 child) - likely somewhat different from `SPAWNCHILD`, but unknown.

### Operands

`<file name>` - The path to the desired script.

## REMOVECHILD

`REMOVECHILD` - Remove an existing child script.

### Operands

None

## SETVARINCHILD

`SETVARINCHILD <variable ID> <value>` - Set the value of a variable in the current script's child.

### Operands

`<variable ID>` - The ID of the desired variable.

`<value>` - The desired value to set.

## GETVARINCHILD

`GETVARINCHILD <dest> <variable ID>` - Get the value of a variable in the current script's child.

### Operands

`<variable ID>` - The ID of the desired variable.

`<dest>` - The destination for the value of the desired variable.

## SETVARINPARENT

`SETVARINPARENT <variable ID> <value>` - Set the value of a variable in the current script's parent.

### Operands

`<variable ID>` - The ID of the desired variable.

`<value>` - The desired value to set.

## GETVARINPARENT

`GETVARINPARENT <dest> <variable ID>` - Get the value of a variable in the current script's parent.

### Operands

`<variable ID>` - The ID of the desired variable.

`<dest>` - The destination for the value of the desired variable.

## BOUNCESETNODE

`BOUNCESETNODE <node>` - Set the base that slot node numbers are counted from. A visitor placed by
`BOUNCE` is given the node `<node> + slot index`.

The two setters are named the opposite way round to what they do: this one sets the node base, and
`BOUNCESETBASE` writes an unrelated field.

Its handler performs no variable-tag test, so the operand is stored **raw** - a variable operand would
be stored as its tagged word rather than as its value. Every shipped use passes a literal, so the two
readings cannot differ in practice.

### Operands

`<node>` - The node number given to the first slot.

## BOUNCESETBASE

`BOUNCESETBASE <base>` - Set a 16-bit field that nothing else in the bounce family reads. It is used
when placing whoever is bouncing, alongside the slot table and the thing's model, so it affects
presentation rather than bookkeeping.

See `BOUNCESETNODE` for the naming trap: despite its name, this does *not* set the node base.

### Operands

`<base>` - The value to store.

## BOUNCE

`BOUNCE <visitor ID> <seconds>` - Put a visitor into the first free slot of the ride's bounce table for
a given duration. A slot is free when its visitor handle is zero, and the scan starts from the first
slot every time.

The slot's node becomes the `BOUNCESETNODE` base plus the slot's index. The duration is in **seconds**;
the engine multiplies it by 1000 before adding it to the clock.

The outcome is reported through the script's result register rather than into an operand: non-zero if a
slot was free, zero if the table was full. How many slots exist is declared in the script's header and
is not `VAR_CAPACITY` - scripts gate themselves on that variable before calling this.

### Operands

`<visitor ID>` - The visitor to put on the ride.

`<seconds>` - How long their go lasts.

## UNBOUNCE

`UNBOUNCE <dest>` - Take off whoever is ready to come off and **write** their ID into the operand. This
instruction writes its operand rather than reading it. Zero means nobody was ready.

It walks from the first slot and takes the first occupied one whose duration has elapsed, clearing that
slot and decrementing the bouncing tally.

Releases are only made during a window at the start of each second of a rider's go, so a rider whose
time is up may still wait briefly before coming off.

### Operands

`<dest>` - Receives the visitor who came off, or zero.

## FORCEUNBOUNCE

`FORCEUNBOUNCE <dest>` - As `UNBOUNCE`, but without requiring the rider's duration to have elapsed.

It still observes the same release window, so "forcible" means "regardless of the duration", not
"unconditionally".

### Operands

`<dest>` - Receives the visitor who came off, or zero.

## BOUNCING

`BOUNCING <dest>` - Write the number of visitors currently bouncing into the operand, as a signed
16-bit value.

Scripts use this to gate admission, comparing it against `VAR_CAPACITY` before calling `BOUNCE`.

### Operands

`<dest>` - Receives the number currently bouncing.

## WALKON

`WALKON <visitor ID> <unknown1> <unknown2> <unknown3> <unknown4> <action> <unknown5>` - Unknown

### Operands

Unknown

## WALKOFF

`WALKOFF <visitor ID>` - Unknown

### Operands

Unknown

## WALKGET

`WALKGET <dest>` - Unknown

### Operands

Unknown

## WALKST_FLOAT

`WALKST_FLOAT <unknown1> <unknown2> <unknown3>` - Unknown

### Operands

Unknown

## WALKFLOATSTAT

`WALKFLOATSTAT <unknown>` - Unknown

### Operands

Unknown

## WALKFLOATSTOP

`WALKFLOATSTOP`

### Operands

Unknown

## ENABLELIGHT

`ENABLELIGHT <unknown>`

### Operands

Unknown

## DISABLELIGHT

`DISABLELIGHT <unknown>`

### Operands

Unknown

## SETLIGHT

`SETLIGHT <unknown> <unknown>`

### Operands

Two. What each means is not established, but the arity is: `SETLIGHT` takes 2, where its neighbours
`ENABLELIGHT` and `DISABLELIGHT` take 1 and `COLOURLIGHT` takes 4.

## COLOURLIGHT

`COLOURLIGHT <unknown1> <unknown2> <unknown3> <unknown4>`

### Operands

Unknown

## STARTSCREAM

`STARTSCREAM <visitor ID> <unknown>` - Unknown, likely causes a visitor to scream.

### Operands

Unknown

## STOPSCREAM

`STOPSCREAM`

### Operands

None

## SINGLESCREAM

`SINGLESCREAM <visitor ID> <unknown>`

### Operands

Two - the visitor, and one whose meaning is not established.

## SCREAMLEVEL

`SCREAMLEVEL <level>`

### Operands

`<level>` - The desired ride scream level (from 0 to 100).

## FINDSCRIPTRAND

`FINDSCRIPTRAND <ride / object name> <dest>`

### Operands

## GETREMOTEVAR

`GETREMOTEVAR <unknown1> <unknown2> <unknown3>`

### Operands

Unknown

## SETREMOTEVAR

`SETREMOTEVAR <script ID> <variable ID> <value>` - Set the value of another script's variable.

### Operands

`<script ID>` - The ID of the script containing the desired variable.

`<variable ID>` - The ID of the desired variable.

`<value>` - The desired value of the variable.

## REPAIREFFECT

`REPAIREFFECT <value>` - Show or hide repair effect.

### Operands

`<value>` - 0 or 1: 0 for hide, 1 for show.

## GETCUSTPTCLCODE

`GETCUSTPTCLCODE <unknown1> <unknown2>` - Unknown

### Operands

Unknown

## SETTIMER

`SETTIMER <time>` - Unknown

### Operands

Unknown

## GETTIMER

`GETTIMER <unknown>` - Unknown

### Operands

Unknown

## YEAR

`YEAR <dest>` - Get the current in-game year, and put it in the destination variable.

### Operands

`<dest>` - The destination variable for the instruction's result.

## MONTH

`MONTH <dest>` - Get the current in-game month, and put it in the destination variable.

### Operands

`<dest>` - The destination variable for the instruction's result.

## DAY

`DAY <dest>` - Get the current in-game day, and put it in the destination variable.

### Operands

`<dest>` - The destination variable for the instruction's result.

## HOUR

`HOUR <dest>` - Get the current in-game hour, and put it in the destination variable.

### Operands

`<dest>` - The destination variable for the instruction's result.

## MIN

`MIN <dest>` - Get the current in-game minute, and put it in the destination variable.

### Operands

`<dest>` - The destination variable for the instruction's result.

## SEC

`SEC <dest>` - Get the current in-game second, and put it in the destination variable.

### Operands

`<dest>` - The destination variable for the instruction's result.

## SETREVERB

`SETREVERB <level>` - Set the reverb level used for any sounds that output from this ride.

### Operands

`<level>` - The desired reverb level, from 0 (no reverb) to 10 (max reverb).

## DIPMUSIC

`DIPMUSIC <value>` - Mute or unmute music.

### Operands

`<value>` - 0 or 1: 0 for unmute, 1 for mute

## SPARK

`SPARK <unknown1> <unknown2> <unknown3> <unknown4>`

### Operands

Unknown
