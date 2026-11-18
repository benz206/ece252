## How Files Are Stored
To understand concurrency in file systems, we need a **high-level** idea of how they're **implemented**. Files can be **any size** (up to some file-system limit), so they're allocated on disk by some **strategy**.

### Contiguous Allocation
A file takes up **consecutive blocks**. Starting at block *b* with *n* blocks: $b, b+1, b+2, \dots, b+(n-1)$.
- A **big** file might be hard to **find room for**.
- We **can't predict** how big a file will get, so how much space do we **leave** for it?

### Linked Allocation
Keep a **linked list** of blocks, which can be **anywhere** on disk. The directory just stores pointers to the **first and last** blocks (head and tail).
- Getting to block *i* means **following *i* pointers**. To reach the middle of a file, why load **every block on the way**?

### Indexed Allocation
The compromise: put **all the pointers in one place**, an ***index block***. To get block *i*, look at **entry *i*** of the index block. Pointers start as **null**, and an entry is filled in when a block is **added**. Variations:

1. **Linked scheme**: an index block is a disk block, and several can be **linked together**. The last entry is **null** or a **pointer to the next index block**.
2. **Multilevel index**: the first-level block points to **second-level blocks**, which point to the **data**. As many levels as the max file size needs. With **4 KB blocks** and **4-byte pointers**, a block holds **1024** pointers, so two levels gives $1024 \times 1024 \times 4\text{ KB} =$ **4 GB** max.
3. **Combined scheme**: all of the above. **Used in UNIX.** The ***inode*** holds the first **15 pointers**:
	- **12 direct**: point straight to file data.
	- **13th**: ***single indirect***: an index block of **data block addresses**.
	- **14th**: ***double indirect***: addresses of blocks containing **addresses of blocks**.
	- **15th**: ***triple indirect***. (Yo dawg, we heard you like index blocks...)

![[unix-inode.png]]

An inode is just another **data structure**, except it lives in **persistent storage**. Keep it in mind: <u>*when we lock a file, we're really locking an inode*</u>.

## Record Locking with `fcntl`
**`flock()`** (from L03) locks the **entire file**. **`fcntl`** can lock just a **byte range**: ***record locking***.

More concurrency: one process can write the **beginning** of a file while another writes the **end**. They **don't overlap**, so the second doesn't have to **wait**.

```c
#include <fcntl.h>
int fcntl( int file_descriptor, int command, ... /* struct flock * flockptr */ );
```

Strange signature! **`fcntl` does a lot of things**, and only **some commands** need the third argument. So it's a `...` (variable args), and we pass one **`struct flock*`** when the command needs it. (They could have just overloaded it, but here we are.)

```c
struct flock {
  short  l_type;   /* F_RDLCK, F_WRLCK, or F_UNLCK */
  short  l_whence; /* SEEK_SET, SEEK_CUR, or SEEK_END */
  off_t  l_start;  /* offset in bytes, relative to l_whence */
  off_t  l_len;    /* length, in bytes; 0 means lock to EOF */
  pid_t  l_pid;    /* returned with F_GETLK */
};
```

### Lock Types
Like **readers-writers** (L16): **read** and **write** locks.

|  | Read held | Write held |
| --- | --- | --- |
| **Request read** | ✅ | ❌ |
| **Request write** | ❌ | ❌ |

- **Unlock** by "setting" a lock with type **`F_UNLCK`**.
- **Deadlock is possible**: process A locks file 1 and wants file 2, while B locks file 2 and wants file 1.

### Where the Range Starts
**`l_whence`** says where **`l_start`** is measured from:
- **`SEEK_SET`**: the **start** of the file. `SEEK_SET` + `1000` means the region starts **1000 bytes in**.
- **`SEEK_END`**: the **end** of the file.
- **`SEEK_CUR`**: the **current position** (makes sense if you've moved around with `seek()`).

A locked region **can extend past the end of the file**. That's for **appending**: you don't need to know in advance **how much** you'll add. **`l_len = 0`** means "to EOF", and **includes anything appended later**.

### Commands
- **`F_GETLK`**: is the lock described by `flockptr` **blocked** by another lock? If so, `flockptr` is **overwritten** with that lock's info. If not, `l_type` is set to **`F_UNLCK`**.
- **`F_SETLK`**: **set** the lock. If it can't be acquired, returns an **error** and sets **`errno`**. This is ***trylock*** behaviour, which can be used to **avoid deadlock**.
- **`F_SETLKW`**: the **blocking** (**W**ait) version of `F_SETLK`. If the region is in use, the caller **blocks**.

**Partial unlocking** works too (unusual, but why not?). The system **combines or splits** locks as needed.

### Examples
Write-lock the whole file, then unlock it:

```c
int write_lock_file( int fd ) {
  struct flock fl;
  fl.l_type = F_WRLCK;
  fl.l_start = 0;
  fl.l_whence = SEEK_SET;
  fl.l_len = 0;
  
  return fcntl( fd, F_SETLK, &fl );
}

int unlock_file( int fd ) {
  struct flock fl;
  fl.l_type = F_UNLCK;
  fl.l_start = 0;
  fl.l_whence = SEEK_SET;
  fl.l_len = 0;
  
  return fcntl( fd, F_SETLK, &fl );
}
```

Check whether bytes **1024–1279** are locked:

```c
int fd = open ( "example.txt", O_RDONLY );
struct flock lock;

lock.l_type = F_RDLCK;
lock.l_start = 1024;
lock.l_whence = SEEK_SET;
lock.l_len = 256;

fcntl( fd, F_GETLK, &lock );
if ( lock.l_type == F_UNLCK ) {
  /* Lock is unlocked; we may proceed */
} else if ( lock.l_type == F_WRLCK ) {
  /* File is write locked by a different process */
  printf( "File locked by process ID %d.\n", lock.l_pid );
  return -1;
}
```

> [!warning] `F_GETLK` is for information only
> **Don't decide whether to proceed** based on `F_GETLK`. "Check, then act" **isn't atomic**: someone can grab the lock **in between** (the same check-then-act race from L12). Instead:
> - **`F_SETLK`**: actually **try** to lock. **`-1`** means it failed.
> - **`F_SETLKW`**: if the plan is to **wait** anyway.

**`fcntl` modifies the `struct flock`**, so **reset it** before reusing it. You *can* reuse the same struct to unlock what you locked, just **carefully**.

> [!warning] `fcntl` locks are per-process (beyond the lecture)
> - Locks belong to the **process**, so **threads in the same process don't block each other**. Use a mutex between threads.
> - **Closing *any* fd** for that file **releases all** of the process's locks on it, even ones taken through a **different** fd.
> - Locks are **not inherited** by a **`fork()`**ed child.
> - When `F_SETLK` fails because of a conflict, `errno` is **`EACCES`** or **`EAGAIN`** (check both).

## `lockf`
A **simplified** way to lock. `fcntl` is more flexible, but sometimes **simple is all we need**.

```c
int lockf( int file_descriptor, int command, off_t length );
```

| Command | Meaning |
| --- | --- |
| **`F_LOCK`** | Acquire an **exclusive** lock on the (section of the) file. |
| **`F_TLOCK`** | **Try** to acquire an exclusive lock (trylock). |
| **`F_ULOCK`** | **Unlock** the section. |
| **`F_TEST`** | Is it locked? **0** = unlocked, **-1** = locked. |

- **`length`** is measured from the **current position** in the file. **0** locks the whole file.
- The file is **automatically unlocked** when the **fd is closed**.
- On some systems `lockf` **just calls `fcntl`**; on others it's a **different mechanism**. <u>*Don't mix and match: unlock with the same function you locked with.*</u>

> [!note] What `length = 0` really means (beyond the lecture)
> It locks from the **current position to the end of the file** (and beyond, as it grows). That's the **whole file** only if you're at **position 0**.

### Advisory Locks
**Both** kinds of lock are ***advisory***: like a semaphore or mutex, they only work if **everyone** follows the protocol and **checks** before accessing. Nothing stops a process from **ignoring** the lock and writing anyway.

***Mandatory*** locks exist, but they're **hard to use** and **not recommended**. (See the [kernel.org docs](https://www.kernel.org/doc/Documentation/filesystems/mandatory-locking.txt) for why not, or how, if you must.)

> [!tip] Mandatory locking is gone (beyond the lecture)
> Linux **removed** mandatory locking support entirely in **kernel 5.15** (2021).

## Using a File as a Lock
The **existence of a file** can be the lock. **`git`** creates **`index.lock`** while an operation runs, so two `git` clients don't work on the same repo at once.
- File **exists**: "**locked**".
- File **doesn't exist**: "**unlocked**".

The catch: if two processes both **check, then create**, they **both** might succeed. Same race as before. Use the right **flags** and **atomic** calls:

```c
int open( const char *filename, int flags );  /* Returns a file descriptor if successful, -1 on error */
int rename( const char *old_filename, const char *new_filename ); /* Returns 0 on success, operates atomically */
int remove( const char *filename ); /* Deletes a file or directory, returns 0 on success, operates atomically */
```

### `open` Flags
Combine with bitwise OR (`|`):

| Value | Meaning |
| --- | --- |
| **`O_RDONLY`** | Open **read-only** |
| **`O_WRONLY`** | Open **write-only** |
| **`O_RDWR`** | Open for **reading and writing** |
| **`O_APPEND`** | **Append** to the end of the file |
| **`O_TRUNC`** | Initially **clear** all data from the file |
| **`O_CREAT`** | **Create** the file |
| **`O_EXCL`** | With `O_CREAT`: the caller **MUST create** the file; **fails if it exists** |

<u>*`O_CREAT | O_EXCL` is the atomic "create if it doesn't exist"*</u>. Only **one** process can win.

### `open` + `rename`
Programs that **share nothing but a file system** can get lock-like behaviour:
- **Create** the lock file with `open( ..., O_CREAT | O_EXCL ... )` (fails if it exists).
- To let the next process go: **`remove`** it. Or...
- Use **`rename`**, which is also **atomic**. Instead of creating and deleting every time:
	- **Lock**: rename `file.lock` → `file.locked`. If **you** succeeded, it's **your turn**. If it **fails** (`file.lock` doesn't exist), **wait**.
	- **Unlock**: **rename it back**.

Example with **threads** (rather than processes):

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <pthread.h>

#define NUM_THREADS 10

int lock_fd;
int shared = 0;

void* run( void* arg ) {
  int* id = (int*) arg;
  while( rename( "file.lock", "file.locked" ) == -1 ) {
    printf("Thread %d waiting.\n", *id); 
  }

  printf("Thread %d in critical section.\n", *id);
  printf("Shared incremented from %d", shared);
  shared++;
  printf(" to %d.\n", shared);
  rename("file.locked", "file.lock"); /* Unlock */

  free( arg );
  pthread_exit(NULL);
}

void* writer( void* arg ) {
  /* Write data implementation not shown */
  pthread_exit(NULL);
}

int main( int argc, char** argv ) {
  lock_fd = open( "file.lock", O_CREAT | O_EXCL | O_WRONLY, 0644 );
  if (lock_fd == -1 ) {
   printf( "File creation failed.\n");
   return -1;
  }

  pthread_t threads[NUM_THREADS];
  for (int i = 0; i < NUM_THREADS; i++) {
    int * id = malloc( sizeof( int ) );
    *id = i;
    pthread_create( &threads[i], NULL, run, id );
  }
  for (int i = 0; i < NUM_THREADS; i++) {
    pthread_join( threads[i], NULL );
  }
  close( lock_fd );
  remove( "file.lock" );

  return 0;
}
```

- `main` creates **`file.lock`** with **`O_CREAT | O_EXCL`** (fails if it's already there).
- Each thread tries to **rename** it to `file.locked`. Only **one** rename can succeed; the rest get **-1** and loop.
- The winner increments **`shared`**, then **renames it back** to unlock.
- At the end, `main` **removes** the lock file.

> [!warning] Things to watch for (beyond the lecture)
> - The `while` loop is **busy-waiting**: waiting threads spin and print as fast as they can. (L25's **inotify** is one way to wait without spinning.)
> - If a process **crashes while holding** a lock file, the file **stays there** and everyone waits **forever**. That's why git sometimes says *"Another git process seems to be running... remove the file manually"*. Writing your **PID** into the lock file helps you check if the holder is still alive.
