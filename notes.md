ml-101: A Quick Intro to Machine Language on the Commodore 64
=============================================================

Units
-----

### $00

Computers work in binary; Humans work in decimal. They don't
match up well, but hexadecimal is a nice compromise.

    0000     0      $0
    0001     1      $1
    0010     2      $2
    1001    10      $A
    1111    15      $F

### $01

Commodore BASIC always wants to work in decimal, which is a pain. It
has no [`&hF3`][necpc-10] or [`HEX$(n)`][necpc-36], like NEC PC-8001
BASIC.

So we can work in hex, and get the computer to do conversions for us
when we need to convert, I've writen a little BASIC program, "Machine
Language Assistant." This is on disk image [`ml-101.d64`](ml-101.d64):
`load "ml assistant",8`. The source code is also available as text
in [`ml assistant.txt`](ml\ assistant.txt).

### $02

Oddly enough, Commodore BASIC does use binary for `AND`, `OR`, etc.

    print 11 and 6      :rem 1011 & 0110 ⇒ 0010   $B & $6 ⇒ $2
    print 10 or 4       :rem 1010 | 0100 ⇒ 1110   $A | $4 ⇒ $E

### $03

`$C000`-`$CFFF` is a nice unused place [in the address map][se-addr]
to put programs and data. We'll use `$C000` for data and `$C1000` for
our programs.

The `SYS` command in basic jumps to a machine language routine, just
like the `JSR` (jump to subroutine) machine-language instruction does.
If we examine `$C100` we see it contains `$00`, a `BRK` instruction.
We can G)OTO it and we get a soft reset; that's how C64 BASIC works.
If we convert it to decimal we can confirm that `SYS 49408` does the
same thing.

### $04

Let's put the smallest possible program there:

    C100 60                 RTS     ; return from subroutine

Examine to see that `$C1000` is all zeros, deposit the opcode there,
examine again to see that it's there. Now G)OTO it; nothing happens,
as expected, because it just returned without doing anything. Confirm
that `SYS 49408` also now does nothing.

### $06

In our routine we can call other routines using the `JSR` instruction.
This takes a _word_ (16-bit) address; the 6502 (like the 8080; unlike
the 6800) is _little-endian_ so memory order is from LSB to MSB. Thus
we "reverse" the address when assembling it into memory.

    C100 20 F9 AB           JSR $ABF9   ; BASIC ROM: do input prompt
    C103 60                 RTS

### $07

A [standard kernal routine][c64w-kernal] is [`CHROUT`][c64w-chrout].
It takes a [PETSCII] code in the A register and prints it. We use
`LDA` with _immediate_ addressing mode to load a constant into the A
register.

    C100 A9 5C              LDA #$5C    ; pound sign
    C102 20 D2 FF           JSR $FFD2   ; CHROUT
    C105 A9 0D              LDA #$0D    ; carriage return
    C107 20 D2 FF           JSR $FFD2   ; CHROUT
    C10A 60                 RTS

### $08

Using the _absolute_ addressing mode we can access a location in
memory. And of course there's a parallel `STA` instruction to store
the A register, with the same addressing modes. Note how opcodes
are different for different adressing modes, though the high-level
assembler instruction is the same.

The [screen memory][c64w-screen] starts at `$0400`. At 40 char/line,
12 lines would be 480 bytes, convert to hex and add to the screen base
location: `$400 + $1E0 = $5E0`:

    C100 AD E0 05           LDA $05E0
    C103 09 80              ORA #$80    ; set high bit
    C105 8D E1 05           STA $05E1   ; store next door
    C108 60                 RTS

### $09

BASIC sidetrack: invert characters in screen RAM.

    20000 for i = 1504 to 1504+255 : rem $05e0 to $06df
    20010 c = peek(i)
    20020 c = c or 128 : rem $80
    20030 poke i,c
    20040 next i

This gives you a sense of BASIC's speed.

### $0A

Using an _indexed_ addressing mode, we can add the X or Y register to
a value. This is handy in loops.

BNE checks the Z (zero) flag which is `1` (set) if a recent operation
produced zero, or `0` (reset) if not. (Compares do subtraction,
comparing 13 to 13 is 13-13=0; comparing 13 to 7 is 13-7=6.) Note that
in the opcode chart, `INX` is marked as affecting the zero flag.

Unlike `JSR` and `JMP`, branch instructions are _relative_ to the next
instruction. In our case, that's `$C10D`, and we need to go back to
`$C102`; so subtract to get `$-B`, which in 2s complement (count down
from -1, `$FF`) is `$F5`.

    C100 A2 00              LDX #$00
    C102 BD E0 05   loop    LDA $05E0,X
    C105 09 80              ORA #$80
    C107 9D E0 05           STA $05E0,X
    C10A E8                 INX
    C10B D0 F5              BNE loop
    C10D 60                 RTS

Note the speed difference!

### $FF

Carry on with whatever's interesting next.
- Passing registers to and getting values back from `SYS`?
  A:`$030C`, X:`$030D`, Y:`$030E`, P:`$030F`.
- `USR(n)` (jump address at `$0311`) and the FAC?
- Clever use of BASIC in ML, even using BASIC error handling?
  ["Borrowing ML from BASIC."][pickett85]


<!-------------------------------------------------------------------->
[c64w-chrout]: https://www.c64-wiki.com/wiki/CHROUT
[c64w-kernal]: https://www.c64-wiki.com/wiki/Kernal
[necpc-10]: https://archive.org/details/PC8001600100160011982/page/n11
[necpc-36]: https://archive.org/details/PC8001600100160011982/page/n37
[opcode]: http://www.oxyron.de/html/opcodes02.html
[petscii]: http://sta.c64.org/cbm64pet.html
[pickett85]: https://www.atarimagazines.com/compute/issue67/292_1_Readers_Feedback_Borrowing_ML_From_BASIC.php/
[se-addr]: https://github.com/0cjs/sedoc/blob/master/8bit/cbm/address-decoding.md
[c64w-screen]: https://www.c64-wiki.com/wiki/Screen_RAM
