## Network Communication
Another form of inter process communication is the idea of network communication. If 2 processes are not running on the same machine in the same environment, then get them to communicate over the network. This can even be done if both processes are on the same device.

### Sockets
The *socket* API is a standard dating back many of years. It describes how to communicate over the network in a standard way. The socket is the concept for how to establish a communication channel between 2 processes. There are 2 ways of doing this, datagrams and connection streams.

A datagram is like mail, you can mail letters that might be delivered in any order. They could get lost, and are unidirectional. There is technically no connection because neither knows if they are connected to each other.

Streams are bidirectional and are like a telephone call where communication and connection must be established.

Much like everything else in Unix a socket is handled like a file, it just so happens that the data to be read or wrote from a file is being routed over the network somehow. To create a socket we use:

```c
int socket(int domain, int type, int protocol);
```

Domain defines the address format ipv4 or ipv6, we will just use ipv4 for AF_INET just for this course.

The type argument gives the kind of information we will be sending, SOCK_DGRAM is for datagrams and SOCK_STREAM is for a bidirectional byte stream

The protocol argument is how the data to be transported over the connection, we will just use default 0. This is TCP/IP we will just use this, datagram the default is UDP. This is not reliable since it may work, it may not.

We do need to close the socket when we are finished with it.

There is a short blurb on network order (big endian) and little endian (post order). We use big endian normally, and little endian is the one machines occasionally use

You can use `arpa/inet.h` to get the conversion functions, there are a few for translating 4 and 2 bytes to and from post and network order.
### Addresses
The structure for a socket address is `struct sockaddr_in`

We can use the below
```c
struct sockaddr_in {
	sa_family_t sin_family; // The address family
	in_port_t sin_port; // The port number
	struct in_addr sin_addr; // the ipv4 address
};

struct sockaddr_in addr;
addr.sin_family = AF_INET:
addr.sin_port = htons(2520);
addr.sin_addr.s_addr = htonl(INADDR_ANY);
```

We use a DNS to translate domains into ip addresses to then request website content.

C actually has a way to do this, this is:
```c
int getaddrinfo(const char *node, const char *service, const struct addrinfo *hints, struct addrinfo **res);
```

We should be always using a port number 80 for http, this helps restrict the kind of connection you want. Https is 443.

Trying it out

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

Assuming that everything went well, the `serverinfo` pointer is now pointing to a linked list of `struct sockaddr` (the generic form of sockaddr_in) which gives us the information we need (the ip address) The actual info struct is in the linked list node's `ai_addr` attribute and the pointer to the next one is just `ai_next`. Most of the time we just need the first result, though. The `struct sockaddr` structure we got from this may also be used in future calls although we might need to typecast it to properly use it.

If we are interested instead about getting the structure for the local computer, we may instead initialize the `struct sockaddr_in` as we did before, learning about how `getaddrinfo()` works. Or we may also call `getaddrinfo()` with NULL as the node parameter.

### Client: Connect
Up until now, all the tools we've learned about, `socket` and creating the `struct sockaddr` apply to both the client and server side in network communication. Now we will be discussing how the paths diverge depending on whether the code we're running is on the client or on the server.

If we are the client, we'd like to connect to a server, this is the easier workflow, and we will just call this `connect()`. This is done with a simple function listed below

```c
int connect(int sockfd, struct sockaddr *addr, socklen_t len);
```

- The parameters are simple enough, the first argument `sockfd` is the socket file descriptor (the int we got back from the call initially to the socket)
- The second parameter is a pointer to the `struct sockaddr` that we have, whether manually created or if it was a returned value from `getaddrinfo`.
- The last parameter is about the size of the second parameter. If we manually created the structure, we just use `sizeof`, if it was retuned from `getaddrinfo()` then there is also an attribute `ai_addrlen` provided. Consider an example using `getaddrinfo`

```c
struct addrinfo hints;
struct addrinfo *res;
int sockfd;

memset(&hints, 0, sizeof(hints))l
hints.ai_family = AF_INET;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo("www.uwaterloo.ca", "80", &hints, &res);
sockfd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);

int status = connect(sockfd, res->ai_addr, res->ai_addrlen);
```

The return value of this connection function determines whether we have successfully conncted or not. **Zero** means we were successful, while anything else indicates an error as normal.

The man pages describe specific error codes, printing it out unfortunately isn't super helpful but you can compare it against the constants defined in the man page.

Thus if you connect  and see the status variable equals `ETIMEDOUT` then you know the connection attempt timed out, e.g. you know what went wrong. There aren't always specific numbers associated with the error so you'll have to check assigned constants in the implementation you have.

Assuming that you have connected successfully, you're now ready to start using the connection, but let's also take a look at the server side of things.

### Server: Bind, Listen, and Accept
The overview of what steps the server is going to do in order is first bind, then listen, and finally accept. The bind step is how we choose the port we are going to connect to.

The listen step is then where we say the socket is ready for connections from a client. The final step being that we establish the connection and start talking.

1. Binding is how we associate the socket with whatever port we want to use. When the `ssh` daemon is available for connection, it's because it has to bound itself to the port 22 using `bind`
A quick example of this `bind()` function is as follows

```c
int socketfd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in addr;
addr.sin_fmaily = AF_INET;
addr.sin_port = htons(2520);
addr.sin_addr.s_addr = htonl(INADDR_ANY);

bind(socketfd, (struct sockaddr*) &addr, sizeof(addr));
```

This acquires port 2520 for our use. We haven't done anything with it yet but we've like taken it for ourselves, this did not happen on the client side, we usually do not care on the client side what the port number is so we usually just skip that step unless we have some reason not to.

2. Step 2 is to `listen()` which is basically just marking the socket as ready. This is the simplest step and you may call `int listen(int sockfd, int backlog);`
We listen on a socket that has been bound with bind and we allow a backlog for `backlog` connections that's usually limited to 20 or so, depending on your system. If the queue is full the server system will reject further requests.

Once we've acquired a socket we can start accepting incoming connect requests using `accept()`

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *len);
```

First parameter is the same as always, second is the information about the client, we must allocate them, pass them in, and they are then updated by the call to accept.

If we don't care about who the client is we can just pass in NULL for the second and third params. We don't really care about those values for communication in both directions, but it could be helpful in many contexts to know who exactly the client is.

The return value is a new file descriptor which describes a new socket. Further communication will take place over that socket and not the original one. The original one will remain for accepting connections and the new one is the socket just used for communication with the client.

If `accept` is called and no requests are in the queue, the server is **blocked** until a request arrives. We simply wait for the connection.

```c
struct sockaddr_in client_addr;
socklen_t client_addr_size = sizeof(struct sockaddr_in);
int newsockfd;

int sockfd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(2520);
server_addr.sin_addr = htonl(INADDR_ANY);

bind(socketfd, (struct sockaddr*) &server_addr, sizeof(server_addr));
listen(socketfd, 5);
new sockfd = accept(socketfd, (struct sockaddr*) &client_addr, &client_addr_size);

close(newsockfd);

close(socketfd);
```

Then we're finally ready for the client and server to communicate. It is likely that the program will do a lot with the sockets for example you might create some `connect_to` function which can do the initialization, getting address info, and also creating the socket, calling `connect` checking errors, etc.

