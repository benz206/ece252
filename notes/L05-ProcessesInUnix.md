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
