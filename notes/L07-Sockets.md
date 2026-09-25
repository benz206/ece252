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

There is a short blurb on network order (big endian) and little endian (post order).

You can use `arpa/inet.h` to get the conversion functions

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

Assuming that everything went well, the `serverinfo` pointer is now pointing to a linked list of `struct sockaddr` (the generic form of sockaddr_in) which gives us the information we need (the ip address) The actual info struct is in the linked list node's `ai_addr` attribute and the pointer to the next one 