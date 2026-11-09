## Classical Synchronization Problems
OS textbooks have a few "**classical problems**": scenarios phrased in **real-world terms** that are really **analogies** for problems the OS deals with. They're the standard **test cases** for any new synchronization scheme, and the solutions use **semaphores** for mutual exclusion. We'll look at three in depth:
1. **Producer-consumer**
2. **Readers-writers**
3. **Dining philosophers**

## The Producer-Consumer Problem
The most common one, also called the ***bounded-buffer problem***. Two processes share a **fixed-size buffer**:
- The ***producer*** generates data and **puts it in** the buffer.
- The ***consumer*** **takes data out** of the buffer.

It generalizes to ***p*** producers and ***c*** consumers, but start with **one of each**.

Rules:
- **Can't write to a full buffer**: with capacity *N* and *N* items in it, the producer must **wait for space**.
- **Can't read from an empty buffer**: with 0 items, the consumer must **wait for an item**.

### Attempt 1: Busy-Waiting
Track the number of items in a shared **`count`** (so it needs **mutual exclusion**), with capacity **`BUFFER_SIZE`**. If we **don't care about wasting CPU time**, one `mutex` is enough. Each thread probably loops forever, but one iteration is enough to explain it:

| Producer | Consumer |
| --- | --- |
| 1. `[produce item]` | 1. `removed = false` |
| 2. `added = false` | 2. `while removed is false` |
| 3. `while added is false` | 3. `    wait( mutex )` |
| 4. `    wait( mutex )` | 4. `    if count > 0` |
| 5. `    if count < BUFFER_SIZE` | 5. `        [remove item from buffer]` |
| 6. `        [add item to buffer]` | 6. `        count--` |
| 7. `        count++` | 7. `        removed = true` |
| 8. `        added = true` | 8. `    end if` |
| 9. `    end if` | 9. `    post( mutex )` |
| 10. `    post( mutex )` | 10. `end while` |
| 11. `end while` | 11. `[consume item]` |

It works, but it's **inefficient**: a thread with nothing to do keeps **spinning** through the loop.

### Attempt 2: Two General Semaphores
New rule: **no busy-waiting**. A producer waiting for space, or a consumer waiting for an item, should be **blocked**. Use two **general semaphores**:
- **`items`**: starts at **0**. How many slots are **full**.
- **`spaces`**: starts at **`BUFFER_SIZE`**. How many slots are **empty** (the mirror image).

| Producer | Consumer |
| --- | --- |
| 1. `[produce item]` | 1. `wait( items )` |
| 2. `wait( spaces )` | 2. `[remove item from buffer]` |
| 3. `[add item to buffer]` | 3. `post( spaces )` |
| 4. `post( items )` | 4. `[consume item]` |

The producer keeps going **until the buffer is full**, and the consumer keeps going **until it's empty**. This works under **two assumptions**:
1. Adding and removing always use the **"next" slot**.
2. There's **exactly one producer and one consumer**.

With **two producers**, they could both try to write into the **same slot** at the same time.

### Attempt 3: Adding a Mutex
For **multiple producers and consumers**, add a **binary semaphore `mutex`** (starts at **1**), combining the two previous attempts:

| Producer | Consumer |
| --- | --- |
| 1. `[produce item]` | 1. `wait( items )` |
| 2. `wait( spaces )` | 2. `wait( mutex )` |
| 3. `wait( mutex )` | 3. `[remove item from buffer]` |
| 4. `[add item to buffer]` | 4. `post( mutex )` |
| 5. `post( mutex )` | 5. `post( spaces )` |
| 6. `post( items )` | 6. `[consume item]` |

This should set off **alarm bells**: **one `wait` inside another** is the warning sign for **deadlock**. It's **not a guarantee** of deadlock (that would be too easy), but it means **analyze the code**.

How to analyze: try **proof by contradiction**. Try to build a scenario that **deadlocks**. If you can't, you **may** have a good solution (best to get **someone else to check** too). Not a formal proof, but the reasoning should be **convincing**. Here, it **won't get stuck**: a thread only waits on `mutex` **after** it already got past `spaces`/`items`, and nobody **blocks while holding `mutex`**.

### A Broken Alternative: Swapped Waits

| Producer | Consumer |
| --- | --- |
| 1. `[produce item]` | 1. `wait( mutex )` |
| 2. `wait( mutex )` | 2. `wait( items )` |
| 3. `wait( spaces )` | 3. `[remove item from buffer]` |
| 4. `[add item to buffer]` | 4. `post( spaces )` |
| 5. `post( items )` | 5. `post( mutex )` |
| 6. `post( mutex )` | 6. `[consume item]` |

Same as before, but the **`wait`s are swapped**. This one **deadlocks**:
1. The buffer is **empty** and the **consumer runs first**. It gets `mutex`, then **blocks on `items`**.
2. The producer runs and **blocks on `mutex`**, because the consumer is holding it.
3. The producer can **never produce**, so the consumer is **never unblocked**. Deadlock.

This can happen **any time the buffer is empty** (and symmetrically, with the producer, when it's full).

It **won't deadlock every time**. It'll probably work **most of the time**. But <u>*one deadlock scenario is enough to reject a solution*</u>: no need to find more failure cases, just replace it.

> [!tip] Same mistake as the barrier (beyond the lecture)
> This is the L14 rule again: **don't block on a semaphore while holding a mutex**. Waiting for a **resource** (`spaces`/`items`) goes **outside** the mutex; only the quick buffer update goes **inside**.

### Producer-Consumer Example
One producer, one consumer, as real C:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <semaphore.h>

#define BUFFER_SIZE 20
sem_t spaces;
sem_t items;
int counter = 0;
int* buffer;

int produce() {
  ++counter;
  return counter;
} 

void consume( int value ) {
  printf("Consumed %d.\n", value);
}

void* producer( void* arg ) {
  int pindex = 0;
  while( counter < 10000 ) {
    int v = produce();
    sem_wait( &spaces );
    buffer[pindex] = v;
    pindex = (pindex + 1) % BUFFER_SIZE;
    sem_post( &items );
  }
  pthread_exit( NULL );
}

void* consumer( void* arg ) {
  int cindex = 0;
  int ctotal = 0;
  while( ctotal < 10000 ) {
    sem_wait( &items );
    int temp = buffer[cindex];
    buffer[cindex] = -1;
    cindex = (cindex + 1) % BUFFER_SIZE;
    sem_post( &spaces );
    consume( temp );
    ++ctotal;
  }
  pthread_exit( NULL );
}

int main( int argc, char** argv ) {
  buffer = malloc( BUFFER_SIZE * sizeof( int ) );
  for ( int i = 0; i < BUFFER_SIZE; i++ ) {
    buffer[i] = -1;
  }  
  sem_init( &spaces, 0, BUFFER_SIZE );
  sem_init( &items, 0, 0 );

  pthread_t prod;
  pthread_t con;

  pthread_create( &prod, NULL, producer, NULL );
  pthread_create( &con, NULL, consumer, NULL );
  pthread_join( prod, NULL );
  pthread_join( con, NULL );

  free( buffer );
  sem_destroy( &spaces );
  sem_destroy( &items );
  pthread_exit( 0 );
}
```

- This is **Attempt 2** in C: **`spaces`** starts at **`BUFFER_SIZE`**, **`items`** at **0**, and there's **no mutex**.
- The buffer is a **circular buffer**: `pindex` and `cindex` wrap around with **`% BUFFER_SIZE`**, which is what "always use the **next** slot" means.
- **`pindex`** and **`cindex`** are **local**: only the producer touches `pindex` and only the consumer touches `cindex`. The semaphores guarantee they're **never on the same slot** at the wrong time.
- **`[produce item]`** (`produce()`) and **`[consume item]`** (`consume()`) are **outside** the semaphore calls, same as the pseudocode.
- Empty slots are marked **`-1`**, just to make it visible which slots hold data.

### Mutex Syntax
You can use a semaphore as a mutex, but usually we use the **specialized tool**. The type is **`pthread_mutex_t`**. We don't care about its internals: it's **locked or unlocked**, that's all.

```c
pthread_mutex_init( pthread_mutex_t *mutex, pthread_mutexattr_t *attributes )
pthread_mutex_lock( pthread_mutex_t *mutex )
pthread_mutex_trylock( pthread_mutex_t *mutex ) /* Returns 0 on success */
pthread_mutex_unlock( pthread_mutex_t *mutex )
pthread_mutex_destroy( pthread_mutex_t *mutex )
```

**`pthread_mutex_init`**: sets up the mutex you pass a pointer to, and returns an `int`. The **attributes** are optional (mostly about **priorities**); **`NULL`** is fine. There's also a **static initializer** shortcut if you don't need attributes:

```c
pthread_mutex_t mymutex = PTHREAD_MUTEX_INITIALIZER;
```

A new mutex starts **unlocked**.

| Function | Behaviour |
| --- | --- |
| **`pthread_mutex_lock`** | **Blocking**. If locked, the thread **blocks** until its turn. If unlocked, it **locks it** and enters. |
| **`pthread_mutex_trylock`** | **Non-blocking**. Not needed yet; comes back with another classical problem. |
| **`pthread_mutex_unlock`** | Unlocks it. **Error** to unlock a mutex that **isn't locked**, or one **owned by another thread**. |
| **`pthread_mutex_destroy`** | Cleans it up when you're done. |

> [!tip] What `trylock` does (beyond the lecture)
> If the mutex is free, `trylock` **locks it and returns 0**, same as `lock`. If it's already locked, it **returns `EBUSY` immediately** instead of blocking, so the thread can go do something else.

> [!warning] "Error" depends on the mutex type (beyond the lecture)
> As in L13: with the **default** type, unlocking an unlocked mutex or another thread's mutex is **undefined behaviour**, not a guaranteed error. Only **`PTHREAD_MUTEX_ERRORCHECK`** reliably **returns an error**.

If you created attributes with **`pthread_mutexattr_init`**, destroy them with **`pthread_mutexattr_destroy`**.

Destroying an **unlocked** mutex is fine. Destroying a **locked** one is ***undefined behaviour***: "the worst thing ever". The code might work **some of the time**, or **on some systems**, or **fine for a while** and then break suddenly when something unrelated changes. (C and POSIX are **full** of these, e.g. reading an **uninitialized variable** in C.)

### Parallelizing the Producer-Consumer Solution
Now **ten producers and ten consumers**. This is **Attempt 3** in C, with a **`pthread_mutex_t`** as the mutex:

```c
#include <stdlib.h>
#include <pthread.h>
#include <stdio.h>
#include <math.h>
#include <semaphore.h>

#define BUFFER_SIZE 100
int buffer[BUFFER_SIZE];
int pindex = 0;
int cindex = 0;
sem_t spaces;
sem_t items;
pthread_mutex_t mutex;

int produce( int id ) {
  int r = rand();
  printf("Producer %d produced %d.\n", id, r);
  return r;
}

void consume( int id, int number ) {
  printf("Consumer %d consumed %d.\n", id, number);
}

void* producer( void* arg ) {
  int* id = (int*) arg;
  for(int i = 0; i < 10000; ++i) {
    int num = produce(*id); 
    sem_wait( &spaces );
    pthread_mutex_lock( &mutex );
    buffer[pindex] = num;
    pindex = (pindex + 1) % BUFFER_SIZE;
    pthread_mutex_unlock( &mutex );
    sem_post( &items );
  }
  free( arg );
  pthread_exit( NULL );
}

void* consumer( void* arg ) {
  int* id = (int*) arg;
  for(int i = 0; i < 10000; ++i) {
    sem_wait( &items );
    pthread_mutex_lock( &mutex );
    int num = buffer[cindex];
    buffer[cindex] = -1;
    cindex = (cindex + 1) % BUFFER_SIZE;
    pthread_mutex_unlock( &mutex );
    sem_post( &spaces );
    consume( *id, num );
  }
  free( id );
  pthread_exit( NULL );
}

int main( int argc, char** argv ) {
  sem_init( &spaces, 0, BUFFER_SIZE );
  sem_init( &items, 0, 0 );  
  pthread_mutex_init( &mutex, NULL );

  pthread_t threads[20];

  for( int i = 0; i < 10; i++ ) {
    int* id = malloc(sizeof(int));
    *id = i;
    pthread_create(&threads[i], NULL, producer, id);
  }
  for( int j = 10; j < 20; j++ ) {
    int* jd = malloc(sizeof(int));
    *jd = j-10;
    pthread_create(&threads[j], NULL, consumer, jd);
  }
  for( int k = 0; k < 20; k++ ){  
    pthread_join(threads[k], NULL);
  }
  sem_destroy( &spaces );
  sem_destroy( &items );
  pthread_mutex_destroy( &mutex );
  pthread_exit( 0 );
}
```

What changed from the 1-and-1 version:
- **`pindex`** and **`cindex`** are now **globals**, since many producers share `pindex` and many consumers share `cindex`. So the slot access and index update go **inside the mutex**.
- Order is **semaphore wait, then lock** (the **working** order), and **unlock, then semaphore post**.
- Each thread gets its **own `malloc`'d id** (not `&i`, which would change under it as the loop continues), and **frees it** when done.
- **10 producers × 10,000** items = **10 consumers × 10,000** items, so every consumer finishes and the joins return.

> [!tip] One mutex or two? (beyond the lecture)
> Producers only touch `pindex` and consumers only touch `cindex`, and the semaphores already keep them off each other's slots. So you could use **one mutex for producers and another for consumers**, letting a producer and a consumer work **at the same time**. One shared mutex is simpler and still correct.
