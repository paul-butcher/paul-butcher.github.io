---
layout: post
title: Class Properties in Python
categories: python, airflow
date: 2022-01-14 12:33 +0000
---

During a code review, I spotted this interesting misunderstanding, which could have caused quite a lot of unexpected trouble miles away from the actual change.

We have a class with a class property.  The actual property is used to choose which properties should be rendered using a templating engine, and which should just 
be emitted as they are. The class below shows the concept, by using it to choose which properties are included in the string representation

```python
class MyClass:
    props = ["x", "y", "z"]

    def __init__(self, x,y,z, omega):
        self.x = x
        self.y = y
        self.z = z
        self.omega = omega

    def __str__(self):
        return ", ".join((str(v) for k, v in vars(self).items() if k in self.props))
```

This is a popular class, and instances of it are all over the codebase. So, for example, a file might contain something like this:

```python
from xxx import MyClass
my_obj = MyClass(1, 2, 3, 4)
print(my_obj)
```

That obviously prints 1,2,3


However, someone wanted their particular object to use different properties for templating, and wrote a file like this:
```python
from xxx import MyClass

MyClass.props = ["x", "z", "omega"]
my_obj = MyClass(1, 2, 3, 4)
print(my_obj)
```

As you might expect, this prints 1,3,4.

However, because `props` is a class property, if this new file is imported before the first one, then that first one would also print 1,3,4.  
Even if it is imported later, if you later import my_obj from that first file and ask for its string representation, it will also be incorrect.

In the actual code I was reviewing, the answer was simply to remove the MyClass.props line, because it was actually a subset, and they didn't really need the excluded 
property to be excluded.

For completeness - if you really do need your object to have a property with a different value to its corresponding class property, you can just do that by overwriting
the property on the _object_ itself, thus:


```python
from xxx import MyClass
my_obj = MyClass(1, 2, 3, 4)
my_other_obj = MyClass(1, 2, 3, 4)

my_obj.props = ["omega"]

print(my_obj) # 4
print(my_other_obj) # 1, 2, 3
```

In this isolated example, it's really easy to spot and to predict what is going to happen. In the real application, this feature is used to template parameters
to be passed on to calls to remote services etc.  Not all values in template-enabled fields actually contain templates, so a lot of the application would have 
just carried on working as expected, and the errors would manifest in the remote services when they start receiving unreplaced template identifers, rather than
the application itself.

If this had slipped through the reviewing and testing net, then we would suddenly have seen some really surprising failures occurring in places far 
removed from the actual change, and that would have probably caused some serious head-scratching and panic.
