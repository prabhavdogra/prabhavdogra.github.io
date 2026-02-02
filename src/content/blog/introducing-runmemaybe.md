---
title: "Introducing RunMeMaybe: A Go-based Stack VM"
date: 2026-01-05
description: "A deep dive into building a custom stack-based virtual machine and assembly language in Go, exploring its architecture, instruction set, and tooling ecosystem."
---

Have you ever wondered how your code *actually* runs? We type high-level syntax, hit "run," and magic happens. To understand that magic better, I decided to build my own virtual machine (VM) and assembly language from scratch in Go.

Meet **RunMeMaybe** (rmm)—a lightweight, stack-based VM that features a custom assembly language, a preprocessor with macro support, and a growing ecosystem of tools.

> [!NOTE]
> This project is heavily inspired by [tim](https://github.com/CobbCoding1/tim) by CobbCoding.

## Why Build a VM?

Building a VM is one of the best ways to demystify the internal workings of computers and compilers. It forces you to think about:
*   **Memory Management**: How data is stored, retrieved, and managed (stack vs. heap).
*   **Instruction Parsing**: How raw text or bytecode is translated into actionable operations.
*   **Execution Flow**: How jumps, calls, and returns control the path of execution.

RunMeMaybe started as a curiosity project and evolved into a fully functional environment where I could execute complex logic using a simple, custom instruction set.

## Architecture: Stack vs. Register

RunMeMaybe is a **stack-based VM**, similar to the Java Virtual Machine (JVM) or Python's bytecode interpreter.

In a register-based architecture (like x86 or ARM), operations typically occur between explicitly named registers (e.g., `ADD R1, R2`). In a stack-based VM like RunMeMaybe, operands are pushed onto a central stack, and operations consume values from the top.

For example, to add two numbers:
1.  Push the first number.
2.  Push the second number.
3.  Execute `add` (pops both, adds them, and pushes the result).

This simplifies the instruction set and parsing logic, making it an excellent model for learning and experimentation.

## Key Features

### 1. Custom Assembly Language (`.rmm`)
The VM executes `.rmm` files, which natively support integers, floats, and characters. The syntax is designed to be clean and readable.

```rmm
push 10
push 20
add
print
```

### 2. Powerful Preprocessor
I wanted the language to feel usable, so I included a preprocessor that handles:
*   **Imports**: `@imp "stdlib.rmm"` lets you modularize your code.
*   **Macros**: `@def MAX_VALUE 100` allows for constant definitions and simple text substitution.

### 3. Native Syscalls
To interact with the outside world, the VM provides native syscalls for file I/O, memory management, and strings. You can `open` files, `write` output, or even access `malloc` and `free` for heap operations, bridging the gap between the virtual environment and the host OS.

### 4. Rich Instruction Set
The [Instruction Set](https://github.com/prabhavdogra/vm?tab=readme-ov-file#instruction-set) supports:
*   **Arithmetic**: `add`, `sub`, `mul`, `div`, `mod`
*   **Stack Manipulation**: `dup`, `swap`, `rot` (via `inswap`/`indup`)
*   **Control Flow**: `jmp`, `call`, `ret`, and conditional jumps (`zjmp`, `nzjmp`)
*   **Comparisons**: `cmpe` (equal), `cmpg` (greater), etc.

## The Tooling Ecosystem

A language is only as good as its tools. To improve the developer experience, I created a **VSCode Extension** that provides syntax highlighting for `.rmm` files.

Additionally, the VM includes a **Debug Mode**. Running `go run . file.rmm --debug` prints the token stream, parsed instructions, and the final state of the stack, which is invaluable for troubleshooting your assembly code.

## Quick Start

Want to try it out? You'll need Go 1.25+.

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/prabhavdogra/vm
    cd vm
    ```

2.  **Run a program:**
    Create a file named `hello.rmm`:
    ```rmm
    push_str "Hello, World!"
    native 1  ; syscall for write (stdout)
    ```

    Run it:
    ```bash
    go run . hello.rmm
    ```

## Conclusion

Building RunMeMaybe has been a fantastic journey into the low-level details of system design. It’s open source, and I’m constantly adding new features like better string handling and pointer operations.

Check out the code on GitHub and feel free to try it out!

[View on GitHub](https://github.com/prabhavdogra/vm)
