## IPC: Same System
So far, the IPC we've seen has been **message passing** (signals, message queues, the network). When both processes are **on the same system**, we can also use **pipes** and **shared memory**. Pipes are **similar to both** message passing and shared memory, so they bridge the two nicely.

### UNIX Pipes
A ***pipe*** sets up communication between a **producer** and a **consumer**: the producer **writes into one end**, and the consumer **reads from the other**.

Pipes are ***unidirectional***. If we want communication **in both directions**, we need **two pipes** (one going each way).

The function to create one is:

```c
int pipe(int fileDescriptors[]);
```

- **`fileDescriptors[0]`** is the **read end**
- **`fileDescriptors[1]`** is the **write end**

Yes, these are **file descriptors**: UNIX treats a pipe **as a file** (UNIX thinks *everything* is a file), even though it <u>*lives in memory*</u>, not on disk. That means we read and write it with the usual **`read`**/**`write`**/**`close`**.

![[unix-pipe.png]]

#### How a Pipe Works
The pipe is a **block of main memory** treated as a ***circular queue***. Each entry is **fixed in size**, usually **one character**.
- The **sender** can put the message in **in small chunks**.
- The **receiver** gets the data **one character at a time**.

So, just like with sockets, the sender and receiver need to <u>*agree on how to tell when the message is finished*</u>:
- a **termination character** (e.g. line feed `\n` or the null character `\0`), or
- the message **starts with its length** (the number of characters to expect).

#### Named Pipes
A pipe can be given a **name on disk** (the data is **still in memory**). This is a ***named pipe***.

| | Regular pipe | Named pipe |
| --- | --- | --- |
| **Lifetime** | Only exists **while the processes are communicating** | **Persists** even after the creating process terminates |
| **Who can use it** | Needs a **parent-child relationship** (the file descriptors have to be passed down via `fork`) | **Any process** can use it |
| **Direction** | **One way only** | **Bidirectional**, but <u>*only one direction at a time*</u> |

If we need communication going **both directions at once**, we still need a **second pipe** even with named pipes.

A named pipe is created with **`mkfifo`** ("make first-in-first-out"), since a named pipe is sometimes called a ***FIFO***. Because it's a file, it's used with the normal file calls: **`open`**, **`read`**, **`write`**, and **`close`**.

#### Command Line Pipes
We've already used pipes on the command line:

```bash
cat fork.c | less
```

This creates a pipe that takes the **output of `cat`** and feeds it as the **input to `less`** (which lets you scroll/paginate through it).

#### Code Example
Combining pipes with **`fork`**: the parent creates a pipe, spawns a child, and sends it the message `"Greetings"`.

```c
#include <sys/types.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#define BUFFER_SIZE 25
#define READ_END 0
#define WRITE_END 1

int main( int argc, char** argv ) {
	char write_msg[BUFFER_SIZE] = "Greetings";
	char read_msg[BUFFER_SIZE];
	int fd[2];
	pid_t pid;

	if (pipe(fd) == -1) {
		fprintf(stderr,"Pipe failed");
		return 1;
	}

	pid = fork();
	if (pid < 0) { /* error occurred */
		fprintf(stderr, "Fork Failed");
		return 1;
	}

	if (pid > 0) { /* parent process */
		/* close the unused end of the pipe */
		close(fd[READ_END]);

		/* write to the pipe */
		write(fd[WRITE_END], write_msg, strlen(write_msg));

		/* close the write end of the pipe */
		close(fd[WRITE_END]);

	} else { /* child process */
		/* close the unused end of the pipe */
		close(fd[WRITE_END]);

		/* read from the pipe */
		read(fd[READ_END], read_msg, BUFFER_SIZE);
		printf("read %s",read_msg);
		/* close the write end of the pipe */
		close(fd[READ_END]);
	}
	return 0;
}
```

- The pipe is created **before `fork`**, so the child **inherits both file descriptors**. This is why regular pipes need a parent-child relationship.
- Each side **closes the end it doesn't use**: the parent only writes, the child only reads.
- **`READ_END`** and **`WRITE_END`** are just `#define`s for **0** and **1** to make the code readable.

**Sometimes the output has random characters after "Greetings".** Why? Strings in C are **null-terminated**, so `printf` with `%s` keeps printing <u>*until it happens to hit a 0 byte*</u>. `strlen(write_msg)` **doesn't count the null terminator**, so it's never sent, and `read_msg` is **uninitialized**, so whatever garbage follows the message gets printed.

The fix: when you have a buffer, **initialize it to 0** first, e.g. **`memset(read_msg, 0, BUFFER_SIZE);`** before reading.

> [!warning] Things the example glosses over (beyond the lecture)
> - The last comment in the child says "close the **write** end" but the code closes the **read** end (the comment is wrong, the code is right).
> - Closing unused ends isn't just tidiness: `read` only returns **0 (end of file)** once **every** copy of the write end is closed. If the child kept its own copy of the write end open, a reader waiting for EOF would wait forever.
> - Neither `write` nor `read` is checked for `-1`, and `read` might return fewer bytes than were sent (same idea as partial `recv` in L08).

### Shared Memory
The idea is simple: a region of memory is **designated as shared** between multiple processes, and **all of them can read and write it**. To do this, the **OS must be told**.

Normally, a region of memory belongs to **exactly one process** (its owner). If another process tries to access it, the **OS intervenes** and that's an **error**. So to share memory, a process has to:
- tell the OS it's **okay for the other process to access** that area, and
- let the OS know the memory is **referenced by more than one process**. If the first process terminates and is reaped, the second may <u>*still be using it*</u>, so it must **not be freed** until nobody is using it.

Once the memory is shared, accessing it is **just a normal memory access**. The **kernel is only involved in setup and cleanup**, which is what makes shared memory fast.

![[shared-memory.png]]

In the diagram the shared region sits **between** A and B, but it <u>*could be anywhere*</u>. It tends to be in the memory of the process that **created** it: process A asks the OS for memory, then asks the OS to treat a block A **already owns** as shared.

> [!tip] Coordination
> With shared memory, one process can **overwrite another's changes**. Preventing that needs a **coordination mechanism**, which comes later in the course.

#### Keys
To share memory we need a ***key*** that identifies the segment. We get it **the same way as for message queues** (L06):
- the constant **`IPC_PRIVATE`**, or
- the "file to key" function from **`sys/ipc.h`**:

```c
key_t ftok(char *pathname, int proj);
```

#### Workflow
The functions are in **`sys/shm.h`**. The steps are:
1. **Create** a shared memory segment: **`shmget`**
2. **Attach** it (add the segment to the calling process): **`shmat`**
3. **Use** the shared memory
4. **Detach** from it: **`shmdt`** (like deallocation; if you forget, it happens automatically when the process ends)
5. **Delete** the segment, done by **one process only**, after everyone has detached: **`shmctl`**

```c
int shmget( key_t key, size_t size, int shmflg );
void* shmat( int shmid, const void* shmaddr, int shmflg );
int shmdt( const void* shmaddr );
int shmctl( int shmid, int cmd, struct shmid_ds *buf );
```

#### Create: `shmget`
Creates a segment, or **gets a reference to an existing one**.
1. **`key`**: the result of **`ftok()`**, or **`IPC_PRIVATE`**
2. **`size`**: **how many bytes** to share
3. **`shmflg`**: the **access permissions**, using the usual UNIX format (e.g. **`0600`**). Optionally OR'd (`|`) with:
	- **`IPC_CREAT`**: if no segment with this key exists, **create it**
	- **`IPC_EXCL`**: used with create; **fail if a segment with this key already exists**

This works just like the flags for opening a file. The **return value** is the **integer ID** of the segment, which we use to attach it.

#### Attach: `shmat`
1. **`shmid`**: the ID returned by **`shmget`**
2. **`shmaddr`**: where to put the memory. <u>*Always use `NULL`*</u> and let the kernel decide. Otherwise the program may break on other machines (or even at different times) because that address might already be in use.
3. **`shmflg`**: optionally **`SHM_RDONLY`** to prevent writing (attempting to write then causes a **segmentation fault**)

The **return value** is a **normal C pointer** to the shared memory. From here we use it **like any other memory**: <u>*no `read`/`write` calls*</u>, just read and assign variables.

**How does another process find the segment?** If we created it, we obviously know. For other processes:
- If two processes call **`ftok()` with the same inputs**, they get the **same key**.
- If a parent **attaches a segment and then calls `fork()`**, the child **inherits** the attached segment, so it's already set up.

#### Detach: `shmdt`
Takes the **address returned by `shmat`** (not the ID). If we forget, it happens automatically at process termination, but **detach as soon as you're done**, the same way you'd free memory you no longer need.

#### Delete: `shmctl`
`shmctl` can do a lot more (e.g. modify the properties of the segment's control structure), but at minimum we use it to **delete** the segment:
- Pass the **segment ID** and the command **`IPC_RMID`** ("remove ID")
- The last argument **must be `NULL`** for deletion

<u>*Deletion may be deferred*</u>: as long as the segment is **still attached somewhere**, it can't be cleaned up. It's like a UNIX file that's still open: it isn't actually removed until the **last process closes it**. Linux lets you attach to a segment that's **marked for deletion** but still exists, but this <u>*isn't consistent across UNIX systems*</u>, so **don't do it**.

#### Shared Memory Example
The child sends a message to the parent through shared memory:

```c
#define _XOPEN_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sys/shm.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

int main( int argc, char** argv ) {

	int shmid = shmget( IPC_PRIVATE, 32, IPC_CREAT | 0666 );

	int pid = fork();
	if ( pid > 0 ) { /* Parent */
		waitpid( pid, NULL, 0 );
		void* mem = shmat( shmid, NULL, 0 );
		printf("The message the parent received from the child is %s.\n", (char*) mem );
		shmdt( mem );
		shmctl( shmid, IPC_RMID, NULL );
	} else if ( pid == 0 ) { /* Child */
		void* mem = shmat( shmid, NULL, 0 );
		memset( mem, 0, 32 );
		sprintf( mem, "Hello World" );
		shmdt( mem );
	}

	return 0;
}
```

- **`shmget`** is called **before `fork`**, so both processes **know the ID** (`shmid` is just an `int` copied into the child). Each side then attaches on its own.
- The child **attaches**, **zeroes** the 32 bytes with `memset` (so the string is definitely null-terminated), writes `"Hello World"`, and **detaches**.
- The parent **waits for the child** with `waitpid` before reading, so the message is guaranteed to be there. The segment **survives the child exiting** because it hasn't been deleted.
- The parent reads it, **detaches**, and **deletes** the segment (only one process deletes).

#### Security
Can anyone who knows the key or ID attach to your shared memory? **No.** You can only attach to segments **from your own processes**, or ones whose **permission bits allow it**. A process run by user `j999doe` can only attach to memory shared by another process **run by that same user**. The owner can reassign ownership with **`shmctl()`**, but that's beyond the scope of the course.

### Alternative: `mmap`
Another way to share memory is **`mmap()`**, which is nominally for ***mapping a file into memory***. Reading from the mapped buffer **reads the file**, and writing to it **writes the file**, so we can treat file access **like memory access**. We can also use it **for IPC**.

```c
void* mmap( void* address, size_t length, int protection, int flag, int fd, off_t offset );
int mprotect( void* address, size_t length, int prot );
int msync( void* address, size_t length, int flags );
int munmap( void* address, size_t length );
```

#### Mapping: `mmap`
1. **`address`**: where the mapped region should go. Unless you have a **very good reason**, use **`NULL`** and let the system choose.
2. **`length`**: **how many bytes** to map
3. **`protection`**: the rules for how the memory **can be used** (below)
4. **`flag`**: the **mode** the memory is mapped in (below)
5. **`fd`**: the **file descriptor** of the file to map
6. **`offset`**: **how far from the start of the file** the mapping begins

The **return value** is a **pointer to the start** of the mapped memory.

#### Protection Flags
Valid values, combined with **bitwise OR**:
- **`PROT_READ`**, **`PROT_WRITE`**, **`PROT_EXEC`**: what you'd expect
- **`PROT_NONE`**: **forbids all access**. Any access is a **segmentation fault**.

The flags have to be <u>*consistent with how the file was `open`ed*</u>: you can't `open` a file read-only and then `mmap` it with `PROT_WRITE`.

**Why would you ever use `PROT_NONE`?** To put ***guards*** at the **start and end** of a region of memory. If the process accidentally goes **out of bounds**, it gets a **segmentation fault**, which **enforces the edges** of the area.

#### Mapping Flags
There are two options:
- **`MAP_PRIVATE`**: changes are **not visible** to other processes mapping the same file and are **not written** to the underlying file. For when we just want to **work with a file ourselves**.
- **`MAP_SHARED`**: changes **are visible** to other processes and **are written** to the file (though <u>*maybe not instantly*</u>). This is the one for **IPC**.

![[memory-mapped-file.png]]

#### `mprotect`, `msync`, and `munmap`
- **`mprotect`**: **changes the protection rules** for a section. Takes the **address**, the **size**, and the **new protection rules**.
- **`msync`**: In shared mode, changes **aren't necessarily written to the file immediately** (partly for performance), so processes might **disagree** about the file's content. `msync` forces the sync. Takes the **address** and **length**, and for flags just use **`MS_SYNC`**, which makes it a **blocking** call. It **does nothing** if the mapping is **private**.
- **`munmap`**: **unmaps** a segment when we're done. Takes the **address** and **length**. Segments are unmapped automatically when the process exits, but it's polite to **unmap as soon as you're done**.

#### `mmap` Example
`example.txt` contains `"This is sample text."`. The child **overwrites** it with `"It is now Overwritten"`, and the parent **sees the change**.

```c
#define _XOPEN_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sys/shm.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/mman.h>

int main( int argc, char** argv ) {

	int fd = open( "example.txt", O_RDWR );

	struct stat st;
	stat( "example.txt", &st );
	ssize_t size = st.st_size;
	void* mapped = mmap( NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0 );

	int pid = fork();
	if ( pid > 0 ) { /* Parent */
		waitpid( pid, NULL, 0 );
		printf("The new content of the file is: %s.\n", (char*) mapped);
		munmap( mapped, size );
	} else if ( pid == 0 ) { /* Child */
		memset( mapped, 0, size ); /* Erase what's there */
		sprintf( mapped, "It is now Overwritten");
		/* Ensure data is synchronized */
		msync( mapped, size, MS_SYNC );
		munmap( mapped, size );
	}
	close( fd );
	return 0;
}
```

- The file is opened **`O_RDWR`**, which matches mapping it with **`PROT_READ | PROT_WRITE`**.
- **`stat`** gets the **file size** so we know **how many bytes to map**.
- It's mapped **`MAP_SHARED`** and **before `fork`**, so the child **inherits the mapping** and both processes see the same memory.
- The child **erases** the content, writes the new message, calls **`msync`** to make sure it reaches the file, then **unmaps**.
- The parent **waits** for the child, prints the new content, and **unmaps**.

**The example is flawed.** It works in the sense that the data is overwritten and the parent sees it, but <u>*the mapped area's size can't change*</u>:
- Write **less** than the original and things look **strange** (the rest of the file is left as the null bytes from `memset`; the file doesn't shrink).
- Write **more** and the file **can't grow** to hold it.

Linux has **`mremap`** to resize a mapping, but it's **not portable** (doesn't work on all systems). Fixed size is fine for some problems though, like **sorting an array**: the sorted array is the **same size as the input**, and multiple processes could **share the work**.

> [!warning] The example is also flawed the other way (beyond the lecture)
> `"It is now Overwritten"` is **22 bytes** with its null terminator, but `"This is sample text."` is only **20** (21 with a trailing newline). So the example actually writes **past the end of the file**. It doesn't crash because mappings are rounded up to a whole **page**, and the parent still prints the full message (it's reading memory, not the file), but bytes past the file's size **never make it into the file**. With a 20-byte file, `example.txt` ends up as `It is now Overwritte` (the final `n` is lost).
