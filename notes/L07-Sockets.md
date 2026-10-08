## Network Communication
Another form of inter-process communication is ***network communication***. If 2 processes are **not running on the same machine** in the same environment, we can't use pipes or shared memory, so we get them to communicate **over the network**. This can <u>*even be done if both processes are on the same device*</u>.

The actual network will be a blackbox for the purposes of this course, we will not discuss how to actually implement it.
### Sockets
The ***socket*** API is a standard dating back many years. It describes how to **communicate over the network** in a standard way. The socket is the concept for how to **establish a communication channel between 2 processes**. There are **2 ways** of doing this: ***datagrams*** and ***connection streams***.

A **datagram** is like **mail**: you can mail letters, but they might be **delivered in any order**, could **get lost**, and are **unidirectional**. There is **no connection** established; it's just message delivery (the recipient can write back, but that's a separate letter).

**Streams** are **bidirectional** and are like a **telephone call**: the other side has to be **available and answer**, then a line of communication is established, and eventually one side hangs up.

Much like everything else in UNIX, <u>*a socket is handled like a file*</u>; it just so happens that the data read from or written to the "file" is being **routed over the network** somehow. To create a socket we include **`sys/socket.h`** and use:

```c
int socket(int domain, int type, int protocol);
```

- **`domain`** defines the **address format**, e.g. IPv4 vs IPv6. We will just use IPv4 for this course, which is **`AF_INET`** (address family: internet).
- **`type`** gives the kind of information we will be sending: **`SOCK_DGRAM`** is for **datagrams** and **`SOCK_STREAM`** is for a **bidirectional byte stream**.
- **`protocol`** is how the data is transported over the connection. We will just use **0 for the default**:
	- For a **stream**, the default is **TCP/IP**, which is (generally) **reliable**: data gets where it needs to go, with all the pieces **in the correct order**.
	- For a **datagram**, the default is **UDP**, which is **not reliable**: packets might get there, or they might not.

> [!warning] Correction: which protocol is unreliable (L07: Sockets)
> Your notes ran TCP/IP and UDP together, so it read like TCP/IP "may work, it may not". It's **UDP** (datagrams) that's unreliable; **TCP/IP** (streams) is the reliable, in-order one.

> [!warning] Missing: why `socket` returns an `int` (L07: Sockets)
> The return value is a **file descriptor**, which is just an integer (same as when opening a file).

We do need to **close the socket** when we are finished with it, using the regular **`close`**.

> [!warning] Missing: why there's only one `close` (L07: Sockets)
> There are different calls to *open* things (`open`, `pipe`, `socket`) because you have to say **what kind of thing** you're opening. To close it, the type is **already known**, so a single `close` works for all of them.

### Byte Order (Endianness)
When communicating over the network, both sides have to **speak the same dialect**: the other system might have a different idea of how data is organized. For a 4-byte integer there are 2 reasonable orders for storing its bytes: **little-endian** (smallest byte first) and **big-endian** (largest byte first).

**Network protocols specify big-endian** (***network byte order***). The machine's own order is the ***host byte order***, which is often **little-endian** (e.g. **x86**), but not always (e.g. PowerPC), so we <u>*can make no assumptions about the other side*</u>.

> [!warning] Correction: "post order" and which order is normal (L07: Check the Boot of the Car for your Jumper!)
> - It's **host order**, not "post order".
> - Your notes said "we use big endian normally, and little endian is the one machines occasionally use". More accurately: **big-endian is what the network uses**, and the host can be either. The most common architecture (**x86**) is actually **little-endian**, so translating is very common.

**`arpa/inet.h`** provides the conversion functions, for **4-byte (32-bit)** and **2-byte (16-bit)** values, to and from **host** and **network** order:

```c
uint32_t htonl(uint32_t hostint32); // 4 byte int to network format
uint16_t htons(uint16_t hostint16); // 2 byte int to network format
uint32_t ntohl(uint32_t netint32);  // 4 byte int to host format
uint16_t ntohs(uint16_t netint16);  // 2 byte int to host format
```

Use them <u>*even if you're sure your system is big-endian*</u>, for **portability**. The exact-size types (`uint32_t`, `uint16_t`) matter here since you can't swap bytes without knowing how many there are.

### Addresses
The structure for a socket address is **`struct sockaddr_in`**:
```c
struct sockaddr_in {
	sa_family_t sin_family; // The address family
	in_port_t sin_port; // The port number
	struct in_addr sin_addr; // the ipv4 address
};

struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(2520);
addr.sin_addr.s_addr = htonl(INADDR_ANY);
```

> [!warning] Missing: what the fields mean (L07: Addresses)
> - **IPv4 addresses** look like `XXX.XXX.XXX.XXX`, each group **0 to 255** (e.g. `192.168.0.1` for your router).
> - **`INADDR_ANY`** means "use **an IP address of the current computer**" (it may have several if it has more than one network connection). We don't want a specific one, just one the computer has.
> - Note the port and address both go through **`htons`/`htonl`** to get into network order. (Also fixed a `:` typo on the `sin_family` line above.)

> [!warning] Missing: ports (L07: Addresses)
> If the computer's IP is a **street address** for an apartment building, the **port** is **which apartment**. Different services (processes) communicate over different ports, and <u>*no two processes can use the same port at the same time*</u>.
> - Ports **below 1024** are **reserved for system services** (and need superuser access). We'll **always choose ports above 1024**.
> - ***Well-known ports*** are agreed on up front, e.g. **`ssh` uses 22**, so the daemon listens there and the client connects there by default.

### Looking Up the Address
We use **DNS** to translate **domain names** into **IP addresses** (e.g. `uwaterloo.ca` → `129.97.208.23`). Humans like names; the computer uses the number. (The command line tool for this is **`nslookup`**.)

C has a way to do this, prototyped in **`netdb.h`**:
```c
int getaddrinfo(const char *node, const char *service, const struct addrinfo *hints, struct addrinfo **res);
```

- **`node`**: the **hostname** to connect to (can also be an IP address)
- **`service`**: a protocol name like `"http"` or a **port number** like `"80"`
- **`hints`**: (optional) used to **restrict the kind of connection** you want (IPv4, TCP stream, etc.)
- **`res`**: a pointer to a pointer, which gets **updated with the result**
- Returns **0 on success**

> [!warning] Missing: `gethostbyname` is deprecated (L07: Looking Up the Address)
> Older examples use **`gethostbyname()`**, but it's **deprecated** and replaced by **`getaddrinfo()`**. You may still see it in the wild.

We should always pass an **explicit port number** like `"80"` for HTTP rather than `"http"`. HTTPS is **443**.

> [!warning] Correction: what the port advice was about (L07: Looking Up the Address)
> Your notes said "always use port 80 for http, this helps restrict the kind of connection you want". The recommendation is just to give **`service` an explicit port number string** (like `"80"`) instead of a name (like `"http"`). Restricting the kind of connection is the job of **`hints`**.

Trying it out:

```c
struct addrinfo hints;
struct addrinfo *serverinfo;

memset(&hints, 0, sizeof hints);
hints.ai_family = AF_INET;
hints.ai_socktype = SOCK_STREAM;
hints.ai_flags = AI_PASSIVE;

int result = getaddrinfo("www.example.com", "2520", &hints, &serverinfo);

if (result != 0) return -1;
struct sockaddr_in *sain = (struct sockaddr_in*) serverinfo->ai_addr;

// then free
freeaddrinfo(serverinfo);
```

- `memset` makes sure `hints` starts **empty**
- `AF_INET` = **IPv4**, `SOCK_STREAM` = **TCP stream sockets**, `AI_PASSIVE` = **fill in my IP for me**

Assuming everything went well, `serverinfo` now points to a **linked list of `struct addrinfo`** nodes. The actual address is in each node's **`ai_addr`** (a **`struct sockaddr`**, the generic form of `sockaddr_in`), and the pointer to the next node is **`ai_next`**. <u>*Most of the time we just need the first result*</u>. The `struct sockaddr` can be used in future calls, although we might need to **typecast** it to the type we need (as with `sain` above).

> [!warning] Correction: what the list contains (L07: Looking Up the Address)
> The list nodes are **`struct addrinfo`**, not `struct sockaddr`. The `struct sockaddr` is the **`ai_addr`** field *inside* each node.

If we instead want the structure for the **local computer**, we can either initialize a `struct sockaddr_in` **manually** (as we did before learning `getaddrinfo()`), or call `getaddrinfo()` with **`NULL` as the `node`** parameter.

> [!warning] Missing: `NULL` hints and cleanup (L07: Looking Up the Address)
> - You can pass **`NULL` for `hints`** if you're willing to accept the **defaults**.
> - The result list is **allocated for you**, so free it with **`freeaddrinfo()`** when done.

### Client: Connect
Up until now, everything we've learned (`socket` and creating the `struct sockaddr`) applies to **both the client and server** side. Now the paths diverge depending on whether the code we're running is the **client** or the **server**.

If we are the client, we'd like to connect to a server. This is the **easier workflow**; we just call **`connect()`**:

```c
int connect(int sockfd, struct sockaddr *addr, socklen_t len);
```

1. **`sockfd`**: the **socket file descriptor** (the `int` we got back from `socket`)
2. **`addr`**: a pointer to the **`struct sockaddr`**, whether manually created or returned from `getaddrinfo`
3. **`len`**: the **size of `addr`**. If we created the struct manually, use **`sizeof`**; if it came from `getaddrinfo()`, use its **`ai_addrlen`** attribute

Example using `getaddrinfo`:

```c
struct addrinfo hints;
struct addrinfo *res;
int sockfd;

memset(&hints, 0, sizeof(hints));
hints.ai_family = AF_INET;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo("www.uwaterloo.ca", "80", &hints, &res);
sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

int status = connect(sockfd, res->ai_addr, res->ai_addrlen);
```

Note the `socket` call takes its arguments **straight from the lookup result**. (Fixed a `)l` typo on the `memset` line.)

The return value of `connect` tells us whether we successfully connected: **zero means success**, anything else indicates an **error**.

The **man pages** describe the specific error codes. Printing the code out directly isn't super helpful, but you can **compare it against the named constants** defined in the man page. E.g. if it's **`ETIMEDOUT`**, you know the connection attempt **timed out**. The specs don't always tie a specific number to a specific error, so <u>*compare against the constants*</u> in your implementation, not raw numbers.

> [!warning] Correction: where the error code actually lives (beyond the lecture)
> The lecture says to compare `status` against `ETIMEDOUT`, but on Linux/POSIX `connect` returns **-1** on failure and puts the specific code in **`errno`**. So the real check is `if (status == -1 && errno == ETIMEDOUT)` (include `errno.h`). Same pattern as `kill` setting `errno` to `ESRCH` in L06.

Assuming we connected successfully, we're **ready to use the connection**. But first, the server side.

### Server: Bind, Listen, and Accept
The server does **3 steps in order**: **bind**, **listen**, then **accept**.
- **Bind**: choose **which port** we're going to use
- **Listen**: say the socket is **ready for connections** from a client
- **Accept**: **establish the connection** and start talking

**1. `bind()`** associates the socket with whatever **port** we want to use. When the `ssh` daemon is available for connections, it's because it has **bound itself to port 22** using `bind`. A quick example (without `getaddrinfo`):

```c
int socketfd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(2520);
addr.sin_addr.s_addr = htonl(INADDR_ANY);

bind(socketfd, (struct sockaddr*) &addr, sizeof(addr));
```

This **acquires port 2520** for our use. We haven't done anything with it yet, but we've **taken it for ourselves**. This step does **not happen on the client side**: we usually <u>*don't care what the client's outgoing port is*</u>, so we skip it unless we have a reason to care. (Fixed a `sin_fmaily` typo above.)

**2. `listen()`** marks the socket as **ready**. This is the simplest step:

```c
int listen(int sockfd, int backlog);
```

We listen on a socket that has **already been bound**, and allow a queue of up to **`backlog`** pending connections (usually limited to **~20**, depending on your system). If the queue is **full**, the server system **rejects further requests**.

**3. `accept()`** accepts incoming **`connect` requests**. (Phone analogy: `bind` = get a phone number, `listen` = turn the phone on, `accept` = press the green button.)

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *len);
```

The first parameter is the **socket we're listening on**. The second and third are **information about the client**: we **allocate them, pass them in**, and they're **filled in by `accept`**.

If we don't care who the client is, we can pass **`NULL` for the second and third params**. We don't need those values to communicate in both directions, but it's often helpful to know who the client is.

The return value is a <u>*new file descriptor for a new socket*</u>. **Further communication with the client happens over the new socket**, not the original. The **original socket stays open for accepting more connections**.

If `accept` is called and **no requests are in the queue**, the server is **blocked** until a request arrives. We simply wait for the connection.

Putting it all together (skipping error checking):

```c
struct sockaddr_in client_addr;
socklen_t client_addr_size = sizeof(struct sockaddr_in);
int newsockfd;

int socketfd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(2520);
server_addr.sin_addr.s_addr = htonl(INADDR_ANY);

bind(socketfd, (struct sockaddr*) &server_addr, sizeof(server_addr));
listen(socketfd, 5);
newsockfd = accept(socketfd, (struct sockaddr*) &client_addr, &client_addr_size);

close(newsockfd);

close(socketfd);
```

- `newsockfd` is closed when we're **done with this client**; `socketfd` is closed only **when all is done**.

> [!warning] Fixes to the server example (L07: Server: Bind, Listen, and Accept)
> - The socket was declared as `sockfd` but used as `socketfd`; now it's **`socketfd`** everywhere.
> - `server_addr.sin_addr = htonl(...)` should be **`server_addr.sin_addr.s_addr`** (`sin_addr` is a struct, `s_addr` is the integer inside it).
> - `new sockfd = accept(...)` → **`newsockfd = accept(...)`**.

> [!warning] Missing: accept in a loop, and the `NULL` version (L07: Server: Bind, Listen, and Accept)
> - Unless communication is a one-time thing, `accept` is usually called **in a loop**: accept a connection, do something useful with it, then go on to the next.
> - If we don't care about the client's address, we can drop `client_addr` and `client_addr_size` entirely:
> ```c
> newsockfd = accept(socketfd, NULL, NULL);
> ```

Then we're finally ready for the client and server to communicate: the **client uses its original socket fd**, and the **server uses the new fd** from `accept`.

There's a lot of setup, so in a program that does a lot with sockets the boilerplate usually gets wrapped in functions. For example, a client-side helper:

```c
int connect_to(const char *host, const char *port);
```

It does all the initialization: **gets the address info**, **creates the socket**, **calls `connect`**, **checks for errors**, and **returns the file descriptor** to use.

