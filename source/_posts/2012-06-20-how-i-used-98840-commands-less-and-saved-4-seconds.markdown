---
layout: post
title: "How I used 98840 commands less and saved 4 seconds"
date: 2012-06-20 18:58
comments: true
categories: 
---

In my last post I explained how to implement a search autocompletion backend
using Redis. This week I used the described implementation with its `add_movie` method to put thousands of movies into my Redis database
in order to use them for the autocompletion on
[anygood.heroku.com](http://anygood.heroku.com "AnyGood - Check if a movie was
any good!"). It's the same method as described in the last post, but with
important change I didn't think about too much: I did not overwrite the score of the members in the sorted
sets. The method I used for adding movies looked like this: 

```ruby
def add_movie(movie_hash)
  prefixes    = prefixes_for(movie_hash[:name])
  hashed_name = data_hash_key_for(movie_hash)

  prefixes.each do |prefix|
    score = REDIS.zscore(index_key_for(prefix), hashed_name).to_i || 0
    REDIS.zadd(index_key_for(prefix), score, hashed_name)
  end

    REDIS.hset(data_key, hashed_name, movie_hash.to_json)
end
```

And adding movies took suspiciously long, longer than I thought it would and
looking at it now it's pretty easy to tell why: it was exactly that small change
that lowered the speed of my importing process, I *did* cache the member score
in *every* prefix set and then used in the `REDIS.zadd` line. Every time a movie
was added, for each of its prefixes the score of the correspondent sorted set
member was saved and used again. That's not bad at all, no, it didn't slip in
there either! Heck, I wrote tests for this. But looking at the code I concluded
that it was not a feature I needed. Using the code above, every movie would have
a different rank in the autocomplete output according to the user input. One
movie could be ranked higher when the user typed in *the* and ranked lower when
the input was *dark*. That is pretty cool, but I wanted a different behaviour:
the sorted set members associated with one movie should have one score, the same
one in every set. And after running some benchmarks I found out how implementing
the wrong behaviour slowed me down.
<!-- more -->

Testing this was simple: I wrote a small script that allowed me to add ten
thousand movies with the aforementioned method. The rest is just a shell and
`time`. The script looks like this: 

```ruby
#!/usr/bin/env ruby -wKU
require 'redis'
require 'digest'

REDIS = Redis.new

# We need to generate the prefixes for every moviename
def prefixes_for(string)
  prefixes = []
  words    = string.downcase.split(' ')

  words.each do |word|
    (1..word.length).each {|i| prefixes << word[0...i] unless i == 1}
  end

  prefixes
end

# Adding 10000 movies
(1..10000).each do |i|
  movie_name  = "The Number#{i}"
  hashed_name = Digest::MD5.hexdigest(movie_name)
  prefixes    = prefixes_for(movie_name)

  prefixes.each do |prefix|
    score = REDIS.zscore("testing:redis:index:#{prefix}", movie_name).to_i || 0
    REDIS.zadd("testing:redis:index:#{prefix}", score, movie_name)
  end

  REDIS.hset("testing:redis:data:Number#{i}", movie_name, "This is Number #{i}")
end
```
It's a good idea to delete all keys in our instance (using `FLUSHALL`) before
running this script and monitoring is especially and always great: Redis' handy
`MONITOR` command outputs all the commands our instance receives, complete
with timestamp. So, monitoring our instance in the background, its time to see
how often `ZSCORE` gets called when we add those ten thousand movies:

```bash
$ redis-cli FLUSHALL
OK
$ redis-cli MONITOR | grep zscore > zscore_for_every_prefix.txt &
[1] 94937 94938
$ ruby insert_10000_movies.rb
$ fg
[1]  + 94937 running    redis-cli MONITOR | 
       94938 running    grep zscore > zscore_for_every_prefix.txt
^C
$ wc -l zscore_for_every_prefix.txt
  108840 zscore_for_every_prefix.txt
```

## Use less commands!

That is a huge number. But don't you worry, lowering it is pretty easy. Just
change a couple of lines of `insert_10000_movies.rb` to get the behaviour we
want and it looks like this:

```ruby
# [...]
score = REDIS.zscore("testing:redis:index:#{prefixes.first}", movie_name).to_i || 0
prefixes.each do |prefix|
  REDIS.zadd("testing:redis:index:#{prefix}", score, movie_name)
end
# [...]
```

With this in place, `ZSCORE` will get called exactly one time for each movie and
not for every prefix of every movie. Instead of using `ZSCORE` 108840 times,
after that small change it only gets called ten thousand times.  And 108840
minus 10000 is *98840*. That's *98840 commands* less!

But let's not stop here. On my MacBook Air it took roughly *7* seconds to insert
those ten thousand movies using `ZSCORE` on every prefix. With *98840*
commands less it only takes around *4*: 

```bash
$ redis-cli FLUSHALL
OK
$ time ruby insert_10000_movies.rb
ruby insert_10000_movies.rb  7.39s user 3.78s system 54% cpu 20.448 total
$ redis-cli FLUSHALL
OK
$ time ruby insert_10000_with_one_prefix_score.rb
ruby insert_10000_movies_with_one_prefix.rb  4.42s user 2.19s system 54% cpu 12.112 total
```

## Redis and the Pipeline

I wasn't satisfied with this result and while examining the code again I
remembered [Redis pipelining abilities.](http://www.redis.io/topics/pipelining "Redis pipelining documentation").

When sending requests to Redis using pipelining the server doesn't wait
for the client (our code) to progress its responses, it just accepts them and
does as it's told. And the client doesn't read any responses, it just fires
the next request until all commands in the pipeline are sent.
[redis-rb](https://github.com/redis/redis-rb "redis-rb") fully supports
pipelining to Redis and using it is pretty straightforward: Just put the part of your code
that's sending requests to Redis into a `REDIS.pipelined` block and everything
in the block will be going through the pipeline. That should save us some
seconds, right? Time to make a use of it:

```ruby
# [...]
score = REDIS.zscore("testing:redis:index:#{prefixes.first}", movie_name).to_i || 0

REDIS.pipelined do
  prefixes.each do |prefix|
    REDIS.zadd("testing:redis:index:#{prefix}", score, movie_name)
  end

  REDIS.hset("testing:redis:data:Number#{i}", movie_name, "This is Number #{i}")
end
# [...]
```

What we're doing here is precisely what I just described: for each movie we open
a pipeline to Redis and send all the `ZADD` and finally the `HSET` command
straight to Redis without blinking an eye. The response won't get processed
until the block is finished. Doing that our script runs noticeably
faster. But since we all love benchmarking, hard numbers and speed, let's see
how fast exactly: 

```bash
$ redis-cli FLUSHALL
OK
$ time ruby insert_10000_movies_pipelined.rb
ruby insert_10000_movies_optimized.rb  3.42s user 1.26s system 86% cpu 5.414 total
```

*4* seconds less! That is more than 50% faster! Wow! This is great, this is
magnificient! Let's pipeline all the commands!

## Not so fast, buddy!

Woah, easy, buddy, hold it right there! Let me tell you something: It would be
great if we could put all our Redis interactions within a `REDIS.pipeline`
block, but there is one small problem. As I said, the server doesn't wait for
the client to process its responses and the client waits until the pipeline is
empty. That means you can't pipeline blocks of code in which you work
with the servers response. The code at the beginning of this article did use
the server's responses, remember? We used `ZSCORE` for every prefix to get the score
of the member in that particular set, saved it and then updated the member of the
set with exactly that score. That's not possible when sending our orders through
the pipeline.

I guess an example should clear things up so let's add a couple of movies with
the following code, resulting in every member of every index set having a score
of 10:

```ruby
REDIS.pipelined do
  prefixes.each do |prefix|
    REDIS.zadd("testing:redis:index:#{prefix}", 10, movie_name)
  end
end
```
Using `redis-cli MONITOR` in a parallel shell session, we can see what's being
sent to Redis:

```
[...]
1341173848.796741 "zadd" "testing:redis:index:number1000" "10" "The Number10000"
1341173848.796772 "zadd" "testing:redis:index:number10000" "10" "The Number10000"
1341173848.796923 "hset" "testing:redis:data:Number10000" "The Number10000" "This is Number 10000"
[...]
```

That looks fine. Now, if we were to add those movies again (with the keys still in our Redis
instance), and try to cache the scores inside a `pipeline` block, it won't
work, since the client doesn't process the servers response until after the
pipelining block is finished. The following change in the code will show when
the response is there and when it isn't:

```ruby
scores = []
scores << REDIS.pipelined do
  prefixes.each do |prefix|
    scores << REDIS.zscore("testing:redis:index:#{prefix}", movie_name)
    puts scores.inspect
  end
end
```

This will output the following for each movie: 

```ruby
[nil]
[nil, nil]
[nil, nil, nil]
[nil, nil, nil, nil]
[nil, nil, nil, nil, nil]
[nil, nil, nil, nil, nil, nil]
[nil, nil, nil, nil, nil, nil, nil]
[nil, nil, nil, nil, nil, nil, nil, nil]
[nil, nil, nil, nil, nil, nil, nil, nil, ["10", "10", "10", "10", "10", "10", "10", "10"]]
```

The servers responses are only there (the last element in the `scores` array)
after the block is finished, inside the block the `scores` array is filled with
`nil`s: no response has been read yet. Using nearly the same code but
*without pipelining* we get totally different results:

```ruby
scores = []
prefixes.each do |prefix|
  scores << REDIS.zscore("testing:redis:index:#{prefix}", movie_name)
  puts scores.inspect
end
puts scores.inspect
```

```ruby
["10"]
["10", "10"]
["10", "10", "10"]
["10", "10", "10", "10"]
["10", "10", "10", "10", "10"]
["10", "10", "10", "10", "10", "10"]
["10", "10", "10", "10", "10", "10", "10"]
["10", "10", "10", "10", "10", "10", "10", "10"]
["10", "10", "10", "10", "10", "10", "10", "10"]
```

We can see now that pipelining is a great way to gain speed when progressing huge lists
of commands, which is exactly what I was after when adding thousands and
thousands of movies, but when you need to work with the servers response, say
when getting a value and using that value in another request, the value just
won't be there inside a pipelining block, only after it finished executing.
Pipelining is a great thing to know, but only when the requirements are right
and it's applicable.

That means I used 98840 commands less by adjusting the code for its use case and
saved 4 seconds using the right tool for the job, which is always great, isn't
it?

