## Network Communication Continued
We should now have a good idea about how we **establish connections** for communication, but we haven't actually **sent or received any data**. We will assume the same setup from lecture 7 is done and a **connection is established**. If we want to send **datagrams** the workflow is different, but we will come back to that later.

**Either side can send and receive data**, but figuring out <u>*whose turn it is to talk and whose turn it is to listen*</u> is the job of the **client and server themselves**. Likewise, **what the content is and how to interpret it** is up to the client and server, so it has to be agreed on in advance (a ***protocol***).

### Send
The function for sending data is:

```c
int send(int sockfd, const void* msg, int length, int flags);
```

1. **`sockfd`**: the socket file descriptor, either the one returned by **`socket()`** (**client-side**) or by **`accept()`** (**server-side**)
2. **`msg`**: the data to send, just **some arbitrary bytes**
3. **`length`**: how many bytes of `msg` we are sending
4. **`flags`**: for simple use cases we can just use **`0`**

The return value is the **number of bytes sent**. If something went wrong, it's **-1** and **`errno`** tells you more about what exactly went wrong. Under ideal circumstances, the number of bytes sent **equals the `length` parameter**.

```c
char *msg = "Hello world!";
int len = strnlen(msg, 13);
int sent = send(sockfd, msg, len, 0);
```

> [!warning] Correction: the example code (L08: Send)
> - It's **`strnlen(msg, 13)`**, not `strlen(msg, 13)`. `strlen` only takes one argument; `strnlen` takes a **maximum length** so it won't run off the end of a string that's missing its null terminator.
> - The message is `"Hello world!"` (a space, not an underscore).

In real code we <u>*should check for `-1`*</u>. Unlike `malloc()`, where failure is rare, **networks are tricky and can fail**, so checking is worthwhile.

> [!warning] Correction: checking for -1 (L08: Send)
> Your notes said a `-1` is "a rare occurrence that we probably do not have to worry about". The lecture says the opposite: that's true of something like `malloc`, but **network failures are common enough that you should check**.

#### Partial Sends
"Under ideal circumstances" is doing a lot of work: there's a **limit to how much data can go out in one chunk**. It's reasonably sized (**around 1KB**), but you <u>*can't memorize a number and assume it holds on every system*</u>. So if you have a significant amount of data, **check how much was actually sent** and **you are responsible for sending the rest**.

For anything beyond a test/demo, track the bytes sent and **keep calling `send`, advancing the pointer** each time until nothing is left:

```c
int sendall(int socket, char *buf, int *len) {
	int total = 0;        // how many bytes we've sent
	int bytesleft = *len; // how many we have left to send
	int n;

	while (total < *len) {
		n = send(socket, buf + total, bytesleft, 0);
		if (n == -1) {
			break;
		}
		total += n;
		bytesleft -= n;
	}
	*len = total; // return number actually sent here
	return n == -1 ? -1 : 0; // return -1 on failure, 0 on success
}
```

- `buf + total` is where the **unsent part** starts
- On return, **`*len` holds how many bytes actually went out**, so on failure the caller knows how far it got

### Receive
To receive data we use **`recv()`**. The signature is **almost the same** as `send`, but not identical:

```c
int recv(int sockfd, void *buffer, int length, int flags);
```

1. **`sockfd`**: the socket to **receive data from**
2. **`buffer`**: the **destination** where the data goes (not `const`, since it gets written to)
3. **`length`**: the **maximum size** of that buffer
4. **`flags`**: can also be **`0`**

The return value is the **number of bytes actually read** into the buffer:
- **`-1`**: an **error** occurred; check **`errno`**
- **`0`**: the <u>*other side hung up*</u> (they **closed the socket**), so there's no point waiting anymore
- Anything else: that many bytes were received

#### Knowing When the Other Side Is Done
Knowing that the sender is **finished sending** isn't necessarily easy; it's something to **work out with the sender**. It's like saying **"over"** on a walkie-talkie so the other person knows you're done talking. In network communication we might:
- **know in advance** the size of the data we're supposed to get,
- be **told the size** as part of the (negotiated) **protocol**, or
- **wait for the other side to close** the connection (then they're definitely done).

#### Serialization
Suppose we're sending more than a string. Could we **receive straight into a `struct`**, using its address as the buffer and its **`sizeof`** as the length? **Yes**, but only if the representation coming over the network is <u>*exactly the same as your struct*</u> (same layout, padding, sizes, and byte order on both sides).

The more sensible approach is ***serialization***: convert the data into some **byte representation** that can be sent across the network (or stored in a database, or anything really), then **reconstruct it on the other side** via ***deserialization***. That way **no particular in-memory format is needed**, and systems with **different software or even different architectures** can communicate easily.

In practice, **don't write your own** (de)serialization routine: libraries like **`protobuf-c`** exist explicitly for this. Pick a good one and use it.

When we're done communicating, we just call **`close()`** on the socket and that's the end. That's the whole stream workflow: **set up, send/recv, close**.

### Datagrams
If we don't want to establish a communication channel and just want to **send a message** (calling is for old people, we just want to **text**), there are two related functions:

```c
int sendto(int sockfd, const void* msg, int length, unsigned int flags,
	const struct sockaddr* to, socklen_t tolength);

int recvfrom(int sockfd, void* buffer, int length, unsigned int flags,
	struct sockaddr* from, int* fromlength);
```

Since there's **no connection**, <u>*each send says where the data goes*</u> (**`to`**, `tolength`), and <u>*each receive tells you where it came from*</u> (**`from`**, `fromlength`). Otherwise they behave like `send`/`recv`: `sendto` still returns the **number of bytes sent**, which may be **less than expected** for a large amount of data.

If you call **`connect()` on a datagram socket**, you can then use the **regular `send` and `recv`**. The transport is **still UDP** (still unreliable); you just don't have to supply the source/destination every time.

### cURL
In most situations we **don't work with sockets directly** when dealing with URLs. Instead we use ***cURL*** (or similar), a **network communication and transfer library**. There's a command-line tool, but the **client-side library (`libcurl`)** is what we care about, since we can build on it in our programs. It speaks **lots of protocols**, but it's <u>*client-side only*</u>; it isn't meant for server-side operations.

Web services have ***endpoints*** that clients connect to over **HTTP** to get a response, often following design principles like **REST** (REpresentational State Transfer). To do a **GET** request (to "get" some resource) we *could* open a socket, build the string `"GET / HTTP/1.0\r\n"` and `send()` it by hand, but libcurl does this for us easily. (Docs: https://curl.haxx.se/libcurl/c/)

#### Basic Example
```c
#include <stdio.h>
#include <curl/curl.h>

int main(int argc, char** argv) {
	CURL *curl;
	CURLcode res;

	curl_global_init(CURL_GLOBAL_DEFAULT);

	curl = curl_easy_init();
	if (curl) {
		curl_easy_setopt(curl, CURLOPT_URL, "https://example.com/");
		res = curl_easy_perform(curl);

		if (res != CURLE_OK) {
			fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));
		}
		curl_easy_cleanup(curl);
	}

	curl_global_cleanup();
	return 0;
}
```

Going through it from the top:
- **`curl/curl.h`**: if it's not found on your own system, install **`libcurl-dev`**.
- **`CURL *curl`** is our ***handle***: initialized before use, cleaned up when done. **`CURLcode`** is an **enum** that tells us how operations went. On the network, <u>*always check what happened*</u> rather than assuming success.
- **`curl_global_init()` / `curl_global_cleanup()`** bracket **all the code that uses the network**. `CURL_GLOBAL_DEFAULT` is sufficient. If you forget the global init, `curl_easy_init` will do it for you, but that's **dangerous** because it being **called more than once is an issue**.
- **`curl_easy_init()`** initializes the handle; it's **`NULL` on failure** (hence the `if (curl)`). It's "easy" not because there's a "hard" interface, but because there's a **"multi" interface** we'll see later.
- **`curl_easy_setopt()`** sets options on the connection. The **only mandatory option is the URL** (`CURLOPT_URL`). There are **~200 options** in total.
- **`curl_easy_perform()`** is the step that <u>*actually sends data over the network and gets a response*</u>. The result goes in `res`.
- **`CURLE_OK`** means it worked. Otherwise, **`curl_easy_strerror()`** turns the code into a readable message (much nicer than looking up `errno` values).
- **`curl_easy_cleanup()`** cleans up the handle, then **`curl_global_cleanup()`** when everything is finished.

A handle **can be reused** for multiple requests (you may need to update its options). To reuse it with **all settings cleared**, use **`curl_easy_reset()`**.

#### Callbacks
Almost certainly we want to **do something with the data we get**, or we have **data to send**. For each direction we set up a ***callback***: like saying "when you're finished, **call me back**", we tell the handle **which function to call** when the time comes.

- The ***read callback*** is used when **uploading data to the server** (e.g. **POST**).
- The ***write callback*** is used when **receiving data from the server** (e.g. **GET**).

> [!tip] Read vs write naming
> The names are from **cURL's point of view**: cURL *reads* your data to upload it, and *writes* the data it received to you.

You may set up a read callback, a write callback, or both, and **different handles can have different callbacks**.

**Write callback** signature:
```c
size_t write_callback(char *ptr, size_t size, size_t nmemb, void *userdata);
```
- The **name can be anything**. `size_t` represents a size and can be treated like an integer.
- **`ptr`** points to the **data we received**, and **`nmemb`** is the **size of that data**.
- **`size`** is **always 1** (the docs don't explain why).
- **`userdata`** lets us **pass our own data** into the callback, e.g. the **name of a file** to write the received data to.
- **Return** the **number of bytes successfully processed**. If it's <u>*not equal to `nmemb`*</u>, the library treats it as a **write error**.

**Read callback** signature:
```c
size_t read_callback(char *buffer, size_t size, size_t nitems, void *inputdata);
```
- **`buffer`** is where **you put the data to send**; it can't hold more than **`size * nitems`** bytes (multiply them to get the max buffer size).
- **Return** the **number of bytes you put there**. It may take **several calls (chunks)** to send everything.
- **Returning 0** signals **end-of-file**: no more upload.

Registering takes **two options per direction**: one for the **function**, one for the **data pointer** passed to it:
```c
curl_easy_setopt(handle, CURLOPT_READFUNCTION, read_callback);
curl_easy_setopt(handle, CURLOPT_READDATA, pointer);

curl_easy_setopt(handle, CURLOPT_WRITEFUNCTION, write_callback);
curl_easy_setopt(handle, CURLOPT_WRITEDATA, pointer);
```

#### POST Example with a Read Callback
```c
#include <stdio.h>
#include <string.h>
#include <curl/curl.h>

const char data[]="Lorem ipsum dolor sit amet, consectetur adipiscing "
	"elit. Sed vel urna neque. Ut quis leo metus. Quisque eleifend, ex at "
	"laoreet rhoncus, odio ipsum semper metus, at tempus ante urna in mauris. "
	"Suspendisse ornare tempor venenatis. Ut dui neque, pellentesque a varius "
	"eget, mattis vitae ligula. Fusce ut pharetra est. Ut ullamcorper mi ac "
	"sollicitudin semper. Praesent sit amet tellus varius, posuere nulla non, "
	"rhoncus ipsum.";

struct data {
	char *readptr;
	size_t sizeleft;
};

size_t read_callback(void *dest, size_t size, size_t nmemb, void *userp) {
	struct data *d = (struct data *) userp;
	size_t buffer_size = size * nmemb;

	if (d->sizeleft > 0) {
		/* copy as much as possible from the source to the destination */
		size_t copy_this_much = d->sizeleft;
		if (copy_this_much > buffer_size) {
			copy_this_much = buffer_size;
		}
		memcpy(dest, d->readptr, copy_this_much);

		d->readptr += copy_this_much;
		d->sizeleft -= copy_this_much;
		return copy_this_much;
	}
	return 0; /* no more data left to deliver */
}

int main(int argc, char** argv) {
	CURL *curl;
	CURLcode res;
	struct data * d = malloc(sizeof(struct data));

	d->readptr = data;
	d->sizeleft = strlen(data);

	res = curl_global_init(CURL_GLOBAL_DEFAULT);
	if (res != CURLE_OK) {
		fprintf(stderr, "curl_global_init() failed: %s\n", curl_easy_strerror(res));
		return 1;
	}

	curl = curl_easy_init();
	if (curl) {
		curl_easy_setopt(curl, CURLOPT_URL, "https://example.com/index.cgi");

		/* Now specify we want to POST data */
		curl_easy_setopt(curl, CURLOPT_POST, 1L);

		curl_easy_setopt(curl, CURLOPT_READFUNCTION, read_callback);
		curl_easy_setopt(curl, CURLOPT_READDATA, d);

		res = curl_easy_perform(curl);
		/* Check for errors */
		if (res != CURLE_OK) {
			fprintf(stderr, "curl_easy_perform() failed: %s\n", curl_easy_strerror(res));
		}
		curl_easy_cleanup(curl);
	}
	free(d);
	curl_global_cleanup();
	return 0;
}
```

- **`data`** is deliberately **large** sample data, and **`struct data`** tracks **where we're reading from** (`readptr`) and **how much is left** (`sizeleft`).
- **`read_callback`**: compute the **buffer size** (`size * nmemb`), copy **whichever is smaller** (what's left vs. what fits) with **`memcpy`**, **advance** `readptr` and **shrink** `sizeleft`, and **return the amount copied**. Once everything's been copied, **return 0**. This is the same "keep going until nothing is left" idea as `sendall`.
- In `main`, the new parts are **initializing `d`**, setting **`CURLOPT_POST`** (the **default is GET**), and registering the **read callback** plus **`d` as its data**.

> [!warning] Missing from the example (beyond the lecture)
> `malloc`/`free` need **`#include <stdlib.h>`**, and assigning the `const char data[]` to `char *readptr` drops the `const` (the compiler will warn). Neither changes how the example works.

This only **scratches the surface** of what libcurl can do, but it's enough to get the flavour and start doing useful work with it.
