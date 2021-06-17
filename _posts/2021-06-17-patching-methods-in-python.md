---
layout: post
title: Patching Methods in Python
categories: python testing
date: 2021-06-17 21:33 +0100
---
Today, a colleague was adding tests to an old and previously untested class.
One problem was that the `__init__` method was long and made several 
calls to set up connections with remote services etc.  He asked me how
to go about adding tests in this situation, so I made a quick example.

The example shows various ways you can patch part of a class while
leaving the rest of it intact.

```python
from unittest import mock


class MyClass:
    def __init__(self):
        print('hello')
        self.my_property = 'dolly'

    def do_something(self):
        print('world')
        
    def do_something_else(self):
        print(self.my_property)

    def do_yet_another_thing(self):
        o = MyOtherClass()
        print(f"{o.do_something()} {self.do_something()}")


class MyOtherClass:
    def do_something(self):
        return 'goodbye'


print('exercise the class normally:')
o = MyClass()
o.do_something()
o.do_something_else()
o.do_yet_another_thing()


print('\nexercise the patched class:')
with mock.patch.object(MyClass, '__init__', return_value=None):
    # Note that it does not say "hello" at this point
    o2 = MyClass()


# Here, it prints "world"
o2.do_something()

# If the real function you are exercising relies on
# properties set in __init__, you will have to
# set them here instead.
o2.my_property = "Is it me you're looking for"
o2.do_something_else()

# You can patch any methods.
# Using the context manager means that the patch disappears
# when no longer in scope
with mock.patch.object(MyClass, 'do_something', return_value="banana"):
    o2.do_yet_another_thing()  # goodbye banana

o2.do_yet_another_thing()  # goodbye None

# If you are calling another class, you can mock out the constructor
# as though a function, but it's a bit more clunky, so you may be better off patching the 
# __init__ and the method you are calling.
with mock.patch('patching.MyOtherClass') as mocked:
    mocked.return_value.do_something.return_value = 'farewell'
    o2.do_yet_another_thing()  # farewell None

```

Obviously, the best thing to do is to refactor the class to make it more
testable, but refactoring without tests is a risky business, so the more
tests you can add beforehand, the better.  A good way to do this is to mock 
out anything that is difficult to test, cover everything else, then 
refactor the hard part.
