## The Process in UNIX
Earlier on, we mentioned that in UNIX, a process can create other processes. The creating process is the [**parent**](L04-Processes.md#process-family-tree) and the newly created processes are its **children**. Again, every process has a parent.

Each process has a unique ID, which we call the **`pid`** (process ID). For the most part we don't care about it unless we're running a `kill` command.

> [!warning] Missing: the root of the tree (L05: The Process in UNIX)
> Every parent chain ends at **`init`** (or `launchd` on macOS), which **always has PID 1**. The `pid` is stored in the process's [PCB](L04-Processes.md#the-process-control-block). Don't try to kill `init`: it usually ignores you, but you might crash or reboot the system.

![[Pasted image 20260918091834.png]]

In a UNIX system, we can obtain a list of processes at any time with the **`ps`** command. The diagram shows a basic hierarchy of how this might be set up.

> [!warning] Missing: what the diagram shows (L05: The Process in UNIX)
> Each user who logs in gets a **`login`** process, which spawns the user's **shell** (usually `bash`). Every command you type is then a **child of the shell**.

When you issue a command, like `ls` or `top` (table of processes), a new process is created and the shell will **`wait`** on that process to finish.

When control goes back, you will be prompted to run more commands again. This would seem kind of limiting: do I have to log in to the system in a second terminal window to run 2 things at a time? Not really, we can get around this.

To do this using `gcc`, we can just use the ampersand operator **`&`** to signify that this should run in the **background**. Notably, any console output will *still be printed* even after control is returned. So instead, maybe you want it to go to a file, and you can use `cat fork.c > logfile.txt`.

> [!warning] Clarification: `&` and `>` are shell features, not `gcc` features (L05: The Process in UNIX)
> `gcc` was just the example command: `gcc fork.c &`. Adding **`&`** to *any* command tells the **shell** not to `wait` for it. The shell prints something like `[1] 34429` (job number and child PID), and later `[1]+ Done gcc fork.c`.
>
> **`>`** redirects a command's output into a file. The lecture's example combines both: `cat fork.c > logfile.txt &`.

> A common example of a command I use involving the `&`:
> `sudo service xyz start &`
>
> This will (with **super user permissions**, which is the purpose of `sudo`) start up the service `xyz` but return control to the console, so I don't have to wait for the `xyz` service to start before entering my next command. This is good, because the next thing I'd like to do is `tail -f /var/log/xyz/console.log`, which will allow me to watch the console log of the `xyz` service as it starts up to see if there are any errors.
>
> The other alternative to get something to run in the background is the **`screen`** command. While having something run in the background is nice, it **does not work for interactive processes**. Suppose you are working on some code in `vi` and you would like to pause that for a minute and write an e-mail (with `pine` or whatever the cool kids use for command-line e-mail these days). One approach is to save and exit `vi` and open up `pine`. The other would be to start up each of these in `screen` and switch between them.
>
> Thus, instead of just opening `vi fork.c`, I can issue the command `screen vi fork.c`, and this spawns `screen` and takes me right to editing the file. The key difference is that I can **"detach"** from this screen and go back to the command line that spawned it. And if I log out, `screen` **keeps running** with the `vi` inside it. If I have multiple screens running, I can just **"reattach"** to the one I want to use next. To get a full understanding of `screen`, try the command `man screen` and the user manual will appear to give you some information and instructions about how to use it. Or you can use Google.

TL;DR: I'm assuming `screen` is basically the same as **tmux**.

> [!warning] Confirmation: `screen` ≈ tmux (L05: The Process in UNIX)
> Yes, both are **terminal multiplexers**: detach, log out, reattach later, and the programs inside keep running. The takeaway from the lecture is that **`&` works for non-interactive jobs**, while **`screen`/tmux is for interactive ones** (like an editor).

## Show Me the Code!
Basically, a parent can **`fork`** itself to create a child process, and can then use the system call **`wait`** to wait for that process to complete.

> [!warning] Missing: what `fork` and `exec` actually do (L05: Show Me The Code!)
> - **`fork()`** makes a **copy of the calling process**. *Both* processes continue from the line right after the `fork`, and the only difference between them is the **return value**:
> 	- **`< 0`**: the fork **failed** (no child exists)
> 	- **`== 0`**: you are the **child**
> 	- **`> 0`**: you are the **parent**, and the value is the **child's PID**
> - **`exec`** (e.g. `execlp`) **replaces** the calling process's memory with a new program. It doesn't create a new process, and it **only returns if it fails**. A child doesn't *have* to call `exec`; it can stay a clone of the parent (see the [design problem](#show-me-the-code) below).
> - While in `wait`, the parent is **Blocked** until the child exits. The child's `exit` value comes back to the parent through `wait`.

```c
#include <sys/types.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char** argv) {
	pid_t pid;
	int child_status;
	
	pid = fork();
	
	if (pid < 0) {
		// Some error occurred
		fprintf(stderr, "Fork failed");
		return 1;
	} else if (pid == 0) {
		// child process
		execlp("/bin/ls", "ls", NULL);
	} else {
		// parent process that will wait for the child
		// to complete
		wait(&child_status);
		printf("child complete with status word %i\n", child_status);
	}
	
	return 0;
}
```

> [!warning] Correction: `fprintf`, not `printf`, for `stderr` (L05: Show Me The Code!)
> Your original error branch called `printf(stderr, ...)`. Printing to a specific stream like `stderr` needs **`fprintf`**, exactly like [writing to a file](L03-TheFileSystem.md#reading-and-writing); it's been fixed above. (In real code, `wait` also needs `#include <sys/wait.h>`; the lecture leaves it out.)

We just get a simple output of:

```
fork   fork.c
Child Complete with status: 0
```

> [!warning] Missing: why the output looks like that (L05: Show Me The Code!)
> After a successful fork, **two processes** reach the `if`. The child replaces itself with `ls`, which prints the directory contents (`fork   fork.c`). The parent blocks in `wait`, and only prints once `ls` exits, so **the `ls` output always comes first**.

Or, to represent this visually:
![[Pasted image 20260918093436.png]]

What about **termination** though? Assuming the process is terminating normally and not being killed, the system call for that is just **`exit`**. If a program has no explicit call to `exit`, the **`return` statement at the end of `main`** will have the same effect.

- **Use of Fork Design Problem**
	- It's not necessary for a child to replace itself with another program
	- What if we want to make a program where both parts are in the *same source file*?
	- We can use the `fork()` function to create a child process. In this example, the child should call **`execute_B()`** and return the result to the parent, while **`execute_A()`** should be called by the parent

```c
pid_t pid;
int child_result;
int parent_result;

pid = fork();

if (pid < 0) {
	// Fork failed
	return -1;
} else if (pid == 0) {
	return execute_B();
} else {
	parent_result = execute_A();
	wait(&child_result);
}

if (child_result == 0 && parent_result == 0) {
	// completed
	return 0;
}

if (child_result != 0) {
	printf("Error %d Occurred.\n", WEXITSTATUS(child_result));
}

if ( parent_result != 0 ) {
	printf( "Error %d Occurred.\n", parent_result);
}
return -1;
```

> [!warning] Clarification: status word vs. return value, and the missing print (L05: Use of Fork Design Solution)
> - This code is the body of **`main`**. The child's `return execute_B();` returns from `main`, which is the same as **`exit(execute_B())`**. That's how the child "returns the result to the parent".
> - What `wait` stores in `child_result` is a **status word**, *not* the raw return value. It packs the exit code together with other info (e.g. whether the process was killed by a signal). **`WEXITSTATUS(child_result)`** extracts the actual exit code, which is why the error message uses it. Checking `child_result != 0` still works, because a status word of 0 means a normal exit with code 0. This is also why the first example printed "status **word**".
> - Only the **low 8 bits** of an exit code survive (0–255), so don't try to return large numbers this way.
> - The spec also requires printing **`Completed.`** before `return 0`; your `// completed` comment stands in for that `printf`.

## The Fork Bomb
- A simple example of how `fork` can be used **maliciously** is to just have an *infinite loop* calling `fork`
- Easily defended against by just limiting/killing processes over a limit

> [!warning] Clarification: why it explodes and how it's defended against (L05: The Fork Bomb)
> Every process in the loop forks, so the count **doubles** each round: **2ⁿ** processes after *n* rounds. That's a **denial-of-service** attack that quickly hits the system's limits. The defenses in the lecture are **preventive limits**: (1) a cap on the **total number of processes per user**, and (2) a cap on the **rate** a user can spawn them. Don't try it on school machines; it can get you banned.

## Signals
UNIX systems use **signals** to indicate events
- A signal is **synchronous** if it can be attributed to a single line of code
- A signal is **asynchronous** if it comes from outside the process, e.g. `Ctrl-C`, or one process/thread sending a signal to another

> [!warning] Missing: what a signal is and the default behaviour (L05: Signals)
> - "Synchronous" means it's caused by the **process's own execution**, e.g. **division by zero** or a **segfault**.
> - A signal is basically an **[interrupt](L02-InteruptsSysCalls.md#the-interrupts) with an integer ID**.
> - By default, the **kernel** handles every signal with a **default handler**. For some signals that means ignoring it (`SIGCHLD`), and for others it means terminating the process (`SIGSEGV`, `SIGINT`, `SIGTERM`...).
> - Numbers worth remembering from the table: **`SIGINT` = 2** (`Ctrl-C`), **`SIGKILL` = 9**, **`SIGTERM` = 15**, and **`SIGSEGV` = 11**.

![[Pasted image 20260921083943.png]]

Alternatively, a process could inform the OS that it is prepared to **handle the signal itself** (e.g. doing some cleanup on `Ctrl-C` instead of just dying). In any event, the signal eventually <u>*needs to be handled*</u>, even if the handling is just to ignore it.

Signals **`SIGKILL`** and **`SIGSTOP`** <u>*cannot be blocked, caught, or ignored*</u>.

On the command line, the command to send a signal is also just **`kill`** followed by the PID, as you know.

Using the flag **`-9`** will send `SIGKILL` instead of `SIGTERM`.

> [!warning] Missing: why `kill -9` is a last resort (L05: Signals)
> Plain `kill <pid>` sends **`SIGTERM`**, which *can* be caught, so the process gets a chance to **clean up** before dying. `SIGKILL` can't be caught, so the process dies immediately with **no cleanup**. Try a gentler signal first (`SIGTERM`, `SIGINT`, `SIGHUP`), and only use `-9` if it's still stuck.
>
> Signals are also a basic form of **inter-process communication**: one process sending a message to another. That leads into [L06](L06-IPC.md).
>
> Comparison sources: [L05 lecture text](../lectures/L05.tex) and [L05 slides](../lectures/L05-slides.tex).
