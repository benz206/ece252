## Network Communication Continued
We should now have a good idea about how we establish connections for communication, but we haven't practiced actually sending or receiving any data. We will assume the same setup from lecture 7. If we want to send datagrams the workflow will be different but we will come back to that later.

Either side can send and receive data but figuring out whose turn it is to talk and whose turn it is to listen is on the client and server. Also what the content is, and how to interpret it is something that should be either predetermined or with some methodology.

The function for sending data is:

```c
int send(int sockfd, const void* msg, int length, int flags);
```

The first param is the same fd, and this is either the one returned to by the call to socket() or accept(). Then there is also same arbitrary bytes for you to send as data, and the third param is the length of what we are sending. The last parameter is the flags and for simple use cases we can just use 0 for this.

The return value of the function is the number of bytes sent. However, if something went wrong, this will be -1 and the `errno` variable will tell you more about what exactly went wrong. Under ideal circumstances, the number of bytes sent equals the length parameter.

```c
char *msg = "Hello_world!";
int len = strlen(msg, 13);
int sent = send(sockfd, msg, len, 0);
```

We technically should check if we got `-1` here but this will be a rare occurrence that we probably do not have to worry about.

