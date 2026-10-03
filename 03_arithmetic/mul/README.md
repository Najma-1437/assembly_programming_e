# Multiplication and EFLAGS

For this part of the assignment, I checked `mul1` and `mul2` in GDB.
Both use `MUL`, which multiplies unsigned numbers. I checked the registers
and flags right after the multiplication, then checked the stored result.

## Running the programs

These are the commands to build and run `mul1` from `03_arithmetic/mul`:

```sh
nasm -f elf32 -g -F dwarf mul1.asm -o mul1.o
ld -m elf_i386 mul1.o -o mul1
./mul1
gdb -q ./mul1
```

To check it in GDB:

```text
break _start
run
stepi 2
info registers ax al ah eflags
stepi
x/1uh &result
continue
quit
```

`stepi 2` runs the first `MOV` and then `MUL`. The next step stores the
result. `MOV` does not change the flags.

For `mul2`, replace `mul1` with `mul2` in the build commands and use:

```text
break _start
run
stepi 2
info registers ax dx eflags
stepi 2
x/1uw &result
continue
quit
```

`mul2` needs two steps after the multiplication to store both halves of
the result. Neither program prints anything, so I used GDB to see the
answers. I checked before the exit code because `mov eax, 1` overwrites
part of the result and `xor ebx, ebx` changes the flags.

## mul1: 25 multiplied by 10

`mul byte [num2]` multiplies the value in `AL` by the byte in `num2`.
The full 16-bit product goes into `AX`, with the upper byte in `AH` and
the lower byte in `AL`.

```text
25 * 10 = 250 = 0x00FA
```

After the multiplication, GDB showed:

```text
ax       0xfa   250
al       0xfa   -6
ah       0x0    0
eflags   0x202  [ IF ]
```

The stored result was 250. GDB shows `AL` as signed -6, but `MUL` is
unsigned, so `0xFA` represents 250 here. The upper byte, `AH`, is zero.

| Flag | Displayed status | Explanation |
| --- | --- | --- |
| CF (carry) | Cleared (0) | 250 fits in an unsigned byte (0 to 255). The upper half of the product, `AH`, is zero, so `MUL` clears CF. |
| OF (overflow) | Cleared (0) | For `MUL`, OF follows the same rule as CF. `AH` is zero, so OF is cleared too. |
| PF (parity) | Cleared (0), undefined | `MUL` does not define PF. Even though `0xFA` has six 1s, I cannot use that even count to predict this flag after multiplication. |
| AF (auxiliary carry) | Cleared (0), undefined | `MUL` does not define AF, so the displayed value does not tell me about a carry from bit 3 to bit 4. |
| ZF (zero) | Cleared (0), undefined | The product is nonzero, but `MUL` does not define ZF. That is not a reliable explanation for the displayed 0. |
| SF (sign) | Cleared (0), undefined | `MUL` does not define SF. I cannot use it to determine the sign of the product. |

Unlike `ADD`, unsigned `MUL` does not use OF to check the signed range.
250 is above the signed byte maximum of 127, but CF and OF are still
cleared because the unsigned product fits in the lower byte.

## mul2: 3000 multiplied by 200

`mul word [num2]` multiplies the value in `AX` by the word in `num2`.
The full 32-bit product goes into `DX:AX`. `DX` holds the upper 16 bits
and `AX` holds the lower 16 bits.

```text
3000 * 200 = 600000 = 0x000927C0
DX = 0x0009 = 9
AX = 0x27C0 = 10176
600000 = (9 * 65536) + 10176
```

After the multiplication, GDB showed:

```text
ax       0x27c0  10176
dx       0x9     9
eflags   0xa03   [ CF IF OF ]
```

The stored result was 600000. Looking only at `AX` would give 10176,
which is not the full answer. The program stores both `AX` and `DX`
to keep all 32 bits of the product.

| Flag | Displayed status | Explanation |
| --- | --- | --- |
| CF (carry) | Set (1) | 600000 is above the unsigned 16-bit maximum of 65535. The upper half, `DX`, is 9 rather than zero, so the product does not fit in `AX` alone. |
| OF (overflow) | Set (1) | For `MUL`, OF follows the same rule as CF. `DX` is nonzero, so OF is set too. |
| PF (parity) | Cleared (0), undefined | `MUL` does not define PF. The lowest byte, `0xC0`, has two 1s, but that does not guarantee PF will be set. |
| AF (auxiliary carry) | Cleared (0), undefined | `MUL` does not define AF, so I cannot explain this value as a carry or lack of carry between bits 3 and 4. |
| ZF (zero) | Cleared (0), undefined | `MUL` does not define ZF. The nonzero product is not a reliable reason for the displayed flag value. |
| SF (sign) | Cleared (0), undefined | `MUL` does not define SF, so it cannot reliably describe the product's sign. |

CF and OF being set does not mean the full answer was lost. The full
product fits in `DX:AX`; the flags tell me it does not fit in the lower
half alone.

## What I noticed about the flags

For unsigned `MUL`, CF and OF are both cleared when the upper half of
the product is zero, and both set when it is nonzero. PF, AF, ZF, and
SF are undefined. Their displayed values are observations from these
runs, not guaranteed results on every processor or run. Undefined does
not mean cleared or guaranteed to keep the old value.

IF was set in both runs. It is the interrupt enable flag and is not
changed by `MUL`, so it is not caused by the multiplication. The EFLAGS
values also include reserved bit 1, which is always set.