---
layout: blog
title: RediSQL, Road to ~50K inserts per seconds.
---


[RediSQL][1] is a redis module that embed SQLite.

The module lets you do things like the following:

```
127.0.0.1:6379> REDISQL.CREATE_DB DB                                                
OK
127.0.0.1:6379> REDISQL.EXEC A "create table 
                    user(username text, password text, counter int);"
DONE
127.0.0.1:6379> REDISQL.EXEC A "insert into user 
                    values('foo', 'secret', 3);"
DONE
127.0.0.1:6379> REDISQL.EXEC A "insert into user 
                    values('bar', 'verysecret', 5);"
DONE
127.0.0.1:6379> REDISQL.EXEC A "select * from user where counter > 4;"
1) 1) "bar"
   2) "verysecret"
   3) (integer) 5
127.0.0.1:6379> REDISQL.EXEC A "UPDATE user 
        SET counter = counter + 1 WHERE username = 'foo'; "                        
DONE
127.0.0.1:6379> REDISQL.EXEC A "SELECT * from user;"
1) 1) "foo"
   2) "secret"
   3) (integer) 4
2) 1) "bar"
   2) "verysecret"
   3) (integer) 5
```
To bring a little more order in the Redis world.

When I start to write the module I was very naive.
I wasn't familiar with neither API, nor the Redis one nor the SQLite one.

After I understood how every piece fit together I started to be a little more sophisticated and I started to look for performances.

## Establish Baselines

When looking for performance fundamental to measure everything and to compare, the first thing I did was then to understand what numbers I should expect.

A Redis serve can reply to roughly ~70K requests per second.
In memory SQLite can handle roughly ~50K "simple inserts" per second. Caching the prepared statement this number raise to ~200K "simple inserts" per seconds.

Where I have define a "simple insert" as inserting a tuple of three integers into the database.

Now I know, at least the magnitude, that I should expect from my module.

## Blocked Client

When Redis need to execute a long command it "blocks" the clients.
The connection with the client is kept alive, the main Redis thread is used to receive and complete other requests while the work is done in the background.
When the background works is finished the client is unblocked and a reply is send.

Implementing blocked clients was a necessity that brought two benefits:

1. The main Redis thread is free to serve other request, either other RediSQL commands or other simple Redis commands.
2. A new thread can be allocated to serve exclusively the necessity of SQLite

## A thread for request

The first implementation of Blocked Client wasn't much efficient, it spawned a new thread for every and each request.

Of course the benchmark weren't good, it reached maybe 1000 inserts per seconds.

## Introducing Thread Pools

The second implementation was less naive, it used a Thread Pool.
`pthread`s were spawned and every request was served from the first free thread in the pool.

The performances raised a little to ~1600 inserts per seconds.

Spawning new thread wasn't the main bottleneck.

From my investigation with perf I believe that most of the time was spent in context switch and in SQLite internals locks.

## A separated single thread

Given the insight from the previous investigation I decide to use a single thread.

Now when a new database is created using the command `REDISQL.CREATE_DB` a new thread is spawned.

That thread will be the only responsible for the database and it will listen a simple channel.

When a new command arrive the flow is the following:
1. The client is blocked, so the main redis thread is free to serve other request
2. The SQL is send to the database thread
3. The database thread execute the statement
4. The client is unblocked and a reply is send back

This approach is the one yielding the best performance so far.

Overall I finally reached the 50K simple inserts per second using ~100% of one processor.

If you are curious about more details I encourage you to open an issues in the repo.

Similarly you are free to replicate the benchmarks yourself and share your results or uncertainty.

[RediSQL source code is hosted on github :)][1]


[1]: https://github.com/RedBeardLab/rediSQL
