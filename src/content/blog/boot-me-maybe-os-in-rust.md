---
title: "BootMeMaybe: An Operating System in Rust"
date: 2026-02-09
description: "Building a bare-metal OS in Rust: Disabling std and implementation details."
---

When starting an OS project, the first hurdle is breaking free from the existing operating system. By default, Rust programs link against `std`, which relies on an operating system for threads, files, networking, io streams, and memory allocation. But when you *are* making the operating system, you can't rely on these features.

This post walks through the initial setup of `boot-me-maybe-os` as a freestanding Rust binary.

## Part 1: Disabling the Standard Library

To run on bare metal, we need to disable the standard library. This is done by adding the `#![no_std]` attribute:

```rust
// Removing the use of std rust library for creating a freestanding rust binary
// This binary doesn't use any of the operating system functionalities
#![no_std]
```

## The Panic Handler

The standard library provides a panic handler that prints a message to standard output. Without `std`, we must define our own. For `boot-me-maybe-os`, we want a minimal panic handler that simply loops forever:

```rust
use core::panic::PanicInfo;

// Creating a panic handler, since we are not using the std library
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

Since we don't have an OS to return to, the panic handler is a diverging function (returns `!`), meaning it never returns.

## Stack Unwinding

By default, when a Rust program panics, it initiates a process called **stack unwinding**. This process walks back up the stack, cleaning up data and running destructors for every object in scope. This ensures that resources (like open files or memory allocations) are properly released even during a crash.

However, stack unwinding is complex and requires significant runtime support (often provided by `libunwind` or similar libraries). In this initial phase, let's omit stack unwinding to keep the kernel footprint minimal. Feel free to implement this yourself, for the first iteration we will be focusing on the core functionality first.

For our OS kernel, if a panic occurs, there is no "higher authority" to catch it or clean up after it—the system is effectively dead. Therefore, the simplest and most robust approach is to **abort** immediately.

We can disable unwinding by setting `panic = "abort"` in our `Cargo.toml`. This tells the compiler to simply generate a "trap" instruction (like a breakpoint or infinite loop) on panic, drastically reducing the binary size and removing the need for unwinding infrastructure.

```toml
[profile.dev]
panic = "abort" # disable stack unwinding on panic

[profile.release]
panic = "abort" # disable stack unwinding on panic
```

## The Entry Point

In a standard Rust application, `fn main()` isn't actually the first thing the computer runs. There is a hidden "preparation" phase handled by a piece of software called `crt0` (C Runtime Zero).

When you move to a freestanding environment (like writing an OS kernel or firmware for a bare-metal microcontroller), you are essentially telling the compiler: "I'm taking over. Don't assume there is an operating system, a filesystem, or a pre-configured environment."

### 1. The Standard Flow (Hosted Environment)
In a typical Windows, macOS, or Linux environment, the execution flow looks like this:

1. **The OS Loader**: Loads your compiled binary into memory.
2. **`crt0` (The C Runtime)**: This is the "pre-game" setup. It:
    - Initializes the stack.
    - Places arguments (`argc`, `argv`) into the correct registers.
        - `argc` (Argument Count): An integer representing the number of arguments passed (e.g., 3).
        - `argv` (Argument Vector): A pointer to an array of strings (the actual arguments, like ["./my_program", "arg1", "arg2"]).
    - Sets up the C Library (e.g., glibc).
3. **Rust Runtime Entry**: The Rust standard library has its own entry point that handles panic hooks and stack overflow guards.
4. **`fn main()`**: Your code finally runs.

### 2. The Freestanding Flow (Bare Metal)
In a freestanding binary, you use `#![no_std]` and `#![no_main]`. Since there is no `crt0`, the hardware (CPU/BIOS/UEFI) jumps directly to a specific memory address where it expects your code to be.

#### Key Missing Components
Because you lack the C runtime, you lose several "comforts" we usually take for granted:

| Feature | Standard Rust | Freestanding Rust |
| :--- | :--- | :--- |
| **Memory** | `Box`, `Vec`, `String` (Dynamic allocation) | Only Stack (unless you write an Allocator) |
| **Output** | `println!` | Must write to serial ports or VGA buffer |
| **Threads** | `std::thread` | Must implement a scheduler manually |
| **Panic** | Unwinds stack and prints error | You must define a `#[panic_handler]` |

### 3. Why we use `_start`
Since the linker (the tool that stitches your code together) can no longer find the standard entry point, we have to define our own. By convention, this is named `_start`.

```rust
#![no_main]

// ...

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    // The CPU starts here.
    // We must initialize everything ourselves.
    loop {}
}
```

- `#[no_mangle]`: Tells the Rust compiler not to change the name of the function to something like `_ZN3app6_startE...`. The hardware/linker needs to see the exact name `_start`.
- `extern "C"`: Tells the compiler to use the C calling convention. This ensures that the way parameters are passed (via registers vs. stack) matches what the hardware expects.
- `-> !`: This is the "Never" type. Since there is no OS to return to, this function must never exit. If it did, the CPU would simply keep executing whatever random data happens to be next in memory.

### What happens if you use `std`?
If you try to compile a standard program for a freestanding target (like an ARM Cortex-M or x86_64 bare-metal), the compiler will throw errors because it cannot find the system libraries for things like file I/O or threading.

## Target Configuration

When you build a binary, you are not just compiling “Rust code.”
You are producing a machine-specific executable artifact that must match:
- A specific CPU architecture
- A specific operating system
- A specific ABI (Application Binary Interface)
- A specific C runtime / linking environment

Normally, when you run cargo build, Rust looks at your computer (e.g., `Windows x86_64`) and builds for that. For an OS project, you are cross-compiling—writing code on your PC to run on a completely different CPU architecture.
Since we are cross-compiling: writing code on your PC to run on a completely different architecture with zero OS support.
To achieve this, we define a Freestanding Target in .cargo/config.toml:

```toml title="config.toml"
[build]
target = "thumbv7em-none-eabihf"
```

We will discuss this in depth later.

### System Startup Flow
```mermaid
%%{init: {'flowchart': {'htmlLabels': true}, 'theme': 'dark'}}%%
flowchart TD
    A["🔌 Power On / CPU Reset"] --> B["💾 CPU Jumps to<br/>Reset Vector"]

    subgraph FW["🧠 Firmware (BIOS / UEFI)"]
        C["Start Firmware Execution"]
        D["🧪 Perform Power-On Self-Test"]
        E{"✓ Power-On Self-Test Passed?"}
        F["⛔ Halt System"]
        G["🔍 Initialize Hardware<br/>Find Boot Device"]
    end

    B --> C
    C --> D
    D --> E
    E -->|❌ No| F
    E -->|✓ Yes| G
    G --> H["🚀 Bootloader: Load & Start OS Kernel"]

    style FW fill:none,stroke:#ffffff,stroke-width:3px,stroke-dasharray: 8 6
    style A fill:#444,stroke:#222,stroke-width:3px,color:#fff,font-weight:bold
    style B fill:#1a472a,stroke:#0f2b1a,stroke-width:3px,color:#fff,font-weight:bold
    style C fill:#2d5aa6,stroke:#1e3f7a,stroke-width:3px,color:#fff,font-weight:bold
    style D fill:#06a77d,stroke:#047857,stroke-width:3px,color:#fff,font-weight:bold
    style E fill:#ff9f1c,stroke:#cc6d00,stroke-width:3px,color:#000,font-weight:bold
    style F fill:#e63946,stroke:#b81d22,stroke-width:3px,color:#fff,font-weight:bold
    style G fill:#06a77d,stroke:#047857,stroke-width:3px,color:#fff,font-weight:bold
    style H fill:#06a77d,stroke:#047857,stroke-width:3px,color:#fff,font-weight:bold
```

When you turn on a computer, the following steps happen in the following order:
1. **CPU resets:** All registers are set to predefined values.
2. **Read Firmware (BIOS/UEFI):** After reset, Firmware runs from motherboard ROM, the CPU immediately begins executing instructions from a fixed physical address called the reset vector.
3. Firmware Execution: It starts executing immediately.
    - One of the first things firmware does is run **Power-On Self-Test (POST)**. In which it checks:
        - CPU basic functionality
        - RAM presence
        - GPU
        - Keyboard controller
        - Essential peripherals
4. If POST succeeds → firmware continues boot process.
5. Firmware loads bootloader / OS.

## Firmware
There are two firmware standards: 
1. **“Basic Input/Output System“ (BIOS)** The BIOS standard is old and outdated, but simple and well-supported on any x86 machine since the 1980s.
2. **“Unified Extensible Firmware Interface” (UEFI)** UEFI, in contrast, is more modern and has much more features, but is more complex to set up.

## Bootloader:
When BIOS finds a bootable device, it:
- Reads the first 512 bytes from the disk. Those 512 bytes are called **Master Boot Record (MBR)**

    **Why Only 512 Bytes?**

    Because the BIOS boot protocol is ancient.
- Loads it to memory at address `0x7C00`
- Jumps to it

Because the BIOS boot protocol is very old, it only loads these 512 bytes initially. Since this space is too small for a full bootloader, most bootloaders use multiple stages: a tiny first stage in the MBR that loads a larger second stage.

The bootloader’s responsibilities are:
- Locate the OS kernel image on disk and load it into memory.
- Switch the CPU from 16-bit real mode → 32-bit protected mode → 64-bit long mode, enabling full memory access and 64-bit registers.

    **Real Mode: (16-bit):**
    This is the legacy state the CPU enters immediately after power-on. It mimics the original Intel 8086 processor.

    - **Registers:** Limited to 16 bit registers.
    - **Memory Access:** You can only access about 1 MB of RAM.
    - **Security:** There is no memory protection. Any code can write to any part of memory, including the BIOS or other programs.

    > [!NOTE] 
    > **1MB of RAM How?**
    > 
    > The original 8086 CPU had:
    >  - 16-bit registers
    >  - But a 20-bit address bus
    >       - An address bus is the set of physical wires the CPU uses to denote a memory address to RAM.
    >
    > Why?
    > Because Intel wanted:
    > - Cheap 16-bit internal design
    > - But ability to address more than 64KB
    >
    >  A 16-bit register alone can only address:
    > - `2^16 = 65,536 bytes = 64KB`
    > - But Intel wanted:
    > - `2^20 = 1,048,576 bytes = 1MB`
    >
    > The Trick: Segmentation
    > Instead of increasing register size, they introduced:
    > - 16-bit segment register
    > - 4-bit offset register
    > 
    > And combined 2 registers (16-bit + 4-bit) to produce a 20-bit physical address.

    **Protected Mode: (32-bit):**
    In this mode, programs are isolated. The CPU enforces boundaries.
    - **Registers:** Limited to 32 bit registers.
    - **Memory Access:** You can only access about 4 GB of RAM.
    - **Security:** Introduces different hardware-enforced access levels.
    **Ring 0:** Kernel run here, has full control
    **Ring 3:** User applications run here, restricted access, can't access kernel memory, can't run privileged instructions.

    If somebody tries to write in kernel:
    - CPU immediately stops execution
    - Raises a **General Protection Fault (GPF)**
    - OS kills the process

    **Long Mode (64-bit):**  
    This is the modern 64-bit operating mode used by today’s operating systems.  
    It provides a massive virtual address space, mandatory paging, and stronger memory control.

    - **Registers:** Expanded to 64-bit registers (`RAX`, `RBX`, etc.), plus additional registers (`R8–R15`).  
    This allows direct 64-bit arithmetic and much larger integer values.

    - **Memory Access:** Theoretical limit is extremely large (2⁶⁴ bytes).  
    Practically, current CPUs use 48-bit virtual addressing (≈256 TB of virtual memory).  
    Paging is mandatory — all memory access goes through page tables.

    - **Security:** Same differen hardware-enforced access levels as protected mode.


## Conclusion

With these pieces in place—`no_std`, a custom panic handler, strictly defined entry point, and the correct target—`boot-me-maybe-os` can now be built as a standalone binary ready for bare-metal execution. The next steps will involve interacting with hardware and getting something to appear on the screen!
