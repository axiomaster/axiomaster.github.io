---
layout:     post
title:      Making debugger for Golang
subtitle:   part II
date:       2019-08-11
author:     axiomaster
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - debugger
---

> 转载自 [Making debugger for Golang (part II)](https://medium.com/golangspec/making-debugger-in-golang-part-ii-d2b8eb2f19e0)

During the first part we’ve bootstrapped development environment and made a simple program (tracer) which stops child process (tracee) at the very beginning and then continues it execution together with showing its standard output. Now it’s time to extend its basic capabilities.

Usually debuggers allow to single-step through debugged program. It can be done using ptrace *PTRACE_SINGLESTEP* request which tells tracee to stop after execution of single instruction:

```golang
package main

import (
    "flag"
    "log"
    "os"
    "os/exec"
    "syscall"
)

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
    wpid := cmd.Process.Pid
    pgid, err := syscall.Getpgid(cmd.Process.Pid)
    if err != nil {
        log.Panic(err)
    }
    err = syscall.PtraceSetOptions(cmd.Process.Pid, syscall.PTRACE_O_TRACECLONE)
    if err != nil {
        log.Fatal(err)
    }
    err = syscall.PtraceSingleStep(wpid)
    if err != nil {
        log.Fatal(err)
    }

    steps := 1
    for {
        var ws syscall.WaitStatus
        wpid, err = syscall.Wait4(-1*pgid, &ws, syscall.WALL, nil)
        if wpid == -1 {
            log.Fatal(err)
        }
        if wpid == cmd.Process.Pid && ws.Exited() {
            break
        }
        if !ws.Exited() {
            err := syscall.PtraceSingleStep(wpid)
            if err != nil {
                log.Fatal(err)
            }
            steps += 1
        }
    }
    log.Printf("Steps: %d\n", steps)
}
```

Building and output looks like this (number of steps can vary between each calls):

```
> go install -gcflags="-N -l" github.com/mlowicki/hello
> go install github.com/mlowicki/debugger
> debugger /go/bin/hello
2017/06/09 19:54:42 State: stop signal: trace/breakpoint trap
hello world
2017/06/09 19:54:49 Steps: 297583
```

First part is the same as in our initial program from previous post. What is new is the use of syscall.PtraceSingleStep. It stops tracee (hello in our case) after execution of single instruction.

Option ```PTRACE_O_TRACECLONE``` has been also set:

```
PTRACE_O_TRACECLONE (since Linux 2.5.46)
Stop the tracee at the next clone(2) and automatically start tracing the newly cloned process...
```

(http://man7.org/linux/man-pages/man2/ptrace.2.html)

Thanks to that our debugger knows when new thread has been started and can step through it as well so the number of steps at the very end is a sum of executed instructions across all processes.

Number of instructions may seem pretty huge but it contains among others initialization of the whole Golang runtime (libc initialization should be known to those with C experience). We can prepare very simple program to check that our counting works fine. Let’s create *src/github.com/mlowicki/hello/hello.asm:*

```assembly
section .data
    msg db "hello, world!", 0xA
    len equ $ — msg

section .text
    global _start

_start:
    mov rax, 1 ; write syscall (https://linux.die.net/man/2/write)
    mov rdi, 1 ; stdout
    mov rsi, msg
    mov rdx, len
    ; Passing parameters to `syscall` instruction described in
    ; https://en.wikibooks.org/wiki/X86_Assembly/Interfacing_with_Linux#syscall
    syscall
    mov rax, 60 ; exit syscall (https://linux.die.net/man/2/exit)
    mov rdi, 0 ; exit code
    syscall
```

Inside the container let’s build our “hello world” program and see how many instructions we’ll count there:

```
> pwd
/go
> apt-get install nasm
> nasm -f elf64 -o hello.o src/github.com/mlowicki/hello/hello.asm && ld -o hello hello.o
> ./hello
hello, world!
> debugger ./hello
2017/06/17 17:58:43 State: stop signal: trace/breakpoint trap
hello, world!
2017/06/17 17:58:43 Steps: 8
```

It looks good since we get the exact number of instructions inside hello.asm.

So far we’ve learned how to stop the program at the very beginning, step through it instruction by instruction and track its processes / threads. Now it’s time to set trap at desired place and inspect state of the process like variable values.

Let’s start with some basics. We have a function main from our hello.go:

```
package main
import "fmt"
func main() {
    fmt.Println("hello world")
}
```

How to set a trap at the beginning of this function? Ultimately our program after compilation and linking is a set of machine instructions. How to express that we want to set a breakpoint at particular place in the source code having only cooked binary (stuffed with instructions only CPUs understand)?

## LineTable

Golang has built-in support to access debug information contained in Go binaries. The struct which maps program counter (PC) to line number and vice versa is called LineTable. Let’s see it in action:

```
package main

import (
    "debug/elf"
    "debug/gosym"
    "flag"
    "log"
)

func main() {
    flag.Parse()
    path := flag.Arg(0)
    exe, err := elf.Open(path)
    if err != nil {
        log.Fatal(err)
    }

    var pclndat []byte
    if sec := exe.Section(".gopclntab"); sec != nil {
        pclndat, err = sec.Data()
        if err != nil {
            log.Fatalf("Cannot read .gopclntab section: %v", err)
        }
    }

    sec := exe.Section(".gosymtab")
    symTabRaw, err := sec.Data()
    pcln := gosym.NewLineTable(pclndat, exe.Section(".text").Addr)
    symTab, err := gosym.NewTable(symTabRaw, pcln)
    if err != nil {
        log.Fatal("Cannot create symbol table: %v", err)
    }

    sym := symTab.LookupFunc("main.main")
    filename, lineno, _ := symTab.PCToLine(sym.Entry)
    log.Printf("filename: %v\n", filename)
    log.Printf("lineno: %v\n", lineno)
}
```

If we pass as an argument to above program something like:

```
1 package main
2
3 import "fmt"
4
5 func main() {
6     fmt.Println("hello world")
7 }
```

then it gives expected result:

```
> go install github.com/mlowicki/linetable
> go install — gcflags=”-N -l” github.com/mlowicki/hello
> linetable /go/bin/hello
2017/06/30 18:47:38 filename: /go/src/github.com/mlowicki/hello/hello.go
2017/06/30 18:47:38 lineno: 5
```

ELF stands for Executable and Linkable Format. It’s among others a format of executables:

```
> apt-get install file
> file /go/bin/hello
/go/bin/hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```

ELF file consists of sections and we’re using three of them: .text, .gopclntab and .gosymtab. The first one contains machine instructions, 2nd one maps instruction counter to source code lines and the last one is a symbol table.

In following story we’ll learn how to set a trap at desired place to make more practical use of LineTable and will see how to inspect program’s state.

Click ❤ below to help others discover this story. Please follow me if you want to get updates about new posts or boost work on future stories.