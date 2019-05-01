---
layout: post
date: 2019-04-29
title: DB Connection Pools
tags: [databases, design patterns, object pooling]
---
Surprisingly enough, up to this point in my journey writing software I never
really had the need to think about managing a pool of database connections.
Most of my experience writing web applications has been using frameworks like
Spring Boot which thanks to JPA abstracts away most of the complexity when
dealing with databases — it also adds its own complexity to the game, but
that's a different talk. So I never had to worry about this and in the cases
where the applications I was working on had to manage the connections
themselves I didn't get to touch that part of the codebase.

But then recently I found myself having to think about this in different
occasions. As I did not know what were the best practices when working with
this kind of code, I had to search on the web for a while and felt like
blogging what I found.

### Database Connections Are Expensive:

Which essentially means that establishing a connection is something that makes
the computer wait a little, usually more than regular operations which depend
exclusively on data that is local to the process: RAM and disk. Database
connections are io-bound operations, like a disk read. However, unlike
regular disk reads, database connections are established over a wire and
depending on the authentication mechanism being used[^1] may require multiple
TCP transactions in order to complete.

Knowing that, we can conclude that we do not want to spawn new connections
whenever the need to query the database arrises. This is specially true for a
web application where multiple queries can be triggered by each HTTP request
hitting the service. Since now every request is a bit slower, the system's
overall performance tends to degrade over time as requests are allocated per
thread which is a finite resource: the more time a request takes, the more it
takes to free a thread for a new request. The more requests are kept waiting,
the more memory the application consumes and so forth.

To solve this problem we could spawn a new connection during application
startup and pass it to be used by every database query triggered by the
application. That solves the request time issue but is still not ideal, let's
see why.

### Database Connections Are Stateful:

When connecting to a host on the Internet one the first things we need know
is whether that connection is still up. Connections can bork due to a varied
number of reasons like: you lost access to the Internet, the other host went
offline, the operating system decided to free up some resources and your
connection happened to be one of them etc. So, attached to the thing that
represents your connection goes some way of telling clients whether that
connection is busy, closed, open etc. We can assume that connections carry
with them a small blob representing their state.

So now that we know that, why is it that sharing connections between parts of
the system a bad thing? Well, because we all know that **shared global state
is bad**™️. Any of the parts using the connection at a given moment can
alter the state of the connection in a way that affects the other parts
causing the entire system to fail. Also, if for a reason external to the
system that connection borks, the entire system gets affected because
multiple parts of it use the same connection to everything.

So then, how can we reuse connections — to avoid creating new ones, an
expensive operation — whithout taking the risks associated to shared state? I
guess you already know where this is leading.

### Be Cool Jump in the Pool

A connection pool is a way of having a reserve of connections — all of them
already set up during application startup — so they can be reused accross the
system without any two parts sharing the same resource. Yeah cool, but how
that works? The answer is: there are multiple distinct implementations.
Though for the sake of comprehending the concept let's attain ourselves to a
very simple one: a **queued connection pool**.

Let's start with a nice diagram that can summarise pretty much this entire
post:

![](/assets/connection-pooling.jpg){: .center-image }
*Clients connecting to a data source through a connection pool.*[^2]

As we can see, we have a couple connections sitting in the pool that start to
be allocated to individual clients as they request them, pretty simple.
Notice how each client gets its own connection without sharing it with anyone
else.

What this picture does not illustrate is what happens when a new client
arrives and all the available connections have alredy been taken. For a
queued connection pool with a fixed amount of connections, the new clients
will just have to wait until there's a free connection to take. That's
basically it. End of story.

This might not be ideal for applications that are highly concurrent and
demand many connections to be available. In those scenarios we could increase
the fixed amount of connections, implement a priority queue in front of the
pool so important clients have to wait less or allocate new connections
whenever the pool has been exhausted. The right implementation is going to
depend on the use case. For the applications I've been working with a simple
queued pool works just fine.

### Further Reading

It turns out this design pattern can be applied to a variety of resources,
not only database connections. Check this nice Wikipedia article for more
details on object pooling[^3].

[^1]: [PostgreSQL Backend/Frontend protocol.](https://www.postgresql.org/docs/9.3/protocol-overview.html)
[^2]: [Using JDBC to Connect to a Database.](https://ejbvn.wordpress.com/category/week-2-entity-beans-and-message-driven-beans/day-09-using-jdbc-to-connect-to-a-database/)
[^3]: [Object Pooling.](https://en.wikipedia.org/wiki/Object_pool_pattern)