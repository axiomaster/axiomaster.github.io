---
layout:     post
title:      Making debugger for Golang
subtitle:   part III
date:       2019-08-11
author:     axiomaster
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - debugger
---

> 转载自 [Making debugger for Golang (part III)](https://medium.com/golangspec/making-debugger-in-golang-part-iii-5aac8e49f291)

So far we’ve learned how to single-step ptraced process (tracee) and get some debugging info out of binary (read it here if you haven’t done it yet). Now it’s time to set a breakpoint at desired place, wait until reached and then investigate process state.

Let’s start with assembly code used before:

```assembly
section .data
    msg db      "hello, world!", 0xA
    len equ     $ - msg

section .text
    global _start

_start:
    mov     rax, 1 ; write syscall (https://linux.die.net/man/2/write)
    mov     rdi, 1 ; stdout
    mov     rsi, msg
    mov     rdx, len
    ; Passing parameters to `syscall` instruction described in
    ; https://en.wikibooks.org/wiki/X86_Assembly/Interfacing_with_Linux#syscall
    syscall
    mov    rax, 60 ; exit syscall (https://linux.die.net/man/2/exit)
    mov    rdi, 0 ; exit code
    syscall
```

The goal set a breakpoint at:

```
mov     rdi, 1
```

so the trap will be activated before running this instruction. Then we’ll check that RDI register stores 0, run single step and confirm that register stores 1.

## Breakpoint

For x86 processors there’s a INT instruction which generates software interrupt. These interrupts are used f.ex. to do syscall on Linux. x86–64 introduced dedicated syscall instruction which is faster this is why I’ve used it above but we could achieve the same with regular int. To set a breakpoint we’ll use ```int 3``` which has an opcode ```0xCC```:

> The INT 3 instruction generates a special one byte opcode (CC) that is intended for calling the debug exception handler. (This one byte form is valuable because it can be used to replace the first byte of any instruction with a breakpoint, including other one byte instructions, without over-writing other code).

[(Intel® 64 and IA-32 Architectures Software Developer’s Manual)](https://software.intel.com/en-us/articles/intel-sdm)

We’ll use 0xCC to replace with it instruction at desired spot. Once that place will be reached we will:

1 investigate process state,

2 replace 0xCC back with its original value,

3 decrease program counter by 1,

4 run a single step.

The first question is where to put ```0xCC```. We don’t know where exactly in memory instruction ```mov rdi, 1``` will be stored. It’ll be the size of mov rax, 1 bytes further than the first instruction of the program since it’s a second instruction. Instruction lengths on x86 vary which makes the whole task more difficult. Location of the first instruction can be checked by stopping the program before processing any instruction (we’ve done that before). The length of the first instruction can be retrieved with a help of objdump program:

```
> nasm -f elf64 -o hello.o src/github.com/mlowicki/hello/hello.asm && ld -o /go/bin/hello hello.o
> objdump -d -M intel /go/bin/hello
/go/bin/hello:     file format elf64-x86-64
Disassembly of section .text:
00000000004000b0 <_start>:
  4000b0:       b8 01 00 00 00          mov    eax,0x1
  4000b5:       bf 01 00 00 00          mov    edi,0x1
  4000ba:       48 be d8 00 60 00 00    movabs rsi,0x6000d8
  4000c1:       00 00 00
  4000c4:       ba 0e 00 00 00          mov    edx,0xe
  4000c9:       0f 05                   syscall
  4000cb:       b8 3c 00 00 00          mov    eax,0x3c
  4000d0:       bf 00 00 00 00          mov    edi,0x0
  4000d5:       0f 05                   syscall
```

From above we see that the size of first instruction is 5 bytes (```4000b5 — 4000b0```). So for the address of first instruction we need to add 5 bytes and put ```0xCC``` there. Let’s see it in action:

```
package main

import (
    "flag"
    "log"
    "os"
    "os/exec"
    "syscall"
)

func step(pid int) {
    err := syscall.PtraceSingleStep(pid)
    if err != nil {
        log.Fatal(err)
    }
}

func cont(pid int) {
    err := syscall.PtraceCont(pid, 0)
    if err != nil {
        log.Fatal(err)
    }
}

func setPC(pid int, pc uint64) {
    var regs syscall.PtraceRegs
    err := syscall.PtraceGetRegs(pid, &regs)
    if err != nil {
        log.Fatal(err)
    }
    regs.SetPC(pc)
    err = syscall.PtraceSetRegs(pid, &regs)
    if err != nil {
        log.Fatal(err)
    }
}

func getPC(pid int) uint64 {
    var regs syscall.PtraceRegs
    err := syscall.PtraceGetRegs(pid, &regs)
    if err != nil {
        log.Fatal(err)
    }
    return regs.PC()
}

func setBreakpoint(pid int, breakpoint uintptr) []byte {
    original := make([]byte, 1)
    _, err := syscall.PtracePeekData(pid, breakpoint, original)
    if err != nil {
        log.Fatal(err)
    }
    _, err = syscall.PtracePokeData(pid, breakpoint, []byte{0xCC})
    if err != nil {
        log.Fatal(err)
    }
    return original
}

func clearBreakpoint(pid int, breakpoint uintptr, original []byte) {
    _, err := syscall.PtracePokeData(pid, breakpoint, original)
    if err != nil {
        log.Fatal(err)
    }
}

func printState(pid int) {
    var regs syscall.PtraceRegs
    err := syscall.PtraceGetRegs(pid, &regs)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("RAX=%d, RDI=%d\n", regs.Rax, regs.Rdi)
}

func main() {
    flag.Parse()
    input := flag.Arg(0)
    cmd := exec.Command(input)
    cmd.Args = []string{input}
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.SysProcAttr = &syscall.SysProcAttr{Ptrace: true}
    err := cmd.Start()
    if err != nil {
        log.Fatal(err)
    }
    err = cmd.Wait()
    log.Printf("State: %v\n", err)
    pid := cmd.Process.Pid
    breakpoint := uintptr(getPC(pid) + 5)
    original := setBreakpoint(pid, breakpoint)
    cont(pid)
    var ws syscall.WaitStatus
    _, err = syscall.Wait4(pid, &ws, syscall.WALL, nil)
    clearBreakpoint(pid, breakpoint, original)  
    printState(pid)
    setPC(pid, uint64(breakpoint))
    step(pid)
    _, err = syscall.Wait4(pid, &ws, syscall.WALL, nil)
    printState(pid)
}
```

Set of helpers are placed at the top. Functions setPC and getPC are there to manipulate program counter. It’s worth to note that PC register holds the next instruction to be executed. If we’re stopping the process before running anything then PC hods an address of memory where program’s first instruction is placed. Functions to manipulate breakpoints (setBreakpoint and clearBreakpoint) are juggling the memory by either inserting 0xCC or removing it respectively. Output is as follows:

```
> go install github.com/mlowicki/breakpoint
> breakpoint /go/bin/hello
2017/07/16 21:06:33 State: stop signal: trace/breakpoint trap
2017/07/16 21:06:33 RAX=1, RDI=0
2017/07/16 21:06:33 RAX=1, RDI=1
```

It looks fine. When process reaches trap, RDI register isn’t set (holds 0). After single step (running the 2nd instruction) register RDI has desired value set by instruction:

```
mov     rdi, 1
```

We’ve accomplished the task defined at the very top of this story. It requires extra work like calculating instructions lengths but we’ll fix that at some point so no worries.

## REPL

Now it’s time to create the basic structure of our debugger. It’s a simple program which handles in a loop commands like “set a breakpoint at”, “go single step”.

```
package main
import (
    "bufio"
    "flag"
    "fmt"
    "io"
    "log"
    "os"
    "os/exec"
    "strings"
    "syscall"
)
func initTracee(path string) int {
    cmd := exec.Command(path)
    cmd.Args = []string{path}
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.SysProcAttr = &syscall.SysProcAttr{Ptrace: true}
    err := cmd.Start()
    if err != nil {
        log.Fatal(err)
    }
    err = cmd.Wait()
    // Process should be stopped here because of trace/breakpoint trap
    if err == nil {
        log.Fatal("Program exited")
    }
    return cmd.Process.Pid
}
func main() {
    flag.Parse()
    _ = initTracee(flag.Arg(0))
    for {
        reader := bufio.NewReader(os.Stdin)
        fmt.Print("> ")
        command, err := reader.ReadString('\n')
        if err != nil {
            if err == io.EOF {
                fmt.Println()
                break
            }
            log.Fatal(err)
        }
        command = command[:len(command)-1] // get rid of ending newline character
        if strings.HasPrefix(command, "register ") {
            fmt.Println("register...")
        } else if strings.HasPrefix(command, "breakpoint ") {
            fmt.Println("breakpoint...")
        } else if command == "help" {
            fmt.Println("help...")
        } else if command == "step" {
            fmt.Println("step...")
        } else if command == "continue" {
            fmt.Println("continue")
        } else {
            fmt.Println("unknown command")
        }
    }
}
```

This is the backbone for our debugger. It doesn’t provide too much (yet) but already has the necessary logic to provide elementary REPL environment. We know more or less how to handle step, continue, help and register commands and we’ll implement these gaps soon. The one which isn’t that obvious for Golang is a breakpoint command. It’ll thoroughly explained in the upcoming post why it’s a bit harder than expected and how to overcome that extra complexity.

Click ❤ below to help others discover this story. Please follow me if you want to get updates about new posts or boost work on future stories.