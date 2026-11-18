## Asynchronous (Non-Blocking) I/O
The usual file read:

```c
int fd = open( "example.txt", O_RDONLY );
int bytes_read = read( fd, buffer, num_bytes );
close( fd );
```

**`read` blocks**: the program waits for the I/O to **finish** before going on. Sometimes that's **sensible** (if you need the data **in the next statement**, you can't continue without it). Sometimes not.

Waiting for the bus, you don't **stare blankly into space**. You pull out your phone and **do something** with the time (productive or not).

### Why Not Just Threads?
Our main answer so far: **threads**. If one blocks on I/O, the others keep going. But maybe you **don't want to** or **can't**, because of:
- **Race conditions**.
- **Thread stack** size overhead.
- **Limits on the number of threads**. Sounds ridiculous, but in some **embedded systems** you can't make new threads (or not as many as you want).

And some languages (e.g. **JavaScript**) don't let you make threads at all, so **async I/O is the only option**. A useful tool to have.

### `O_NONBLOCK` on a File
Simplest attempt:

```c
int fd = open( "example.txt", O_RDONLY | O_NONBLOCK );
int bytes_read = read( fd, buffer, num_bytes ); /* Returns instantly! */
close( fd );
```

**Doesn't help here.** `O_NONBLOCK` means "**don't wait if no data is available**", but a file **always** has data available. Loading it from disk might be **slow**, but the data is **there**.

### `O_NONBLOCK` on a Socket
Where is data **not** always available? **Sockets.** If nothing's arrived, we'd block waiting. Make the socket non-blocking:

```c
sockfd = socket( AF_INET, SOCK_STREAM, 0 );
fcntl( sockfd, F_SETFL, O_NONBLOCK );
```

Now **`accept()`**, **`recv()`**, and **`recvfrom()`** **don't block**. If there's nothing, they return **-1** with `errno` set to **`EAGAIN`** or **`EWOULDBLOCK`**. The spec doesn't say **which**, so <u>*check for both*</u>.

> [!tip] Two small details (beyond the lecture)
> - On **Linux**, `EAGAIN` and `EWOULDBLOCK` are the **same value**, but checking both is the **portable** way.
> - `F_SETFL` **replaces** all the status flags. To keep any existing ones:
> ```c
> int flags = fcntl( sockfd, F_GETFL );
> fcntl( sockfd, F_SETFL, flags | O_NONBLOCK );
> ```

### Listening on Several Sockets
A **server listening on several sockets** is common. One thread per socket? See above for why we might not be able to, and now we **don't have to**.

But if **nothing's coming in**, what do we do?
- **Poll each socket** with `accept()` in a loop: ***tight polling***. **CPU intensive**, wastes CPU time.
- **Block on one socket**: what if the next packet arrives on a **different** one?

We need a **third option**.

## `select()`
Monitor a **group** of sockets and find out the **state** of each. A socket can be:
- **Ready to read**.
- **Ready to write** (a **small** amount; a huge write will **block** at some point).
- In an **exceptional** state.

So `select` takes **three sets**:

```c
int select( int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout );
```

Calling it **blocks** until:
- Something happens on one of the sockets so it becomes **ready** (data to read, space to write, etc.), or
- The **timeout** runs out, or
- We get **interrupted** (e.g. by a **signal**).

### `nfds`
**The highest-numbered fd in any of the three sets, plus 1.** So. It has come to this. `select()` is pretty **arcane**.

Why? `select` **scans every fd from 0 to `nfds - 1`** to see if we care about it. An **`fd_set`** holds up to **1024** fds and is a **bitfield**, so telling the kernel the **highest** one lets it **stop early** (and the `fd_set` doesn't need to be a big linked list or array).

### `fd_set`
A **set of file descriptors**. Four functions (macros) to manipulate one:

```c
void FD_ZERO( fd_set *set );           /* Clear the set */
void FD_SET( int fd, fd_set *set );    /* Add fd to the set */
void FD_CLR( int fd, fd_set *set );    /* Remove fd from the set */
int  FD_ISSET( int fd, fd_set *set );  /* Tests if fd is a part of the set */ 
```

- **`FD_ZERO`**: **initialize** a new set first (or **reset** one later).
- **`FD_SET`** / **`FD_CLR`**: **add** / **remove** an fd.
- **`FD_ISSET`**: not because we **forgot** what we added. It's for **after** `select` returns: is this fd **still in the set** (i.e. **ready**)?

### The Three Sets
- **`readfds`**: sockets to **read** from.
- **`writefds`**: sockets to **write** to.
- **`exceptfds`**: not Java exceptions (no `SocketDoesNotFeelLikeDoingWorkRightNowException`). It's for sockets in an ***exceptional state***, which usually means ***Out-Of-Band (OOB) data*** on a TCP socket. We won't go into it.

You **don't need all three**. Only reading? Put everything in `readfds` and pass **`NULL`** (or empty sets) for the others.

### Timeout

```c
struct timeval {
  long tv_sec;  /* seconds */
  long tv_usec; /* microseconds */
};
```

| Timeout | Behaviour |
| --- | --- |
| Both fields **0** | Returns **immediately** |
| **`NULL`** pointer | **No timeout**: wait as long as it takes for **anything** to happen |
| Otherwise | Wait **at most** that long |

### After `select` Returns
**Everything except `nfds` may have been modified** (argh):
- The **fd sets** are changed **in place**: only the **ready** fds remain.
- The **`timeval`** **may or may not** be updated with the **time left**. It varies by system, so it's **not safe to reuse**: overwrite it before the next call.

Then **check every fd** in your sets. **All of them.** Yes, really. `select` tells you **something** is ready, not **which**, so use **`FD_ISSET`** on each.

And since the sets were modified, <u>*rebuild them before each call*</u>. If you were waiting on three sockets, maybe only one became ready, so put all three **back** in.

> [!tip] Return value
> **-1**: error. **0**: timed out, nothing happened. **> 0**: the **number** of ready fds.

### Example: Three Services

```c
void listen_for_connections( int service1_sock, int service2_sock, int service3_sock ) {
  int nfds = 1 + (service1_sock > service2_sock
    ? service1_sock > service3_sock ? service1_sock : service3_sock
    : service2_sock > service3_sock ? service2_sock : service3_sock);
    
  fd_set s;
  struct timeval tv;
  printf( "Going to start listening for socket events.\n" );
  
  while( !quit ) {
  
    FD_ZERO( &s );
    FD_SET( service1_sock, &s );
    FD_SET( service2_sock, &s );
    FD_SET( service3_sock, &s );
    
    tv.tv_sec = 30;
    tv.tv_usec = 0;
  
    int res = select( nfds, &s, NULL, NULL, &tv );
    if ( res == -1 ) { /* An error occurred */
      printf( "An error occurred in select(): %s.\n", strerror( errno ) );
      quit = 1;
    } else if ( res == 0 ) { /* 0 sockets had events occur */
      printf( "Still waiting; nothing occurred recently.\n" );
    } else { /* Things happened */
      if ( FD_ISSET( service1_sock, &s ) ) {
        service1_activate( ); 
      }
      if ( FD_ISSET( service2_sock, &s ) ) {
        service2_activate( ); 
      }
      if ( FD_ISSET( service3_sock, &s ) ) {
        service3_activate( ); 
      }
    }
  }
}
```

- `nfds` is **max of the three + 1** (that nested ternary).
- **Every iteration**: rebuild the set and reset the timeout to **30 s**.
- Check **each** socket individually, since **more than one** can be ready at once.
- If an `activate()` takes a while and data arrives meanwhile, the next `select` **returns immediately** (data is waiting).

This is the **simple** case: wait for `accept()`, send a quick response, done. If connections **stay open** (occasional communication), you also have to **add and remove** sockets from the sets as connections **open and close**. That's **non-trivial**, since every `select` call **changes the sets**.

### Example: Chat Server
Think **IRC**. People join a chat room by connecting to the server. People **don't always talk**, so the server uses `select` to wait for something on either:
- The **listening socket** (new connection), or
- A **client socket** (someone said something).

**Listening socket ready**: `accept` the connection and **add the new socket** to the watch list. Maybe tell the other clients someone **joined** (a **write** to their sockets).

**Client socket ready**: probably someone said something.
- **Read** it and **send it on** to all the other clients.
- **Except** sometimes there's nothing: <u>*a closed socket also shows up as ready to read*</u>, but `read` returns **0** (or negative). The client **disconnected**: **remove** the socket from the watch list, and maybe tell everyone they **left**.

Real chat servers also need things like **keep-alive** messages and **kicking/banning** spammers, but the point is just to show `select` doing something **useful**. (Don't give out personal info to strangers.)

## `pselect()`

```c
int pselect( int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, 
             const struct timespec *timeout, const sigset_t *sigmask );
```

Two differences from `select`:

**1. `struct timespec`** instead of `timeval`, and it's **never modified**:

```c
struct timespec {
  long tv_sec;  /* seconds */
  long tv_nsec; /* nanoseconds */
};
```

Yes, the only real difference: **nanoseconds** instead of **microseconds**.

**2. A signal mask** (from **signals**, L06). It **atomically** swaps in a signal mask for the duration of the wait, so you choose **which signals can wake you up**:

```c
ready = pselect( nfds, &readfds, &writefds, &exceptfds, timeout, &sigmask );
```

is equivalent to doing all of this **atomically**:

```c
sigset_t origmask;
pthread_sigmask( SIG_SETMASK, &sigmask, &origmask );
ready = select( nfds, &readfds, &writefds, &exceptfds, timeout );
pthread_sigmask( SIG_SETMASK, &origmask, NULL );
```

Doing those three steps **yourself** risks a **signal arriving** between changing the mask and calling `select`. **`pthread_sigmask`** is just the POSIX threads version of **`sigprocmask()`**, and works the same way.

### Just a Nap
You can **"misuse"** `select`/`pselect` as a **portable sleep**. Sleep functions don't always **translate** across UNIX-like systems, especially for **very short** sleeps, but `select` is **in the standard**. Call it with **all three sets empty**, **`nfds = 0`**, and the timeout you want. Clever!

```c
struct timeval tv = { 0, 500 }; /* 500 microseconds */
select( 0, NULL, NULL, NULL, &tv );
```

## `poll()`
Very much like `select()`: watch fds and see if any are **ready for I/O**. Much **simpler signature**:

```c
int poll( struct pollfd *fds, nfds_t nfds, int timeout );
```

- **`fds`**: an **array** of `struct pollfd`, one per fd to watch.
- **`nfds`**: just the **number of items in the array**. (Finally.)
- **`timeout`**: **milliseconds**. **0** = don't wait at all, **negative** = wait forever.

```c
struct pollfd {
  int fd;        /* file descriptor */
  short events;  /* requested events */
  short revents; /* returned events */
};
```

Someone found a use for a **`short`**! (Just kidding, hardware people.)
- **`fd`**: the fd to watch.
- **`events`**: what **we** want to wait for.
- **`revents`**: set by the **kernel**: what **happened**.

### Event Flags
Request (combine with **`|`**):

| Flag | Meaning |
| --- | --- |
| **`POLLIN`** | Data ready to **read** |
| **`POLLOUT`** | Can **write** without blocking |
| **`POLLPRI`** | **Exception** (OOB data, as above) |

`revents` can also contain:

| Flag | Meaning |
| --- | --- |
| **`POLLERR`** | An **error** occurred |
| **`POLLHUP`** | The other side **hung up** (closed the connection) |
| **`POLLNVAL`** | Problem with the fd (e.g. **not open** / uninitialized) |

### Example: Three Services with `poll`

```c
void listen_for_connections( int service1_sock, int service2_sock, int service3_sock ) {
  struct pollfd pollfds[3];
  pollfds[0].fd = service1_sock;
  pollfds[0].events = POLLIN;
  pollfds[1].fd = service2_sock;
  pollfds[1].events = POLLIN;
  pollfds[2].fd = service3_sock;
  pollfds[2].events = POLLIN;
 
  int timeout = 30 * 1000; /* 30 seconds in ms */ 
  printf( "Going to start listening for socket events.\n" );
  
  while( !quit ) {
    
    int res = poll( pollfds, 3, timeout );
    if ( res == -1 ) { /* An error occurred */
      printf( "An error occurred in poll(): %s.\n", strerror( errno ) );
      quit = 1;
    } else if ( res == 0 ) { /* 0 sockets had events occur */
      printf( "Still waiting; nothing occurred recently.\n" );
    } else { /* Things happened */
      if ( pollfds[0].revents & POLLIN ) {
        service1_activate( ); 
      }
      if ( pollfds[1].revents & POLLIN ) {
        service2_activate( ); 
      }
      if ( pollfds[2].revents & POLLIN ) {
        service3_activate( ); 
      }
  }
}
```

Set up the array **once**, outside the loop. Check **`revents & POLLIN`** to see who's ready.

> [!warning] Missing brace
> The lecture code is **one `}` short**: the `else { /* Things happened */` block never closes, so the `while` and the function don't close properly. Add a `}` at the end.

## `select` vs. `poll`
**Same job**, different API. `poll` is nicer:
- **No max fd** to compute.
- **No three separate sets** to manage.
- **No rebuilding** every iteration (the kernel writes to **`revents`**, leaving **`events`** alone).

But <u>*both are slow*</u>: **linear** in the number of fds. Around **a hundred fds**, you spend **significant CPU time** just figuring out **which** socket changed.

> [!tip] More differences (beyond the lecture)
> - `select` can't handle fds **≥ 1024** (`FD_SETSIZE`); using one is **undefined behaviour**. `poll` has **no such limit**.
> - In `poll`, set an entry's `fd` to a **negative** number to have it **ignored** without reshuffling the array.
> - The **O(1)** answer on Linux is **`epoll`** (and **`kqueue`** on BSD/macOS): you register fds **once**, and the kernel hands back **only the ready ones**. That's what **libevent** (L29) uses under the hood.

Next: for something more efficient we'll want **libevent**. But first, **cURL** (L08) can do non-blocking I/O too.
