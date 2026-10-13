# PDP-1 Tiny BASIC

An integer BASIC interpreter for the DEC PDP-1 with 16K of core memory
(memory banks 0–3). It is written in MACRO-1 assembler and draws point graphics
on the Type 30 display. Developed and tested on the PiDP-1 and in the
*blincolnlights* emulator.

```
PDP-1 TINY BASIC
6143 WORDS FREE
>10 FOR I=1 TO 5
>20 PRINT I, I×I
>30 NEXT I
>RUN
1	1
2	4
3	9
4	16
5	25
>
```

> The source files, comments and the example programs are in German, for
> example `primzahlen.bas` (prime numbers) and `zahlenraten.bas` (number
> guessing). Messages printed by the interpreter are in English.

---

## Contents

1. [Overview](#1-overview)
2. [Quick start](#2-quick-start)
3. [Language reference](#3-language-reference)
4. [Programs on paper tape](#4-programs-on-paper-tape)
5. [Memory map](#5-memory-map)
6. [Inside the interpreter](#6-inside-the-interpreter)
7. [Files and tools](#7-files-and-tools)
8. [Development history](#8-development-history)
9. [Limitations](#9-limitations)
10. [Outlook](#10-outlook)

---

## 1. Overview

| Feature | Value |
|---|---|
| Target system | PDP-1 with memory extension, 16K words of 18 bits (banks 0–3) |
| Numbers | integers, 18-bit ones' complement, −131071 … +131071 |
| Variables | A–Z |
| Program memory | 6143 words (bank 2 and the lower half of bank 3) |
| Graphics | Type 30, 1024 × 1024 points, up to 1024 points at a time |
| Input/output | Flexowriter typewriter, Type 30 display, paper tape reader, paper tape punch |
| Options | the multiply/divide option (EAE) is detected at startup and used if present; without it the interpreter multiplies and divides in software |
| Code size | interpreter 0100–4027 in bank 0, 355 words of data in bank 1 |
| Load tape | `basic.tape`, 8410 tape lines, about 21 s to read |

The interpreter follows the Tiny BASIC dialects of the 1970s: line numbers, one
statement per line, direct mode and program mode. Program lines are tokenized
as they are entered and can be punched to paper tape and read back. With
`PLOT`, programs draw points on the Type 30 display, and the interpreter keeps
the picture on screen while the program runs.

---

## 2. Quick start

### 2.1 Building

Requirements: `macro1_1` (from `pidp1/src/macro`) and Python 3.

```sh
macro1_1 -d basic0.mac                              # bank 0, block format for READ-IN
macro1_1 -r -d basic1.mac                           # bank 1, RIM, bank-relative
python3 xtape.py --check basic0.lst 1:basic1.lst    # cross-check the addresses of both parts
python3 xtape.py -o basic.tape basic0.rim 1:basic1.rim
```

`macro1_1` always names its output `.rim`, even in block format. Only
`basic.tape` goes onto the machine.

### 2.2 Loading

1. Set the **EXTEND switch off** and the extension address switches to **0**.
2. Put `basic.tape` into the paper tape reader.
3. Press **READ-IN**.

After about 21 seconds the banner appears on the typewriter. The interpreter
switches into extend mode by itself.

**Restart** without reloading: START at address **0100**. This clears the
program and the variables.

### 2.3 Operation

| Input | Effect |
|---|---|
| `10 PRINT "HELLO"` | enter or replace line 10 |
| `10` | delete line 10 |
| `PRINT 2+3` | execute immediately (direct mode) |
| Backspace | take back the last character of the input line |
| Carriage return | finish the line |
| any key during RUN, LIST, SAVE, LOAD | abort with `BREAK` |

Keywords may be typed in lower or upper case. LIST shows them in upper case,
and the variable names too. The interpreter removes spaces outside of strings
as you type, and LIST puts one space back around each keyword. So
`20 print "x", a * b` becomes `20 PRINT "x",A×B`. Text in quotes and everything
after `REM` stays exactly as typed.

---

## 3. Language reference

### 3.1 Commands

Commands are normally typed in direct mode, but they may also appear in a
program.

| Command | Description |
|---|---|
| `RUN` | clear the variables and run the program from the first line |
| `LIST` | list the whole program |
| `LIST n` | list the program from line *n* onwards |
| `NEW` | clear the program, the variables and the display |
| `SAVE` | punch the program to paper tape (section 4) |
| `LOAD` | read a program from the paper tape reader (section 4) |

### 3.2 Statements

| Statement | Description |
|---|---|
| `[LET] v = expression` | assign a value; `LET` is optional |
| `PRINT item, item, …` | print; an *item* is an expression or `"text"` |
| `PRINT` | blank line |
| `INPUT v [, v …]` | read values, prompt `? ` |
| `IF expression THEN statement` | conditional statement; `THEN` is optional |
| `IF expression THEN line` | conditional jump |
| `GOTO expression` | jump; in direct mode, start the program at that line |
| `GOSUB expression` | call a subroutine |
| `RETURN` | return from the subroutine |
| `FOR v = a TO b [STEP c]` | counting loop, `STEP` may be negative |
| `NEXT [v]` | end of loop |
| `PLOT x, y` | put a point on the display (section 3.5) |
| `CLS` | clear the display |
| `REM text` | comment |
| `END` | end the program |

Details:

- **PRINT:** A comma moves to the next tab stop. A comma at the end of the
  line suppresses the line feed.
- **INPUT:** Each value is a full expression, for example `A+1` or `RND(6)`.
  Several values may be typed on one line, separated by commas. Missing values
  are asked for with a new `? `. An empty input (just RETURN) sets 0.
- **IF:** Any value other than 0 counts as true. If the condition is false,
  execution continues with the next line.
- **GOTO and GOSUB** accept computed line numbers, such as `GOTO 100+10*K`.
- **FOR/NEXT:** Loops may be nested. A loop always runs at least once. With a
  positive step it continues while *v* ≤ *b*, with a negative step while
  *v* ≥ *b*. A new `FOR` on the same variable replaces the old loop, so jumping
  out of a loop does not use up memory permanently. FOR is only allowed in a
  program.
- **GOSUB** may be recursive, up to 170 levels deep.

### 3.3 Expressions

Precedence from highest to lowest:

| Level | Operators |
|---|---|
| parentheses, functions, sign | `( )` `RND()` `ABS()` `-x` `+x` |
| multiplication, division | `×` (or `*`, `x`, `X`) and `/` |
| addition, subtraction | `+` `-` |
| comparison | `=` `<` `>` `<=` `>=` `<>` `#` |

- **Comparisons** return 1 (true) or 0 (false). Only one comparison is allowed
  per expression.
- **Division** truncates toward zero: `7/2` = 3, `-7/2` = −3.
- **Remainder** of a division: `A-A/B*B`.
- **Overflow** beyond ±131071 stops with `?OVERFLOW`.

### 3.4 Functions

| Function | Result |
|---|---|
| `RND(n)` | random number from 1 to *n*; *n* ≥ 1 |
| `ABS(x)` | absolute value of *x* |

The random number generator is the one from Spacewar!. Its seed keeps running
while the interpreter waits for a key, so every program run is different.

### 3.5 Graphics

`PLOT x, y` puts a point on the Type 30 display. The coordinates follow the
built-in coordinate system of the Type 30:

| | |
|---|---|
| Range | −511 … +511 for x and y |
| Origin | centre of the screen |
| Direction | x to the right, y upwards |
| Resolution | 1023 × 1023 points |
| Points at a time | 1024 |

- **Keeping the picture:** The Type 30 has no picture memory, and a point fades
  after about a quarter of a second. The interpreter therefore keeps every
  point in a display list and redraws it continuously while the program runs,
  including while it prints or waits for input at `INPUT`.
- **Fading:** When the interpreter returns to the `>` prompt, the display list
  is cleared and the picture fades as it would on a real screen. This happens
  at the end of the program, at `END`, after an error message, after `BREAK`
  and after every command typed in direct mode. A `PLOT x, y` typed directly
  therefore flashes once and fades.
- **Showing the picture at the end of a program:** A program whose picture
  should stay on screen waits at the end with `INPUT`. The interpreter then
  redraws at full rate, and RETURN ends the program:

  ```basic
  900 PRINT "PRESS RETURN TO END"
  910 INPUT Z
  ```

- **Clearing while running:** `CLS` clears the picture.
- **Errors:** Coordinates outside ±511 give `?ILLEGAL VALUE`, the 1025th point
  gives `?DISPLAY FULL`.
- **Refresh rate:**

  | Situation | points redrawn per second (emulator) |
  |---|---|
  | waiting at `INPUT`, printing | about 10,000 |
  | while the program computes (32 points per program line) | about 4,000 |

  With many points the picture therefore flickers a little while the program
  computes, and becomes steady once the program waits at `INPUT`.
- **Speed:** Programs without graphics are not slowed down by the refresh.
  Programs with many points spend a large part of their time redrawing: for
  the Lissajous figure with 1001 points it is almost half.

Example, a circle drawn with Minsky's circle algorithm, which originated on the
PDP-1:

```basic
10 CLS
20 X=400
30 Y=0
40 FOR I=1 TO 101
50 PLOT X,Y
60 X=X-Y/16
70 Y=Y+X/16
80 NEXT I
90 INPUT Z
```

**Adapting programs from other BASICs.** Many graphics programs were written
for floating-point BASICs and screens 40 to 320 points wide. Here the
arithmetic is integer, and `B/X` is 0 as long as B is smaller than X. Such
programs therefore need to be scaled up:

- Multiply all amplitudes and lengths by a factor such as 10 to 20, so the
  divisions stay accurate enough.
- Put the origin at the centre of the screen, i.e. drop offsets such as `32+…`.
- Watch intermediate values: `X*X` already overflows at X = 363.
  `(X/2)*(X/2)/100` gives the same as `X*X/400` without overflowing.

`lissajous.bas` is such an adapted program. The original used amplitudes of 15
and 6 and a centre at (32, 22), which here produced only a square 30 points
across. Multiplied by 20 and centred on 0, 0, it gives the proper figure of
1001 points.

### 3.6 Characters on the Flexowriter

The Flexowriter has no asterisk, no semicolon and no colon.

| wanted | type | note |
|---|---|---|
| multiply | `×`, or `*` on a PC | `x` and `X` are also read as "times" in this position |
| not equal | `<>` | `#` also works, but prints as `⊃` |
| separator in PRINT | `,` | there is no `;` |
| several statements per line | — | not possible |

In an expression a letter can never follow a value directly. `DXD` therefore
unambiguously means "D times D", and the variable X remains usable.

### 3.7 Error messages

When a program is running, the line number follows, for example
`?DIVISION BY ZERO IN 60`.

| Message | Cause |
|---|---|
| `?SYNTAX ERROR` | statement not understood |
| `?OVERFLOW` | result or number outside ±131071 |
| `?DIVISION BY ZERO` | division by 0 |
| `?UNDEFINED LINE` | GOTO/GOSUB to a line that does not exist |
| `?RETURN WITHOUT GOSUB` | RETURN without an open GOSUB |
| `?NEXT WITHOUT FOR` | NEXT without a matching FOR |
| `?ILLEGAL VALUE` | line number 0, `RND(n)` with *n* < 1, PLOT outside ±511 |
| `?ILLEGAL DIRECT` | FOR in direct mode |
| `?OUT OF MEMORY` | program memory full |
| `?DISPLAY FULL` | more than 1024 points on the display |
| `?STACK OVERFLOW` | nested too deeply (GOSUB, FOR or expression) |
| `BREAK` | aborted by a key press |

---

## 4. Programs on paper tape

### 4.1 SAVE

`SAVE` punches the program as readable text, in this order:

1. 40 lines of blank tape as leader
2. shift to lower case (072)
3. the program, exactly as LIST prints it
4. stop code (013)
5. 40 lines of blank tape as trailer

A key press aborts; output then goes back to the typewriter.

### 4.2 LOAD

`LOAD` clears the program, the variables and the display, then reads the tape
line by line. Each line is entered as if it had been typed.

- At the **stop code** reading ends with the message `n LINES LOADED`.
- **Lines without a number** and blank lines are skipped.
- **If the stop code is missing,** a key press ends reading. The lines read up
  to that point are kept.
- `LOAD` may be typed **before** the tape is in the reader. The interpreter
  waits for the tape.
- **Errors** while reading, such as `?ILLEGAL VALUE` for line 0, end LOAD.

### 4.3 Tape format

The format is that of genuine PDP-1 text tapes. The PiDP-1's Lisp tape
`lisp-defs.pt` served as reference.

| Element | Punching |
|---|---|
| character | 6-bit FIO-DEC code in holes 1–6 |
| hole 8 | odd parity over the six data holes |
| space | 200 (code 0 with parity hole) |
| blank tape | 0, skipped |
| rubbed-out line | 377, skipped |
| upper/lower case | shift codes 074 / 072 |
| end of line | 077 (277 with parity) |
| end of tape | stop code 013 |

### 4.4 Exchange with a PC

`bastape.py` converts in both directions:

```sh
python3 bastape.py zahlenraten.bas zahlenraten.tape    # text file → tape
python3 bastape.py -d gesichert.tape gesichert.bas     # tape → text file
```

- `*` becomes `×` and turns back into `*` on the way back.
- A tape punched with `SAVE` is byte-for-byte identical to the one
  `bastape.py` produces from the same file.
- The PiDP-1 tools `encode_fiodec` and `decode_fiodec` use the same format.
  Answer `y` when `encode_fiodec` asks about the stop code.

---

## 5. Memory map

The interpreter runs in **extend mode**:

- Direct addresses only reach the bank the code is running in, i.e. bank 0.
- Data in banks 1–3 is accessed through pointer words holding full 16-bit
  addresses (`lac i pointer`). In this mode indirection is single-level.

### 5.1 Overview

| Bank | Addresses | Contents | on tape |
|---|---|---|---|
| 0 | 00000–07777 | interpreter, constants, variables, loader | yes |
| 1 | 10000–17777 | tables, messages, buffers, stacks | tables and messages |
| 2 | 20000–27777 | program memory | no |
| 3 | 30000–33777 | program memory (continued) | no |
| 3 | 34000–37777 | display list for PLOT | no |

### 5.2 Bank 0: interpreter

| Addresses | Contents |
|---|---|
| 0000–0003 | reserved for the sequence break (unused) |
| 0004–0077 | free |
| 0100–1647 | start-up with EAE detection, main loop, dispatch table, error handling, line editor, all statements including LIST, PLOT, CLS, SAVE and LOAD |
| 1650–2472 | expression evaluation (`expr`, `aexpr`, `trm`, `factor`), comparison, multiplication, division, random numbers |
| 2473–3502 | number input and output, line input, character classes, tokenization (`crunch`, `gettok`, `prtok`, `kwfind`), typewriter and punch (`inch`, `outch`, `rawch`, `punch`), text output, display refresh (`clsdl`, `dslice`) |
| 3503–3634 | constants and pointers into banks 1–3 |
| 3635–4027 | interpreter variables |
| 4030–7677 | **free, 1960 words** |
| 7700–7737 | `xld`, loader for banks 1–3 (needed only while reading in) |
| 7740–7750 | free |
| 7751–7777 | macro1_1 bootstrap loader, **do not use** |

The section boundaries come from the listing `basic0.lst` and move with every
code change. `xtape.py --check` reports the end of the code (`endcod`) and warns
if it reaches `xld`.

### 5.3 Bank 1: data

| Addresses | Contents | Size | on tape |
|---|---|---|---|
| 10000–10031 | variables A–Z | 26 words | no (cleared by RUN/NEW) |
| 10100–10377 | messages, 16 words each | | yes |
| 10400–10602 | keyword table (23 entries), room up to 10777 | 131 words | yes |
| 11000–11177 | input buffer for program lines | 126 characters | no |
| 11200–11377 | input buffer for INPUT | 126 characters | no |
| 11400–11577 | further messages | | yes |
| 11600–11777 | character classes for all 128 characters | 128 words | yes |
| 12000–12777 | GOSUB stack, 3 words per level | 170 levels | no |
| 13000–13777 | FOR stack, 5 words per loop | 102 loops | no |
| 14000–17777 | expression evaluation stack | 2048 words | no |

The areas that are not on the tape are cleared at start-up or only written at
run time.

**Keyword table format:** Each entry consists of the letters of the word (one
FIO-DEC code per word, without the case bit), then `777777` and the token.
`777776` ends the table.

**Character classes:** indexed by the 7-bit character.

| Value | Class |
|---|---|
| `100+n` | letter *n* (0 = A … 31 = Z) |
| `200+n` | digit *n* |
| `400` | case-independent character (space, tab, backspace, carriage return) |
| `0` | any other character |

### 5.4 Banks 2–3: program memory and display list

Program lines are stored in ascending order from 20000 and may extend up to
33777. Each line is one entry:

```
+0   line number
+1   length of the entry in words (2 + text including 077)
+2   text, one word per element:
       character    0 … 177
       token        200 + number
       number       400000 + value (value 0 … 131071)
     ...
     077  end of line
```

After the last line comes a word 0 as end marker. The pointer `ptop` points to
it; everything after it up to 33777 is free. Thanks to the length word, GOTO,
GOSUB and moving to the next line jump from entry to entry without searching
the text.

Example: `20 PRINT I*12` takes 7 words: `24` (20 decimal), `7`, `201` (PRINT),
`171` (I), `173` (×), `400014` (number 12) and `77`.

Numbers above 131071 stay as digits and report `?OVERFLOW` at run time.

**Display list (34000–37777):** Each point takes two words, first Y, then X,
both already shifted left by 8 bits. This is the form in which `dpy` expects
the coordinate, in the upper 10 bits of AC and IO. The pointer `dlend` points
to the next free slot, `dcur` to the next point to redraw. 2048 words hold
1024 points.

### 5.5 Tokens

| Token | Word | Token | Word | Token | Word |
|---|---|---|---|---|---|
| 1 | PRINT | 10 | NEXT | 17 | THEN |
| 2 | LET | 11 | INPUT | 20 | TO |
| 3 | IF | 12 | REM | 21 | STEP |
| 4 | GOTO | 13 | END | 22 | RND |
| 5 | GOSUB | 14 | RUN | 23 | ABS |
| 6 | RETURN | 15 | LIST | 24 | SAVE |
| 7 | FOR | 16 | NEW | 25 | LOAD |
| | | | | 26 | PLOT |
| | | | | 27 | CLS |

All values are octal. In memory a token is stored as 200 + number.

---

## 6. Inside the interpreter

### 6.1 Two-stage loading

`macro1_1` only produces 12-bit addresses and therefore cannot write banks 1–3
directly onto a tape. The load tape consists of two parts:

1. **Block tape for bank 0**, produced by `macro1_1`. READ-IN loads the
   bootstrap into 7751–7777. The bootstrap loads the interpreter and `xld`. The
   last word jumps to `xld` (7700).
2. **X-blocks for banks 1–3**, appended by `xtape.py`. `xld` switches into
   extend mode with `eem` and reads the blocks:

   ```
   n  address16  data[n]  checksum      (XOR over n, address and data)
   ```

   A word `0` ends the tape, after which `xld` jumps to the start at 0100.

   On a checksum error the machine halts at 7733. CONTINUE reads the next
   block.

The sources for bank 1 are assembled bank-relative (0000–7777). `xtape.py`
relocates them by 10000 when appending. Bank 0 knows the bank-1 addresses
through equates such as `kwtab=10400`. `xtape.py --check` compares these with
the `basic1` listing.

### 6.2 Calling conventions

| Kind | Call | Mechanism |
|---|---|---|
| no argument | `jsp sub` | `sub, dap ret` stores the return address in the exit |
| with argument | `jda sub` | argument in AC; `jda` stores it in `sub` and jumps to `sub+1` |
| recursive | `jsp expr` | return address pushed onto the software stack with the `pushm` macro, exit through `rexit` with `popm` and `jmp i` |

The result is always in AC. The PDP-1 has neither a stack nor index registers.
`pushm` (`dac i sp`, `idx sp`) and `popm` (`law i 1`, `add sp`, `dac sp`,
`lac i sp`) are macros, so they are expanded inline instead of being called.
Stack overflow is checked only in `factor`, which every deeper nesting passes
through. Only the four expression routines (`expr`, `aexpr`, `trm`, `factor`)
are reentrant; all other subroutines are not. These four also push left
operands and operators onto the stack. In extend mode the return address
delivered by `jsp`/`jda` includes the bank bits, and `jmp i` returns with a
full 16-bit address.

### 6.3 Tokenization

1. After every input, `crunch` processes the line in place in the buffer. The
   result is never longer than the input.
   - Keywords become tokens.
   - Numbers are converted into a number word (400000 + value). The set sign
     bit distinguishes it from all characters and tokens.
   - Spaces and tabs are dropped.
   - Lower-case letters become upper case.
2. Text in quotes and everything after REM remain unchanged.
3. At run time the interpreter recognises a token or a number with a single
   test. Numbers no longer need converting, and spaces no longer need skipping.
4. LIST writes tokens out with `prtok` and numbers with `prnum`, and puts the
   spaces back around keywords.

### 6.4 Speed

Measured in the emulator: the prime number program up to 200 without printing,
and `lissajous.bas` up to its input prompt.

| Version | Primes | Lissajous |
|---|---|---|
| text rescanned on every execution (steps 2/3) | 74.5 s | – |
| keywords tokenized, line length (step 4) | 15.8 s | – |
| with graphics (step 6) | 16.0 s | 52.0 s |
| numbers and spaces handled at entry, stack as macros, without EAE (step 7) | 10.1 s | 43.6 s |
| the same with EAE | **8.9 s** | **40.1 s** |

Samples of the program counter, taken through the debug service, showed where
the time goes:

- **Before step 7:** a good third of the time in `push`/`pop`/`rexit` and
  almost a fifth in `skpsp`.
- **For the Lissajous figure after step 7:** almost half in redrawing the
  display list (`dslice`).

A BASIC statement still costs a few hundred machine instructions. Compared with
assembler, a factor of the order of 100 remains.

### 6.5 Arithmetic

- **Ones' complement:** There is +0 (`000000`) and −0 (`777777`). Every `cma`
  is followed by `sad m0 / cla` so that comparisons and `sza` work.
- **Overflow** is detected with the overflow flag, which `szo` tests and
  clears.
- **EAE detection:** At start-up the interpreter computes `3 × 5 / 5` with
  `mul` and `div`. If the result is 3 and `div` skips the following instruction
  as it should, the multiply/divide option is present (`eae` = 1). Without the
  option the same opcodes are the step instructions `mus`/`dis`, which are
  harmless on every PDP-1; the test then fails.
- **Multiplication with EAE:** `mul`, then `scr 1s`. The product is then a
  double word in AC:IO and fits if AC = 0 and IO is positive.
- **Division with EAE:** move the dividend, as the double word `0 : value`, into
  the EAE position with `scl 1s`, then `div`. The quotient is in AC and the
  remainder in IO (used for `RND`).
- **Without EAE:** multiplication by shifting and adding. Division: double the
  divisor until it no longer fits into the dividend, then subtract place by
  place.
- **Signs** are handled separately in both cases; the interpreter computes with
  absolute values.
- **Comparison:** with different signs the sign decides, otherwise the
  difference. This way the comparison cannot overflow.
- **Decimal output:** repeated subtraction of the powers of ten from 100000
  down to 1.

### 6.6 Graphics and display refresh

- **Plotting:** `PLOT` checks both coordinates (`pchk`), shifts them into the
  upper 10 bits with `sal 8s` and appends them to the display list. It then
  draws the point once immediately. `sal` fills with the sign bit, which matches
  the Type 30's ones' complement coordinates.
- **Redrawing:** Each call of `dslice` draws up to 16 points starting at `dcur`
  and wraps around at the end of the list. With an empty list it returns
  immediately. It is called from three places:

  | Caller | Occasion |
  |---|---|
  | `inch` | in the loop waiting for a key press |
  | `rawch` | during a keystroke: `tyo` without waiting, then refresh until `cks` reports the typewriter ready again |
  | `runat` | at the start of every program line, twice (32 points) |

- **Fading:** `loop`, the return to the prompt, clears the display list with
  `clsdl`. Without redrawing, the points fade by themselves after about a
  quarter of a second.
- **Point output:** `dpy` without waiting (`720307`) at intensity 3, the
  brightest level. The loop takes about 95 µs per point, longer than the 50 µs
  the Type 30 needs for one point.

### 6.7 Typewriter and character set

- **Shared shift state:** The Flexowriter has a single shift state for typing
  and printing. The variable `tcase` tracks it. It is kept up to date by
  `inch` (input), `rawch` (output) and `brk` (the key discarded on BREAK).
- **Character format:** Internally each character is 7 bits: case bit 100 plus
  the FIO-DEC code. Space, tab, backspace and carriage return have no case bit.
- **Output target:** During SAVE, `rawch` redirects output to `punch`, with its
  own shift state. `unpm` switches back to the typewriter afterwards, also when
  an error occurs.
- **Reader:** LOAD reads with `rpa` without waiting and polls the status with
  `cks`, so that a key press can abort at any time. A leftover character is
  discarded. If nothing arrives for about 0.1 s, `rdch` restarts the reader.
  This keeps LOAD from hanging after an abort.

---

## 7. Files and tools

| File | Contents |
|---|---|
| `basic0.mac` | interpreter and loader, bank 0 (MACRO-1, about 2300 lines) |
| `basic1.mac` | messages, keywords, character classes, bank 1 |
| `xtape.py` | assembles the load tape, checks addresses (`--check`) |
| `bastape.py` | converts BASIC programs between text files and paper tape |
| `basic.tape` | ready-made load tape of the interpreter |
| `primzahlen.bas` / `.tape` | example: prime numbers up to 200, ten per line |
| `zahlenraten.bas` / `.tape` | example: number guessing with `RND` and `INPUT` |
| `kreis.bas` / `.tape` | example: Minsky circle on the Type 30 |
| `parabel.bas` / `.tape` | example: function graph with axes on the Type 30 |
| `lissajous.bas` / `.tape` | example: Lissajous figure from two coupled oscillations |

### Example program: number guessing

`zahlenraten.bas` as it is on tape, with German texts. In English: "I am
thinking of a number from 1 to 100", "too small", "too big", "correct after …
tries".

```basic
10 REM ZAHLENRATEN
20 N=RND(100)
30 T=0
40 PRINT "ICH DENKE MIR EINE ZAHL VON 1 BIS 100."
50 INPUT G
60 T=T+1
70 IF G<N THEN PRINT "ZU KLEIN"
80 IF G>N THEN PRINT "ZU GROSS"
90 IF G<>N THEN 50
100 PRINT "RICHTIG NACH", T, "VERSUCHEN"
110 END
```

### Test environment

Development used the `pdp1` emulator from *blincolnlights*, started with `-t`.
It runs without display window and panel, with the typewriter on port 1041 and
the debug service on port 1040. Through the debug service, tapes are mounted
(`r file`, `p file`), READ-IN is triggered and breakpoints are set on labels
from the listing. Every development step was checked there with the tape
loaded and real keyboard input.

---

## 8. Development history

### Step 1: Skeleton and loader

- Memory layout across the four banks.
- Two-stage loading with `xld` and `xtape.py`; the bootstrap at 7751–7777 stays
  untouched.
- Line input with backspace, number input and output, `PRINT`, `REM`, `NEW`,
  `END`.
- Software stack in preparation for recursion.

### Step 2: The language

- Recursive expression parser with precedence, comparisons, `RND` and `ABS`.
- Multiplication and division in software.
- Program memory with inserting, replacing and deleting lines.
- `LIST`, `RUN`, `GOTO`, `IF/THEN`, `GOSUB/RETURN`, `FOR/NEXT`, `INPUT`, `LET`.
- Abort by key press.
- First working programs: prime numbers and number guessing.

### Step 3: Adjustments after first use on the PiDP-1

- **Multiply sign:** The user's typewriter connection sent an `x` for `*`.
  Since then `x` and `X` count as "times" in operator position.
- **Error messages:** The terse messages `WHAT?`, `HOW?` and `SORRY` were
  replaced by ten specific messages.

### Step 4: Tokenization, SAVE and LOAD

- Keywords become tokens on entry.
- Every program line stores its length.
- This made the prime number program about 4.7 times faster.
- SAVE and LOAD in genuine FIO-DEC tape format.
- `bastape.py` for exchanging programs with a PC.

### Step 5: Graphics on the Type 30

- `PLOT x, y` and `CLS`.
- Display list for 1024 points in bank 3; program memory was reduced to 6143
  words to make room.
- Continuous display refresh while waiting, printing and running.
- The character class table moved to 11600 because the grown keyword table
  would otherwise have run into it. `xtape.py` had reported the overlap while
  building the tape.
- Verified with captures of the display data stream (port 3400): position of
  the points, completeness of the picture during LIST and during computation,
  `CLS`, `NEW`, `?DISPLAY FULL`.

### Step 6: Graphics after first use

The first version kept every picture until `CLS` or `NEW`. Points typed in
direct mode stayed as well, and a second run of a program with many points
ended with `?DISPLAY FULL`. That matched neither the behaviour of a Type 30 nor
what the user expected when testing.

- The display list now only lives while a program runs. On return to the
  prompt the picture fades.
- `INPUT` accepts an empty input (0). A program can thus keep its picture on
  screen until RETURN is pressed.
- 32 instead of 16 points are redrawn per program line.
- The Lissajous test program was scaled up for integer arithmetic.

### Step 7: Speed

- Numbers are converted to binary values on entry.
- The multiply/divide option is detected and used.
- A profile from program counter samples then showed the real bottlenecks:
  stack handling and skipping spaces. So `push`/`pop` became macros, and spaces
  are dropped on entry.
- Result: the prime number program runs in 8.9 instead of 16.0 s.
- Side effect: LIST shows normalised spacing.

### Bugs found and fixed

| Bug | Cause | Fix |
|---|---|---|
| `BREAK` did not work on `20 GOTO 20` | only checked when moving to the next line, not after jumps | check in `runat`, which every line passes through |
| after BREAK, `/` instead of `?` and `8` instead of `>` | the discarded key was a shift code, `tcase` was out of step | `brk` evaluates shift codes |
| LOAD hung after an aborted LOAD | read still pending; another `rpa` switches the reader clutch off again | discard the stale character, restart the reader after a timeout |
| error message right after output on the same line | no line feed before the message | column flag `col`; messages always start on a new line |

---

## 9. Limitations

- Integers only, from −131071 to +131071; no floating point.
- Variables are single letters; there are no arrays and no string variables.
- One statement per line.
- At most one comparison per expression; no logical operators AND/OR.
- Input lines are limited to 126 characters.
- Keywords are recognised even without spaces around them. `GO TO` with a space
  is therefore not allowed, and a sequence of letters that spells a keyword
  becomes that keyword.
- Spaces outside of strings are not stored. LIST therefore shows the program
  with normalised spacing, not exactly as typed.
- Typing faster than the interpreter can process may lose characters, for
  example when pasting a whole program. For longer programs, LOAD from tape is
  the intended route.

---

## 10. Outlook

Possible next steps; 1960 words are still free in bank 0:

- `LINE x1, y1, x2, y2` for drawing line segments
- `MERGE` to add a program from tape to the one in memory
- Trigonometry using the sine routine from Spacewar!, scaled to integers
