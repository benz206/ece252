## File Systems
- Important and very useful to programs
- It provides both **persistent data storage** and **organization** through a **directory structure**, while maintaining **metadata** related to files
- A **file** is just a *logical unit* to organize 0s and 1s
- The **UNIX approach** is to just treat <u>*everything as a file*</u>, which gives you a very broad array of functions that can be generalized to everything
- Files typically have **attributes**, which are generally as follows:
	- **Name**: symbolic, human-readable form
	- **Identifier**: the unique identifier (usually a number), which identifies the file *inside the file system*
	- **Type**: what the file type is
	- **Location**: physical location and what device it is on
	- **Size**: current, and possibly max, size of the file
	- **Protection**: who owns it, and what r/w permissions exist
	- **Time, Date, User ID**: the owner of the file, time of creation, last change, basically other metadata
- Files are maintained in a **tree-like directory structure**. At the end of the day, [**directories**](#directories) are also just files: they store information about what files are in what locations, and are also stored on the disk

## File Operations
- The **6 basic operations** on files are as follows:
	- **Create**
	- **Read**
	- **Update** (write)
	- **Delete**
	- **Repositioning**
	- **Truncating**
- An example of opening and closing a file is below

```c
FILE *f = fopen(argv[1], "r");
if (f == NULL) {
	printf("unable to open file\n");
	return -1;
}
readfile(f);
fclose(f);
```

> [!warning] Missing: keep files open while you still need them (L03: File Operations)
> Repeatedly opening and closing the same file is **unnecessary and inefficient**. If you will write some data, do other work, then write more, keep the file open in between.

### Creating a File
Like allocating memory, creating a new file has some essential steps: first **find a place** to put that file, **allocate** that space and mark it as allocated, and finally **put the file in its appropriate directory**.

### Writing a File
This requires the **name/identifier** of the file and the **data** to be written. The system will then find the file and put in the data. It can either *replace* the contents or *append* to the end, and there is also a **pointer** used to track where the next write happens (you can seek it to write at specific parts of the file).

### Reading a File
This requires the **name/id** of the file and **where in memory** the next block of the file should be put. There also exists a **pointer** which indicates where the next read will take place.

### Truncating a File
If a file's contents should be erased but we want to **keep its metadata**, we can **truncate** it and cut off all of the contents. The file length is then reset to *zero* and the data area may be marked as free, while the rest of the attributes remain the same.

For everything above (create, read, write, truncate), it turns out that we need an **open call**. All modes are listed in the PDF.

> [!warning] Missing: the `fopen` modes you are expected to know (L03: File Operations)
> The call is `FILE *fopen(const char *filename, const char *mode)`.
>
> | Mode | Meaning | Creates? | Truncates? | Starts at |
> |---|---|---|---|---|
> | `r` | read | no (fails if missing) | no | beginning |
> | `w` | write | yes | **yes** | beginning |
> | `a` | append | yes | no | end |
> | `r+` | read + write | no | no | beginning |
> | `w+` | read + write | yes | **yes** | beginning |
> | `a+` | read + append | yes | no | writes always go to the end; the initial *read* position is platform-dependent (glibc: beginning; macOS/BSD/Android: end) |
>
> Adding `b` (e.g. `rb`) opens the file in **binary** mode. Since C11, adding `x` (e.g. `wx`) makes the open **fail if the file already exists**. Note that `w` silently destroys existing contents; this is how truncation happens in practice.

### Repositioning a File
Since files can be read or written, but usually only one at a time, the pointer for the write location might be the *same* pointer as for reading (a **current position pointer**). As such, we might need the **`fseek()`** call, which adjusts the pointer's position for both reading and writing.

This should be done with *caution* though, as you can go to an **arbitrary location**.

Generally, we only need to seek if we want to **skip ahead or go back**.

> [!warning] Missing: why seeking is dangerous and when it's unnecessary (L03: Repositioning within a File)
> - An arbitrary location can land in the **middle of a multi-byte character**.
> - You **can't seek for writing** in a file opened in **append** mode; writes still go to the end.
> - Reading or writing *n* bytes already **advances the pointer by *n* automatically**. If you read 48 bytes and then seek 48 forward, you end up **96** bytes from where you started, not 48.

### Deleting a File
Deletion works as one would expect: **find the file**, **mark its space as free**, and then just **remove it from the directory listing**. This is the equivalent of a *soft delete*.

> [!warning] Missing: what "soft" deletion means and how to delete in C (L03: Deleting a File)
> The data is **not actually erased**; the file system just forgets the file exists. The data may be **recoverable** until that space is overwritten, similar to how a freed pointer in C may still point at the old data. Some systems offer **secure deletion**, which overwrites the old space with zeros.
>
> In C, delete a file with **`remove(path)`**, e.g. `remove(argv[1]);`.

These **6 fundamental operations** can be combined to do basically everything else, such as *copying a file* (create a new file, read from the old one, write into the new one).

> [!warning] Missing: the OS tracks open files (L03: File Operations)
> Apart from **create** and **delete**, every operation only works on a file that is **open**. Opening a file gives the program a **reference** to it, and the OS keeps track of **which files are open in which process**. You should close files when you're done, but when a process terminates, its open files are closed automatically.

### File Locking
We can also **lock** files, so that other programs that try to access the file are either *warned* or *denied*.

**Windows** uses locking such that any file that is *open in a program* cannot be deleted. **UNIX**, on the other hand, will still let that file be deleted, and it will be removed from the directory. But once the program finishes with it, it will then be removed properly.

> [!warning] Correction: it's open *files*, not locked programs, and UNIX doesn't lock by default (L03: File Operations)
> - Locks can be **exclusive** or **shared** (non-exclusive).
> - Windows prevents deleting a file that is **open in some program**; this isn't about programs being locked.
> - UNIX does **not lock by default**. Programs *can* lock files if they need to, but otherwise another user can delete a file that is open elsewhere. The deleted file disappears from the directory, but the program that still has it open can **keep reading and writing it**. Its storage is only freed once **no program has it open** anymore.

Locking in Linux uses **`flock()`**, which takes a **file descriptor**, not a `FILE *`, so we convert with **`fileno()`**:

```c
FILE *f = fopen("file.txt", "r");
int file_descriptor = fileno(f);
int result = flock(file_descriptor, LOCK_EX);
// LOCK_UN for unlock, and shared is LOCK_SH
```

## Reading and Writing
Writing to a file is basically the same as using `printf`. In fact, the function is simply **`fprintf`**, and the only difference between it and `printf` is that the **first argument** is the file pointer `fp` where you want the data to be written to.

```c
void write(point_t* p, FILE *fp) {
	while (p != NULL) {
		fprintf(fp, "something %s", p->data);
		p = p->next;
	}
}
```

> [!warning] Correction: the example was missing the file pointer (L03: Reading and Writing)
> Your original example called `fprintf("something %s", p->data)`, which leaves out exactly the argument that makes it `fprintf`. It's been fixed above to `fprintf(fp, ...)`.

Reading is a bit more of a pain: you need to use **`fscanf`**, which is a *mirror* of `fprintf` (same format specifiers).

```c
int main(int argc, char **argv) {
	FILE *fp;
	int i, isquared;
	
	fp = fopen("results.txt", "r");
	if (fp == NULL) return -1;
	
	while (fscanf(fp, "%d,%d\n", &i, &isquared) == 2) {
		printf("blahblahblah\n");
	}
	
	fclose(fp);
	return 0;
}
```

The **return value** of `fscanf` is the <u>*number of elements successfully read*</u>, which in this case is supposed to be two. However, it's worth noting that there's no space in the format string, because `%d` skips leading whitespace, occasionally leading to *hard-to-find bugs*.

> [!warning] Clarification: what the "no space" remark means (L03: Reading and Writing)
> The format is `"%d,%d\n"`, with **no space after the comma**. It still reads a line like `3, 9` correctly, because **`%d` skips any leading whitespace on its own**. The bugs come from forgetting that this happens. The loop stops when `fscanf` returns anything other than `2`, meaning either **end of file** (`EOF`) or a line that doesn't match the format.

We could also use **`getline`**, which is said to be in previous examples, but I don't recall seeing it.

> [!warning] Clarification: `getline` (L03: Reading and Writing)
> This lecture only mentions `getline` in passing and doesn't show it. `getline(&line, &len, fp)` reads **one entire line**, including the `\n`, into a heap buffer that it grows as needed; you `free(line)` at the end. For reading user input from the keyboard, there is also plain `scanf`.

## File Types
The file type can sometimes be determined from the **extension** of a filename (e.g. `.txt`). However, this is just a *hint* to the operating system as to what program should open the file; the actual contents may differ significantly, and any program can open any file.

Operating systems typically give you an option to select what **default program** opens each file type.

### Directories
- A **directory** is just a *symbol table* that translates file names to their **directory entries**
- Common operations are listed below:
	- **Search**: we want to be able to find a file, and searching is typically not just by file name, but may also include the actual contents of the file if they are human-readable
	- **Add** a file to the directory
	- **Remove** a file from the directory
	- **List** the contents of a directory
	- **Rename** a file
	- **Navigate** the file system
- The lecture talks about some very uninteresting other structures, but just assume we have the standard **tree** with subtrees
- In UNIX, the **root directory** is just **`/`**. You also do not need to use **absolute paths**; you can use **relative paths**, and the current directory will be *prepended* automatically for you
	- e.g. from `/home/jz/ece252/`, `gcc code/example.c` resolves to `/home/jz/ece252/code/example.c`
- OS designers have to make a choice about how **deleting a non-empty directory** goes: either you *delete everything inside it*, or you *refuse* until it's empty
	- Separately, many systems don't truly delete at all at first, and instead move things to a **Recycle Bin/Trash**, where they can still be restored
- File systems can also support **sharing files**: there is *one copy* of the file but it has *more than one name*. In UNIX this is called a **link**, and it's essentially a pointer to another file
	- We have either **hardlinks** or **symlinks**

### Symlinks/Hardlinks
**Symbolic links** are just *references by file name*, so they act as a **shortcut**. If the target file is then deleted, the symlink is left pointing to nothing, and trying to use it will error out.

**Hardlinks** create a *second pointer to the underlying file* in the file system. If a hardlink exists and the user deletes the file, the file will remain on the disk until the **last hardlink is removed**.

We will see later how this actually works, but we just maintain a **count of hardlinks** (*reference counting*) and only delete the file when the count becomes **0**.

## File Permissions
- To protect users from one another and to maintain the **confidentiality** and **integrity** of data, files usually have some **permissions** associated with them, which may control access to the following operations:
	- **Read**, **write**, **execute**, **append** (write at the end), **delete**, **list** (view the file's attributes)
- **UNIX-style permissions** are commonly used and work as follows:
	- Each file has an **owner** and a **group**, as well as a set of permissions that can be assigned to the *owner*, the *group*, and *everyone*
	- There are **3 basic permissions**: read, write, and execute (run as a program)
	- We have **10 bits** to represent all the information we need
		- The first is whether it's a **directory**, the next 3 are the **rwx** bits for the *owner*, then the *group*, and finally *everyone*
	- The user determines these permissions; owner takes precedence, then group, then user
	- The permissions can be shown in a **human-readable format**, where `-` represents a 0, `d` marks a directory, and `r`, `w`, `x` mark read, write, and execute when the bit is 1
	- Permissions can also be written in **octal**, where **r = 4**, **w = 2**, **x = 1**
	- You basically **add** the permissions for each group and then write the three sums side by side as digits

> [!warning] Correction: how precedence actually works (L03: UNIX-Style Permissions)
> The **effective permissions** depend on *which class you fall into*, checked in order: **owner → group → everyone** (not "user"). If you are the owner, you get the **owner bits only**, even if the group or everyone bits are more permissive. The permissions aren't chosen by the person accessing the file; they're set on the file.

> [!warning] Missing: worked examples (L03: UNIX-Style Permissions)
> - `-rwxr-----` → not a directory; owner can **rwx**; group can **read only**; everyone else has **no access**.
> - `750` → owner `4+2+1 = 7` (**rwx**), group `4+0+1 = 5` (**r-x**), everyone `0` (**none**), i.e. `-rwxr-x---`.
>
> The obvious weakness is that this is **coarse-grained**: you can only set permissions for three classes of users. (`setuid`, `setgid`, and the sticky bit exist but are not covered in this course.)
>
> Comparison sources: [L03 lecture text](../lectures/L03.tex) and [L03 slides](../lectures/L03-slides.tex).
