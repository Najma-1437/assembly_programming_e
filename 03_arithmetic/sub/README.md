# Subtraction and EFLAGS

For this part of the assignment, I checked `sub1` and `sub2` in GDB.
`sub1` uses `AL` for an 8-bit subtraction, while `sub2` uses `AX` for
a 16-bit subtraction. I checked the flags right after `SUB` and then
looked at the stored results.

## Running the programs

These are the commands to build and run `sub1` from `03_arithmetic/sub`:

```sh
nasm -f elf32 -g -F dwarf sub1.asm -o sub1.o
ld -m elf_i386 sub1.o -o sub1
./sub1
gdb -q ./sub1
```

To check it in GDB:

```text
break _start
run
stepi 2
info registers al eflags
stepi
x/1ub &result
x/1db &result
continue
quit
```

`stepi 2` runs the first `MOV` and then `SUB`. The next step stores the
result without changing the flags. The two memory commands display the
same byte as unsigned and signed so I can compare the interpretations.

For `sub2`, replace `sub1` with `sub2` in the build commands and use:

```text
break _start
run
stepi 2
info registers ax eflags
stepi
x/1uh &result
x/1dh &result
continue
quit
```

Neither program prints anything, so I used GDB to see the answers.
I checked before the exit code because `mov eax, 1` overwrites the result
register and `xor ebx, ebx` changes the flags, leaving AF undefined.

## sub1: 50 minus 80

```text
50 - 80 = -30
8-bit result: 0xE2 = 11100010
Unsigned interpretation: 256 - 30 = 226
```

After `sub al, [num2]`, GDB showed:

```text
al       0xe2   -30
eflags   0x287  [ CF PF SF IF ]
```

The stored byte was 226 when displayed as unsigned and -30 when
displayed as signed. These are two interpretations of the same bits.
The signed result is correct because -30 fits in the 8-bit signed range.

| Flag | Status | Explanation |
| --- | --- | --- |
| CF (carry/borrow) | Set (1) | 50 is smaller than 80, so unsigned subtraction needs a borrow. The 8-bit unsigned result wraps around to 226. |
| PF (parity) | Set (1) | The result byte `11100010` has four 1s. Four is even, so PF is set. |
| AF (auxiliary borrow) | Cleared (0) | The lowest four bits subtract as `0x2 - 0x0 = 0x2`. No borrow is needed from bit 4 into bit 3. |
| ZF (zero) | Cleared (0) | The result is -30, not zero. |
| SF (sign) | Set (1) | Bit 7, the highest bit of the 8-bit result, is 1. The result is negative when read as signed. |
| OF (overflow) | Cleared (0) | -30 fits in the signed 8-bit range of -128 to 127, so there is no signed overflow. Subtracting these two positive numbers can produce a valid negative result. |

CF is set but OF is cleared because needing an unsigned borrow is not
the same as signed overflow. Also, CF being set does not mean AF must
be set: AF only checks the borrow at the lowest four bits.

## sub2: 1000 minus 2000

```text
1000 = 0x03E8
2000 = 0x07D0
1000 - 2000 = -1000
16-bit result: 0xFC18 = 11111100 00011000
Unsigned interpretation: 65536 - 1000 = 64536
```

After `sub ax, [num2]`, GDB showed:

```text
ax       0xfc18  -1000
eflags   0x287   [ CF PF SF IF ]
```

The stored word was 64536 when displayed as unsigned and -1000 when
displayed as signed. The signed answer fits in 16 bits.

| Flag | Status | Explanation |
| --- | --- | --- |
| CF (carry/borrow) | Set (1) | 1000 is smaller than 2000, so unsigned subtraction needs a borrow. The 16-bit unsigned result wraps around to 64536. |
| PF (parity) | Set (1) | Parity only checks the lowest byte, `0x18` (`00011000`). It has two 1s, which is an even count. |
| AF (auxiliary borrow) | Cleared (0) | The lowest four bits subtract as `0x8 - 0x0 = 0x8`. No borrow is needed from bit 4 into bit 3. |
| ZF (zero) | Cleared (0) | The result is -1000, not zero. |
| SF (sign) | Set (1) | Bit 15, the highest bit of the 16-bit result, is 1, so the signed result is negative. |
| OF (overflow) | Cleared (0) | -1000 fits in the signed 16-bit range of -32768 to 32767. The negative answer is valid, so there is no signed overflow. |

## What I noticed about the flags

Both programs set CF, PF, and SF, and clear AF, ZF, and OF. The negative
answers do not automatically mean overflow happened. SF tells me the
result's sign bit, CF tells me about the unsigned borrow, and OF tells
me whether the signed answer fits in the destination.

GDB also showed IF set in both runs. This is the interrupt enable flag,
and `SUB` does not change it, so it is not caused by the subtraction.
The EFLAGS values also include reserved bit 1, which is always set.
