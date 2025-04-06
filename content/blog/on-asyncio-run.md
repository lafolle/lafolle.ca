---
title: "On asyncio run"
date: 2022-03-10T22:08:29-08:00
draft: false
tags: []
---
Python formally introduced concurrency using asncio in 3.7+. A simple program that can
demonstrate the hello world of async stuff goes like this-

```
import asyncio

async def main():
	print('hello')
	await asyncio.sleep(1)
	print('world')

asyncio.run(main())
```

In this case `asyncio.run` is the blocking call. It takes coroutine as a param, that is 
created by `main()` due to the `async` qualifer. 

Does it also create an event loop inside? From the doc, yes it does-
> This function always creates a new event loop and closes it at the end.

What if an event loop is already present in the same thread? Will it scheudle the coroutine
in that existing event loop?  From the docs-
> This function cannot be called when another asyncio event loop is running in the same thread.

If that is the case then calling `run` from `run` should throw an error-

```
async def boom():
    await asyncio.sleep(3)

async def main():
    asyncio.run(boom())
    await asyncio.sleep(3)
    print('hello')

asyncio.run(main(), debug=True)
```
Indeed it does-

```
Traceback (most recent call last):
    <NOT RELEVANT>
   asyncio.run(boom())
  File "runners.py", line 33, in run
    raise RuntimeError(
RuntimeError: asyncio.run() cannot be called from a running event loop
sys:1: RuntimeWarning: coroutine 'boom' was never awaited
```

As expected. This is infact the first check in the `run` function.
```
if events._get_running_loop() is not None:
	raise RuntimeError(
		"asyncio.run() cannot be called from a running event loop")
```

Note, the runtime also tells that the `boom` coroutine never got executed. I believe that is just
a side effect of the process exiting before finishing all the coroutines in the event loop.

Next, the run function checks if the passed param is a coroutine or not. It is indeed in our case, so we're good.  
Next, the event loop is created.
```
loop = events.new_event_loop()
try:
	events.set_event_loop(loop)
```

Next the event loop is "set".
```
try:
	events.set_event_loop(loop)
``` 
What does this mean? Going inside `set_event_loop` I see-
```
def set_event_loop(loop):
    """Equivalent to calling get_event_loop_policy().set_event_loop(loop)."""
    get_event_loop_policy().set_event_loop(loop)
```
Ok, so get an instance of an event loop policy and set the loop on it (not sure what the last part means).
The event loop policy is maintained in `_event_loop_policy` global var. If it is set, then that policy is used.
Otherwise a new event loop policy is created using `DefaultEventLoopPolicy`. 

What is `DefaultEventLoopPolicy`?  
By default this is set to `WindowsProactorEventLoopPolicy`. That does not sound right 
because I'm on mac but looks like a type related to windows is being used. At least that is what vim tells me.
Confusing. Lets put breakpoint to decipher that-
```
breakpoint()
asyncio.run(main)
```
After a few `s` and `n`s the true identity of default event loop policy is revealed-
```
(Pdb) ll
722     def get_event_loop_policy():
723         """Get the current event loop policy."""
724         if _event_loop_policy is None:
725             _init_event_loop_policy()
726  ->     return _event_loop_policy
(Pdb) _event_loop_policy
<asyncio.unix_events._UnixDefaultEventLoopPolicy object at 0x104c6bc40>
```

It is correctly set to `_UnixDefaultEventLoopPolicy` rather than the windows one.

Ah, python's vim jedi plugin dropped me to the windows selector for whatever reason and hence the confusion.

But during stepping through the debugger i also realize that the event loop policy is already created beforehand.
That is, L725 in the above code snippet was not hit. Which means somebody set it. When was this done?

Its done in `Lib/asyncio/__init__.py` [link](https://github.com/python/cpython/blob/3.9/Lib/asyncio/__init__.py#L42-L47)-
```
if sys.platform == 'win32':  # pragma: no cover
    from .windows_events import *
    __all__ += windows_events.__all__
else:
    from .unix_events import *  # pragma: no cover
    __all__ += unix_events.__all__
```

On unix based systems `Lib/asyncio/unix_events.py` is executed and the last line there sets the default policy to
`_UnixDefaultEventLoopPolicy`.

## On Event loop policy
I am staring at two concepts-
1. event loop
2. event loop policy  
and both are binded together using `set_event_loop` method on the policy.

What is this policy thing?  
Hmmm, the unix policy inherits from a base class-
```
class _UnixDefaultEventLoopPolicy(events.BaseDefaultEventLoopPolicy):
```
Lets take a look at the base class first.  
Ok, that inherits from an abstract class-
```
class BaseDefaultEventLoopPolicy(AbstractEventLoopPolicy):
```

Lets take a look at the abc class. It states just this in the doc-
> Abstract policy for accessing the event loop.

Apart from that it expects following 3 methods to be implemented in child classes-

1. get_event_loop
2. set_event_loop
3. new_event_loop

And two additional methods which are only supported in unix platform-

1. get_child_watcher
2. set_child_watcher

From the look of it the policy appears to manage the event loop instance and watchers of forked processes,
in some "context". This "context" thing is not clear yet.

Ok. Lets go back to the concrete child class `BaseDefaultEventLoopPolicy`. The documentation here is 
neat and is understandable now that I have traversed its base. The doc goes like-

> In this policy, each thread has its own event loop.  However, we only automatically create an event loop by default for the main thread; other threads by default have no event loop.
>
> Other policies may have different rules (e.g. a single global event loop, or automatically creating an event loop per thread, or using some other notion of context to which an event loop is associated).

Right. The policy does determine the relationship between "context", that in our case is thread for this base default policy, and the event loop.
One loop in main thread and none for the others. Or each thread with its own event loop. Apart from these two points there is also a concept of main thread.
To digress a bit, what is this main thread? The first thread that is created in the process by the kernel? Lets take a quick look at `threading.main_thread`-

> In normal conditions, the main thread is the thread from which the Python interpreter was started.

Neat. That is the documentation of the `main_thread` in the threading module. For now, I would not go into abnormal conditions.

Back to policy stuff. The `_UnixDefaultEventLoopPolicy` stores the event loop instance in the thread local storage. For this it defines 
an internal class that inherits from `threading.local` which represents TLS (transport layer secu...kidding)-
```
class _Local(threading.local):
	_loop = None
	_set_called = False
```

When an instance of this policy is created `self._local` is set to TLS+two extra data-
```
def __init__(self):
	self._local = self._Local()
```

The most interesting is the `get_event_loop` method-

```
def get_event_loop(self):
        if (self._local._loop is None and
                not self._local._set_called and
                threading.current_thread() is threading.main_thread()):
            self.set_event_loop(self.new_event_loop())

        if self._local._loop is None:
            raise RuntimeError('There is no current event loop in thread %r.'
                               % threading.current_thread().name)

        return self._local._loop
```
Here, the first condition states that if-
+ local loop is not already set and
+ setting local loop has not already been called and
+ this method is being executed on the main thread (on which the py interpreter first started)

is true, then create a new event loop and set it in the TLS (thread local storage).

Next, if no loop is present in the current thread, then raise exception. Lastly return the loop.

Only thing left here now is the `_loop_factory`. This is the thing that actually creates the event loop.
And it is not defined in the default policy class. Where is it?  
Aha, the unix's event loop policy that derives from the default policy has it-
```
class _UnixDefaultEventLoopPolicy(events.BaseDefaultEventLoopPolicy):
    """UNIX event loop policy with a watcher for child processes."""
    _loop_factory = _UnixSelectorEventLoop
```
So, what is `_UnixSelectorEventLoop`? And how is that involved in giving birth to event loop? This i'll cover in next step.

