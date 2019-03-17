---
layout: post
title: Understanding Condition Variables
date: '2012-02-10T00:29:00.000-08:00'
author: David Coles
tags: 
modified_time: '2019-03-16T23:21:25-07:00'
redirect_from:
  - /blog/2012/02/understanding-how-to-use-condition.html
---

One concept that I often end up explaining a several times is
POSIX Condition Variables. Taken straight from the official
[POSIX.1-2008/SUSv4](http://pubs.opengroup.org/onlinepubs/9699919799/)
documentation:

[**Condition Variable**](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap03.html#tag_03_107)

> A synchronization object which allows a thread to suspend execution,
> repeatedly, until some associated predicate becomes true. A thread whose
> execution is suspended on a condition variable is said to be blocked on the
> condition variable.

So clearly it's related to multi-threading but what do we use it for?

Well if you've looked into multi-threaded programming you've probably
encountered the concept of a **mutex**. A mutex helps us ensure that our program
runs correctly by protecting some shared state from being modified by
two threads at the same time.

Condition Variables solve a problem that often pops up shortly after we make a
part of our program thread safe: What do we do when a thread encounters a
shared state which it can't do any work on?

As an example consider a producer and a consumer thread passing integers across
a stack. We have a mutex on the stack to ensure that the producer doesn't try
and add items to the stack while the consumer is removing them and vice-versa.
But what happens if the consumer acquires the mutex and finds that the stack is empty?

A really dumb method would be to unlock the mutex and then immediately try again.
This has the downside of it eats up heaps of CPU time while we keep
locking-checking-unlocking over and over again. A better option is to sleep for
a little bit of time if we find that the stack is empty, but then the question
is how long should we wait for?

Condition Variables solves this problem by allowing a thread to sleep on the
condition variable until another thread wakes it up by signalling that
condition variable.

## Sample Code

[This sample](http://bazaar.launchpad.net/~dcoles/+junk/condition-example/view/head:/src/condition_example.c)
is a fairly extensive walk through of using condition variables to implement
a blocking stack in C.
 
You can [download the full project source here](https://code.launchpad.net/~dcoles/+junk/condition-example)
 
## Notes about POSIX Thread implementation
 
### Use a while loop for checking state
 
This serves two purposes. On some platforms threads may be spuriously woken up
even if the condition variable wasn't signaled. A while loop means we'll go
back to waiting.
 
```c
pthread_mutex_lock(&mutex);

while (!correct_state) {
    pthread_cond_wait(&cond, &mutex);
}

// Do some work here...

pthread_mutex_unlock(&mutex);
```

A second use is that sometimes it's easier to write code where we signal a
condition variable for a more general condition than we're interested in.
A thread can wake up and check the exact state and if it's not correct, go back
to waiting.

### Signal vs. Broadcast

The POSIX Thread API has two different methods for signalling a condition variable:
 
[`pthread_cond_signal` and `pthread_cond_broadcast`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cond_broadcast.html).
The difference is `pthread_cond_signal` will **wake at least one thread**,
while `pthread_cond_broadcast` will **wake all threads** waiting on a
condition variable.

If each thread checks the state with a while loop like above,
`pthread_cond_broadcast` will always be safe (threads that can't do any work
will go back to sleep). The advantage of `pthread_cond_signal` is if you know
only one thread will be able to do work, this won't wake all of the thread
(usually it will only wake one, but don't rely on this - again, use a while
loop to check state).

### Timed wait

If you don't want a thread waiting forever you can use
[`pthread_cond_timedwait`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cond_timedwait.html)
to try waiting for a condition up to a certain time (specified by absolute
[`timespec`](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/time.h.html)
such as one returned by
[`clock_gettime`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/clock_getres.html).
The function will return `ETIMEDOUT` if/when the absolute time has been passed.

### Dynamic initialization

Like mutexs the easiest way to use the static initializer, but in some cases
you may need to dynamically allocate a condition variable.

```c
// Static
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

// Dynamic
pthread_cond_t cond;
pthread_cond_init(&cond, NULL);
// ...
pthread_cond_destroy(&cond);
```

### Windows

Newer versions of Windows (Vista and above) have
[support for condition variables](http://msdn.microsoft.com/en-us/library/windows/desktop/ms682052(v=vs.85).aspx),
though most Windows developers are more familiar with
[events](http://msdn.microsoft.com/en-us/library/windows/desktop/ms682655(v=vs.85).aspx).

**Events** are similar to condition variables, but they're not associated with
a mutex. You can think of them like a mutex/condition variable pair wrapped
around a boolean (it's pretty easy to implement events with condition variables
his way).

It's possible to implement condition variables with events, but
[it's not trivial](https://github.com/pocoproject/poco/blob/2731c5fcfa38bff13a429ea7a3d9993138c51eae/Foundation/include/Poco/Condition.h).

## References

- [POSIX Thread API (POSIX.1-2008/SUSv4)](http://pubs.opengroup.org/onlinepubs/9699919799/functions/V2_chap02.html#tag_15_09)
- [Windows Synchronization Objects](http://msdn.microsoft.com/en-us/library/windows/desktop/ms686364(v=vs.85).aspx)
- [The Linux Programming Interface](http://man7.org/tlpi/) &mdash; Has an excellent coverage of threads.
    Also see these two code examples: [`prod_no_condvar.c`](http://man7.org/tlpi/code/online/dist/threads/prod_no_condvar.c.html)
    and [`threads/prod_condvar.c`](http://man7.org/tlpi/code/online/dist/threads/prod_condvar.c.html).
