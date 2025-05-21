# Opreating system interfaces

| **系统调用**                            | **描述**                                                    |
| --------------------------------------- | ----------------------------------------------------------- |
| `int fork()`                            | 创建一个进程，返回子进程的PID                               |
| `int exit(int status)`                  | 终止当前进程，并将状态报告给wait()函数。无返回              |
| `int wait(int *status)`                 | 等待一个子进程退出; 将退出状态存入*status; 返回子进程PID。  |
| `int kill(int pid)`                     | 终止对应PID的进程，返回0，或返回-1表示错误                  |
| `int getpid()`                          | 返回当前进程的PID                                           |
| `int sleep(int n)`                      | 暂停n个时钟节拍                                             |
| `int exec(char *file, char *argv[])`    | 加载一个文件并使用参数执行它; 只有在出错时才返回            |
| `char *sbrk(int n)`                     | 按n 字节增长进程的内存。返回新内存的开始                    |
| `int open(char *file, int flags)`       | 打开一个文件；flags表示read/write；返回一个fd（文件描述符） |
| `int write(int fd, char *buf, int n)`   | 从buf 写n 个字节到文件描述符fd; 返回n                       |
| `int read(int fd, char *buf, int n)`    | 将n 个字节读入buf；返回读取的字节数；如果文件结束，返回0    |
| `int close(int fd)`                     | 释放打开的文件fd                                            |
| `int dup(int fd)`                       | 返回一个新的文件描述符，指向与fd 相同的文件                 |
| `int pipe(int p[])`                     | 创建一个管道，把read/write文件描述符放在p[0]和p[1]中        |
| `int chdir(char *dir)`                  | 改变当前的工作目录                                          |
| `int mkdir(char *dir)`                  | 创建一个新目录                                              |
| `int mknod(char *file, int, int)`       | 创建一个设备文件                                            |
| `int fstat(int fd, struct stat *st)`    | 将打开文件fd的信息放入*st                                   |
| `int stat(char *file, struct stat *st)` | 将指定名称的文件信息放入*st                                 |
| `int link(char *file1, char *file2)`    | 为文件file1创建另一个名称(file2)                            |
| `int unlink(char *file)`                | 删除一个文件                                                |

## Processes and memory

​	An xv6 process consistes of user-space memory(instructions , data , ans stack) and per-process private to the kernel.
xv6的分时进程，透明地在等待执行的进程集合中转换CPU。

​	When a process is not executinig ,xv6 saves its CPU registers , restoring them when it next runs the process. The kernel associates a process identifier ,or  PID ,with each process（内核利用进程id或PID标识每个进程）

​	Fork gives the new process exactly the same memory contents(both instructions and data) as the calling process.(Fork创建了一个新的进程，其内存内容与调用进程（称为父进程）完全相同，称其为子进程。)  

```c
//fork()在父进程中返回子进程的PID
//在子进程中返回0
int pid = fork();
if(pid > 0){
    printf("parent :child =%d\n" , pid);
    pid = wait((int*) 0 ); //这里，(int *) 0 意味着父进程不关心子进程的退出状态。
    printf("child %d is done\n", pid);
}
else if(pid == 0){
    printf("child: exiting\n");
    exit(0);
}else {
    printf("fork error\n");
}
```

​	The exit system call cause the calling process to stop executing and to release resources such as memory and open files.
The wait:

```
wait系统调用返回当前进程的已退出(或已杀死)子进程的PID，并将子进程的退出状态复制到传递给wait的地址；如果调用方的子进程都没有退出，那么wait等待一个子进程退出。如果调用者没有子级，wait立即返回-1。如果父进程不关心子进程的退出状态，它可以传递一个0地址给wait。
```

​	Although the child has the same memory contents as the parent initially , the parent and child are executing whit different memory and different registers : changing a variable in one does not affect the other.

`exec`系统调用使用从文件系统中存储的文件所加载的新内存映像替换调用进程的内存。

```
在操作系统中，exec是一种非常重要的系统调用，它的作用是替换当前进程的地址空间 ，即：用一个新的程序覆盖原有程序的内存。简单来说，exec 使得一个进程能够加载并执行另一个程序，而不需要创建一个新的进程。
```

​	Exec takes two arguments : the name of the file containing the executable and an array of string arguments.For example:

```c
char *argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;
exec("/bin/echo" , argv);
printf("ecex errror\n");
```

​	This fragment replaces the calling program with an instance of the program /bin/echo running with the argument list echo hello. Most programs ignore the first element of the argument array, which is conventionally the name of the program.

**补充 ：** 

```c
char *argv[] = {"echo" , "this" , "is" , "echo" , 0};
exec("echo" , argv);
//输出 this is echo
//argv[0]是程序名，  从echo文件中加载指令 ， 此时实际就是echo程序
```

**fork and exec**

```c
//比如说我们 shell中想运行一个ls ， 但是exec会覆盖掉shell进程，就需要让子进程来执行
int pid , status;
pid = fork();
if(pid == 0){
    char *argv[] = {"echo" , "This" ,"is" , "echo" , 0};
    exec("echo" , argv); //子进程被替换
    printf("exec failed!\n");
    exit(1);
}
else{
    printf("parent waiting\n");
    wait(&status); //只等待一个子进程
    printf("the child exited with status %d\n" , status);
}
exit(0);
```

```c
write(fd ,buf , cnt); //数据将被写入到fd
read(fd , buf , cnt); //fd读到buf中
```



## I/O and file desciptors

​	A process may obtain a file descriptor by opening a file , directory ,device ,or by creating a pipe , or by duplicating an existing descriptor.

​	在内部，xv6内核使用文件描述符作为每个进程表的索引，这样每个进程都有一个从零开始的文件描述符的私有空间。By convention , a process reads from file descriptor 0(standard input) , writes output to file descriptor 1(standard output) , and write error messages to file descriptor 2(standard error).

​	The call `read(fd , buf ,n)`reads at most n bytes from the file descriptor fd ,copies them into buf , and returns the number of bytes read. When there are no bytes to read ,`read`will returns zero to indicate the end of the file.	
​	The call `write(fd , buf ,n)`writes n bytese from buf to the file descriptor fd and returns the number of bytes written. Fewer than n bytes are written only when an error occurs.

以下程序片段（构成程序`cat`的本质）将数据从其标准输入复制到其标准输出。如果发生错误，它将消息写入标准错误：

```C
char buf[512];
int n;
for(;;){
    n = read(0 , buf , sizeof(buf));
    if(n == 0)break;
    if(n < 0){
        fprintf(2 , "read error\n");
        exit(1);
    }
    if(write(1 , buf , n) != n){
        fprintf(2 , "write error\n");
        exit(1);
    }
}
```



​	The close system call releases a file descriptor , making it free for reuse by a future `open` ,`pipe` , or `dup` system call(see below). A newly allocated file  descriptor is always the lowest-number unused descriptor of the current process.

​	File descriptors and `fork` interact to make I/O redirection easy to implement. `Fork` copies the parent's file descriptor table along with its memory , so that the child starts whit exactly the same open files as the parent. The system call `exec` replaces the calling process's memory but preserveese its file table. This behavior allows the shell to implement I/O redirection by forking , reopening chosen file descriptors in the child , and then calling `exec` to run the new program. 下面是shell运行命令`cat < input.txt`的代码的简化版本。

```c
char * argv[2];
argv[0] = "cat";
argv[1] = 0;
if(fork() == 0){
    close(0);
    open("input.txt" , O_RDONLY);
    exec("cat" , argv);
}
```

在子进程关闭文件描述符0之后，`open`保证使用新打开的***input.txt\***：0的文件描述符作为最小的可用文件描述符。`cat`然后执行文件描述符0(标准输入)，但引用的是***input.txt\***。父进程的文件描述符不会被这个序列改变，因为它只修改子进程的描述符。

​	Now it should be clear why it is helpful that `fork` and `exec` are separate calls : between the two , the shell has a chance to redirect the child's I/O whithout disturbing the I/O setup of the main shell.

`dup`系统调用复制一个现有的文件描述符 ， 返回一个引用自同一个底层I/O对象的新文件描述符。两个文件描述符共享一个偏移量，就像fork复制的文件描述符一样。这是另一种将“hello world”写入文件的方法：

```c
fd = dup(1);
write(1 , "hello" , 6);
write(fd , "world\n" , 6);
```

**补充：**

```c
/*
cho hello > out
cat < out
*/
int main(){
    int pid =0;
    pid = fork();
    if(pid == 0){
        close(1);
        open("output.txt" , O_WRONLY | O_CREATE); //先关闭了输出符号，然后将output.txt重定向到 1（输出）中（因为最小原则）
        char *argv[] = {"echo" , "this" , "is" , "redirected" ,"echo" , 0};
        exec("echo" , argv);//这里是把this is redirected 输出重定向到了文件 output.txt中而不是shell上
    }
    
    
 exit(0);   
}
```



## Pipes

A pipe is a small kernel buffer exposed to processes as a pair of file descriptors , one for reading and one for writing.(管道是作为一对文件描述符公开给进程的小型内核缓冲区，一个用于读取，一个用于写入) Wrinting data to one end of the pipe makes that data available
for reading form the other end of the pipe. Pipes provide a way for processes to communicate.

​	The following example code runs the program `wc` with standard input connected to the read end of a pipe.

```c
int p[2];
char *argv[2];
argv[0] = "wc";
argv[1] = 0;

pipe(p);
if(fork() == 0){
    close(0);
    dup(p[0]);//由于文件描述符0已关闭，dup(p[0])将p[0]复制到文件描述符0，即标准输入。最小的
    close(p[0]);//关闭管道的读端，因为已经通过dup或dup2将其复制到标准输入
    close(p[1]);//关闭管道的写端，因为子进程不需要写入数据。
    exec("/bin/wc" , argv);
}
else{
    close(p[0]);
    write(p[1] , "hello world\n" , 12);//将字符串"hello world\n"写入管道的写端。
    close(p[1]);
}
```

程序调用`pipe`，创建一个新的管道，并在数组p中记录读写文件描述符。在`fork`之后，父子进程都有指向管道的文件描述符。子进程调用`close`和`dup`使文件描述符0指向管道的读取端（前面说过优先分配最小的未使用的描述符），然后关闭p中 所存的文件描述符，并调用`exec`运行`wc`。当`wc`从它的标准输入读取时，就是从管道读取。父进程关闭管道的读取端，写入管道，然后关闭写入端。

如果没有可用的数据，则管道上的`read`操作将会进入等待，直到有新数据写入或所有指向写入端的文件描述符都被关闭，在后一种情况下，`read`将返回0，就像到达数据文件的末尾一样。事实上，`read`在新数据不可能到达前会一直阻塞，这是子进程在执行上面的`wc`之前关闭管道的写入端非常重要的一个原因：如果wc的文件描述符之一指向管道的写入端，wc将永远看不到文件的结束。

管道相比临时文件至少有四个优势

- 首先，管道会自动清理自己；在文件重定向时，shell使用完`/tmp/xyz`后必须小心删除
- 其次，管道可以任意传递长的数据流，而文件重定向需要磁盘上足够的空闲空间来存储所有的数据。
- 第三，管道允许并行执行管道阶段，而文件方法要求第一个程序在第二个程序启动之前完成。
- 第四，如果实现进程间通讯，管道的**阻塞**式读写比文件的非阻塞语义更高效。

## File system

​	The xv6 file system provides data files , which contain uninterpreted(未解释的) byte array , and directories , which contain named references to data diles and other directories. The directories form a tree , starting at a special directory called the root . A path like /a/b/c refers to the file or directory named c inside the directory named b inside the directory named a in the root directory / .  Paths  that don't begin with /  are evaluated relative to the calling process's current directory , which can be changed with the chdir system call .(不以`/`开始的路径相对于调用进程的当前工作目录进行计算，当前工作目录可以通过`chdir`系统调用进行更改。)  **Both these code fragments open the same file **

```c
chdir("/a");
chdir("b");
open("c" , O_RDONLY);

open("/a/b/c" , O_RDONLY);
```

​	There are system calls to create new files and directories : `mkdir` creates a new directory , `open` with the `O_CREATE` flag creates a new data file , and `mknod` creates a new device file. **This example illustrates all three:**

```c
mkdir("/dir");
fd =open("/dir/file" , O_CREATE | O_WRONLY);
close(fd);
mknode("/console" , 1 , 1);
```

**文件系统基础概念解释**
在文件系统中， 文件名和文件本身是两个不同的概念。 

- 文件名与inode

  - 文件名
    - 文件名是用户或程序用于表示和访问文件的名称
    - 文件名存在于**目录**中，目录实际上是一个包含多个**目录条目（Directory Entry）**的特殊文件。
    - 每个目录条目包含一个文件名和一个指向文件的**inode**的引用
  - inode（索引节点）
    - inode是文件系统中的一个数据结构，用于存储关于文件的元数据
    - 一个inode包含的信息包括：文件类型，文件大小，文件权限，所有者和所属组 ， 文件内容的位置 ，链接计数

- 链接

  - 一个inode可以有多个文件名（硬链接）指向它。
  - 这意味着同一个底层文件（inode）可以通过不同的路径或名称被访问。
  - 只有当所有指向该inode的链接都被删除后，文件的空间才会被释放。

  **软链接（Symbolic Link）**（在某些文件系统中存在）：

  - 软链接是一个特殊类型的文件，包含另一个文件的路径。
  - 它类似于快捷方式，可以指向不同文件系统中的文件。
  - 与硬链接不同，软链接有自己的inode，并且可以指向目录。

- 目录条目

  - 目录是一个特殊的文件，包含多个目录条目
  - 每个目录条目包含 ： 文件名 ， inode号

- fstat系统调用

  - `fstat` 是一个系统调用，用于从一个打开的文件描述符中检索文件的状态信息。

    它将文件描述符引用的inode中的信息填充到一个 `struct stat` 结构体中。

**文件名和inode的关系**：

- 当你在文件系统中创建一个文件时，比如`/home/user/document.txt`，你实际上创建了一个目录条目，其中包含文件名`document.txt`和指向该文件的inode编号（例如inode 1001）。
- inode 1001包含了`document.txt`的所有元数据和文件内容的存储位置。

**硬链接（Hard Link）**：

- 硬链接就像图书馆中同一本书在不同书架上的多个标签。
- 一个文件可以有多个文件名（硬链接），这些文件名都指向同一个inode编号。例如，`/home/user/document.txt`和`/home/user/docs/document.txt`都可以指向inode 1001。
- 只有当所有指向该inode的硬链接都被删除后，文件的数据才会被实际删除。

**软链接（Symbolic Link）**：

- 软链接类似于图书馆中的指向另一本书的参考卡片。
- 软链接是一个特殊的文件，包含另一个文件的路径。它有自己的inode编号，但其内容是指向目标文件的路径。
- 软链接可以跨文件系统，并且可以指向目录。

**目录遍历和递归**：

- 当程序（如`find`命令）遍历目录时，它会读取目录文件中的所有目录条目，获取每个文件名和对应的inode编号。
- 使用`fstat`系统调用，可以从文件描述符获取inode的信息，如文件类型和大小等。
- 递归遍历允许程序进入子目录，继续查找特定文件名的文件。



# Operating system oranization

​	A key requirement for any operating system is to support several activities at once.

​	The operating system must `time-share` the resources of the computer among these processes.

​	The operating system must also arrange for isolation between the processes.Thus an operating system must fulfill three erquirements : multiplexing , isolation , and interaction.(多路复用 ， 隔离 ， 交互)

## Abstracting phsical resources

​	To achieve strong isolation it's helpful to forbid applications from directly accessing sensitive hardware resources , and instead to abstract the resources into services.

## User mode , supervisor mod , and system calls

​	Strong isolation requires a hard boundary between application and the operating system. If the application makes a mistake , we don't want the operating system to fail or other applications  to fail. Instead , the operating system should be able to clean up the failed application and continue running other applications. To achieve strong isolation , the operating system must arrange that applications cannot modify(or even read) the operating system's data structures and instructions and that applications cannot access other processes's memory.

​	CPUs provide hardware support for strong isolation.

## Kernel organization

​	A key design question is what part of the operating system should run in supervisor mode. One possibility is that the entire operating system resides in the kernel , so that the implementations of all system run in supervisor mode. This organization is called a `monolithic kernel` 
​	In this organization the entire operating system runs with full hardware privilege. This organzition is convenient because the OS designer doesn't have to decide which part of the operating system doesn't need full hardware privilege. Furthermore , it is easier for different parts of the operating system to cooperate.



## Code : xv6 organization

​	 The xv6 kernel source is in the `kernel/`sub-directory. The source is divided   into files , following a rough notion of modularity.图2.2列出了这些文件，模块间的接口都被定义在了 def.h\（**kernel/defs.h\**）。

| **文件**             | **描述**                                    |
| -------------------- | ------------------------------------------- |
| ***bio.c\***         | 文件系统的磁盘块缓存                        |
| ***console.c\***     | 连接到用户的键盘和屏幕                      |
| ***entry.S\***       | 首次启动指令                                |
| ***exec.c\***        | `exec()`系统调用                            |
| ***file.c\***        | 文件描述符支持                              |
| ***fs.c\***          | 文件系统                                    |
| ***kalloc.c\***      | 物理页面分配器                              |
| ***kernelvec.S\***   | 处理来自内核的陷入指令以及计时器中断        |
| ***log.c\***         | 文件系统日志记录以及崩溃修复                |
| ***main.c\***        | 在启动过程中控制其他模块初始化              |
| ***pipe.c\***        | 管道                                        |
| ***plic.c\***        | RISC-V中断控制器                            |
| ***printf.c\***      | 格式化输出到控制台                          |
| ***proc.c\***        | 进程和调度                                  |
| ***sleeplock.c\***   | Locks that yield the CPU                    |
| ***spinlock.c\***    | Locks that don’t yield the CPU.             |
| ***start.c\***       | 早期机器模式启动代码                        |
| ***string.c\***      | 字符串和字节数组库                          |
| ***swtch.c\***       | 线程切换                                    |
| ***syscall.c\***     | Dispatch system calls to handling function. |
| ***sysfile.c\***     | 文件相关的系统调用                          |
| ***sysproc.c\***     | 进程相关的系统调用                          |
| ***trampoline.S\***  | 用于在用户和内核之间切换的汇编代码          |
| ***trap.c\***        | 对陷入指令和中断进行处理并返回的C代码       |
| ***uart.c\***        | 串口控制台设备驱动程序                      |
| ***virtio_disk.c\*** | 磁盘设备驱动程序                            |
| ***vm.c\***          | 管理页表和地址空间                          |







## Process overview

​	The unit of isolation in xv6(as in other Unix operating system) is a `process`.  The kernel must implement the process abstraction with care.
The mechanisms used by the kernel to implement processes include the user/supervisor mode flag , address spaces , and time-slicing of threads.

​	To help enforce isolation , the process abstraction provides the illusion to a program that it has its own private machine.

​	Xv6 uses page tables(which are implemented by hardware) to give each process its own address space. 

​	 Xv6 maintains a separate page table for each process that defines that process's address space. 如图2.3所示，以虚拟内存地址0开始的进程的用户内存地址空间。首先是指令，然后是全局变量，然后是栈区，最后是一个堆区域（用于`malloc`）以供进程根据需要进行扩展。有许多因素限制了进程地址空间的最大范围： RISC-V上的指针有64位宽；硬件在页表中查找虚拟地址时只使用低39位；xv6只使用这39位中的38位。因此，最大地址是2^38-1=0x3fffffffff，即`MAXVA`（定义在***kernel/riscv.h\***:348）。在地址空间的顶部，xv6为`trampoline`（用于在用户和内核之间切换）和映射进程切换到内核的`trapframe`分别保留了一个页面，正如我们将在第4章中解释的那样。
![image-20241230133558373](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20241230133558373.png)



 	The xv6 kernel maintains many pieces of state for each process,which it gathers into a `struct proce`  . A process's most important pieces of kernel state are its page table , its kernel stack , and its run state.

​	 Each process has a thread of execution(or thread for shrot) that executes the process's instructions. A thread can be suspended and later resumed. To switch transparent between processes, the kernel suspends the currently running thread and resumes another process's thread. Much of the state of a thread is stored on the thread's stacks. Each process has two stacks: a user stack and a kernel stack(`p->kstack` ). When the process enters the kernel , the kernel code executes on the process's kernel stack. While a process is in the kernel , its user stack still contains saved data, but isn't actively used.  内核栈是独立的（并且不受用户代码的保护），因此即使一个进程破坏了它的用户栈，内核依然可以正常运行。

​	 A process can make a system call by executing the RISC-V `ecall` instruction.  This instruction raises the hardware privilege level and changes the program counter to a kernel-defined entry point. The code at the entry point switches to a kernel stack and executes the kernel instructions that implement the system call.When the system call completes , the kernel switch back to the user stack and returns to user space by calling the `sret`  instruction , which lowers the hardware privilege level and resumes executing user instructions just after the system call instruction. A process's thread can "block" in the kernel to wait for I/O , and resume where it left off when the I/O has finished.

`p->state`表明进程是已分配、就绪态、运行态、等待I/O中（阻塞态）还是退出。
`p->pagetable`以RISC-V硬件所期望的格式保存进程的页表。当在用户空间执行进程时，Xv6让分页硬件使用进程的`p->pagetable`。一个进程的页表也可以作为已分配给该进程用于存储进程内存的物理页面地址的记录。



## class

### isolation

应用程序和操作系统之间有强隔离

内存隔离：让一个应用程序不会覆盖另一个应用程序的内存

操作系统必须确保任何东西都能正常运行，所以它必须设置一些东西，防止应用程序破坏操作系统。 应用程序不能打破它的隔离。这意味着应用程序和操作系统之间必须有强隔离。 **通常，实现强隔离的方法时硬件支持** ： 一种称为用户内核模式 ， 另一种是页表，虚拟内存。

### kernel and user mode

处理器有两种模式：用户模式， 内核模式。   当运行在内核模式，cpu**可以执行特权指令**，当运行在用户模式，cpu**只能执行非特权指令**
特权指令是引入直接操作硬件的指令(配置页表寄存器，设置禁止时钟中断)

操作系统给每个进程提供自己的页表，通过这种方式，进程只能访问它页表中显示的物理内存。如果操作系统设置每个进程使用**不相交**的物理内存，**那么进程甚至不能访问其他进程的物理内存**，所以这提供了内存的强隔离。



### system call

用户程序访问内核的一种方式
实际上，比如用户空间调用了函数`fork()`并不是直接调用内核中的函数，在xv6中是调用了` ecall`+某个数字 ， 通过ecall进入kernel,然后调用`syscall(数字)`    

### kernel是如何编译的

makefile 选取C文件中的一个，比如proc.c,调用GCC编译器生成文件proc.s , 通过汇编器 ，proc.o（二进制版本）,最后是链接器把所有文件链接在一起，生成kernel	



## 补充

### 启动过程

源代码根目录下的 Makefile 文件就是用来配置代码编译顺序的。

```shell
K=kernel
U=user

OBJS = \
  $K/entry.o \
```

第一行是声明一个字符串K ， 它的值是 kernel
第五行是一个文件路径 kernel/entry.o

答案就在眼前！操作系统的第一行代码对应的文件是 `kernel/entry.o`。
啊哦！糟糕！我们打开 kernel 文件夹，结果发现源代码里根本没有 `kernel/entry.o`。
但是我们看到有一个非常类似的文件 `kernel/entry.S`。`kernel/entry.S` 就是我们要找的第一行代码，而 `kernel/entry.o` 则是编译后的可执行文件。

我们打开这个文件 `entry.S` ，学习一下它。文件开头有这样的注释：`entry.S` 被放在 `0x80000000` 的位置，并且这个位置在 `kernel.ld` 中配置。

第一步，根据`Makefile`的配置，内核源码依次被编译成以`.o` 结尾的[可执行文件](https://zhida.zhihu.com/search?content_id=248102359&content_type=Article&match_order=2&q=可执行文件&zhida_source=entity)；
第二步，根据`kernel.ld`文件的配置，这些可执行文件被堆叠成一个可执行文件。这个可执行文件分为四个部分 `.text`，`.rodata`，`.data`，`bss`。

![image-20250101150714744](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250101150714744.png)



### 从用户态到内核态

RiscV处理器从硬件层面给我们提供了一个指令叫做 `Ecall` 。它做了两件事情：

- 将risc-v中表示处理器权限等级的标志位从0变成1 ， 标志着处理器硬件层面从U-Mode变到了S-Mode。
- 触发一个异常信号，告诉硬件电路，下一条指令不是顺序执行代码，需要要跳转到异常处理[地址寄存器](https://zhida.zhihu.com/search?content_id=250701039&content_type=Article&match_order=1&q=地址寄存器&zhida_source=entity)`stvec`处继续执行代码。写成数学的形式就是从 `pc = pc + 1` 变成 `pc = $stvec` 。

从上面的总结我们可以知道，用户态调用`ecall`最终对应着内核态调用`syscall`。我们来看一下`syscall`的逻辑，一句话总结就是执行 `syscalls[p->trapframe->a7]()` 并把返回结果赋值给 `p->trapframe->a0`。
其中，`syscalls`是一个函数数组，在操作系统里，把这个数组里的函数统称为**系统调用**。

我们来看一下系统调用数组。它本质上就是一个64位的指针数组。

```c
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
};
```

**用户视角看系统调用**

如果你也研究过XV6的代码，你一定会产生过这样的疑惑：[用户程序](https://zhida.zhihu.com/search?content_id=250701039&content_type=Article&match_order=1&q=用户程序&zhida_source=entity)中，我也没有看到`SYS_`开头的东西诶。在用户态的代码声明中，系统调用是下面这个样子的：

![image-20250101151512421](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250101151512421.png)

从代码来看，这些声明是在user文件夹下 ， 一点系统调用的样子都没有。用IDE的代码跳转功能，也没有办法从`fork()`跳转到`sys_fork()`。这是怎么回事呢？

这里有一个文件，就是那个脚本文件和里面的 usys.S

![image-20250101151649537](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250101151649537.png)

用户程序中调用 `fork()`函数，实际上是跳转到`usys.S`中运行：寄存器`a7`中存入对应的系统调用编号 `SYS_fork`，然后调用 `ecall`命令。当系统调用结束，会返回到这里继续执行 `ret` 命令。
这个跳转过程也比较复杂，我这里进行一个总结。

![image-20250101151720248](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250101151720248.png)



# Page tables

## Paging hardware

​	xv6 runs on Sv39 RISC-V means that only the bottom 39 bits of a 64-bit virtual address are used; the top 25 bits are not used. In this Sv39 confifiguration, a RISC-V page table is logicallyan array of $2^{27}$ (134,217,728) *page table entries (PTEs)*(==RISC-V页表在逻辑上是一个由 $2^{27}$ 个页表条目组成的数组==). Each PTE contains a 44-bit physical pagenumber (PPN) and some flflags. The paging hardware translates a virtual address by using the top 27 bits of the 39 bits to index into the page table to find a PTE . and making a 56-bits physical address whose top 44 bits come from the PPN in the PTE and whos bottom 12 bits  are copied from the original virtual address.

![image-20250110202338105](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250110202338105.png)

​	A RISC-V CPU translates a virtual address into a physical in three steps.
A page table is stored in physical memory as a three-level tree.The root of the tree is a 4096-byte page-table page that contains 512 PTEs , which contain the physical addresses for page-table pages in next level of the tree. Each of those pages contains 512 PTEs for the fifinal level in the tree.The paging hardware uses the **top 9 bits** of the 27 bits to select a PTE in the root page-table page,the **middle 9 bits** to select a PTE in a page-table page in the next level of the tree, and the **bottom 9 bits** to select the fifinal PTE.

![image-20250110204658740](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250110204658740.png)

 	If any of the three PTEs required to translate an address is not present, the paging hardwareraises a *page-fault exception*, leaving it up to the kernel to handle the exception

​	The three-level structure of Figure 3.2 allows a memory-effificient way of recording PTEs, compared to the single-level design of Figure 3.1.

​	因为 CPU 在执行转换时会在硬件中遍历三级结构，所以缺点是 CPU 必须从内存中加载三个 PTE 以将虚拟地址转换为物理地址。为了减少从物理内存加载 PTE 的开销，RISC-V CPU 将页表条目缓存在 Translation Look-aside Buffer (TLB) 中。

​	Each PTE contains flflag bits that tell the paging hardware how the associated virtual addressis allowed to be used. `PTE_V` indicates whether the PTE is present: if it is not set, a reference tothe page causes an exception (i.e. is not allowed). `PTE_R` controls whether instructions are allowedto read to the page. `PTE_W` controls whether instructions are allowed to write to the page. `PTE_X` controls whether the CPU may interpret the content of the page as instructions and execute them.`PTE_U` controls whether instructions in user mode are allowed to access the page; if `PTE_U` is notset, the PTE can be used only in supervisor mode. Figure 3.2 shows how it all works. The flags and all other page hardware-related structures are defifined in (kernel/riscv.h)

​	To tell the hardware to use a page table , the kernel must write the physical address of  the root page-table page into the `satp` register. Each CPU has its own satp. A CPU will translate all addresses generated by subsequent instructions using the page table pointed to by its own satp.Each CPU has its own satp so that different CPUs can run  different processes, each with a private address space described by its own page table.



## Kernel address space 

​	Xv6 maintains one page table per process , describing each process's user address space , plus a single page table that describes the kernel's address space.The kernel confifigures the layout of its address space to give itself access to physical memory and various hardware resources at predictablevirtual addresses. Figure 3.3 shows how this layout maps kernel virtual addresses to physical addresses. The fifile (kernel/memlayout.h) declares the constants for xv6’s kernel memory layout.

​	QEMU simulates a computer that includes RAM (physical memory) starting at physical address 0x80000000 and continuing through at least 0x86400000, which xv6 calls PHYSTOP.
在`0x80000000`以下，对应着io设备

![image-20250111102938946](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250111102938946.png)

​	直接映射是一种将物理内存地址直接映射到虚拟地址空间的方法。这种方法的主要优点是简化了内核访问内存和设备寄存器的过程，减少了地址转换的开销。例如，当`fork`为子进程分配用户内存时，分配器返回该内存的物理地址；`fork`在将父进程的用户内存复制到子进程时直接将该地址用作虚拟地址。

有几个内核虚拟地址不是直接映射

- 蹦床页面。它映射在虚拟地址空间的顶部；用户页表具有相同的映射。
- 内核栈页面 Each process has its own kernel stack , which is mapped high so that below it xv6 can leave an unmapped `guard page`.The guard page's PTE is invalid . So that if the kernel overflows a kernel stack , it will likely cause an exception and the kernel will panic.



## Physical memory allocation

​	The kernel must **allocate and free** physical memory at run-time for page tables , user memory,kernel stacks , and pipe buffer.

​	xv6 uses the physical memory between the end of the kernel and `PHYSTOP` for run-time allocation. It allocates and frees whole 4096-byte pages at a time. It keeps track of which pages are free by threading a linked list through the pages themselves. Allocation consists of removing a page from the linked list; freeing consists of adding the page to the list.

## Process address space

​	Each process has a separate page table , and when xv6 switches between processes , it also changes page tables.

​	When a process asks xv6 for more user memory , xv6 first uses `kalloc` to allocate physical pages. It then adds PTEs to the process's page table that point to the new physical pages.Xv6 sets the PTE_W , PTE_X , PTE_R , PTE_U , and PTE_V flags in these PTEs.

​	First, different processes’ page tables translate user addresses to different pages of physical memory, so that each process has private user memory. Second, each process sees its memory as having contiguous virtual addresses starting at zero, while the process’s physical memory can be non-contiguous. Third, the kernel maps a page with trampoline code at the top of the user address space, thus a single page of physical memoryshows up in all address spaces.

​	Figure shows the layout of the user memory of an executing process in xv6 in more detail. The stack is a single page , and is shown with the initial contents as cerated by exec. Strings containing the command-line arguments , as well as an array of pointers to them , are at the very top of the stack. Just under that are values that allow a program to start at `main` as if the function `main(argc , argv)` had just been called.
![image-20250112152957752](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250112152957752.png)

​		To detect a user stack overflowing the allocated stack memory , xv6 places an inaccessible guard page right below the stack by clearing the  `PTE_U` flag.	**If the user stack overflflows and the process tries to use an address below the stack,** the hardware will generate a page-fault exception because  the **guard page is inaccessible to a program running in user mode.** A real-world operating system might instead automatically allocate more memory for the user stack when it overflflows.





## 补充

虚拟内存有64位 ， 其中最高位的25位是`unused` ，27位的`index`，12位`offset`
物理内存有64位，8个未使用 ， 剩下的56位中， 有44位`PPN` , 12位的`offset`
PTE : 有 10位未使用 44位PPN ， 10位 flags

三级页表：将27位分成三部分。CPU中的satp寄存器 存储了第一个页的物理地址，**可以从第一个页中找到我们需要的PTE**，取它的PPN，然后在物理内存中，找到下一个表的位置。  然后又通过第二个9位找到第二个表中所对应的PTE ，取PPN ， 找到最后一个表的位置。 通过最后一个9位找PTE，然后PPN，然后+ `offset`组成56位的物理地址

### 源码部分

**riscv.h**

```c
typedef uint64 pte_t;
typedef uint64 *pagetable_t; // 512 PTEs
#define PGSIZE 4096 // bytes per page
#define PGSHIFT 12  // bits of offset within a page 这里是offset的12位
#define PGROUNDUP(sz)  (((sz)+PGSIZE-1) & ~(PGSIZE-1)) //上取整
#define PGROUNDDOWN(a) (((a)) & ~(PGSIZE-1)) //下取整

#define PTE_V (1L << 0) // valid  这里是几个flag标志位，也就是pte的后十位
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access

// shift a physical address to the right place for a PTE.
#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10) //把物理地址转成 pte中的PPN部分

#define PTE2PA(pte) (((pte) >> 10) << 12) // pte 转 PPN

#define PTE_FLAGS(pte) ((pte) & 0x3FF) //取flag位 ，也就是pte后十位

#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))//最大内存地址
```

**vm.c**

```c
/*对kernel的虚拟地址空间初始化*/
pagetable_t kernel_pagetable;
pagetable_t
kvmmake(void)
{
  pagetable_t kpgtbl;

  kpgtbl = (pagetable_t) kalloc();
  memset(kpgtbl, 0, PGSIZE);

  // uart registers
  kvmmap(kpgtbl, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(kpgtbl, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // PLIC
  kvmmap(kpgtbl, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap(kpgtbl, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

  // map kernel stacks
  proc_mapstacks(kpgtbl);
  
  return kpgtbl;
}
```

```c
// Switch h/w page table register to the kernel's page table,
// and enable paging.
/*切换cpu的stap寄存器 内部函数是用risc-v汇编实现的*/
void
kvminithart()
{
  w_satp(MAKE_SATP(kernel_pagetable));
  sfence_vma();
}
```

```c
/*三级页表的转换*/
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte); //把这9位所对应的pte所对应的物理地址拿到
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0) //如果缺页的时候不允许提供页 或者 提供页失败，返回0
        return 0;
      memset(pagetable, 0, PGSIZE); // 否则的话， 提供一个页
      *pte = PA2PTE(pagetable) | PTE_V; //将物理地址转成PPN ， 在或一个标志位
    }
  }
  return &pagetable[PX(0, va)];//返回最后一个9位地址所指向的pte
}
```

注意，上面返回了一个指针，但它不是未定义行为， 因为页表的生命周期长

```c
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);//pte转ppn 但此时还没有加上offset
  return pa;
}
```

```c
/*添加一个映射关系*/
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  if(size == 0)
    panic("mappages: size");
  
  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("mappages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

### 过程

当xv6刚开始运行时，里面还没有page，它会自己设置好内核所使用的地址空间。 **这里xv6设计者为了简化这个mapping的关系** ，决定使用虚拟地址与物理地址一一对应的方式来实现。

当内核启动一个新进程时，首先会分配物理空间，这个物理空间既给页表，也给进程的地址空间，分别是`uvmcreate` 和 `uvmalloc`(其中都通过kalloc分配)。

**内核映射在内核页表中**：通过 `kvmmake()` 创建并初始化 `kernel_pagetable`，其中包含所有内核空间的映射。

**用户进程的页表独立**：通过 `uvmcreate()` 创建的用户页表仅包含用户空间的映射，不包含内核映射。

而 CPU 无论是在用户态还是在内核态，访问的均是虚拟内存地址，不管是用户空间的虚拟内存地址还是[内核空间](https://zhida.zhihu.com/search?content_id=231478167&content_type=Article&match_order=1&q=内核空间&zhida_source=entity)的虚拟内存地址最终都是要与物理内存进行映射的，而通过前边的介绍我们也知道了，虚拟内存与物理内存的映射关系是通过页表来管理的。 **所以页表分为两个部分**

1. 进程用户态页表：主要负责管理进程用户态虚拟内存空间到物理内存的映射关系。
2. 内核态页表 ：主要负责管理内核态虚拟内存空间到物理内存的映射关系，这一部分主要供内核使用。

# Traps and system calls

​	There are three kinds of event which cause the CPU to set aside ordinary execution of instructions and force a transfer of control to special code that handles the event.

- **system call** when a user program executes the `ecall` instruction to ask the kernel to do somthing for it.
- **exception** an instruction(user or kernel) does somthing illegal , such as divide by zero or use an invalid virtual address.
- **device interrupt** when a device signals that it needs attention , for example when the disk hardware finishes a read or write request.

​	This book uses `trap` as a generic term for these situation.Traps to be transparent. The usual sequence is that a trap forces a transfer of control into the kernel; the kernel save registers and other state so that execution can be resumed.

​	Xv6 handles all traps in the kernel.

​	Xv6 trap handling proceeds in four stages

1. hardware actions taken by the RISC-V CPU
2. 为内核C代码执行而准备的汇编程序集“向量”
3. a C function that decides what to do with the trap
4. the system call or device-driver service routine.

对于三种不同的情况：来自用户空间的陷阱、来自内核空间的陷阱和定时器中断，分别使用单独的程序集向量和C陷阱处理程序更加方便。

## RISC-V trap machinery

​	Each RISC-V CPU has a set of control registers that the kernel  writes to tell the CPU how to handle traps ,and that the kernel can read to find out about a trap that has occured. `riscv.h`contains definitions that xv6 uses.

- `stvec` : The kernel writes the address of its trap handler here.the RISC-V jumps to the address in `stvec` to handle a trap.
- `sepc` : When a trap occurs , RISC-V saves the program counter here(since `pc` is then overwritten with the value in `stvec`). The `sret`(return from trap) instruction copies `sepc` to the `pc`. The kernel can write `sepc` to control where `sret` goes.
- `scause` : RISC-V puts a number here that describles the reason 



## Traps from user space 

​	. The high-level path of a trap from user space is uservec (kernel/trampoline.S:16), then usertrap (kernel/trap.c:37); and when returning, usertrapret (kernel/trap.c:90) and then userret (kernel/trampoline.S:88)
来自用户代码的陷阱比来自内核的陷阱更具有挑战性。因为`stap`指向**不映射内核**的用户页表 ， 栈指针可能包含无效甚至恶意的值

由于RISC-V硬件在陷阱期间不会切换页表，所以用户页表必须包含`uservec`的映射。 `uservec`必须切换`satp`以指向内核页表；为了在切换后继续执行指令，`uservec`必须在内核页表中与用户页表中映射相同的地址

```
当trap后，cpu仍然使用的是用户页表，这个页表无法访问内核空间
在stvec中，存储的是解决trap函数的起始地址（虚拟地址），所以用户页表必须包括这个地址的映射，而且映射后的物理地址还必须与内核页表中映射这个地址的物理地址相同
```

因为虽然成功映射到了物理地址，但是由于用户页表无法访问内核空间，所以使用了一个特殊机制，**trampoline page**  这是一个特殊区域，它映射在 **用户页表** 和 **内核页表** 相同的虚拟地址上。这个虚拟地址通常被定义为 `TRAMPOLINE` 这个页面的内容会在**陷阱处理程序**（如`uservec`）跳转时执行。
蹦床页面的映射被设置在 **用户页表**和**内核页表**中相同的虚拟地址。例如，假设 **`TRAMPOLINE`** 地址为 `0x80000000`，那么 **用户页表** 和 **内核页表** 都会将这个虚拟地址映射到同一个物理地址。
这样当**陷阱发生**时，不论是用户页表还是内核页表，都能通过相同的虚拟地址访问到这个蹦床页面，从而执行陷阱处理程序。

```
当trap发生时，用户页表映射解决trap的函数的起始地址，然后通过trampline进入内核，让内核去访问这个映射到的物理地址
```



当trap发生时，cpu会将用户空间的寄存器保存到某些特殊寄存器或内存中。对于陷阱处理程序来说，它需要确保

- 保存用户态的寄存器值 ， 以便在陷阱处理后恢复
- 可以操作内核的状态（比如`satp`寄存器）

`sscratch`寄存器：它用于存储在**内核态**下的上下文信息。 在`uservec`（陷阱处理程序）执行时，`sscratch`可以帮助保存和交换一些重要寄存器的内容。 当**用户程序**触发**陷阱**时，CPU会保存一组寄存器的值，这些值包括 **a0-a7** 等寄存器，这些值可能会在陷阱处理时发生变化。**`sscratch`** 寄存器是一个临时寄存器，可以用于保存某些重要的寄存器值（比如 `a0`），以便在陷阱处理程序（`uservec`）中恢复或修改它们。

在xv6中，trap发生时，内核需要保存用户程序的寄存器状态，执行适当的处理，然后恢复用户程序的执行。为了保证这种上下文切换的顺利进行，操作系统需要为每个进程分配一个 **陷阱帧**，用于保存 **用户寄存器** 的内容。

1. **陷阱帧**（Trap Frame）
   - 每个进程都有一个陷阱帧 ， 用于保存用户程序在陷阱发生时的**寄存器状态**
   - 陷阱帧是一个结构体，包含了用户程序中被保存的寄存器值，如 `a0-a7`、`s0-s11`、`sp` 等，这些寄存器的值会在陷阱发生时被保存到陷阱帧中。
   - `kernel/proc.h` 文件中的 **`trapframe`** 结构体定义了这些寄存器的保存格式。陷阱帧存储在一个页面中，并且每个进程都有自己独立的陷阱帧。
2. `sscratch` 寄存器和陷阱帧的位置
   - **`uservec`** 在处理陷阱时需要先保存用户程序的寄存器。为了保存这些寄存器的状态，内核将 **`sscratch`** 寄存器设置为指向每个进程的 **陷阱帧**。
   - **`sscratch`** 寄存器存储的指针是该进程陷阱帧的虚拟地址，指向一个物理页面，页中包含了 **保存用户寄存器的空间**。
3. **映射陷阱帧到用户空间**
   - **陷阱帧**必须映射到**用户地址空间**中， 这样`uservec`才能将用户寄存器的值保存到其中。在 **xv6** 中，内核为每个进程分配一个页面，专门用来存储 **陷阱帧**。
   - 为了让用户程序在陷阱发生时能够访问陷阱帧，**`uservec`** 会将该陷阱帧映射到 **用户空间** 中的一个特定地址，这个地址通常是 `TRAPFRAME`，并且 `TRAPFRAME` 的虚拟地址会在 **`TRAMPOLINE`** 下面。
   - 在用户页表中，**`TRAPFRAME`** 作为一个固定的虚拟地址，这样 **`uservec`** 就可以通过该地址将用户寄存器的状态保存到陷阱帧。

**总结**
当陷阱发生时，内核通过`sscratch`寄存器指向该进程的**陷阱帧**，但是陷阱帧中的数据（即用户程序的寄存器状态）时如何获得的呢？这个过程涉及到 **用户态寄存器的保存** 和 **陷阱帧的存储**。

1. 陷阱帧的概念

   - 陷阱帧时一个数据结构，用来保存用户程序的寄存器状态。当陷阱发生时，内核会将用户程序的寄存器（如 `a0-a7 s0-s11`等）保存到**陷阱帧**中，这样当陷阱处理完成后，内核可以恢复用户程序的上下文，继续从中断或异常发生前的状态执行。

2. `sscratch`寄存器指向陷阱帧

   - 在xv6操作系统中，`sscratch`寄存器用于临时存储信息。在陷阱发生时，`sseratch`会被设置为指向该进程的陷阱帧的位置。陷阱帧本身是一个物理页面，其中保存了用户程序的寄存器状态。
   - 每个进程在创建时会为其分配一个陷阱帧页面，并将该页面映射到用户空间的虚拟地址上。这个页面在用户页表和内核页表中都被映射，但访问权限不同：用户页表映射该页面让陷阱帧存在于用户虚拟空间，内核页表则可以让内核访问它。

3. 陷阱发生时保存用户寄存器

   当陷阱发生时，CPU 会自动保存某些重要的寄存器（如 `pc`、`a0-a7`、`sp` 等），这些寄存器的值是用户程序的执行状态。如果陷阱发生时，CPU 处于用户空间，那么用户程序的寄存器（也包括栈指针）会被 **保存到陷阱帧中**。
   **具体步骤如下**

   1. **陷阱触发** ：当陷阱发生时，CPU 会通过 **`stvec`** 寄存器跳转到 **`uservec`**。此时，CPU 会根据预先定义的陷阱处理程序的地址开始执行，并且会根据 CPU 的硬件机制保存当前的用户寄存器值。

   2. **保存寄存器到陷阱帧** ： 

      - 内核通过 **`sscratch`** 寄存器指向该进程的陷阱帧。这个陷阱帧位于一个物理内存页中，包含了所有的用户寄存器。
      - 在 **`uservec`**（内核的陷阱处理程序）中，内核将 **`a0-a7`**、`s0-s11`、`sp` 等寄存器的值保存到 **陷阱帧**。这些寄存器的值来自 **陷阱发生时用户程序的状态**。

   3. **`uservec` 操作**：

      - 内核需要使用**`sscratch`** 寄存器来确定 **陷阱帧的位置**，并将用户的寄存器值存入该位置。
      - 通常情况下，**`sscratch`** 寄存器在陷阱发生时会指向 **该进程的陷阱帧**。然后，内核会将保存寄存器状态的逻辑实现为一组内存操作，将寄存器的值存储到该内存位置（即陷阱帧）。

   4.  **陷阱帧的具体结构**

      陷阱帧中保存的数据不仅仅是寄存器的值，它还包括了 **栈指针（`sp`）**、**程序计数器（`pc`）** 以及 **其他状态信息**。这些信息是在 **陷阱发生时自动保存的**，然后由内核进一步处理。

      在 **xv6** 中，**`trapframe`** 的结构定义通常包含如下内容：

      - `a0` 到 `a7`：系统调用或异常传递的参数或返回值。
      - `s0` 到 `s11`：保存的用户寄存器（这些寄存器在用户代码中使用的保存区域）。
      - `sp`：栈指针，指向当前堆栈位置。
      - `pc`：程序计数器，指向当前正在执行的指令地址。

      当陷阱发生时，这些信息会被保存到 **陷阱帧** 中。

   5.  **陷阱帧的存储位置**

      - **陷阱帧页面**是由操作系统为每个进程分配的物理内存页面，并且该页面会映射到 **用户虚拟地址空间** 中。通常，该页面的映射会放置在 **`TRAPFRAME`** 地址附近，这个地址就在 **`TRAMPOLINE`** 下面。
      - 对于每个进程，**`p->trapframe`** 变量会指向该进程的 **陷阱帧**，这样内核就可以在处理过程中通过 **内核页表** 访问陷阱帧。

> ecall指令
>
> 会从user model -> supervisor model
>
> pc指针存储到sepc寄存器
>
> pc设置为stvec（这个寄存器里面是我们预先设置好的处理代码，也就是handler）

来自用户空间的陷阱的高级路径是`uservec` -> `usertrap`  返回时`usertrapret` -> `userret`

当trap发生时，RISC-V硬件的处理流程

1. 如果是设备中断，且处于关中断的状态(sstatus的SIE位为0)，忽略该中断。
2. 清空SIE位，即关中断。
3. 将`pc`的值复制到`spec`中，即保存断点。
4. 将当前mode保存到SPP位中
5. 将当前mode设置为supervisor mode
6. 将stvec的地址复制到pc中，跳到Trap处理的地方（handler）。
7. 开始执行新的pc。

注意到CPU没有自动切换[内核页表](https://zhida.zhihu.com/search?content_id=211630923&content_type=Article&match_order=1&q=内核页表&zhida_source=entity)，没有切换内核栈，除了pc以外没有保存任何寄存器，所以内核的软件部分应该做这些任务。

**跳入**
stvec中的虚拟地址就是指向uservec，uservec主要是保存32个寄存器的值到内存中。

对于[xv6](https://zhida.zhihu.com/search?content_id=211630923&content_type=Article&match_order=2&q=xv6&zhida_source=entity)中每个进程的页表，都有一个trampoline页和trapframe页，它们位于[虚拟地址空间](https://zhida.zhihu.com/search?content_id=211630923&content_type=Article&match_order=1&q=虚拟地址空间&zhida_source=entity)的顶端，它们都没有PTE_U标志。

trampoline页里面的内容就是**uservec**。因为用户页表的trampoline映射没有PTE_U标志，所以在用户空间时无法执行uservec中的代码，只有trap发生时进入到supervisor模式才可以执行。在内核页表同样的虚拟地址中也映射了一样的trampoline页，所以当切换到内核页表后，trap handler还可以继续执行。

在uservec的最后一步会跳到usertrap中

usertrap就是查看trap的原因，根据不同的原因调用不同的函数处理。

**跳出**

`usertrapret` 函数负责将控制权从内核模式切换回用户模式，恢复用户程序的状态。

**`usertrapret`** 函数的作用是将进程从内核态切换回用户态。它通过设置一些寄存器，恢复用户程序的状态，并调用 trampoline 页中的代码来完成真正的用户态切换。

**关键步骤**：保存内核相关状态，设置 SRET 指令的必要寄存器，更新 `sepc` 和 `satp`，并最终跳转到 trampoline 页，进行用户态的恢复。

这样，整个 `usertrapret` 函数的过程就是一个典型的上下文切换流程，其中包括内核态到用户态的状态保存与恢复。

**`usertrapret`** 是一个“准备”函数，设置好用户态所需的环境，并调用 `userret` 来实际恢复用户进程的状态。

**`userret`** 是实际从内核返回到用户态的函数，恢复用户程序的状态并跳转到用户程序的位置。

## gdb跟踪trap（可能与上文有重复）

```
make CPUS=1 qemu-gdb
```

```shell
gdb-multiarch kernel/kernel

set confirm off
set architecture riscv:rv64
set riscv use-compressed-breakpoints yes
target remote localhost:25000
```

如果要调试用户态下的程序，需要

```
file user/_call
```

![image-20250124212040663](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250124212040663.png)

这是shell程序的page table ， 这6条映射关系是有关shell的指令和数据，以及一个无效的page用来作为guard page(红线的那里)，以防shell尝试使用过多的stack page

最后两条的虚拟地址非常大， 没错它们就是`trapframe page`和`trapoline page`

### ecall指令

涉及到的寄存器有stvec , sepc , scause , sstatus
执行如下操作

1. 如果陷阱是设备中断，并且状态**SIE**位被清空，则不执行以下任何操作。
2. 清除**SIE**以禁用中断。
3. User mode -> supervisor mode
4. Let sepc = pc
5. Let pc = stvec
6. Jump to pc
7. 设置`scause`以反映产生陷阱的原因。
8. 将当前模式（用户或管理）保存在状态的**SPP**位中。

**通过ecall进入内核态**

![image-20250124212956068](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250124212956068.png)

### uservec函数

- 保存用户寄存器
- 切换成内核页表，内核栈，把当前执行的进程CPU号装载到寄存器
- 跳入`usertrap`
  ![image-20250124220253971](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250124220253971.png)

**疑问**

```
什么是32个用户寄存器？
在uservec代码中，使用了SSCRATCH寄存器里的值来定位trapframe结构体，那么SSCRATCH这个寄存器的值是如何被写入的？
```

```
对于第二个问题的解答
Robert教授：在内核前一次切换回用户空间时，内核会执行set sscratch指令，将这个寄存器的内容设置为0x3fffffe000，也就是trapframe page的虚拟地址。所以，当我们在运行用户代码，比如运行Shell时，SSCRATCH保存的就是指向trapframe的地址。之后，Shell执行了ecall指令，跳转到了trampoline page，这个page中的第一条指令会交换a0和SSCRATCH寄存器的内容。所以，SSCRATCH中的值，也就是指向trapframe的指针现在存储与a0寄存器中。
第一个问题需要查看RISC-V手册
```



### usertrap函数

- 分情况，执行系统调用/中断/异常处理逻辑
- 修改了stvec的值 ， 还可能会修改sepc的值

### usertrapret函数

- 填入了trapframe的内容，这样下一次从用户空间转换到内核空间时可以用到这些数据
- 恢复stvec ， sepc的值

**疑问**

```
那么第一次从用户空间到内核空间怎么办
```

```
总结来说，第一次使用trap时，trapframe 的内容是通过硬件自动保存的寄存器状态获得的，而后续的trap则通过内核中的 usertrapret 函数来设置和管理这些信息。
```



### userret函数

- 恢复用户寄存器
- 把用户空间的page table ， 用户空间的stack装载到寄存器
- 执行sret指令

### sret指令

- 程序切换回user mode
- SEPC寄存器中的数值会被拷贝到pc寄存器
- 重新打开中断

## Traps from kernel space



## Page-fault exception

RISC-V有三种页面异常

- 加载页面错误 ： 加载指令无法转换其虚拟地址
- 存储页面错误 ： 存储指令无法转换其虚拟地址
- 指令页面错误 ： 指令的地址无法转换

`scause`寄存器中的值指示页面错误的类型，`stval`寄存器包含无法翻译的地址。

COW fork中的基本计划是让父子最初共享所有物理页面，但将它们映射为只读。因此，当子级或父级执行存储指令时，risc-v CPU引发页面错误异常。为了响应此异常，内核复制了包含错误地址的页面。它在子级的地址空间中映射一个权限为读/写的副本，在父级的地址空间中映射另一个权限为读/写的副本。更新页表后，内核会在导致故障的指令处恢复故障进程的执行。由于内核已经更新了相关的PTE以允许写入，所以错误指令现在将正确执行。

COW策略对`fork`操作很有效， 因为通常子进程在fork之后会执行`exec` ，用新的地址空间替换其地址空间。在这种常见情况下，子级只会触发很少的页面错误，内核可以避免拷贝父进程内存完整的副本。

**另一个广泛使用的特性叫做惰性分配——lazy allocation** 

**回顾**
虚拟内存的好处 ： 1、隔离  2、提供了间接性，处理器指令只使用虚拟地址

### 想要实现动态映射（改变页表）需要的信息

如果发生页面错误 ，内核需要相应这个页面错误。 显然我们需要 **错误的虚拟地址** 它在`stval`寄存器中 ，**错误类型**在`scause`寄存器中 ， **引起页面错误的指令的虚拟地址**(sepc寄存器)
这里第一个和第三个可能会造成疑惑 ，第三个是指令所在的va地址 ， 第一个是指令中想要访问的va地址

### 基本机制

- Allocation : `sbrk`是xv6提供的系统调用，它允许应用程序增加自己的堆空间 **eager allocation**

### Lazy allocation

在`sbrk`中我们基本上什么也不做，唯一要做的就是：记住增加了地址空间 `p->sz += n`.
之后在某个时刻，程序可能会要使用这个内存，此时会导致`page fault`，因此，如果我们引用虚拟内存`>p->sz && <p->sz+n`，我们希望的是，内核分配一个页面并重新启动指令。

在页面错误处理程序中，我们可以分配一个页面，使用`kalloc`分配一页，置零，并映射到页表中，更新页表，然后重启指令

### 按需补零

`BSS`段内的数据都是0 ， 在虚拟地址空间中，这个段可能会有很多页，并且数据都是0，所以在物理空间中，可以用一个全是0的页，作为这些0数据页的映射，这会节省很多物理空间
这些映射必须是特殊的，我们不允许对它写入，所以它是只读的。

那么在这种情况下，如何处理页面错误呢？
类似于`COW`,新分配一个物理页，修改映射和PTE权限，重启指令

操作系统为什么会这样做呢？

### 写入时复制(copy-on-write)

比如shell的执行，实际上是`fork`了一个子进程，所以我们既有父进程又有子进程，而子进程做的第一件事就是`exec`,那么exec会丢掉现在的空间而执行其它内容

我们采用写时复制，不是给子进程复制或分配新的物理内存，而是共享父进程已经分配的物理页面。所以，我们只是把子进程的pte指向父进程物理页面的相同位置。 如果子进程想要修改其中一个页面，那么这个更新不应该对父进程可见，为了做到这一点，将pte设置为只读，这样一来，在写的时候就会页面错误

这时，需要把那个页面复制一份，先分配一个新页面，然后复制出现错误的页面内容到新页面。把新页面映射到子进程中

对于什么时候可以释放物理页，使用引用计数的方法，没有进程使用时释放

### 按需调页

 



# 拓展 ： Memory Layout of C Programs

![image-20250128220605970](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250128220605970.png)

## Text Segment

The **text segment** (also known as **code segment**) is where the executable code of the program is stored. It contains the compiled machine code of the program’s functions and instructions. This segment is usually read-only and stored in the lower parts of the memory to prevent accidental modification of the code while the program is running.

The size of the text segment is determined by the number of instructions and the complexity of the program.

## Data Segment

The ***\*data segment\**** stores global and static variables that are created by the programmer. It is present just above the code segment of the program. It can be further divided into two parts:

**A. Initialized Data Segment**

```c
int a = 10;
static int b = 20;
```

The above variables a and b will be stored in the Initialized Data Segment.

**B. Uninitialized Data Segment(BSS)**
Uninitialized data segment often called the “***\*bss\****” segment, named after an ancient assembler operator, that stood for “Block Started by Symbol” contains global and static variables that are not initialized by the programmer. These variables are automatically initialized to zero at runtime by the operating system. For example, the below shown variables will be stored in this segment:

```c
int a;
static int b;
```

## Heap Segment

Heap segment is where dynamic memory allocation usually takes place. The heap area begins at the end of the BSS segment and grows towards the larger addresses from there. It is managed by functions such as malloc(), realloc(), and free() which in turn may use the brk and sbrk system calls to adjust its size.

The heap segment is shared by all shared libraries and dynamically loaded modules in a process. For example, the variable pointed by ***\*ptr\**** will be stored in the heap segment:

```c
int *ptr = (int*)malloc(sizeof(int) * 10);
```

## Stack Segment

The ***\*stack\**** is a region of memory used for ***\*local variables\**** and function call management. Each time a function is called, a ***\*stack frame\**** is created to store local variables, function parameters, and return addresses. This stack frame is stored in this segment.

The stack segment is generally located in the higher addresses of the memory and grows opposite to heap. They adjoin each other so when stack and heap pointer meet, free memory of the program is said to be exhausted.

Example of data stored in stack segment:

```c
void function()
{
    int local_var = 10;
}
```

# Interrupts and device 

​	A driver is the code in an operating system that   manages a particular device :  it confifigures the device hardware, tells the device to perform operations , handles the resulting interrupts, and interacts with processes that may be waiting for I/O from the device.In addition, the driver must understand the device’s hardware interface, which can be complex and poorly documented.

## Iuterrupts

中断与系统调用和异常略有不同

1. asynchronous(异步) : 当硬件生成中断时，中断处理程序运行，中断处理程序可能与CPU上当前运行的进程无关
2. concurrency（并发）:CPU和生成的设备是并行运行的
3. program devices : 

### 中断从哪里来 - 硬件，驱动

这里主要关注外部中断，而不是时钟中断或软件中断

外部中断来自电路板上的设备。 通常，管理设备的代码称为驱动程序 ，`uart.c`控制uart

设备出现在物理地址空间的特定地址，操作系统需要知道这些设备位于物理内存空间的哪个位置

### risc-v中断

`SIE寄存器` 只有一位 ， 硬件中断，软件中断，计时器中断
`SSTATUS` 有一个位来禁用和启用特定内核上的中断 ， 所以每个核心都有这些寄存器
`SIP`  管理程序中断挂起寄存器。 当中断发生时，查看寄存器，看看是什么中断
`SCAUSE` 
`STVEC`


## Code : Console input

​	The console dreiver is a simple illustration of driver structure.
控制台驱动程序通过连接到RISC-V的UART串口硬件接受人们键入的字符。控制台驱动程序一次累积一行输入，处理如`backspace`和`Ctrl-u`的特殊输入字符。



# Locking

## Race conditions

 As an example of why we need locks, consider two processes calling wait on two different CPUs.
`wait` frees the child's memory.Thus on each CPU, the kernel will call kfree to free the children’s pages. 为了获得最佳性能，我们希望两个父进程的`kfree`可以并行执行，而不必等待另一个进程，但是考虑到xv6的`kfree`实现，这将导致错误。

如果存在隔离性，那么这个实现是正确的。但是，如果多个副本并发执行，代码就会出错。当两次执行位于第16行的对`list`的赋值时，第二次赋值将覆盖第一次赋值；第一次赋值中涉及的元素将丢失。

**竞态条件是指多个进程读写某些共享数据（至少有一个访问是写入）的情况**。避免竞争的通常方法是使用锁。锁确保互斥，这样一次只有一个CPU可以执行`push`中敏感的代码行；这使得上述情况不可能发生。

避免竞争的方法通常是使用锁。锁确保互斥，这样一次只有一个cpu可以执行`push`中敏感的代码行；使得上述例子不可能发生

```c
struct element{
    int data;
    struct element *next;
}
struct element * list = 0;
struct lock listlock;
void
push(int data){
    struct element *l;
    l = malloc(sizeof *l);
    l->data = data;
    acqeuire(&listlock);
    l->next = list;
    list = l;
    release(&listlock);
}
```

`acquire`和`release`之间的指令序列通常被称为临界区域。锁的作用通常被称为保护`list`

## Code : locks

Xv6 has two types of locks: spinlocks and sleep-locks. We’ll start with spinlocks. `struct spinlock` , 结构体中重要字段是`locked`，当锁可用时位0，当它被持有时为非0.

```c
void
acquire(struct spinlock * lk)
{
    for(;;){
        if(lk->locked == 0){
            lk->locked = 1;
            break;
        }
    }
}
```

但是，这样仍然无法保证多处理器的互斥，可能会发生两个CPU同时到达第5行，看到`lk->locked`为0，然后都通过执行第六行占有锁。此时就有两个不同的CPU持有锁，从而违反了互斥属性。于是，我们需要一种方法**使第五行和第六行作为原子步骤执行**

多核处理器通常提供实现第五行和第六行的原子版本的指令。在RISC-V上，这条指令时`amoswap r , a`.交换r和a。 **它原子地执行这个指令序列，使用特殊的硬件来防止任何其他CPU在读取和写入之间使用内存地址**

```c
void 
acquire(struct spinlock *lk)
{
    push_off(); //disable interrupts to avoid deadlock
    if(holding(lk))
		panic("acquire");
    
    while(__sync_lock_test_and_set(&lk->locked, 1) != 0);
    
    __sync_synchronize();
    
    lk->cpu = mycpu();
}
```

Xv6的`acquire`使用可移植的C库调用归结为`amoswap`的指令`__sync_lock_test_and_set`；返回值是`lk->locked`的旧内容。
`acquire`函数将swap包装在一个循环中，直到它获得了锁前一直重试自旋，每次迭代将1与`lk->locked`进行swap操作，并检查`lk->locked`之前的值。如果之前为0，swap已经把`lk->locked`设置为1，那么我们就获得了锁；如果前一个值是1，那么另一个CPU持有锁，我们原子地将1与`lk->locked`进行swap的事实并没有改变它的值。

获取锁后，用于调试，`acquire`将记录下来获取锁的CPU。`lk->cpu`字段受锁保护，只能在保持锁时更改。

**这里对于push_off();** 
为什么要关中断？在课程中，教授给出了一个直接的例子
我们假设在uartputc中获得了锁，而urat正忙于传输一些字符，那么当uart完成了传输字符，它会导致中断，并且uartintr运行。它会获取同一把锁，但uartputc已经持有了这个锁，假设只有一个CPU，那么他就死锁了。因为相同的CPU正在再次尝试获取相同的锁

```c
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}
```

函数`release`(***kernel/spinlock.c\***:47) 与`acquire`相反：它清除`lk->cpu`字段，然后释放锁。从概念上讲，`release`只需要将0分配给`lk->locked`。C标准允许编译器用多个存储指令实现赋值，因此对于并发代码，C赋值可能是非原子的。因此`release`使用执行原子赋值的C库函数`__sync_lock_release`。该函数也可以归结为RISC-V的`amoswap`指令。

## Code : Using locks

使用锁的一个困难部分是决定要使用多少锁，以及每个锁应该保护哪些数据和不变量。 有几个基本原则，首先，任何时候可以被一个CPU写入，同时可以被另一个CPU读写的遍历，都应该使用锁来防止两个操作重叠。
其次，请记住锁保护不变量（invariants）：如果一个不变量涉及多个内存位置，通常所有这些位置都需要由一个锁来保护，以确保不变量不被改变。

对于什么时候不需要锁

- **并行性不重要时** ： 如果内核操作不依赖于并行执行，或者系统中只有一个 CPU（例如单处理器系统），则可以在进入内核时加一个大锁（大内核锁）来简化锁的管理。
- **某些操作不涉及共享资源时**：如果某些操作完全是局部的，或者不会影响其他线程的执行，就不需要加锁。比如，只操作线程私有的数据，就不需要加锁。

**这里来看什么是大内核锁**

它在整个内核中只有一个锁。当一个线程或CPU进入内核时，它必须先获得这个锁，才能执行内核操作。在执行完毕后，它再释放这个锁。这种方法的优点是简单实现，特别适用于早期单处理器系统，因为没有并行性的担忧，只需要保护内核代码的访问。

然而，在多核系统中，这种大内核的缺点 ：**一次只能由一个CPU执行内核操作** ， 这意味着在多核系统上，只有一个CPU可以在内核中运行，其他的CPU必须等待，这极大地限制了并行性和性能。

**于是引出了细粒度锁**
为了提高效率，现代操作系统通常会使用“细粒度锁”，也就是在内核中为不同的资源（如内存、文件系统、网络等）引入多个独立的锁。这样，多个 CPU 就可以并行地执行不同的内核操作，而不是被一个大内核锁串行化，极大地提升了并行性和性能。

例如，在多核系统中，如果内核需要处理多个并行任务（如多个进程的调度、内存管理、IO 操作等），使用细粒度锁可以让这些任务同时在不同的 CPU 上执行，不会因为一个锁而造成性能瓶颈。

作为细粒度锁的一个例子，xv6对每个文件都有一个单独的锁，这样操作不同文件的进程通常可以不需等待彼此的锁而进程。

在后面的章节解释xv6的每个部分时，他们将xv6使用锁来处理并发的例子

| **锁**                | **描述**                                               |
| --------------------- | ------------------------------------------------------ |
| `bcache.lock`         | 保护块缓冲区缓存项（block buffer cache entries）的分配 |
| `cons.lock`           | 串行化对控制台硬件的访问，避免混合输出                 |
| `ftable.lock`         | 串行化文件表中文件结构体的分配                         |
| `icache.lock`         | 保护索引结点缓存项（inode cache entries）的分配        |
| `vdisk_lock`          | 串行化对磁盘硬件和DMA描述符队列的访问                  |
| `kmem.lock`           | 串行化内存分配                                         |
| `log.lock`            | 串行化事务日志操作                                     |
| 管道的`pi->lock`      | 串行化每个管道的操作                                   |
| `pid_lock`            | 串行化next_pid的增量                                   |
| 进程的`p->lock`       | 串行化进程状态的改变                                   |
| `tickslock`           | 串行化时钟计数操作                                     |
| 索引结点的 `ip->lock` | 串行化索引结点及其内容的操作                           |
| 缓冲区的`b->lock`     | 串行化每个块缓冲区的操作                               |

 Figure 6.3: Locks in xv6

## Deadlock and lock sorting

如果在内核中执行的代码路径必须同时持有数个锁，那么所有代码路径以相同的顺序获取这些锁是很重要的。如果它们不这样做，就有死锁的风险。 假设xv6中的两个代码路径需要锁A和B ， 但是代码路径1按照先A后B的顺序获取锁 ， 另一个路径按照先B后A的顺序获取锁。 假设线程T1执行代码路径1获取锁A， 线程T2执行代码路径2。**之后T1将尝试获取锁B ， T2将尝试获取锁A 。 两个获取都将无限期阻塞，因为在这两种情况下，另一个线程都持有所需的锁，并且不会释放它，直到它的获取返回。** 为了避免，所有代码路径必须以相同的顺序获取锁。全局锁获取顺序的需求一位着锁实际上是每个函数规范的一部分 ： **调用者必须以一种使锁按照约定顺序被获取的方式调用函数**

**模块化（暂时还不会）**

**锁与性能**
如果有一个大内核锁，这将使你的性能限于单个CPU上。 如果你想要具有多个CPU扩展的性能，你就得拆分数据结构。**Best split is a chanlleng** 
对于这个问题，有一种普遍的做法是，从粗粒度的锁开始，然后测量

## Code : study uart

**lock roles**

- protect this data struct
- tailend is in flight
- hardware registers have one writer 

## memory ordering

```c
acquir(r->locked);
x = x+1;
realse(r->locked);
```

在单一的串行执行中，编译器为了更好的性能，将`x = x + 1`优化到最后一行
但是在并发执行中是错误的，所以为了禁止或告诉编译器和硬件不要这么做，有一个叫做内存屏障的东西或者某个`synchronize` 这个指令表示，在这个点之前的加载或保存，不允许移动到这一点之后。

## lecture sum

- locks good for correctness but can be bad for performance
- locks complicate programming
- don't share if you don't hanve to
- start with 粗->细

## Sleeplock

在`Spinlock`中，存在一个明显的问题：CPU资源的浪费。当多个进程同时尝试一个锁时，如果锁无法立即释放，等待的进程会陷入不断的自选循环，占用CPU资源却无法执行实际的工作。这种设计在锁持有时间较短的时候效率较高，但对于锁持有时间较长的场景，会极大地降低系统性能。

为了应对这种情况，xv6 提供了另一种锁机制：`Sleeplock`。相比于自旋锁，`Sleeplock` 更适合锁持有时间较长的场景，因为它通过让等待的进程进入休眠状态（调用sleep系统调用），避免了无谓的CPU资源消耗。

```c
struct sleeplock{
    uint locked; //是否被获取
    struct spinlock lk; //用来保护该数据结构
    int pid ;//获取进程pid
}
```

在`Sleeplock`中我们利用一个自旋锁来保护其中的数据结构，当我们尝试修改`Sleeplock`中的数据的时候，我们需要首先获取自旋锁，之后立马释放自旋锁。保证在多进程环境下数据的一致性。

**获取Sleeplock的实现acquiresleep**

```c
void acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk); // 获取自旋锁
  while(lk->locked)
  {
    sleep(lk, & lk->lk); //如果sleeplock已经被获取，在sleep中修改进程状态为睡眠调用调度器
  }
  lk->locked = 1; // 获取锁成功，将状态改为locked
  lk->pid = myproc()->pid;
  release(&lk->lk);// 释放自旋锁
}
```

**逻辑解析**： 1. 快速获取自旋锁：由于自旋锁只用于保护`sleeplock`的内部状态，因此很快就会被释放 2. 进入阻塞状态等待锁被释放：当`Sleeplock`已经被锁定的时候，会调用`Sleep`系统调用，将当前进程处于睡眠状态并释放锁 3. 修改`Sleeplock`的状态

**释放`sleeplock`的实现`releasesleep`**

```c
void
releasesleep(struct sleeplock *lk)
{
  acquire(&lk->lk); //获取自旋锁
  lk->locked = 0; //将sleeplock状态置空
  lk->pid = 0;
  wakeup(lk); // 唤醒所有sleep的进程
  release(&lk->lk);//释放自旋锁
}
```

# Thread Switching

## 线程概述和线程调度

使用线程的一个原因是 ： 为了从多核机器获取并行加速
那么线程是什么 ？ **所以，线程可以认为是一种在有多个任务时简化编程的抽象。一个线程可以认为是串行执行代码的单元。如果你写了一个程序只是按顺序执行代码，那么你可以认为这个程序就是个单线程程序，这是对于线程的一种宽松的定义。虽然人们对于线程有很多不同的定义，在这里，我们认为线程就是单个串行执行代码的单元，它只占用一个CPU并且以普通的方式一个接一个的执行指令。**
所以 ， THREAD - one serial execution

除此之外，线程还具有状态，我们可以随时保存线程的状态并暂停线程的运行，并在之后通过恢复状态来恢复线程的运行。线程的状态包含了三个部分：

- PC
- register
- 程序的Stack。通常来说每个线程都有属于自己的Stack，Stack记录了函数调用的记录，并反映了当前线程的执行点。

多线程的并行运行主要有两个策略 ：

- 第一个策略是在多核处理器上使用多个CPU ， 那么显然，当有成百上前的线程时，无法使用那么多的CPU并行运行
- 所以第二个策略是主要的策略，  **how each CPU is going to switch among different threads**

Thread will share memory , **so we must use lock**
xv6内核共享了内存 ，并且XV6支持内核线程的概念，对于每个用户进程都有一个内核线程来执行来自用户进程的系统调用。所有的内核线程都共享了内核内存，所以XV6的内核线程的确会共享内存。

另一方面，XV6还有另外一种线程。每一个用户进程都有独立的内存地址空间（注，详见4.2），并且包含了一个线程，这个线程控制了用户进程代码指令的执行。所以XV6中的用户线程之间没有共享内存，你可以有多个用户进程，但是每个用户进程都是拥有一个线程的独立地址空间。XV6中的进程不会共享内存。

实现内核中的线程系统存在以下挑战：

- 第一个是如何实现线程间的切换。我们将会看到XV6为每个CPU核都创建了一个线程调度器
- 第二个挑战是，当你想要实际实现从一个线程切换到另一个线程时，你需要保存并恢复线程的状态，所以需要决定线程的哪些信息是必须保存的，并且在哪保存它们。
- 最后一个挑战是如何处理运算密集型线程

每个CPU上都有一个属性`Timer interrupts`，这里的基本流程是，定时器中断将CPU控制权给到内核，内核再自愿的出让CPU。
线程会有很多状态 ： 

- RUNNING , 线程当前正在某个CPU上运行
- RUNABLE , 线程当前还没有在某个CPU上运行，但是一旦有空闲的CPU就可以运行
- SLEEPING , 线程在等一些I/O

对于RUNNING线程，它的pc和寄存器位于正在运行它的CPU硬件中。而RUNABLE线程，因为并没有CPU与之关联，所以对于每一个RUNABLE线程，当我们将它从RUNNING转变成RUNABLE时，我们需要将它还在RUNNING时位于CPU的状态拷贝到内存中的某个位置，注意这里不是从内存中的某处进行拷贝，而是从CPU中的寄存器拷贝。我们需要拷贝的信息就是程序计数器（Program Counter）和寄存器。

## Xv6线程切换

当用户程序在运行时，**实际上是用户进程中的一个用户线程在运行**。如果程序执行了一个系统调用 或者因为响应中断走到了内核中，那么相应的用户空间状态会被保存在程序的`trapframe`中，同时**属于这个用户程序的内核线程被激活**。所以首先，用户的PC，寄存器等被保存到了`trapframe`中之后CPU被切换到内核栈上运行，实际上会走到`trampoline和usertrap`代码中。之后内核会运行一段时间处理系统调用或者执行中断处理程序。在处理完成之后，如果需要返回到用户空间，trapframe中保存的用户进程状态会被恢复。

```
这段说了系统调用或中断，从用户程序进入内核 并返回回用户程序
接下来看时钟中断，从用户程序A进入内核最后回到用户程序B
```

当xv6从CC程序的内核线程切换到LS程序的内核线程时：

1. XV6会首先会将CC程序的内核线程的内核寄存器保存在一个context对象中
2. 类似的，因为要切换到LS，所以LS程序现在的状态必然是RUNABLE，表明LS程序之前运行了一半。这同时也意味着LS程序的用户空间状态已经保存在了对应的trapframe中，更重要的是，LS程序的内核线程对应的内核寄存器也已经保存在对应的context对象中。所以接下来，xv6会恢复LS程序的内核线程的context对象，也就是恢复内核线程的寄存器。
3. 之后LS会继续在它的内核线程栈上，完成它的中断处理程序
4. 然后通过恢复LS程序trapframe中的用户进程状态，返回到用户空间的LS程序中
5. 最后恢复执行LS

**BUT 实际上，要复杂很多**
我们仍然假设有进程P1在运行 ，P2是RUNABLE。假设有两个CPU核，这意味着在硬件我们有CPU0 , CPU1

更完整的情况是：

1. 首先与我之前介绍的一样，一个定时器中断强迫CPU从用户空间进程切换到内核，trampoline代码将用户寄存器保存于用户进程对应的trapframe对象中；
2. 之后在内核中运行usertrap，来实际执行相应的中断处理程序。**这时，CPU正在进程P1的内核线程和内核栈上，执行内核中普通的C代码；**
3. 假设进程P1对应的内核线程决定它想出让CPU，它会做很多工作，这个我们稍后会看，但是**最后它会调用swtch函数** 
4. swtch函数会保存用户进程P1对应内核线程的寄存器至context对象。所以目前为止有两类寄存器：用户寄存器存在trapframe中，内核线程的寄存器存在context中。

但是，实际上**swtch函数并不是直接从一个内核线程切换到另一个内核线程。** 在Xv6中，一个CPU上运行的内核线程可以直接切换到的是这个CPU对应的**调度器线程**。所以如果我们运行在CPU0，**swtch函数会恢复之前为CPU0的调度器线程保存的寄存器和stack pointer** ,之后就在调度器线程的context下执行schedulder函数

在schedulder函数中会做一些清理工作，例如将进程P1设置成RUNABLE状态。之后再通过进程表单找到下一个RUNABLE进程。假设找到的下一个进程是P2（虽然有可能找到的还是p1），schedulder函数会再次调用swtch函数：

1. 先保存自己的寄存器到调度器线程的context对象
2. 找到进程P2之前保存的context，恢复其中的寄存器
3. 返回到P2的系统调用或中断处理程序中
4. 当P2的内核程序执行完成之后，trapframe中的用户寄存器会恢复

在这里，每一个内核线程都有一个context对象，**用户进程的内核线程**的context保存在用户进程对应的proc结构体中。
每一个调度器线程的context对象保存在CPU结构体中。

每个CPU核在一个时间只会做一件事，只会运行一个线程，它要么是运行用户进程的线程，要么运行内核线程，要么运行这个CPU核对应的调度器线程。

在XV6的代码中，context对象总是由swtch函数产生，所以context总是保存了内核线程在执行swtch函数时的状态。当我们在恢复一个内核线程时，对于刚恢复的线程所做的第一件事情就是从之前的swtch函数中返回（注，有点抽象，后面有代码分析）。

## CODE

```c
enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

struct proc{
struct spinlock lock; 
struct trapframe *trapframe; //保存了用户级别的寄存器
struct context context; //保存内核线程寄存器
uint64 kstack;               // Virtual address of kernel stack
 enum procstate state;        // Process state
}
```

当发生一个时钟中断，进入`trap.c`，之后会来到`yield`

```c
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield(); //进程中让出CPU的第一步 切换到调度器
```

```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

yield做了几件事 ， 它获取这个进程的锁 ， 接着改变了进程状态，然后`sched`

```c
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context); //当前上下文保存在p->context , 切换到cpu的上下文
  mycpu()->intena = intena;
}
```

它几乎什么都不做，它做了一些可用行检查 ， 然后来到`swtch`

```c
void            swtch(struct context*, struct context*);
```

```asm
# Context switch
#
#   void swtch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.	
.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        ret
```

将保存目前的内核线程寄存器到`p->context`中 ， 更换到核心调度器的线程状态，并继续运行这个核心的调度器线程
![image-20250202173253346](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250202173253346.png)

最有趣的就是ra（Return Address）寄存器，因为ra寄存器保存的是当前函数的返回地址，所以调度器线程中的代码会返回到ra寄存器中的地址。通过查看kernel.asm，我们可以知道这个地址的内容是什么。也可以在gdb中输入“x/i 0x80001f2e”进行查看。输出中包含了地址中的指令和指令所在的函数名。**所以我们将要返回到scheduler函数中**。

```
这时候有一个问题是为什么swtch只保存了14个寄存器？
因为switch是按照一个普通函数来调用的，对于有些寄存器，swtch函数的调用者默认swtch函数会做修改，所以调用者已经在自己的栈上保存了这些寄存器，当函数返回时，这些寄存器会自动恢复。所以swtch函数里只需要保存Callee Saved Register就行。
```

```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context); //当进程被找到后，swtch将ra转换到那个进程之前的位置，q上是swtch里面的swtch的下一行，然后那个进程拿回trampoline跳进用户态去执行

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0; //一般来说 ，之前的swtch将ra寄存器转换到这里 然后继续找下一个进程
      }
      release(&p->lock);
    }
  }
}
```

这就是`scheduler`函数 ，根据代码，找到下一个RUNABLE然后，swtch之后，把C->proc = 0

# Scheduling

​	Any operating system is likely to run with more  processes than the computer has CPUs, so a plan is needed to time-share the CPUs among the processes.A common approach is to provide each process with the illusion that it has its own virtual CPU by *multiplexing* the processes onto the hardware CPUs. 

## Multiplexing

  Xv6 multiplexes by switching each CPU from one process to another in two situations.

- First, xv6’s sleep and wakeup mechanism switches when a process waits for device or pipe I/O to complete, or waits for a child to exit, or waits in the sleep system call.
- Second, xv6 periodically forces a switch to cope with processes that compute for long periods without sleeping.

This multiplexing creates the illusion that each process has its own CPU, much as xv6 uses the memory allocator and hardware page tables to create the illusion that each process has its own memory.

​	Implementing multiplexing poses a few chanllenges. First how to switch from one process to  another? Although the idea of context switching is simple , **the implementation is some of the most opaque(不透明) code in xv6.**  Second ,  how to force switches in a way that is transparent to user processes? **Xv6 uses the standard technique in which a hardware timer’s interrupts drive context switches.**  Third, all of the CPUs switch among the same shared set of processes, **and a locking plan is necessary to avoid races.**  Fourth, **a process’s memory and other resources must be freed when the process exits**, but it cannot do all of this itself because (for example) it can’t free its own kernel stack while still using it. Fifth, **each core of a multi-core machine must remember which process it is executing** so that system calls affect the correct process’s kernel state. 最后，`sleep`允许一个进程放弃CPU，`wakeup`允许另一个进程唤醒第一个进程。需要小心避免导致唤醒通知丢失的竞争。
Xv6试图尽可能简单地解决这些问题，但结果代码很复杂。

## Code : 上下文切换

1. **用户-内核转换**：当一个用户进程（旧进程）需要执行一个系统调用或者因为某种中断（比如时钟中断、I/O操作等）进入内核模式时，发生了从用户模式到内核模式的转换。这个过程通常是由系统调用触发的。
2. **上下文切换**：这是指从旧进程的执行上下文（CPU寄存器、栈等）切换到一个新的上下文。这里提到的上下文切换包含两个步骤：
   - 从旧进程的内核线程切换到当前CPU的调度程序线程。
   - 从当前调度程序线程切换到新进程的内核线程。
3. **陷阱返回用户进程**：最后，系统会通过一个"陷阱"（trap）返回到用户模式，这样进程就会继续执行用户代码，而不是内核代码。
4. **内核栈的安全问题**：执行是不安全的：其他一些核心可能会唤醒进程并运行它，而在两个不同的核心上使用同一个栈将是一场灾难，因此xv6调度程序在每个CPU上都有一个专用线程（保存寄存器和栈）。

从一个线程切换到另一个线程需要保存旧线程的CPU寄存器，并恢复新线程先前保存的寄存器。栈指针和程序计数器被保存和恢复的事实意味着CPU将切换栈和执行中的代码。

## Code : mycpu and myproc

Xv6通常需要指向当前进程的proc结构体指针。在单处理器系统上，可以有一个指向当前`proc`的全局变量。但这不能用于多核系统，因为每个核执行的进程不同。解决这个问题的方法是基于每个核心都有自己的寄存器集，从而使用其中一个寄存器来帮助查找每个核心的信息。

Xv6为每个CPU维护一个`struct cpu`，它记录了当前在该CPU上运行的进程，为CPU的调度线程保存寄存器。函数`mycpu` (kernel/proc.c\:60)返回一个指向当前CPU的`struct cpu`的指针。RISC-V给它的CPU编号，给每个CPU一个`hartid`。Xv6确保每个CPU的`hartid`在内核中存储在该CPU的`tp`寄存器中。这允许`mycpu`使用`tp`对一个cpu结构体数组（即`cpus`数组，***kernel/proc.c\***:9）进行索引，以找到正确的那个。

`cpuid`和`mycpu`的返回值很脆弱：如果定时器中断并导致线程让步（yield），然后移动到另一个CPU，以前返回的值将不再正确。为了避免这个问题，xv6要求调用者禁用中断，并且只有在使用完返回的`struct cpu`后才重新启用。



## sleep and wake

```c
acquire(&p->lock)
p->state = RUNBALE
swtch() ----------------------------> swtch()
    								rease(&p->lock)
```

这里进程上锁的原因是 为了 再它state变成RUNBASLE之后，不会被其他调度器调度

当你调用`swtch`时，不允许持有其他锁

**协调**
当线程想要访问I/O， 我们可能会写出这样的代码

```c
while(I/O is empty);
```

一直自旋，但是可能需要等很长时间，CPU利用率低下，于是可以让线程放弃CPU，当事件发生时获得CPU ，也就是 SLEEP / WAKE

```c
sleep(&tx_chan , &uart_tx_lock); //第二个参数是一个锁参数
wake(&tx_chan)
```

通过睡眠通道唤醒

在解释sleep函数为什么需要一个锁使用作为参数传入之前，我们先来看看假设我们有了一个更简单的不带锁作为参数的sleep函数，会有什么样的结果。这里的结果就是**lost wakeup**。

```c
broke_sleep(chan)
p->state = SLEEPIN
p->chan = chan
swtch()
wakeup(chan)
    for(){
        if p->state == SLEEPING && P->chan == chan
            	p->state = RUNBALE
    }
```

```c
void
uartwrite(char buf[] , int n)
{
    acquire(&uart_tx_lock);
    int i =0;
    while(i < n){
        while(tx_done == 0){
            //sleep(&tx_chan , &uart_tx_lock);
            release(&uart_tx_lock);
            // RIGHT HERE -- INTERRUPT 
        	broken_sleep(&tx_chan);
            acquire(&uart_tx_lock);
        }
    	Write(THR , buf[i]);
        i +=1;
        tx_done= 0;
    }
    release(&uart_tx_lock);
}
void
uartintr(void){
    acquire(&uart_tx_lock);
    if(ReadReg(LSR) && LSR_TX_IDLE){
        tx_done = 1;
        wakeup(&tx_chan);
    }
    release(&uart_tx_lock);
}
```

当`make qemu`的时候，只打印了`init : sta`
一旦释放了锁，当前CPU的中断会被重新打开。因为这是一个多核机器，所以中断可能发生在任意一个CPU核。在上面代码标记的位置，其他CPU核上正在执行UART的中断处理程序，并且正在acquire函数中等待当前锁释放。所以一旦锁被释放了，另一个CPU核就会获取锁，并发现UART硬件完成了发送上一个字符，之后会设置tx_done为1，最后再调用wakeup函数，并传入tx_chan。目前为止一切都还好，除了一点：现在写线程还在执行并位于release和broken_sleep之间，也就是写线程还没有进入SLEEPING状态，所以中断处理程序中的wakeup并没有唤醒任何进程，因为还没有任何进程在tx_chan上睡眠。之后写线程会继续运行，调用broken_sleep，将进程状态设置为SLEEPING，保存sleep channel。但是中断已经发生了，wakeup也已经被调用了。所以这次的broken_sleep，没有人会唤醒它，因为wakeup已经发生过了。这就是lost wakeup问题。

**如何解决呢**

```c
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.

  acquire(&p->lock);  //DOC: sleeplock1
  release(lk);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  release(&p->lock);
  acquire(lk);
}
```

多给sleep传入一把锁 ，当release(lk)

# File System

## 概述

xv6文件系统分为七层

| **文件描述符**               | 使用文件系统接口抽象了许多Unix资源（例如，管道、设备、文件等），简化了应用程序员的工作。 |
| ---------------------------- | ------------------------------------------------------------ |
| 路径名(Pathname)             | 提供了分层路径名，如***/usr/rtm/xv6/fs.c\***，并通过递归查找来解析它们 |
| 目录(Directory)              | 将每个目录实现为一种特殊的索引结点，其内容是一系列目录项，每个目录项包含一个文件名和索引号。 |
| 索引结点(Inode)              | 提供单独的文件，每个文件表示为一个索引结点，其中包含唯一的索引号和一些保存文件数据的块 |
| 日志(Logging)                | 允许更高层在一次事务（transaction）中将更新包装到多个块，并确保在遇到崩溃时自动更新这些块（即，所有块都已更新或无更新） |
| 缓冲区高速缓存(Buffer cache) | 缓存磁盘块并同步对它们的访问，确保每次只有一个内核进程可以修改存储在任何特定块中的数据 |
| 磁盘(Disk)                   | 磁盘层读取和写入virtio硬盘上的块                             |

**Virtio硬盘**是一种通过虚拟化技术实现的硬盘设备，它被设计为在虚拟机（VM）与宿主机之间进行高效的 I/O 操作。

文件系统必须有将索引节点和内容块存储在磁盘上哪些位置的方案。为此，xv6将**磁盘**划分为几个部分，如图8.2所示。文件系统不使用块0（它保存引导扇区）。块1称为超级块：它包含有关文件系统的元数据（文件系统大小（以块为单位）、数据块数、索引节点数和日志中的块数）。从2开始的块保存日志。日志之后是索引节点，每个块有多个索引节点。然后是位图块，跟踪正在使用的数据块。其余的块是数据块：每个都要么在位图块中标记为空闲，要么保存文件或目录的内容。超级块由一个名为`mkfs`的单独的程序填充，该程序构建初始文件系统。
![image-20250207201105531](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250207201105531.png)



## Buffer cache层 and Code

Buffer cache有两个任务：

1. 同步对磁盘块的访问，以确保磁盘块在内存中只有一个副本，并且一次只有一个内核线程使用该副本
2. 缓存常用块，以便不需要从慢速磁盘重新读取它们。代码在***bio.c\***中。

先利用buf数组固定好了每个节点的位置，再用链表连接 ： **所有节点的prev都指向head ，next指向上一个节点** 

缓冲区有两个与之关联的状态字段。字段`valid`表示缓冲区是否包含块的副本。字段`disk`表示缓冲区内容是否已交给磁盘，这可能会更改缓冲区（例如，将数据从磁盘写入`data`）。

`Bread`（***kernel/bio.c\***:93）调用`bget`为给定扇区（***kernel/bio.c\***:97）获取缓冲区。

```c
/*
在缓冲区缓存中查找设备 dev 上的块。
如果没有找到，则分配一个缓冲区。
无论是哪种情况，都返回已锁定的缓冲区。
*/
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){ //这个for表示从最后一个开始找，往前面一直找，直到遍历到头节点
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) { 
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}

// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0); //如果锁定的缓冲区没有找到缓存，写入缓存
    b->valid = 1;
  }
  return b;
}
```

每个磁盘扇区最多有一个缓存缓冲区是非常重要的，并且因为文件系统使用缓冲区上的锁进行同步，可以确保读者看到写操作。

一旦`bread`读取了磁盘（如果需要）并将缓冲区返回给其调用者，调用者就可以独占使用缓冲区，并可以读取或写入数据字节。如果调用者确实修改了缓冲区，则必须在释放缓冲区之前调用`bwrite`将更改的数据写入磁盘。`Bwrite`（***kernel/bio.c\***:107）调用`virtio_disk_rw`与磁盘硬件对话。

```c
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}
```

接下来就需要释放掉这个缓冲区！
不过在此之前需要回答之前的一个疑问 ： **为什么使用两个锁？**

```
一个是大的链表锁 ，当你寻找缓存的时候，要把大的链表锁住
另一个是小的缓存锁，这里用睡眠锁提高效率， 当你找到后，你需要把大的链表锁释放 ， 然后持有这个小的锁，对这个缓冲区进行读写操作，直到最后要释放的时候 ， 这里可以保证 ： 一次只有一个内核线程使用该副本
```

这里同样也遇到了LRU算法 ， 先来看释放缓冲区代码
Buffer cache为新块回收最近使用最少的缓冲区。这样做的原因是认为最近使用最少的缓冲区是最不可能近期再次使用的缓冲区。

```c
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock); //先释放掉小锁 ， 根据LRU算法，需要对链表进行重拍，于是拿到大锁

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```

这里把b放到链表头部（head->next head->prev 所指向的位置）

## inode

一、 inode是什么？

理解inode，要从文件储存说起。
文件储存在硬盘上，硬盘的最小存储单位叫做“扇区” 。 每个扇区储存512字节
操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性连续读取多个扇区，即一次性读取一个"块"（block）。这种由多个扇区组成的"块"，是文件存取的最小单位。"块"的大小，最常见的是4KB，即连续八个 sector组成一个 block。
文件数据都储存在“块”中，那么我们还必须找到一个地方储存文件的元信息 ： 比如文件的创建者，文件的创建日期，文件的大小等。 这种储存文件元信息的区域叫做`inode` ， 中文翻译为 ： 索引节点
每个文件都有一个对应的inode ， 里面包含了与该文件有关的一些信息。

二、inode的内容

```
文件的字节数
文件拥有者的User ID
文件的读 写 执行权限
文件的时间戳
链接数，即有多少文件名指向这个inode
文件数据block的位置
```

可以用stat命令，查看某个文件的inode信息：

```shell
stat example.txt
```

```shell
╰─ stat time.txt                                                                                                     ─╯
  File: time.txt
  Size: 2               Blocks: 0          IO Block: 4096   regular file
Device: 54h/84d Inode: 106679016174382981  Links: 1
Access: (0777/-rwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2025-02-09 15:34:04.809137400 +0800
Modify: 2025-02-09 15:34:04.809137400 +0800
Change: 2025-02-09 15:34:04.809137400 +0800
 Birth: -
```

总之，除了文件名以外的所有文件信息，都存在inode之中。至于为什么没有文件名，下文会有详细解释。

三、 inode的大小

inode也会消耗硬盘的空间 ，所以硬盘格式化的时候，操作系统自动将硬盘分成两个区域。一个是数据区，存放文件数据；另一个是inode区（inode table），存放inode所包含的信息。每个inode节点的大小，一般是128字节或256字节。inode节点的总数，在格式化时就给定，一般是每1KB或每2KB就设置一个inode。假定在一块1GB的硬盘中，每个inode节点的大小为128字节，每1KB就设置一个inode，那么inode table的大小就会达到128MB，占整块硬盘的12.8%。

查看每个硬盘分区的inode总数和已经使用的数量，可以使用df命令。

由于每个文件都必须有一个inode，因此有可能发生inode已经用光，但是硬盘还未存满的情况。**这时，就无法在硬盘上创建新文件。**

磁盘上的inode由`struct dinode`定义。字段`type`区分文件、目录和特殊文件（设备）。`type`为零表示磁盘inode是空闲的。字段`nlink`统计引用此inode的目录条目数，以便识别何时应释放磁盘上的inode及其数据块。字段`size`记录文件中内容的字节数。`addrs`数组记录保存文件内容的磁盘块的块号。

`struct inode`是磁盘上的`struct dinode`的内存副本。ref字段统计引用内存中inode的指针数量，如果ref为0，内核讲从内存中丢弃该inode。

xv6中的inode有四种锁或类似锁的机制。

**目录中的inode**

```
dinode->type = T_DIR;
指向的数据块有很多这样的目录项
0  myfile1  19  //然后根据块编号指回
1  myfile2  20
2  hello.txt  24
3
```



## Code ： Inodes

首先先来看一下`Block allocator`

```c
// Blocks.

// Allocate a zeroed disk block.
static uint
balloc(uint dev) //dev：磁盘设备号 ，指示在哪个磁盘设备上进行分配操作  返回值是分配的块号，表示在磁盘上的块位置
{
  int b, bi, m; //b：当前块的编号 bi当前块中第bi位的编号，用于检查磁盘块的位图  m位图的掩码
  struct buf *bp;

  bp = 0;
  for(b = 0; b < sb.size; b += BPB){//sb.size表示磁盘上的所有块的总数  BPB：每个块位图包含多少个磁盘块
    bp = bread(dev, BBLOCK(b, sb));//这个宏计算出包含块 b 的块位图的块号，通常磁盘块位图是按块分组的。
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);//用掩码来查找空闲块
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        log_write(bp); //将修改后的位图写回磁盘，确保磁盘上该块的状态被更新。
        brelse(bp); //释放缓冲区，可以被再次使用
        bzero(dev, b + bi); //将这个块清零
        return b + bi;
      }
    }
    brelse(bp); 
  }
  panic("balloc: out of blocks");
}
```

这段代码是一个文件系统中用于分配一个未使用的磁盘块的函数 `balloc`。具体的作用是从磁盘设备中分配一个零化的（空白的）块，并将该块标记为“已使用”。

在xv6中，有两种inode 一种是磁盘中的inode ,里面包括了文件类型，大小，链接数量，以及data区的blockno ， 在code中定义为dinode.  还有一种inode，就是内存中储存的inode 

```c
struct dinode{
    short type; //File type
    short major; //Major device number
    short mino; //Minor device number
    short nlink; //链接
    uint size; //Size of file
    uint addrs[NDIRECT + 1]; //Data block addressses
}
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number //用来定位是哪一个inode
  int ref;            // Reference count 
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // dinode的数据的拷贝
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

```c
struct{
    struct spinlock lock;
    struct inode inode[NINODE];
}icache; //inode 集合，完全类似于bcache
void init() //初始化集合
{
    int i =0;
    initlock(&icache.lock , "icache");
    for(i = 0;i<NINODE;++i){
        initsleeplock(&icache.inode[i].lock , "inode");
    }
}
static struct inode*
iget(uint dev , uint inum){
    struct inode *ip , *empty;
    acquire(&icache.lock);
    //Is the inode already cached?
	empty = 0;
    for(ip = &icache.inode[0];ip<&icache.inode[NINODE];ip++){
        if(ip->ref >0 && ip->dev == dev && ip->inum ==inum){
            //如果找到了这个inode当前正在被使用，直接将引用加一返回
            ip->ref++;
            release(&icache.lock);
            return ip;
        }
        //找到第一个没被使用的inode块，作为备用返回
        if(empty == 0 && ip->ref == 0){
            empty = ip;
        }
    }
   	//遍历结束没有找到已缓存的，那么就用上面的备用
  // Recycle an inode cache entry.
  if(empty == 0)
    panic("iget: no inodes");
   
    ip = empty;
    ip->dev = dev;
    ip->inum = inum;
    ip ->ref = 1;
    ip -> valid = 0; //这里0 代表需要从磁盘中拷贝，那么如何拷贝呢？ 是否能够跟buf一样使用disk层提供的硬件接口？ 后面我们会用到！
    release(&icache.lock);
    return ip;
}
```

这段代码实现了一个简单的**inode缓存管理**系统，类似于**块缓存（block cache）**，用于管理和缓存磁盘上的 inode 数据。它通过一个名为 `icache` 的缓存来存储 inode，当文件系统需要访问 inode 时，首先会检查该 inode 是否已经缓存。如果缓存中没有，就从磁盘中读取并放入缓存。

```c
//这里就是要在磁盘中创建一个新的inode，简单了来说就是简单了来说就是找到一个type为0的inode然后将其赋予我们要创建的类型，file/dir/div
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;
  //根据序号遍历所有的inode，查看是否是满足type = 0
  //很显然这里涉及到将磁盘中数据读取到内存中的步骤，到目前为止（xv6所有的操作）所有的从磁盘中读取数据到内存中
  //我们只能通过buf cache层给我们提供的接口 也就是bread来实现，这里也是一样。
  for(inum = 1; inum < sb.ninodes; inum++){
    //全程是有锁的。
    bp = bread(dev, IBLOCK(inum, sb));  //#define IBLOCK(i, sb)     ((i) / IPB + sb.inodestart)
                                     // #define IPB           (BSIZE / sizeof(struct dinode))
                                     //简单来说就是计算出这个序号的inode在哪一个block中，将这个block读                                      入磁盘
    dip = (struct dinode*)bp->data + inum%IPB; //对IPB取模来得到目前的dinode*
    if(dip->type == 0){  // a free inode 在disk中找到一个free的inode
      memset(dip, 0, sizeof(*dip));
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk，将找到的buf写入disk，注意这里函数中没有begin_op
                      // end_op  那么在开启这个ialloc函数前肯定有这两个函数。
      brelse(bp);
      return iget(dev, inum);//此时我们以及找到了对应的inum号，将内存中icache与这个找到的inode关联。
    }
    brelse(bp);
  }
  panic("ialloc: no inodes");
}
```

这段代码实现了**inode分配**的功能。它的作用是从磁盘中分配一个未使用的inode，并将其标记为已分配，同时返回一个指向该inode的指针

```c
//将对应的修改后的inode，找到dinode，然后写入磁盘
void
iupdate(struct inode * ip)
{
    struct buf * bp;
    struct dinode *dip;
    
    bp = bread(ip->dev , IBLOCK(ip->inum , sb));
    dip = (struct dinode*)bp->data + ip->num % IPB;
    dip->type = ip->type;
      dip->major = ip->major;
      dip->minor = ip->minor;
      dip->nlink = ip->nlink;
      dip->size = ip->size;
    memmove(dip->addrs , ip->addrs , sizeof(ip->addrs));
    log_write(bp); //日志写入
    brelse(bp);
}
```

```c
//将传入的inode上锁，并且有必要的话从磁盘中读取该inode的对应的dinode中，上面也提到过valid为0时该如何操作
void
ilock(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;
​
  if(ip == 0 || ip->ref < 1)
    panic("ilock");
​
  acquiresleep(&ip->lock);  //获取该inode的锁

  if(ip->valid == 0){ //如果是valid==0 表示还没有读取磁盘中的dinode
    bp = bread(ip->dev, IBLOCK(ip->inum, sb)); //还是要通过bread来读取磁盘中的数据
    dip = (struct dinode*)bp->data + ip->inum%IPB;
    ip->type = dip->type;
    ip->major = dip->major;
    ip->minor = dip->minor;
    ip->nlink = dip->nlink;
    ip->size = dip->size;
    memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
    brelse(bp);
    ip->valid = 1;
    if(ip->type == 0)
      panic("ilock: no type");
  }
}
```

这段代码实现了 **inode 锁定** 操作。具体来说，`ilock` 函数的作用是 **锁定一个 inode**，并确保该 inode 在内存中是有效的。如果 inode 数据尚未加载到内存中，它会从磁盘加载该 inode 的数据。其核心目的是确保在对 inode 进行操作时，能够保持数据一致性并防止并发访问冲突。

之后还有一个inode中比较重要的内容，在lecture中也提到了，就是inode布局。 inode中的addrs数组里面存储的是inode中指向真正储存数据的block num. 接下来介绍一些跟这个有关的函数

```c
//接收一个cache Inode ,和一个bn这个inode中储存的第几个block

//每个inode关联的数据存储在磁盘上的块中 ， 前NDIRECT个块号在ip->addrs[]中 ， 后NINDIRECT个块在ip->addrs[NDIRECT]中
// 返回 inode ip 中第 n 个数据块的磁盘块地址。
// 如果没有这样的块，bmap 会分配一个新的块。
static uint
bmap(struct inode * ip , uint bn)
{
	uint addr , *a;
    struct buf * bp;
    if(bn < NDIRECT){ //小于12 说明是直接映射
        if((addr = ip->addrs[bn]) == 0){
			//如果这个位置存储的值是0，代表这个块还没有被映射
            ip->addrs[bn] = addr = balloc(ip->dev);
        }
        return addr;
    }
    bn -= NDIRECT; //如果大于等于12,那么先减去12 代表的就是在第十二块中对应的第几个
    if(bn < NINDIRECT){//当然这个bn必须要小于256
        if( (addr = ip->addrs[NDIRECT]) == 0)//如果这个代表第12个块的块是0，说明没有被分配，那么就分配一个
            ip->addrs[NDIRECT] = addr = balloc(ip->dev);
        bp = bread(ip->dev , addr); //根据这个地址获得这块block的buf
        a = (uint *)bp->data; //读取这个块内容
        if((addr = a[bn]) == 0){            //读取的内容是0仍然重复的创建块
      		a[bn] = addr = balloc(ip->dev);
     		 log_write(bp);                    //此时这个块已经创建好了，同时buf也是修改过的将这个buf写入日志中
    		}
        brelse(bp);
        return addr;
	}
    panic("bmap: out of range");
}
```

**疑问**

```
那两个宏是什么，如何计算
位图的表现是什么样子的
```



## 块IO + 索引节点操作

### 盘块读写

由于磁盘设备访问太慢， 于是在内存中有一个`bcache`块缓存
只有第一次写入内存 ，或者缓存不命中要写回磁盘才会与磁盘交互

```
bcache 1 1 1 1 1 1 1 1  通过bget查找 分配

磁盘设备 1 1 1 1 1 1 1  调用bread()->iderw()读入内存
```

`bcache`上有很多buf（缓存块 默认30个）  每个buf又是一个结构体，大小为一个块（xv6上是1024） 。

### 超级快

```c
struct superblock {
  uint magic;        // Must be FSMAGIC
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
  uint bmapstart;    // Block number of first free map block
};
```

```c
// Read the super block.
static void
readsb(int dev, struct superblock *sb)
{
  struct buf *bp;

  bp = bread(dev, 1); //读dev这个设备上的1号盘块 因为超级块的编号为1
  memmove(sb, bp->data, sizeof(*sb));//把块缓存的内容拷贝到 sb上面
  brelse(bp);
}
```

### 数据盘块与位图

- 位图操作关联于superblock
- BBLOCK(b , sb); 定位 “第b个数据盘块对应的位图块中的编号“

在`fs.c`中的块操作函数

- `bzero` 将设备上的指定块清零
- `balloc` 分配使用（并且含内容清零）一个盘块 （然后设置位图中为1）
- `bfree` 释放（并且内容清零）一个盘块 （然后设置位图中为0）

==超级块/inode表/位图/文件数据盘块== 的访问 ： **都依赖于bio.c中的块操作**

### 索引节点的操作

- 索引节点自身的元数据操作
  - ialloc() 分配一个索引节点，在磁盘上找到一个空闲的索引节点拿出来用
  - iget() 找到空闲一个内存索引节点 ，配合ialloc使用
  - update() 内存中的索引节点写回磁盘
  - idup() 如果一个文件增加一个链接 ， 那么增加一个引用
  - iput()  减少一个引用 ， 为0的使用删除文件内容 -> itrunc()
  - stati() 读索引节点信息
  - ilock() 锁
  - IBLOCK(i)  得到第i个索引节点在哪个盘块
- 所管理的文件数据的读写操作
  - readi()  writei（）
  - bmap() 基于inode索引来定位 数据盘块

## file操作 + FS的系统调用

### 文件操作

元数据操作 ： filealloc , diledup , dileclose , filestat

```c
// Allocate a file structure.
struct file*
filealloc(void)
{
  struct file *f;
  acquire(&ftable.lock);
  for(f = ftable.file; f < ftable.file + NFILE; f++){
    if(f->ref == 0){
      f->ref = 1;
      release(&ftable.lock);
      return f;
    }
  }
  release(&ftable.lock);
  return 0;
}
```

```c
//读inode中的文件
if(f->type == FD_INODE){
    ilock(f->ip);
    if((r = readi(f->ip, 1, addr, f->off, n)) > 0) // 由readi读入
      f->off += r;
    iunlock(f->ip);
  }
```

- 目录查找 —— dirlookup()  skipelem() (==一个目录文件内== ， 在同一级目录的操作)

- 文件定位（全路径）给一个完整的路径，从根目录开始找目标文件
  - namei() 直接找到目标文件 `ip = namei(old) 查找并返回路径 old 所对应的 inode。`
  
  - namex()  找路径父目录
  
  - nameiparent() ==(借助于 skipelem取得下一级子目录 ， 再借助于 dirlookup 查找下一级子目录中目标文件在不在)==  
  
    `dp = nameparent(new , name) 查找new的父目录inode同时将文件名存储在name中`
    `比如new路径是 /a/b/e.txt name里面存储e.txt , inode是父目录/a/b的inode`
  
- 创建和删除 —— dirlink() ， 删除（inode.nlin + 1 或 -1 ， 如果为0真的删除）

  `dirlink是一个在指定目录中创建目录项的函数 `
  `dirlink(dp , name , ip->inum) 把dp的这个目录下创建一个文件name,然后inode是ip`

```c
struct inode* //在某一级目录内，查找
dirlookup(struct inode *dp, char *name, uint *poff)
{
  uint off, inum;
  struct dirent de; //目录项

  if(dp->type != T_DIR) //必须是目录文件
    panic("dirlookup not DIR");

  for(off = 0; off < dp->size; off += sizeof(de)){ //逐个目录项处理
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de)) //读入当前目录项
      panic("dirlookup read");
    if(de.inum == 0)
      continue;
    if(namecmp(name, de.name) == 0){ //找到名字相等的目录项
      // entry matches path element
      if(poff)
        *poff = off;
      inum = de.inum; //确定索引节点号
      return iget(dp->dev, inum); //iget得到
    }
  }

  return 0;
}
```

 `dirlookup` 函数的目的是在指定的目录中查找与给定名称 `name` 相匹配的目录项。它会遍历目录中的所有目录项，找到匹配的目录项后，返回对应的 inode（索引节点）。该函数用于文件系统的目录操作。

```c
// Write a new directory entry (name, inum) into the directory dp.
int
dirlink(struct inode *dp, char *name, uint inum)
{
  int off;
  struct dirent de;
  struct inode *ip;

  // Check that name is not present.
  if((ip = dirlookup(dp, name, 0)) != 0){ //不能重名
    iput(ip);
    return -1;
  }

  // Look for an empty dirent.
  for(off = 0; off < dp->size; off += sizeof(de)){ //找一个空闲目录项
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlink read");
    if(de.inum == 0)
      break;
  }

  strncpy(de.name, name, DIRSIZ); //目录项中写入文件名字符串
  de.inum = inum; //目录项中记录对应的inode
  if(writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de)) //写出到磁盘
    panic("dirlink");

  return 0;
}
```

### 文件操作的系统调用接口

- 文件的打开与关闭 : `sys_open()`  `sys_close()` `sys_dup()`
- 文件读写 `sys_read` `sys_write`
- 目录操作`sys_mkdir` `sys_chdir`  `sys_mknod` `sys_link`  `sys_unlink`
- 其他 `sys_exec` `sys_pipe`

```
对于open这个系统调用 ， 如果没有的话就创建一个，有的话就打开，这里重要的一点是 不仅需要inode这个物理结构，还需要一个 ftable.file[]（表示已打开文件列表）这个逻辑结构

下层分配物理结构，inode的一些操作
上层写入到ftable.file[]中，然后这个文件列表写入进程 x的proc->ofile[]，然后给到了fd文件描述符
上下层关联代码： f->ip  = ip;
```

```
fileread() -> readi() ->bread() ->iderw()
文件			索引节点	块  		磁盘
```

## 日志层 - 设备 - 管道

所有涉及盘块写的操作都经过**日志层**

- 先写入日志区
- 然后在合适的时机写入到指定位置
- 清除日志

系统启动前检查日志，并尝试完成
**日志工作原理**
`st.logstart`是日志起始点 ， 有一个日志头表 `struct logheader`

```c
struct loghreader{
    int n;
    int block[LOGSIZE]; //block最多可以记录30个日志
}
```

`begin_op() / end_op() 用于同步控制 ， 不能在commit进行写`

日志系统写盘过程

1. 进程从`begin_op()`进入 写入缓存块，`log_write()将b->blockno = x 写入日志头表block[]`
2. `end_op`触发`commit()` `commit()->write_log将数据写到日志区`
3. `commit()->write_head()`将内存中的logheader写入到磁盘日志去
4. `commit()->install_trans()将数据从日志区拷贝到目标盘块`
5. commit()清除 log.lh.n = 0

**第一个进程首次运行forkret()时调用`initlog()->recover_from_log()`**

```c
void
log_write(struct buf *b)  //登记到block数组中
{
  int i;
  acquire(&log.lock);
  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorption
      break;
  }
  log.lh.block[i] = b->blockno; //这里
  if (i == log.lh.n) {  // Add new block to log? 
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```

```c
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock); //没人在commit的时候，才能进入，否则睡眠
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```

其他进程在commit时无法进入 ， 通过`begin_op`后，用outstanding计数，防止过多数量的操作，日志无法承担

```c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1; //这个进程要退出了， 所以-1
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){ //如果是最后一个退出的， 那么负责commit
    do_commit = 1;
    log.committing = 1;
  } else { 
    wakeup(&log); //唤醒其他进程
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

### 设备文件

`sys_mknod()`借助`create()`创建设备节点

- ialloc()创建inode设置 type = T_DEV ，填写 `.major` 和`.minor`
- `dirlink`创建目录项，并关联上述设备索引节点

`devsw[]`数组记录个设备的读写操作函数

- devsw[CONSOLE].write = consolewrite;

### 管道文件

`file->type= FD_PIPE`

# lecture - File system

- Abstraction is useful

- Crash safely

- Disk layout

- Preformance

  Storage devices are slow —— 所以尽量避免使用磁盘是很重要的 ，我们会看到很多方法 ， 比如：所有文件系统都有某种缓冲区缓存 ， 并发性（比如你执行路径名查找，其他进程进行别的）

## API example / file system syscall

```c
fd = open("x/y" , _);
write(fd , "abc" , 3);
这里pathname是用户可以看出来的 ， 3没有偏移量下一次就是4
link("x/y , "x/z");使得同一个文件有多个名称

```

## The system structures

**inode** : 这是一个表示文件的对象，独立于名称所以 ，文件信息，独立于名称。 inode只是一个数字，文件系统内部引用inode是通过编号而不是实际路径名称。  inode必须有链接计数，为了记录名字的数目，它们指向特定的inode，文件只能 当链接计数为0时和open fd（打开的文件描述符）为0时才能删除

## FS layers

| names                | memory |
| -------------------- | ------ |
| inode  -- read write | memory |
| icache               | memory |
| logging              | memory |
| buf cache            | memory |
| disk                 | device |

**storage devices**

SSD固态硬盘
HDD磁盘

sector(扇区)  blocks（块） 历史上扇区是最小单位 ，磁盘可以读取或写入，所以通常是512字节。而块通常是操作系统或文件系统的说法，在xv6中是1024字节

**Disk layout**
磁盘是一个巨大的块数组 通常块0不使用，用于引导扇区以引导操作系统     块1是超级块，超级块描述了文件系统 
logging块， inodes块，bitmap块也叫做元数据库，为文件系统存储了元数据

比如我们像访问inode 10 -> 32 + 10*64 / 1024 （==一个block是1024字节==）

- block0要么没有用，要么被用作boot sector来启动操作系统。
- block1通常被称为super block，它描述了文件系统。它可能包含磁盘上有多少个block共同构成了文件系统这样的信息。我们之后会看到XV6在里面会存更多的信息，你可以通过block1构造出大部分的文件系统信息。
- 在XV6中，log从block2开始，到block32结束。实际上log的大小可能不同，这里在super block中会定义log就是30个block。
- 接下来在block32到block45之间，XV6存储了inode。我之前说过多个inode会打包存在一个block中，一个inode是64字节。
- 之后是bitmap block，这是我们构建文件系统的默认方法，它只占据一个block。它记录了数据block是否空闲。
- 之后就全是数据block了，数据block存储了文件的内容和目录的内容。

**inode**

- 通常来说它有一个type字段，表明inode是文件还是目录。
- nlink字段，也就是link计数器，用来跟踪究竟有多少文件名指向了当前的inode。
- size字段，表明了文件数据有多少个字节。
- 不同文件系统中的表达方式可能不一样，不过在XV6中接下来是一些block的编号，例如编号0，编号1，等等。XV6的inode中总共有12个block编号。这些被称为direct block number。这12个block编号指向了构成文件的前12个block。举个例子，如果文件只有2个字节，那么只会有一个block编号0，它包含的数字是磁盘上文件前2个字节的block的位置。
- 之后还有一个indirect block number，它对应了磁盘上一个block，这个block包含了256个block number，这256个block number包含了文件的数据。所以inode中block number 0到block number 11都是direct block number，而block number 12保存的indirect block number指向了另一个block。

![image-20250210162039378](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20250210162039378.png)

max file size : (256 + 12) * 1024 byts =  268kb


ok ， now 让我们看看实现read系统调用！
我们从操作系统引导开始，要读取字节8000， 如何找出包含字节8000的块编号？
8000/1024 = 7，这意味着第七个块，在直接块编号里的第七个 。 然后8000 % 1024 =832表示offset

**文件夹 / 目录**
目录由目录项组成，而每个条目都有固定的格式。 它在前两个字节中包含inode编号 ，在剩余14个字节中包含文件名。
比如我们想要查找路径名 `/y/x` 从根inode开始，有一个固定的inode编号(1) ,然后查看块，你知道名字 y，所以到达inode所在的块，文件inode1，查看它里面的每个块，看看y是否在其中

一个真正的文件系统可能会使用更复杂的数据结构，让查找变快

对于8000，我们首先除以1024，也就是block的大小，得到大概是7。这意味着第7个block就包含了第8000个字节。所以直接在inode的direct block number中，就包含了第8000个字节的block。为了找到这个字节在第7个block的哪个位置，我们需要用8000对1024求余数，我猜结果是是832。所以为了读取文件的第8000个字节，文件系统查看inode，先用8000除以1024得到block number，然后再用8000对1024求余读取block中对应的字节。

## code

当`make qemu`的时候  可以看到xv6提供了一些关于文件系统的信息

```bash
nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
```

有70个元数据块， 

OK，当我们执行`echo "hi" > x`

```bash
----create the file
write : 33
write : 33
write : 46  first block of the root directory(inode 1) 我们刚刚在根目录中添加了一个条目 使用我们分配的inode
write : 32
write : 33

----write "hi" to file x
write : 45 bitmap 文件系统扫描位图块 为了找出没有使用过的块
write : 595 h 这里应该是找到了第595个块， 然后写入 h  和 i
write : 595 i
write : 33 再次更新inode大小  (size update , bn0)

---write "\n" to file
write : 595 同理写入 \n
write : 33   更新大小
```

# lecture - Crash Recovery

**Problem : crash can lead the on-disk file system to be in an inconsistent state or an incorrect state**
文件系统操作 ： 比如创建文件 ，写文件，都包含了多个步骤的写磁盘操作。这里多个步骤的顺序是

- 分配inode，或者在磁盘上将inode标记为已分配
- 之后更新包含了新文件的目录的data block

如果在这两个步骤之间，操作系统crash了。这时可能会使得文件系统的属性被破坏。如果属性被破环，重启之后就会发生一些不好的事情

**我们这里有多个写磁盘的操作，这些操作必须作为一个原子操作出现在磁盘上。**

## File system logging

针对文件系统crash之后的问题的解决方案。 它有一些好的属性：

- 确保文件系统的系统调用是原子的
- 支持快速恢复。
- 原则上来说 ，它可以非常高效 ， 尽管我们在xv6中看到的实现不是很高效

当你需要更新文件系统时，我们并不是更新文件系统本身。假设我们在内存中缓存了`bitmap block` ， 也就是block 45.当需要更新bitmap时，我们并不是直接写block 45，而是将数据写入到log中，并记录这个更新应该写入到block 45。对于所有的写 block都会有相同的操作，例如更新inode，也会记录一条写block 33的log。

**所以基本上，任何一次写操作都是先写入到log，我们并不是直接写入到block所在的位置，而总是先将写操作写入到log中**

**之后，当文件系统的操作结束后，我们会commit文件系统的操作。这意味着我们需要在log的某个位置记录属于同一个文件系统的操作的个数**

**当我们在log中存储了所有写block的内容时，如果我们要真正执行这些操作，只需要将block从log分区移到文件系统分区。我们知道第一个操作该写入到block 45，我们会直接将数据从log写到block45，第二个操作该写入到block 33，我们会将它写入到block 33，依次类推。**

**一旦完成了，就可以清除log。清除log实际上就是将属于同一个文件系统的操作的个数设置为0。**

这个方法之所以能起作用，就是因为可以确保当发生crash并重启后 ，要么我们将写操作所有相关的block都在文件系统中更新了， 要么 没有更新任何一个block ， 我们永远也不会只写了一部分block。（下面是几种情况）

- 在第1步和第2步之间crash会发生什么？**在重启的时候什么也不会做**，就像系统调用从没有发生过一样，也像crash是在文件系统调用之前发生的一样。这完全可以，并且也是可接受的。
- 在第2步和第3步之间crash会发生什么？在这个时间点，所有的log block都落盘了，因为有commit记录，所以完整的文件系统操作必然已经完成了。我们可以将log block写入到文件系统中相应的位置，这样也不会破坏文件系统。所以这种情况就像系统调用正好在crash之前就完成了。
- 在install（第3步）过程中和第4步之前这段时间crash会发生什么？在下次重启的时候，我们会redo log，我们或许会再次将log block中的数据再次拷贝到文件系统。这样也是没问题的，因为log中的数据是固定的，**我们就算重复写了文件系统，每次写入的数据也是不变的。重复写入并没有任何坏处，因为我们写入的数据可能本来就在文件系统中，所以多次install log完全没问题**。当然在这个时间点，我们不能执行任何文件系统的系统调用。我们应该在重启文件系统之前，在重启或者恢复的过程中完成这里的恢复操作。换句话说，**install log是幂等操作**（注，idempotence，表示执行多次和执行一次效果一样），**你可以执行任意多次，最后的效果都是一样的。**

> 这里有一个问题很好 ： 当我们在在commit Log的时候crash了会发生什么 ？ 此时只提交了一半
>
> 文件系统可以这么假设 ， 单个block的write是原子操作。这里的意思是，如果你执行写操作，要么整个sector被写入，要么不写。commit操作本身只是写log的header ， 如果它成功了只是在commit header 中写入log的长度，例如5，这样我们就知道log的长度为5。这时crash并重启，我们就知道需要重新install 5个block的log。如果commit header没能成功写入磁盘，那这里的数值会是0。我们会认为这一次事务并没有发生过。这里本质上是write ahead rule，它表示logging系统在所有的写操作都记录在log中之前，不能install log。

XV6的log结构很简单， 我们最开始有一个header block，也就是我们的commit record 。 里面包含了

- 数字n代表有效的log block的数量
- 每个log block的实际对应的block编号

之后就是log的数据，也就是每个block的数据，依次为bn0对应的block的数据，bn1对应的block的数据以此类推。这就是log中的内容，并且log也不包含其他内容。

当文件系统在运行时，在内存中也有header block的一份拷贝，拷贝中也包含了n和block编号的数组。这里的block编号数组就是log数据对应的实际block编号，并且相应的block也会缓存在block cache中

## Code

刚刚提到了事务 ， 这意味着文件系统必须标明事务的开始和结束。 在xv6中，以创建文件的`sys_open`为例，每个文件系统操作都有`begin_op`和`end_op`分别表示事务的开始和结束

