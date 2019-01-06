---
layout: post
title: "gevent Loop Control"
date: 2019-01-06 15:50:10
categories: gevent queue python3 python
---
[gevent](http://www.gevent.org/) is a coroutine-based python library that provides constructs to help you build powerful non-blocking applications.

I recommend checking out the excellent tutorial created by sdiehl here: [https://sdiehl.github.io/gevent-tutorial/](https://sdiehl.github.io/gevent-tutorial/) to get started.

One of my most common gevent use cases is to use the `Queue` class to process items. 

An example from gevent-tutorial:
{% highlight python %}
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Worker %s got task %s' % (n, task))
        gevent.sleep(0)

    print('Quitting time!')

def boss():
    for i in range(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
{% endhighlight %}

This works great if you can ensure the queue is never empty, but not so great where producers may not always be active. A better way to define your workers would be to take advantage of Python's generator idiom:

{% highlight python %}
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    for task in tasks:
        print('Worker %s got task %s' % (n, task))

    print('Quitting time!')

def boss():
    for i in range(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
{% endhighlight %}

But that leads to this error: 
```
gevent.exceptions.LoopExit: This operation would block forever
```
Why? simple, once there are no other greenlets for gevent to switch to (no producers producing...) the last greenlet would block the main loop forever.


To ensure a timely exit of your worker either when the producer shuts down or you receive a signal you can use the `StopIteration` builtin in Python 3 to gracefully stop your worker.

{% highlight python %}
import gevent
from gevent.queue import Queue

tasks = Queue()


def worker(n):
  for task in tasks:
    print('Worker %s got task %s' % (n, task))

  print('%s quitting time!' % (n))


def boss():
  for i in range(1, 25):
      tasks.put_nowait(i)
  tasks.put(StopIteration)
  tasks.put(StopIteration)
  tasks.put(StopIteration)


gevent.spawn(boss).join()
gevent.joinall([
  gevent.spawn(worker, 'steve'),
  gevent.spawn(worker, 'john'),
  gevent.spawn(worker, 'nancy'),
])
{% endhighlight %}
{% highlight bash %}
Worker steve got task 1
Worker steve got task 2
Worker steve got task 3
Worker steve got task 4
Worker steve got task 5
Worker steve got task 6
Worker steve got task 7
Worker steve got task 8
Worker steve got task 9
Worker steve got task 10
Worker steve got task 11
Worker steve got task 12
Worker steve got task 13
Worker steve got task 14
Worker steve got task 15
Worker steve got task 16
Worker steve got task 17
Worker steve got task 18
Worker steve got task 19
Worker steve got task 20
Worker steve got task 21
Worker steve got task 22
Worker steve got task 23
Worker steve got task 24
steve quitting time!
john quitting time!
nancy quitting time!
{% endhighlight %}

Hopefully this enables you to write clearer and better gevent code.