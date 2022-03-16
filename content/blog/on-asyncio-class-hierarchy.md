---
title: "On asyncio class hierarchy"
date: 2022-03-16
draft: False
tags: [python, asyncio, signals]
---

`_UnixSelectorEventLoop` is used as a `_loop_factory`. That is, it is used to create new event loops in unix based system.

What does it look like?

```
class _UnixSelectorEventLoop(selector_events.BaseSelectorEventLoop)
```

It is a subclass of `BaseSelectorEventLoop`.  Rather than going into the details of the unix selector, i'll first dive 
into the base class. 

```
class BaseSelectorEventLoop(base_events.BaseEventLoop)
```

And it inherits from `BaseEventLoop`.
```
class BaseEventLoop(events.AbstractEventLoop)
```

And it inherits from `AbstractEventLoop`.  I'm glad that this is the end to the inheritance hierarchy.

## `AbstractEventLoop`
This abc event loop class defines a lot of methods. All the methods can be grouped in the following categories
based on the comments in the code-
+ Running and stopping the event loop, like run_forever, run_until_complete(future), stop, close
+ Methods scheduling callbacks.  All these return Handles. like cal_soon, call_later, call_at, time
+ Method scheduling a coroutine object: create a task. Only one is here- create_task
+ Methods for interacting with threads, like call_soon_threadsafe, run_in_executor
+ Network I/O methods returning Futures, like getaddrinfo, create_connection, create_server
+ Ready-based callback registration method, like add_reader, remove_reader
+ Completion based I/O methods returning Futures, like sock_recv, sock_accept
+ Signal handling, like add_signal_handler, remove_signal_handler
+ Task factory, like set_task_factory, get_task_factory
+ Error handlers, like get_exception_handler, set_exception_handler
+ Debug flag management, like get_debug, set_debug

Note, this is an abstract class. It does not implement any of the above listed methods. 
Next, lets unwind and drop to the first child- `BaseEventLoop`.

## `BaseEventLoop`

This class implements methods of the abc class.

If there is an event loop then there must be two concepts I should look for. First, a queue where events can park themselves until someone picks them for processing.
This is the place all the new events would go into. And second, something that should process the events in this queue. In some order or priority.

These three instance variables look interesting-
```
self._ready = collections.deque()
self._scheduled = []
self._selector = None
```

### On `self._ready`
- This is just a normal list with objects of type `Handle`
- Populated only by `self._call_soon` method
- Also used for signaling

### On `self._scheduled`
- This is a priority queue with objects of type `TimeHandle`
	- dunder methods `lt`, `le`, `gt`, `ge`, `eq` are implemented for TimeHandle
	- handle with the lowest `_when` is at the top of the queue
- Populated by `call_at` and `call_later`

### On `self._selector`
- this selector monitors multiple file descriptors to see if i/o is possible on them (from man of select)
- it is set in `BaseSelectorEventLoop` to `selectors.DefaultSelector`
- `selectors.DefaultSelector` is [platform dependent](https://github.com/python/cpython/blob/3.9/Lib/selectors.py#L610-L619). For example, for macs it is KqueueSelector, for linux its EpollSelector etc.

These both queues are processed in the `_run_once` method. From the look of it, it is the core of the event loop and deserves its own writing. 
To summarize it processes the events (handles) in `_ready` and `_scheduled` queues. For i/o bound tasks the selector is used. Knowledge of handler, task, future, coroutine is 
needed to completely understand this code. Will do that later.

Unwinding the inheritance hierarchy, lets move to `BaseSelectorEventLoop`.

## `BaseSelectorEventLoop`

This class provides i/o methods on socket. The socket must be in non-blocking mode otherwise 
```
ValueError("the socket must be non-blocking")
```
error is thrown.

The general flow of operations on the socket is-

```
fut = self.create_future()
self._sock_operation(fut, sock)
await fut
```

1. create a future
2. do the specified operation on the socket
3. if 2 is successful then write the results of 2 to future
4. if 2 is unsuccessful due to socket not being ready, then register the socket for read/write event with the selector so that a callback is triggered when the socket becomes readable/writable.  This is done using either `self._add_reader` or `self._add_writer`

For example, consider the case of reading data from a socket.

```
async def sock_recv(self, sock, n):
	"""Receive data from the socket.

	The return value is a bytes object representing the data received.
	The maximum amount of data to be received at once is specified by
	nbytes.
	"""
	base_events._check_ssl_socket(sock)
	if self._debug and sock.gettimeout() != 0:
		raise ValueError("the socket must be non-blocking")
	try:
		return sock.recv(n)
	except (BlockingIOError, InterruptedError):
		pass
	fut = self.create_future()
	fd = sock.fileno()
	self._ensure_fd_no_transport(fd)
	handle = self._add_reader(fd, self._sock_recv, fut, sock, n)
	fut.add_done_callback(
		functools.partial(self._sock_read_done, fd, handle=handle))
	return await fut
```
This method is used to receive n bytes from the socket.

First a read on socket is tried-
```
return sock.recv(n)
```
If its successful then return the bytes read. 

If the read call fails then-
- `fut = self.create_future()` - create a future
- `handle = self._add_reader(fd, self._sock_recv, fut, sock, n)` - ask the selector to call `self._sock_recv` with `fut` and `n` when the socket represented by file descriptor `fd` is ready to be read
- `fut.add_done_callback(...` - when the future is done call `self._sock_read_done` with `fd` and `handle` args to unregister the fd for read events (that is to undo what is done in step above)

Lastly, lets take a look at `self._sock_recv`-
```
def _sock_recv(self, fut, sock, n):
	...
	try:
		data = sock.recv(n)
	except (BlockingIOError, InterruptedError):
		return  # try again next time
	...
	else:
		fut.set_result(data)
```
This method is called by the event loop when the kernel gets to know that the socket is ready to be read.
The data is read from the socket. If successful the data is written to the future. This writing of data to future marks it as done and corresponding "done" callbacks
are triggered.

Apart from this, this class also introduces self socket.
```
self._ssock, self._csock = socket.socketpair()
```
where `csock` is the writable socket, while `ssock` is the readable socket. Writing only involves a single byte. Not sure what they are used for as of now.
Though i do see references to it in next class.

Unwinding the inheritance hierarchy, lets move to `_UnixSelectorEventLoop`.

## `_UnixSelectorEventLoop`

This class provides management for signal handlers and unix server/connection. It integrates signal handling with the event loop.

### How signal handling works

User registers a callback that should be called when the thread receives a signal from os using `add_signal_handler` method like this-
```
event_loop.add_signal_handler(signum, callback, *args)
```
When the signal is received callback is triggered by the event loop along with other callbacks.

The flow starts with asking the signal module to write the signal number to csock when a signal arrives.
```
signal.set_wakeup_fd(self._csock.fileno())
```
Its important to note that csock is a pair socket with ssock as the readable part of the pair. When anything is written to csock, it comes out of ssock. ssock has already been registered with the event loop for read events when `BaseSelectorEventLoop` is initialized-
```
def _make_self_pipe(self):
	# A self-socket, really. :-)
	self._ssock, self._csock = socket.socketpair()
	self._ssock.setblocking(False)
	self._csock.setblocking(False)
	self._internal_fds += 1
	self._add_reader(self._ssock.fileno(), self._read_from_self)
```
When a signal arrives, its signum is written to csock, that wakes up the event loop on ssock and trigger `_read_from_self` method.
This method just reads upto 4096B of data from the socket and then triggers `self._process_self_data(data)`-
```
def _read_from_self(self):
	while True:
		data = self._ssock.recv(4096)
		...
		self._process_self_data(data)
		...
```

The method `_process_self_data` is not defined in `BaseSelectorEventLoop`. It is defined in the child class `_UnixSelectorEventLoop`-
```
def _process_self_data(self, data):
	for signum in data:
		if not signum:
			# ignore null bytes written by _write_to_self()
			continue
		self._handle_signal(signum)
```
This method is simple. It just triggers the `_handle_signal` method.
```
def _handle_signal(self, sig):
	"""Internal helper that is the actual signal handler."""
	handle = self._signal_handlers.get(sig)
	if handle is None:
		return  # Assume it's some race condition.
	if handle._cancelled:
		self.remove_signal_handler(sig)  # Remove it properly.
	else:
		self._add_callback_signalsafe(handle)
```
Here the Handle for the signal is fetched if present and then `_add_callback_signalsafe` is called. This is a fun method defined in `BaseEventLoop`-
```
def _add_callback_signalsafe(self, handle):
	self._add_callback(handle)
	self._write_to_self()
```
Here `_add_callback` simply appends the handle to `self._ready` queue. And, a brilliant move, `_write_to_self` is called. This method is defined in child class `BaseSelectorEventLoop`-
```
def _write_to_self(self):
	...
	csock.send(b'\0')
	...
```
It writes a byte to the csock.  Anything written to csock gets passed on to ssock. When anything comes to ssock selector wakes up the event loop and says- "hi, there is some data in the ssock socket. Please read from it". But in this particular case, the intention of the 0 byte written is to just wake up the event loop so that it can process
`self._ready` queue. This `self._ready` queue has the handle[r] for the signum in current context. And this handle's callback is [called](https://github.com/python/cpython/blob/ccb0e6a3452335a4c3e2433934c3c0c5622a34cd/Lib/asyncio/base_events.py#L1890) at this point.

