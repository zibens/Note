这是一个教学使用的操作系统 ， 在这里主要讨论RISC-V架构 。

所使用的RISC-V版本是64位处理器 ,它是一个多核操作系统。

# Overview



- 进程管理功能 ： 这些进程运行在自己的虚拟地址空间中，因此每个地址空间都有页表来支持虚拟地址
- 支持文件，类似于Unix的文件系统 ： 目录，层次结构
- 管道
- 定时器中断 ： 因此支持多任务处理
- 21个系统调用

User Programs : 

> ​	sh  , cat  ,echo , grep , kill , ln , ls , mkdir , rm ,wc

**It's A True Unix!**



# General Features

Xv6设计用于运行在拥有多个核心但共享同一主存范围的系统上。
**核心** 在这里`CPU = CORE = HART` 通常情况下，一个核心会执行一个硬件线程，但对于某些高性能核心，你可能会有两个硬件线程

**主存** main memory (RAM) is shared and it's just hardwired in with 128 Mbytes

**DEVICES** 

- UART : 是处理串行通信的设备  这代表通用异步收发传输器， 这是提供打印和从键盘读取输入的通信通道设备
- DISK :
- TIMER INTERRUPTS  UART和DISK在所有核心之间共享，但每个核心有自己的定时器

**MEMORY MANAGERMENT**

- page size = 4096 bytes

- single freelist : 内存分配，从一个空闲列表中进行。这个列表包含未使用的页面，它是一个简单的链表，每当内核需要更多内存时，他就从空闲页面中分配页面（==用户分配和内核分配一样吗==）

- page table : three levels        one table for per process  + one table for the kernel

- Scheduler

  Round - Robin  循环调度器

  每个进程被分配一个时间片，然后它回到就绪队列中进入休眠状态

- Boot sequence 启动序列(==不太懂==)

  Qemu 模拟 RISC-V 处理器并运行内核
  从你主机上的某个可执行文件加载内核，并将其放入内存。 是放入它所模拟的RISC处理器固定物理内存中，并将内核代码放置在一个固定位置（`0x80000000`）
  Xv6中没有 bootloader/bootblock/bio

- Locking

  Spin locks 
  sleep()   wakeup()

### User Address Space

Xv6与Linux不一样
`trapframe` : 陷阱帧，保存用户代码的寄存器
每个虚拟地址空间都有自己的陷阱帧页，它们都映射到相同的位置，但对应不同的物理内存页，因此每个进程有自己的陷阱帧
而`trampoline page`由所有进程共享，因此完全相同的物理内存页被映射到所有虚拟地址空间的相同位置

## RISC-V Virtual addresses

xv6 uses "sv39" architecture , 支持三级页表
Virtual addr size = 39bits : $2^{39} = 512GB$ = `0x8000000000`
Xv6 uses only 38 bits for virtual addresses $2^{38} = 256GB$ =`0x4000000000` = MAXVA
所以我们的地址范围是 `0` 到 `0x3FFFFFFFFF`

## Startup + Organization

**这里主要介绍主函数 简要讲解启动过程**

**xv6的文件组织**

- kernel

  `.c`  `.h`  `.S`

- user

  `sh  cat  `

- makefile , readme , license

Xv6系统运行在多核计算机上，启动时，每个核心会同时开始执行。

这是一个共享内存系统，所有核心会共享完全相同的内存，它们都将执行相同的代码，这段代码位于一个名为`entry.S` , 这段代码随后会将控制权转移到一个名为`start.c`的C函数中(Machine mode) , 然后会将控制权转移给`main.c`(Supervisor mode)

(**可以到`源码.md`中查看源码**)

```
RISC-V
		bit		bytes
char     8		  1
short    12       2
int      32       4
long     64       8  
```



# Spinlocks

```c
struct spinlock{
    uint looked;
    //For debugging
    char * name;
    struct cpu * cpu;
}
```

自旋锁处理所之外，还有一个指向字符串的指针，可用于调试，还有一个名为CPU的字段，每个核心都有一个与之关联的结构，称为CPU结构

自旋锁的关键功能是释放和获取`acquire(ptr) release(ptr) initlock(ptr) holding(ptr)`

要获取锁，通常我们只需要将字段设置为1。
RISC-V使用硬件来进行原子操作

自旋锁不应该长时间持有。 设想一下 ： 一个核心持有锁，另一个核心想要获取锁。 那么`acquire`函数将会在一个紧密的循环中等待自旋，等待锁被释放。 因此作为一般经验法则，你应该总是计划在获取锁后尽快释放。xv6中，我们有`sleep  wakeup`可以在需要长时间持有锁的情况下使用。

**锁用于保护共享数据**
xv6book指出 锁实际上保护的是约束条件

**Example**
Console Input (from keyboard) , 每当用户在键盘上输入时，中断处理程序就会被唤醒并将一个字符添加到共享缓冲区中。因此数据从一端进入，另一个线程从缓冲区中读取数据。当它需要一个字符时，它只需要访问这个共享数据结构。我们需要对它进行保护。 使用自旋锁

```c
//Keyboard
acquire(lk);
add to buffer ;
release(lk)
```

```c
//T
acquire(lk);
remove a char;
release(lk);
```

**Interrupt handler code**
当有人在键盘上输入字符时，处理程序会被调用，中断。 `handler`从陷阱处理开始，这会禁用中断

TRAP -> Interrupt Disabled 
..........(做一些事情 ， 读数据)
SRET ->Interrupt Re-enabled

你可以想象一种情况，如果T恰好是这个获取锁的线程，然后再错误的时间点，键盘输入引发中断，处理程序被调用，处理程序第一件事就是尝试获取那个锁 ， 然后此时T已经持有这个锁 ， 那么会造成**死锁**

- 我们解决Deadlock的一种方法是 ：**在获取函数中禁用中断  ， 然后在释放函数中重新启用。**
  这还有一个额外的额好处 ： 我们不希望自旋锁太长时间，因此通过禁用中断，我们防止了在持有锁期间发生时间片轮转，并阻止了中断处理程序发生

我们在`struct CPU`中定义了一个`noff`字段，获取函数将增加计数器然后禁用中断 ， 释放函数减少计数器，**只有计数器为0时，才重新启用中断**

- **noff** : Each Core has its own counter.
- **intena** : We must remember the interrupt status before we started.(==没看懂== : ==最好结合着代码看，这里就是在noff=0的时候 来记录最开始这个数据结构的中断状态，有可能它是允许中断的，有可能本来就不允许，最后每次关中断noff=0的时候恢复这个状态==)

# kalloc , Mem Management

xv6所有的内存管理都是基于4k的块，称之为页。这些页被维护在一个空闲列表中

(详细的看`yuanma`)

# Syscalls from Userland

`kill.c`

系统调用函数通过脚本生成汇编语言

```assembly
li a7 ,SYS_(open / 或其他函数)
ecall
ret
```

`ecall`将结束用户模式下的执行，切换到内核模式，内核将执行一些代码，这些代码将完成打开文件所需的所有操作

`init.S`
`就是exec("\init" , argv)`
简单调用确切的系统调用，传递一个指向文件名的指针和一个指向数组的指针 , 数组中包含两个元素



# RiscV Architecture

Risc-V指令集架构有32个通用寄存器和一个程序计数器 都是64位的

| Zero    | 被硬连线成0，所以在进程间上下文切换时不需要保存他            |
| ------- | ------------------------------------------------------------ |
| ra      | 返回地址Return Address(Risc-v返回时返回地址会保存在这里而不是压入栈中) |
| sp      | 栈指针Stack Pointer                                          |
| tp      | 线程指针Thread Pointer（包含核心编号，即硬件线程的核心ID）   |
| gp      | 全局指针Global Pointer（它由编译器使用，并且会被设置后不再改变） |
| a0 ~ a7 | Function Args/working Regs(它们用于向函数传递参数 A0用于存储返回值) |
| t0~t6   | Temp / Working Regs(A和T寄存器可以在函数中自由使用，完成函数需要执行的任何操作) |
| s0~s11  | 被调用者保存寄存器Callee-Saved（调用者会假设它调用的任何函数都不会修改这些寄存器） |
| PC      | P                                                            |

**被调用者保存寄存器** ：如果一个函数想要使用这些寄存器中的任何一个，那么该函数在使用之前必须先保存它们。 因此它们通常会被压入栈中 ， 并且该函数在返回之前必须恢复这些寄存器。

这是用户模式线程的全部状态。 用户代码无法访问寄存器，因此在用户代码中，状态寄存器是不可见的。

在每次上下文切换时，即当我们结束一个进程的时间片并准备开始另一个进程的时间片时。内核需要保存前一个线程的状态，**内核会将前一个进程的寄存器状态保存在某个地方**，然后再下一个时间片开始之前内核需要加载构成**下一个进程状态的寄存器**。

每个核心都有一组自己的寄存器，并且在每个时刻每个核心都恰好运行在一种模式下。

> Machine mode

机器模式是最高且最强大的，拥有最大的权限。 在核心启动或重置后，它会进入机器模式。
XV6：This mode is not used very much.
在启动时，有一些代码以机器模式运行，进行一些初始化
另一个需要机器模式的原因是处理定时器中断

> Supervisor mode

All kernel code runs in this mode. 
Some instructions are PRIVILEGED.(only allowed in M and S)

> User Mode

All user code runs in this mode.
Privileged instructions cause trap + abort. （用户尝试执行特权操作，将会trap，内核将终止该进程）



**Control and status registers(SCRs)**

Up to 4096 CSRs
xv6:Only 19 are important.
`Privileged Instructions`

- csrr a0 , sstatus  (Read指令 将cs寄存器中的值移动到a0寄存器中)
- csrw sstatus , a0(Write指令 将a0中的值写入这个控制寄存器)
- csrrw a0 , mscratch , a0 (swap Atomic)

| Machine  | Supervisor |                                                              |
| -------- | ---------- | ------------------------------------------------------------ |
| mhartid  |            | Core(Hart) ID                                                |
| mstatus  | sstatus    | status Register                                              |
| mtvec    | stvec      | trap vector / handler addr                                   |
| mepc     | sepc       | Previous PC(当陷阱发生时，前一个pc保存在)                    |
|          | scause     | Trap cause code( indicates the type of the page fault)       |
| mscratch | sscratch   | 工作寄存器可以被我们的陷阱处理程序使用                       |
|          | satp       | Addr Translation Ptr(指向页表的指针)                         |
| mie      | sie        | 允许我们选择性地启用中断                                     |
|          | stval      | Bad address or instruction（contains the address that couldn’t） |

一些尽在机器模式下使用和访问，而其他以S开头的寄存器则在机器和监督模式下都可访问。

(==通用寄存器和状态寄存器有什么区别？==  ==有些寄存器忘了去看xv6book第四章==)

**一些术语**
我们有异常和中断，这两者都属于陷阱的范畴。 陷阱处理程序来处理异常或中断。

**异常是同步的**，这意味着它们是**由某些指令引起的**，某些指令被执行，并且该指令引发了异常。系统调用指令，在RISC-V被命名为ecall。 有`syscall instruction , Program Error`

**中断是异步的**，它们来自于**当前指令之外的某个地方**。一个典型的例子就是**定时器中断**，另一个例子是**当设备发出中断时** 所以我们有两个设备，我们有异步收发传输单元，即串行通信设备，还有磁盘。 这些设备可以在任何时候发出中断。
`Timer` Only in Machine mode . 或者说它的handler是在machine mode

`Software Interrupt` 当定时器中断发生时，处理程序在机器模式下执行，它需要通知监管程序代码。所以它在监管级别引发了一个所谓的软件中断



**pmpcfg  pmpaddr**

有一个物理内存保护系统。它不在xv6内核中使用，但我们仍需处理它，因为它存在。
基本上，它将能够**限制在管理模式或用户模式下运行的任何代码** 对物理内存的访问

（==没听懂（但好像暂时不太重要）==）

# RISC-V page table

`satp` CSR , points to page table

==每个CPU核都只有一个satp寄存器，所以每个cpu在任何情况下都只有一个页表可以使用==

## Kernel Page Table

有多个页表，但只有一个页表指向内核，并且所有处理器共享这个页表
内核中虚拟地址和物理地址是直接映射的

## User Page Table

> RISC-V
>
> ```
> Sv32 - Two Level  32位
> Sv39 — Three Level 64位 （xv6）
> Sv48 — Four Level 64位
> ```

每次从内存中加载 ，取指，存储，都会遍历页表， 遍历页表会涉及多次内存访问，这会导致极大的。

所以每个CPU都会有一个 `Translation Lookaside Buffers(TLB)`
这些寄存器基本上会作为最近页表条目的缓存 ， 对程序员来说都是不可见的

每当更新satp寄存器时，我们需要清空所有的TLB。 RISC-V指令集中有一个`sfence.vma` , 在kernel中有一个函数是`sfence_vma`实际就是使用这个



# RiscV Trap Process

**Traps**

- Exception

  Syscall and   Error

- Interrupt

Example

- System Call

  "ecall"

- Program Error

  Illegal Instruction , Alignment Error

- Device Interrupt

  User Mode                                        

  Supervisor mode  

  **Handler**,running in Supervisor mode

`stvec`是一个控制和状态寄存器 ， 它包含**指向处理程序的指针**
`Kernelvec` 处理在监管模式下执行时发生的trap
`Uservec`用户模式

**sstatus寄存器** 
有一个位控制是否启用中断 `SIE`
先前的中断位 `Spie`
记住陷阱发生时我们处于哪种模式`SPP`

**陷阱发生时会发生什么**

首先确定中断是否被禁用 ， 如果中断被禁用了，那么就先保留trap ， 直到中断打开

如果是异常，无论中断是否被禁用，都会立即处理

当陷阱被处理时，**硬件会先做一些操作**

1.  sepc <- pc  PC指针的副本保存在SEPC的控制和状态寄存器中
2. PC <- stvec  stvec寄存器中的值复制到pc指针中
3.  scause  <-  ....   （1 =software[TIMER]   8 =System call  0 =External Device）
4. stval <- additional info (eg : Bad Instruction)
5. sstatus.SPP <- Previous Mode(硬件立即保存之前的状态 -=user ，1 =supervisor)
6. sstauts.SPIE <- sstatus.SIE (保存中断启用位的先前值)
7. sstatus.SIE <- 禁止中断（如果尚未处于管理模式，将禁用中断并切换到supervisor）



最后使用名为 sret的RISC指令用于从管理模式返回

```
sstatus.SIE <- sstatus.SPIE
mode <- sstatus.SPP
pc <- sepc
```



**上面是管理模式下的， 下面介绍一下机器模式下的trap**

`mstatus寄存器`与sstatus类似
只需要处理时钟中断

所以中断时启用的



# Context Switching

从用户模式切换到内核模式是由trap引起的 ， 可能时一个中断，比如来自请求关注的设备

大概介绍了一下上下文切换，后面又详细的





# Memory Layout

trap先在硬件中处理 切换状态 ，禁用中断等
`uservec`将用户进程的寄存器保存在trapframe中 ， 加载一些内核寄存器比如内核栈指针寄存器，TP寄存器包含核心编号。 最后切换到内核的地址空间

`usertrap`

`usertrapret`当我们准备返回到用户代码时，调用这个 ， 这里将会调用一个`userret`

`userret` 恢复satp寄存器 ， 恢复用户的寄存器
sret硬件指令将切换到用户模式 ， 最后启用中断



## Trampoline page

包含代码 ， 特别是它包含了uservec和userret函数。
它被映射到所有与虚拟地址空间完全相同的地址，即顶部的页面

## Trapframe

每个进程有它自己的这样一个页面。其中包含数据



所有的trapoline page都被映射到同一个物理页，但trapframe页面，每个都映射到不同的物理页面，因此它们不共享

内核只有一个页表和一个虚拟地址空间，所有核心共享这个空间

与64个用户进程关联的每个线程都将拥有自己的栈。因此当用户模式切换到内核模式下执行时，它将需要访问其内核栈

代码 ： memlayout.h



# Link

`kernel.ld`是一个链接器脚本文件，用于向链接器发出指令，告诉它如何将内容放入内存。

首先，先讨论内核如何使用内存。

链接器的工作是将所有目标文件中的 数据整合到可执行文件中。

接下来来详细看`entry.S`中的代码

**所以机器模式都干了些什么？**

- 初始化硬件、定时器、I/O设备等
- 设置异常向量和中断处理 ， 比如定时器中断
- 切换到更高层的内核模式
- 处理硬件中断和异常
- 系统调用的实现
- 调度和上下文切换

# Trap Handing

硬件会执行一些操作 ： 关中断 ， 切换到管理员模式 ， pc保存到sepc ， 陷阱信息到Scause寄存器

**来到`uservec` ：** 

- 保存通用寄存器和程序计数器到trapframe中
- Restore kernel's sp , tp
- satp <- kernel page table
- sepc <- pc
- jump to usertrap

**usertrap:**

首先 , stvec <-  kernel's Trap vector

有一个if语句，用来判断具体发生了什么,通过`scause`寄存器的数字

- Exception : print() , exit()

- Device :

  if killed  exit() 

- syscall : 
  要打开中断

- Timer :
  yield

**usertrapret**

关中断 , ints <- DISABLED
stvec <- uservec
save sp , tp
sepc <- saved PC
jump

**userret**

这段代码在`trampline page`

satp <- users page table

restore user's regs

sstatus : spp = "u mode"  spie = enabled

sret

**介绍一些数据结构**

- CPU和CPUs数组

  数组中每一个元素都是一个结构体，该结构体包含proc指针 ， noff中断计数器 ， intena , context

- proc

  可以进入proc.h查看

  

# Trapframe and trampoline

先来看看uservec中发生了什么（源码`trampoline.S`）

==`ecall` 指令不仅用于系统调用，还可以用于其他类型的 Trap，但在操作系统中，它最常见的用途是实现系统调用。==



# scheduling and sched

`yield` -> `sched` -> `swtch`(将上下文保存在p->context中)并切换到先前保存在`cpu->scheduler`中的调度程序上下文

在`yield`中我们会检查并修改cpu和proc的状态

函数`swtch`为内核线程切换执行保存和恢复操作。它只是保存和恢复寄存器集，称为上下文
它只保存被调用者寄存器。
swtch传入proc的上下文和cpu的上下文







