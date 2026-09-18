## Processes
Early computers, as well as current many modern embedded systems, did exactly one thing or basically one thing at a time. At the time, programs had all the access they wanted to resources within the system. Now, we expect that the OS handles supporting multiple programs running concurrently.
For this to work reliably, the operating system needs a way to manage the complexity and this has resulted in the notion of a process. We've already worked with processes, but most likely we didn't know it at the time.

A **program** is a program in execution. It is composed of three things:
1. The instructions and data of the program (the executable)
2. The current state of the program
3. Any resources that are needed to execute the program

Having 2 instances of the same program running counts as 2 separate processes. Thus you might have like 2 of the same app open, and they use 2 different processes.

## The Process Control Block
The process control block will usually have a:
- Identifier, a unique id to identify the process, increments when a new process is created and resets when the system is rebooted
- State, the current state of the process
- Priority, how important this process is compared to others
- Program counter (PC)
- Register Data
- Memory pointers, pointers to the code as well as data associated with the entire process, and any memory that the OS has allocated by request
- I/O Status Information, any outstanding files, requests, or I/O devices currently assigned to the process
- Accounting Information (optional, I believe for stats)
![[Pasted image 20260918083831.png]]
## The Circle of Life
Unlike energy, processes can be created and destroyed. Upon creation, the OS will create a new PCB for the process and initialize the data in that block. This means setting the variables to their initial values, setting initial state, setting the instruction pointer to the first instruction in main, and so on.

The PCB will then be added to the set of PCBs the OS maintains, after the process completes and is terminated/cleaned up, the OS might collect some data, and then remove it from the active list of processes.
## Process Creation
There are, generally speaking, three main events that can lead to the creation of a process.
1. System boot up
2. User request
3. One process spawns another
When the computer boots up, the OS is started and will begin creating processes. This is sometimes through like a `init()` function

At boot time the OS starts up various processes, some of which will be in the foreground (visible to the user) and some in the background. A user visible process could be like the login screen, background processes are like servers that share media on a local network.

The unix term for a background process is a Daemon.

Users are well known for starting processes whenever they feel like it. Basically anytime you double click an icon or run a command.

An already executing program could also spawn another, e.g. ok it just talks about really obvious examples.
## Process Distribution
Eventually, most processes die.
1. Normal exit (voluntary)
2. Error exit also voluntary
3. Fatal error which is involuntary
4. Killed by another process which is also involuntary
It talks about examples of the above but I think it's pretty obvious...
## Process Family Tree
Processes can have parents and children in almost a tree like structure. We call this a process group, certain operations like `ctrl+c` can also be propagated to the entire group letting each process decide what to do.
Processes will also have a return value but like the main function in c++ we don't always do anything with it.
Usually, when a child process finishes execution, a parent will wait for the data or the value returned. If the child continues in a state where it doesn't have anything to do, we call this a **zombie**.
Also accurately, if the child's parent dies before the child does, we call the child an orphan.
In UNIX, it will be adopted by the init process so it just ends on closure of the OS.
This can sometimes be intentional but not always.
## The Five State Model
The five states a process can be in:
1. Running, actively executing right now
2. Ready, not running, but ready to execute if selected by the scheduler
3. Blocked, not running usually because needs some external input from either a user or another process
4. New, just created but not yet added to the list of processes ready to run
5. Terminated, just finished executing but not yet reaped
![[Pasted image 20260918085318.png]]This gives us 8 total transitions most of which are similar to what we saw before.
- Create, the process is created and enters the new state
- Admin, a process in the new state is added to the list of processes ready to start (in the ready start)
- Dispatch, a process that is not currently running begins executing and moves to the running state
- Suspend, a program pauses execution, but can still run if allowed, and moves to the ready state
- Exit, a running program finishes and moves to the terminated state, its return value is available
- Block, a running program requests a resource but does not get it so it must wait
- Unblock, a program gets what it asks for and continues
- Reap, a terminated program return value is collected by a wait and its resources can fully be released
There are 2 other ones, I assume one is like idk fatal error where the OS decides to shut down a process and immediately just terminate it.

## Swapping Processes to Disk
We can expand the 5 state model with other stuff, one issue that could come up is that we might have multiple processes, but we don't have enough space to fully accomadate all of the space.
We can then use swap memory, which is just disk used as memory. Unfortunately, this is very very slow relative to the normal RAM. This is thus done only when necessary.
Because the OS does not want to spend time swapping processes in and out of memory, we need a new state, which we can call swapped.
Ideally we only swap when we have no other option, but this also raises issue if a process is running and needs to be swapped or if its blocked and needs to be swapped, this thus introduces 2 new states for each.
![[Pasted image 20260918090014.png]]
"The Admit transition is modified to show that by default the new process does not start in main memory. Two new
transitions, Swap In and Swap Out, are added to show a process being loaded into main memory and written out
to disk respectively. Finally, there is a second Unblock transition, where a Blocked/Swapped process gets whatever
it was waiting for and moves to the Ready/Swapped state, because it can now run (but is still on disk).
As in the five-state model, there are additional “Exit” transitions that may happen but are not shown. If a process
is killed, for example, regardless of whether it is in memory or on disk, it will move to the Terminated state.