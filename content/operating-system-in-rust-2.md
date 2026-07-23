+++
title = "Building an Operating System In Rust Part 2"
slug = "building-an-operating-system-in-rust-part-2"
date = 2026-06-19
description = "When running programs on an existing operating system, there are a lot of things that are handled under the hood for us by the operating system. They are invisible to us and we take them for granted, we expect them to just work. In kernel space there are no training wheels"

[extra]
header_img = "/images/header-rust.jpg"
header_alt = "Rust"
section = "Programming"
word_count = 3193
author = "Benjamin Chibuzor-Orie"
tags = ["Rust", "Operating System", "Low Level"]

[taxonomies]
series = ["Building an Operating System in Rust"]
+++

In [Part 1 we built the foundation of our operating system using Rust](operating-system-in-rust.md), dove deep into murky low level waters and broke free from the shackles of stdlib while still keeping the compiler happy. Now we are going into the meat of it all - the entry point.

Feel free to clone the <a href="https://github.com/iambenkay/kluster/tree/part-2" target="_blank" rel="noopener">source code for part 2</a> from Github and follow along.

## Prepare for Entry

When running programs on an operating system, there are a lot of things that are handled under the hood for us by the operating system. They are invisible to us and we take them for granted, we expect them to just work. In kernel space however, there are no training wheels, no invisible cogs turning in the background, everything is laid bare for us to see. Everything that works does so because we made it happen. It may sound overwhelming but it's this kind of foundational control that makes you truly appreciate how computers work at the lowest level.

At the lowest level, the way a CPU executes instructions is very different from the mental model that high level languages try to paint. If you are familiar with assembly then you already know this. Let's look at some core concepts you need to know in order to fully get into the vibe of low level program execution:

### Register

Registers are ultra-fast, small-capacity temporary storage locations built directly into the CPU. They hold the exact data, instructions, or memory addresses the processor is currently working with, acting as the CPU’s immediate "scratchpad" before calculations are sent to the cache or main memory (RAM). There are general purpose registers and registers designed to perform a specific task.

### Execution stack

As instructions execute, the stack is a LIFO data structure used to keep track of the execution context which may include local variables, function parameters or return addresses.

### Stack pointer

The stack pointer is a special CPU register that automatically tracks the top of the execution stack. In Aarch64, the stack grows downward, so logically it tracks the bottom of the stack pointer but the idea remains the same. When the CPU pushes into the stack, the stack pointer gets decremented and the entry is written to that address where the stack is pointing to. When the CPU pops from the stack, it reads the entry at the current stack pointer and then increments the stack pointer effectively freeing the previous memory slot.
{% mermaid() %}
flowchart LR
subgraph Stack [Execution Stack]
Item1["Address: 0x000F"]
Item2["Address: 0x000E"]
Item3["Address: 0x000D"]
Item4["Address: 0x000C"]
end

    SP["Stack Pointer"] --> Item4

    style Stack fill:#ffffff, stroke: #000, stroke-width: 1px, gap: 0
    style Item4 fill:#ff9999
    style SP fill: #000, color: #fff

{% end %}

### Instruction

An instruction is the smallest unit of work that can be performed by a CPU. It takes one or more operands and stores it's result in a register depending on the operation. There are tons of instructions built into a CPU to perform almost any operation you can think of and it varies from CPU to CPU so you have to learn the Instruction Set of the CPU you are working with if you want to write Assembly for it.

### Program Counter

The program counter is a special CPU register that always contains the memory location of the next instruction to execute. As soon as the CPU finishes executing an instruction it checks the program counter and goes on to execute the instruction at the location stored in the program counter. After the CPU reads the program counter, the program counter gets incremented to point to the next instruction and the cycle continues.

## Implementing the boot image

Now that we have a foundational understanding of the building blocks, let's write some assembly code for booting into our kernel. I know I promised we will write as much Rust as possible but this is one of those scenarios where Rust does not cut it. This part of the process cannot be handled by Rust because Rust expects a stack pointer to have already been set up so it is a chicken and egg problem. Remember, the training wheels are off and everything is being done manually so this is one time when we will have to drop down to Assembly in order to configure our entrypoint. This is a delicate point so I want you to pay as much attention as you can afford to.

For your information, any code related to the entrypoint of a kernel is called the boot code. We will be writing the our boot code, or as I prefer to call it "boot image", in a bit but let us first structure our crate.

Let's create a main function for our kernel in a new file `src/kernel.rs`:

```rust
pub fn main() -> ! {
    loop {}
}
```

This function will be called from our entrypoint and will handle all our kernel code but for now it just runs an infinite loop. Load this module in `src/main.rs`:

```rust
mod kernel;
```

Create a file `src/processors/aarch64/bootimage.rs`:

```rust
use core::arch::global_asm;

global_asm!(
    include_str!("boot.S"),
    CONST_CORE_ID_MASK = const 0b11
);

#[unsafe(no_mangle)]
pub fn _start_rust() -> ! {
    crate::kernel::main()
}
```

This is processor specific code. In this case it is expected to run only on aarch64 cpus. We load some assembly from the file `boot.S` substituting a variable `CONST_CORE_ID_MASK` in the assembly with the value `0b11`. Then we define `fn _start_rust()` which will serve as the first point of contact between assembly land and rust land. It calls our kernel main which we defined earlier. Note the `no_mangle` directive. It is required in order for the name of the function to not become scrambled during compilation since we will be calling the function from Assembly.

Then load the module in `src/processors/aarch64/mod.rs`:

```rust
mod bootimage;
```

Add the following content to `src/processors/aarch64/boot.S`:

```as
//--------------------------------------------------------------------------------------------------
// Definitions
//--------------------------------------------------------------------------------------------------

// Load the address of a symbol into a register, PC-relative.
//
// The symbol must lie within +/- 4 GiB of the Program Counter.
.macro ADR_REL register, symbol
	adrp	\register, \symbol
	add	\register, \register, #:lo12:\symbol
.endm

//--------------------------------------------------------------------------------------------------
// Public Code
//--------------------------------------------------------------------------------------------------
.section .text._start

//------------------------------------------------------------------------------
// fn _start()
//------------------------------------------------------------------------------
_start:
	// Only proceed on the boot core. Park it otherwise.
	mrs	x0, MPIDR_EL1
	and	x0, x0, {CONST_CORE_ID_MASK}
	ldr	x1, BOOT_CORE_ID      // provided by __board__/<board_name>/cpu.rs
	cmp	x0, x1
	bne	.L_parking_loop

	// If execution reaches here, it is the boot core.

	// Initialize DRAM.
	ADR_REL	x0, __bss_start
	ADR_REL x1, __bss_end_exclusive

.L_bss_init_loop:
	cmp	x0, x1
	beq	.L_prepare_rust
	stp	xzr, xzr, [x0], #16
	b	.L_bss_init_loop

	// Prepare the jump to Rust code.
.L_prepare_rust:
	// Set the stack pointer.
	ADR_REL	x0, __boot_core_stack_end_exclusive
	mov	sp, x0

	// Jump to Rust code.
	b	_start_rust

	// Infinitely wait for events (aka "park the core").
.L_parking_loop:
	wfe
	b	.L_parking_loop

.size	_start, . - _start
.type	_start, function
.global	_start
```

Whoa! That's a huge chunk of foreign code. Lucky you, I can demystify what's going on there. Let's take it apart line by line.

### Assembly Breakdown

#### The ADR_REL macro

```as
.macro ADR_REL register, symbol
	adrp	\register, \symbol
	add	\register, \register, #:lo12:\symbol
.endm
```

This is a two-instruction helper that loads the address of a symbol into a register in a PC-relative way (relative to the program counter, no absolute addressing, so the code works regardless of where it's loaded as long as the symbol is within ±4 GiB).

1. **adrp** ("Address of Page") computes the 4 KiB-aligned page address containing the symbol, relative to the PC. So it gets you within 4 KiB of the symbol.
2. **add** … `#:lo12:symbol` adds the low 12 bits of the symbol's address (the offset within the page) to finish the address.
   Together they materialize an arbitrary symbol address with two instructions and no memory load. This matters because the BSS hasn't been zeroed yet and the stack isn't set; we can't yet rely on memory loads of address tables.

Let's use a practical example, Assume the Program Counter is at `0x00BC3000`. We want to get the PC relative address of a symbol `msg` which for the sake of this example is stored at `0x00BC5B70`. The question that this macro answers is how to get that symbol's address into a register. First we get the page address of the symbol relative to the program counter. The way this works is by finding which 4KiB slice (called a page) the symbol's address falls into:
{% mermaid() %}
flowchart LR
subgraph Memory
Page1["Page 1: 0x00BC3000 to 0x00BC3FFF"]
Page2["Page 2: 0x00BC4000 to 0x00BC4FFF"]
Page3["Page 3: 0x00BC5000 to 0x00BC5FFF"]
Page4["Page 4: 0x00BC6000 to 0x00BC6FFF"]
Page5["Page 5: 0x00BC7000 to 0x00BC7FFF"]
end

Msg["msg"]-- in this page --> Page3
{% end %}
So we know that the symbol is 3 pages away from the program counter. Next we get the lower 12 bits of the symbol's address using `#:lo12:\symbol`. Under the hood, this operation does this:

```rust
0x00BC5B70 & 0xFFF = 0xB70 // retains only the lower 12 bits
```

Then we add the result to the page address to get the symbol address:

```rust
0x00BC5000 + 0xB70 = 0x00BC5B70 // returned by ADR_REL macro
```

#### Code section

Assembly code is organized into sections. Each section contains a sequence of instructions and directives that define a logical unit of our program. The following labels a new section named `.text._start` which is the start of our code for this module:

```as
//--------------------------------------------------------------------------------------------------
// Public Code
//--------------------------------------------------------------------------------------------------
.section .text._start
```

#### Boot into primary CPU core

The next piece of code is responsible for selecting one of the available cores as our primary core and parking the remaining cores. By default all the CPU cores will pick up the kernel and try to run it so this is a crucial step to disqualify all but one, our primary core.

```as
_start:
    mrs   x0, MPIDR_EL1
    and   x0, x0, {CONST_CORE_ID_MASK}
    ldr   x1, BOOT_CORE_ID
    cmp   x0, x1
    bne   .L_parking_loop
```

It starts by reading the value of the Multiprocessor Affinity Register, `MPIDR_EL1` into register `x0`. The system register `MPIDR_EL1` identifies which CPU we're on, specifically the low bits which contain the core number. Assuming we are on raspberry pi 4 or 5 then there are only 4 CPUs available which can be represented with two bits. So the value of `MPIDR_EL1` may look like this on the different cores:

```
Core 1 - 01110000
Core 2 - 01110001
Core 3 - 01110010
Core 4 - 01110011
```

so we need a mask that can select only the last two bits of this register value and that mask is `0b11` because we can take any of the values and `AND` them with this mask to retrieve the last two bits:

```rust
0b01110000 & 0b11 = 0b00
0b01110001 & 0b11 = 0b01
0b01110010 & 0b11 = 0b10
0b01110011 & 0b11 = 0b11
```

We defined the mask `CONST_CORE_ID_MASK` a while ago in `src/processors/aarch64/bootimage.rs` and this is where it gets used. Depending on the board we are running on we can change this mask to `0b111` to support up to 7 cores or `0b1111` to support up to 15 cores.

Next, we load the value of the variable `BOOT_CORE_ID` (which we will define soon in Rust) and store that in the register `x1`. At this point `x0` contains the CPU number which the code is currently executing on and `x1` contains the target CPU we want to take control while parking the rest. We compare their values and branch to the parking logic if `x0` is not equal to `x1`, else we continue execution.

Before we continue describing our startup logic, let's define the `BOOT_CORE_ID` variable that we used above. Create a file `src/motherboards/raspberrypi/cpu.rs`:

```rust
#[cfg(target_arch = "aarch64")]
#[unsafe(no_mangle)]
#[unsafe(link_section = ".text._start_arguments")]
pub static BOOT_CORE_ID: u64 = 0;
```

`link_section = ".text._start_arguments"` adds a new section labelled `.text._start_arguments` and the variable `BOOT_CORE_ID` in that section. We are keeping the value as `0` because we want to boot into the first core as our primary core.

Create another file `src/motherboards/raspberrypi/mod.rs`:

```rust
mod cpu;
```

Then another file `src/motherboards/mod.rs`:

```rust
pub mod raspberrypi;
```

and then add the module to `src/main.rs`:

```rust
mod motherboards;
```

#### Initialize BSS

The BSS is a data segment that stores uninitialized global and static variables. At startup we need to zero all the addresses within the BSS in preparation for uninitialized variables that will be stored in this segment. The following segment handles this:

```as
	// If execution reaches here, it is the boot core.

	// Initialize DRAM.
	ADR_REL	x0, __bss_start
	ADR_REL x1, __bss_end_exclusive

.L_bss_init_loop:
	cmp	x0, x1
	beq	.L_prepare_rust
	stp	xzr, xzr, [x0], #16
	b	.L_bss_init_loop
```

This is a simple routine that loads the start and end of the BSS into `x0` and `x1` using the symbols `__bss_start` and `__bss_end_exclusive` (will be defined later in the linker script). Then we recursively store zeroes in the memory locations stored in `x0` and increment the memory location stored in `x0` by `16`. This continues until the memory location in `x0` matches the end of bss stored in `x1` then we branch to `.L_prepare_rust`.

#### Jumping into Rust land

The next piece of code is the bridge into our Rust code:

```as
// Prepare the jump to Rust code.
.L_prepare_rust:
	// Set the stack pointer.
	ADR_REL	x0, __boot_core_stack_end_exclusive
	mov	sp, x0

	// Jump to Rust code.
	b	_start_rust
```

First we store the address of the end of the stack (configured later in the linker script) in `x0`. Then we set the value of the stack pointer register to that address. Finally we branch to the label for our Rust entry point (defined in `src/processors/aarch64/bootimage.rs`).

The next piece of code just defines the label for our parking loop so the other cores know what to do after being disqualified from running the kernel:

```as
// Infinitely wait for events (aka "park the core").
.L_parking_loop:
	wfe
	b	.L_parking_loop
```

Lastly, we define some metadata for this assembly module at the very end:

```as
.size	_start, . - _start
.type	_start, function
.global	_start
```

The first directive `.size` computes the size of the routine `_start`. The second directive `.type` specifies the type of `_start` which is a function. The last directive `.global` makes the `_start` label a global entity which can be referenced by other libraries.

Summarily, here is the boot flow:
{% mermaid() %}
flowchart TD
A["Firmware loads kernel"] --> B["_start<br/>(boot.S, .text._start)"]
B --> C{"core == 0?"}
C -- "no" --> D["wfe loop forever"]
C -- "yes" --> E["Zero BSS<br/>(__bss_start .. __bss_end_exclusive)"]
E --> F["sp = __boot_core_stack_end_exclusive"]
F --> G["b _start_rust"]
G --> H["_start_rust<br/>(bootimage.rs)"]
H --> I["kernel::main()"]
{% end %}

We have implemented our boot image in pure Assembly. I hope you enjoyed that! One last important step before we can run our kernel for the first time is Linking.

### Writing the Linker Script

Linking is the glue for everything we have done. A linker is responsible for taking both the assembly code and the Rust code and linking them into one binary but we have to explicitly tell the linker how we want our code to be organized; we have to define the memory layout for our binary!

Create the file `src/motherboards/raspberrypi/kernel.ld` and add the following content:

```ld
__rpi_phys_dram_start_addr = 0;

/* The physical address at which the the kernel binary will be loaded by the Raspberry's firmware */
__rpi_phys_binary_load_addr = 0x80000;


ENTRY(__rpi_phys_binary_load_addr)

/* Flags:
 *     4 == R
 *     5 == RX
 *     6 == RW
 *
 * Segments are marked PT_LOAD below so that the ELF file provides virtual and physical addresses.
 * It doesn't mean all of them need actually be loaded.
 */
PHDRS
{
    segment_boot_core_stack PT_LOAD FLAGS(6);
    segment_code            PT_LOAD FLAGS(5);
    segment_data            PT_LOAD FLAGS(6);
}

SECTIONS
{
    . =  __rpi_phys_dram_start_addr;

    /***********************************************************************************************
    * Boot Core Stack
    ***********************************************************************************************/
    .boot_core_stack (NOLOAD) :
    {
                                             /*   ^             */
                                             /*   | stack       */
        . += __rpi_phys_binary_load_addr;    /*   | growth      */
                                             /*   | direction   */
        __boot_core_stack_end_exclusive = .; /*   |             */
    } :segment_boot_core_stack

    /***********************************************************************************************
    * Code + RO Data + Global Offset Table
    ***********************************************************************************************/
    .text :
    {
        KEEP(*(.text._start))
        *(.text._start_arguments) /* Constants (or statics in Rust speak) read by _start(). */
        *(.text._start_rust)      /* The Rust entry point */
        *(.text*)                 /* Everything else */
    } :segment_code

    .rodata : ALIGN(8) { *(.rodata*) } :segment_code

    /***********************************************************************************************
    * Data + BSS
    ***********************************************************************************************/
    .data : { *(.data*) } :segment_data

    /* Section is zeroed in pairs of u64. Align start and end to 16 bytes */
    .bss (NOLOAD) : ALIGN(16)
    {
        __bss_start = .;
        *(.bss*);
        . = ALIGN(16);
        __bss_end_exclusive = .;
    } :segment_data

    /***********************************************************************************************
    * Misc
    ***********************************************************************************************/
    .got : { *(.got*) }
    ASSERT(SIZEOF(.got) == 0, "Relocation support not expected")

    /DISCARD/ : { *(.comment*) }
}
```

The above linker script is a lot to digest but let's take it step by step and see what makes it tick.

First, we are declaring some link time variables that will be used by the rest of the script:

```ld
__rpi_phys_dram_start_addr = 0;

/* The physical address at which the the kernel binary will be loaded by the Raspberry's firmware */
__rpi_phys_binary_load_addr = 0x80000;
```

In Raspberry Pi Aarch64, the kernel image will be loaded into the address `0x80000`. This is standard for Raspberry Pi running in 64 bit mode and will be different for different motherboards so we store the value here for use later in the script.

Next line is:

```ld
ENTRY(__rpi_phys_binary_load_addr)
```

It sets the ELF header's entry-point field to `0x80000`. Some loaders use that field to decide where to jump; the Pi firmware ignores it and just jumps to `0x80000` regardless. Still good practice to set it right.

Then we have the program-header segments:

```ld
PHDRS
{
    segment_boot_core_stack PT_LOAD FLAGS(6);
    segment_code            PT_LOAD FLAGS(5);
    segment_data            PT_LOAD FLAGS(6);
}
```

ELF binaries describe memory in two layers: small sections for the linker (.text, .data, …) and bigger segments that an actual loader maps. PHDRS declares those segments and their permissions.

PT*LOAD = "an ELF loader should make this segment exist in memory."
FLAGS(...) bits: 4 = R, 5 = R+X, 6 = R+W.
So you end up with three logical regions: the stack (RW), the code+rodata (RX), and the mutable data + BSS (RW). Each section below is assigned to one of these via the trailing :segment*… syntax.

The next part is the meat of the script; sections. It walks down memory address-by-address and places things. Describes start and endpoints for the different sections in our binary:

```ld
SECTIONS
{
    . =  __rpi_phys_dram_start_addr;

    /***********************************************************************************************
    * Boot Core Stack
    ***********************************************************************************************/
    .boot_core_stack (NOLOAD) :
    {
                                             /*   ^             */
                                             /*   | stack       */
        . += __rpi_phys_binary_load_addr;    /*   | growth      */
                                             /*   | direction   */
        __boot_core_stack_end_exclusive = .; /*   |             */
    } :segment_boot_core_stack

    /***********************************************************************************************
    * Code + RO Data + Global Offset Table
    ***********************************************************************************************/
    .text :
    {
        KEEP(*(.text._start))
        *(.text._start_arguments) /* Constants (or statics in Rust speak) read by _start(). */
        *(.text._start_rust)      /* The Rust entry point */
        *(.text*)                 /* Everything else */
    } :segment_code

    .rodata : ALIGN(8) { *(.rodata*) } :segment_code

    /***********************************************************************************************
    * Data + BSS
    ***********************************************************************************************/
    .data : { *(.data*) } :segment_data

    /* Section is zeroed in pairs of u64. Align start and end to 16 bytes */
    .bss (NOLOAD) : ALIGN(16)
    {
        __bss_start = .;
        *(.bss*);
        . = ALIGN(16);
        __bss_end_exclusive = .;
    } :segment_data

    /***********************************************************************************************
    * Misc
    ***********************************************************************************************/
    .got : { *(.got*) }
    ASSERT(SIZEOF(.got) == 0, "Relocation support not expected")

    /DISCARD/ : { *(.comment*) }
}
```

The summary of what is going on is it defines the bounds of the Execution Stack, the actual code segment, and the data and BSS segment. I don't want to go too deep into the details of the linker script because it is vanity to master it all but feel free to discuss the details with any LLM in order to fully understand what is going on here.

We are done with our boot image and at this stage we can boot our kernel on either a physical device or the QEMU emulator.
Before we build, add the following to your Cargo.toml so that our Make script can run happily:

```toml
[features]
default = ["device", "rpi5"]
emulator = ["ltr-rgb"]
device = []
rpi5 = []
rpi4 = []
ltr-rgb = []
```

We aren't using some of the features listed here yet but in due time we will cover them.

Let's build and run the kernel on QEMU using our emulator:

```bash
cargo make emulate
```

You should see a black QEMU screen pop up with no errors just like this one:
{{ image(src="/images/os-part2-result.png", alt="operating system on qemu part2 result") }}

In order to run on a physical Raspberry Pi 5 device there is also a convenient cargo make task:

```bash
cargo make sync
```

It attempts to copy the binary into a memory stick containing an existing Raspbian installation. Alter the task in `Makefile.toml` so it gets copied into the right path.

## Conclusion

In this part, we implemented a boot image and successfully booted our kernel on a raspberry pi environment but we still have a blank screen. In [Part 3 we will be writing to the display](operating-system-in-rust-3.md); in QEMU that's the screen but on Raspberry Pi 5 that is the HDMI output.
