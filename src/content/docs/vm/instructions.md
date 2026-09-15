---
title: Instruction Set
---

## NOP

`NOP` - This performs no operation, essentially wasting 1 cycle.

### Operands

None

## CRIT_LOCK

`CRIT_LOCK` - Begin a critical section.

Instructions stop counting against the script's time slice, so everything up to the matching `CRIT_UNLOCK` runs in one go rather than being cut short when the slice runs out. It does not lock a ride: this is the interpreter's own scheduling, not anything to do with visitors.

### Operands

None
## CRIT_UNLOCK

`CRIT_UNLOCK` - End a critical section, and give up the rest of this slice.

Instructions count against the slice again, and the script stops running until its next turn - the engine ends the slice as it leaves, so a script cannot hold the interpreter after unlocking.

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

Takes 1 operand, which is not named above and is not yet understood. No script the game ships uses this instruction, so nothing about it can be recovered from the data.

## SUB

`SUB <dest> <value> <value>` - Subtract the third operand from the second.

**The destination comes first.** The engine resolves operands 2 and 3, subtracts, and stores the result in operand 1 - this page previously showed the destination last.

### Operands

`<dest>` - The variable the result is stored in. It must be a variable; the engine ignores the instruction otherwise.

`<value>` - A literal or a variable.
## ENDSLICE

`ENDSLICE` - Stop running for this tick.

The script gives up the rest of its slice and resumes at the next instruction on its next turn. This is how a script yields voluntarily rather than running until its instruction budget is spent.

### Operands

None
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

Takes 5 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

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

Takes 4 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

## FLUSHANIM

`FLUSHANIM` - Stop all active animations

### Operands

None

## TRIGANIM

`TRIGANIM <unknown1> <unknown2> <unknown3>` - Unknown

### Operands

Takes 3 operands, those not named above being unknown. Across the 308 shipped scripts the 74 uses of this instruction write them as: 1 — literal; 2 — literal or variable; 3 — literal or variable.

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

Takes 3 operands, those not named above being unknown. Across the 308 shipped scripts the 133 uses of this instruction write them as: 1 — literal; 2 — literal or variable; 3 — literal or variable.

## GETANIM

`GETANIM <unknown>` - Unknown

### Operands

Takes 1 operand, which is not named above and is not yet understood. No script the game ships uses this instruction, so nothing about it can be recovered from the data.

## TRIGANIMSPEED

`TRIGANIMSPEED <unknown1> <unknown2> <unknown3> <unknown4>` - Unknown

### Operands

Takes 4 operands, those not named above being unknown. Across the 308 shipped scripts the 4 uses of this instruction write them as: 1 — literal; 2 — literal; 3 — variable; 4 — literal.

## FLUSHANIM_CH

`FLUSHANIM_CH <unknown>` - Unknown

### Operands

Takes 1 operand, which is not named above and is not yet understood. No script the game ships uses this instruction, so nothing about it can be recovered from the data.

## TRIGANIM_CH

`TRIGANIM_CH <unknown1> <unknown2> <unknown3> <unknown4>` - Unknown

### Operands

Takes 4 operands, those not named above being unknown. Across the 308 shipped scripts the 63 uses of this instruction write them as: 1 — literal; 2 — literal; 3 — literal or variable; 4 — literal.

## WAITANIM_CH

`WAITANIM_CH <unknown1> <unknown2> <unknown3>` - Unknown

### Operands

Takes 3 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

## LOOPANIM_CH

`LOOPANIM_CH <unknown1> <unknown2> <unknown3>` - Unknown

### Operands

Takes 3 operands, those not named above being unknown. Across the 308 shipped scripts the 1 uses of this instruction write them as: 1 — literal; 2 — literal; 3 — literal.

## TRIGWAITANIM_CH

`TRIGWAITANIM_CH <unknown1> <unknown2> <unknown3> <unknown4>` - Unknown

### Operands

Takes 4 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

## GETANIM_CH

`GETANIM_CH <unknown1> <unknown2>` - Unknown

### Operands

Takes 2 operands, those not named above being unknown. Across the 308 shipped scripts the 15 uses of this instruction write them as: 1 — literal; 2 — literal.

## RAND

`RAND <dest> <max value>` - Generate a random number.

### Operands

`<dest>` - The destination for the instruction's result.

`<max value>` - The highest value to generate.

## JSR

`JSR <subroutine>` - Jump to a subroutine, remembering where to come back to.

The return address is pushed onto the script's own call stack, whose size is set by `#setstack` and recorded in the file header. The stack grows **downwards** and is last-in-first-out, so nested calls return in the order you would expect. Overrunning it, or using `JSR` in a script with no stack at all, is refused.

### Operands

`<subroutine>` - The destination subroutine to jump to.
## RETURN

`RETURN` - Return to the instruction after the most recent `JSR`.

### Operands

None
## BRANCH

`BRANCH <location>` - Branch to another location.

### Operands

`<location>` - The branch to execute.

## BRANCH_Z

`BRANCH_Z <location>` - Branch to another location if the result register is zero.

The result register holds whatever the last instruction computed - see [Information](/vm/info/). A branch whose condition is false simply falls through; one whose operand is not a location stops the script.

### Operands

`<location>` - The branch to execute.
## BRANCH_NZ

`BRANCH_NZ <location>` - Branch to another location if the result register is not zero.

The result register holds whatever the last instruction computed - see [Information](/vm/info/). A branch whose condition is false simply falls through; one whose operand is not a location stops the script.

### Operands

`<location>` - The branch to execute.
## BRANCH_NV

`BRANCH_NV <location>` - Branch to another location if the result register is negative.

The result register holds whatever the last instruction computed - see [Information](/vm/info/). A branch whose condition is false simply falls through; one whose operand is not a location stops the script.

### Operands

`<location>` - The branch to execute.
## BRANCH_PV

`BRANCH_PV <location>` - Branch to another location if the result register is greater than zero.

**Zero does not branch here.** The engine's test is strictly greater than zero, not "not negative".

The result register holds whatever the last instruction computed - see [Information](/vm/info/). A branch whose condition is false simply falls through; one whose operand is not a location stops the script.

### Operands

`<location>` - The branch to execute.
## DBGMSG

`DBGMSG <unknown>` - Unknown

### Operands

Takes 1 operand, which is not named above and is not yet understood. No script the game ships uses this instruction, so nothing about it can be recovered from the data.

## NAME

`NAME <value>` - Set the current script's name (likely used for debugging purposes).

### Operands

`<value>` - The new name for the script.

## TEST

`TEST <variable>` - Load a variable into the result register, so a branch can test it.

### Operands

`<variable>` - The variable to test. It must be a variable; the engine ignores the instruction otherwise, and every one of the 1,318 uses in the shipped scripts is one.
## CMP

`CMP <variable> <value>` - Compare a variable against a value by **subtracting** the second operand from the first.

The result is left in the result register and nothing is stored, so `CMP` followed by `BRANCH_Z` reads as "branch if equal". This page previously described the comparison as a bitwise AND; the engine subtracts.

### Operands

`<variable>` - The variable compared. It must be a variable; all 61 uses in the shipped scripts are.

`<value>` - A literal or a variable to compare against.
## PUSH

`PUSH <value>` - Push a value to the stack.

### Operands

`<value>` - The value to push onto the stack.

## POP

`POP <dest>` - Pop a value from the stack into a variable.

### Operands

`<dest>` - The destination the popped value is written to.

## HUSH

`HUSH <value>` - Push a value onto the script's stack.

`HUSH` and `HOP` share the storage set aside by `#setstack`, but keep their own position in it, filling from the bottom while `JSR` fills from the top. The engine checks the two do not meet.

### Operands

`<value>` - The value to push. All 39 uses in the shipped scripts name a variable.
## HOP

`HOP <dest>` - Pop a value from the script's stack into a variable.

The counterpart to `HUSH`, and unrelated to `RETURN`, which pops from the other end of the same storage.

### Operands

`<dest>` - The variable the popped value is written to. It must be a variable; the engine ignores the instruction otherwise.
## WAIT

`WAIT <time>` - Wait for a period of time before continuing.

The engine does not block. The first time a `WAIT` runs it works out a deadline, **rewinds the program counter so the same `WAIT` runs again**, and ends the slice; on later turns it compares the clock against that deadline and simply falls through once it has passed. A script therefore sits on its `WAIT` instruction, costing one instruction per turn, until the time is up.

The operand is scaled by the script's own speed before being added to the clock, so it is not a count of ticks or of slices. The unit of the clock itself has not been established - do not assume milliseconds.

### Operands

`<time>` - How long to wait. Usually a literal; 13 of the 458 uses in the shipped scripts name a variable.
## WAITABS

`WAITABS <unknown>` - Unknown

### Operands

Takes 1 operand, which is not named above and is not yet understood. No script the game ships uses this instruction, so nothing about it can be recovered from the data.

## WAIT4ANIM

`WAIT4ANIM` - Wait for all of the currently playing animations to finish.

### Operands

None

## ADD

`ADD <dest> <value>` - Add a value to a variable.

### Operands

`<dest>` - The variable added to, and where the result is stored. It must be a variable; the engine ignores the instruction otherwise.

`<value>` - The value to add, either a literal or a variable.
## MULT

`MULT <dest> <value> <value>` - Multiply the second and third operands.

No script the game ships uses this instruction, so its behaviour here is read from the engine rather than from the data. Like the rest of the arithmetic, the destination comes first.

### Operands

`<dest>` - The variable the result is stored in.

`<value>` - A literal or a variable.
## DIV

`DIV <dest> <value> <value>` - Divide the second operand by the third.

**The destination comes first.** Division is signed, and **a divisor of zero does not fault** - the engine tests for it and yields 0.

### Operands

`<dest>` - The variable the quotient is stored in.

`<value>` - A literal or a variable.
## MOD

`MOD <dest> <value> <value>` - The remainder of dividing the second operand by the third.

**The destination comes first.** `DIV` and `MOD` are the same code path - one signed division, with `DIV` keeping the quotient and `MOD` the remainder - so **a divisor of zero yields 0 here too** rather than faulting.

### Operands

`<dest>` - The variable the remainder is stored in.

`<value>` - A literal or a variable.
## TURBO

`TURBO <0 or 1>` - Run this script on every tick instead of every eighth.

Scripts normally get a turn once every eight ticks, staggered by script so the work is spread out. `TURBO 1` opts this script out of that; `TURBO 0` puts it back. All twenty uses in the shipped scripts are literals - ten of each.

### Operands

`<value>` - 1 to run every tick, 0 to return to the usual schedule. The engine stores the low byte of the operand word as written, without resolving a variable.
## END

`END` - Stop the script.

The program counter is parked, and the execution loop tears the script down when it next comes round. **No script the game ships uses this** - they end by branching back on themselves instead.

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

`BOUNCESETNODE <node>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 1 uses of this instruction write it as a literal.

## BOUNCESETBASE

`BOUNCESETBASE <base>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 4 uses of this instruction write it as a literal.

## BOUNCE

`BOUNCE <visitor ID> <unknown>` - Unknown

### Operands

Takes 2 operands, those not named above being unknown. Across the 308 shipped scripts the 4 uses of this instruction write them as: 1 — variable; 2 — variable.

## UNBOUNCE

`UNBOUNCE <visitor ID>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 4 uses of this instruction write it as a variable.

## FORCEUNBOUNCE

`FORCEUNBOUNCE <visitor ID>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 8 uses of this instruction write it as a variable.

## BOUNCING

`BOUNCING <visitor ID>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 25 uses of this instruction write it as a variable.

## WALKON

`WALKON <visitor ID> <unknown1> <unknown2> <unknown3> <unknown4> <action> <unknown5>` - Unknown

### Operands

Takes 7 operands, those not named above being unknown. Across the 308 shipped scripts the 47 uses of this instruction write them as: 1 — variable; 2 — literal; 3 — literal or variable; 4 — literal or variable; 5 — literal; 6 — literal; 7 — literal.

## WALKOFF

`WALKOFF <visitor ID>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 53 uses of this instruction write it as a variable.

## WALKGET

`WALKGET <dest>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 43 uses of this instruction write it as a variable.

## WALKST_FLOAT

`WALKST_FLOAT <unknown1> <unknown2> <unknown3>` - Unknown

### Operands

Takes 3 operands, those not named above being unknown. Across the 308 shipped scripts the 1 uses of this instruction write them as: 1 — variable; 2 — literal; 3 — literal.

## WALKFLOATSTAT

`WALKFLOATSTAT <unknown>` - Unknown

### Operands

Takes 1 operand, which is not named above and is not yet understood. Across the 308 shipped scripts the 1 uses of this instruction write it as a literal.

## WALKFLOATSTOP

`WALKFLOATSTOP`

### Operands

None

## ENABLELIGHT

`ENABLELIGHT <unknown>`

### Operands

Takes 1 operand, which is not named above and is not yet understood. No script the game ships uses this instruction, so nothing about it can be recovered from the data.

## DISABLELIGHT

`DISABLELIGHT <unknown>`

### Operands

Takes 1 operand, which is not named above and is not yet understood. No script the game ships uses this instruction, so nothing about it can be recovered from the data.

## SETLIGHT

`SETLIGHT <unknown1> <unknown2>`

### Operands

Takes 2 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

## COLOURLIGHT

`COLOURLIGHT <unknown1> <unknown2> <unknown3> <unknown4>`

### Operands

Takes 4 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

## STARTSCREAM

`STARTSCREAM <visitor ID> <unknown>` - Unknown, likely causes a visitor to scream.

### Operands

Takes 2 operands, those not named above being unknown. Across the 308 shipped scripts the 40 uses of this instruction write them as: 1 — variable; 2 — literal.

## STOPSCREAM

`STOPSCREAM`

### Operands

None

## SINGLESCREAM

`SINGLESCREAM <visitor ID> <unknown>`

### Operands

`<visitor ID>` - The visitor who screams; always a variable in shipped scripts.

`<unknown>` - Always a literal in shipped scripts. Its meaning is not yet known.

## SCREAMLEVEL

`SCREAMLEVEL <level>`

### Operands

`<level>` - The desired ride scream level (from 0 to 100).

## FINDSCRIPTRAND

`FINDSCRIPTRAND <ride / object name> <dest>`

### Operands

`<ride / object name>` - A string naming what to look for.

`<dest>` - The variable the result is written to.

## GETREMOTEVAR

`GETREMOTEVAR <unknown1> <unknown2> <unknown3>`

### Operands

Takes 3 operands, those not named above being unknown. Across the 308 shipped scripts the 2 uses of this instruction write them as: 1 — literal; 2 — variable; 3 — literal.

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

Takes 2 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

## SETTIMER

`SETTIMER <time>` - Unknown

### Operands

Takes 1 operand. Across the 308 shipped scripts the 40 uses of this instruction write it as a literal.

## GETTIMER

`GETTIMER <unknown>` - Unknown

### Operands

Takes 1 operand, which is not named above and is not yet understood. Across the 308 shipped scripts the 21 uses of this instruction write it as a literal.

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

Takes 4 operands, those not named above being unknown. Across the 308 shipped scripts the 1 uses of this instruction write them as: 1 — variable; 2 — literal; 3 — literal; 4 — literal.
