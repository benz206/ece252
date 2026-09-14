## File Systems
- Important and very useful to programs
- It provides both storing data persistently, and also organization through a directory structure while maintaining metadata related to files
- A file is just a logical unit to organize 0 and 1's
- The UNIX approach is to just treat everything as a file, this gives you a very broad array of functions that can be generalized for everything
- Files typically have attributes, which are generally as follows:
	- Name, symbolic human readable form
	- Identifier, the unique identifier (usually a number), this identifies the file inside the file system
	- Type: What the file type is
	- Location: Physical location and what device it is on
	- Size: Current, and possibly max size of the file
	- Protection, who owns, and what r/w permissions
	- Time, Date, User ID: The owner of the file, time of creation, last change, basically other metadata
- Files are maintained in a tree directory like structure, directories though, at the end of the day are also just files, they store information about what files are in what locations and will also be stored on the disk
## File Operations
- The 6 basic operations on files are as follows:
	- Create
	- Read
	- Update
	- Delete
	- Repositioning
	- Truncating
- Example of how to do so is below
```c
FILE *f = fopen(argv[1], "r");
if (f == NULL) {
	printf("unable to open file')
	return -1;
}
readfile(f);
fclose();
```
### Creating a file
Like allocating memory, creating a new file has some essential steps, first ifnd a place to put that file, allocate that space and mark that file as being allocated, and finally put the file in its appropriate directory.
### Writing a file
This requires the name/identifier of the file and the data to be written to the file, the system will then find the file and put in the data, it can either replace or append to the bottom, but there is also a pointer you can use to seek and write at specific parts of the file.
### Reading a file
This requires the name/id of the file and where in the memory the next block of the file should be put. There also exists a pointer which will be required to indicate where the next read will take place
### Truncating a file
If a file should be erased but we want to keep its metadata, we can truncate it, and cut off all of the contents. The file length is then reset to zero and the data area may be marked as free, while the rest of the attributes remain the same

For everything above, it turns out that we all need an open call. all modes are listed in the pdf
### Repositioning a file
Since files can be r/w, but usually only one at a time, the pointer for the write location might be the same as read, as such we might need the `fseek()` call, this will adjust the pointers position for both reading/writing.

This should be done with caution though as you might go to an arbitrary location.

Generally, we only need to seek if we want to skip content.
### Deleting a file
Deletion works as one would expect, find the file, mark its space as free, and then just remove it from the directory listing, this is the equivalent of like a soft delete

These 6 fundamental operations can be used to basically do everything such as copying a file

We can also lock files so that programs that do decide to try and access files are either warned or denied.

Windows uses locking in the way that any programs that are locked cannot be deleted. Unix on the other hand will still let that file be deleted and can be removed from the directory. But once the program finishes it will then be removed properly.

Locking in Linux uses `flock()`
```c
FILE *f = fopen("file.txt", "r");
int file_descriptor = fileno(f);
int result = flock(file_descriptor, LOCK_EX);
// LOCK_UN for unlock, and shared is LOCK_SH
```

## Reading and Writing
Writing to a file is basically the same as using `printf`. In fact, the function call is simply `fprintf` and the only difference between that and `printf` is that the first argument to it is just the `fp` where you want the data to be written too.

```c
void write(point_t* p, FILE *fp) {
	while (p != NULL) {
		fprintf("something %s", p->data);
		p = p->next;
	}
}
```

Reading is a bit more of a pain, you need to use `fscanf` which is like a mirror of `fprintf`.
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

The return value of this function call is the number of elements successfully read, which in this case is supposed to be two. However, it's worth noting that there's no space in the read from the file, because the %d skips leading whitespaces occasionally leading to hard to find bugs.

We could also use getline which is said to be in previous examples but I don't recall reading this.

## File types
The file type of a file can sometimes be determine from the postfix of a filename. However this is just a hint to the operating system as to what program should be opening the file, the actual contents may differ significantly.

Operating systems typically give you an option to select what program will open what filetype.

### Directories
- A directory is just a symbolic table that translates file names ot their directory entries. A directory should support several common operations.
- Common operations are listed below:
	- Search: we want to be able to find a file, and searching is typically not just on the file name but also may include the actual contents of the file if their content is human readable.
	- Add a file to the directory
	- Remove a file from a directory
	- List contents of a directory
	- Rename a file
	- Navigate the file system
- The lecture talks about some very uninteresting other structures but just assume we have the standard tree with subtrees.
- In UNIX, the root directory is just `/` you also do not need to use absolute paths, you can use relative paths and the current pwd can be appended automatically for you
- OS designers have to make a choice about how deletion of directories goes, if a directory isn't empty. Either you delete it all and maybe move it somewhere else like Recycling, or you can just refuse to do it.
- File systems could also support the sharing of files: there is one copy of the file but it has more than one name, in UNIX this is called a link and is essentially a pointer to another file.
	- We either have hardlinks or symlinks
### Symlinks/Hardlinks
Symbolic links are just references by file name, so it will just act as a shortcut, if the file is then deleted it will error out

Hardlinks is just creating a second pointer to the underlying file in the file system. If the hardlink exists and the user deletes that file, the file will remain on the disk until the last hardlink is removed.

We will see later how this actually works but we just maintain a count of hardlinks and then only delete it if it becomes 0.
## File permissions
- To protect users from one another to maintain the confidentiality and integrity of data, files usually have some permissions associated with them, which may control access to the following operations
	- Read, write, execute, append, delete, list
- Unix style permissions are commonly used and is as foll0ws:
	- Each file has an owner and a group, as well as a set of permissions that can be assigned for the owner, the group, and for everyone.
	- There are 3 basic permissions, read write and execute (run as a program)
	- We have 10 bits to represent all the information we need
		- First is whether it's a directory, next 3 are the read write execute bits for owner, then group, and finally for everyone
	- The user determines these permissions, owner takes precedence, then group, then user
	- The permissions can be shown to the screen in a human readable format where - represents a zero and d will be a directory, then also rwx are read write and execute when the bit is 1
	- Permissions can also be written in octal where r = 4, w = 2, x = 1
	- You just basically add the permissions and then literally concatenate them as strings in groups of 3
	