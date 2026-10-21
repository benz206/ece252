## More on Threads
Last lecture covered **creating** a thread, how one **exits** (with or without a return value), and how to **collect** that value. Now: **cancellation**.

### Thread Cancellation
***Thread cancellation*** is what it sounds like: a running thread is **terminated before it finishes its work**, e.g. the user presses **Cancel** on the file upload and we want to stop the upload thread.

The thread being cancelled is called the ***target***. There are **2 ways** it can be cancelled:
1. ***Asynchronous cancellation***: one thread **immediately terminates** the target.
2. ***Deferred cancellation***: the target is **told** it's cancelled, and it's **responsible for checking regularly** whether it's been cancelled, so it can **clean itself up properly**.

A thread can set **its own cancellation type** with:

```c
pthread_setcanceltype( int type, int *oldtype )
```

1. **`type`**: the new type, either **`PTHREAD_CANCEL_DEFERRED`** or **`PTHREAD_CANCEL_ASYNCHRONOUS`**
2. **`oldtype`**: gets **filled in with the previous type** (we might not care)

> [!warning] Correction: setting the type through attributes (beyond the lecture)
> The lecture says the pthread attributes can set the cancellation type before the thread is created. POSIX actually has **no attribute for this** (there's no `pthread_attr_setcanceltype`). A thread **always starts deferred**, and only the thread itself can change it, with `pthread_setcanceltype`.

With **deferred** cancellation, the thread has to **check** whether it's been cancelled and, if so, **stop and clean up** (close open files, etc.) before terminating. It's possible (but **poor practice**, and very difficult) to **never check**, which effectively **ignores** the cancellation.

**So why ever choose deferred, if the thread can ignore it?** Because of **resources**. If a thread holding resources is terminated in a **disorderly** way, the OS **may not reclaim all of them**. A resource could then **look like it's in use** when it isn't, <u>*denying it to other threads and processes*</u> that want it.

#### Cancelling and Checking
- **`pthread_cancel(thread)`** sends the cancellation. It takes **one parameter**, the thread ID.
- Threads are **deferred by default**.
- Inside the thread's function, **`pthread_testcancel()`** (no parameters) **checks** whether the thread has been cancelled.

Say the background task **uploads a bunch of files** one after another. Good practice is to call **`pthread_testcancel` at the start or end of each loop iteration**. That way, if the thread is told to cancel, it does so <u>*within a fairly short time*</u>.

> [!warning] Clarification: `pthread_testcancel` doesn't come back (beyond the lecture)
> The lecture describes checking `pthread_testcancel` and then cleaning up and calling `pthread_exit`. But if the thread **has** been cancelled, `pthread_testcancel` **terminates it on the spot**; it never returns to let you clean up. The cleanup has to be set up **beforehand**, with the **cleanup handlers** below.

#### Cancellation Points
Many functions are ***cancellation points***: POSIX requires an **implicit cancellation check** when you call them. Even more are ***potential cancellation points***, where the spec says they **might** be (but maybe aren't). If unexpected cancellation would be a problem somewhere, **check the spec** for the specific function.

### Cleanup Handlers
With cancellation points or asynchronous cancellation, a thread can be terminated **before it's cleaned up its resources**. To guard against this, we can register ***cleanup handlers***. For example, if the thread allocated memory, register a handler that **frees it** in case the thread **dies unceremoniously**.

```c
pthread_cleanup_push( void (*routine)(void*), void *argument ); /* Register cleanup handler, with argument */
pthread_cleanup_pop( int execute ); /* Run if execute is non-zero */
```

- **`pthread_cleanup_push`** registers a handler. Its arguments are the **function to run** and a **pointer to the argument** that function needs.
- **`pthread_cleanup_pop`** removes it. Its argument says **whether to run it**: run if **non-zero**.
- If the thread is **cancelled**, the handler **runs**. If the thread **reaches the pop**, **you choose** whether it runs.

<u>*Every push must be paired with a pop at the same level*</u> (same pair of curly braces). Think of them like an **opening and closing brace**: they have to **match up**.

> [!tip] Why they have to match (beyond the lecture)
> `push` and `pop` are usually **macros**: `push` expands to code that **opens a `{`** and `pop` expands to code that **closes it**. An unmatched one won't even **compile**.

#### Example
Without a handler:

```c
void* do_work( void* argument ) {
	struct job * j = malloc( sizeof( struct job ) );
	/* Do something useful with this structure */
	/* Actual work to do not shown */
	free( j );
	pthread_exit( NULL );
}
```

If the thread is **asynchronously cancelled** during the work on `j`, it **never reaches `free()`**, so the memory is **leaked**. With a cleanup handler:

```c
void cleanup( void* mem ) {
	free( mem );
}

void* do_work( void* argument ) {
	struct job * j = malloc( sizeof( struct job ) );
	pthread_cleanup_push( cleanup, j );
	/* Do something useful with this structure */
	/* Actual work to do not shown */
	free( j );
	pthread_cleanup_pop( 0 ); /* Don't run */
	pthread_exit( NULL );
}
```

- If the thread is cancelled during the work, **`cleanup(j)` runs** and frees the memory.
- If it finishes normally, it frees `j` itself, then **pops without running** the handler (`0`).

You can save a line: **remove the `free()`** and use **`pthread_cleanup_pop(1)`**, so the **handler does the freeing** either way.

> [!warning] The `pop(1)` version is also safer (beyond the lecture)
> In the version shown, if an asynchronous cancel lands **between `free( j )` and the pop**, the handler runs and frees `j` a **second time** (a double free). With `pop(1)` and no separate `free`, there's only ever **one** place that frees it.

### Using Memory to Pass Data
The earlier example used the thread's **return value**. But one advantage of threads is that they can pass data **directly through memory**. If we don't need a return value, we pass **`NULL`** as the second argument to `pthread_join`.

This example also shows how to **initialize the attributes** (though it doesn't change any defaults):

```c
#include <pthread.h>
#include <stdio.h>

int sum; /* Shared Data */

void *runner(void *param);

int main( int argc, char **argv ) {

	pthread_t tid; /* the thread identifier */
	pthread_attr_t attr; /* set of thread attributes */

	if ( argc != 2 ) {
		fprintf(stderr,"usage: %s <integer value>\n", argv[0]);
		return -1;
	}
	if ( atoi( argv[1] ) < 0 ) {
		fprintf(stderr, "%d must be >= 0\n", atoi(argv[1]));
		return -1;
	}

	/* set the default attributes */
	pthread_attr_init( &attr );
	/* create the thread */
	pthread_create( &tid, &attr, runner, argv[1] );
	pthread_join( tid, NULL );
	printf( "sum = %d\n", sum );
	pthread_attr_destroy( &attr );
	pthread_exit( NULL );
}

void *runner( void *param ) {
	int upper = atoi( param );
	sum = 0;
	for ( int i = 1; i <= upper; i++ ) {
		sum += i;
	}
	pthread_exit( 0 );
}
```

- Both threads share the **global variable `sum`**: the thread writes it, `main` reads it.
- There **is coordination** here: `main` **joins** the thread (waits for it to finish) **before printing**. Without the join, `main` would print the sum **too early**.
- **`pthread_attr_init`** sets up `attr` with the defaults, and **`pthread_attr_destroy`** frees it once we're done.

> [!warning] Missing from the example (beyond the lecture)
> `atoi` needs **`#include <stdlib.h>`**. Without it, modern compilers (e.g. clang on macOS) **refuse to compile** the example.

### Count to 10...
A different take on the same program, now with **3 threads**:

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

int sum = 0;

void* runner( void *param ) {
	int upper = atoi( param );
	for (int i = 1; i <= upper; i++ ) {
		sum += i;
	}
	pthread_exit( 0 );
}

int main( int argc, char** argv ) {

	pthread_t tid[3];

	if ( argc != 2 ) {
		printf("An integer value is required as an argument.\n");
		return -1;
	}
	if ( atoi( argv[1]) < 0 ) {
		printf( "%d must be >= 0.\n", atoi(argv[1]) );
		return -1;
	}

	for ( int i = 0; i < 3; ++i ) {
		pthread_create( &tid[i], NULL, runner, argv[1] );
	}
	for ( int j = 0; j < 3; ++j ) {
		pthread_join( tid[j], NULL );
	}
	printf( "sum = %d.\n", sum );

	pthread_exit( 0 );
}
```

What goes wrong? For **very small** arguments, **nothing**. But for **large** ones, we get **strange and inconsistent results**. That's because **3 threads are modifying `sum`**, and if threads write to the **same place at the same time**, things can go wrong.

But what does **"at the same time"** mean for a program? To answer that, we need to look at how the **hardware** behaves and how the **OS schedules** work.

> [!tip] What it actually looks like (beyond the lecture)
> Running it with `100` gives the right answer (`15150`). With `30000`, the right answer is `1350045000`, but 4 runs gave `476249146`, `505809570`, `673346438`, and `457049061`: **wrong, and different every time**. The cause: `sum += i` is really **read `sum`, add, write `sum` back**, and two threads can both read the **same old value**, so one of their updates is **lost**. Coordination (coming up) is how we fix it.
>
> Don't test with arguments much past **37,000**: the correct total, 3·n(n+1)/2, no longer fits in an `int`, so the answer would be wrong even without the race.

## Concurrency
Not long ago, a typical computer had **one processor with one core**, so it could do **exactly one thing at a time**. "One processor" means one **general-purpose** processor that runs user processes. There may be **special-purpose processors** too (e.g. a RAID controller, or a GPU, though GPUs are now used for general computation too), but with only one general-purpose processor it's a ***uniprocessor*** system.

Now desktops, laptops, and even phones almost certainly have ***multi-core*** processors. A **quad-core** processor may be executing **4 different instructions from 4 different threads at the same time**. In theory, more processors means **more work in the same (wall clock) time**, but <u>*that's not guaranteed*</u>.

> [!tip] Terminology: core vs. CPU
> A ***core*** is a **logical processing unit**. **CPU** may refer to the **physical chip**, which contains **one or more** cores. The OS **doesn't much care** whether there are 4 cores on 4 chips or 4 cores on 1 chip: either way, there are **4 units that can execute instructions**.

How many cores get used?
- **1 process with 1 thread**: at most **1 core**, no matter how many are available.
- **Multiple processes**: each can run on a **different core**.
- **More processes/threads than cores**: we could **hope** they block often and long enough for everyone to get a turn, but we **can't count on that**.

### Time Slicing
The solution is ***time slicing***: the CPU **switches between tasks**. Thread 1 runs for a set period (say **20 ms**), then thread 2 for 20 ms, then thread 3, then back to thread 1. To the user, it looks like they're all running **in parallel**, because 20 ms is too fast to notice.

![[single-core-execution.png]]

Time slicing **still happens on multiple cores** when it's needed: e.g. **4 threads on a dual-core** system.

![[dual-core-execution.png]]

### Parallelism and Speedup
If multiple threads run at the same time, the task finishes faster, right? **Maybe**. It depends a lot on the **task**. There's some **overhead** in splitting a task up and recombining the results, but it's usually **negligible** compared to the work itself.

There are 3 cases:
- ***Fully parallelizable***: adding a second thread **doubles the speed**. One person paints an apartment in **12 hours**, 2 people in **6**, 3 in **4**, 4 in **3**, etc. That's the ideal, but there's a **limit**: at some point the overhead **isn't negligible** anymore. In theory, 720 painters could do it in **1 minute**, but you <u>*can't fit them in the room*</u>.
- ***Partially parallelizable***: it can be divided, but doubling the workers **doesn't halve the time**. Two chefs might take **75%** of the time one chef would. They can work **independently** some of the time, but sometimes one has to **wait for the other**: the sauce can't go on the chicken until the chicken comes out of the oven.
- ***Not parallelizable***: some tasks can only be done **sequentially**, so **no number of workers helps**. You can't cook a steak in 1 minute by putting it in 5 ovens.

#### Speedup Example
A task takes **5 s**. It contains a **loop that can be parallelized**, and the **initialization and recombination** code takes **0.4 s** (this part is **serial**).
- **1 processor**: the loop takes **4.6 s**, total **5 s**.
- **2 processors**: the loop takes **2.3 s**, plus the 0.4 s serial part = **2.7 s**.

That's about a **46% reduction** in time (from 5 s to 2.7 s).

### Amdahl's Law
***Amdahl's Law*** gives the general case: how much faster a task can get with more processors. Let:
- **$S$** = the **portion** of the task that **must be done serially**
- **$N$** = the number of **processing cores**

$$\text{speedup} \leq \frac{1}{S + \frac{1-S}{N}}$$

Taking the limit as **$N \to \infty$**, the speedup converges to **$\frac{1}{S}$**. So the **limiting factor** is the **size of the serial part**, which matches intuition. If the task is **completely sequential** ($S = 1$), the max speedup is $\frac{1}{1 + 0} = 1$: **no speedup at all**.

Applying it to the 5 s example (here $S = \frac{0.4}{5} = 0.08$):

| Processors | Run Time (s) |
| --- | --- |
| 1 | 5 |
| 2 | 2.7 |
| 4 | 1.55 |
| 8 | 0.975 |
| 16 | 0.6875 |
| 32 | 0.54375 |
| 64 | 0.471875 |
| 128 | 0.4359375 |

Each row is just **0.4 + 4.6 / N**. Two observations:
1. ***Diminishing returns***: going from 1 to 2 processors cuts the time **dramatically**, but 64 to 128 barely helps.
2. The run time **converges to 0.4 s**: the serial part always takes 0.4 s, and with infinite processors the parallel part would be **effectively instant**.

So the **best possible speedup** here is $\frac{5}{0.4} = 12.5$ times. You can't do better, and in practice you'll never **reach** it either, since nobody has **infinite processors** (that would take infinite space and infinite money).

### Merge Sort Example
Recall ***merge sort*** from data structures and algorithms: a **divide-and-conquer** algorithm (like binary search). **Split** the array into smaller pieces, **sort** those, then **merge** the sorted pieces back together. With threads, we can have **many threads sorting** the pieces and **one thread merging** the sorted lists into a larger sorted list.

![[multithread-sort.png]]
