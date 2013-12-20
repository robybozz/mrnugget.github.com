---
layout: post
title: "Watchgopher"
date: 2013-04-07 14:24
comments: true
categories: 
---

In the last couple of months I've been playing a lot with
[Go](http://www.golang.org) and did what I always do when learning a new
language: use it for a small project. That project has been open to the public
on GitHub for a few weeks now and is now in a usable state:
[Watchgopher](http://www.github.com/mrnugget/watchgopher).

Watchgopher allows you to watch certain directories and run custom commands
whenever a file in a specified directory changes. It is supposed to be
simple and give you control of what happens to your files: it only notifies
one of your commands whenever something happens it should know about.

Let me guide you through a simple example to show you what Watchgopher can do.
But first, make sure you have [Go](http://www.golang.org) installed and then run
the following command to install Watchgopher on your system:

```bash
go get -u github.com/mrnugget/watchgopher
```

Now the `watchgopher` command should be available on your system and it's time
to tell Watchgopher which directories to watch and what to do about any file
events occurring in them. So let's create a simple configuration file in JSON
format:

```json
{
  "/Users/mrnugget/Downloads": [
    {"pattern": "*.zip", "run": "/Users/mrnugget/.watchgophers/unzip.rb"}
  ]
}
```

This file tells Watchgopher to watch the `Downloads` directory in my home
directory and whenever something happens to a file whose name matches `*.zip`,
Watchgopher will run the specified command. If the command is in your `$PATH`
you won't need to put the absolute 

When running the command it will pass two arguments to it:

1. The type of the file event. This can be `CREATE`, `MODIFY`, `DELETE`,
   or `RENAME`.
2. The absolute path to the file triggering the event.

After saving the config file we need to create the command which will be run. As
you may have guessed by reading the file names, I want to create a small command
that unzips every newly created zip file in my `Downloads` directory. The code
to do this is pretty simple: 

```ruby
#!/usr/bin/env ruby

exit(0) if ARGV[0] != "CREATE"

success = system("unzip #{ARGV[1]} -d #{File.dirname(ARGV[1])}")
success ? exit(0) : exit(1)
```

This script does three things:

1. Exit with exit code 0 if the file event is not `CREATE`, since I'm not
   interested in `DELETE` events here, because unzipping deleted files is a
   pretty difficult thing to do.
2. Run the `unzip` command with the filename as first argument and the directory
   the file is in as `-d` option, which means it will unzip to the same
   directory the zip file was created in, no matter from where you run Watchgopher.
3. It checks whether the `unzip` command exited successfully. If it did, it
   exits successfully too and if it didn't, it exits with error code 1. (You can
   achieve the same thing with Ruby's `Kernel#exec`, but explaining what exec
   does is not part of this post)

Let's the save the file at the specified place
(`/Users/mrnugget/.watchgophers/unzip.rb`) and give it the right permissions:

```bash
$ chmod +x ~/.watchgophers/unzip.rb
```

Now is the time to launch Watchgopher and point it towards the configuration
file:

```bash
$ watchgopher watchgopher_config.json
2013/04/06 13:50:25 Successfully loaded configuration file. Number of rules: 1
2013/04/06 13:50:25 Watchgopher is now ready process file events
```

Watchgopher now watches the `Downloads` folder for zip files. We should give him
one: 

```bash
$ mv Octocats.zip ~/Downloads/
```

This should prompt Watchgopher to output the following:

```bash
2013/04/06 13:53:23 /Users/mrnugget/.watchgophers/unzip_files.rb, ARGS: [CREATE /Users/mrnugget/tmp/zips/Octocats.zip] -- SUCCESS
```

And if we take a look inside the `Downloads` directory we will find a newly
extracted `Octocat` directory, which means everything went as planned! Great!

As you can see, this concept of handing over the handling of certain file events
over to specified commands gives you a lot of freedom when dealing with file
changes on your system. Since Watchgopher passes the two arguments to every
specified command, the command can decide what to do about it: delete newly
created files, move them to another directory, upload them to a server, or
ignore the file events altogether and just have a look around a directory for
old files and delete them whenever something happens in there. To make it
short: It's up to you how you will react to file changes, Watchgopher just tells
you about them.

There are still a lot more features that I want to implement that are currently
missing: including the output of the specified commands in Watchgopher's log
output, allow to specify the current working directory of a command, allow path
expansion and many more features and tweaks.

I'd be happy if you give Watchgopher a try and I'm also thankful for every
comment, question, issue opened and pull request sent, so don't hesitate!
