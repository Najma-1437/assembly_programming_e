# Addition and EFLAGS

For this part of the assignment, I looked at `add1` and `add2` and checked
the flags after the addition in GDB. Both programs are built as 32-bit
executables, but `add1` adds 8-bit values using `AL`, while `add2` adds
16-bit values using `AX`.

## Running the programs

These are the commands to build and run `add1` from `03_arithmetic/add`:

```sh
nasm -f elf32 -g -F dwarf add1.asm -o add1.o
ld -m elf_i386 add1.o -o add1
./add1
gdb -q ./add1
```

To check the result and flags in GDB:

```text
set disassembly-flavor intel
break _start
run
stepi 2
info registers al eflags
stepi
x/1ub &result
continue
quit
```

I checked the flags right after `ADD`. `stepi 2` runs the first `MOV` and
the addition. The next `stepi` stores the result without changing the flags.
It is important to check before `xor ebx, ebx`, because that instruction
changes the flags again and leaves AF undefined.

For `add2`, the commands are the same except that `add1` is replaced with
`add2`. In GDB, use `info registers ax eflags` and `x/1uh &result` because
the result is 16 bits. Neither program prints anything, so I used GDB to
see the results.

## add1: 8-bit addition

```text
	120 = 0x78 = 01111000
+  10 = 0x0A = 00001010
----------------------
	130 = 0x82 = 10000010
```

After `add al, [num2]`, GDB showed:

```text
al       0x82    -126
eflags   0xa96   [ PF AF SF IF OF ]
```

The answer is `120 + 10 = 130`. GDB shows `-126` for `AL` because it displays
that value as signed. The bits are still `10000010`, which represent `130`
when read as unsigned.

| Flag | Status | Explanation |
| --- | --- | --- |
| CF (carry) | Cleared (0) | 130 is below the unsigned 8-bit maximum of 255, so there is no carry beyond bit 7. |
| PF (parity) | Set (1) | `10000010` has two 1s. Two is even, so the parity flag is set. |
| AF (auxiliary carry) | Set (1) | The lowest four bits add as `0x8 + 0xA = 0x12`. This carries from bit 3 to bit 4. |
| ZF (zero) | Cleared (0) | The result is 130, not zero. |
| SF (sign) | Set (1) | The highest bit of the 8-bit result, bit 7, is 1. This makes the result negative when read as signed. |
| OF (overflow) | Set (1) | 120 and 10 are both positive, but 130 is above the signed 8-bit maximum of 127. The result has a negative sign bit, so signed overflow occurred. |

The main thing I noticed is that OF can be set while CF is cleared.
130 fits as an unsigned 8-bit number, but not as a positive signed 8-bit
number. The two flags check different kinds of overflow.

## add2: 16-bit addition

```text
	32000 = 0x7D00
+   500 = 0x01F4
----------------
	32500 = 0x7EF4 = 01111110 11110100
```

After `add ax, [num2]`, GDB showed:

```text
ax       0x7ef4  32500
eflags   0x202   [ IF ]
```

The answer is `32000 + 500 = 32500`. This fits in both the signed and
unsigned 16-bit ranges, so it has the same value either way.

| Flag | Status | Explanation |
| --- | --- | --- |
| CF (carry) | Cleared (0) | 32500 is below the unsigned 16-bit maximum of 65535, so there is no carry beyond bit 15. |
| PF (parity) | Cleared (0) | Only the lowest byte counts for parity. Here it is `0xF4`, or `11110100`, which has five 1s. Five is odd. |
| AF (auxiliary carry) | Cleared (0) | The lowest four bits add as `0x0 + 0x4 = 0x4`, so there is no carry from bit 3 to bit 4. |
| ZF (zero) | Cleared (0) | The result is 32500, not zero. |
| SF (sign) | Cleared (0) | The highest bit of the 16-bit result, bit 15, is 0, so the signed result is positive. |
| OF (overflow) | Cleared (0) | 32500 is below the signed 16-bit maximum of 32767. Adding the two positive numbers still gives a positive result, with no signed overflow. |

## A note about IF

GDB showed IF set in both programs. This is the interrupt enable flag,
and `ADD` does not change it, so it is not caused by the addition.
The hexadecimal EFLAGS values also include bit 1, which is reserved and
always set. The flags affected by `ADD` are CF, PF, AF, ZF, SF, and OF.
