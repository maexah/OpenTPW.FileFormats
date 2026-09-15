---
title: Information
---

Theme Park World uses a stack-based VM in order to give rides and sideshows their functionality. This VM reads from the [RSSE](https://github.com/ThemeParkWorld/OpenTPW/wiki/RSSE-(Ride-&-Sideshow-Script-Engine)) format, and includes exactly 106 instructions.

The dispatcher accepts an instruction only while its opcode is below `0x6A`, which is 106, and refuses anything else with `RSSE: Unknown instruction`. All 106 are listed on the [Instruction Set](/vm/instructions/) page, in the order the engine's own name table gives them. Of those, 84 are used by at least one of the 308 scripts the game ships.

## Common Variable Set

The following variables are part of a "common variable set" that must be contained within every ride (these do not apply to features).


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
