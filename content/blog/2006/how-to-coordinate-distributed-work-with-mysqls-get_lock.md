---
title: "How to coordinate distributed work with MySQL's GET_LOCK"
date: "2006-07-26"
url: /blog/2006/07/26/how-to-coordinate-distributed-work-with-mysqls-get_lock/
categories:
  - Databases
---
This article explains how I replaced file-based methods to ensure only one running instance of a program with MySQL's `GET_LOCK` function. The result is mutual exclusivity that works in a distributed environment, and it's dead simple to implement.

My current employer used to use a technique similar to the classic [Perl 'Highlander' method](http://www.stonehenge.com/merlyn/WebTechniques/col54.html) to ensure only one instance of a certain program would run at any given time. The method was to create a file with a certain name and then get an exclusive, non-blocking lock on the file with the `flock()` call. If another program tried to get the same lock, it would find the file already locked and exit with a message about "another running instance found."

This type of locking can be pretty tricky to get right. There are always lots of edge cases to consider. For example, what if the same program tries to get the lock twice and blocks itself?

Trying to roll your own locking functionality is asking for trouble, unless you really know what you're doing. It's not enough to just create some "sentinel" that indicates "something else is running." It must be done atomically and in a non-blocking fashion---already tough to hand-roll---and there are lots of other requirements, such as "the lock must be released if the program dies without having a chance to release it." This is all old news to readers who've worked with threading or other concurrency issues in programs, of course.

The file-based solution works well on a single machine, but it doesn't work well if you need programs on different machines to play nicely together. Using a network filesystem such as NFS doesn't solve that problem either. It may work on certain systems, but it's not portable.

We had this problem at my employer. We were beginning to distribute parts of the workload out to cloned servers, each of whom would do part of the work based on which machine and what time of day, and we needed to make sure they didn't stomp on each other if they miscalculated what work they should be doing. Several options came to mind, all of them using some functionality already provided for us. For instance, we could choose a server as the arbitrator and write a daemon on it to accept network connections. Holding a socket open to that daemon would represent a lock. If the daemon already had a connection from a certain program, it would deny the connection to other programs. If the program with the lock died, the lock server's operating system would know, the daemon would know, and it could make the socket open to another instance.

I was just about to take this approach when I stumbled upon MySQL's [`GET_LOCK`](http://dev.mysql.com/doc/refman/5.0/en/miscellaneous-functions.html) function, completely by accident. I immediately realized we could use it. The syntax is `GET_LOCK(str, timeout)`, and the behavior is to attempt for `timeout` time to get a "lock," returning a value indicating whether it succeeded. The "lock" is exclusive, and setting `timeout` to zero makes the call non-blocking, which is perfect.

I'm putting "lock" in quotes because it's not really a normal database lock, in the way you might be used to thinking of it. It's not a lock on a row, or a lock on a table, or anything like that. It's a lock on an arbitrary string value, and like `flock()`, it's completely advisory; nothing forces a program to wait for the lock---it is up to the programs to play nicely together.

The actual SQL call is `select COALESCE(GET_LOCK('some_string_value', 0), 0)`. The call returns 1 if the lock was granted, and 0 or `NULL` otherwise (hence the `COALESCE()`). The lock is released either explicitly, or when the connection closes. There are more subtleties to the function's behavior, but for this purpose, this is all we need to worry about (you can read the manual to learn more about the subtleties).

We did raise the question "what if the DB server is down?" but quickly answered it: every program that does anything important talks to the DB server, so it's a moot point. Besides, if it's down, we have bigger problems than two programs processing the same data twice.

Unfortunately, as far as I know no other database vendor has provided something similar to this incredibly handy function.

Are there other ways to implement this easily? I can't imagine an easier way---even the `flock()` solution was harder to get right than this---but if you have other ideas, let me know. I'm especially interested in how you'd do this without a DB server.


