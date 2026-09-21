## The Process in UNIX
Earlier on, we mentioned that in UNIX, a process can create other processes. The creating process is the parent and the newly-created processes are its children. Every process again has a parent.

Each process has a unique id which we call the `pid` or the process ID, for the most part we don't care unless we're running a kill command.
![[Pasted image 20260918091834.png]]
In a unix system, we can obtain a list of processes at any time with the `ps` command. The diagram shows a basic hierarchy of how this might be setup.
When you issue a command, like ls or top (table of processes), the new process is created and the shell will `wait` on that process to finish.
When control does go back, you will be prompted to again run more commands, this would, seem kind of limiting, do I have to login to a system in a second terminal window to urn 2 things at a time? Not really, we can get around them.

To do this using `gcc` we can just use the ampersand operator to signify that this should run in the background. Notably any console output will still be outputted even after control is returned. So instead maybe you want this to be put in some file, you can use `cat fork.c > logfile.txt`.

> A common example of a command I use involving the &:
sudo service xyz start &
This will (with super user permissions - that’s the purpose of sudo) start up the service xyz but return control
to the console so I don’t have to wait for the xyz service to be started to enter my next command. This is good,
because the next thing I’d like to do is tail -f /var/log/xyz/console.log which will allow me to watch the
console log of the xyz service as it starts up to see if there are any errors.
The other alternative to get something to run in the background is with the screen command. While having
something run in the background is nice, it does not work for interactive processes. Suppose you are working on
some code in vi and you would like to pause that for a minute and write an e-mail (with pine or whatever the
cool kids use for command line e-mail these days). One approach is to save and exit vi and open up pine. The
other would be to start up each of these in screen and switch between them.
Thus instead of just opening vi fork.c I can issue the command screen vi fork.c and this spawns screen
and takes me right to editing the file. The key difference is that I can “detach” from this screen and go back to
the command line that spawned it. And if I log out, screen keeps running with the vi inside it. If I have multiple
screens running, I can just “reattach” to the one I want to use next. To get a full understanding of screen, try
the command man screen and the user manual will appear to give you some information and instructions about
how to use this. Or you can use Google

Can format this later but I just assume its the same as like tmux.

## Show me the Code!
Basically a parent can fork itself to create a child process, in which we can then use the system call `wait` to wait for a process to complete.
```c
#include <sys/types.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char** argv) {
	pid_t pid;
	int child_status;
	
	pid = fork();
	
	if (pid < 0) {
		// Some error occured
		printf(stderr, "Fork_failed");
		return 1;
	} else if (pid == 0) {
		// child process
		execlp("/bin/ls", "ls", NULL);
	} else {
		// parent process that will wait for the child
		// to complete
		wait(&child_status);
		printf("child complete with status word %i\n", child_status);
	}
	
	return 0;
}
```

We just get a simple output of `fork    fork.c Child Complete with status: 0`

Or, to represent this visually:
![[Pasted image 20260918093436.png]]
What about termination though? On the assumption that the process is terminating normally and not being killed, the system call for that is just exit. If a program has no explicit call to exit, the return statement at the end of main will have the same effect.
- **Use of the Fork Design Problem**
	- It's not necessary for a child to replace itself with another one.
	- What if we want to make a program where both parts are part of the same source file.
	- We can use the `fork()` function to create a child process. In this example, the child should call `execute_B()` and return the result to the parent, while `execute_A()` should be called by the parent.
```c
pid_t pid;
int child_result;
int parent_result;

pid = fork();

if (pid < 0) {
	// Fork failed
	return -1;
} else if (pid == 0) {
	return execute_B();
} else {
	parent_result = execute_A();
	wait(&child_result);
}

if (child_result == 0 && parent_result == 0) {
	// completed
	return 0;
}

if (child_result != 0) {
	printf("Error %d Occurred.\n", WEXITSTATUS(child_result));
}

if ( parent_result != 0 ) {
	printf( "Error %d Occurred.\n", parent_result);
}
return -1;
```

## The fork bomb
- A simple example of how fork can be used maliciously is just have an infinite loop calling fork
- Easily defendable by just limiting/killing processes over a limit

## Signals
UNIX systems use signals to indicate events
- A signal is synchronous if the signal can be attributed to a singular line of code
- A signal is asynchronous if the signal is from some outside process e.g. ctrl-c or one process/thread sending a signal to another.
![[Pasted image 20260921083943.png]]

Alternatively, a process could inform the OS it is prepared to handle the signal itself. In any event the signal eventually needs to be handled, even if the handling is to just ignore it. Note that the signals need to be handled, even if its just to simply ignore it.

Signals `SIGKILL` and `SIGSTOP` cannot be blocked, caught or ignored.

On the command line, the command to send a signal is also just kill and the pid as you know.

Using a flag of `-9` will send SIGKILL instead of SIGTERM

