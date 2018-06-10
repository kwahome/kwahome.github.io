---
layout: post
title:  Python 2 vs 3 hasattr() behaviour
image: ''
date:   2018-06-10 10:00:00
tags:
    - Python
    - Programming
description: 'A walkthrough on how hasattr() has changed behaviour between Python 2 and 3'
categories:
    - Python
---

<img src="../../media/images/python-2-vs-python-31-780x350.jpg" alt="">

`hasattr()` presents a huge edge case when writing Python 2 and 3 compatible code.

You might want to rethink using it unless you can put up with it’s kinks or are writing Python 3 only code and understand how it works.

Especially, if you’re handling property attributes or classes that aren’t your own.


### __hasattr(object, name)__
`hasattr()` checks for the existence of an attribute by trying to retrieve it.

This is implemented by calling `getattr(object, name)`, which does a lookup, and seeing whether an exception is raised.

An attribute lookup is the only reliable way of knowing if an attribute exists, since there are so many dynamic ways to inject attributes on Python objects; `__getattr__`, `__getattribute__`, `property` objects, meta classes, etc.

Thus `hasattr(foo, 'bar')` below:

{% highlight python %}
    if hasattr(foo, 'bar'):
        # do something with bar in Foo
        print(Foo.bar)
    else:
        # no bar found in Foo
        print("no bar found in Foo")
{% endhighlight %}

approximately equates to:

{% highlight python %}
    try:
        getattr(foo, 'bar') # try lookup bar in foo
        return True
    except:
        # catch all exceptions and return false
        return False
{% endhighlight %}
The handling of exceptions raised in the lookup process has changed, rightly so, in Python 3 resulting in a change in behaviour and an edge case while writing 2–3 hybrid code.

### <a href="https://docs.python.org/2.7/library/functions.html#hasattr" target="_blank" style="color:#3b5998">__Python 2__</a>:
```
hasattr(object, name)

The arguments are an object and a string. The result is True if the string is the name of one of the object’s attributes, False if not. (This is implemented by calling `getattr(object, name)` and seeing whether it raises an exception or not.)
```

In python 2, `hasattr()` <a href="https://github.com/python/cpython/blob/2.7/Python/bltinmodule.c#L907" target="_blank" style="color:#3b5998">catches all attribute lookup exceptions</a> and returns False.

The implication here is that `hasattr()` masks all exceptions raised whilst trying to retrieve an attribute from an object, then assumes the attribute sought does not exist when it in fact does.

### <a href="https://docs.python.org/3/library/functions.html#hasattr" target="_blank" style="color:#3b5998">__Python 3__</a>:
```
hasattr(object, name)

The arguments are an object and a string. The result is True if the string is the name of one of the object’s attributes, False if not. (This is implemented by calling getattr(object, name) and seeing whether it raises an AttributeError or not.)
```
In python 3, `hasattr()` <a href="https://github.com/python/cpython/blob/3.4/Python/bltinmodule.c#L995" target="_blank" style="color:#3b5998">catches `AttributeErrors` only</a> then returns False. All other exceptions bubble up the call stack.


It’s thus a much more truthful representation of the existence of an object.

However, it’s impossible to distinguish an `AttributeError` from a missing attribute and an `AttributeError` that‘s as a result of a buggy attribute particularly if a property is involved.

### __Property attribute__
Python has a concept called a <a href="https://docs.python.org/2.7/library/functions.html#hasattr" target="_blank" style="color:#3b5998">__property__</a> that can do several very useful things.

The property function, largely used with the `@decorator` syntax implements data encapsulation in Python classes.

When Python encounters the following code:

{% highlight python %}
    class Foo(object):
         """
         class Foo with methods bar and baz
         """
         @property
         def bar(self):
             return "this is bar"

         def baz(self):
             return "this is baz"

    foo = Foo()
    print(foo.bar)
    print(foo.baz)
{% endhighlight %}

it looks up bar in foo, then inspects bar to see if it has a `__get__`, `__set__`, or `__delete__` method; if it does, it's a property.

If bar is a property, rather than just returning the bar object (as it would for any other attribute e.g. `foo.baz`) the `__get__` method will be called (since we were doing an attribute lookup) and return whatever that method returns.

Thus, the result of the block of code above is:

{% highlight python %}
    >>> foo = Foo()
    >>>
    >>> print(foo.bar)
    'this is bar'
    >>>
    >>> print(foo.baz)
    <bound method Foo.baz of <Foo object at 0x7fd2c7828c50>>
{% endhighlight %}

If a property raises an exception, the exception propagates up the call stack:

{% highlight python %}
    class Foo(object):
         """
         class Foo with methods bar and baz
         """
         @property
         def bar(self):
             raise SyntaxError

         def baz(self):
             raise SyntaxError

    foo = Foo()
    print(foo.bar)
    print(foo.baz)
{% endhighlight %}

Thus, the result of the modified block of code above is::

{% highlight python %}
    >>> foo = Foo()
    >>>
    >>> print(foo.bar)
    Traceback (most recent call last):
      File "<input>", line 11, in <module>
      File "<input>", line 4, in bar
    SyntaxError: None
    >>>
    >>> print(foo.baz)
    <bound method Foo.baz of <Foo object at 0x7fe06f8ce350>>
{% endhighlight %}


### __Property & hasattr() — an unholy alliance__

At this point your premonition of what I’m getting at is absolutely spot on!

`hasattr()` in Python 2 will shadow all exceptions in properties and wrongly communicate that an attribute does not exist.

In Python 3 `hasattr()` blows up with the exceptions raised by the lookup process and only communicates that an attribute does not exist on `AttributeErrors`

{% highlight python %}
    class Foo(object):
         """
         class Foo with methods bar and baz
         """
         @property
         def bar(self):
             raise SyntaxError

         def baz(self):
             raise SyntaxError

    foo = Foo()
{% endhighlight %}

Python 2:

{% highlight python %}
    >>> sys.version
    '2.7.12 (default, Dec 16 2016, 03:08:23) \n[GCC 4.9.2]'
    >>>
    >>> hasattr(foo, 'bar')
    False
    >>>
    >>> hasattr(foo, 'baz')
    True
    >>>
{% endhighlight %}

Python 3:

{% highlight python %}
    >>> sys.version
    '3.4.8 (default, Feb 17 2018, 09:40:38) \n[GCC 4.9.2]'
    >>>
    >>> hasattr(foo, 'bar')
    Traceback (most recent call last):
      File "/usr/local/lib/python3.4/code.py", line 90, in runcode
        exec(code, self.locals)
      File "<input>", line 1, in <module>
      File "<input>", line 4, in bar
    SyntaxError: None
    >>>
    >>> hasattr(foo, 'baz')
    True
    >>>
{% endhighlight %}

Another vivid ramification of `hasattr()`` on properties is that it executes their getter functions which makes the name quite a misnomer.

### __Alternatives?__
I am in no way implying that `hasattr()`` is to be avoided like the plague. Certainly not! It’s still a perfectly applicable builtin function with a lot to offer.

Trend with caution however, lest you wallow in it’s nasty surprises.

Especially when treading the Python 2–3 hybrid code line or when dealing with third party library classes; since in classes that aren’t yours you may not know whether an attribute is a `property` or becomes one in a future release.

You’d also need to keep tabs with changes in your own classes and evolve (fix) usages of `hasattr()` which adds on to the maintenance burden.

Some alternatives are:

{% highlight python %}
    foo = Foo()

    try:
         print(foo.bar)
    except AttributeError:
         print("no bar found in Foo")
{% endhighlight %}

or

{% highlight python %}
    # assumes attribute absence equivalent to value being None
    foo = Foo()

    attr = getattr(foo, "bar", None)
    if attr is not None:
         print(attr)
    else:
         print("no bar found in Foo")
{% endhighlight %}

If you’re thinking *“what about performance?”*, `hasattr()` is not any faster than `getattr()` since it does the same attribute lookup process before throwing away the result.
