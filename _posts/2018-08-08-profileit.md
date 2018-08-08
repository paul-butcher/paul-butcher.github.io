---
layout: post
title: profileit
categories: python, performance
---

Timers, like the
[timeit module](https://docs.python.org/3/library/timeit.html), or
a [timing decorator](https://medium.com/pythonhive/python-decorator-to-measure-the-execution-time-of-methods-fa04cb6bb36d)
can help to find candidates for optimisation.  Proper profiling is better,
but it can sometimes be difficult to isolate the specific area in order to 
run the [cProfile](https://docs.python.org/3/library/profile.html) module.
A decorator can help.

Recently, I was trying to speed up a slow running web response. 
Certain requests, fetching certain data, were much slower, and it
looked tricky to set up an appropriate harness and set of stubs to 
be able to run `python -m cProfile somefile.py`.

Enter the profileit decorator, which outputs results in the same manner
as teh cProfile module.

```python
import cProfile
import pstats


def profileit(limit=30):

    def inner_profileit(func):
        def wrapper(*args, **kwargs):
            prof = cProfile.Profile()
            retval = prof.runcall(func, *args, **kwargs)
            prof.create_stats()
            pstats.Stats(prof).sort_stats('cumtime').print_stats(limit)
            return retval
        return wrapper
    return inner_profileit
```

This decorator is particularly useful for profiling user-actuated functions
in longer running applications such as a desktop GUI or web application.

As a pleasant side-effect, because it shows a list of (some of) the 
functions it can be a quick way to get an overview of the path a
request may take through an unfamiliar application before diving in 
and reading the code or stepping through with a debugger.


