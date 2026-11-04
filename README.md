# asm-arcade

A bare-metal retro arcade machine in pure x86-64 assembly (Intel syntax, NASM). Boots via [Limine](https://github.com/limine-bootloader/limine) straight into 64-bit long mode with a linear framebuffer, then drops into a game-selection menu. No OS, no libc, no syscalls. The kernel owns the CPU, framebuffer, interrupts, input, and timing.

## Games

| # | Cartridge | Status | Techniques |
|---|-----------|--------|------------|
| 1 | Snake | [ ] | Tile grid, ring-buffer body, xorshift PRNG, fixed-timestep tick |
| 2 | Breakout | [ ] | 16.16 fixed-point physics, AABB collision, brick bitmap |
| - | Menu | [ ] | Bitmap font renderer, cartridge table, selection state |

Later candidates: falling sand (SSE2/AVX2 inner loop), scrolling platformer.

## Target

- **Firmware:** UEFI (primary, QEMU + OVMF), BIOS as optional fallback
- **Bootloader:** Limine (pin the release branch; config syntax and request structs differ across major versions)
- **CPU mode:** x86-64 long mode, entered by Limine, higher-half kernel, HHDM for physical access
- **Video:** Limine framebuffer request (honor `pitch`, `bpp`, and channel masks; never assume 32bpp XRGB)
- **Input:** PS/2 keyboard via IRQ1 (QEMU first; real-hardware USB needs an xHCI/HID stack and is out of scope)
- **Assembler/linker:** NASM `-f elf64`, GNU `ld` with a linker script
- **Debug:** COM1 serial (`0x3F8`) to `-serial stdio`, QEMU `-s -S` + gdb

## Dependencies (CachyOS / Arch)

```sh
sudo pacman -S --needed nasm binutils make gdb xorriso mtools qemu-full edk2-ovmf limine
```

## Build and run

```sh
make            # kernel ELF + ISO in build/
make run        # boot ISO in QEMU + OVMF, serial on stdio
make debug      # QEMU paused with gdbstub on :1234
make clean
```

Reference QEMU invocation:

```sh
qemu-system-x86_64 -M q35 -m 256M \
	-drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/x64/OVMF_CODE.4m.fd \
	-cdrom build/arcade.iso -serial stdio -no-reboot -no-shutdown -d int,cpu_reset -D build/qemu.log
```

(Adjust the OVMF path to what your `edk2-ovmf` package installs.)

## Repo layout

```
.
├── Makefile
├── linker.ld
├── limine.conf              # name/syntax depends on pinned Limine version
├── README.md
├── src/
│   ├── boot.asm             # _start, Limine requests, stack setup
│   ├── serial.asm           # COM1 init + putc/puts/hex dump
│   ├── gdt.asm              # GDT + TSS, IST stack for #DF
│   ├── idt.asm              # IDT, exception stubs, IRQ stubs
│   ├── pic.asm              # PIC remap + EOI (or apic.asm later)
│   ├── kbd.asm              # PS/2 scancode set 1 decoder, key event queue
│   ├── timer.asm            # PIT/TSC calibration, tick counter
│   ├── gfx.asm              # back buffer, blit, rect, pixel format handling
│   ├── font.asm             # 8x16 bitmap font data + draw_char/draw_str
│   ├── util.asm             # itoa, memcpy/memset (rep movs/stos), PRNG
│   ├── menu.asm             # cartridge selection loop
│   └── games/
│       ├── snake.asm
│       └── breakout.asm
└── build/                   # gitignored
```

## Cartridge ABI

Each game exports a descriptor; the menu holds a table of pointers to them.

```nasm
struc cart
	.name:		resq 1	; ptr to zero-terminated string
	.init:		resq 1	; void init(void)
	.update:	resq 1	; void update(void)            fixed-timestep logic
	.render:	resq 1	; void render(void)            draws into back buffer
	.on_key:	resq 1	; void on_key(u8 scancode, u8 released)
	.exit:		resq 1	; void exit(void)
endstruc
```

- Calling convention: SysV AMD64 (args in `rdi, rsi, ...`; callee-saved `rbx, rbp, r12-r15`; 16-byte stack alignment at `call`).
- Cartridges draw only through `gfx.asm` into the back buffer. The main loop owns blit and timing.
- A cartridge returns to the menu by setting a global `want_exit` flag, never by `jmp`ing out.

## Core loop

1. Wait for tick (`hlt` until timer IRQ advances the counter).
2. Drain the key event queue into `cart.on_key`.
3. Run `cart.update` once per fixed step (catch up if behind, cap iterations).
4. Run `cart.render` into the back buffer.
5. Blit the back buffer to the framebuffer (`rep movsq`, row-wise if `pitch != width*bpp/8`).

## Design notes

- **Interrupts:** every IRQ handler must send EOI and preserve all registers. `#DF` runs on its own IST stack so a bad stack does not triple-fault.
- **Memory:** static `.bss` arenas only (back buffer sized from the framebuffer response, or reserved for the max resolution). No allocator until something forces one.
- **Rendering:** never draw directly to the framebuffer. Consider PAT write-combining once everything works.
- **Timing:** calibrate the TSC against PIT channel 2 once at boot, then drive the fixed timestep from the TSC or LAPIC timer.
- **Input:** scancode set 1 make/break codes; handle `0xE0` extended prefixes for arrow keys.

## Conventions

- Intel syntax, NASM `default rel`, tabs for indentation.
- Local labels with `.dot` prefix; one `global` entry point per cartridge.
- Header comment per routine: inputs, outputs, clobbers.
- Align hot loops to 16 B; measure with `rdtsc` before and after any micro-optimization.

## Milestones

- [ ] Boot in QEMU + OVMF from an `xorriso` ISO, print over COM1
- [ ] Parse the Limine framebuffer request, fill the screen with a color
- [ ] Own GDT/IDT with exception handlers (`#DF` on IST)
- [ ] Back buffer, blit, bitmap font
- [ ] PS/2 keyboard IRQ
- [ ] Timer and fixed-timestep loop
- [ ] Cartridge ABI, menu, Snake
- [ ] Breakout
- [ ] Optional: PC speaker via PIT channel 2, real-hardware boot from USB stick

