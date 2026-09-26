---
title: Information
---

Theme Park World uses a stack-based VM in order to give rides and sideshows their functionality. This VM reads from the [RSSE](https://github.com/ThemeParkWorld/OpenTPW/wiki/RSSE-(Ride-&-Sideshow-Script-Engine)) format, and includes exactly 106 instructions.

The dispatcher accepts an instruction only while its opcode is below `0x6A`, which is 106, and refuses anything else with `RSSE: Unknown instruction`. All 106 are listed on the [Instruction Set](/vm/instructions/) page, in the order the engine's own name table gives them. Of those, 84 are used by at least one of the 308 scripts the game ships.

## Common Variable Set

Ride scripts share a "common variable set": the twelve variables below, in this order, at the start of the script's own variable table. **Of the 308 scripts the game ships, 129 carry it** - 98 declare no variables at all, and the rest begin with a different family entirely, such as `VAR_EVT0`-`VAR_EVT9` or `VAR_TRIGGER`/`VAR_STATUS`. So it is a convention among rides rather than something every script has.

The order matters more than the names do. The VM addresses a variable by its **index**, not by its name, so what makes the set usable is that it is a prefix: `VAR_ONRIDE` is operand `5` in any script that follows the convention. Scripts that use it carry exactly these twelve first and then their own; there is no thirteenth common variable, and index 12 is whatever that script wanted (most often `VAR_TEMP`, `VAR_COUNT` or `VAR_PEEPID`).


| Name             | ID  | Description                                                               |
| ---------------- | --- | ------------------------------------------------------------------------- |
| `VAR_LETMEON`    | 0   | First ID of visitor in ride queue                                         |
| `VAR_LETMEOFF`   | 1   | First ID of visitor wishing to leave                                      |
| `VAR_CAPACITY`   | 2   | How many visitors can use the ride at once                                |
| `VAR_DURATION`   | 3   | The time for the ride to run for                                          |
| `VAR_BREAKSTAT`  | 4   | Whether the ride is broken (1) or not (0)                                 |
| `VAR_ONRIDE`     | 5   | The number of visitors on the ride                                        |
| `VAR_RIDECLOSED` | 6   | Is the ride closed? (0/1)                                                 |
| `VAR_BROKEN`     | 7   | Is the ride broken? (0/1)                                                 |
| `VAR_WORN`       | 8   | How worn is the ride?                                                     |
| `VAR_RUNNING`    | 9   | Is the ride running? (0/1)                                                |
| `VAR_PAD`        | 10  | Unused                                                                    |
| `VAR_PARAM`      | 11  | Sideshows: Determine whether the current visitor will win (1) or lose (0) |

## The result register

There are no condition flags. The VM keeps a single **result register**: every instruction that computes a value writes it there, and the conditional branches test that register against zero.

- `TEST <variable>` loads a variable into it.
- `CMP <variable> <value>` subtracts the second from the first and leaves the difference in it.
- Arithmetic (`ADD`, `SUB`, `MULT`, `DIV`, `MOD`) leaves its result in it as well as storing it.

`BRANCH_Z` and `BRANCH_NZ` then branch on the register being zero or non-zero, and `BRANCH_NV` and `BRANCH_PV` on it being negative or greater than zero. The comparison is signed, and `BRANCH_PV` does not branch on zero.

An instruction whose destination operand is not a variable is ignored rather than refused - the engine steps over it and carries on.

## When a script runs

A script does not get a turn every tick. The engine keeps one counter for the whole system and steps it at the top of each tick, **before** any script runs, so the first tick is 1 and the counter decides who is due on it. A script is due when:

```
turbo || ((id ^ tick) & 7) == 0
```

Each script therefore gets one turn in eight ticks, and the eight are spread apart by the scripts' own ids rather than all falling on the same tick. `TURBO` opts a script out of the spread and into every tick.

On its turn a script runs instructions until its time slice is spent or it gives the rest up. The slice is a count of **instructions**, not of time, and it comes from the script's own header - 50 in every shipped script. `ENDSLICE` and `CRIT_UNLOCK` both end the turn immediately, and `WAIT` ends it by putting the program counter back onto itself so the same instruction runs again next turn.

`CRIT_LOCK` stops instructions counting against the slice until the matching `CRIT_UNLOCK`. The flag behind that is cleared as each script's turn begins, so **a critical section cannot outlive the turn that took it** - a script that locks and then yields comes back with instructions counting normally again.

A script whose program counter has been parked by `END` is taken off the list at the end of the same tick.
