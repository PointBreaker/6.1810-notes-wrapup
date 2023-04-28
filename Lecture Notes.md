# 6.1810 2022 Lecture 1: O/S overview

## Overview

 - 6.1810 goals
  
    - Understand operating system (O/S) **design** and **implementation**
    - Hands-on experience **extending** a small O/S
    - Hands-on experience **writing systems software**
  
 - What is the purpose of an O/S?
    - **Abstract** the hardware for convenience and portability
    - **Multiplex** the hardware among many applications
    - **Isolate** applications in order to contain bugs
    - Allow **sharing** among cooperating applications
    - Control sharing for **security**
    - Don't get in the way of **high performance**
    - Support a wide range of **applications**

- Organization: layered picture

  [user/kernel diagram]
  - user applications: vi, gcc, DB, &c
  - kernel services
  - h/w: CPU, RAM, disk, net, &c
  - we care a lot about the interfaces and internal kernel structure
  
- What **services** does an O/S kernel typically provide?
  - process (a running program)
  - memory allocation
  - file contents
  - file names, directories
  - access control (security)
  - many others: users, IPC, network, time, terminals

- What's the application / kernel interface?
  - "System calls"
  - Examples, in C, from UNIX (e.g. Linux, macOS, FreeBSD):

       ```c
       fd = open("out", 1);
       write(fd, "hello\n", 6);
       pid = fork();
       ```

  - These look like function calls but they aren't 
  
- Why is O/S design+implementation hard and interesting?
  - many design tensions:
    - efficient vs abstract/portable/general-purpose
    - powerful vs simple interfaces
    - flexible vs secure
  - features interact: `fd = open(); fork()`
  - uses are varied: laptops, smart-phones, cloud, virtual machines, embedded
  - evolving hardware: NVRAM, multi-core, fast networks
  - unforgiving environment: quirky h/w, hard to debug

- You'll be glad you took this course if you...
  - care about what goes on under the hood
  - like infrastructure
  - need to track down bugs or security problems
  - care about high performance

## Class structure

- Online course information:
  https://pdos.csail.mit.edu/6.1810/ --- schedule, assignments, labs
  Piazza --- announcements, discussion, lab help

- Lectures

  - O/S ideas

  - case study of xv6, a small O/S, via code and xv6 book

  - lab background

  - O/S papers

  - submit a question about each reading, before lecture.

- Labs: 
  - The point: hands-on experience
  - Mostly one week each.
  - Three kinds:
    - Systems programming (due next week...)
    - O/S primitives, e.g. thread switching.
    - O/S kernel extensions to xv6, e.g. network.
  - Use piazza to ask/answer lab questions.
  - Discussion is great, but please do not look at others' solutions!

- Grading:
- 70% labs, based on tests (the same tests you run).
- 20% lab check-off meetings: we'll ask you about randomly-selected labs.
- 10% home-work and class/piazza discussion.
- No exams, no quizzes.
- Note that most of the grade is from labs. Start them early!

## Introduction to UNIX system calls

- Applications see the O/S via system calls; that interface will be a big focus.
  let's start by looking at how programs use system calls, you'll use these system calls in the first lab, and extend and improve them in subsequent labs.
- I'll show some examples, and run them on xv6.
  xv6 has similar structure to UNIX systems such as Linux, but much simpler, you'll be able to digest all of xv6, accompanying book explains how xv6 works, and why.
  - why UNIX?
    - open source, well documented, clean design, widely used
    - studying xv6 will help if you ever need to look inside Linux 
  - xv6 has two roles in 6.1810: example of core functions: 
    - virtual memory, multi-core, interrupts, &c
    -  starting point for most of the labs
  - xv6 runs on RISC-V, as in 6.004
  - you'll run xv6 under the qemu machine emulator

#### example: copy.c, copy input to output read bytes from input, write them to the output

```c
// copy.c: copy input to output.

#include "kernel/types.h"
#include "user/user.h"

int
main()
{
  char buf[64];

  while(1){
    int n = read(0, buf, sizeof(buf));
    if(n <= 0)
      break;
    write(1, buf, n);
  }

  exit(0);
}
```

- copy.c is written in C
  
   - Kernighan and Ritchie (K&R) book is good for learning C, you can find these example programs via the schedule on the web site
       `read()` and `write() `are system calls
      
   - first `read()/write()` argument is a "file descriptor" (fd) 

       - passed to kernel to tell it which "open file" to read/write must previously have been opened an FD connects to a` file/device/socket/&c`
   
       - a process can open many files, have many FDs
       
       - UNIX convention: fd `0` is "standard input", `1` is "standard output"    
      
   - second `read()` argument is a pointer to some memory into which to read
   
  - third argument is the number of bytes to read
        `read()` may read less, but not more
  
   - return value: number of bytes actually read, or -1 for error
        - note: copy.c does not care about the format of the data
      - UNIX I/O is 8-bit bytes
      - interpretation is application-specific, e.g. database records, C source, &c
 - where do file descriptors come from?
   
#### example: open.c, create a file

  ```c
  // open.c: create a file, write to it.
  
  #include "kernel/types.h"
  #include "user/user.h"
  #include "kernel/fcntl.h"
  
  int
  main()
  {
    int fd = open("out", O_WRONLY | O_CREATE | O_TRUNC);
    write(fd, "ooo\n", 4);
  
    exit(0);
  }
  ```

  `$ cat out`

  `open() `creates a file or open a existing file, returns a file descriptor (or `-1` for error)

  - FD is a small integer
  - FD indexes into a per-process table maintained by kernel
    [user/kernel diagram]
  - different processes have different FD name-spaces
      i.e. FD 1 usually means different things to different processes
  - these examples ignore errors ** don't be this sloppy!

  <img src="https://picgo-1252947055.cos.ap-guangzhou.myqcloud.com/image-20230426010023069.png" alt="Fig1.2"  />

  - Figure 1.2 in the xv6 book lists system call arguments/return or look at UNIX man pages, e.g. "man 2 open"

- what happens when a program calls a system call like` open()`?

  - looks like a function call, but it's actually a special instruction
  - hardware saves some user registers
  - hardware increases privilege level
  - hardware jumps to a known "entry point" in the kernel
  - now running C code in the kernel
  - kernel calls system call implementation
      `sys_open() `looks up name in file system, it might wait for the disk, it updates kernel data structures (file block cache, FD table)
  - restore user registers
  - reduce privilege level
  - jump back to calling point in the program, which resumes
  - we'll see more detail later in the course

- I've been typing to UNIX's command-line interface, the shell.

  - the shell prints the `$` prompts.

  - the shell lets you run UNIX command-line utilities useful for system management, messing with files, development, scripting

     `$ ls`

     `$ ls > out`

     ` $ grep x < out`

  - UNIX supports other styles of interaction too
      window systems, GUIs, servers, routers, &c.

  - but time-sharing via the shell was the original focus of UNIX.

  - we can exercise many system calls via the shell.

#### example: fork.c, create a new process

  ```c
  // fork.c: create a new process
  
  #include "kernel/types.h"
  #include "user/user.h"
  
  int
  main()
  {
    int pid;
  
    pid = fork();
  
    printf("fork() returned %d\n", pid);
  
    if(pid == 0){
      printf("child\n");
    } else {
      printf("parent\n");
    }
    
    exit(0);
  }
  ```

- the shell creates a new process for each command you type, e.g. for
    `$ echo hello`
- the `fork() `system call creates a new process
    `$ fork`
- the kernel makes a copy of the calling process
  - instructions, data, registers, file descriptors, current directory
      "parent" and "child" processes
- only difference: `fork()` returns a `pid` in parent, `0` in child
- a pid (process ID) is an integer, kernel gives each process a different pid
  thus:
  - fork.c's "`fork()` returned" executes in -both- processes
  - the "`if(pid == 0)`" allows code to distinguish 
- ok, fork lets us **create** a new process
- how can we **run** a program in that process?

#### example: exec.c, replace calling process with an executable file

  ```c
  // exec.c: replace a process with an executable file
  
  #include "kernel/types.h"
  #include "user/user.h"
  
  int
  main()
  {
    char *argv[] = { "echo", "this", "is", "echo", 0 };
  
    exec("echo", argv);
  
    printf("exec failed!\n");
  
    exit(0);
  }
  ```

- how does the shell run a program, e.g.
    `$ echo a b c`

- a program is stored in a file: instructions and initial memory created by the **compiler** and **linker**

- so there's a file called echo, containing instructions
  `$ exec`

  - `exec()` replaces current process with an executable file

    - discards instruction and data memory

    - loads instructions and memory from the file

  - preserves file descriptors

  - ` exec(filename, argument-array)`

    - argument-array holds command-line arguments; exec passes to main()
    - cat user/echo.c
    - echo.c shows how a program looks at its command-line arguments 

#### example: forkexec.c, `fork()` a new process, `exec() `a program
  ```c
  #include "kernel/types.h"
  #include "user/user.h"
  
  // forkexec.c: fork then exec
  
  int
  main()
  {
    int pid, status;
  
    pid = fork();
    if(pid == 0){
      char -argv[] = { "echo", "THIS", "IS", "ECHO", 0 };
      exec("echo", argv);
      printf("exec failed!\n");
      exit(1);
    } else {
      printf("parent waiting\n");
      wait(&status);
      printf("the child exited with status %d\n", status);
    }
  
    exit(0);
  }
  ```

- forkexec.c contains a common UNIX idiom:
  - `fork()` a child process
  - `exec()` a command in the child
  - parent `wait()`s for child to finish
- the shell does `fork/exec/wait` for every command you type
  - after `wait()`, the shell prints the next prompt
      to run in the background ** & ** the shell skips the `wait()`
  - `exit(status)`->` wait(&status)`
      status convention: 0 = success, 1 = command encountered an error
  - note: the `fork() `copies, but `exec() `discards the copied memory
      this may seem wasteful, you'll transparently eliminate the copy in the "copy-on-write" lab

#### example: redirect.c, redirect the output of a command

  ```c
  #include "kernel/types.h"
  #include "user/user.h"
  #include "kernel/fcntl.h"
  
  // redirect.c: run a command with output redirected
  
  int
  main()
  {
    int pid;
  
    pid = fork();
    if(pid == 0){
      close(1);
      open("out", O_WRONLY | O_CREATE | O_TRUNC);
  
      char *argv[] = { "echo", "this", "is", "redirected", "echo", 0 };
      exec("echo", argv);
      printf("exec failed!\n");
      exit(1);
    } else {
      wait((int *) 0);
    }
  
    exit(0);
  }
  ```

- what does the shell do for this?
   ` $ echo hello > out`
- answer: fork, change FD 1 in child, exec echo
  `$ redirect`
  `$ cat out`
- note: `open()` always chooses lowest unused FD; `1` due to `close(1)`.
- fork, FDs, and exec interact nicely to implement I/O redirection
  - separate fork-then-exec give child a chance to change FDs before exec
  - FDs provide indirection
        commands just use FDs `0` and `1`, don't have to know where they go
  - exec preserves the FDs that sh set up
- thus: only sh has to know about I/O redirection, not each program
- It's worth asking "why" about design decisions:
  - Why these I/O and process abstractions? Why not something else?
  - Why provide a file system? Why not let programs use the disk their own way?
  - Why FDs? Why not pass a filename to write()?
  - Why are files streams of bytes, not disk blocks or formatted records?
  - Why not combine `fork() `and `exec()`?
  - The UNIX design works well, but we will see other designs!

#### example: pipe1.c, communicate through a pipe

```c
// pipe1.c: communication over a pipe

#include "kernel/types.h"
#include "user/user.h"

int
main()
{
  int fds[2];
  char buf[100];
  int n;

  // create a pipe, with two FDs in fds[0], fds[1].
  pipe(fds);
  
  write(fds[1], "this is pipe1\n", 14);
  n = read(fds[0], buf, sizeof(buf));

  write(1, buf, n);

  exit(0);
}
```

- how does the shell implement

   `$ ls | grep x`

   `$ pipe1`
- an FD can refer to a "pipe", as well as a file
  - the `pipe()` system call creates two FDs
    - read from the first FD
    - write to the second FD

- the kernel maintains a buffer for each pipe
    [u/k diagram]
  - `write() `appends to the buffer
  - `read() `waits until there is data 

#### example: pipe2.c, communicate between processes

  ```c
  #include "kernel/types.h"
  #include "user/user.h"
  
  // pipe2.c: communication between two processes
  
  int
  main()
  {
    int n, pid;
    int fds[2];
    char buf[100];
    
    // create a pipe, with two FDs in fds[0], fds[1].
    pipe(fds);
  
    pid = fork();
    if (pid == 0) {
      write(fds[1], "this is pipe2\n", 14);
    } else {
      n = read(fds[0], buf, sizeof(buf));
      write(1, buf, n);
    }
  
    exit(0);
  }
  ```

- pipes combine well with `fork()` to implement `ls | grep x`
  - shell creates a pipe,
  - then forks (twice),
  - then connects ls's FD 1 to pipe's write FD,
  - and grep's FD 0 to the pipe
      [diagram]
    `$ pipe2` ** a simplified version 
- pipes are a separate abstraction, but combine well w/` fork()`

#### example: list.c, list files in a directory

  ```c
  #include "kernel/types.h"
  #include "user/user.h"
  
  // list.c: list file names in the current directory
  
  struct dirent {
    ushort inum;
    char name[14];
  };
  
  int
  main()
  {
    int fd;
    struct dirent e;
  
    fd = open(".", 0);
    while(read(fd, &e, sizeof(e)) == sizeof(e)){
      if(e.name[0] != '\0'){
        printf("%s\n", e.name);
      }
    }
    exit(0);
  }
  ```

- how does ls get a list of the files in a directory?
- you can open a directory and read it -> file names
  `.` is a pseudo-name for a process's current directory
- see ls.c for more details

### Summary
  - We've looked at UNIX's I/O, file system, and process abstractions.
  - The interfaces are simple --- just integers and I/O buffers.
  - The abstractions combine well, e.g. for I/O redirection.

You'll use these system calls in the first lab, due next week.

# 6.1810 2022 Lecture 2: programming xv6 in C

## why C?
 - good for low-level programming
    - easy mapping between C and RISC-V instructions
    - easy mapping between C types and hardware structures
    - e.g.., set bit flags in hardware registers of a device
 - minimal runtime
    - easy to port to another hardware platform
    - direct access to hardware
 - explicit memory management
    - no garbage collector
    - kernel is in complete control of memory management
 - efficient: compiled (no interpreter)
    - compiler compiles C to assembly
 - popular for building kernels, system software, etc.
    - good support for C on almost any platform
 - why not?
    - easy to write incorrect code
    - easy to write code that has security vulnerabilities
- today's lecture: use of C in xv6
    - memory layout
    - pointers
    - arrays
    - strings
    - lists
    - bitwise operators
      [not a general intro to C]

- memory layout of a C program in xv6

  ![Fig 3.4](https://picgo-1252947055.cos.ap-guangzhou.myqcloud.com/image-20230426023505163.png)  [draw figure, see fig 3.4 of text]

  -  text: code, read-only data
  - data: global C variables
  - stack: function's local variables
  - heap: dynamic memory allocation using sbrk, malloc/free

- example: compile cat.c

    Makefile defines how

  - `gcc` compiles to `.o`
  - `ld` links `.o `files into an executable
       ` ulibc.o `is xv6 minimal C library
  - executable has `a.out `format with sections for:
        text (code), initialized data, symbol table, debug info, and more

- explore `a.out` of `_cat`
  
   ` riscv64-linux-gnu-objdump -S user/_cat`

    same as user/cat.asm

  - 0x0: cat
  
    what if we run two cat programs at the same time?

    see pgtbl lecture

  - 0x8e: _main
  
    user.ld:
        entry: _main

  - what is _main?
  
    defined in ulib.c, which calls `main()` and `exit(0)`
    
  - where is data memory? (e.g., buf)
  
    in data/bss segment

    must be setup by kernel

  - but we know address where buf should be
  
       ` riscv64-linux-gnu-nm -n user/_cat`

## C pointers

- a pointer is a memory address
  - every variable has a memory address (i.e., `p = &i`)
  - so each variable can be accessed through its pointer (i.e., `*i`)
  - a pointer can be variable (e.g., int *p)
    - and thus has a memory address, etc.
-  pointer arithmetic

    ```c
    char *c;
    int *i;
    ```

​    what is the value of c+1 and i+1?

- referencing elements of a struct

    ```c
    struct {
        int a;
        int b;
    } *p;
    p->a = 10
    ```

    ```c
    #include "kernel/types.h"
    #include "user/user.h"
    
    int g = 3;
        
    int
    main(int ac, char **av)
    {
    int l = 5;   // local variables don't have a default value
    int *p, *q;
    
    // take address of variable
    p = &g;
    q = &l;
    printf("p %p q %p\n", p, q);
    
    // assign using pointer
    *p = 11;
    *q = 13;
    printf("g %d l %d\n", g, l);
    
    // struct
    struct two {
        int a;
        int b;
    } s;
    s.a = 10;
    struct two *ptr = &s;
    printf("%d %d\n", s.a, ptr->a);
    
    // can take address of any variable
    int **pp;
    pp = &p;    // take address of a pointer variable
    printf("pp %p %p %d\n", pp, *pp, **pp);
    
    int (*f)(int, char **);
    f = &main;  // take address of a function<
    printf("main: %p\n", f);
    
    return 0;
    }
    ```

## C arrays

- contiguous memory holding same data type (char, int, etc.)
      no bound checking, no growing

- two ways to access arrays:
      through index: `buf[0]`, `buf[1]`
      through pointer:` *buf`, `*(buf+1)`

  ```c
  #include "kernel/types.h"
  #include "user/user.h"
  
  int a[3] = {1, 2, 3};    // an array of 3 int's
  char b[3] = {'a', 'b', 'c'};  // an array of 3 char's
  	
  int
  main(int ac, char **av)
  {
  
    // first element is at index 0
    printf("%d\n", a[0]);
    
    a[1] += 1;  // use index access
    *(a+2) = 5; // pointer access
    
    printf("%d %d\n", a[1], a[2]);
  
    // pointers to array elements
    printf("a %p a1 %p a2 %p a2 %p\n", a, a+1, a+2, &a[2]);
  
    // pointer arithmetic uses type
    printf("%p %p\n", b, b+1);
    
    return 0;
  }
  ```

## C strings

- arrays of characters, ending in 0
  ```c
  #include "kernel/types.h"
  #include "user/user.h"
  
  char *s = "123";
  	
  int
  main(int ac, char **av)
  {
    char s1[4] = {'1', '2', '3', '\0'}; 
  
    // s and s1 are strings
    printf("s %s s1 %s\n", s, s1);
  
    // can use index or pointer access
    printf("%c %c\n", s[0], *s);
    printf("%c %c\n", s[2], *(s+2));
  
    // read beyond str end; DON'T DO THIS
    printf("%x %p %p\n", s1[4], s1, &s1[4]);
  
    // write beyond str end; DON'T DO THIS
    s1[4] = 'D';
    
    return 0;
  }
  ```

- ulib.c has several functions for strings

    `strlen()` --- use array access

    `strcmp()` --- use pointer access

- ls.c

  - argv: array of strings
    - each entry has the address of a string
    - xv6's exec puts them on the stack as arguments to main
  - print out argv
          [draw diagram; see fig 3.4 in book]
  - T_DIR code fragment
  
      `mkdir d`

      `echo hi > d/f`
      
      `ls d`

## C lists (more pointers)

- single-linked list
  - kernel/kalloc.c implements a memory allocator
  - keeps a list of free "pages" of memory
          a page is 4096 bytes
          free prepends
          kalloc grabs from front of list
- double-linked list
  - kernel/bio.c implements an LRU buffer cache
  - `brelse() `needs to move a buf to the front of the list
  - see buf.h
          two pointers: prev and next

## bitwise operators
- `char/int/longs/pointers` have bits (8, 32, 64 respectively, on RISC-V).
    you can manipulate them with `|`,` &`,` ~`,` ^`

    ```c
    10001 & 10000 = 10000
    10001 | 10000 = 10001
    10001 ^ 10000 = 00001
    ~1000 = 0111
    ```

- example:

  ​    user/usertests.c
  ​    kernel/fcntl.h
  ​    kernel/sysfile.c
    more interesting examples later

## keywords:
- static: to make a variable's visibility limited to the file it is declared
      but global within the file
- void: "no type", "no value", "no parameters"

## common C bugs
-   use after free
-   double free
-   uninitialized memory
-   memory on stack or returned by malloc are not zero
-   buffer overflow
-    write beyond end of array
-   memory leak
-   type confusion
-   wrong type cast

References:
https://blog.regehr.org/archives/1393

# 6.1810 2022 Lecture 3: OS design

- Lecture Topic:
  - OS design
    - system calls
    - micro/monolithic kernel
  - First system call in xv6

- OS picture
  - apps: sh, echo, ...
  - system call interface (open, close,...)
            OS

- Goal of OS
  - run multiple applications
  - isolate them
  - multiplex them
  - share

- Strawman design: No OS
  - Application directly interacts with hardware
    - CPU cores & registers
    - DRAM chips
    - Disk blocks
          ...
  - OS library perhaps abstracts some of it

- Strawman design not conducive to multiplexing
  - each app periodically must give up hardware
  - BUT, weak isolation
    - app forgets to give up, no other app runs
    - apps has end-less loop, no other app runs
    - you cannot even kill the badly app from another app
  - but used by real-time OSes
    - "cooperative scheduling"

- Strawman design not conducive to memory isolation
  - all apps share physical memory
  - one app can overwrites another apps memory
  - one app can overwrite OS library

- Unix interface conducive to OS goals
  - abstracts the hardware in way that achieves goals
  - processes (instead of cores): fork
     - OS transparently allocates cores to processes
       - Saves and restore registers
     - Enforces that processes give them up
       - Periodically re-allocates cores     
  - memory (instead of physical memory): exec
     - Each process has its "own" memory
     - OS can decide where to place app in memory
     - OS can enforce isolation between memory of different apps
     - OS allows storing image in file system
  - files (instead of disk blocks)
     - OS can provide convenient names
     - OS can allow sharing of files between processes/users
  - pipes (instead of shared physical mem)
     - OS can stop sender/receiver

- OS must be defensive
  - an application shouldn't be able to crash OS
  - an application shouldn't be able to break out of its isolation
- => need strong isolation between apps and OS
  - approach: hardware support
    - user/kernel mode
    - virtual memory

- Processors provide user/kernel mode
  - kernel mode: can execute "privileged" instructions
    - e.g., setting kernel/user bit
    - e.g., reprogramming timer chip
  - user mode: cannot execute privileged instructions
  - Run OS in kernel mode, applications in user mode
  
  [RISC-V has also an M mode, which we mostly ignore]

- Processors provide virtual memory
  - Hardware provides page tables that translate virtual address to physical
  - Define what physical memory an application can access
  - OS sets up page tables so that each application can access only its memory

- Apps must be able to communicate with kernel
  - Write to storage device, which is shared => must be protected => in kernel
  - Exit app
  - ...

- Solution: add instruction to change mode in controlled way
  - ecall <n>
  - enters kernel mode at a pre-agreed entry point

- Modify OS picture
  - user / kernel (redline)
  - app -> printf() -> write() -> SYSTEM CALL -> sys_write() -> ...
    - user-level libraries are app's private business
  - kernel internal functions are not callable by user

  - other way of drawing picture:
    - syscall 1  -> system call stub -> kernel entry -> syscall -> fs
    - syscall 2                                                 -> proc

  - system call stub executes special instruction to enter kernel
    - hardware switches to kernel mode
    - but only at an entry point specified by the kernel

  - syscall need some way to get at arguments of syscall

  [**syscalls** the topic of this week's lab]

- Kernel is the Trusted Computing Base (**TCB**)
  - Kernel must be "correct"
    - Bugs in kernel could allow user apps to circumvent kernel/user
      - Happens often in practice, because kernels are complex
      - See CVEs
  - Kernel must treat user apps as suspect
    - User app may trick kernel to do the wrong thing
    - Kernel must check arguments carefully
    - Setup user/kernel correctly
    - Etc.
  - Kernel in charge of separating applications too
    - One app may try to read/write another app's memory
  - => Requires a security mindset
    - Any bug in kernel may be a security exploit

- Aside: can one have process isolation WITHOUT h/w-supported
  - kernel/user mode and virtual memory?
    - yes! use a strongly-typed programming language
  - For example, see Singularity O/S
    - the compiler is then the trust computing base (TCB)
    - but h/w user/kernel mode is the most popular plan

- Monolothic kernel
  - OS runs in kernel space
  - Xv6 does this.  Linux etc. too.
  - kernel interface == system call interface
  - one big program with file system, drivers, &c
  - good: easy for subsystems to cooperate
    - one cache shared by file system and virtual memory
  - bad: interactions are complex
    - leads to bugs
    - no isolation within

- Microkernel design
  - many OS services run as ordinary user programs
    - file system in a file server
  - kernel implements minimal mechanism to run services in user space
    - processes with memory
    - inter-process communication (IPC)
  - kernel interface != system call interface		
    - good: more isolation
    - bad: may be hard to get good performance
  - both monolithic and microkernel designs widely used

- Xv6 case study
  - Monolithic kernel
    - Unix system calls == kernel interface
  - Source code reflects OS organization (by convention)
    - user/    apps in user mode
    - kernel/  code in kernel mode
  - Kernel has several parts
    - kernel/defs.h
      - proc
      - fs
      - ..
  - Goal: read source code and understand it (without consulting book)

- The RISC-V computer
  - A very simple board (e.g., no display)
    - RISC-V processor with 4 cores
    - RAM (128 MB)
    - support for interrupts (PLIC, CLINT)
    - support for UART
      - allows xv6 to talk to console
      - allows xv6 to read from keyboard
    - support for e1000 network card (through PCIe)
 - Qemu emulates several RISC-V computers
   - we use the "virt" one
     - https://github.com/riscv/riscv-qemu/wiki
   - close to the SiFive board (https://www.sifive.com/boards)
     - but with virtio for disk

- Boot xv6 (under gdb)

  `$ make CPUS=1 qemu-gdb`

    runs xv6 under gdb (with 1 core)
  - Qemu starts xv6 in kernel/entry.S (see kernel/kernel.ld)
    ```text
        set breakpoint at _entry
          look at instruction
          info reg
        set breakpoint at main
          Walk through main
        single step into userinit
          Walk through userinit
          show kalloc
          show proc.h
          show allocproc()
          show initcode.S/initcode.asm
        break forkret()
          walk to userret
        break syscall
          print num
          syscalls[num]
          exec "/init"
        points to be made:
          page table in userinit
          ecall: U -> K
          a7: syscall #
          exec: defensive
    ```

# 6.1810 2022 Lecture 4: Virtual Memory/Page tables

> plan:
> 
>  - address spaces
> 
>  - paging hardware
> 
>  - xv6 VM code

## Virtual memory overview

- today's problem:

  [user/kernel diagram]

  [memory view: diagram with user processes and kernel in memory]

  - suppose the shell has a bug:
    - sometimes it writes to a random memory address
  - how can we keep it from wrecking the kernel?
    - and from wrecking other processes?

- we want isolated address spaces
  - each process has its own memory
  - it can read and write its own memory
  - it cannot read or write anything else
  - challenge: 
    - how to multiplex several memories over one physical memory?
	- while maintaining isolation between memories

- xv6 uses RISC-V's paging hardware to implement AS's
  - ask questions! this material is important
  - topic of next lab (and shows up in several other labs)

- paging provides a level of indirection for addressing
  ```txt
  CPU -> MMU -> RAM
      VA     PA
  ```
  - s/w can only ld/st to virtual addresses, not physical
  - kernel tells MMU how to map each virtual address to a physical address
    - MMU essentially has a table, indexed by va, yielding pa
    - called a "page table"
    - one page table per address space
  - MMU can restrict what virtual addresses user code can use
  - By programming the MMU, the kernel has complete control over va->pa mapping
    - Allows for many interesting OS features/tricks

- RISC-V maps 4-KB "pages"
  - and aligned -- start on 4 KB boundaries
  - 4 KB = 12 bits
  - the RISC-V used in xv6 has 64-bit for addresses
  - thus page table index is top 64-12 = 52 bits of VA
    - except that the top 25 of the top 52 are unused
      - no RISC-V has that much memory now
      - can grow in future
    - so, index is 27 bits.

- MMU translation
  - see Figure 3.1 of book
  - ![Fig 3.1](https://picgo-1252947055.cos.ap-guangzhou.myqcloud.com/image-20230429014612536.png)
  - use index bits of VA to find a page table entry (PTE)
  - construct physical address using PPN from PTE + offset of VA
  
- what is in PTE?
  - each PTE is 64 bits, but only 54 are used
  - top 44 bits of PTE are top bits of physical address
    - "physical page number"
  - low 10 bits of PTE flags
    - Present, Writeable, &c
  - note: size virtual addresses != size physical addresses

- where is the page table stored?
  - in RAM -- MMU loads (and stores) PTEs
  - o/s can read/write PTEs
    - read/write memory location corresponding to PTEs   

- would it be reasonable for page table to just be an array of PTEs?
  - how big is it?
  - 2^27 is roughly 134 million
  - 64 bits per entry
  - 134*8 MB for a full page table
    - wasting roughly 1GB per page table
    - one page table per address space
    - one address space per application
  - would waste lots of memory for small programs!
    - you only need mappings for a few hundred pages
    - so the rest of the million entries would be there but not needed

- RISC-V 64 uses a "three-level page table" to save space
  - see figure 3.2 from book
  - ![Fig 3.2](https://picgo-1252947055.cos.ap-guangzhou.myqcloud.com/image-20230429014631423.png)
  - page directory page (PD)
    - PD has 512 PTEs
    - PTEs point to another PD or is a leaf
    - so 512*512*512 PTEs in total
  - PD entries can be invalid
    - those PTE pages need not exist
    - so a page table for a small address space can be small
  
- how does the mmu know where the page table is located in RAM?
  - satp holds phys address of top PD
  - pages can be anywhere in RAM -- need not be contiguous
  - rewrite satp when switching to another address space/application

- how does RISC-V paging hardware translate a va?
  - need to find the right PTE
  - satp register points to PA of top/L2 PD
  - top 9 bits index L2 PD to get PA of L1 PD
  - next 9 bits index L1 PD to get PA of L0 PD
  - next 9 bits index L0 PD to get PA of PTE
  - PPN from PTE + low-12 from VA

- flags in PTE
  - V, R, W, X, U
  - xv6 uses all of them

- what if V bit not set? or store and W bit not set?
  - "page fault"
  - forces transfer to kernel
    - trap.c in xv6 source
  - kernel can just produce error, kill process
    - in xv6: "usertrap(): unexpected scause ... pid=... sepc=... stval=..."
  - or kernel can install a PTE, resume the process
    - e.g. after loading the page of memory from disk

- indirection allows paging h/w to solve many problems
  - e.g. phys memory doesn't have to be contiguous
    - avoids fragmentation
  - e.g. lazy allocation (a lab)
  - e.g. copy-on-write fork (another lab)
  - many more techniques
  - topic of next lecture
  
- Q: why use virtual memory in kernel?
  - it is clearly good to have page tables for user processes
  - but why have a page table for the kernel?
    - could the kernel run with using only physical addresses?
  - top-level answer: yes
    - most standard kernels do use virtual addresses
  - why do standard kernels do so?
    - some reasons are lame, some are better, none are fundamental
    - the hardware makes it difficult to turn it off
	  - e.g. on entering a system call, one would have to disable VM
    - the kernel itself can benefit from virtual addresses
      - mark text pages X, but data not (helps tracking down bugs)
      - unmap a page below kernel stack (helps tracking down bugs)
      - map a page both in user and kernel (helps user/kernel transition)

## Virtual memory in xv6

- kernel page table 
  - See figure 3.3 of book
  - ![Fig 3.3](https://picgo-1252947055.cos.ap-guangzhou.myqcloud.com/image-20230429003224937.png)
  - simple maping mostly
    - map virtual to physical one-on-one
  - note double-mapping of trampoline
  - note permissions
  - why map devices?
  
- each process has its own address space
  - and its own page table
  - see figure 3.4 of book
  - ![Fig 3.4](https://picgo-1252947055.cos.ap-guangzhou.myqcloud.com/image-20230429003252688.png)
    - note: trampoline and trapframe aren't writable by user process
  - kernel switches page tables (i.e. sets satp) when switching processes
  
- Q: why this address space arrangement?
  - user virtual addresses start at zero
    - of course user va 0 maps to different pa for each process
  - 16,777,216 GB for user heap to grow contiguously
    - but needn't have contiguous phys mem -- no fragmentation problem
  - both kernel and user map trampoline and trapframe page
    - eases transition user -> kernel and back
    - kernel doesn't map user applications
  - not easy for kernel to r/w user memory
    - need translate user virtual address to kernel virtual address
    - good for isolation (see spectre attacks)
  - easy for kernel to r/w physical memory
    - pa x mapped at va x

- Q: does the kernel have to map all of phys mem into its virtual address space?

## Code walk through

- setup of kernel address space 
  
  `kvmmap()`
  - Q: what is address 0x10000000 (256M)
  - Q: how much address space does 1 L2 entry cover? (1G)
  - Q: how much address space does 1 L1 entry cover? (2MB)
  - Q: how much address space does 1 L0 entry cover? (4096)
  - print kernel page table
  - Q: what is size of address space? (512G)
  - Q: how much memory is used to represent it after 1rst kvmmap()? (3 pages)
  - Q: how many entries is CLINT? (16 pages)
  - Q: how many entries is PLIC? (1024 pages, two level 1 PDs)
  - Q: how many pages is kernel text (8 pages)
  - Q: how many pages is kernel total (128M = 64 * 2MB)
  - Q: Is trampoline mapped twice? (yes, last entry and direct-mapped, entry [2, 3, 7])
  `kvminithart()`
  - Q: after executing `w_satp()` why will the next instruction be sfence_vma()?
	
- `mappages()` in vm.c
  - arguments are top PD, va, size, pa, perm
  - adds mappings from a range of va's to corresponding pa's
  - rounds b/c some uses pass in non-page-aligned addresses
  - for each page-aligned address in the range
    - call walkpgdir to find address of PTE
      - need the PTE's address (not just content) b/c we want to modify
    - put the desired pa into the PTE
    - mark PTE as valid w/ PTE_P

- `walk()` in vm.c
  - mimics how the paging h/w finds the PTE for an address
  - PX extracts the 9 bits at Level level
  - `&pagetable[PX(level, va)]` is the address of the relevant PTE
  - if PTE_V
    - the relevant page-table page already exists
    - PTE2PA extracts the PPN from the PDE
  - if not PTE_V
    - alloc a page-table page
    - fill in pte with PPN (using PA2PTE)
  - now the PTE we want is in the page-table page

- `procinit()` in proc.c
  - alloc a page for each kernel stack with a guard page

- setup user address space
  - `allocproc()`: allocates empty top-level page table
  - `fork()`: `uvmcopy()`
  - `exec()`: replace proc's page table with a new one
    - uvmalloc
    - loadseg
  - print user page table for sh
  - Q: what is entry 2? 

- a process calls `sbrk(n)` to ask for n more bytes of heap memory
  - user/umalloc.c calls `sbrk()` to get memory for the allocator
  - each process has a size
    - kernel adds new memory at process's end, increases size
  - `sbrk()` allocates physical memory (RAM)
  - maps it into the process's page table
  - returns the starting address of the new memory

- `growproc()` in proc.c
  - proc->sz is the process's current size
  - `uvmalloc()` does most of the work
  - when switching to user space satp will be loaded with updated page table

- `uvmalloc()` in vm.c
  - why PGROUNDUP?
  - arguments to `mappages()`...

References:
recursive page tables: https://pdos.csail.mit.edu/6.828/2018/lec/l-josmem.html

# 6.1810 2022 Lecture 5: RISC-V calling convention, stack frames, and gdb

- C code is compiled to machine instructions.
  - How does the machine work at a lower level?
  - How does this translation work?
  - How to interact between C and asm
  - Why this matters: sometimes need to write code not expressible in C
    - And you need this for the syscall lab!

- RISC-V abstract machine
  - No C-like control flow, no concept of variables, types ...
  - Base ISA: Program counter, 32 general-purpose registers (x0--x31)

```txt
reg    | name  | saver  | description
-------+-------+--------+------------
x0     | zero  |        | hardwired zero
x1     | ra    | caller | return address
x2     | sp    | callee | stack pointer
x3     | gp    |        | global pointer
x4     | tp    |        | thread pointer
x5-7   | t0-2  | caller | temporary registers
x8     | s0/fp | callee | saved register / frame pointer
x9     | s1    | callee | saved register
x10-11 | a0-1  | caller | function arguments / return values
x12-17 | a2-7  | caller | function arguments
x18-27 | s2-11 | callee | saved registers
x28-31 | t3-6  | caller | temporary registers
pc     |       |        | program counter
```
Running example: sum_to(n)
```c
  int sum_to(int n) {
    int acc = 0;
    for (int i = 0; i <= n; i++) {
      acc += i;
    }
    return acc;
  }
```
- What does this look like in assembly code?
```riscv
  # sum_to(n)
  # expects argument in a0
  # returns result in a0
  sum_to:
    mv t0, a0          # t0 <- a0
    li a0, 0           # a0 <- 0
  loop:
    add a0, a0, t0     # a0 <- a0 + t0
    addi t0, t0, -1    # t0 <- t0 - 1
    bnez t0, loop      # if t0 != 0: pc <- loop
    ret
```
- Limited abstractions
  - No typed, positional arguments
  - No local variables
  - Only registers

- Machine doesn't even see assembly code
  - Sees binary encoding of machine instructions
    - Each instruction: 16 bits or 32 bits
  - E.g. `mv t0, a0` is encoded as 0x82aa
  - Not quite 1-to-1 encoding from asm, but close

- How would another function call sum_to?
```riscv
  main:
    li a0, 10          # a0 <- 10
    call sum_to
```
- What are the semantics of call?
```riscv
  call label :=
    ra <- pc + 4       ; ra <- address of next instruction
    pc <- label        ; jump to label
```
- Machine doesn't understand labels
  - Translated to either pc-relative or absolute jumps

- What are the semantics of return?
```riscv
  ret :=
    pc <- ra
```

- Let's try it out: demo1.S
```shell
  (gdb) file user/_demo1
  (gdb) break main
  (gdb) continue
  Why does it stop before running demo1?
  (gdb) layout split
  (gdb) stepi
  (gdb) info registers
  (gdb) p $a0
  (gdb) advance 18
  (gdb) si
  (gdb) p $a0
```
- What if we wanted a function calling another function?
```riscv
  # sum_then_double(n)
  # expects argument in a0
  # returns result in a0
  sum_then_double:
    call sum_to
    li t0, 2           # t0 <- 2
    mul a0, a0, t0     # a0 <- a0 * t0
    ret

  main:
    li a0, 10
    call sum_then_double
```
- Let's try it out: demo2.S
  - We get stuck in an infinite loop
  - Why: overwrote return address (ra)

- How to fix: save ra somewhere
  - In another register? Won't work, just defers problem.
  - Solution: save on stack
```riscv
  sum_then_double:
    addi sp, sp, 16    # function prologue:
    sd ra, 0(sp)       # make space on stack, save registers
    call sum_to
    li t0, 2
    mul a0, a0, t0
    ld ra, 0(sp)       # function epilogue:
    addi sp, sp, -16   # restore registers, restore stack pointer
    ret
```
- Let's try it out: demo3.S
```shell
  (gdb) ...
  (gdb) nexti
```
- So far, our functions coordinated with each other
  - This worked because we were writing all the code involved
  - Could have written it any other way
    - E.g. passing arguments in t2, getting return value in t3

- Conventions surrounding this: "calling convention"
  - How are arguments passed?
    - a0, a1, ..., a7, rest on stack
  - How are values returned?
    - a0, a1
  - Who saves registers?
    - Designated as caller or callee saved
    - Could ra be a callee-saved register?
  - Our assembly code should follow this convention
  - C code generated by GCC follows this convention
  - This means that everyone's code can interop, incl C/asm interop
  - Read: demo4.c / demo4.asm
    - Can see function prologue, body, epilogue
    - Why doesn't it save ra? Leaf function, not needed
    - What is going on with s0/fp?
      - =We compiled with -fno-omit-frame-pointer

## Stack
```txt
                   .
                   .
      +->          .
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
      +-> |       ...       |   |
      |   +-----------------+   |
      |   | return address  |   |
      |   |   previous fp ------+
      |   | saved registers |
      |   | local variables |
      |   |       ...       | <-+
      |   +-----------------+   |
      |   | return address  |   |
      +------ previous fp   |   |
          | saved registers |   |
          | local variables |   |
  $fp --> |       ...       |   |
          +-----------------+   |
          | return address  |   |
          |   previous fp ------+
          | saved registers |
  $sp --> | local variables |
          +-----------------+
```
- Demo program: demo5.c
```shell
  (gdb) break g
  (gdb) si
  (gdb) si
  (gdb) si
  (gdb) si
  (gdb) p $sp
  (gdb) p $fp
  (gdb) x/g $fp-16
  (gdb) x/g 0x0000000000002fd0-16
```
- Stack diagram:
```txt
          0x2fe0 |
          0x2fd8 | <garbage ra>       \
          0x2fd0 | <garbage fp>       / stack frame for main
          0x2fc8 | ra into main       \
  $fp --> 0x2fc0 | 0x0000000000002fe0 / stack frame for f
          0x2fb8 | ra into f          \
  $sp --> 0x2fb0 | 0x0000000000002fd0 / stack frame for g
```
- GDB can automate this reasoning for us
  - Plus, it can use debug info to reason about leaf functions, etc.
```shell
  (gdb) backtrace
  (gdb) info frame
  (gdb) frame 1
  (gdb) info frame
  (gdb) frame 2
  (gdb) info frame
```
- Calling C from asm / calling asm from C
  - Follow calling convention and everything will work out
  - Write function prototype so C knows how to call assembly
  - Demo: demo6.c / demo6_asm.S
    - Why do we use s0/s1 instead of e.g. t0/t1?
    ```shell
    (gdb) b sum_squares_to
    (gdb) si ...
    (gdb) x/4g $sp
    (gdb) si ...
    ```

## Inline assembly

- Structs
  - C struct layout rules
    - Why: misaligned load/store can be slow or unsupported (platform-dependent)
  - `__attribute__((packed))`
  - How to access and manipulate C structs from assembly?
    - Generally passed by reference
    - Need to know struct layout
    - Demo: demo7.c / demo7_asm.S

- Debugging
  - examine: inspect memory contents
    ```shell
        x/nfu addr
        n: count
        f: format
        u: unit size
    ```
  - step/next/finish
    - step: next line of C code
    - next: next line of C code, skipping over function calls
    - finish: continue executing until end of current function call
  - stepi/nexti
    - stepi: next assembly instruction
    - nexti: next assembly instruction, skipping over function calls
  - layout next
    - steps through layouts
  - conditional breakpoints
    - break, only when a condition holds (e.g. variable has a certain value)
  - watchpoints
    - break when a memory location changes value
  - GDB is a very powerful tool
    - Read the manual for more!
    - But you probably don't need all the fancy features for this class

## References
  - RISC-V ISA specification: https://riscv.org/specifications/
    - Contains detailed information
  - RISC-V ISA Reference: https://rv8.io/isa
    - Overview of instructions
  - RISC-V assembly language reference: https://rv8.io/asm
    - Overview of directives, pseudo-instructions, and more