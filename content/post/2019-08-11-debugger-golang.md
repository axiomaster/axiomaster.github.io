---
layout:     post
title:      Making debugger for Golang
subtitle:   part I
date:       2019-08-11
author:     axiomaster
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - debugger
---

> 转载自 [Making debugger for Golang (part I)](https://medium.com/golangspec/making-debugger-for-golang-part-i-53124284b7c8)

The goal of this series is not to write full-featured debugger for Go programming language. If you’re looking for some then please take a look at Delve. We’ll try to understand here how debuggers work in general and how to implement basic one on Linux which takes into account Golang’s features like goroutines.

Creating debugger isn’t easy though. We won’t even strive to cover this topic within a single story. Instead, this post starts a series with ultimate goal to have working solution handling most common scenarios. Along the way we’ll discuss topics like ELF, DWARF and will touch some architecture-specific issues.

## Environment

Throughout this series we’ll use Docker to get repeatable playground environment based on Debian Jessie. I’m using x86–64 and this come into play at some point when we’ll move to low-level discussions. Project’s layout is like this:

```
> tree
.
├── Dockerfile
└── src
    └── github.com
        └── mlowicki
            ├── debugger
            │   └── debugger.go
            └── hello
                └── hello.go
```

The main file of upcoming debugger is debugger.go and file hello.go holds source code of sample program being debugged while our journey. For now you can put there absolute minimum:
package main

```golang
func main() {
}
```

Let’s start with very simple Dockerfile:

```
FROM golang:1.8.1
RUN apt-get update && apt-get install -y tree
```

To build Docker image, go to the top-level directory (where Dockerfile sits) and run:

```
> docker build -t godebugger .
```

To spin the container up execute:

```
> docker run --rm -it -v "$PWD"/src:/go/src --security-opt seccomp=unconfined godebugger
```

Secure computing mode (*seccomp*) is described here. Now what is left is to compile both programs inside the container. First one can be done using:

```
> go install --gcflags="-N -l" github.com/mlowicki/hello
```

Flag ```—-gcflag``` has been used to disable function inlining (```-l```) and compiler optimizations (```-N```) to make debugging easier. Debugger can be built using:

```
> go install github.com/mlowicki/debugger
```

Inside the container ```PATH``` environment variable contains ```/go/bin``` so to run any of just prepared programs type either hello or debugger without full path.

## First step

Our first task will be simple. Let’s stop the program before executing any instruction and then start it again, until it terminates (either voluntarily or caused by error). This is how you start working with most debuggers. You set some traps (breakpoints) and then run something like ```continue``` to actually launch till it’ll stopped at one of desired places. Let’s see how it works with Delve:

```
> cat hello.go

package main

import "fmt"

func f() int {
    var n int
    n = 1
    n = 2
    return n
}

func main() {
    fmt.Println(f())
}

> dlv debug
break Type ‘help’ for list of commands.
(dlv) break main.f
Breakpoint 1 set at 0x1087050 for main.f() ./hello.go:5
(dlv) continue
> main.f() ./hello.go:5 (hits goroutine(1):1 total:1) (PC: 0x1087050)
     1: package main
     2:
     3: import "fmt"
     4:
=>   5: func f() int {
     6:     var n int
     7:     n = 1
     8:     n = 2
     9:     return n
    10: }
(dlv) next
> main.f() ./hello.go:6 (PC: 0x1087067)
     1: package main
     2:
     3: import "fmt"
     4:
     5: func f() int {
=>   6:     var n int
     7:     n = 1
     8:     n = 2
     9:     return n
    10: }
    11:
(dlv) print n
842350461344
(dlv) next
> main.f() ./hello.go:7 (PC: 0x108706f)
     2:
     3: import "fmt"
     4:
     5: func f() int {
     6:     var n int
=>   7:     n = 1
     8:     n = 2
     9:     return n
    10: }
    11:
    12: func main() {
(dlv) print n
0
(dlv) next
> main.f() ./hello.go:8 (PC: 0x1087077)
     3: import "fmt"
     4:
     5: func f() int {
     6:     var n int
     7:     n = 1
=>   8:     n = 2
     9:     return n
    10: }
    11:
    12: func main() {
    13:     fmt.Println(f())
(dlv) print n
1
```

Let’s see how to implement it on our own.

The first step is to have a mechanism for a process (our debugger) to control other process (program being debugged). Luckily on Linux we’ve something in place — ptrace. It’s not the last good news. Golang’s syscall package provides an interface to it like PtraceCont to restart traced process. So it covers the 2nd part but to have a chance to f.ex. set breakpoints before program starts its execution we need something more. While creating new process we can specify its behaviour through set of attributes— SysProcAttr. One of them is Ptrace which enables tracking and process will stop and send SIGSTOP signal to its parent before start. Let’s put everything we’ve just learned into a working machinery…

```
> cat src/github.com/mlowicki/hello/hello.go

package main

import "fmt"

func main() {
    fmt.Println("hello world")
}

> cat src/github.com/mlowicki/debugger/debugger.go

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
    log.Println("Restarting...")
    err = syscall.PtraceCont(cmd.Process.Pid, 0)
    if err != nil {
        log.Panic(err)
    }
    var ws syscall.WaitStatus
    _, err = syscall.Wait4(cmd.Process.Pid, &ws, syscall.WALL, nil)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Exited: %v\n", ws.Exited())
    log.Printf("Exit status: %v\n", ws.ExitStatus())
}

> go install -gcflags="-N -l" github.com/mlowicki/hello
> go install github.com/mlowicki/debugger
> debugger /go/bin/hello
2017/05/05 20:09:38 State: stop signal: trace/breakpoint trap
2017/05/05 20:09:38 Restarting...
hello world
2017/05/05 20:09:38 Exited: true
2017/05/05 20:09:38 Exit status: 0
```

One first version of debugger works in a very simple way. It start new process which is traced so it stops before executing first instruction and sends signal to the parent process. Parent process waits for such signal and issues logs like ```log.Printf("State: %v\n", err)```. Afterwards process is restarted and parent waits for its termination. Such behaviour will give as an opportunity to f.ex. set breakpoints, start process and while reaching certain trap, inspect program’s state like current values of variables placed on stack or inside registers.

Even knowing so little we can do some pretty powerful things. It’ll lay the foundations for future enhancements and experiments (more soon).

Click ❤ below to help others discover this story. Please follow me if you want to get updates about new posts or boost work on future stories.

