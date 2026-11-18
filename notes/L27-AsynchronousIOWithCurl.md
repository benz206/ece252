## Using cURL Asynchronously
Network communication is a great fit for async I/O: start a request and **move on** to the next one **without waiting** for the first, especially for requests to **different servers**. Socket programming is important, but for a lot of things we'd rather use **cURL**, and it can be **async** too.

### Recap: The Easy Interface
From L08 (modified from the official docs):

```c
#include <stdio.h>
#include <curl/curl.h>
 
int main( int argc, char** argv ) {
  CURL *curl;
  CURLcode res;
 
  curl_global_init(CURL_GLOBAL_DEFAULT);
 
  curl = curl_easy_init();
  if( curl ) {
    curl_easy_setopt(curl, CURLOPT_URL, "https://example.com/" );
    res = curl_easy_perform( curl );
    
    if( res != CURLE_OK ) {
      fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));
    }
    curl_easy_cleanup(curl);
  }
 
  curl_global_cleanup();
  return 0;
}
```

**`curl_easy_perform()` blocks** until the transfer is done. We want to change that.

## The Multi Interface
The ***multi handle*** holds **more than one easy handle**. Instead of waiting, we **start them all** and **check on their progress**.

### Setup
1. **`curl_global_init`**, same as before.
2. Create the multi handle: type **`CURLM`** (instead of `CURL`), made with **`curl_multi_init()`**.
3. Create **easy handles** exactly as before (**`curl_easy_init()`** + however many **`curl_easy_setopt`** calls).
4. Add each one: **`curl_multi_add_handle( CURLM* cm, CURL* eh )`**. As many as you need.

### Perform

```c
curl_multi_perform( CURLM* cm, int* still_running );
```

**Starts** all the transfers, and **doesn't block**. `still_running` is set to how many easy handles are **still in progress**:
- **0**: all **done**.
- **Nonzero**: some are still going.

You'll call it **more than once**. Calling again **doesn't restart** anything; it just **makes progress and updates the status**. Check as often as you like, but the point is to **do something useful** in the meantime. (Otherwise, why not make it synchronous?)

### Wait
Out of useful things to do? **Block** until something happens:

```c
curl_multi_wait( CURLM *multi_handle, struct curl_waitfd extra_fds[],
                 unsigned int extra_nfds, int timeout_ms, int *numfds );
```

- **`multi_handle`**: the multi handle.
- **`extra_fds`** / **`extra_nfds`**: extra fds to wait on, and how many. In this course: always **`NULL`** and **`0`**.
- **`timeout_ms`**: **max time** to wait.
- **`numfds`**: set to the number of "**interesting**" events that happened (that's the spec's word, and what it means is **mysterious**).

For simple use, ignore most of it: **wait for something, then go from there**.

Meanwhile, transfers (and any **callbacks** you set up) are running. Each easy handle's **state** is updated as it moves through its life cycle.

### Read the Results
Each easy handle has a **status message** and a **return code**. Why both? The message might say "**done**", but done **successfully** or done **with an error**? The return code tells you.

```c
CURLMsg* curl_multi_info_read( CURLM* cm, int* msgs_left );
```

Returns a pointer to info about the **next** easy handle (if there is one). **`msgs_left`** is set to how many **messages remain**, so you don't need to track it yourself.

For a returned message `m`:
1. Is **`m->msg == CURLMSG_DONE`**? If not, it's **still in progress**; move on to the next.
2. If done, check **`m->data.result`**: **`CURLE_OK`** means **success**, anything else is an **error**.
3. The handle itself is **`m->easy_handle`**.

### Cleanup
1. **Remove** each finished handle: **`curl_multi_remove_handle( CURLM* cm, CURL* eh )`**.
2. **Clean it up** as usual: **`curl_easy_cleanup( CURL* eh )`**.
3. When **all** easy handles are done: **`curl_multi_cleanup( CURLM* cm )`**.
4. Finally, **`curl_global_cleanup()`**.

## Full Example
By **Clemens Gruber** (slightly modified for compactness, formatting, and cleanup). The callback does **nothing**, just to show where you'd handle data.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <curl/multi.h>

#define MAX_WAIT_MSECS 30*1000 /* Wait max. 30 seconds */

const char *urls[] = {
  "http://www.microsoft.com",
  "http://www.yahoo.com",
  "http://www.wikipedia.org",
  "http://slashdot.org"
};
#define CNT 4

size_t cb(char *d, size_t n, size_t l, void *p) {
  /* take care of the data here, ignored in this example */
  return n*l;
}

void init( CURLM *cm, int i ) {
  CURL *eh = curl_easy_init();
  curl_easy_setopt( eh, CURLOPT_WRITEFUNCTION, cb );
  curl_easy_setopt( eh, CURLOPT_HEADER, 0L );
  curl_easy_setopt( eh, CURLOPT_URL, urls[i] );
  curl_easy_setopt( eh, CURLOPT_PRIVATE, urls[i]) ;
  curl_easy_setopt( eh, CURLOPT_VERBOSE, 0L );
  curl_multi_add_handle( cm, eh );
}

int main( int argc, char** argv ) {
    CURLM *cm = NULL;
    CURL *eh = NULL;
    CURLMsg *msg = NULL;
    CURLcode return_code = 0;
    int still_running = 0; 
    int msgs_left = 0;
    long http_status_code;
    const char *szUrl;

    curl_global_init( CURL_GLOBAL_ALL );
    cm = curl_multi_init( );

    for ( int i = 0; i < CNT; ++i ) {
        init( cm, i );
    }

    curl_multi_perform( cm, &still_running );

    do {
        int numfds = 0;
        int res = curl_multi_wait( cm, NULL, 0, MAX_WAIT_MSECS, &numfds );
        if( res != CURLM_OK ) {
            fprintf( stderr, "error: curl_multi_wait() returned %d\n", res );
            return EXIT_FAILURE;
        }
        curl_multi_perform( cm, &still_running );

    } while( still_running );

    while ( ( msg = curl_multi_info_read( cm, &msgs_left ) ) ) {
        if ( msg->msg == CURLMSG_DONE ) {
            eh = msg->easy_handle;

            return_code = msg->data.result;
            if ( return_code != CURLE_OK ) {
                fprintf( stderr, "CURL error code: %d\n", msg->data.result );
                curl_multi_remove_handle( cm, eh );
                curl_easy_cleanup( eh );
                continue;
            }

            // Get HTTP status code
            http_status_code = 0;
            szUrl = NULL;

            curl_easy_getinfo( eh, CURLINFO_RESPONSE_CODE, &http_status_code );
            curl_easy_getinfo( eh, CURLINFO_PRIVATE, &szUrl );

            if( http_status_code == 200 ) {
                printf( "200 OK for %s\n", szUrl ) ;
            } else {
                fprintf( stderr, "GET of %s returned http status code %d\n", szUrl, http_status_code );
            }

            curl_multi_remove_handle( cm, eh );
            curl_easy_cleanup( eh );
        } else {
            fprintf( stderr, "error: after curl_multi_info_read(), CURLMsg=%d\n", msg->msg );
        }
    }
    curl_multi_cleanup( cm );
    curl_global_cleanup();
    return 0;
}
```

Walking through it:
- **`init`** makes an easy handle per URL, sets the (do-nothing) **write callback**, and stashes the **URL** in **`CURLOPT_PRIVATE`** so we can get it back later with **`CURLINFO_PRIVATE`**. Then adds it to the multi handle.
- **`curl_multi_perform`** kicks off all 4 transfers.
- The **`do`/`while`** loop: **wait** (up to 30 s) for something to happen, then **perform** again to make progress. Repeat until **`still_running` is 0**.
- Then **`curl_multi_info_read`** in a loop to go through **each result**: check it's **`CURLMSG_DONE`**, check **`CURLE_OK`**, read the **HTTP status code**, then **remove and clean up** the handle.
- Finally, clean up the multi handle and global state.

> [!warning] Small fix (beyond the lecture)
> `http_status_code` is a **`long`**, so the `fprintf` should use **`%ld`**, not `%d`. Compile with **`-lcurl`**.

> [!tip] Handle results as they finish (beyond the lecture)
> This example waits for **all** transfers before reading **any** results. You can also call `curl_multi_info_read` **inside** the `do`/`while` loop, right after each `curl_multi_perform`, to deal with each transfer **as soon as it finishes** (or to **add a new handle** in its place).

### Reusing Easy Handles
You **can** reuse an easy handle instead of destroying it and making a new one: **remove** it from the multi handle, change whatever you want, then **re-add** it with `curl_multi_add_handle`. (The example doesn't.)

Since handles can be **replaced** as they finish, there may **always** be handles in progress and you might **never** reach "no messages left". **That's okay.**

Each easy handle can have its **own callback** if you want them to do different things.

### Scaling
The developer claims **multiple thousands** of connections in one multi handle work. 60k ought to be enough for anyone!

> [!tip] `curl_multi_poll` (beyond the lecture)
> Newer libcurl (7.66.0+) has **`curl_multi_poll`**, same arguments as `curl_multi_wait`, but another thread can **wake it early** with **`curl_multi_wakeup`**.

## cURL with `select()` (I Enjoy Pain!)
You **can**, but the prof's **anti-recommendation**: **don't**. cURL at least does some of the **grunt work**.

### 1. Get the `fd_set`s
After setting up the multi handle and adding the easy handles:

```c
CURLMcode curl_multi_fdset( CURLM *multi_handle, fd_set *read_fd_set, fd_set *write_fd_set, 
                            fd_set *exc_fd_set, int *max_fd );
```

- **You** must **`FD_ZERO`** the sets before calling this. (You can pass **`NULL`** for sets you don't want.)
- It kindly tells you the **max fd**. You still add the **`+1`** for `select`.

```c
fd_set fdr;
fd_set fdw;
fd_set fde;
FD_ZERO( &fdr );
FD_ZERO( &fdw );
FD_ZERO( &fde );
int maxfd = -1;

CURLMcode res = curl_multi_fdset( cm, &fdr, &fdw, &fde, &maxfd );
```

### 2. Get the Timeout

```c
CURLMcode curl_multi_timeout( CURLM *multi_handle, long *timeout );
```

It gives a **`long`** in **milliseconds**, not a `struct timeval`, so you do the **conversion yourself** (worth wrapping in a function). From the docs:

```c
struct timeval timeout;
long timeo;
 
curl_multi_timeout( cm, &timeo );
if(timeo < 0) {
  /* no set timeout, use a default */
  timeo = 980;
}
 
timeout.tv_sec = timeo / 1000;
timeout.tv_usec = (timeo % 1000) * 1000;
```

### 3. Start the Transfers
**`curl_multi_perform`**, as before. The cURL examples do this **before** the setup above (you can set up **while transfers are running**), but either way it must be <u>*before `select`*</u>. Otherwise `select` waits for something that **won't happen**: no transfers, no data.

### 4. Loop
When `select` wakes up, something happened:
- Call **`curl_multi_perform`** to make progress and get **how many are still running**.
- Use **`curl_multi_info_read`** for each handle's **status**.
- **Repopulate** the sets and timeout before the next `select` (same as plain `select`).

And **clean up** everything: cURL **and** `select` related.

### Why Not
- `select` is limited in **how many fds** it can handle (you can **overflow the bitmasks**).
- Even if you don't, it **doesn't scale**.
- The regular **`curl_multi_wait`** is **better in pretty much every way**.

## cURL with `poll()`?
**Generally, no.** The lead cURL developer says the way forward is **`curl_multi_wait`**, and there isn't really a `poll` interface. Long story short: **don't**.
