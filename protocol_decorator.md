- [Typed Decorators with Protocols](#typed-decorators-with-protocols)
   * [The Boring Way](#the-boring-way)
   * [Crazy Way No1: Inherit at Type-Check Time](#crazy-way-no1-inherit-at-type-check-time)
   * [Crazy Way No2: Protocols and Workarounds](#crazy-way-no2-protocols-and-workarounds)


# Typed Decorators with Protocols

Python is amazing for implementing the decorator pattern. It is so good at it that young
pythonistas tend to abuse this pattern (I was definitely guilty of that in the good old days).

The instantiation of the decorator problem I want to discuss is as follows:

Say you have a class `A` and you want to extend it without inheriting from it.

```python
class A:
    def __init__(self, a_field: str = "a_field"):
        self.a_field = a_field

    def amethod(self):
        print(f"Greetings from A with {self.a_field=}")
```

This is a common requirement - for example, some mechanism might already give you instances of 
`A` that you just want to extend with new functionality.


You can write a normal wrapper:

```python
class AWrapper:
    def __init__(self, a: A):
        self.a = a

    def new_functionality(self):
        print("New functionality")
```

but then you can't use `AWrapper` in place of `A` because it doesn't have the same interface.


In this unpattern, I will show three different ways to implement a typed decorator that truly
extends `A` without inheritance.

- The first will be sane, boring, and involve a lot of boilerplate. Could work in any language, and it's the one I'd recommend. Understandable, clear, and booooring.
- The second will be more crazy and have some super-weird (pun intended) side effects that will almost never appear, but when they do,
you'll be awfully confused.
- The third one is even weirder, goes deep into python magic and might actually be sorta safe - though the WTF factor is off the charts!

Let's go!

## The Boring Way

You can make `AWrapper` have "the same interface" as `A` by copying
all its methods and just forwarding them to the wrapped instance. At that point,
it would be appropriate to make the member holding the wrapped instance private.
The result would look like this:

```python
class AWrapper:
    def __init__(self, a: A):
        self._a = a
        
    @property
    def a_field(self):
        return self._a.a_field

    def new_functionality(self):
        print("New functionality")

    def amethod(self):
        return self.a.amethod()
```

Just having the "same interface" accidentally is not enough, we want to have
a proper type associated with both `A` and `AWrapper`! 

The boring, javaesc solution is to introduce an interface and make both 
`A` and `AWrapper` implement it:

```python
from abc import ABC, abstractmethod

class ABase(ABC):
    @abstractmethod
    def amethod(self) -> None:
        pass
        
    @property
    @abstractmethod
    def a_field(self) -> str:
        pass

class A(ABase):
    def __init__(self, a_field: str = "a_field"):
        self._a_field = a_field
        
    @property
    def a_field(self) -> str:
        return self._a_field

    def amethod(self):
        print(f"Greetings from A with {self.a_field=}")

class AWrapper(ABase):
    def __init__(self, a: A):
        self._a = a
        
    @property
    def a_field(self) -> str:
        return self._a.a_field

    def new_functionality(self):
        print("New functionality")

    def amethod(self):
        return self._a.amethod()
```

Notice that I had to actually adjust `A` in order to introduce an abstract property as
part of the interface! I would also need to copy all the methods and public fields
(the latter possibly as properties) from `A` to `AWrapper` and to `ABase`.
This is just an actual, normal interface, resulting in a lot of boilerplate. If the class `A` is defined in an external library, 
you can't do that and will need to wrap it in some class of your own for this approach to work.

Moreover, it's not exactly the same as it was before, since now `a_field` is read-only. One could add
a few more lines making it writeable, but that'd be even more code.

I want to repeat: this is the sensible thing to do! Sure, it's a bunch of boilerplate, but it's straightforward and 
there are no surprises. IDE's, type-checkers, and other tools will understand what's going on.

Now, let's get to the fun stuff!

## Crazy Way No1: Inherit at Type-Check Time

One of the first magic python things a beginner learns is the `__getattr__` method, 
which is perfect for forwarding calls to a wrapped object. Functionally, the 
decoration of `A` is achieved trivially:

```python
class AWrapper:
    def __init__(self, a: A):
        self._a = a

    def new_functionality(self):
        print("New functionality")

    def __getattr__(self, item):
        return getattr(self._a, item)
```

The problem is that this will not work with type-checkers or IDEs. I don't know about you,
but I can't live without autocompletion and type-safety catching my (many) simple mistakes. 
Whenever I see nothing proposed on typing `AWrapper(A).am <TAB>`, I get incredibly sad.


With python, types are ignored by the interpreter anyway. So if we want to express that `AWrapper` 
has the same interface as `A`, why not do it at type-checking time only and 
"remove" the inheritance at runtime? In fact, it can be done:

```python

from typing import TYPE_CHECKING

class A:
    def __init__(self, a_field: str = "a_field"):
        self.a_field = a_field

    def amethod(self):
        print(f"Greetings from A with {self.a_field=}")
        
ABase = A if TYPE_CHECKING else object

class AWrapper(ABase):
    def __init__(self, a: A):
        self._a = a

    def new_functionality(self):
        print("New functionality")

    def __getattr__(self, item):
        return getattr(self._a, item)
```

Now you get full completion, mypy and IDE support, all while not inheriting from `A` at runtime!

Neat, right? I stole this shamelessly from the very cool project 
[tinydb](https://github.com/msiemens/tinydb),
where it's used to [pretend that a TinyDB instance is a Table](https://github.com/msiemens/tinydb/blob/master/tinydb/database.py).

However, the downside is that super-weird things happen! 
(Which is the only reason I actually had to dig into that codebase and discover their
brilliant `with_typehint` implementation).

I was just an innocent user, I inherited from `TinyDB` without reading the docs, called `super`, and ran into one of the
weirdest errors of my life: super told me that the method was not available in the parent. 
But the method was there,
I saw it, I was using it! What was going on?

Here's how it would look with the classes defined above:

```python
class AWrapperExtension(AWrapper):
    def method_in_extension(self) -> None:
        super().amethod()
```

running:
```python
AWrapperExtension(A()).method_in_extension()
```

results in

`AttributeError: 'super' object has no attribute 'amethod'`.

I though I was going crazy! The IDE autocompleted the method for me! Moreover, 
calling `AWrapperExtension(A()).amethod()` works and gives the expected result!

The reason behind this failure is some very specific things that happen with `super()` 
that I don't even want to understand... It suffices to say that `super()` is weird, 
and playing around with python magic can easily break it. By trial and error I found
a way to fix it, but this is a serious WTF:

```python
class AWrapperExtension(AWrapper):
    def method_in_extension(self) -> None:
        super().__getattribute__("amethod")()
```
will actually work (contrary to using `getattr(super(), "amethod")()` for some reason).
If you know why and feel like you want to explain it, fire up a PR ;).

Of course, this fix is ugly as hell. Note that you'd probably only run into this if
you want to override a "forwarded" method from `AWrapper` in a subclass and use
`super` inside of it. Otherwise, there is no need to call super:

```python
class AWrapperExtension(AWrapper):
    def method_in_extension(self) -> None:
        self.amethod()
```

works perfectly fine. This kind of override tends to be rather rare, 
your code can work for a very long time until 
a confused colleague or user will run into the `AttributeError`
described above.

## Crazy Way No2: Protocols and Workarounds

This way would actually not be crazy if `Protocol`s in python worked as they should.
Unfortunately, they don't really. A `Protocol` is a perfect way to express behavior
without enforcing an inheritance hierarchy - so exactly what we want to achieve!

In an ideal world (or maybe in a future version of python), we could do:

```python
from typing import Protocol

class AProtocol(Protocol):
    a_field: str
    
    def amethod(self) -> None:
        ...
    

class A:
    def __init__(self, a_field: str = "a_field"):
        self.a_field = a_field

    def amethod(self):
        print(f"Greetings from A with {self.a_field=}")


class AWrapper(AProtocol):
    def __init__(self, a: A):
        self._a = a

    def new_functionality(self):
        print("New functionality")

    def __getattr__(self, item):
        return getattr(self._a, item)
```

Note that `A` doesn't have to inherit from `AProtocol`, so it could perfectly well
come from an external library. In the protocol we could express only the part of `A`
that we actually care about (though the overridden `__getattr__` would forward
all methods and fields of `A`, of course...).

Unfortunately, this doesn't work. Static analysis shows that everything is fine, but
at runtime `AWrapper(A()).amethod()` doesn't do anything. `__getattribute__`, which 
is called before `__getattr__`, forwards the call
to the empty implementation of `amethod` inside the protocol instead of forwarding it to
the wrapped `self._a`. Since this didn't raise an `AttributeError`, it turns out the
`__getattr__` is never called, so our decorator falls apart. Infuriatingly, 
`AWrapper(A()).a_field` **does work**, since the field is just declared in the prototype, 
and not implemented.

The solution is obvious, right? We just need to raise an `AttributeError` in the
prototype, then `__getattr__` will finally be called, and we can all go home happy!

If only... This doesn't work either. I don't know why. It should work! The [documentation](https://docs.python.org/3/reference/datamodel.html#object.__getattribute__)
of `__getattribute__` sais it should work! But it doesn't... 

With

```python
from typing import Protocol

class AProtocol(Protocol):
    a_field: str
    
    def amethod(self) -> None:
        raise AttributeError
    

class A:
    def __init__(self, a_field: str = "a_field"):
        self.a_field = a_field

    def amethod(self):
        print(f"Greetings from A with {self.a_field=}")


class AWrapper(AProtocol):
    def __init__(self, a: A):
        self._a = a

    def new_functionality(self):
        print("New functionality")

    def __getattr__(self, item):
        return getattr(self._a, item)
```

we get that `AWrapper(A()).amethod()` starts to cause an `AttributeError` and
`__getattr__` is still not called.

Well, if `__getattribute__` doesn't want to play along, we will force it! Fortunately,
we can check what methods are inside a class without instantiating it. This leads us
to the following dirty and ugly hack, which however works and which is the main reason
I wrote this article:

```python
from typing import Protocol

class AProtocol(Protocol):
    a_field: str
    
    def amethod(self) -> None:
        ...
    

class A:
    def __init__(self, a_field: str = "a_field"):
        self.a_field = a_field

    def amethod(self):
        print(f"Greetings from A with {self.a_field=}")


class AWrapper(AProtocol):
    def __init__(self, a: A):
        self._a = a

    def new_functionality(self):
        print("New functionality")

    def __getattribute__(self, item):
        if hasattr(AProtocol, item) and item not in AWrapper.__dict__:
            return getattr(self._a, item)
        return super().__getattribute__(item)
```

With this everything works: static type checking, autocompletion, runtime functionality, and
even inheritance:

```python
class AWrapperExtension(AWrapper):
    def method_in_extension(self) -> None:
        super().amethod()
```

no longer leads to the super-weird error.

But yeah, the hack will definitely raise eyebrows... Do try at home but not at work!
