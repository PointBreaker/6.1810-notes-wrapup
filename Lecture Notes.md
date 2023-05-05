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
  - $2^{27}$ is roughly 134 million
  - 64 bits per entry
  - $134\times8$ MB for a full page table
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
    - so $512\times512\times512$ PTEs in total
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


# 6.1810 2022 Lecture 6: System Call Entry/Exit

- Today: user -> kernel transition
  - system calls, faults, interrupts enter the kernel in the same way
  - lots of careful design and important detail
  - important for isolation and performance

- What needs to happen when a program makes a system call, e.g. write()?
  - [CPU | user/kernel diagram]
  - CPU resources are set up for user execution (not kernel)
    - 32 registers, sp, pc, privilege mode, satp, stvec, sepc, ...
  - what needs to happen?
    - save 32 user registers and pc
    - switch to supervisor mode
    - switch to kernel page table
    - switch to kernel stack
    - jump to kernel C code
  - high-level goals
    - don't let user code interfere with user->kernel transition
      - e.g. don't execute user code in supervisor mode!
    - transparent to user code -- resume without disturbing

- Today we're focusing on the user/kernel transition
  - and ignoring what the system call implemenation does once in the kernel
  - but the sys call impl has to be careful and secure also!

- What does the CPU's "mode" protect?
  - i.e. what does switching mode from user to supervisor allow?
  - supervisor can read/write CPU control registers:
    - satp -- page table physical address
    - stvec -- ecall jumps here in kernel; points to trampoline
    - sepc -- ecall saves user pc here
    - sscratch -- temporary for a0
  - supervisor can use PTEs that have no PTE_U flag
  - but supervisor has no other powers!
    - e.g. can't use addresses that aren't the in page table
    - so kernel has to carefully set things up so it can work

- preview:
    ```text
    write()                        write() returns              User
    ecall
    -----------------------------------------------------------------
                                    sret
    uservec() in trampoline.S      userret() in trampoline.S   Kernel
    usertrap() in trap.c           usertrapret() in trap.c
    syscall() in syscall.c           ^
    sys_write() in sysfile.c      ---|
    ```
- let's watch an xv6 system call entering/leaving the kernel
  - xv6 shell writing its $ prompt
  - sh.c line 137: write(2, "$ ", 2);
  - user/usys.S line 29
    - this is the write() function, still in user space
  - a7 tells the kernel what system call we want -- SYS_write = 16
  - ecall -- triggers the user/kernel transition

- let's start by putting a breakpoint on the ecall
  - user/sh.asm says write()'s ecall is at address 0xe18

    ```shell
    $ make qemu-gdb
    (gdb) b *0xe18
    (gdb) c
    (gdb) delete 1
    (gdb) x/3i 0xe16
    ```

- let's look at the registers
    ```shell
    (gdb) print $pc
    (gdb) info reg
    ```
- `$pc` and `$sp` are at low addresses -- user memory starts at zero
- C on RISC-V puts function arguments in `a0`, `a1`, `a2`, &c
- `write()` arguments: `a0` is `fd`, `a1` is `buf`, `a2` is `n`

    ```shell
    (gdb) x/2c $a1
    ```
- the shell is printing the $ prompt

- what page table is in use?
    ```shell
    (gdb) print/x $satp
    ```
    not very useful
  - qemu: control-a c, info mem
    - there are mappings for seven pages
    - [address space diagram]
    - instructions x2, data, stack guard (no PTE_U), stack
    - then two high mystery pages: trapframe and trampoline
    - there are no mappings for kernel memory, devices, physical mem

- let's execute the ecall

    ```shell
    (gdb) stepi
    
    where are we?
    (gdb) print $pc
            we're executing at a very high virtual address
    (gdb) x/6i 0x3ffffff000
            these are the instructions we're about to execute
            see uservec in kernel/trampoline.S
            it's the start of the kernel's trap handling code
    (gdb) info reg
            the registers hold user values (except $pc)
    qemu: info mem
            we're still using the user page table
            note that $pc is in the trampoline page, the very last page
    ```

> we're executing in the "trampoline" page, which contains the start of
the kernel's trap handling code. ecall doesn't switch page tables, so
these kernel instructions have to exist somewhere in the user page
table. the trampoline page is the answer: the kernel maps it at the
top of every user page table. the kernel sets $stvec to the trampoline
page's virtual address. the trampoline is protected: no `PTE_U` flag.

```shell
(gdb) print/x $stvec
```

- can we tell that we're in supervisor mode?
  - I don't know a way to find the mode directly
  - but observe `$pc` is executing in a page with no PTE_U flag
    - lack of crash implies we are in supervisor mode

- how did we get here?
  - ecall did three things:
    - change mode from user to supervisor
    - save `$pc` in `$sepc`
      ```shell
      (gdb) print/x $sepc
      ```
    - jump to `$stvec` (i.e. set $pc to $stvec)
      - the kernel previously set $stvec, before jumping to user space

- note: ecall lets user code switch to supervisor mode
  - but the kernel immediately gains control via $stvec
  - so the user program itself can't execute as supervisor

- what needs to happen now?
  - save the 32 user register values (for later transparent resume)
  - switch to kernel page table
  - set up stack for kernel C code
  - jump to kernel C code

- why didn't the RISC-V designers have ecall do these things for us?
  - ecall does as little as possible
  - to give O/S designers scope for very fast syscalls / faults / intrs
    = maybe O/S can handle some traps w/o switching page tables
    - maybe we can map BOTH user and kernel into a single page table
       - so no page table switch required
    - maybe some registers do not have to be saved
    - maybe no stack is required for simple system calls

- what are the options at this point for saving user registers?
  - can we just write them somewhere convenient in physical memory?
    - no, even supervisor mode is constrained to use the page table
  - can we first set satp to the kernel page table?
    - supervisor mode is allowed to set satp...
    - but we don't know the address of the kernel page table at this point!
    - and we need a free register to even execute csrw satp, $xx

- two parts to the solution for where to save the 32 user registers:
  - 1) xv6 maps a 2nd kernel page, the trapframe, into every user page table
     - it has space to hold the saved registers
     - the kernel gives each process a different trapframe page
     - the page at 0x3fffffe000 is the trapframe page
     - see struct trapframe in kernel/proc.h
     - (but we still need a register holding the trapframe's address...)
  - 2) RISC-V provides the sscratch register
     - supervisor code can use sscratch for temporary storage
     - user code isn't allowed to use sscratch, so no need to save

- see this at the start of uservec in trapframe.S:
  - csrw sscratch, a0
- then a few instructions to load TRAPFRAME into a0

    ```shell
    (gdb) stepi
    (gdb) stepi
    (gdb) stepi
    (gdb) stepi
    (gdb) print/x $a0
        address of the trapframe
    (gdb> print/x $sscratch
        0x2, the old first argument (fd)
    ```
- now `uservec()` has 32 saves of user registers to the trapframe, via a0
  - so they can be restored later, when the system call returns
  - let's skip them

    ```shell
    (gdb) b *0x3ffffff07e
    (gdb) c
    ```
- now we're setting up to be able to run C code in the kernel
- first a stack
  - previously, kernel put a pointer to top of this process's
    - kernel stack in trapframe
  - look at struct trapframe in kernel/proc.h
  - "ld sp, 8(a0)" fetches the kernel stack pointer
  - remember a0 points to the trapframe
  - at this point the only kernel data the code can
    - get at is the trapframe, so everything has to be loaded from there.
  
    ```shell
    (gdb) stepi
    ```
- retrieve hart ID into tp
    ```shell
    (gdb) stepi
    ```
- we want to jump to the kernel C function usertrap(), which
- the kernel previously saved in the trapframe.
  - "ld t0, 16(a0)" fetches it into t0, we'll use it in a moment,
    - after switching to the kernel page table

    ```shell
    (gdb) stepi
    ```

- load a pointer to the kernel pagetable from the trapframe,
and load it into satp, and issue an sfence to clear the TLB.

    ```shell
    (gdb) stepi
    (gdb) stepi
    (gdb) stepi
    ```

- why isn't there a crash at this point?
  - after all we just switched page tables while executing!
  - answer: the trampoline page is mapped at the same virtual address
    - in the kernel page table as well as every user page table

    ```shell
    (gdb) print $pc
    qemu: info mem
    ```
- with the kernel page table we can now use kernel functions and data

- the jr t0 is a jump to `usertrap()` (using t0 retrieved from trapframe)

    ```shell
    (gdb) print/x $t0
    (gdb) x/4i $t0
    (gdb) stepi
    (gdb) tui enable
    ```

- we're now in `usertrap()` in kernel/trap.c
  - various traps come here, e.g. errors, device interrupts, and system calls
  - `usertrap()` looks in the scause register to see the trap cause
    - see Figure 10.3 on page 102 of The RISC-V Reader
    - ![exception code](https://picgo-1252947055.cos.ap-guangzhou.myqcloud.com/image-20230502001842344.png)
  - scause = 8 is a system call
  
    ```shell
    (gdb) next ... until syscall()
    (gdb) step
    (gdb) next
    ```
  
- now we're in `syscall()` kernel/syscall.c
- `myproc()` uses tp to retrieve current struct proc *
- p->xxx is usually a slot in the current process's struct proc

- `syscall()` retrieves the system call number from saved register a7
  - p->trapframe points to the trapframe, with saved registers
  - p->trapframe->a7 holds 16, SYS_write
  - p->trapframe->a0 holds `write()` first argument -- fd
  - p->trapframe->a1 holds buf
  - p->trapframe->a2 holds n

    ```shell
    (gdb) next ...
    (gdb) print num
    ```

- then dispatches through syscall[num], a table of functions

    ```shell
    (gdb) next ...
    (gdb) step
    ```

- aha, we're in sys_write.
- at this point system call implementations are fairly ordinary C code.
- let's skip to the end, to see how a system call returns to user space.

    ```shell
    (gdb) finish
    ```

- notice that `write()` produced console output (the shell's $ prompt)
- back to `syscall()`
- the p->tf->a0 assignment causes (eventually) a0 to hold the return value
  - the C calling convention on RISC-V puts return values in a0

    ```shell
    (gdb) next
    ```

- back to `usertrap()`

    ```shell
    (gdb) print p->trapframe->a0
    ```

- `write()` returned 2 -- two characters -- $ and space

    ```shell
    (gdb) next
    (gdb) step
    ```

- now we're in `usertrapret()`, which starts the process of returning
  - to the user program

- we need to prepare for the next user->kernel transition
  - stvec = uservec (the trampoline), for the next ecall
  - trapframe satp = kernel page table, for next uservec
  - trapframe sp = top of kernel stack
  - trapframe trap = usertrap
  - trapframe hartid = hartid (in tp)

- at the end, we'll use the RISC-V sret instruction
  - we need to prepare a few registers that sret uses
  - sstatus -- set the "previous mode" bit to user
  - sepc -- the saved user program counter (from trap entry)

- we'll need to switch to the user page table
  - not OK in `usertrapret()`, since it's not mapped in the user page table.
  - need a page that's mapped in both user and kernel page table -- the trampoline.
  - jump to userret in trampoline.S

    ```shell
    (gdb) tui disable
    (gdb) step
    (gdb) x/8i 0x3ffffff09c
    ```

- a0 holds user page table address
- the csrw satp switches to the user address space

    ```shell
    (gdb) stepi
    (gdb) stepi
    (qemu) info mem
    ```

- now 32 loads from the trapframe into registers
  - these restore the user registers
  - let's skip over them

    ```shell
    (gdb) b *0x3ffffff11a
    (gdb) c
    ```

- a0 is restored last, after which we can no longer get at TRAPFRAME

    ```shell
    (gdb) print/x $a0 -- the return value from write()
    ```

- now we're at the sret instruction

    ```shell
    (gdb) print $pc
    (gdb) stepi
    (gdb) print $pc
    ```

- now we're back in the user program (`$pc` = 0xe1c)
  - returning 2 from the `write()` function

    ```shell
    (gdb) print/x $a0
    ```

- and we're done with a system call!

- summary
  - system call entry/exit is far more complex than function call
  - much of the complexity is due to the requirement for isolation
    - and the desire for simple and fast hardware mechanisms
  - a few design questions to ponder:
    - can an evil program abuse the entry mechanism?
    - can you think of ways to make the hardware or software simpler?
    - can you think of ways to make traps faster?


# 6.1810 2022 Lecture 7: Page faults

- plan: cool things you can do with vm
  - Better performance/efficiency
    - e.g., one zero-filled page
    - e.g., copy-on-write fork
  - New features
    - e.g., memory-mapped files
  
- virtual memory: several views
  - primary purpose: isolation
    - each process has its own address space
  - Virtual memory provides a level-of-indirection
    - provides kernel with opportunity to do cool stuff
	- already some examples:
    	- shared trampoline page
    	- guard page
	- but more possible...
	
- Key idea: change page tables on page fault
  - Page fault is a form of a trap (like a system call)
  - Xv6 panics on page fault
    - But you don't have to panic!
  - Instead:
    - update page table instead of panic
	- restart instruction (see userret() from traps lecture)
  - Combination of page faults and updating page table is powerful!
	
- RISC-V page faults
  - 3 of 16 exceptions are related to paging
  - Exceptions cause controlled transfers to kernel
    - See traps lecture
	
- Information we might need at page fault to do something interesting:
  - 1) The virtual address that caused the fault
    - See stval register; page faults set it to the fault address
  - 2) The type of violation that caused the fault
    - See scause register value (instruction, load, and Store page fault)
  - 3) The instruction and mode where the fault occurred
    - User IP: trapframe->epc
    - U/K mode: implicit in usertrap/kerneltrap
  
- lazy/on-demand page allocation
  - `sbrk()` is old fashioned;
    - applications often ask for memory they need
    - for example, the allocate for the largest possible input but
      - an application will typically use less
    - if they ask for much, `sbrk()` could be expensive
    - for example, if all memory is in use, have to wait until
      - kernel has evicted some pages to free up memory
    - sbrk allocates memory that may never be used.
  - moderns OSes allocate memory lazily
    - plan:
      - allocate physical memory when application needs it
      - adjust p->sz on sbrk, but don't allocate
      - when application uses that memory, it will result in page fault
      - on pagefault allocate memory
      - resume at the fault instruction
    - may use less memory
      - if not used, no fault, no allocation
    - spreads the cost of allocation over the page faults instead
    of upfront in `sbrk()`
  - demo
    - modify sysproc.c
      - run ls and panic
      - look at sh.c
    - modify trap.c
    - modify vm.c
      - vmfault, ismapped,  uvmunmap, copyin, copyout, uvmcopy
  - run top

- one zero-filled page (zero fill on demand)
  - applications often have large part of memory that must zero
    - global arrays, etc.
    - the "block starting symbol" (bbs) segment
  - thus, kernel must often fill a page with zeros
  - idea: memset *one* page with zeros
    - map that page copy-on-write when kernel needs zero-filled page
    - on write make copy of page and map it read/write in app address space

- copy-on-write fork
  - observation:
    - xv6 fork copies all pages from parent (see fork())
    - but fork is often immediately followed by exec
  - idea: share address space between parent and child
    - modify fork() to map pages copy-on-write
      - use extra available system bits (RSW) in PTEs
    - on page fault, make copy of page and map it read/write
    - need to refcount physical pages
      - easy in xv6 but can be challenging in real OSes
        - for example, see https://lwn.net/Articles/849638/

- demand paging
  - observation: exec loads the complete file into memory (see exec.c)
    - expensive: takes time to do so (e.g., file is stored on a slow disk)
    - unnecessary: maybe not the whole file will be used
  - idea: load pages from the file on demand
    - allocate page table entries, but mark them on-demand
    - on fault, read the page in from the file and update page table entry
    - need to keep some meta information about where a page is located on disk
      - this information is typically in structure called virtual memory area (VMA)
  - challenge: file larger than physical memory (see next idea)

- use virtual memory larger than physical memory
  - observation: application may need more memory than there is physical memory
  - idea: store less-frequently used parts of the address space on disk
    - page-in and page-out pages of the address address space transparently
  - works when working sets fits in physical memory
    - most popular replacement strategy: least-recently used (LRU)
    - the A(cess) bit in the PTE helps the kernel implementing LRU
  - replacement policies huge topic in OS literature
    - many OS allow applications to influence its decisions (e.g., using
    - madvise and mlock).
  - demo: run top and vmstat
    - on laptop and dialup.athena.mit.edu
    - see VIRT RES MEM SHR columns
  
- memory-mapped files
  - idea: allow access to files using load and store
    - can easily read and writes part of a file
    - e.g., don't have to change offset using lseek system call
  - Unix systems a new system call for m-mapped files:
    - void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
  - kernel page-in pages of a file on demand
    - when memory is full, page-out pages of a file that are not frequently used

- pgfaults for user applications
  - many useful kernel tricks using pgfaults
    - allow user apps also to do such tricks
  - linux: mmap and sigaction
  - https://pdos.csail.mit.edu/6.828/2018/homework/mmap.html
  
- shared virtual memory
  - idea: allow processes on different machines to share virtual memory
    - gives the illusion of physical shared memory, across a network
  - replicate pages that are only read
  - invalidate copies on write

- TLB management
  - CPUs caches paging translation for speed
  - xv6 flushes entire TLB during user/kernel transitions
    - why?
  - RISC-V allows more sophisticated plans
    - PTE_G: global TLB bits
      - what page could use this?
    - ASID numbers
      - TLB entries are tagged with ASID, so kernel can flush selectively
      - SATP takes an ASID number
      - sfence.vma also takes an ASID number
    - Large pages
      - 2MB and 1GB

- Virtual memory is still evolving
  - Recent changes in Linux
    - PKTI to handle meltdown side-channel
      - (https://en.wikipedia.org/wiki/Kernel_page-table_isolation)
    - xv6 basically implements KPTI
  - Somewhat recent changes
    - Support for 5-level page tables (57 address bits!)
    - Support for ASIDs
  - Less recent changes
    - Support for large pages
    - NX (No eXecute) PTE_X flag

- Summary
  - paging plus page tables provide a power layer of indirection
  - You will implement COW and memory-mapped files
  - xv6 is simple, but you have enough information to extrapolate