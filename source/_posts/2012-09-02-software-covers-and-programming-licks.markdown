---
layout: post
title: "Software Cover Versions And Programming Licks"
date: 2012-09-02 22:43
comments: true
categories: 
---

The next time you're trying to think of something you could put into code and
you shrug of ideas because they've already been implemented, try this: Write a
[cover version](http://en.wikipedia.org/wiki/Cover_(music)) of a piece of
software.

Try to imitate software. A project you like, a project you use, a project that
implemented the idea you just shrugged of. Imagine yourself in a software cover
band and try your best to replicate it.

Why? It's surely a waste of time, I hear you say. And if we're talking about
paid software development I fully agree. But consider yourself having a couple
of hours to spare and you're eager to write some code: covering a piece
of software proves to be a great way to gain knowledge and become a better
programmer.

<!-- more -->

I've been playing guitar for nearly eight years now and spent a great deal of
that time learning and playing other peoples songs. Creatively speaking this is
not as fulfilling as writing your own material and coming up with new, fresh
ideas is certainly better than playing *The Thrill Is Gone* at a bar gig. But by
doing covers you learn new chord sequences, new
[licks](http://en.wikipedia.org/wiki/Lick_(music) 'Licks (Wikipedia)), new
tricks and probably discover something new you haven't thought of yet. And once
you've got a song down and can play it note-perfect, when your muscles remember
how to play a difficult part without struggling, then comes the best part: you
can have fun and play around with it. You can improvise over it, you can change
it, you can disassemble and reassemble it and (this is most important bit) you
can use those little tricks and ideas and incorporate them in your own material.

I'm not talking about copying here, I'm talking about learning solutions to
problems and applying them when the need arises. Learning to play a Led Zeppelin
song on guitar might sound dull since every guitar player on the planet knows
how to play at least one of them. There is nothing left to prove. But there is
still something left to be learned: when you've got the solo of *Since I've Been
Loving You* down to the note, the next time your solo spot comes up and you
want have a intense dynamic build-up, try to think of Jimmy Page and how he'd
do it, which tricks he'd use.

You do this with software and programming too. I've been playing around with
[statsd](https://github.com/etsy/statsd 'statsd') a couple of weeks ago and I
thought about implementing a stats collector in Ruby until I found out that it's
already [been done](https://github.com/noahhl/batsd). So I was about to shrug it
of and do something else when I decided to do it anyway. Just for fun. Just to
find out how to do it.

I wouldn't have learned as much as I did in the last week about buffering,
concurrency, streaming and UNIX sockets if I hadn't tried it. And the best part?
Whenever I got stuck, I read through the *statsd* or the *batsd* code to see how
they solve a certain problem and I learned something new. Surely you learn
something new too when trying to implement a piece of software that hasn't been
written before. But this is different: you already have solutions to problems to
look at and learn from. And when facing the same problem you can learn about the
problem and the solutions at the same time. This is a great way of learning software
development, since you can't fully understand and judge a solution if you don't
fully understand the problem you're facing.

And once you know and understand different solutions to different problems you
can then apply them in other situations. By trying to write software covers you
get to know the problem and how a particular piece of software solved it. Or, as
in my case with *statsd* and *batsd*, you start to understand different
solutions to one problem. Doing this, you can learn a whole lot of new
programming tricks and licks and use them whenever they fit.
