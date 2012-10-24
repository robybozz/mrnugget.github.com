---
layout: post
title: "Command Line Ride"
date: 2012-10-24 09:38
comments: true
categories: 
---

When your harddrive is running out of space, chances are good that there are
files you can safely delete in order to free up some of that space.
A safe bet are archive files, like ZIP and RAR-files, you've already
decompressed.

I went on a little ride to put together a command that shows me how much space a
certain type of file take up. There first thing I did was `find`ing those
files...
<!-- more -->

```bash
find / -type f -name '*.rar'
```

What does `find` do here? It looks up all the files on the harddrive that are
really files (`-type f`) and not directories and whose names end in `.rar`. That
got me pretty far. But still not far enough: sometimes RAR-archives are splitted
up into several files called `archive.rar archive.r01 archive.r02` and so on.
They should be listed as well:

```bash
find / -type f -regex '.*r[0-9][0-9]' -o -name "*.rar"
```
That's better! Here `find` lists the filenames matching the provided regex or
(`-o`) those ending in `.rar`. Running this command in a directory containing
such an array of RAR-files outputs the following:

```bash
$ find . -type f -regex '.*r[0-9][0-9]' -o -name "*.rar"
./big_archive.r01
./big_archive.r02
./big_archive.r03
./big_archive.r04
./big_archive.r05
./big_archive.rar
```

But I still didn't know how big those files are. So the first thought was: use
`ls -l` on every file since that gives me the size of every file. But `ls` takes
filenames as command line arguments and doesn't read from the standard input
stream. So I couldn't just pipe the list to `ls`, since a Unix pipe connects the
standard output stream of one program to the standard input of another program.
Just try it:

```bash
find . -name '*.txt' | ls -l
```

That shouldn't give you the desired output. What happens here is that `ls`
doesn't get an argument and lists the contents of the current directory. So how
does one call `ls -l` on every file in the list above? Pipe the list of files to
`xargs`. `xargs`'s job is to construct argument lists for the provided command.
It does so by splitting up the data it receives on the standard input and using
each chunk as an argument. By default `xargs` splits the incoming data by
newlines or blanks, which is normally fine but could lead to problems when
`find` outputs a filename containing whitespaces. In that case, be sure to use
`man find` and `man xargs`: You can specify a delimiter other than blank or
newline. So far the output should look like this:

```bash
$ find . -type f -regex '.*r[0-9][0-9]' -o -name "*.rar" | xargs ls -l
-rw-r--r--  1 mrnugget  staff  1024000 Oct 18 13:58 ./big_archive.r01
-rw-r--r--  1 mrnugget  staff  1024000 Oct 18 13:58 ./big_archive.r02
-rw-r--r--  1 mrnugget  staff  1024000 Oct 18 13:58 ./big_archive.r03
-rw-r--r--  1 mrnugget  staff  1024000 Oct 18 13:58 ./big_archive.r04
-rw-r--r--  1 mrnugget  staff  1024000 Oct 18 13:58 ./big_archive.r05
-rw-r--r--  1 mrnugget  staff  1024000 Oct 18 13:57 ./big_archive.rar
```

Great, I thought, now I just need to get all the different filesizes, add them together
and print the total sum! Shouldn't be too hard, right? Well, it isn't if you got
`awk`. `awk` has too numerous capabilities to explain in this blog post. So
let me make it short: `awk` read its input from either the STDIN or from files
passed in as arguments and then performs actions on matching lines. To make it
even shorter: `awk` is awesome. There is a lot of free information available on
the internet about `awk`, but a single `man awk` goes a long way.

Using `awk` to output the sum of the filesizes looks like this:

```bash
$ find . -type f -regex '.*r[0-9][0-9]' -o -name "*.rar" | xargs ls -l | awk '{sum = sum + $5} END {print sum}'
6144000
```

Here `awk` takes the fifth field (the fields are by default separated by blanks)
and increments the variable `sum` by it. At the end of the `awk`-program (after
`awk` ran it over each line) it prints out the sum, which gives us the sum of
the filesizes. But that's not really readable since the output of `ls -l`
contains filesizes in bytes and I think it's safe to say that megabytes would be
far more handy in this case. So I had to divide the sum by 1024 to get
kilobytes and then again by 1024 to get megabytes and I did this with the help
of `xargs` and `bc`:

```bash
$ find . -type f -regex '.*r[0-9][0-9]' -o -name "*.rar" | xargs ls -l | awk '{sum = sum + $5} END {print sum}' | xargs -I sum echo sum/1024/1024 | bc -l
5.85937500000000000000
```

That looks great! So what happens here? `xargs` uses a name for the data it
reads from standard input, `sum`, and then the `echo` command to output the
calculation that needs to be fed to `bc`. Without the `bc` the command above
would just output `6144000/1024/1024`. `bc` then takes this as input and gives
us the result. Be sure to do this: `man bc`. This example here doesn't even
scratch the surface of what `bc` is capable of.

So now the job is done, right? The command line above now outputs the total size
of all the RAR-files on the harddrive or in the current directory. Well,
technically yes, it's done. But as you can see, that was pretty heavy lifting,
nobody will remember that command above and when first looking at it nobody will
know what it exactly does.

And here's the kicker: it's useless. That command above is obsolete. As soon as
I finished hacking up that command line I remembered a tool that I basically use
every day but totally forgot about while hacking together the right `find`-regex,
looking up `awk` formulas and how `bc` works. There is one tool that does
exactly what that long line above does and it's called `du`. `du` is built for
the job. It's a simple tool that does one thing very well (and I quote the man
page here) and that is to "display disk usage statistics". I can't for the life
of me explain how I forgot it. With `du` in hand, the line shrinks down to this:

```bash
$ find / -type f -regex '.*r[0-9][0-9]' -o -name "*.rar" | xargs du -ch
1000K ./example_dir/big_archive.r01
1000K ./example_dir/big_archive.r02
1000K ./example_dir/big_archive.r03
1000K ./example_dir/big_archive.r04
1000K ./example_dir/big_archive.r05
1000K ./example_dir/big_archive.rar
5.9M  total
```

That looks a lot better than that monster I hacked together. And it's easier to
understand too: 1) find all the files matching a certain pattern 2) then pass
them to `du` to display how much space they take up. After remembering `du` I
thought: "Well, maybe I can shrink this down even further". And I did.

When you use `find` and the `-regex` option you're in for a challenge. `find`
allows you to use [many different types of Regular Expressions](http://www.gnu.org/software/findutils/manual/html_mono/find.html#Regular-Expressions 'Find & Regex')
and the differences sometimes make it really difficult and frustrating to get
the regex you want to work. Just have a look [here](http://www.greenend.org.uk/rjk/tech/regexp.html).

Most of the time it's probably easier to use the globbing functionality of
your shell rather than using `find` and regex. Especially ZSH is [pretty good with globbing](http://grml.org/zsh/zsh-lovers.html)
 and bash also does its job very well. Since I'm using ZSH I tried to get rid of `find` and use my shell's
built-in globbing functionality.  And what I came up with is so much better than
that long line above:

```bash
$ ls **/*.r(ar|<-99>) | xargs du -ch
1000K example_dir/big_archive.r01
1000K example_dir/big_archive.r02
1000K example_dir/big_archive.r03
1000K example_dir/big_archive.r04
1000K example_dir/big_archive.r05
1000K example_dir/big_archive.rar
5.9M  total
```

That line recursively lists all files ending in either `rar` or `r01` up to
`r99` and then passes them over to `du`. Easy to read, easy to understand
and more importantly: easy to reuse.

There is a lot I learned on this ride and most importantly it was seeing the
[Unix philosophy](http://www.catb.org/~esr/writings/taoup/html/ch01s06.html) in action:

> "This is the Unix philosophy: Write programs that do one thing and do it well.
> Write programs to work together. Write programs to handle text streams,
> because that is a universal interface." - Doug McIlroy

All of those programs work very well together, they are combinable, they are
reusable and they do one thing well. And in this case `du` was that program that
did the one thing I wanted to achieve very well and could be used to replace
another complex "program", if you want to call that line above a program.

That is not to say that complex command lines are always wrong to use, no. Sometimes
you need that many programs to work together in order to get the desired output.
And when that happens it's great to see how good every tool is at doing its own
job and which problems can be solved by combining them. And Unix pipes make it
dead easy to combine them by offering a clean and easy to understand inteface.

Seeing that philosophy in action shows extremely well how much code and programs
can profit from complying with it.
