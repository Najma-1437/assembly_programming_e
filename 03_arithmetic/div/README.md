# Division and EFLAGS

For this part of the assignment, I checked `div1` and `div2` in GDB.
Both use `DIV`, which divides unsigned numbers. I checked the quotient,
remainder, and flags immediately after the division.

## Running the programs

These are the commands to build and run `div1` from `03_arithmetic/div`:

```sh
nasm -f elf32 -g -F dwarf div1.asm -o div1.o
ld -m elf_i386 div1.o -o div1
./div1
gdb -q ./div1
```

To check it in GDB:

```text
break _start
run
stepi 3
info registers ax al ah bl eflags
continue
quit
```

`stepi 3` runs the two `MOV` instructions and then `DIV`. For `div2`,
replace `div1` with `div2` in the build commands and use these GDB commands:

```text
break _start
run
stepi 4
info registers ax dx bx eflags
continue
quit
```

There are three `MOV` instructions before the division in `div2`, so it
takes four steps. Neither program prints anything. I checked the registers
before the exit code because `mov eax, 1` overwrites the quotient register
and `xor ebx, ebx` changes the flags.

## div1: 100 divided by 7

`div bl` divides the 16-bit value in `AX` by the 8-bit value in `BL`.
The quotient goes into `AL` and the remainder goes into `AH`.

```text
100 = (7 * 14) + 2
```

After the division, GDB showed:

```text
al       0xe    14
ah       0x2    2
ax       0x20e  526
eflags   0x212  [ AF IF ]
```

The quotient is 14 and the remainder is 2. The remainder is smaller than
the divisor, 7, which is what I expect. `AX` shows 526 because it contains
both answers together: `AH` is `0x02` and `AL` is `0x0E`, giving `0x020E`.
526 is not the quotient.

## div2: 50000 divided by 300

`div bx` divides the 32-bit value in `DX:AX` by the 16-bit value in `BX`.
`DX` holds the upper 16 bits of the dividend and `AX` holds the lower 16
bits. Here `DX` starts at zero, so the full dividend is 50000.
`DIV` treats 50000 as unsigned, even though it is above the signed 16-bit
maximum of 32767.

```text
50000 = (300 * 166) + 200
```

After the division, GDB showed:

```text
ax       0xa6   166
dx       0xc8   200
bx       0x12c  300
eflags   0x212  [ AF IF ]
```

The quotient is 166 in `AX` and the remainder is 200 in `DX`.
The remainder is smaller than 300, and multiplying the quotient by the
divisor and adding the remainder gives the original 50000.

## What happened to the flags?

The main difference from addition is that `DIV` leaves CF, PF, AF, ZF,
SF, and OF undefined. GDB still displays a value for each flag, but the
instruction does not guarantee what those values mean. Undefined does
not mean cleared, and it does not guarantee the old value is preserved.

Both runs showed the following values:

| Flag | Displayed value in both runs | Explanation |
| --- | --- | --- |
| CF (carry) | Cleared (0) | Undefined after `DIV`. I cannot use this value to say whether the quotient fits. |
| PF (parity) | Cleared (0) | Undefined after `DIV`. It does not reliably describe the number of 1s in the quotient's lowest byte. |
| AF (auxiliary carry) | Set (1) | Undefined after `DIV`. The displayed 1 does not mean the division caused a carry from bit 3 to bit 4. |
| ZF (zero) | Cleared (0) | Undefined after `DIV`. Although both quotients are nonzero, that is not a valid explanation for this flag value. |
| SF (sign) | Cleared (0) | Undefined after `DIV`. It cannot be used to interpret the quotient's sign; these divisions are unsigned. |
| OF (overflow) | Cleared (0) | Undefined after `DIV`. Division does not report a quotient that is too large by setting OF. |

So I cannot explain these flags from the result the way I did for `ADD`.
The displayed values are observations from these runs, not guaranteed
results on every processor or run.

If the divisor is zero or the quotient is too large for its destination,
`DIV` raises a divide error (`#DE`) instead of setting CF or OF to report
the problem. Here 14 fits in `AL` (0 to 255), and 166 fits in `AX`
(0 to 65535), so both divisions completed normally.

GDB also showed IF set. IF is the interrupt enable flag and is not changed
by `DIV`, so it is not caused by the division. The EFLAGS value also
includes reserved bit 1, which is always set.
