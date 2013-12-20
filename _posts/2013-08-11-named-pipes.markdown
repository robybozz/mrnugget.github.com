---
layout: post
title: "Named Pipes"
date: 2013-08-11 16:41
comments: true
categories: 
---

A lot of people know and love Unix pipes, myself included, since they let you do
stuff like this: `cat access.log | awk '{print $9}' | sort | uniq -c`. What a
lot of people don't know are [Named Pipes](http://en.wikipedia.org/wiki/Named_pipe),
which are pretty interesting and worth knowing about.

In contrast to "unnamed pipes" (`|`) a named pipe has a file name on your file
system and can be accessed by independent processes that were not spawned by the
same parent process. To create a named pipe use [mkfifo(1)](http://linux.die.net/man/1/mkfifo):

```bash
$ mkfifo my_named_pipe
$ ls -l
total 0
prw-r--r--  1 mrnugget  staff  0 Aug 11 16:59 my_named_pipe
```

The `p` in the left column of the `ls -l` output indicates that `my_named_pipe`
is a pipe. You can change the permission bits as with any other file.

Using it is straightforward, just write something to it in one process and read
from it in another. In the first shell:

```bash
$ cat access.log | awk '{print $9}' > my_named_pipe
```

As you can see, this will block until another process reads from the pipe. So
open up another shell and read from it:

```bash
$ sort my_named_pipe | uniq -c
```

The first process will exit when it's done writing and sends EOF. The second
process stops when it sees the EOF and exits. 

Even if you have never used a named pipe that's been created with `mkfifo(1)`,
you might have used one without knowing about it, since shells (at least Bash
and ZSH) use named pipes whenever they encounter command substitution:

```bash
$ diff <(ls ./old/) <(ls ./new/)
```

The shell will spawn two subshells here, running `ls ./old/` and `ls ./new/`
respectively, redirecting their output to two named pipe it creates and names.
It then passes the name of the pipes to `diff(1)`, which expects filenames as
arguments.

## Communication Between Processes

What else can we do with named pipes? Since they can be read from and written to
by independent processes, we can use them for communicating between them. Of
course, we can write to and read text from them, yes. But since writing or
reading from the named pipes will block until the other end is doing
something, we can use that behaviour for means of communication.

Imagine one process waiting for another process to finish. Set up a named pipe
and have it read from it:

```bash
$ mkfifo /tmp/finished
$ cat /tmp/finished && echo "The other process is finished! Yay! Let's do some work!"
```

Now whenever you want to signal this process that it can stop waiting, just
write something to the pipe:

```bash
$ ./doing_a_lot_of_work_here && (echo 'I am done here.' > /tmp/finished)
```

This reminds me of using dedicated quit channels in Golang to signal other
goroutines when to quit. The cool thing about this is that multiple
processes can wait on one process. Or multiple processes can write to the
named pipe and just one is reading.

## Sharing Terminal Output With `script(1)`, netcat And Named Pipes

Let's use a named pipe to do a "terminal screen sharing" session, by using
`script(1)`. `script(1)` allows us to record a shell session by writing a
typescript to a file. We will use a named pipe instead of a file. So open up the
first shell and type in the following:

```bash
$ mkfifo screenshare
$ script -t 0 screenshare
```

Again, this will block. So let's read from the pipe in another shell:

```bash
$ cat screenshare
```

The first shell shouldn't block anymore and open a new `script(1)` session.
Switch to it and type something in, e.g. `ls -l`. You should now see your first
shell session mirrored in the second one, by the power of `script(1)` and
`mkfifo(1)`.

Let's take it one step further. Let's use netcat (`nc(1)`) to stream the shell
session over the network! The first thing we need is a listener on one computer:

```bash
$ nc -l -p 9999
```

This computer will now wait for connections and data on port 9999. Now we need
a named pipe and `script(1)` in one shell session just like before:

```bash
$ mkfifo screenshare
$ script -t 0 screenshare
```

But instead of using `cat(1)`, we use `nc(1)` to send the typescript to our
listener in another shell:

```bash
$ nc <ip-of-listener> 9999 < screenshare
```

If you now start working in the shell that ran `script(1)` a typescript will be
written to the named pipe, which netcat will read from to send it the listener
computer. Of course, this is totally insecure, but it's really, really cool
nonetheless, right?

Another cool more practical thing to do is using named pipes to asynchronously
run tests, by writing test commands in on one end and running those commands on
the other end. Gary Bernhardt demonstrated that in a great
[screencast](https://www.destroyallsoftware.com/screencasts/catalog/running-tests-asynchronously).

I have to admit, learning about named pipes is by no means a world changing
event and experience. Mostly because an "unnamed pipe" is more often than not
the better and easier way to go. Still, I think it's useful to know about and to
understand them. Having named pipes in your tool belt is certainly not something
you will regret.

## Resources

* [Linuxjournal: Introduction to named pipes](http://www.linuxjournal.com/article/2156?page=0,1)
* [Wikipedia: Named Pipe](http://en.wikipedia.org/wiki/Named_pipe)
* [Unix & Linux Stackexchange - Process Subtitution and Pipe](http://unix.stackexchange.com/questions/17107/process-substitution-and-pipe)
* [About.com - Process Substitution And Named Pipes](http://linux.about.com/od/Bash_Scripting_Solutions/a/Process-Substitution-And-Named-Pipes.htm)
* [The Linux Documentation Project - Named Pipes](http://www.tldp.org/LDP/lpg/node15.html)
