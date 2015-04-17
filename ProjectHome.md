## Introduction ##

multitask allows [Python](http://python.org/) programs to use
[generators](http://www.python.org/doc/2.5/whatsnew/pep-342.html) (a.k.a. coroutines) to
perform cooperative multitasking and asynchronous I/O.  Applications
written using multitask consist of a set of cooperating tasks that
yield to a shared task manager whenever they perform a (potentially)
blocking operation, such as I/O on a socket or getting data from a
queue.  The task manager temporarily suspends the task (allowing other
tasks to run in the meantime) and then restarts it when the blocking
operation is complete.  Such an approach is suitable for applications
that would otherwise have to use `select()` and/or multiple threads to
achieve concurrency.

## Requirements ##

multitask requires Python 2.5 or later.  No additional software is required.

## Examples ##

The functions and classes in the multitask module allow tasks to yield
for I/O operations on sockets and file descriptors, adding/removing
data to/from queues, or sleeping for a specified interval.  When
yielding, a task can also specify a timeout.  If the operation for
which the task yielded has not completed after the given number of
seconds, the task is restarted, and a `Timeout` exception is raised at
the point of yielding.

As a very simple example, here's how one could use multitask to allow
two unrelated tasks to run concurrently:

```
>>> def printer(message):
...     while True:
...         print message
...         yield
... 
>>> multitask.add(printer('hello'))
>>> multitask.add(printer('goodbye'))
>>> multitask.run()
hello
goodbye
hello
goodbye
hello
goodbye
[and so on ...]
```

For a more useful example, here's how one could implement a
multitasking server that can handle multiple concurrent client
connections:

```
def listener(sock):
    while True:
        conn, address = (yield multitask.accept(sock))
        multitask.add(client_handler(conn))

def client_handler(sock):
    while True:
        request = (yield multitask.recv(sock, 1024))
        if not request:
            break
        response = handle_request(request)
        yield multitask.send(sock, response)

multitask.add(listener(sock))
multitask.run()
```

Tasks can also yield other tasks, which allows for composition of
tasks and reuse of existing multitasking code.  A child task runs
until it either completes or raises an exception.  To return output to
its parent, a child task raises `StopIteration`, passing the output
value(s) to the `StopIteration` constructor.  An unhandled exception
raised within a child task is propagated to its parent.  For example:

```
>>> def parent():
...     print (yield return_none())
...     print (yield return_one())
...     print (yield return_many())
...     try:
...         yield raise_exception()
...     except Exception, e:
...         print 'caught exception: %s' % e
... 
>>> def return_none():
...     yield
...     # do nothing
...     # or return
...     # or raise StopIteration
...     # or raise StopIteration(None)
... 
>>> def return_one():
...     yield
...     raise StopIteration(1)
... 
>>> def return_many():
...     yield
...     raise StopIteration(2, 3)  # or raise StopIteration((2, 3))
... 
>>> def raise_exception():
...     yield
...     raise RuntimeError('foo')
... 
>>> multitask.add(parent())
>>> multitask.run()
None
1
(2, 3)
caught exception: foo
```