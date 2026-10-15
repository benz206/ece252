## Threads
Recall that a process has 3 major components: the **executable program**, the **data** it creates or needs, and its **execution context** (open files, allocated resources, etc.). A process has **at least one *thread***, and can have **many**.

"Thread" is short for ***thread of execution***: a **sequence of executable commands** that can be **scheduled to run on the CPU**. A thread also has some **state** (where it is in that sequence) and some **local variables**. Most programs we've written so far have had **one thread**: the code runs one statement at a time, in order.

A ***multithreaded*** program uses **more than one thread**, at least some of the time. A program starts with an **initial thread** (the one running **`main`**), and that thread can **create more threads** as needed. Threads can be created and destroyed **dynamically**:
- a thread can be created for a **specific background task** (e.g. writing changes to a database) and **terminate when it's done**, or
- a thread can be **persistent** and stick around.

In a process with multiple threads, **each thread has its own**:
1. **Execution state** (like a process: running, ready, blocked, ...)
2. **Saved context** for when it's not running
3. **Execution stack**
4. **Local variables**
5. **Access to the process's memory and resources**, which is <u>*shared with all the other threads*</u> in that process

![[mthread2.png]]

So all threads in a process **share its state and resources**: if one thread **opens a file**, the **other threads can access it too**.

There's also **no mandatory hierarchy**: **any thread can create other threads**, not just the main one.

![[peerThreads.png]]

#### Example: The UI Thread
Almost every program today is multithreaded in some way. A common split is to separate the **user interface** from a **time-consuming action**.

Consider a **file-transfer program**. If the UI and the upload **share one thread**, then once an upload starts, the user <u>*can't use the UI at all*</u>, not even to click **Cancel** (this is when Windows adds **"(Not Responding)"** to the title bar). Users hate that.

There are 2 ways to fix this when an upload starts:
- call **`fork`** and have a **new process** do the upload, or
- **spawn a new thread** to do the upload.

Either way, the UI stays **responsive** because the UI thread **isn't waiting** for the upload to finish.

### Motivation for Threads
Why a thread instead of a new process? Mainly (but not only) **performance**:
1. **Creating** a thread is much faster than creating a process, on the order of **10 times faster**.
2. **Terminating and cleaning up** a thread is faster than a process.
3. **Switching** between two threads of the **same process** is faster, since **less data** has to be saved/restored. (On Solaris, switching between processes is about **5 times slower** than between threads.)
4. Threads **share the same memory**, so they can communicate **directly**, with <u>*no IPC mechanisms needed*</u>. This is a bigger deal than it sounds, since IPC can be **very expensive**.
5. The program stays **responsive** even when part of it is **blocked** (like the file transfer).

#### Common Uses of Threads
That last point is one of **4 common uses** of threads in a general-purpose OS:
1. ***Foreground and background work***: run something in the background so the program **stays responsive**.
2. ***Asynchronous processing***: e.g. a word processor **periodically saving** the document to disk in the background to protect against a crash or power failure, <u>*without interrupting the user*</u>. This is how Word can offer a **"recovered"** document.
3. ***Speed of execution***: a multithreaded program can **get more done in the same time**. If one thread is **blocked** (e.g. on a disk read), **another can run**. And with **multiple cores**, we can actually **use all of them** at once.
4. ***Modular structure***: a program that does several things can be **structured** with threads, each with a specific **"job"** that it steps in and does when appropriate.

#### Drawbacks
- There's **no protection between threads** in the same process: one thread can easily **mess with memory another thread is using**. This brings us back to **coordination**, which comes after threads.
- If **any thread hits an error** (e.g. **division by zero** or a **segmentation fault**), the OS may terminate the **whole process**. With **separate processes**, the others **aren't affected**; with **threads in one process**, one bad thread can <u>*bring them all down*</u>.

### Thread States
Each thread has **its own state**. Processes had the seven-state model, but threads use the simpler ***five-state model***: if a thread is blocked, **we don't care why** (even if the OS does).

![[5-state-model.png]]

- The transitions work **the same as for processes** (see [[L04-Processes]]).
- Like a process, a thread in **any state** can go to **terminated**, even though the diagram doesn't show it.
- When a **process is terminated**, **all of its threads are terminated**, whatever state they're in.

Cancelling the file upload from earlier is an example of a termination worth thinking about: ***thread cancellation***.

### POSIX Threads
***`pthread`*** refers to the **POSIX standard** (**IEEE 1003.1c**) that defines **how threads behave** on UNIX and UNIX-like systems (Linux, macOS, Solaris, ...). Because it's a standard, thread code written for one of these systems **runs easily on another**.

It defines around **100 functions**, but we only need a few. Several are similar to what we saw with **parent and child processes**:

| Function | What it does | Process equivalent |
| --- | --- | --- |
| **`pthread_create`** | **Create** a new thread | **`fork`** |
| **`pthread_exit`** | **End** the calling thread and **return a value** | **`exit`** |
| **`pthread_join`** | **Wait** for a specific thread to exit; the caller can't continue until it does. <u>*Joining a thread that's already been joined is an error.*</u> | **`wait`** |
| **`pthread_detach`** | Make a thread ***detached*** so it **can't be joined** | |
| **`pthread_yield`** | **Give up the CPU** and let another thread run. Threads in the same program are expected to **cooperate** rather than compete, so a thread can decide when it's a good time to let another one go. | |
| **`pthread_attr_init`** | **Create and initialize** a thread's **attributes** (e.g. its priority) | |
| **`pthread_attr_destroy`** | **Free** the memory holding a thread's attributes. This <u>*does not terminate the thread*</u>. | |
| **`pthread_cancel`** | **Signal cancellation** to a thread; either **asynchronous** or **deferred**, depending on the thread's attributes | |
| **`pthread_testcancel`** | A thread **checks whether it's been cancelled**; if so, this **terminates** it | |

> [!warning] `pthread_yield` isn't actually POSIX (beyond the lecture)
> `pthread_yield` is a **non-standard Linux (glibc) extension** and is now **deprecated** there. The standard way to give up the CPU is **`sched_yield()`** from `<sched.h>`. It doesn't exist on macOS.

#### Creating a Thread: `pthread_create`
When we start a thread, we have to say **what it should do**:

```c
pthread_create( pthread_t *thread, const pthread_attr_t * attr, void *(*start_routine)( void * ), void *arg );
```

1. **`thread`**: a pointer to a **`pthread_t` identifier**, which gets **filled in** when the thread is created
2. **`attr`**: the **attributes**; pass **`NULL`** for the defaults
3. **`start_routine`**: the **function the new thread runs**
4. **`arg`**: the **argument passed to `start_routine`**

The weird one is `start_routine`. Its type means "**any function that takes one `void*` and returns a `void*`**". The **names** of the function and its argument can be **anything**, as long as the signature matches:

```c
void* do_something( void* start_params )
```

After `pthread_create`, the process has **two threads**. The OS makes <u>*no guarantee about which one runs next*</u>; that's up to the **scheduler**. It could be **either** thread, **both at once** (on different cores), or a **different process entirely**.

> [!warning] Missing: the return value (beyond the lecture)
> `pthread_create` (and the other `pthread_` functions) return an **`int`**: **`0` on success**, or an **error number** on failure. Unlike most system calls, they **don't return `-1` and set `errno`**; the error code *is* the return value.

#### Passing Multiple Arguments
We only get **one argument** (and one return value). To pass more, either:
- pass a **pointer to an array**, or
- (better) define a **`struct`** for the **parameters** (and another for the **return type**) and pass a **pointer to it**.

The array trick is also how you'd return multiple values from a function in Java or C# (`public Object[] foo()`), but it's **not good practice**.

Inside the thread function, **cast** the `void*` back to the **real type**:

```c
void* function( void * void_arg ) {
	parameters_t *arguments = (parameters_t*) void_arg;
	/* continue after this */
}
```

This means the caller of `pthread_create` **has to know what type the function expects**. That's normal for any function, but here we <u>*don't get the type hints*</u> from the compiler since everything is `void*`.

#### Attributes
Attributes set things like whether a thread is **detached or joinable**, its **scheduling policy**, etc.
- By default, threads are usually ***joinable***: another thread can call `pthread_join` on them.
- Joining the **same thread more than once** is a **logical error**.
- To make sure a thread is **never joined**, create it in the **detached state**, or call **`pthread_detach`** on a joinable thread. **Joining a detached thread** is also a **logical error**.

For virtually everything in this course, the **defaults are fine** (`NULL`).

### Terminating a Thread
Once created, the thread **runs its code** until it reaches the end, where it usually terminates with **`pthread_exit`**. Sometimes we want a thread to **persist** instead, but if we want a **return value** from it, it **has to exit**.

A thread can terminate in several ways:
- **`pthread_exit(value)`**: the normal way.
- **Returning** from its function: if there's nothing to return, **`return NULL;`** has the **same effect as `pthread_exit`**, sending `NULL` back to whoever joins it. Returning normally **still terminates** the thread.
- **`pthread_cancel`** targeting it: if the cancellation is **deferred** (not asynchronous), the thread is **responsible for cleaning up** after itself before stopping.
- **Indirectly**: if the **whole process is terminated**, or if **`main` finishes first** without calling `pthread_exit`.

That last one matters: if `main` just returns, it <u>*won't wait*</u> for unjoined threads, and they all get **suddenly terminated**. If `main` calls **`pthread_exit`** as the last thing it does, the other threads get to **finish**.

> [!tip] What "`main` calls `pthread_exit`" actually does (beyond the lecture)
> Returning from `main` calls **`exit()`**, which ends the **whole process**. `pthread_exit` instead ends **only the main thread**; the process keeps running until its **last thread exits**. The lecture describes this as `main` being "blocked" until the others finish, which is the same effect.

### Collecting Return Values: `pthread_join`
Like `wait`, **`pthread_join`** is how we get a value **out of** a thread:

```c
pthread_join( pthread_t thread, void** retval );
```

1. **`thread`**: the thread to **join** (wait for)
2. **`retval`**: <u>*two stars*</u>, a **pointer to a `void*`**. We give it the **address of our own `void*` variable**, and join **updates that pointer** to point at the value the thread returned.

#### Example
```c
#include <stdlib.h>
#include <stdio.h>
#include <pthread.h>

void * run( void * argument ) {
	char* a = (char*) argument;
	printf("Provided argument is %s!\n", a);
	int * return_val = malloc( sizeof( int ));
	*return_val = 99;
	pthread_exit( return_val );
}

int main( int argc, char** argv ) {
	if (argc != 2) {
		printf("Invalid args.\n");
		return -1;
	}
	pthread_t t;
	void* vr;

	pthread_create( &t, NULL, run, argv[1] );
	pthread_join( t, &vr );
	int* r = (int*) vr;
	printf("The other thread returned %d.\n", *r);
	free( vr );
	pthread_exit( 0 );
}
```

- `main` passes **`argv[1]`** (a `char*`) as the thread's argument, and `run` **casts it back** to `char*`.
- `run` **`malloc`s** its return value. It <u>*can't return the address of a local variable*</u>: that lives on the **thread's own stack**, which is gone once the thread exits. Heap memory survives.
- `main` passes **`&vr`** to `pthread_join`, so after the join, **`vr` points at the `99`**. It's cast to `int*` to read it.
- Since `run` allocated the memory, **`main` must `free` it**.
- `main` ends with **`pthread_exit`**. By this point the only other thread has already been joined, so it doesn't change anything here.

> [!warning] Compiling (beyond the lecture)
> Compile with **`-pthread`** (e.g. `gcc -pthread example.c`), or the linker may not find the `pthread_` functions on older Linux systems.
