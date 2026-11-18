## Event-Driven I/O
We've seen **select/poll**, **cURL multi**, and **POSIX AIO**. They all have **drawbacks**. **libevent** is built for **high-performance** applications and **scalable network servers**.

New way of thinking: instead of asking whether an operation **blocks**, think of I/O as ***events***: <u>*when something happens, take some action*</u>.

## Getting Ready
Lots of configuration options, mostly **too advanced** for this course (fatal error handling, custom memory allocation...). The important one: tell libevent to use **pthreads**.

```c
int evthread_use_pthreads( );
```

- Returns **0** on success, **-1** on failure.
- There's a **Windows** equivalent (useless on Linux).
- It's a **composite** of several smaller settings (locks, condition variables, threads), so just call this **one** and you're set.

## Event Base
Every event belongs to an ***`event_base`***: a **container** for a set of events, plus some **configuration**.
- If **locking** is enabled, it's **safe to use from multiple threads**...
- ...but its **loop** can only run in **one thread**. So multiple threads might need **multiple bases**.

### Backends
The base uses some **method** to check which events are ready. Familiar ones: **select** and **poll**. Others: **epoll**, **kqueue**, **devpoll**, **evport**, **win32**.

### Create and Free

```c
struct event_base* event_base_new( );                                       /* Default settings */
struct event_base* event_base_new_with_config( const struct event_config* cfg ); /* With configuration */
struct event_config* event_config_new( );
void event_config_free( struct event_config* cfg );

void event_base_free( struct event_base* base );
```

Like pthreads, the **config is optional** (with separate allocate/free functions). It can **avoid** certain backends and set **arcane options**. <u>*Defaults are fine for this course.*</u>

### Example: Which Methods Are Available?

```c
#include <stdlib.h>
#include <stdio.h>
#include <event2/event.h>

int main( int argc, char** argv ) { 
    int i;
    const char **methods = event_get_supported_methods();
    printf("Starting Libevent %s.  Available methods are:\n",
            event_get_version());
    for (i=0; methods[i] != NULL; ++i) {
        printf("    %s\n", methods[i]);
    }   
    return 0;
}
```

On **`ecetesla0`**:

```
jzarnett@ecetesla0:~/ece252$ gcc -std=c99 -g -levent -o le1 le1.c
jzarnett@ecetesla0:~/ece252$ ./le1 
Starting Libevent 2.1.8-stable.  Available methods are:
    epoll
    poll
    select
```

### Example: Which Method Is Used?

```c
#include <stdlib.h>
#include <stdio.h>
#include <event2/event.h>

int main( int argc, char** argv ) { 
    struct event_base *base;
    enum event_method_feature f;

    base = event_base_new();
    if (!base) {
        puts("Couldn't get an event_base!");
    } else {
        printf("Using Libevent with backend method %s.",
                event_base_get_method(base));
        f = event_base_get_features(base);
        if ((f & EV_FEATURE_ET))
            printf("  Edge-triggered events are supported.");
        if ((f & EV_FEATURE_O1))
            printf("  O(1) event notification is supported.");
        if ((f & EV_FEATURE_FDS))
            printf("  All FD types are supported.");
        puts("");
    }   
    return 0;
}
```

```
jzarnett@ecetesla0:~/ece252$ ./le2
Using Libevent with backend method epoll.  
 Edge-triggered events are supported.  
 O(1) event notification is supported.
```

We never learned **epoll**. Do we need to? **No**: the whole point of libevent is that <u>*the backend isn't our concern*</u>.

> [!tip] What epoll is (beyond the lecture)
> **`epoll`** is Linux's answer to select/poll being **linear**. You register fds **once**, and each wait returns **only the ready ones**, so cost doesn't grow with the **number of watched fds**. That's the **"O(1) event notification"** above. BSD/macOS have **kqueue** for the same job.

## Events
An event happens on a **file descriptor**, like every other I/O method we've seen.

### Life Cycle
1. **Created**.
2. **Initialized**: associated with an **event base**.
3. **Pending**: after you **add** it. We're **waiting** for it to happen.
4. **Active**: it happened while pending, so the user-defined **callback runs**.
5. **Delete** it to make it **non-pending** (stop waiting). A non-pending event can be **re-added**.

### Create and Free

```c
typedef void (*event_callback_fn)( evutil_socket_t fd, short what, void* arg );
struct event* event_new( struct event_base* base, evutil_socket_t fd, short what, 
                         event_callback_fn cb, void* arg );
void event_free( struct event* event );
```

- **`event_callback_fn`**: the callback must return **`void`** and take **three** parameters (the fd, what happened, and your argument).
- **`event_new`**:
	- **`base`**: which base it belongs to.
	- **`fd`**: `evutil_socket_t` is really just an **`int`** (on non-Windows), so use a **regular fd**.
	- **`what`**: what to be **notified about** (below).
	- **`cb`** / **`arg`**: the **callback** and the **argument** passed to it.
- We never touch `struct event`'s **internals**. Fine with us.

### `what`: The Flags
We found a use for **`short`**! Only took half your degree...

```c
#define EV_TIMEOUT      0x01
#define EV_READ         0x02
#define EV_WRITE        0x04
#define EV_SIGNAL       0x08
#define EV_PERSIST      0x10
#define EV_ET           0x20
```

- **`EV_TIMEOUT`**: **ignored** when you set what you want, but can be **returned** to the callback (meaning it **timed out**).
- **`EV_READ`** / **`EV_WRITE`**: fd ready to read / write.
- **`EV_SIGNAL`**: a **signal** arrived.
- **`EV_PERSIST`**: after triggering, the event is **automatically pending again** (otherwise it fires **once** and you'd have to re-add it).
- **`EV_ET`**: ***edge-triggered***: report a **change** in readiness, rather than readiness itself.

Combine with **`|`**: e.g. **`EV_READ | EV_WRITE`**.

> [!tip] Level vs. edge triggered
> - ***Level-triggered*** (default): "the fd **is** readable" fires **every time you check**, as long as data's sitting there.
> - ***Edge-triggered*** (`EV_ET`): fires only when it **becomes** readable (new data arrives). If you don't read **everything**, you won't be told again until **more** data comes in.

Want the **event itself** as the callback's `void*` argument? Normally impossible (it doesn't **exist yet** when you call `event_new`), but **`event_self_cbarg()`** does a little magic for that.

**`event_free`** is safe even on a **pending or active** event: it checks and **deactivates** it first.

### Add and Remove

```c
int event_add( struct event* ev, const struct timeval* tv );
int event_del( struct event* ev );
```

- **`event_add`**: make the event **pending**. **`tv`** is an optional **timeout** (`NULL` for none): if nothing happens in that time, the callback runs with **`EV_TIMEOUT`**.
- **`event_del`**: make it **non-pending**.

## Dispatching
Nothing happens until we **run the loop**. The easy way and the "hard" way:

```c
int event_base_dispatch( struct event_base* base );
int event_base_loop( struct event_base *base, int flags );
```

**`event_base_dispatch`** = **`event_base_loop`** with **no flags**, so the hard way isn't that hard.

```c
#define EVLOOP_ONCE             0x01
#define EVLOOP_NONBLOCK         0x02
#define EVLOOP_NO_EXIT_ON_EMPTY 0x04
```

| Flag | Meaning |
| --- | --- |
| **`EVLOOP_ONCE`** | Wait for some events to become active, run active events **until there are none**, then **return** |
| **`EVLOOP_NONBLOCK`** | **Don't wait**: run events that are ready **now**, skip the rest |
| **`EVLOOP_NO_EXIT_ON_EMPTY`** | **Don't exit** when no events are pending or active (normally it does) |

### Stopping the Loop
With `NO_EXIT_ON_EMPTY` (or persistent events), exit **manually**:

```c
int event_base_loopexit( struct event_base* base, const struct timeval* tv );
int event_base_loopbreak( struct event_base* base );
```

- **`loopexit`**: stop **after some time**. Any **active** events still get processed first.
- **`loopbreak`**: stop **right after** the **current** callback, skipping other active ones.

Usually called **from a callback**, when you know you're done.

### Example

```c
void cb_func( evutil_socket_t fd, short what, void *arg ) {
        const char *data = arg;
        printf("Got an event on socket %d:%s%s%s%s [%s]",
            (int) fd,
            (what&EV_TIMEOUT) ? " timeout" : "",
            (what&EV_READ)    ? " read" : "",
            (what&EV_WRITE)   ? " write" : "",
            (what&EV_SIGNAL)  ? " signal" : "",
            data);
}

void main_loop( evutil_socket_t fd1, evutil_socket_t fd2 ){
        struct event *ev1, *ev2;
        struct timeval five_seconds = {5,0};
        struct event_base *base = event_base_new();

        /* The caller has already set up fd1, fd2 somehow, and made them nonblocking. */

        ev1 = event_new(base, fd1, EV_TIMEOUT|EV_READ|EV_PERSIST, cb_func,(char*)"Reading event");
        ev2 = event_new(base, fd2, EV_WRITE|EV_PERSIST, cb_func, (char*)"Writing event");

        event_add(ev1, &five_seconds);
        event_add(ev2, NULL);
        event_base_dispatch(base);
}
```

- **`cb_func`** prints which flags are set in **`what`**, plus the string passed as **`arg`**.
- **`ev1`**: **persistent** read event on `fd1`, with a **5-second timeout**: the callback runs when `fd1` is **readable**, or after **5 s with nothing** (`what` has `EV_TIMEOUT`).
- **`ev2`**: **persistent** write event on `fd2`, **no timeout**.
- **`event_base_dispatch`** runs the loop. Both events are persistent, so it **never runs out** of pending events.

**Only one thread** can dispatch a given base at a time. That's why you can have **multiple bases**.

## Cleaning Up
libevent has some **global** structures, initialized once. When **completely** done:

```c
void libevent_global_shutdown( );
```

- This **doesn't** free things libevent **returned to you** (bases, events...). Free those **normally**.
- Call it **last**, after **all** libevent use, or behaviour is **unpredictable**.

> [!tip] Compiling (beyond the lecture)
> Link with **`-levent`**. `evthread_use_pthreads` lives in a separate library, so also add **`-levent_pthreads`** if you call it.

## Buffered Events
Sometimes we want a **decent chunk of data** before processing. Receiving one **character** at a time and firing an event **per character** is very **wasteful**. Better: fire when a **condition** is met, like **enough data available**.

***Bufferevents*** do this:
- A **normal** event fires when the **transport** (e.g. socket) is ready to read/write.
- A **buffer event** fires when **enough data** has been **read or written**.

They really only work for **TCP**, so it's back to **sockets**. (Files are always "ready", even if a read is slow.)

Each bufferevent has:
- **Two buffers**: **input** and **output**.
- **Two data callbacks**: **read** and **write**.
	- By default, the **read** callback runs whenever **any** data arrives.
	- By default, the **write** callback runs when **enough** of the output buffer has been **sent**.
	- Both can be **overridden**.

So received data **accumulates** in a buffer until it's all there, and data is **sent** once enough has built up.

### Watermarks
Four levels that trigger things:

| Watermark | Default | Effect |
| --- | --- | --- |
| **Read low** | 0 | When a read leaves the input buffer **at least this full**, the **read callback** runs. Default 0: **every** chunk triggers it. |
| **Read high** | Unlimited | Once the input buffer reaches this, **stop reading** from the socket until data is **taken out** and it drops below. Default: fill as much as possible. |
| **Write low** | 0 | When writing gets the output buffer **down to** this level, the **write callback** runs. Default 0: only when the output buffer is **empty**. |
| **Write high** | — | Strangely, **doesn't do what you'd expect**. Used for some special situations. |

### API

```c
struct bufferevent* bufferevent_socket_new( struct event_base* base, evutil_socket_t fd, 
                                            enum bufferevent_options options );
void bufferevent_free( struct bufferevent* bev );

typedef void (*bufferevent_data_cb)( struct bufferevent* bev, void* ctx );
typedef void (*bufferevent_event_cb)( struct bufferevent* bev, short events, void* ctx );
void bufferevent_setcb( struct bufferevent* bufev, bufferevent_data_cb readcb, 
                        bufferevent_data_cb writecb, bufferevent_event_cb eventcb, void* cbarg );

int bufferevent_socket_connect( struct bufferevent* bev, struct sockaddr* address, int addrlen );
```

- **`bufferevent_socket_new`**: which **base**, the **fd**, and **options**. The two we care about:
	- **`BEV_OPT_CLOSE_ON_FREE`**: **close the socket** when the bufferevent is freed.
	- **`BEV_OPT_THREADSAFE`**: use **locks**, so multiple threads can access it.
- **`bufferevent_free`**: self-explanatory.
- **Callback types**: both return `void` and get the **bufferevent**.
	- ***Data callback*** (read/write): plus your **argument**.
	- ***Event callback***: plus **`events`** (what happened, e.g. **connected** or **error**).
- **`bufferevent_setcb`**: set the **read**, **write**, and **event** callbacks (**`NULL`** for none, or to clear one), plus **one argument shared** by all three.
- **`bufferevent_socket_connect`**: **connect** to an address (and its size).

### Example: Connect

```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <sys/socket.h>
#include <string.h>

void eventcb(struct bufferevent *bev, short events, void *ptr) {
    if (events & BEV_EVENT_CONNECTED) {
         /* We're connected to 127.0.0.1:8080.   Ordinarily we'd do
            something here, like start reading or writing. */
    } else if (events & BEV_EVENT_ERROR) {
         /* An error occurred while connecting. */
    }
}

int main_loop( ) {
    struct event_base *base;
    struct bufferevent *bev;
    struct sockaddr_in sin;

    base = event_base_new();

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */
    sin.sin_port = htons(8080); /* Port 8080 */

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);

    bufferevent_setcb(bev, NULL, NULL, eventcb, NULL);

    if (bufferevent_socket_connect(bev,
        (struct sockaddr *)&sin, sizeof(sin)) < 0) {
        /* Error starting connection */
        bufferevent_free(bev);
        return -1;
    }

    event_base_dispatch(base);
    return 0;
}
```

1. Set up the **address** (127.0.0.1, port 8080), as in socket programming.
2. Create the bufferevent with fd **`-1`**: a **placeholder**, since the socket **doesn't exist yet**.
3. Set only the **event callback** (no read/write callbacks).
4. **Connect**. If it fails to even **start**, free the bufferevent.
5. **Dispatch**. When the connection is made, **`eventcb`** runs with **`BEV_EVENT_CONNECTED`**.

> [!tip] Actually reading and writing (beyond the lecture)
> ```c
> bufferevent_enable( bev, EV_READ | EV_WRITE );          /* start watching */
> bufferevent_write( bev, data, len );                    /* queue data to send */
> size_t n = bufferevent_read( bev, buf, sizeof( buf ) ); /* take data out (in readcb) */
> bufferevent_setwatermark( bev, EV_READ, low, high );    /* change watermarks */
> ```

We haven't covered **every** option or tool; libevent is **powerful** and **effective**. But now we've seen **a lot of ways to do async I/O**.

## Recap: Async I/O Options (beyond the lecture)

| Approach | Works on | Good | Bad |
| --- | --- | --- | --- |
| **`select`** (L26) | Sockets (any fd) | Portable, in the standard | Arcane; max fd + 1; rebuild sets each time; ≤ 1024 fds; linear |
| **`poll`** (L26) | Sockets (any fd) | Simpler API, no rebuild | Still linear |
| **cURL multi** (L27) | HTTP etc. via cURL | Thousands of transfers, easy | cURL only |
| **POSIX AIO** (L28) | **Files** | Portable; real async file reads | Linux implements it with threads; doesn't scale |
| **libevent** (L29) | Sockets, signals, timers | Picks the fastest backend (epoll/kqueue); callbacks; buffering | Another library to learn |
