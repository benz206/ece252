## Inter-Process Communication (IPC)
When 2 or more processes want to **co-ordinate** or **exchange information**, the mechanism for doing so is called ***inter-process communication***, usually just abbreviated as **IPC**. If a process shares data with another process in the system, the OS will also provide facilities to make this possible.

Some basic terminology: the data being transferred is typically referred to as the ***message***, while the process sending that message is the ***sender***. The one receiving the message is the ***receiver***.

Also, obviously, all communicants need to **agree on some sort of standard** (what the message contains and how it's formatted) when transferring the data. How this agreement is reached is <u>*outside the purview of the OS*</u>; most commonly, the sender just publishes a standard online and the receiver's author follows it.

There are 2 types of sending and receiving: **synchronous** (block until it's done) and **asynchronous** (carry on right away):
![[Pasted image 20260921084820.png]]

> [!warning] Missing: the 4 combinations and which one is normal (L06: Inter-Process Communication)
> - **Sync send, sync receive**: the sender blocks until the receiver collects the message; the receiver blocks until a message arrives.
> - **Sync send, async receive**: the sender blocks, but the receiver continues whether or not a message is there. *Very uncommon.*
> - **Async send, sync receive**: the sender continues right away, the receiver waits for the message. **This is the most common**, since the receiver usually needs the message to continue.
> - **Async send, async receive**: neither side waits; the receiver checks for a message and continues either way.
>
> "Async receive" means that if there's no message, the receiver is **told there's nothing** and keeps going.

It is also common for the **async receive** case to send back another message **acknowledging receipt** of the message. (For that acknowledgement, the labels just flip: the original receiver is now the sender.)

A general paradigm for understanding IPC is known as the ***producer-consumer*** problem. This is when the **producer** creates some information which is later used by the **consumer**. For example, the database may produce the data (some records from the database) to be consumed by the shell to be displayed to the user. This is a general problem and applicable to **client-server** situations (e.g. web servers sending pages to browsers).

There are **3 approaches** we will consider for accomplishing IPC:
1. The **file system**
2. **Message passing**
3. **Shared memory**

All of these are common and a system can easily implement them all. There is <u>*no single option that is optimal in every situation*</u>; each has areas of both strength and weakness.

### File System
One way for 2 processes to communicate is through the **file system**. Messages stored in the file system will be **persistent** and **survive a reboot**. It can also be used when the sender and receiver *know nothing about each other*.

They can both agree to write and read at a **specific path/location**. We do have to make sure one process **doesn't overwrite** another's data, but we can often get around this by using **multiple files with unique IDs**.

> [!warning] Missing: the OS is still involved, and the `import/` example (L06: File System)
> The OS still takes part, since it handles **file creation/manipulation** and the **permissions** for who may read and write the file.
>
> Lecture example: a producer writes each XML message as its own file in an agreed-upon **`import/`** directory, and the consumer scans that directory and imports whatever it finds. One process only writes, the other only reads, and as long as the sender picks **distinct file names**, a second message can't overwrite the first before it's picked up.

### Message Passing
**Message passing** is a service provided by the OS, where the sender **gives the message to the OS** and asks that it be delivered to a recipient. There are 2 basic operations: **sending** and **receiving**.

> [!warning] Missing: direct messaging needs the recipient's ID (L06: Message Passing)
> Messages can be of **fixed or variable size**. In the simplest case, the message is sent **directly to the recipient process**, which means the sender must **know that process's ID**. That limitation is what [message queues](#pass-your-message) solve later.

## Using Signals
Signals **do not contain a message**; a signal is more just an **announcement** or **alert/alarm** (like a pager: you only see *that* you were paged).

This means it <u>*can't be used for all IPC scenarios*</u>, but it's still enough for some of them, as long as the recipient already knows what the signal means. The constants (e.g. **`SIGKILL`**) are defined in **`signal.h`**.

> [!warning] Missing: why you should use the names, not the numbers (L06: Using Signals)
> Implementations **don't always agree** on the numbers, especially for higher signals. Writing `SIGKILL` instead of `9` keeps your code portable. (See the [signal table from L05](L05-ProcessesInUnix.md#signals).)

We already learned how to send a signal from the command line. However, there are 2 functions for sending a signal programmatically:

```c
int kill(int pid, int signo);
int raise(int signo);
```

Both functions return **0 on success** and **-1 on failure** (e.g. no such process). The **`raise`** function sends the signal back to the **current process**.

We need to **know the PID** of the recipient, which is fine, but it requires a little negotiation for how the processes find out about each other. A common convention is for services to **register themselves** in some way, which might be as simple as putting a **file on disk in a specific location that contains the PID** (e.g. MySQL writes its PID to `/var/run/mysqld/mysqld.pid`).

We can use `kill` to do some interesting things, like signal all your processes. It depends on the value chosen for the **`pid`** argument:
![[Pasted image 20260921090452.png]]

We can also invoke `kill` with a signal number of **0**, which is the ***null signal***. It doesn't send anything, but can be used to **check if the recipient process exists**.

> [!warning] Missing: how to read the result, and why it's unreliable (L06: Using Signals)
> If the process doesn't exist, `kill` fails and `errno` is set to **`ESRCH`**. The check is of limited use because (1) the process might **exit right after you check**, and (2) **PIDs get reused**, so the same number might now belong to a different process.

A signal can be sent to a given process, but that process can only actually deal with it **when it's running**. A signal is ***generated*** by something and is later ***delivered*** to the recipient. During the time between generation and delivery, we say the signal is ***pending***. The pending signal is then delivered at the **first opportunity** (which might be immediately, if the recipient is currently executing).

Interestingly, for most, but not all, signals, your process can just **refuse to listen**. This is called ***blocking*** signals, and it can be done for everything <u>*except `SIGKILL` and `SIGSTOP`*</u>.

> [!warning] Missing: what happens to a blocked signal (L06: Using Signals)
> A blocked signal **stays pending** until that signal type is unblocked. Blocking is meant to be **temporary**. If the same signal is sent **several times while blocked**, it may only be **delivered once**, depending on the OS.

Signals also have a **default action**. The action taken when a signal is delivered is called the ***disposition*** of the signal. If you don't explicitly change it, the default happens, but we can change it. There are **3 options**: (1) **ignore** it, (2) run a **signal handler**, or (3) run the **default action** (used to *undo* an earlier change, e.g. you ignored it before but no longer want to).

To register a signal handler, the function is:

```c
void (*signal(int signo, void (*handler)(int))) (int);
```

This is difficult to read, but in practice it's pretty simple: it just says "for signal **X**, run function **foo**". The handler must **return `void`** and **take one `int`**:

```c
void sig_handler(int signal_num) {
	// Handle the signal in some way
}
```

> [!warning] Missing: how to actually register the handler (L06: Using Signals)
> ```c
> signal(SIGINT, sig_handler);
> ```
> - The `int` parameter is set to the **number of the signal received**, so one handler can handle several signals and tell them apart.
> - `signal` **returns a pointer to the old handler**, which you could use to restore it later. The simplest usage just ignores the return value.

![[Pasted image 20260921091455.png]]

The contents of a handler are **restricted**, because the handler runs *between two instructions* of your program. We can only use functions that are ***reentrant***: a function that can be interrupted during execution, have **another complete call to the same function** execute, and then finally resume (with everything still okay).

> [!warning] Correction: why `printf`/`malloc`/`free` are unsafe, and blocking is a separate rule (L06: Using Signals)
> Your notes said all of these "could possibly block". The actual reasoning is different:
> - **`printf`, `malloc`, `free`** aren't reentrant. If the signal arrives *in the middle of* `malloc` and the handler also calls `malloc`, memory management can be left in an **invalid state**.
> - **Separately**, avoid **anything that could block** the process (e.g. **reading a file**).

There are tables of what functions are safe to invoke from within a signal handler. In general, what you are looking for is a designation of ***async-signal safe***.

To **block** a signal, **unblock** one, or just find out what the **current state** is, the function is:

```c
int sigprocmask(int how, const sigset_t *set, sigset_t *old_set);
```

The first argument is what we would like to do:
- **`SIG_BLOCK`**: the signals in `set` are **added** to the block list
- **`SIG_UNBLOCK`**: the signals in `set` are **removed** from the block list
- **`SIG_SETMASK`**: `set` is **assigned** to the signal mask (*overwrites* all current values)

The third argument is **optional**. If a pointer is provided, then upon a change to the signal mask, `old_set` is updated to contain the values from **before** the change.

There is also the ability to manage signal disposition in a more advanced way using **`sigaction`**, but this is beyond the scope of the course.

![[Pasted image 20260921092428.png]]

![[Pasted image 20260921092439.png]]

Finally, if you want to pause your program until it's interrupted by a signal, there is the function **`int pause()`**. It **always returns -1** and suspends your program **until a signal handler runs**. This can be useful if we really do need to wait for something.

## Pass Your Message
Earlier, it was mentioned that signals (1) require you to **know the recipient's PID**, and (2) **contain no message**. We can now look at something that overcomes both of these limitations.

To deal with the PID problem, what we would like is ***indirect communication***, where messages are **sent to mailboxes (queues)**. The queue is **owned by the OS**, so it is <u>*persistent and independent of any particular process*</u>. The diagram below shows a simple message queue for communication between processes A and B.

![[Pasted image 20260921093010.png]]

![[Pasted image 20260921093034.png]]

The first step in message passing is to obtain a ***key*** that identifies a specific IPC structure (the queue that we will use). Keys are just **integer values**, so we would like them to be **unique** (or at least close to it).

One method is to generate the key with the **"file to key"** function found in `sys/ipc.h`:

```c
key_t ftok(char *pathname, int proj);
```

The key is generated from the given file name and the value **`proj`**. The file **does have to exist**, because the function uses its **inode** (the structure on disk that contains the file's metadata). The integer argument allows generating **multiple IPC objects from the same file**. There is a very small risk of duplicate keys if you are unlucky, but it's small enough that we don't care.

> [!warning] Missing: `IPC_PRIVATE`, the other way to get a key (L06: Pass Your Message)
> Passing the constant **`IPC_PRIVATE`** where a `key_t` is expected gives a **guaranteed unique** key. It's used when the communicating processes have a **parent-child relationship**, since the child inherits the queue ID from the parent through `fork`. The [final example](#example-parent-sends-a-message-to-the-child) uses this.

Regardless of how we generate the key, we use it to get the queue with:

```c
int msgget(key_t key, int flag);
```

The first parameter is the **key** we previously generated. The **`flag`** parameter starts with the **UNIX permissions** (e.g. `0600`) and can be modified with additional creation options:
- If the queue is being created for the first time, add **`IPC_CREAT`**
- To be sure it's **newly created**, bitwise-OR `IPC_CREAT` with **`IPC_EXCL`**, so the call **fails if the queue already exists**

The return value is the **queue ID** of the queue we will use. Then we can send and receive messages. But what does a message look like? Unlike in a lot of other contexts, here the message has a **defined structure**:

```c
struct msgbuf {
	long mtype;
	char mtext[1];
};
```

This does *not* mean a message can only be 1 character. It means that whatever message struct you send has to have its <u>*first member be a `long`*</u> (the type); anything is fine after that:

```c
struct pirate_msgbuf {
	long mtype;
	struct pirate_info {
		char name[30];
		char ship_type;
		int notoriety;
		int cruelty;
		int booty_value;
	} info;
};
```

> [!warning] Missing: `mtype` must be positive (L06: Pass Your Message)
> The `mtype` value **must be > 0**. (Also, the struct is named `pirate_msgbuf`, not `private_msgbuf`; fixed above.)

To send:

```c
int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag);
```

The parameters are:
1. The **queue** to send to
2. The **message**
3. The **size of the data** to send, <u>*excluding the `mtype` field*</u>
4. What happens if the queue is **full**: normally we just want to wait (a **blocking send**), so pass **0**. Alternatively, **`IPC_NOWAIT`** makes an attempt to add to a full queue **return an error** instead of blocking.

To receive:

```c
ssize_t msgrcv(int msqid, void *ptr, size_t nbytes, long type, int flag);
```

The parameters are:
1. The **queue** to receive from
2. The **destination** the message will be copied to
3. The number of bytes of the message's **payload** (again excluding `mtype`)
4. **`type`**: which kind of message you want, matched against the **`mtype`** field:
	- **`type == 0`**: the first message on the queue (**any type**)
	- **`type > 0`**: the first message on the queue **with exactly that type**
	- **`type < 0`**: the first message whose type is the **smallest value ≤ |`type`|**
5. **`flag`**: like `msgsnd`, **`IPC_NOWAIT`** means don't wait if **there's no message**; 0 blocks until one arrives

> [!warning] Correction: the `type < 0` case (L06: Pass Your Message)
> It's not "smallest or equal to absolute". It picks, among messages with type **≤ |`type`|**, the one with the **lowest type**. E.g. with `type = -5` and messages of types 4, 2, 7 queued, you get the **type 2** message (7 is too big, and 2 < 4).

When we're finished, we clean up using:

```c
int msgctl(int msqid, int command, struct msqid_ds *buf);
```

To clean up, pass the **queue ID** from `msgget`, the command **`IPC_RMID`**, and **`NULL`** for the last parameter. This <u>*immediately deletes the queue and all data in it*</u>. `msgctl` can do other things, but those are beyond the scope of the course.

> [!warning] Missing: why cleanup matters (L06: Pass Your Message)
> Because the queue is **owned by the OS** and independent of any process, it **does not disappear when your processes exit**. If nobody calls `msgctl(..., IPC_RMID, NULL)`, it sticks around.

### Example: Parent Sends a Message to the Child

```c
#include <stdlib.h>
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/types.h>
#include <sys/msg.h>
#include <unistd.h>

struct msg {
	long mtype;
	int data;
};

int main(int argc, char** argv) {
	int msgqid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);

	int pid = fork();
	if (pid > 0) { /* Parent */
		struct msg m;
		m.mtype = 42;
		m.data = 252;
		msgsnd(msgqid, &m, sizeof(int), 0);
	} else if (pid == 0) { /* Child */
		struct msg m2;
		msgrcv(msgqid, &m2, sizeof(int), 42, 0);
		printf("Received %d!\n", m2.data);
		msgctl(msgqid, IPC_RMID, NULL);
	}
	return 0;
}
```

Output:

```
Received 252!
```

- The queue is created **before `fork`**, so both processes have the same **`msgqid`**. This is why [`IPC_PRIVATE`](#pass-your-message) works here: no one else needs to find the queue.
- `0666` gives read/write permission to everyone, and `IPC_CREAT` creates the queue.
- The **parent** fills in a message with `mtype = 42` and `data = 252`, and sends **`sizeof(int)`** bytes, since the size excludes `mtype`. Flag `0` means it would block if the queue were full.
- The **child** asks for a message of **type 42** with flag `0`, so it **blocks until the parent's message arrives**, no matter which process runs first after `fork`. This is the **async send, sync receive** pattern from the start of the lecture.
- The child is the **last user of the queue**, so it's the one that removes it with `IPC_RMID`.

> [!warning] Things the example glosses over (L06: Pass Your Message)
> - The parent **doesn't `wait`** for the child, so it may exit first and the shell prompt can come back **before** `Received 252!` is printed. The output is the same either way.
> - Real code should **check return values**: `msgget` and `fork` return **-1** on failure, and so do `msgsnd`/`msgrcv`.
>
> Comparison sources: [L06 lecture text](../lectures/L06.tex), [L06 slides](../lectures/L06-slides.tex), and [msgq.c](../lectures/code-examples/msgq.c).
