## Condition Variables
Another way to synchronize. Instead of marking out **critical sections** and limiting **how many** threads are inside, a ***condition variable*** lets threads synchronize based on **the value of the data**. Rather than repeatedly **lock, check a variable, unlock** (polling), a thread **waits** until it's told the thing it cares about happened.

Think of them as ***events***: like a **hardware interrupt** saying new data is ready, instead of **polling** for it.

An event is **similar to a counting semaphore**, with one difference: when it happens, you can either
- **signal**: wake **one** waiting thread, or
- **broadcast**: wake **all** waiting threads.

### Syntax

```c
pthread_cond_init( pthread_cond_t *cv, pthread_condattr_t *attributes );
pthread_cond_wait( pthread_cond_t *cv, pthread_mutex_t *mutex );
pthread_cond_signal( pthread_cond_t *cv );
pthread_cond_broadcast( pthread_cond_t *cv );
pthread_cond_destroy( pthread_cond_t *cv );
```

- **`pthread_cond_init`** / **`pthread_cond_destroy`**: create and clean up a **`pthread_cond_t`**. Attributes exist (with their own create/destroy functions), but **defaults are fine** for this course.
- **`pthread_cond_signal`**: wake **one** waiting thread.
- **`pthread_cond_broadcast`**: wake **every** waiting thread. This is what makes condition variables **more interesting** than the plain **signalling** pattern from L14.

#### Always with a Mutex
<u>*A condition variable is always used together with a mutex.*</u> **`pthread_cond_wait`** takes **both**:
1. Call it **only while holding the mutex**.
2. While waiting, it **automatically releases** the mutex.
3. When woken, it **automatically re-locks** the mutex before returning.
4. **You** unlock the mutex when you're done.

Forgetting to lock before or unlock after **causes problems**.

### Example

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM_THREADS  3
#define COUNT_LIMIT 12

int count = 0;
pthread_mutex_t count_mutex;
pthread_cond_t count_threshold_cv;

void* inc_count( void* arg ) {
  for (int i = 0; i < 10; i++ ) {
    pthread_mutex_lock( &count_mutex );
    count++;
    if ( count == COUNT_LIMIT ) {
      printf( "Condition Fulfilled!\n" );
      pthread_cond_signal( &count_threshold_cv );
      printf( "Sent signal.\n" );
    }
    pthread_mutex_unlock( &count_mutex );
  }
  pthread_exit( NULL );
}

void* watch_count( void *arg ) {
  pthread_mutex_lock( &count_mutex );  
  while ( count < COUNT_LIMIT ) {
    pthread_cond_wait( &count_threshold_cv, &count_mutex );
    printf( "Watcher has woken up.\n" );
    /* Do something useful here now that condition is fulfilled. */
  }
  pthread_mutex_unlock( &count_mutex );
  pthread_exit( NULL );
}

int main( int argc, char **argv ) {
  pthread_t threads[3];

  pthread_mutex_init( &count_mutex, NULL );
  pthread_cond_init ( &count_threshold_cv, NULL );

  pthread_create( &threads[0], NULL, watch_count, NULL );
  pthread_create( &threads[1], NULL, inc_count, NULL );
  pthread_create( &threads[2], NULL, inc_count, NULL );

  for ( int i = 0; i < NUM_THREADS; i++ ) {
    pthread_join(threads[i], NULL);
  }
  
  pthread_mutex_destroy( &count_mutex );
  pthread_cond_destroy( &count_threshold_cv );
  pthread_exit( NULL );
}
```

- Two **`inc_count`** threads each add 1 to `count` **10 times** (under the mutex). Whichever one makes it hit **12** signals the condition variable.
- **`watch_count`** locks the mutex and, **while `count < 12`**, waits on the condition variable (which **releases the mutex** so the incrementers can run).

#### Lost Wakeups
If a thread signals and **nobody is waiting**, the event is ***lost***. So it's (almost always) a **logic error** to signal or broadcast **before** anyone is waiting: the ***lost wakeup problem***. Sometimes it's harmless, though: maybe the event is broadcast and **nobody cares**, and that's fine.

> [!tip] Why `watch_count` is safe anyway (beyond the lecture)
> In the example, the incrementers might hit 12 **before** the watcher even starts, so the signal is lost. The watcher still works because it **checks `count` first, under the mutex**, and only waits if the condition **isn't already true**. That's the standard fix: <u>*the condition variable is just a doorbell; the shared variable is the real state*</u>.
>
> Always wait in a **`while`** loop, never an `if`. A thread can wake up with the condition **not actually true** (another thread got there first, or a ***spurious wakeup***, which POSIX allows). The `while` makes it **re-check** before continuing.

### Barrier with a Condition Variable
The L14 barrier: the last thread posts, then each thread **posts to unblock the next**. That's a lot of calls. A **broadcast** does it in one.

| Semaphore barrier | Condition variable barrier |
| --- | --- |
| 1. `wait( mutex )` | 1. `wait( mutex )` |
| 2. `count++` | 2. `count++` |
| 3. `if count == n` | 3. `if count < n` |
| 4. `    post( barrier )` | 4. `    cond_wait( barrier, mutex )` |
| 5. `end if` | 5. `else` |
| 6. `post( mutex )` | 6. `    cond_broadcast( barrier )` |
| 7. `wait( barrier )` | 7. `end if` |
| 8. `post( barrier )` | 8. `post( mutex )` |

The **`cond_wait` happens before the `post( mutex )`**. Looks wrong (waiting while holding the lock, the L14 deadlock!), but **`cond_wait` releases the mutex while waiting**, so it's fine. The **last thread doesn't wait**: it knows it's last, so it **broadcasts** and continues.

In C (assume `count`, `lock` and `cv` are initialized):

```c
int count;
pthread_mutex_t lock;
pthread_cond_t cv; 

void barrier( ) {
  pthread_mutex_lock( &lock );
  count++;
  while ( count < NUM_THREADS ) {
    pthread_cond_wait( &cv, &lock );
  } else {
    pthread_cond_broadcast( &cv );
  }
  pthread_mutex_unlock( &lock );
}
```

Every thread calls `barrier()` and waits until the **last one arrives**, which **broadcasts** to wake everyone. You *could* loop and signal *n* times, but that mostly **defeats the point** of using a condition variable over a semaphore.

> [!warning] `while ... else` isn't C (beyond the lecture)
> C has no `while`/`else`. A version that compiles and still loops on the condition:
> ```c
> void barrier( ) {
>   pthread_mutex_lock( &lock );
>   count++;
>   if ( count == NUM_THREADS ) {
>     pthread_cond_broadcast( &cv );
>   } else {
>     while ( count < NUM_THREADS ) {
>       pthread_cond_wait( &cv, &lock );
>     }
>   }
>   pthread_mutex_unlock( &lock );
> }
> ```

## Monitors
A condition variable can be used to build a ***monitor***: a **higher-level** synchronization construct. Like a **class** in OOP packages **data with its functions**, a monitor packages **shared data with the operations on it**, so the **synchronization is built in**.

The goal: programmers **don't write the synchronization themselves**, so there are fewer chances to get it wrong. And there are **lots** of chances:

```c
void foo( ) {
  pthread_mutex_lock( &l );
  /* Read some data */
  if ( condition ) {
    printf( "Cannot continue due to reasons...\n" );
    return;
  }
  /* More stuff */
   
  pthread_mutex_unlock( &l );
}
```

The early **`return`** leaves `foo` **without unlocking `l`**. Obvious here with everything else cut out, but in a **real function hundreds of lines long**, it's easy to miss.

### Java's `synchronized`
If you've used Java, you've used monitors. Put **`synchronized`** on a method, and there's a **lock around it**: only **one thread** can be in that method (for the **same instance**) at a time. Others wait in the lock's ***entry set*** until it's free.

```java
public synchronized void doSomething() {
    // Synchronized area
}
```

Or lock just a **block**:

```java
public void exampleMethod() {
    synchronized( object ) { // Lock must be acquired to enter this block
           // Critical section 
    } // Lock is automatically released.
}
```

The lock is **taken and released automatically**, including on early returns and exceptions. That abstracts away some of the mutual exclusion details.

Monitors **don't have to be in Java**, but they fit **object-oriented languages** naturally: **encapsulation** (`private` variables) makes it easier to **force** all access through the monitor.

> [!tip] C++ does this too (beyond the lecture)
> In C++, **`std::lock_guard`** / **`std::scoped_lock`** lock a mutex when created and **unlock automatically** when they go out of scope, so the early-return bug in `foo` can't happen.

## Atomic Types
A very common pattern:

```c
pthread_mutex_lock( lock );
shared_var++;
pthread_mutex_unlock( lock );
```

**Correct**, but if it happens **a lot**, that's a lot of **lock/unlock overhead** just to add 1. Like **test-and-set** from L12, wouldn't it be nice to do the increment **atomically in one step**? We can.

- **GCC** provides operations **guaranteed to be atomic**. Where possible it turns them into **uninterruptible hardware instructions**; otherwise it uses a function **with locking**.
- The **kernel** has **`atomic_t`**, but that's **not for use outside the kernel**.
- These are **GCC-specific**. **C11** (2011) finally added atomic types to the **language itself**; before that it was all **implementation-specific**.

In the signatures below, **`type`** is a placeholder (e.g. use `int`). Valid types are **integers or pointers** of **1, 2, 4, or 8 bytes**.

### Set

```c
type __sync_lock_test_and_set( type *ptr, type value );
```

**Sets** `*ptr` to `value` atomically and **returns the old value**. The name is unfortunate: it clashes with the **test-and-set hardware instruction** from L12, but it's really just an **atomic assignment**.

### Compare-and-Swap
**Swap in `newval` only if** the current value equals **`oldval`**:

```c
bool __sync_bool_compare_and_swap( type *ptr, type oldval, type newval );
type __sync_val_compare_and_swap( type *ptr, type oldval, type newval );
```

- **`bool`** version: returns whether the swap **happened**.
- **`val`** version: returns the value that was **there before**.

### Fetch-and-Op vs. Op-and-Fetch
**Return the old value** (fetch, then do the operation):

```c
type __sync_fetch_and_add( type *ptr, type value );
type __sync_fetch_and_sub( type *ptr, type value );
type __sync_fetch_and_or( type *ptr, type value );
type __sync_fetch_and_and( type *ptr, type value );
type __sync_fetch_and_xor( type *ptr, type value );
type __sync_fetch_and_nand( type *ptr, type value );
```

**Return the new value** (do the operation, then fetch):

```c
type __sync_add_and_fetch( type *ptr, type value );
type __sync_sub_and_fetch( type *ptr, type value );
type __sync_or_and_fetch( type *ptr, type value );
type __sync_and_and_fetch( type *ptr, type value );
type __sync_xor_and_fetch( type *ptr, type value );
type __sync_nand_and_fetch( type *ptr, type value );
```

> [!tip] Memory trick
> The name is the **order**: **fetch**\_and\_add **fetches first** (returns the **old** value), add\_and\_**fetch** **adds first** (returns the **new** value). Like `x++` vs `++x`.

So the lock/increment/unlock above becomes just `__sync_fetch_and_add( &shared_var, 1 );`.

### Reading
On **x86** there's **no atomic read operation**: a normal read of **32-bit-aligned** data is **already atomic**. But that's **x86-specific**, and relying on it could give you an **out-of-date value**.
- To be sure you get the latest: use one of the functions above and **add (or subtract) 0**. Also a bit **more portable**.
- For **true portability**: **C11 atomics** or a **semaphore/mutex**.

> [!tip] C11 atomics (beyond the lecture)
> With **`#include <stdatomic.h>`**: declare **`atomic_int x;`**, then **`atomic_fetch_add( &x, 1 )`**, **`atomic_load( &x )`**, **`atomic_store( &x, 5 )`**, **`atomic_compare_exchange_strong(...)`**. Even plain `x++` on an `atomic_int` is atomic. GCC has also replaced `__sync_*` with newer **`__atomic_*`** builtins.

### When Atomics Aren't Enough
Atomics are great for **a single variable** being read and modified. But:

```c
struct point {
  volatile int x;
  volatile int y;
};
__sync_lock_test_and_set( p1->x, 0 );
__sync_lock_test_and_set( p1->y, 0 );

/* Somewhere else in the program */
__sync_lock_test_and_set( p1->x, 25 );
__sync_lock_test_and_set( p1->y, 30 );
```

Each write is atomic, but the **pair isn't**:
- A reader could see **x written, y not yet**: **(25, 0)**, which is probably **not a valid point**.
- Another thread writing **(10, 15)** in between could leave **(10, 30)**: **corrupted**.

<u>*When several writes have to happen as a package, use a mutex.*</u>

> [!warning] Small fixes to that example (beyond the lecture)
> - The functions take a **pointer**, so it should be **`&p1->x`**, not `p1->x`.
> - **`volatile` doesn't make anything atomic** or thread-safe in C. It just stops the compiler from optimizing away reads and writes.

### Spinlocks
Common in **Linux** (especially the kernel). A ***spinlock*** **keeps checking** until it gets the lock, instead of **blocking**:
- An **integer**: if it's **0**, set it to **1** and go in.
- If it's **nonzero**, **keep checking** (spin) until it's 0.

```c
spin_lock( &lock )
    /* Critical Section */
spin_unlock( &lock )
```

That's **busy-waiting**, which is normally **wasteful**: better to let another thread run. **Except** when the wait is **shorter than** the cost of **blocking, switching to another thread, and unblocking** later. Then spinning **wins**.

#### Reader-Writer Spinlocks
Like the **readers-writers** problem: **many readers** or **one writer**. Implemented as a **24-bit reader counter** plus an **unlock flag**:

| Counter | Flag | Meaning |
| --- | --- | --- |
| 0 | 1 | **Released**, available |
| 0 | 0 | Held for **writing** |
| *n* > 0 | 0 | Held for **reading** by ***n*** threads |
| *n* > 0 | 1 | **Invalid** |

More detail is in the **Linux kernel documentation**.
