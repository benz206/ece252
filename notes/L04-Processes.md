## Processes
Early computers, as well as many modern embedded systems, did exactly one thing, or basically one thing at a time. Back then, programs had all the access they wanted to the resources within the system. Now, we expect the OS to support **multiple programs running concurrently**.

For this to work reliably, the operating system needs a way to manage the complexity, and this has resulted in the notion of a **process**. We've already worked with processes, but most likely we didn't know it at the time.

A **process** is a <u>*program in execution*</u>. It is composed of three things:
1. The **instructions and data** of the program (the executable)
2. The **current state** of the program
3. Any **resources** that are needed to execute the program

Having 2 instances of the same program running counts as **2 separate processes**. Thus you might have 2 windows of the same app open, and they use 2 different processes.

## The Process Control Block
The **Process Control Block (PCB)** will usually have:
- **Identifier**: a unique ID for the process, which increments when a new process is created and resets when the system is rebooted
- **State**: the current [state](#the-five-state-model) of the process
- **Priority**: how important this process is compared to others
- **Program Counter (PC)**
- **Register Data**
- **Memory Pointers**: pointers to the code and data associated with the process, and any memory that the OS has allocated by request
- **I/O Status Information**: any outstanding requests, files, or I/O devices currently assigned to the process
- **Accounting Information**: *optional*, I believe for stats

![[Pasted image 20260918083831.png]]

> [!warning] Missing: what the PCB is and when the PC/registers are actually saved (L04: The Process Control Block)
> The PCB is the **OS's data structure for managing a process**. The kernel creates and updates one per process, and keeps them in memory in some container (e.g. a list). Accounting information is data about the process's **resource usage**.
>
> Most fields are kept up to date constantly, but the **PC and register data are only saved "when needed"**. While the process runs, the live values are in the CPU. When a [trap](L02-InteruptsSysCalls.md#traps) or **process switch** suspends it, the OS saves the PC (so it resumes at exactly the right instruction) and the registers (so the CPU state is restored) into the PCB. Switching from P0 to P1 means saving P0's state into its PCB, then loading P1's state from P1's PCB. This is the same idea as [saving state for interrupts](L02-InteruptsSysCalls.md#the-interrupts), applied to whole processes.

## The Circle of Life
Unlike energy, processes can be **created and destroyed**. Upon creation, the OS will **create a new PCB** for the process and **initialize** the data in it. This means setting the variables to their initial values, setting the initial state, setting the instruction pointer to the first instruction in `main`, and so on.

The PCB will then be added to the **set of PCBs the OS maintains**. After the process completes and is terminated/cleaned up, the OS might collect some data (e.g. accounting summaries) and then remove it from the **active list of processes**.

## Process Creation
Generally speaking, there are **three main events** that can lead to the creation of a process:
1. **System boot-up**
2. **User request**
3. **One process spawns another**

When the computer boots up, the OS is started and will begin creating processes. This is sometimes through something like an `init()` function.

> [!warning] Correction: `init` is a process, not a function (L04: Process Family Tree; L05)
> In UNIX, **`init`** is the **first process** created at boot (PID **1**). It's the ancestor of every other process, like how `Object` is the superclass of every class in Java. An embedded system may create every process it will ever run at boot, but general-purpose OSes also allow the other two routes.

At boot time the OS starts up various processes, some of which will be in the **foreground** (visible to the user) and some in the **background**. A user-visible process could be the login screen; background processes are things like servers that share media on a local network.

The UNIX term for a background process is a **daemon** (e.g. `sshd`, which answers your `ssh` connections).

Users are well known for starting processes whenever they feel like it: basically any time you double-click an icon or run a command.

An already-executing program could also spawn another. OK, it just talks about really obvious examples.

> [!warning] Missing: parent/child terminology (L04: Process Creation)
> When a process spawns another, the spawner is the **parent** and the new one is the **child**. Besides the obvious examples (an email client launching a browser), a program may deliberately split its work into child processes for **parallelism** or **fault tolerance**.

## Process Destruction
Eventually, most processes die:
1. **Normal exit** (*voluntary*)
2. **Error exit** (*voluntary*)
3. **Fatal error** (*involuntary*)
4. **Killed by another process** (*involuntary*)

It talks about examples of the above, but I think they're pretty obvious...

> [!warning] Missing: the non-obvious parts of process destruction (L04: Process Destruction)
> - **Error exit vs. fatal error:** In an error exit, the program *itself* detects a problem (e.g. a missing file) and chooses to quit with an error code. A fatal error (e.g. stack overflow, division by zero) is detected by the **OS**, which sends it to the program. A process can tell the OS it wants to **handle** some errors (like try-catch), and it may survive if it does.
> - **Permission to kill:** you need the **rights** to kill a process. Normally a user can only kill processes they created, unless they're an administrator.
> - **Children don't automatically die with their parents:** in both UNIX and Windows, a parent can outlive its child *and vice versa*. See [orphans](#process-family-tree).

## Process Family Tree
Processes can have **parents** and **children** in a tree-like structure. We call this a **process group**. Certain operations, like `Ctrl+C`, can also be propagated to the entire group, letting each process decide what to do.

Processes also have a **return value**, but like the `main` function in C, we don't always do anything with it.

Usually, when a child process finishes execution, the parent will wait for the value it returns. If the child continues in a state where it doesn't have anything to do, we call it a **zombie**.

Also, if the child's parent dies before the child does, we call the child an **orphan**.

In UNIX, it will be **adopted by the `init` process**, so it just ends when the OS shuts down.

This can sometimes be *intentional*, but not always.

> [!warning] Clarification: hierarchy, process groups, and return codes (L04: Process Family Tree)
> - The hierarchy is a **UNIX** thing. Each process has **exactly one parent** and zero or more children, all the way up to `init`. **Windows has no real hierarchy**: a parent gets a *reference* to its child, but can hand it to another process. A UNIX process can't disinherit a child.
> - A **process group** is a process **plus all its descendants**, not the whole tree.
> - By convention, return value **0 = success**, and anything else is an error whose meaning the parent and child agree on.

> [!warning] Correction: what a zombie actually is (L04: Process Family Tree)
> A zombie is a child that has **already finished executing** but whose **return value hasn't been collected yet** by the parent's `wait`. It's "dead but not gone": its **PCB entry still exists**, and it holds its resources until the value is collected. Once the parent calls `wait`, the child is **reaped** and cleaned up. A zombie isn't sitting around doing nothing; it can't run at all.

> [!warning] Correction: orphans don't live until shutdown (L04: Process Family Tree)
> When `init` adopts an orphan, it **`wait`s on it** (and ignores the return value). So when the orphan finishes, it gets **reaped immediately** instead of becoming a zombie; it does not stick around until the OS shuts down. Intentional orphans are usually **daemons/services** that are spawned to run in the background.

## The Five-State Model
The **five states** a process can be in:
1. **Running**: actively executing right now
2. **Ready**: not running, but ready to execute if selected by the **scheduler**
3. **Blocked**: not running, usually because it needs some external input, either from a user or another process
4. **New**: just created, but not yet added to the list of processes ready to run
5. **Terminated**: finished executing, but not yet **reaped** (i.e. a [zombie](#process-family-tree))

> [!warning] Clarification: Blocked and New (L04: The Five-State Model)
> **Blocked** is broader than input: the process is waiting on *any* event or resource (I/O, memory, user input...) and **can't proceed until it arrives**. It's a separate state so the scheduler doesn't waste CPU time picking a process that can't do anything.
>
> **New** means the OS has done the admin work (assigned an ID, made the PCB) but hasn't **committed to running it** yet, e.g. because it limits the number of concurrent processes. A New process is typically **on disk, not in memory**.

![[Pasted image 20260918085318.png]]

This gives us **8 total transitions**, most of which are similar to what we saw before:
- **Create**: the process is created and enters the *New* state
- **Admit**: a process in the *New* state is added to the list of processes ready to start (the *Ready* state)
- **Dispatch**: a process that is not currently running begins executing and moves to the *Running* state
- **Suspend**: a running program pauses execution, but can still run if allowed, and moves to the *Ready* state
- **Exit**: a running program finishes and moves to the *Terminated* state; its return value is available
- **Block**: a running program requests a resource but doesn't get it, so it must wait
- **Unblock**: a program gets what it asked for and continues
- **Reap**: a terminated program's return value is collected by a `wait`, and its resources can be fully released

There are 2 other ones. I assume one is something like a fatal error, where the OS decides to shut down a process and immediately terminate it.

> [!warning] Correction: Unblock goes to Ready, not straight back to Running (L04: The Five-State Model)
> After Unblock, the process moves to **Ready**. It doesn't "continue" until the scheduler **Dispatches** it again. The only way into Running is **Dispatch**.

> [!warning] Correction: the 2 extra transitions (L04: The Five-State Model)
> The two unshown transitions are extra **Exit** transitions: **Ready → Terminated** and **Blocked → Terminated**. They happen when a process that *isn't running* gets **killed**, by the user or its parent. (A fatal error happens while the process is *running*, so it's covered by the normal Exit.)

## Swapping Processes to Disk
We can expand the five-state model further. One issue that could come up is that we might have multiple processes, but not enough **memory** to fully accommodate all of them.

We can then use **swap**, which is just *disk used as memory*. Unfortunately, this is <u>*very, very slow*</u> relative to normal RAM, so it's done **only when necessary**.

Because the OS does not want to spend time swapping processes in and out of memory, we need a new state, which we can call **swapped**.

Ideally we only swap when we have no other option, but this also raises an issue: what if a process is running and needs to be swapped, or it's blocked and needs to be swapped? This introduces 2 new states, one for each.

> [!warning] Correction: the two swapped states are Ready/Swapped and Blocked/Swapped (L04: Swapping Processes to Disk)
> - The memory problem isn't the PCBs (only a few KB each), it's each process's **stack and heap**, which can be gigabytes.
> - The OS prefers to swap out **Blocked** processes, since they can't run anyway. So a single swapped state means "blocked and not in memory".
> - That isn't enough for two reasons: (1) sometimes we must swap out a **Ready** process (never a *Running* one), and (2) when a swapped process's event happens, the OS needs to know it's now runnable **without swapping it in to check**.
> - So the swapped state is split into **Ready/Swapped** (can run, but is on disk) and **Blocked/Swapped** (can't run, and is on disk). This gives the **seven-state model**.
>
> Comparison sources: [L04 lecture text](../lectures/L04.tex) and [L04 slides](../lectures/L04-slides.tex).

![[Pasted image 20260918090014.png]]

> "The **Admit** transition is modified to show that by default the new process does not start in main memory. Two new transitions, **Swap In** and **Swap Out**, are added to show a process being loaded into main memory and written out to disk respectively. Finally, there is a **second Unblock** transition, where a Blocked/Swapped process gets whatever it was waiting for and moves to the Ready/Swapped state, because it can now run (but is still on disk).
>
> As in the five-state model, there are additional 'Exit' transitions that may happen but are not shown. If a process is killed, for example, regardless of whether it is in memory or on disk, it will move to the Terminated state."
