## AIO
L26: opening a **file** with `O_NONBLOCK` doesn't do what we want. The **POSIX Asynchronous I/O interface** (***AIO***) does.

The approach:
1. Create a ***control block*** (**`struct aiocb`**, AIO Control Block) describing the operation.
2. **Enqueue** it: tell the OS you want it to happen.
3. Either set up a **callback** to run when it's done, or **check periodically** whether it's done. Your choice, depending on the application.

## The Control Block

```c
struct aiocb {
  int aio_fildes;               /* File descriptor */
  off_t aio_offset;             /* Offset for I/O */
  volatile void* aio_buf;       /* Buffer */
  size_t aio_nbytes;            /* Number of bytes to transfer */
  int aio_reqprio;              /* Request priority */
  struct sigevent aio_sigevent; /* Signal Info */
  int aio_lio_opcode;           /* Operation for List I/O */
};
```

| Field | Meaning |
| --- | --- |
| **`aio_fildes`** | The **file descriptor** to read/write |
| **`aio_offset`** | How far **into the file** to start |
| **`aio_buf`** | **Destination** (read) or **source** (write) |
| **`aio_nbytes`** | **How many bytes** |
| **`aio_reqprio`** | Scheduling **priority** suggestion. The OS can **ignore** it, and **we will too**. |
| **`aio_sigevent`** | What to do when it **completes** (below) |
| **`aio_lio_opcode`** | For **list** I/O (end of lecture) |

The first four are the **mandatory** ones, and look like a normal `read`/`write`.

> [!warning] Offsets and file positions
> - **`aio_offset` does nothing** if the file was opened in **append** mode.
> - AIO advances a file position, but it's **not the same position** as regular blocking `read`/`write`. **Mixing** the two may not do what you want.

### `struct sigevent`

```c
struct sigevent {
  int sigev_notify;                              /* Notify Type */
  int sigev_signo;                               /* Signal number */
  union sigval sigev_value;                      /* Notify argument */
  void (*sigev_notify_function) (union sigval);  /* Notify Function */
  pthread_attr_t *sigev_notify_attributes;       /* Notify attributes */
};
```

**`sigev_notify`** is one of:
- **`SIGEV_NONE`**: do **nothing** when the request completes.
- **`SIGEV_SIGNAL`**: raise the signal in **`sigev_signo`** when it completes.
- **`SIGEV_THREAD`**: run **`sigev_notify_function`** in a **new, detached thread**, with **`sigev_value`** as its argument. The thread's attributes (including detached-ness) can be overridden with **`sigev_notify_attributes`**.

**`SIGEV_THREAD`** is the one we'll use most: **flexible and convenient**.

#### `union sigval`
A ***union*** is defined like a struct, but where a struct is **all** of its fields, a union is **one** of them **at a time** (they share the same memory).

```c
union sigval {
  int sival_int;
  void* sival_ptr;
};
```

So a `sigval` is **either an `int` or a `void*`**.

> [!warning] Not the pthread signature
> The notify function is **`void f( union sigval )`**, not `void* f( void* )` like `pthread_create`. Different **return type** and different **argument type**.

## Enqueue

```c
int aio_read( struct aiocb* aiocb );
int aio_write( struct aiocb* aiocb );
```

Self-explanatory. The return value is about whether **enqueuing** worked, **not** the result of the actual read/write.

<u>*Once enqueued, don't touch the control block or the buffer until the operation finishes*</u>, or you might get **inconsistent results**.

## Are We Done Yet?

```c
int aio_error( const struct aiocb* aiocb );
ssize_t aio_return( const struct aiocb* aiocb );
```

**`aio_error`** should really be called **`aio_status`**:

| Return | Meaning |
| --- | --- |
| **0** | **Completed successfully** |
| **-1** | The `aio_error` call itself **failed**; check `errno` |
| **`EINPROGRESS`** | Still **waiting or in progress** |
| Anything else | The operation **failed**; the value is the **error code** |

**`aio_return`** gets the **return value of the read/write** (e.g. bytes read), once it's **done**:
- Calling it while **in progress**: **undefined**, may fail.
- Call it **only once** per operation. After that, internal structures can be **freed**, so it's polite to call it <u>*even if you don't need the value*</u>.

## Example: Read While You Eat
Process a **list of files** (given on the command line), **overlapping** the read of file *n+1* with **processing** file *n*.
- The **first** file has nothing to overlap with, so it's a **blocking** read.
- Max file size is **`MAX_SIZE`**, so always read that much (reading less is fine).
- **Two buffers**: one being **processed**, one being **read into**. Then **swap**. (***Double buffering***.)
- Assume **no errors** happen, to keep it short.
- No callback: just **check** if it's done and **sleep** if not.

```c
void process( char* buffer ); /* Implementation not shown */

int main( int argc, char** argv ) {
  char* buffer1 = malloc( MAX_SIZE * sizeof( char ));
  char* buffer2 = malloc( MAX_SIZE * sizeof( char ));
  
  int fd = open( argv[1], O_RDONLY );
  memset( buffer1, 0, MAX_SIZE * sizeof( char ));
  read( fd, buffer1, MAX_SIZE );
  close( fd );
  
  for ( int i = 2; i < argc; i++ ) {
    int nextFD = open( argv[i], O_RDONLY );
    
    struct aiocb cb;
    memset( &cb, 0, sizeof( struct aiocb ));
    
    cb.aio_nbytes = MAX_SIZE;
    cb.aio_fildes = nextFD;
    cb.aio_offset = 0;
    memset( buffer2, 0, MAX_SIZE * sizeof( char ));
    cb.aio_buf = buffer2;
    aio_read( &cb );
 
    process( buffer1 );
    
    while( aio_error( &cb ) == EINPROGRESS ) {
      sleep( 1 );
    }
    aio_return( &cb ); /* This frees some internal structures */
    close( nextFD );
    
    char* tmp = buffer1;
    buffer1 = buffer2;
    buffer2 = tmp;
  }
  process( buffer1 );
  free( buffer1 );
  free( buffer2 );

  return 0;
}
```

1. **Blocking read** of the first file into `buffer1`.
2. For each next file: **zero the control block** (`memset`), fill it in, and **`aio_read`** into `buffer2`.
3. **Process `buffer1`** while that read happens.
4. **Wait** for the read (`aio_error` until not `EINPROGRESS`), then **`aio_return`**.
5. **Swap** the buffers, so the just-read file is next to be processed.
6. After the loop, **process the last file**.

Needs **`#include <aio.h>`** and compiling with **`-lrt`**.

> [!tip] `-lrt` (beyond the lecture)
> Since **glibc 2.34**, the AIO functions live in **libc** itself, so `-lrt` isn't strictly needed on newer systems, but it's **harmless** to include.

### Better Than `sleep`
If there's **nothing else to do**, don't sleep: **block**.

```c
int aio_suspend( const struct aiocb *const list[], int nent, const struct timespec* timeout );
```

- **`list`** / **`nent`**: an array of control blocks, and its size.
- **`timeout`**: max wait.
- Blocks until **any one** of them is **complete**, or the timeout runs out.
- Returns **0** if one finished, **-1** on timeout.
- If they were **already done**, it **doesn't block**.

So the `sleep` loop above could be:

```c
const struct aiocb* list[1] = { &cb };
aio_suspend( list, 1, NULL ); /* NULL timeout: wait as long as it takes */
```

## Callback When Done
Using **`SIGEV_THREAD`**:

```c
#include <stdlib.h>
#include <stdio.h>
#include <aio.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <pthread.h>

#define MAX_SIZE 512

void worker( union sigval argument) {
    char* buffer = (char*) argument.sival_ptr;
    printf("Worker thread here. Buffer contains: %s\n", buffer);
    free( buffer );
}

int main( int argc, char** argv ) {
    char* buffer = malloc( MAX_SIZE * sizeof( char ));

    int fd = open( "example.txt", O_RDONLY );
    memset( buffer, 0, MAX_SIZE * sizeof( char ));
    struct aiocb cb;
    memset( &cb, 0, sizeof( struct aiocb ));

    cb.aio_nbytes = MAX_SIZE;
    cb.aio_fildes = fd;
    cb.aio_offset = 0;
    cb.aio_buf = buffer;
    cb.aio_sigevent.sigev_notify = SIGEV_THREAD;
    cb.aio_sigevent.sigev_notify_function = worker;
    cb.aio_sigevent.sigev_value.sival_ptr = buffer;

    aio_read( &cb );
    
    pthread_exit( NULL );
}
```

- **`sigev_notify = SIGEV_THREAD`**: run a function in a new thread when done.
- **`sigev_notify_function = worker`**: that function.
- **`sigev_value.sival_ptr = buffer`**: pass the **buffer** as its argument.
- `worker` **prints** the buffer and **frees** it.

> [!tip] Why `pthread_exit` at the end of `main`?
> A plain **`return`** from `main` **ends the whole process**, possibly **before** the read finishes and `worker` runs. **`pthread_exit( NULL )`** ends only the **main thread**, so the process keeps going until the **callback thread** is done.

## Cancelling

```c
int aio_cancel( int fd, struct aiocb* aiocb );
```

Cancel a specific request on `fd`. Pass **`NULL`** for the control block to cancel **all** outstanding AIO on that file.

It only **tries**. Four possible results:

| Return | Meaning |
| --- | --- |
| **`AIO_CANCELED`** | Cancelled |
| **`AIO_NOTCANCELED`** | **At least one** couldn't be cancelled |
| **`AIO_ALLDONE`** | **Already finished** before it could be cancelled |
| **-1** | Something went wrong; `errno` set |

(The lecture writes `AIO_CANCELLED`, but the real constants are spelled with **one L**: `AIO_CANCELED`, `AIO_NOTCANCELED`.)

Silly but complete example: start a read, then immediately cancel it.

```c
#include <stdlib.h>
#include <stdio.h>
#include <aio.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>

#define MAX_SIZE 512

int main( int argc, char** argv ) {
    char* buffer = malloc( MAX_SIZE * sizeof( char ));

    int fd = open( "example.txt", O_RDONLY );
    memset( buffer, 0, MAX_SIZE * sizeof( char ));

    struct aiocb cb;
    memset( &cb, 0, sizeof( struct aiocb ));

    cb.aio_nbytes = MAX_SIZE;
    cb.aio_fildes = fd;
    cb.aio_offset = 0;
    cb.aio_buf = buffer;
    aio_read( &cb );
    
    /* Do something */
    
    aio_cancel( fd, & cb );

    sleep( 5 );

    close( fd );
    free( buffer );

    return 0;
}
```

> [!warning] Why the `sleep( 5 )`? (beyond the lecture)
> If the cancel returns **`AIO_NOTCANCELED`**, the read is **still running** and still writing into `buffer`. Freeing it right away would be a **use-after-free**. The `sleep` is a crude wait. Better: check the return value, and if it wasn't cancelled, wait for it properly (`aio_suspend` or loop on `aio_error`), then call `aio_return`.

## List I/O
The last field, **`aio_lio_opcode`**: submit a **group** of AIO requests **in one call**, instead of `aio_read`/`aio_write` on each.

```c
int lio_listio( int mode, struct aiocb * const list[ ], int nent, 
                struct sigevent* sigev );
```

- **`mode`**:
	- **`LIO_WAIT`**: **don't return** until **all** operations finish. (`sigev` is **ignored**.)
	- **`LIO_NOWAIT`**: return as soon as they're **queued**.
- **`list`** / **`nent`**: the array of control blocks and its size.
- In each control block, set **`aio_lio_opcode`** to **`LIO_READ`**, **`LIO_WRITE`**, or **`LIO_NOP`** (do nothing).
- **`sigev`**: an event that fires once **all** are done (**`NULL`** for none). This is **extra**: each control block's **own** notification (signal or thread) still fires if set.

## AIO Is HARD!
On **Linux**, POSIX AIO is actually implemented **in user space with threads doing blocking reads**. (Ouch.) People have worked for a long time on a proper **in-kernel** version. The current one **doesn't scale well**.

Does that affect your program? **No**: it still **works**, and it's the **POSIX-compliant, portable** way. But for a **very large number** of async requests, consider something else, like **libevent**.

> [!tip] io_uring (beyond the lecture)
> Modern Linux (5.1+) has **`io_uring`**: true kernel async I/O using **shared ring buffers** between your program and the kernel. It's what high-performance programs use now, usually through **liburing**.
