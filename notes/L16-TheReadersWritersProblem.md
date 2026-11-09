## The Readers-Writers Problem
Concurrent **reading** and **modifying** of a shared data structure or record:
- A ***writer*** **modifies** the data.
- A ***reader*** only **reads** it, without modifying.

Unlike producer-consumer, **some concurrency is allowed**:
1. **Any number of readers** may be in the critical section at once.
2. **Only one writer** may be in the critical section, and while it is, **no readers** are allowed either.

In short: <u>*a writer can't enter while anyone else (reader or writer) is inside, and while a writer is inside, nobody else can enter*</u>.

This is very often how **file systems** work: a file can be **read by any number of threads** at once, but **only one can write** at a time, and **nobody reads during a write** (so nobody sees **inconsistent data**).

How it differs from what we've seen:
- **Producer-consumer**: consumers **do modify** the buffer (they take things out). Readers **don't**.
- **General mutual exclusion**: would **work** (no errors), but it's a **serious performance hit** compared to letting readers run **concurrently**. So this problem is worth solving on its own.

### Attempt 1: The Light Switch
We need:
- **`readers`**: how many readers are in the room right now.
- **`mutex`**: a binary semaphore protecting `readers`.
- **`roomEmpty`**: signals that the room is **empty**. A writer must **`wait( roomEmpty )`** before entering.

| Writer | Reader |
| --- | --- |
| 1. `wait( roomEmpty )` | 1. `wait( mutex )` |
| 2. `[write data]` | 2. `readers++` |
| 3. `post( roomEmpty )` | 3. `if readers == 1` |
| | 4. `    wait( roomEmpty )` |
| | 5. `end if` |
| | 6. `post( mutex )` |
| | 7. `[read data]` |
| | 8. `wait( mutex )` |
| | 9. `readers--` |
| | 10. `if readers == 0` |
| | 11. `    post( roomEmpty )` |
| | 12. `end if` |
| | 13. `post( mutex )` |

**Writer**: only enters if the room is **empty**, and says the room is empty when it **leaves**. It can be sure the room is empty when it leaves, because **nobody could enter** while it was in there.

**Reader**:
- The **first reader** finds the room empty and **"locks" it** (`wait( roomEmpty )`), which **keeps writers out**.
- **Later readers** don't check; they just **walk in**.
- The **last reader out** posts `roomEmpty`, **"unlocking"** the room for a writer.

This is the ***light switch*** pattern: <u>*the first one in turns on the lights, the last one out turns them off*</u>.

> [!tip] Why `roomEmpty` has to be a semaphore (beyond the lecture)
> The reader that **waits** on `roomEmpty` (the first one in) is often **not** the reader that **posts** it (the last one out). A mutex only lets the **owner** unlock it, so `roomEmpty` must be a plain **semaphore**, like the **signalling** pattern in L14. That's why the C version below uses a `pthread_mutex_t` for `mutex` but a `sem_t` for `roomEmpty`.

#### Deadlock?
The reader has the warning sign: a **`wait( roomEmpty )` inside the `mutex` critical section**. But it's safe:
- A reader only waits on `roomEmpty` when a **writer is inside**.
- Other readers can't get `mutex` meanwhile, but the writer **doesn't need `mutex`**.
- The write takes **finite time**, so the writer eventually **posts `roomEmpty`** and everyone continues.

**No deadlock.**

#### Starvation
A different problem. Some readers are in the room and a **writer arrives**. It must wait for **all readers to leave**. But while those readers finish and leave, **new readers keep arriving** and walking in. Each reader only stays a **finite time**, yet the room is **never empty**.

This **isn't deadlock** (the readers aren't stuck), but the writer, and any writers after it, **wait forever**. This is ***starvation***: <u>*a thread never gets a chance to run*</u>.

It breaks **criterion 3** from L13: a thread wanting the critical section must **not be delayed indefinitely**. Starvation is **as bad as deadlock** here: if it's possible at all, even if **unlikely**, the solution is **rejected**.

### Attempt 2: No-Starve Readers-Writers
Idea: when a writer arrives, readers **already inside** may **finish**, but **no new readers** may start. The room eventually empties, the writer goes, and then the readers who arrived **after** it go in.

New binary semaphore: **`turnstile`**.

| Writer | Reader |
| --- | --- |
| 1. `wait( turnstile )` | 1. `wait( turnstile )` |
| 2. `wait( roomEmpty )` | 2. `post( turnstile )` |
| 3. `[write data]` | 3. `wait( mutex )` |
| 4. `post( turnstile )` | 4. `readers++` |
| 5. `post( roomEmpty )` | 5. `if readers == 1` |
| | 6. `    wait( roomEmpty )` |
| | 7. `end if` |
| | 8. `post( mutex )` |
| | 9. `[read data]` |
| | 10. `wait( mutex )` |
| | 11. `readers--` |
| | 12. `if readers == 0` |
| | 13. `    post( roomEmpty )` |
| | 14. `end if` |
| | 15. `post( mutex )` |

**Writer**:
- Waits on the **turnstile**. Later writers **queue up there**.
- Once through, it **holds the turnstile locked**, so **no new readers** get in, then waits for the room to be empty.
- The readers already inside each stay a **finite time**, so they **finish and leave**, and the writer gets its turn.
- When done, it posts the turnstile, which may unblock a **reader or another writer**.

**Reader**: passes through the **turnstile** (it **queues** there if a writer has it locked). After that it's the **same light switch** as before.

**Starvation**: Attempt 1's one starvation scenario is **fixed**, and neither writers nor readers can starve now.

**Deadlock**: the reader barely changed, and the turnstile alone isn't a problem. The risky part is the **writer's two `wait`s**. If the writer is blocked on `roomEmpty`:
- **Nobody** (reader or writer) can get past the turnstile.
- But being blocked there means **readers are inside**, and they finish **unimpeded**.
- So the room **eventually empties** and the writer is **unblocked**. **No deadlock.**

**No writer priority**: when a writer leaves, posting the turnstile might wake a **reader**, and then a **whole bunch of readers** may enter before the next writer locks the turnstile again. So readers can proceed **even with a writer queued**. Whether that's OK **depends on the application**.

### Attempt 3: Writer Priority
Give **writers priority** over readers (this may **starve readers**, but ignore that). Split `roomEmpty` into **`noReaders`** and **`noWriters`**:
- A **reader** in the critical section holds **`noWriters`**.
- A **writer** in the critical section holds **both `noWriters` and `noReaders`**.

| Writer | Reader |
| --- | --- |
| 1. `wait( writeMutex )` | 1. `wait( noReaders )` |
| 2. `writers++` | 2. `wait( readMutex )` |
| 3. `if writers == 1` | 3. `readers++` |
| 4. `    wait( noReaders )` | 4. `if readers == 1` |
| 5. `end if` | 5. `    wait( noWriters )` |
| 6. `post( writeMutex )` | 6. `end if` |
| 7. `wait( noWriters )` | 7. `post( readMutex )` |
| 8. `[write data]` | 8. `post( noReaders )` |
| 9. `post( noWriters )` | 9. `[read data]` |
| 10. `wait( writeMutex )` | 10. `wait( readMutex )` |
| 11. `writers--` | 11. `readers--` |
| 12. `if writers == 0` | 12. `if readers == 0` |
| 13. `    post( noReaders )` | 13. `    post( noWriters )` |
| 14. `end if` | 14. `end if` |
| 15. `post( writeMutex )` | 15. `post( readMutex )` |

The writer got **much more complicated**. The reader is **barely different**, and the writer is now roughly its **mirror image**.

> [!tip] How it works (beyond the lecture)
> All four semaphores start at **1**; both counts start at **0**.
> - Writers use a **light switch on `noReaders`**: the **first writer** to arrive locks out **new readers** (they block at reader line 1), and the **last writer** to leave lets them back in. In between, writers take turns one at a time with **`noWriters`**.
> - Readers use a **light switch on `noWriters`**, same as `roomEmpty` before.
> - A reader holds `noReaders` only **briefly** (lines 1–8), just long enough to register. So a newly arrived writer can grab `noReaders` **between** readers, then just waits for the readers already inside to drain out.
> - As long as writers **keep arriving**, `writers` never hits 0 and readers **never get back in**. That's the reader starvation we're ignoring.

## Readers-Writers Locks in pthreads
We **can** build all of this from semaphores (optionally swapping a mutex in for some), but pthreads has a **readers-writers lock** built in, so we don't **reinvent the wheel**. Given how complex the last solution was, the hope is that it's **easier to read** and **harder to get wrong**.

The type is **`pthread_rwlock_t`** (like `pthread_mutex_t`):

```c
pthread_rwlock_init( pthread_rwlock_t * rwlock, pthread_rwlockattr_t * attr )
pthread_rwlock_rdlock( pthread_rwlock_t * rwlock )
pthread_rwlock_tryrdlock( pthread_rwlock_t * rwlock )
pthread_rwlock_wrlock( pthread_rwlock_t * rwlock )
pthread_rwlock_trywrlock( pthread_rwlock_t  *  rwlock )
pthread_rwlock_unlock( pthread_rwlock_t * rwlock )
pthread_rwlock_destroy( pthread_rwlock_t * rwlock )
```

Very similar to the mutex (attribute init/destroy exist too, not shown). The differences:
- **lock** and **trylock** are each split into a **read** version (`rdlock`, `tryrdlock`) and a **write** version (`wrlock`, `trywrlock`).
- The **same thread** can take the read lock ***n*** times, as long as it **unlocks *n*** times.
- There's only **one `unlock`**: you don't say whether you're releasing a read or write lock. It **figures out** which kind the calling thread holds, like how `close()` doesn't need to be told what kind of thing it's closing.

**Who gets priority** is **implementation defined**. The spec says that, if possible, for threads of **equal priority**, a **writer goes first**. But **your system may vary**.

> [!warning] Linux prefers readers by default (beyond the lecture)
> The "writer goes first" rule in POSIX only strictly applies with **real-time scheduling** (`SCHED_FIFO`/`SCHED_RR`). On **Linux (glibc)**, the default rwlock actually **prefers readers**, so a steady stream of readers **can starve writers**, just like Attempt 1. glibc has a non-portable **`pthread_rwlockattr_setkind_np`** to switch to writer preference.

### Readers-Writers Example
The **simple** readers-writers (Attempt 1: **no** writer priority, **can starve**), the "old" way:

```c
int readers;
pthread_mutex_t mutex;
sem_t roomEmpty;

void init( ) {
  readers = 0;
  pthread_mutex_init( &mutex, NULL );
  sem_init( &roomEmpty, 0, 1 );
}

void cleanup( ) {
  pthread_mutex_destroy( &mutex );
  sem_destroy( &roomEmpty );
}

void* writer( void* arg ) {
  sem_wait( &roomEmpty );
  write_data( arg );
  sem_post( &roomEmpty );
}

void* reader( void* read ) {
  pthread_mutex_lock( &mutex );
  readers++;
  if ( readers == 1 ) {
    sem_wait( &roomEmpty );
  }
  pthread_mutex_unlock( &mutex );
  read_data( read );
  pthread_mutex_lock( &mutex );
  readers--;
  if ( readers == 0 ) {
    sem_post( &roomEmpty );
  }
  pthread_mutex_unlock( &mutex );
}
```

The same thing with a **rwlock**:

```c
pthread_rwlock_t rwlock;

void init( ) {
  pthread_rwlock_init( &rwlock, NULL );
}

void cleanup( ) {
  pthread_rwlock_destroy( &rwlock );
}

void* writer( void* arg ) {
  pthread_rwlock_wrlock( &rwlock );
  write_data( arg );
  pthread_rwlock_unlock( &rwlock );
}

void* reader( void* read ) {
  pthread_rwlock_rdlock( &rwlock );
  read_data( read );
  pthread_rwlock_unlock( &rwlock );
}
```

No counter, no light switch, no separate mutex: just **lock, work, unlock**.

## Seek and Destroy: the Search-Insert-Delete Problem
An **extension** of readers-writers with **three** kinds of thread instead of two, all working on a **shared linked list**:

| Thread | Calls | Rules | Most like |
| --- | --- | --- | --- |
| ***Searcher*** | `void search( void* target )` | Only **examines** the list, so searchers can run **concurrently with each other**. | **Readers** |
| ***Inserter*** | `node* find_insert_location()`, then `void insert( void* to_insert, node* after )` | Adds items to the **end**. **One insert at a time**, but an insert can run **in parallel with any number of searches** (`insert` is written to allow that). | **Readers**, plus **one-at-a-time** among themselves |
| ***Deleter*** | `void delete( void* to_delete )` | Removes items from **anywhere**. **One deleter at a time**, and while it's working, **no searchers and no inserters**. | **Writers** |

We don't need to change much. Track when there are "**no searchers**" and "**no inserters**" (hints for the semaphores), plus **another mutex around the actual insertion**:

```c
pthread_mutex_t searcher_mutex;
pthread_mutex_t inserter_mutex;
pthread_mutex_t perform_insert;
sem_t no_searchers;
sem_t no_inserters;
int searchers;
int inserters;


void init( ) {
  pthread_mutex_init( &searcher_mutex, NULL );
  pthread_mutex_init( &inserter_mutex, NULL );
  pthread_mutex_init( &perform_insert, NULL );
  sem_init( &no_inserters, 0, 1 );
  sem_init( &no_searchers, 0, 1 );
  searchers = 0;
  inserters = 0;
}

void* searcher_thread( void *target ) {
  pthread_mutex_lock( &searcher_mutex );
  searchers++;
  if ( searchers == 1 ) {
    sem_wait( &no_searchers );
  }
  pthread_mutex_unlock( &searcher_mutex );
  
  search( target );
  
  pthread_mutex_lock( &searcher_mutex );
  searchers--;
  if ( searchers == 0 ) {
    sem_post( &no_searchers );
  }
  pthread_mutex_unlock( &searcher_mutex );
}

void* deleter_thread( void* to_delete ) {
  sem_wait( &no_searchers );
  sem_wait( &no_inserters );
  
  delete( to_delete );
  
  sem_post( &no_inserters );
  sem_post( &no_searchers );
}


void* inserter_thread( void *to_insert ) {
  pthread_mutex_lock( &inserter_mutex );
  inserters++;
  if ( inserters == 1 ) {
    sem_wait( &no_inserters );
  }
  pthread_mutex_unlock( &inserter_mutex );
  
  node * insert_after = find_insert_location( );
  pthread_mutex_lock( &perform_insert );
  insert( to_insert, insert_after ); /* Can update its position if needed */
  pthread_mutex_unlock( &perform_insert );
  
  pthread_mutex_lock( &inserter_mutex );
  inserters--;
  if ( inserters == 0 ) {
    sem_post( &no_inserters );
  }
  pthread_mutex_unlock( &inserter_mutex );
}
```

- **Searchers**: a **light switch** on **`no_searchers`**. The first searcher in locks out deleters; the last one out lets them back in.
- **Inserters**: a **light switch** on **`no_inserters`**, same idea. Then **`perform_insert`** makes the actual **`insert`** one-at-a-time. `find_insert_location` runs **outside** it, so inserters only serialize for the part that **changes the list**.
- **Deleters**: like a **writer**, but it needs the room empty of **both** kinds, so it waits on **`no_searchers` and `no_inserters`**.
- Searchers and inserters **never touch each other's semaphores**, so they **run in parallel**.

**Question**: could you implement this with a **`pthread_rwlock_t`**, despite there being **three** kinds of thread?

> [!tip] Answer: yes (beyond the lecture)
> Searchers and inserters can both be in the list at once, so they're both **"readers"**: each takes **`pthread_rwlock_rdlock`**. Deleters are the **"writers"**: **`pthread_rwlock_wrlock`**. The only extra rule is **one insert at a time**, so inserters still lock **`perform_insert`** around `insert`. That replaces both light switches and both semaphores with **one rwlock plus one mutex**.

> [!warning] Deleters can starve (beyond the lecture)
> Like Attempt 1 of readers-writers, there's **no turnstile**: a steady stream of searchers or inserters keeps `no_searchers` or `no_inserters` locked, so a deleter can **wait forever**.
