## Copy-Modify-Merge
So far, shared data has followed ***Lock-Modify-Unlock***. But **`git`** (or `svn`) uses a different model: ***Copy-Modify-Merge***.

You and your lab partner each **check out** the repo, make changes **separately**, then **merge**. When changes are applied, everything in a group (a **commit**) should **succeed together or fail together**, never **half-done**.

### Transactions
A ***transaction*** is a group of operations that belong together and are treated as **one indivisible unit**.

We've seen what goes wrong when the **intermediate state** of a multi-step operation is **visible**: `x++` is a **read, add, write**, and concurrent access can read or write the **wrong value**. The usual fix is **mutual exclusion** (make others wait), and that scales up to changing `x, y, z, a, b, c` all at once. With copy-modify-merge, though, people change things **separately** and we **put it together** after.

A transaction looks like:
1. **Begin transaction**.
2. The **operations**.
3. **End transaction**.

Execution:
1. Write the transaction into a **log**.
2. Do the **operations**.
3. When the last one finishes, if all went well, **mark the transaction successful**.

On conflict:
- **Version control**: you're told about **merge conflicts**, and the merge can't happen until they're **resolved**.
- **File systems**: ***the last write wins***.

## inotify
Another way to use the file system for synchronization: **`inotify`** (**Linux only!** Not portable). Register your program as **interested in events** on a file or directory, and you're **informed** when one happens.

Key steps:
1. **Initialize** a management structure (you get back a **file descriptor** for it).
2. **Add** files you're interested in (and **remove** them later if you want).
3. **`read`** the fd to collect events. Each call returns **one or more event structures**.
4. When done, **`close`** the fd, which **cleans everything up**.

Watching a directory covers the **files in it**, but <u>*not subdirectories*</u>: it's **not recursive**.

```c
int inotify_init( ); /* Returns file descriptor referring to the struct */
int inotify_add_watch( int fd, const char* pathname, uint32_t mask );
int inotify_rm_watch( int fd, uint32_t wd );
```

- **`inotify_init`**: no arguments. Convenient.
- **`inotify_add_watch`**: the inotify fd, the **path**, and a **mask** of events you care about. You need **read permission** on the file. Adding a file that's **already watched** with a different mask **replaces** the mask.
	- **Save the return value** (the ***watch descriptor***): `inotify_rm_watch` needs it.
- **`inotify_rm_watch`**: remove that watch item.
- When completely done: **`close`** the inotify fd.

> [!note] Real types (beyond the lecture)
> In the man pages, `inotify_add_watch` returns an **`int`** (**-1** on error) and `inotify_rm_watch` takes **`int wd`**, not `uint32_t`.

### Events
There are about **23** events. The highlights:

| Bit Value | Description |
| --- | --- |
| **`IN_ACCESS`** | File **accessed** (read/execute) |
| **`IN_ATTRIB`** | **Metadata** changed, e.g. permissions |
| **`IN_CLOSE_WRITE`** | File opened **for writing** was **closed** |
| **`IN_CLOSE_NOWRITE`** | File **not** opened for writing was **closed** |
| **`IN_CREATE`** | File/directory **created** in watched directory |
| **`IN_DELETE`** | File/directory **deleted** from watched directory |
| **`IN_DELETE_SELF`** | The **watched file/directory itself** was deleted |
| **`IN_MODIFY`** | File **modified** (e.g. a write) |
| **`IN_OPEN`** | File **opened** |
| **`IN_ALL_EVENTS`** | **All** of the above (and a few more) |

### Reading an Event
`read` on the inotify fd gives back one or more:

```c
struct inotify_event {
  int      wd;       /* Watch descriptor */
  uint32_t mask;     /* Mask describing event */
  uint32_t cookie;   /* Unique cookie associating related events (for rename(2)) */  
  uint32_t len;      /* Size of name field */
  char     name[];   /* Optional null-terminated name */
};
```

- **`wd`**: which **watch** the event happened on.
- **`mask`**: what **kind** of event.
- **`cookie`**: a delicious cookie 🍪. A **rename** in a watched directory shows up as **two events**, and the cookie **links** them.
- **`len`** / **`name`**: the name and its length.

So one event takes up **`sizeof( struct inotify_event ) + len`** bytes.

#### How Big a Buffer?
Rare situation: usually we know **how many bytes** to read (e.g. the size of a struct), but here it depends on the **name**. The standard **clairvoyance problem**. Options:
- Make the buffer **really big**: `sizeof( struct inotify_event )` + the **longest filename + 1** (null terminator) works. If the buffer is **too small**, `read` **fails**.
- Ask: **`ioctl( fd, FIONREAD, &numbytes )`** sets `numbytes` to how many bytes are **available right now**.

If **multiple** events happened, you can get **multiple structs** back (if the buffer's big enough).

### Example: Wait for a Lock File to Be Deleted
Combine L24's lock file with inotify: if the lock exists, **wait for it to be deleted** instead of spinning.

```c
#include <stdlib.h>
#include <stdio.h>
#include <sys/inotify.h>
#include <stdbool.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

const char filename[] = "file.lock";

int main( int argc, char** argv ) {
    int lockFD;
    bool our_turn = false;
    
    while( !our_turn ) {
        lockFD = open( filename, O_CREAT | O_EXCL | O_WRONLY, 0644 );
        if ( lockFD == -1 ) {
            printf( "The lock file exists and process %d will wait its turn...\n", getpid() ); 
            int notifyFD = inotify_init( );
            uint32_t watched = inotify_add_watch( notifyFD, filename, IN_DELETE_SELF );
            
            /* Read the file descriptor for the notify -- we get blocked here
               until there's an event that we want */
            int buffer_size = sizeof( struct inotify_event ) + strlen( filename ) + 1;
            char* event_buffer = malloc( buffer_size );
            printf("Setup complete, waiting for event...\n");
            read( notifyFD, event_buffer, buffer_size );

            struct inotify_event* event = (struct inotify_event*) event_buffer;
            /* Here we can look and see what arrived and decide what to do.
               In this example, we're only watching one file and one type
               of event, so we don't need to make any decisions now */

            printf("Event occurred!\n");

            free( event_buffer );
            inotify_rm_watch( notifyFD, watched );
            close( notifyFD );
        } else {
            char* pid = malloc( 32 );
            memset( pid, 0, 32 );
            int bytes_of_pid = sprintf( pid, "%d", getpid() );

            write( lockFD, pid, bytes_of_pid ); 
            free ( pid );
            close( lockFD );
            our_turn = true; 
        }
    } 

    printf("Process %d is in the area protected by file lock.\n", getpid());
    remove( filename );
    return 0;
}
```

- Try to **create** the lock file (`O_CREAT | O_EXCL`).
- **Fails** (it exists): set up inotify to watch for **`IN_DELETE_SELF`**, then **`read`**, which **blocks** until the file is deleted. Clean up and **loop** to try again (someone else might win).
- **Succeeds**: write our **PID** into the lock file, and we're in.
- When done, **`remove`** the lock file, which wakes up any waiters.

Demo: start it while the lock file **already exists**, then delete it.

> [!warning] A lost wakeup (beyond the lecture)
> If the lock file is deleted **after** `open` fails but **before** `inotify_add_watch` runs, the watch fails (**no such file**, returns -1) and `read` waits for an event that **never comes**. Same idea as the **lost wakeup** in L21. Checking `inotify_add_watch`'s return value and **looping back** to `open` on failure fixes it.

> [!tip] Other platforms (beyond the lecture)
> macOS/BSD have **kqueue** (`EVFILT_VNODE`) and macOS has **FSEvents**; Windows has **`ReadDirectoryChangesW`**. Cross-platform file-watcher libraries wrap all of these.

## Consistency: Journalling
An **error, crash, or power failure** can lose data or leave the file system **inconsistent**. Directories, pointers, inodes etc. are all **data structures**, and if they're **corrupted**, things go badly.

### Check and Repair
Scan for inconsistencies periodically (e.g. **on boot**). Many OSes do this:
- UNIX: **`fsck`** (not something you want to say out loud).
- Windows: **`chkdsk`** (check disk).

They look for **inconsistent states** (e.g. a file claims **12 blocks** but its linked list has only **5**) and try to **repair** them. How well that works depends on the **problem** and the **file system**. And scanning the **whole disk** takes a **very long time**.

### Prevent It: Transactions
Better to **prevent** the problem. Same idea as version control: a change happens **entirely or not at all**.

## ZFS (Sun/Oracle)
Pronounced "**Zee Eff Ess**". Built for **scalability**.

- Uses **transactions**: the on-disk state is **always consistent**.
- Like copy-modify-merge: data is **copied**, **changed**, then **rewritten**. <u>*Blocks are never overwritten.*</u>
	1. A transaction writes all **data and metadata** to **new blocks**.
	2. Only when it's **complete** are references to the old blocks **replaced** with the new locations.
	3. Then old blocks are **cleaned up** (reused or disposed of).

This is ***copy-on-write***.

**Weakness**: if the disk is **completely full**, you **can't delete anything** to make space, because deleting needs **new blocks** to write the change. Not hypothetical: it happened to the prof's friend with an overstuffed **NAS**. He was mad. (He was going to buy a new one anyway...)

## APFS (Apple)
**Snapshots**, like some version control systems: **freeze** the file system's state, then later changes are stored as **diffs** against it, so only **new** things take space.
- **Faster backups**: normally backups slow your system down while computing the difference since the last backup. With snapshots, that's **faster**.
- **Avoids corruption**: replay changes to get back to **a** consistent state (of some sort).

### Clones Hurt the "Copy It to Another Folder" Backup
The most common backup for non-technical users: **copy the file into a different folder**. On APFS, copying a file **on the same volume** (with no changes) **doesn't copy the data**: you just get **two references to the same blocks**.
- Looks like **saving space**...
- ...but it's **bad for redundancy**: if that part of the disk is damaged, **all copies are lost**.

Crash consistency: something like **copy-on-write**, but Apple is **vague**. The lead developer (Dominic Giampaolo) calls it a "**novel copy-on-write metadata scheme**", somehow not quite ZFS's single-atomic-update approach.

## NTFS (Windows)
Default Windows file system since **Windows NT** (2000, XP, Vista, 7, 8, 10, 11...). Supports **large disks and files**, and uses ***journalling***.

![[ntfs-volume.png]]

The ***Master File Table (MFT)*** has info about **all files and folders**. After it comes a block of **system files**:
1. **MFT2**: a **mirror** of the first few rows of the MFT, in case the original is **damaged**.
2. **Log File**: the **journalling transaction log**.
3. **Cluster Bitmap**: which clusters are **in use**.
4. **Attribute Definition Table**: attribute types **supported** on this volume.

### How Journalling Works
Goal: the file system is **always consistent**, even after a crash. A service maintains the **log file** used for **recovery**.

<u>*Recovery only guarantees the **metadata** is consistent; **user data can still be lost**.*</u> Microsoft chose this to make recovery much **simpler and faster**.

1. All **metadata changes** are written **sequentially to the log**. Once logged, control **returns to the program**.
2. Meanwhile, the logged changes are **actually carried out**. A **pointer** tracks which entries have **really happened**.
3. When a whole transaction is **complete**, it's **removed** from the log.

After a **crash**, the log has **zero or more** transactions:
- **Zero**: nothing was in progress. No problem.
- **Some, committed**: they weren't finished, so **carry out** the remaining operations.
- **Some, aborted (not committed)**: **walk backwards** through the log, **undoing** completed operations, back to the state **before the transaction**.

A write might still be **lost** in a crash, but the file system stays **consistent**. Bonus: writes can sometimes be **reordered** for better **disk utilization**.

### Implementation
1. Record the change(s) in the **log file, in the cache**.
2. Modify the **volume, in the cache**.
3. The cache manager **flushes the log** to disk.
4. **Only after** the log is on disk, flush the **volume changes**.

<u>*The log always hits the disk before the change it describes*</u>, so after a crash the log knows about **anything** that might be half-done.

> [!tip] Linux too (beyond the lecture)
> **ext3/ext4** journal the same way. By default ext4 (`data=ordered`) journals **metadata** only, like NTFS. **Btrfs** is copy-on-write, like ZFS.

### Asynchronous!
The changes happen **in the background**: a program says it wants to write and **doesn't wait** for the write to finish before moving on. Can our **regular programs** do that? **Yes**: that's the next few lectures, **asynchronous I/O**.
