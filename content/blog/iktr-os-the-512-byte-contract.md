+++
title = "iktr OS: The 512-Byte Contract"
date = 2026-10-06
description = "Writing an x86 bootloader from zero: the boot signature, a 40-year-old A20 bug, the GDT, and the one far jump that actually puts the CPU into 32-bit Protected Mode."
draft = false

[taxonomies]
tags = ["rust", "osdev", "x86", "assembly", "bootloader", "low-level"]
+++

I'm building an operating system called [iktr](https://github.com/victorchukwuemeka/iktr).
Phase 1 is done: the machine now boots, leaves 16-bit Real Mode, and lands in
32-bit Protected Mode. It's 512 bytes of assembly, and it prints two lines.

I want to write down what those 512 bytes actually do, because this is the part
of the project where the gap between "it works" and "I understand it" is widest.
Everything below is checked against the real binary — I assembled it and read the
disassembly rather than trusting the description.

## The premise

When you press the power button, thousands of layers of abstraction stand between
that and your code. In Phase 1 I removed all of them and wrote the first thing
the CPU executes. Within a strict 512-byte limit:

1. Take control of the machine right after the BIOS finishes POST.
2. Say goodbye to 16-bit Real Mode, an artifact of the 1978 Intel 8086.
3. Work around a 40-year-old hardware compatibility quirk (the A20 line).
4. Describe memory to the CPU using the Global Descriptor Table.
5. Flip the CPU into 32-bit Protected Mode and write to video memory directly.

## Act I: the 512-byte contract

### What the BIOS does before you get a single instruction

On power-up:

1. The motherboard runs firmware from ROM — BIOS, or UEFI in legacy/CSM mode.
2. The firmware runs POST (Power-On Self-Test).
3. It walks the boot devices looking for a bootable sector: the first 512 bytes
   of a drive. On a classic BIOS system that's the Master Boot Record.
4. It checks the **last two bytes** of those 512 bytes. If and only if they are
   `0x55 0xAA`, the firmware treats the sector as bootable, loads it into
   physical memory at `0x7C00`, and jumps the CPU to it.

That two-byte check is the whole protocol. There is no header, no magic string,
no format. The contract is: *your code plus data must fit in 510 bytes, and you
must sign the last two.*

```text
+--------------------------------------------------+--------+
|         bootloader code and data (max 510)       | 55  AA |
+--------------------------------------------------+--------+
 = exactly 512 bytes
```

In source that's one directive:

```asm
.fill 510 - (. - _start), 1, 0    # pad with zeros up to byte 510
.byte 0x55, 0xAA                   # the signature
```

`. - _start` is the bytes emitted so far. `.fill N, 1, 0` writes `N` zero bytes.
If your code ever grows past 510 bytes, the expression goes negative and the
assembler errors out. The limit enforces itself.

The binary I actually built:

```text
size            512 bytes
real content    212 bytes   (offsets 0 - 211)
padding         298 zeros
signature       55 aa
```

212 bytes of code and data, and more than half the sector is padding.

### Where you start: Real Mode

The CPU begins in 16-bit Real Mode. The name is doing a lot of work — it means
*real addressing*, i.e. the original 8086 scheme, still used today only because
it's the one state the CPU is guaranteed to be in after reset.

- Registers are 16 bits wide: `AX`, `BX`, `SI`, and the rest.
- Addressing is segmented: `physical = segment × 16 + offset`.
- Addressable memory is about 1 MB. (Strictly, `FFFF:FFFF` computes to
  `0x10FFEF` — 1 MB + 64 KB - 16 — so a little over. The A20 problem below is a
  direct consequence of that wraparound existing.)
- There is no memory protection and no privilege separation. Every instruction
  can write every byte of RAM.

### Step 1: sanitise the environment

The BIOS leaves segment registers holding whatever it last used, and you cannot
predict it. So the first thing the bootloader does is zero them and place the
stack:

```asm
cli                     # maskable interrupts off during setup
xorw %ax, %ax           # AX = 0
movw %ax, %ds           # data segment
movw %ax, %es           # extra segment
movw %ax, %ss           # stack segment
movw $0x7C00, %sp       # stack grows down from just under the bootloader
sti
```

Two details worth naming, because they're the kind of thing that gets skipped:

- `xorw %ax, %ax` assembles to `31 c0`, which is `xor %eax, %eax`. It clears all
  32 bits, not just the low 16. Harmless here, slightly more useful than
  intended.
- The stack starts at `0x7C00` and grows *downward*, into the low memory below
  the bootloader, which is free at this point.

Zeroing `DS` matters for the next step: the message label is a link-time absolute
address around `0x7C86`, and `lodsb` reads `DS:SI`. `DS = 0` makes `SI` a plain
physical address.

### Step 2: printing with BIOS interrupts

In Real Mode the BIOS provides a library of routines through the Interrupt
Vector Table — a table of 256 function pointers at address `0x0000`. We use
`int $0x10` with `AH = 0x0E`, the teletype print service:

```asm
movw $rm_msg, %si
movb $0x0E, %ah
xorw %bx, %bx           # BH = page number, BL = foreground colour

rm_print:
    lodsb               # AL = [DS:SI], SI += 1
    testb %al, %al      # null terminator?
    jz switch_to_pm
    int $0x10           # BIOS: print AL, advance cursor
    jmp rm_print
```

This is the last piece of BIOS code that will ever run.

## Act II: escaping 1978

Real Mode can't run a kernel. 1 MB of address space, no rings, no isolation.
Moving to 32-bit Protected Mode takes five architectural steps, and each one
exists because of a specific failure if you skip it.

```text
 1. cli                    interrupts off, permanently
 2. enable A20 (port 0x92) stop addresses wrapping at 1 MB
 3. lgdt                   hand the CPU a memory map
 4. set CR0.PE             the actual mode switch
 5. ljmp $0x08, $pm_start  make the CPU believe it
 6. write to 0xB8000       proof of life
```

### Step 1: cutting off BIOS

Once you're in 32-bit mode, the 16-bit Interrupt Vector Table at `0x0000` is
meaningless. If a hardware interrupt fires and the CPU tries to vector through a
table it no longer knows how to interpret, you get a **triple fault** — the CPU
fails to handle the fault, fails to handle *that* fault, and resets. So:
`cli`, and interrupts stay off for the rest of this post.

### Step 2: the A20 line

This one is a genuine historical scar.

The original 8086 had 20 address lines, `A0` through `A19`. Twenty bits is
1,048,576 addresses — 1 MB. Addresses above that **wrapped around to zero**.
Programmers noticed, and some wrote code that relied on it.

The 80286 arrived with 24 address lines. Suddenly address `0x100000` was a real,
distinct location instead of `0x000000`. Software that depended on the wrap broke.

IBM's fix was to keep the bug alive on purpose: a gate on the 21st address line
(`A20`), held low unless the operating system explicitly asked for it. Old
software kept wrapping. New software opted in. The gate was originally wired
through the 8042 keyboard controller, which is why "enabling A20" still involves
port I/O four decades later.

We use the later, simpler route — the Fast A20 gate on System Control Port
`0x92`, where bit 1 is the gate:

```asm
inb $0x92, %al
orb $0x02, %al          # set bit 1
outb %al, $0x92
```

### Step 3: the Global Descriptor Table

This is the part that took the longest to make sense of, so here's the model
that finally worked for me.

In Real Mode, a segment register holds a number that gets multiplied by 16. In
Protected Mode, a segment register holds a **selector** — an index into a table
the CPU maintains. The table is the Global Descriptor Table, and each entry is
8 bytes describing one segment:

```text
 31          24 23          16 15           0        (base, split awkwardly)
+--------------+--------------+------------------+
| base 31..24  | flags|limit  |  base 23..16     |  ...
+--------------+--------------+------------------+
... access byte | base 15..0                     |  limit 15..0
+--------------+-------------------------------+--+
```

The field layout is genuinely ugly, because the x86 architecture grew by
patching a table format in place across several generations of processors. The
pieces you care about:

- **Base** — where the segment starts.
- **Limit** — how far it reaches.
- **Access byte** — present flag, privilege ring (0 = kernel, 3 = user),
  code vs. data, readable/writable.
- **Flags** — granularity (1 byte vs. 4 KB units) and 16-bit vs. 32-bit.

We set up a **flat model**: every segment covers the entire 4 GB, base 0. This
sounds like it defeats the point, and right now it does — see the last section.

The three entries in `bootloader.asm`:

| Entry | Selector | Type | Access byte | Meaning |
|---|---|---|---|---|
| 0 | `0x00` | null | `0x00` | mandatory; catches null dereference |
| 1 | `0x08` | code | `0x9A` | ring 0, execute/read, 4 GB |
| 2 | `0x10` | data | `0x92` | ring 0, read/write, 4 GB |

Selectors are `index × 8`, which is why the values are `0x08` and `0x10` rather
than `1` and `2`. The bottom three bits are a requested privilege level and
index; they're zero here.

The access byte `0x9A` is `1001 1010`:

```text
 1   00   1   1   010
 P  DPL  S  exec  code, readable, non-conforming
```

And `0xCF` in the flags byte means granularity = 4 KB, default size = 32-bit.
A limit of `0xFFFFF` with 4 KB granularity spans `0xFFFFF000 + 0xFFF` = 4 GB.

The CPU is told where the table lives with one instruction:

```asm
lgdt gdt_descriptor
```

### Step 4: flipping the switch

Control Register 0 holds the CPU's mode bits. Bit 0 is PE — Protection Enable.

```asm
movl %cr0, %eax
orl $0x01, %eax         # PE = 1
movl %eax, %cr0
```

At the `mov` into `%cr0`, the CPU is in Protected Mode. Sort of. Not really.

### Step 5: the far jump

This is the step people skip in tutorials, and it's the one that actually makes
the transition real.

Setting `CR0.PE` changes how the CPU *interprets* state. It does not update `CS`.
The code segment register still caches the Real Mode segment it was loaded with,
and the instruction prefetch queue was filled under the old decoding rules.

So the very next instruction would still be decoded as 16-bit code, using a
stale segment. You're in a mode the CPU can't actually execute correctly.

The fix is a **far jump** — one that loads a new value into `CS`:

```asm
ljmp $0x08, $pm_start
```

In the built binary that's `ea 38 7c 08 00`: opcode `EA`, offset `0x7C38`,
selector `0x0008`. Because `CS` changes, the CPU has to go fetch descriptor 1
from the GDT. That reload does three things at once:

1. Replaces the cached Real Mode segment with a real protected-mode descriptor.
2. Marks the segment as 32-bit, so subsequent instructions decode as 32-bit.
3. Flushes the prefetch queue, which still held 16-bit-decoded instructions.

Without it you get a machine that claims to be in Protected Mode and executes
garbage.

### Step 6: 32-bit world, direct video memory

Inside `.code32`, everything changes character. No more BIOS:

```asm
movw $0x10, %ax         # data selector
movw %ax, %ds
movw %ax, %es
movw %ax, %fs
movw %ax, %gs
movw %ax, %ss
movl $0x90000, %esp     # 32-bit stack, ~53 KB above the bootloader
```

To print, we write to the VGA text buffer — memory-mapped hardware at `0xB8000`.
The card renders whatever is there. There is no API, no interrupt, no driver.
It's just memory that happens to be a screen.

In 80×25 text mode each cell is two bytes:

```text
byte 0   ASCII character
byte 1   attribute — high nibble background, low nibble foreground
```

`0x0A` is bright green on black. Row 1, column 0 is at `0xB8000 + 160`
(80 columns × 2 bytes × 1 row):

```asm
movl $(0xB8000 + 160), %edi
movl $pm_msg, %esi
movb $0x0A, %ah         # attribute

vga_print:
    lodsb
    testb %al, %al
    jz pm_hang
    movw %ax, (%edi)    # character + attribute, one store
    addl $2, %edi
    jmp vga_print

pm_hang:
    hlt
    jmp pm_hang
```

`movw %ax, (%edi)` writes both bytes in one instruction, which is why `AH` holds
the colour and `AL` holds the character.

## The build

```makefile
ASFLAGS = --32
LDFLAGS = -m elf_i386 -Ttext 0x7C00 --oformat binary

target/boot_sector.bin: bootloader.asm
	as $(ASFLAGS) bootloader.asm -o target/boot.o
	ld $(LDFLAGS) target/boot.o -o target/boot_sector.bin
```

- `-Ttext 0x7C00` — link as if this code will live at `0x7C00`, so labels like
  `pm_start` resolve to `0x7C38`.
- `--oformat binary` — strip the ELF header. The BIOS wants raw bytes, not an
  executable format.

`make run` boots it under QEMU and you get both lines:

```text
Booting iktr OS (16-bit Real Mode)...
iktr OS: 32-bit Protected Mode active.
```

The first comes from `int $0x10`. The second is green, and it comes from
`0xB8000`. That difference is the entire point of the exercise.

## The bit I had to check myself

AI wrote most of this bootloader, and the descriptions I was given for each step
were, as far as I can tell, correct. But there's a class of claim that is
plausible-sounding and unverifiable unless you go look, and one of them sat
right in the middle of the code.

`lgdt gdt_descriptor` is presented everywhere as "load the GDT, which has a
16-bit limit and a 32-bit base." So the descriptor should be six bytes, and the
CPU should read six.

I disassembled it. In `.code16`, the assembler emitted:

```text
7c24:  0f 01 16 80 7c      lgdt [0x7c80]
```

Five bytes. No `0x66` operand-size prefix. With a 16-bit operand, `lgdt` reads a
16-bit limit and a **24-bit** base — five bytes total. The sixth byte of the
descriptor, the top byte of the base address, is never read.

I confirmed it by hand-assembling both forms:

```text
lgdt gdt_descriptor          ->  0f 01 16 ...        (24-bit base)
.byte 0x66 ; lgdt ...        ->  66 0f 01 16 ...     (32-bit base)
```

Does it matter here? No. Our GDT sits at `0x7C68`, and the descriptor's base
field is `68 7c 00 00`. The unread byte is `0x00`, so the loaded base is correct
by luck. If the GDT ever moved above 16 MB — which a linked kernel easily could
— the CPU would load a base with the wrong top byte, and it would fail in a way
that has nothing to do with the GDT contents you can see.

That's the difference between "it boots" and "I know why it boots." The
description was true in the case that happens to be running.

## What I still don't own

The reason this section exists is that AI wrote most of Phase 1, and publishing a
walkthrough I can only parrot would be worse than publishing nothing.

**I can explain every step in order. I can't yet derive them.** Given a blank
file, I would not reconstruct a correct GDT from memory. I know the access byte
`0x9A` is right; I'd have to look up which bit is which to write a new one. The
difference between those two things is the difference between understanding a
topic and having a good index into it.

**"Protected Mode" is currently doing almost no protecting.** The descriptors
say ring 0, there are no ring 3 descriptors, no IDT, no paging, and the flat
model gives every segment the full 4 GB. What I've actually earned is 32-bit
addressing and a working stack. The protection is configured but unused — nothing
in this boot sector would stop ring 0 code from writing anywhere, because we are
ring 0 and there is nothing else.

**Interrupts are off and never come back.** `cli` at the top and again at
`switch_to_pm`, then `hlt` in a loop. With maskable interrupts disabled, `hlt`
never wakes up. The machine is paused permanently, which is fine for a
two-line demo and catastrophic for anything else. Enabling interrupts without a
correct IDT in place is the triple fault described above, so the ordering of
Phase 2 is constrained by this.

**I don't yet know why the stack is at `0x90000`.** It's above the bootloader and
below the Extended BIOS Data Area, which is the usual reasoning, but I chose it
because it's what the references use. The real constraint — that it must not grow
down into `0x7C00` — I only worked out afterwards.

**The A20 port write is the scariest line in the file.** `inb` / `orb` / `outb`
on `0x92` reads a register whose other bits control things like system reset.
Masking only bit 1 is correct, and I verified that, but I could not tell you what
the remaining bits do.

## Run it

```bash
git clone https://github.com/victorchukwuemeka/iktr
cd iktr
make run
```

Requires `binutils`, `make`, and `qemu-system-x86`. The output is
`target/boot_sector.bin`, exactly 512 bytes.

## What's next

Phase 2 is Lab 2.1: a compiled Rust `no_std` kernel entry point, linked so the
bootloader jumps into it instead of hanging at `pm_hang`. That's where the
`iktr` README's real claim — privacy as architecture rather than a feature —
starts being testable, and where my ability to explain the code stops being
optional.
