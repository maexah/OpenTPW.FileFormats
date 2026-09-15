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

`GETTIME <dest>` - Read the game's clock.

**This page used to call it "the time that the ride has been alive for", and that is wrong.** The handler reads the game's own clock object - the same one the whole engine shares - and stores what it hands back with no arithmetic at all. Nothing per-ride is involved and nothing is subtracted, so a script timing something has to take one reading and compare against a later one, which is what the 172 uses across 57 scripts do.

The clock counts **milliseconds**.

### Operands

`<dest>` - The destination for the instruction's result. Every shipped use names a variable.

## ADDOBJ

`ADDOBJ <type> <node> <effect> <tag>` - Start a particle effect or a sound on the ride, and keep a record of it so that a later `KILLOBJ` can stop it again.

With no model to place it on, the effect is still started - at the origin, because the position lookup answers success and a degenerate point rather than failing.

### Operands

`<type>` - Which subsystem, and for a sound which category. 1 and 2 start a particle effect; 3 to 9 play a sample through a named sound category - 3 and 4 through a second `cat_rides` / `cat_ambient` pair, then `cat_rides`, `cat_kids`, `cat_staff`, `cat_ambient` and `cat_ui`; 10 plays the thing's own custom effect, and does nothing at all if no custom sound bank has been registered for it. Anything else is rejected with "RSSE: Unknown object type", starting nothing and leaving no record. The range check is unsigned, so zero and negative values are rejected on the same path.

`<node>` - Which node of the model the effect sits on, or -1 for the model's own origin. -1 is the commonest value in the shipped scripts, at 277 of 644 uses. A node the model does not have is rejected with "RSSE: Invalid Node ID", and nothing is started.

`<effect>` - A particle effect id when `<type>` is 1 or 2, and a sample index within the chosen category otherwise. These are two unrelated numberings, and **neither is range-checked**: bit 15 marks a custom particle code, which the engine remaps before use. No shipped script names a particle above 95, but that is a property of the shipped scripts rather than a rule of the instruction.

`<tag>` - What a later `KILLOBJ` or `FADEOBJ` matches on to stop this again. **It is a tag, not a duration or a lifetime**: the values the shipped scripts kill are exactly the values their own `ADDOBJ`s create. Several objects may carry the same tag, and stopping it stops all of them.

## ADDOBJ_EXT

`ADDOBJ_EXT <type> <node> <effect> <lifetime> <tag>` - As `ADDOBJ`, but naming the lifetime that `ADDOBJ` leaves at its default.

### Operands

The first three and the last are `ADDOBJ`'s. The extra one sits where `ADDOBJ` passes a fixed 1000, and 1000 is the value the engine tests against to decide whether to apply a lifetime at all - so a different value gives the started effect that lifetime. It is applied only to the two particle types.

No script the game ships uses this instruction, so the operand meanings here come from the handler rather than from the data: it allocates the same record `ADDOBJ` does and calls the same worker.

## KILLOBJ

`KILLOBJ <tag>` - Stop every effect this script started carrying that tag.

### Operands

`<tag>` - The tag given as `ADDOBJ`'s fourth operand.

**Every** record carrying the tag is stopped, not merely the first. The handler walks the whole list, and on a match it steps to the next record *before* unlinking and freeing the one it matched, then returns to the same test - there is no break anywhere in it. A tag nothing carries stops nothing, which the shipped scripts do for real: two of the tags they kill are created by no `ADDOBJ` anywhere.

## FADEOBJ

`FADEOBJ <tag>` - Stop every effect carrying that tag, letting a sound fade out rather than cutting it.

### Operands

`<tag>` - As `KILLOBJ`'s, and the walk over the records is the same one.

**For a particle the two instructions are identical** - both reach the same kill with the same argument - so they differ only for a sound, where this one stops it fading and `KILLOBJ` stops it outright.

## SETOBJPARAM

`SETOBJPARAM <slot> <parameter> <value>` - Set a sub-object's parameters.

### Operands

`<slot>` - The slot of the desired object.

`<parameter>` - The parameter to change.

`<value>` - The desired value of the parameter.

## EVENT

`EVENT <type> <node> <effect>` - Start a particle effect or a sound, keeping no record of it.

### Operands

`<type>` - As `ADDOBJ`'s: the subsystem, and for a sound the category.

`<node>` - As `ADDOBJ`'s: a node of the model, or -1 for its origin. -1 is again the commonest value, at 341 of 527 uses.

`<effect>` - As `ADDOBJ`'s: a particle id or a sample index.

The handler does the same work `ADDOBJ` does and then **throws away the handle it gets back**, so nothing an `EVENT` starts can ever be stopped by `KILLOBJ` or `FADEOBJ` - those consult only the list of records, and `EVENT` adds none to it. That is the whole difference between the two instructions, and it is why `EVENT` takes no tag.

## EVENT_EXT

`EVENT_EXT <unknown1> <unknown2> <unknown3> <unknown4>` - Unknown

### Operands

Takes 4 operands, those not named above being unknown. No script the game ships uses this instruction, so nothing about them can be recovered from the data.

## FLUSHANIM

`FLUSHANIM` - Stop all active animations

### Operands

None

## TRIGANIM

`TRIGANIM <animation> <parameter> <dest>` - Start a one-shot animation, and put how long it runs into `<dest>`.

### Operands

`<animation>` - The animation to start.

`<parameter>` - Passed to the model along with the animation.

`<dest>` - Where the length is written, in milliseconds. 64 of the 74 shipped uses write a literal here, which cannot be written to: the length still lands in the result register, and the branch that follows reads it.

The length is whatever the model reports, less 300, floored at 300 by a **signed** comparison - so a script whose model reports nothing gets 300 rather than a negative. Triggering also arms the deadline `WAIT4ANIM` waits on, exactly as `TRIGWAITANIM` does.

## WAITANIM

`WAITANIM <type> <parameter>` - Start playing an animation, and wait for it to end before continuing.

### Operands

`<type>` - The type of animation to play

`<parameter>` - Passed to the model along with the animation, as `TRIGANIM`'s second operand is.

It waits on the same deadline as `WAIT`, not on the separate one `WAIT4ANIM` uses, and it gives up the rest of the turn on the visit that sets that deadline whether or not the deadline has already passed.

Its arithmetic is `TRIGANIM`'s with two differences, and both change the answer. The length-less-300 is stored as the low half of a qword whose high half is nought and read back with `FILD qword`, so a negative arrives as a number just under 2^32; and the 300 floor is then compared **unsigned**, which a negative passes. So where `TRIGANIM` floors at 300, `WAITANIM` does not: a model that reports nothing gives a deadline 300ms in the *past*, and the instruction costs one turn rather than any particular length of time.

## LOOPANIM

`LOOPANIM <type> <parameter>` - Start playing an animation on loop

### Operands

`<type>` - The type of animation to play

`<parameter>` - Passed to the model along with the animation.

The two operands together make the key `(<parameter> << 16) + <type>`, which the script remembers. Asking again for the animation already looping does nothing at all - it is not restarted. Naming any other animation clears the deadline `TRIGANIM` and `TRIGWAITANIM` arm, because a loop never finishes and there would be nothing left for `WAIT4ANIM` to wait for.

## TRIGWAITANIM

`TRIGWAITANIM <animation> <parameter> <dest>` - Start a one-shot animation, put how long it runs into `<dest>`, and wait for it.

### Operands

`<animation>` - The animation to start.

`<parameter>` - Passed to the model along with the animation.

`<dest>` - Where the length is written, in milliseconds, on the same terms as `TRIGANIM`'s third operand. 132 of the 133 shipped uses write a literal here.

It arms the same deadline `TRIGANIM` arms, and then rewinds onto itself - so unlike `TRIGANIM` it does the waiting itself rather than leaving it to a later `WAIT4ANIM`. While it waits it also keeps a cursor of its own, stepping across turns through more than one animation channel. That part has not been read the whole way through, and what the cursor counts is not yet established.

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

The range is **nought to the bound inclusive**: the engine halves a draw from its generator and takes it modulo the bound *plus one*, so the highest value really is reachable. The generator is a multiply-add followed by a thirteen-bit rotate, and the result is made positive twice over - once as it leaves the generator and again after the modulo.

**The bound is not resolved the way other value operands are.** Every instruction that reads a value tests the operand's tag first and looks a variable up; this one does not - it takes the low sixteen bits of the operand word as a signed number whatever the tag says. A bound written as a variable would therefore be read as that variable's *index*. No shipped script does it: all 56 uses name a literal, with bounds of 1 to 10 apart from one 300 and one 5000.

### Operands

`<dest>` - The destination for the instruction's result. 52 of the 56 shipped uses name a variable; the other four name a literal, so the answer stays in the result register for the following branch to test.

`<max value>` - The highest value to generate, and it can be generated.

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

**The clock counts milliseconds**, which this page previously said was unestablished. The engine adds the wait to the clock object the whole game shares, and that object's reading comes from a source which falls back to `timeGetTime` and scales its high-resolution path to agree with it. So `WAIT 3000` is three seconds.

The operand *is* scaled by the script's own speed first - the engine divides it by `0.5 + 0.01 x speed`, worked out afresh for every instruction - **but that scaling can never do anything.** The speed word is written in exactly two places in the whole script system: the loader setting it to 50, and the scheduler copying it into a linked script. No instruction writes it. At 50 the divisor is exactly 1, so every wait in every shipped script is its operand unchanged.

### Operands

`<time>` - How long to wait. Usually a literal; 13 of the 458 uses in the shipped scripts name a variable.
## WAITABS

`WAITABS <time>` - Wait until the clock reaches a given reading.

No script the game ships uses this, so nothing about how it was *meant* to be used can be recovered - but what it does is plain from the handler, which shares most of its code with `WAIT`. The difference is the one the name suggests: `WAIT` adds its operand to the clock to make a deadline, while this one takes the operand **as** the deadline, already on the clock's own scale. It is not speed-scaled, and it rewinds onto itself and ends the slice exactly as `WAIT` does.

### Operands

`<time>` - The clock reading to wait for, in milliseconds.

## WAIT4ANIM

`WAIT4ANIM` - Wait for the animation last triggered to finish.

### Operands

None

It waits on a single deadline - set by `TRIGANIM` or `TRIGWAITANIM`, cleared by `LOOPANIM` - and not on "everything currently playing". **With nothing triggered it does not wait at all:** its first test is whether that deadline is set, and it goes straight on when it is not.

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

`BUMP <command> <params>` - Read or change one property of the bumper-style ride this script drives.

The `<command>` is fetched **without being resolved**, so it has to be a literal: the engine decrements it, refuses anything above 17, and jumps through a seventeen-entry table. All 199 uses in the shipped scripts are literals, spread across twelve scripts, and they use every command except `15`.

Command `15` is not merely unused by the shipped scripts - **its table entry is the error path itself**, so the engine treats it exactly as it treats 0 or 18.

One structural note that the numbering alone does not show: commands `13` and `14` call the **same** engine function, `13` multiplying its operand by 30 first and `14` negating it. Whatever they are named, they are two directions of one quantity.

The ride this reaches is not the one [`COAST`](#coast) reaches, although the two dispatchers begin identically: both look an object up from the script before reading the command. What differs is what the commands then act on. `BUMP`'s act on the object that lookup found; `COAST`'s act on the handle `COAST_INITIALISE` stored on the script, which is a separate table entirely, and the two sets of engine functions do not overlap.

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

`COAST <id> <value>` - Read or change one property of the coaster this script drives.

The `<id>` is fetched **without being resolved**, so it has to be a literal: a variable operand keeps its tag and falls outside the 1-8 the engine accepts, which the engine reports as `RSSE: Unknown bumper ride command`. All 144 uses in the shipped scripts are literals.

That message is not a mistake in this page: `COAST` and [`BUMP`](#bump) share a single error handler, which carries one string between them, and it names only the bumper ride.

The two reading commands, `COAST_GETQUEUE` and `COAST_GETPEEP`, put their answer in the [result register](/vm/info/) first and only then store it, so a script may pass a literal as the destination purely to test the answer and throw it away. That is not a mistake in those scripts - all twelve uses of `COAST_GETQUEUE` are written `COAST 2 0`, and the `BRANCH_Z` that follows reads the register.

### Operands

`<id>` - The ID of the property being set

`<value>` - The value to assign to the property

#### IDs

- `1` (`COAST_ADDPEEP`) - Put a visitor in the queue. The value is the visitor, and the ride declines silently if there is no room.

- `2` (`COAST_GETQUEUE`) - **The room remaining, not the number of people queueing.** The engine computes capacity minus those on the ride minus those queueing, and clamps the result at zero. A script branching on zero after this is branching on "full", not on "empty".

- `3` (`COAST_GETPEEP`) - **Take the next visitor who has finished the ride, or 0 if there is none.** This is one visitor's id, not a count, and the queue it drains is not the one `COAST_ADDPEEP` fills - those are two separate rings, and nothing in `COAST` moves anyone between them.

- `4` (`COAST_SETBROKE`) - Ask for a change of repair state. **Not a boolean:** the handler dispatches on 0, 1 and 2, and passes each to the same state-request function `COAST_SETCLOSED` uses, because the two write different fields of one word. The shipped scripts only ever pass 0 and 1.

- `5` (`COAST_SETCLOSED`) - Ask to open (0) or shut (anything else). **A guarded transition rather than a flag:** the engine checks the ride's current state first and drops the request when it does not fit, so shutting a ride that is already shut does nothing at all.

- `6` (`COAST_SETCAPACITY`) - Set how many the ride will hold. Negative values clamp to zero, and the value is clamped again by the ride's own record and by the track's before it is applied. It is the same number `COAST_GETQUEUE` measures against.

- `7` (`COAST_SETWORN`) - **Does nothing.** The handler fetches its operand, resolves it, tidies the stack and returns without calling anything, so the value is discarded. Twelve shipped scripts use it, which is worth knowing before implementing wear on its account.

- `8` (`COAST_INITIALISE`) - Bind the script to its ride. The engine looks the ride up by the script's **own** id rather than by this instruction's operand - every shipped use passes a literal 0 - and keeps the handle it finds. If it finds none, every other `COAST` command quietly does nothing.

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

`SETTIMER <time>` - Start the script's timer, to run for this long.

A script has **one** timer. This sets it by adding the operand to the clock and keeping the result as a deadline. **Unlike `WAIT`, it is not scaled by the script's speed** - the engine resolves the operand and adds it to the clock reading directly.

It does not wait for anything. Setting a timer and then carrying on is the point: the script gets on with something else and asks `GETTIMER` how much is left whenever it wants to know.

### Operands

`<time>` - How long the timer should run, in milliseconds. All 40 shipped uses name a literal.

## GETTIMER

`GETTIMER <dest>` - How much of the script's timer is left.

The deadline `SETTIMER` stored, less the clock as it now stands, and **never less than nought** - the engine replaces a negative answer with zero, so a timer that has run out reads as nought rather than counting downwards for ever.

All 21 shipped uses name a literal where the destination goes, which means the write is skipped and the answer is left in the result register for the branch that follows - the same idiom as `COAST 2 0`. The 18 scripts using it are the same 18 that use `SETTIMER`.

### Operands

`<dest>` - The destination for the instruction's result.

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
