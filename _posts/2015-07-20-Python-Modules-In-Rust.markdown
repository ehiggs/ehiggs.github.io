---
layout: post
title: "Python Modules in Rust"
date: 2015-07-20 20:00
categories:
- python
- rust
---
I consider myself mainly a Python developer these days, but coming from a
history of C++ development, I've been really interested in seeing how Rust pans
out. Could it be an ideal language to use when I need the performance that
Python just can't provide? Can I happily replace C or C++ (using boost) with
Rust? 

I set out to see the current state of binding Python to Rust.

# rust-cpython

The best library I've found for writing modules in Rust is
[rust-cpython](https://github.com/dgrunwald/rust-cpython/). It had the widest
range of coverage that I've found and it supports both Python 2 and Python 3.

Let's begin with a fibonacci sequence calculator. It's pretty basic in Python:

{% gist 9d7bb6977ea084491c57 %} 

## C
C has a lot of boiler plate where I could just use the `cffi` module. But
using the actual API is worth it in case you want to do any sort of error
handling in the C side. For example, if you pass a wrong type through `cffi`,
you basically just segfault. You can wrap your C code with error handling in
Python. But if the intent is to make a function you call in a loop, you want the
error checking to also be fast.  

If I'm honest, I always feel like I'm putting in a lot of bugs when I use the C
API. Especially when I compare it with the Cython output.

{% gist b57750eae564b1de4b75 %}

## C++ (Boost.Python)
Here's what the equivalent looks like in
[Boost.Python](http://www.boost.org/doc/libs/1_58_0/libs/python/doc/).
Boost.Python is a really nice library for wrapping C++ to call from Python.
Except if you manage to make a mistake you get all the associated C++
compilation errors some of us love to hate. It's allegedly gotten better, but in
making this example, I still found some error explosions which take some
searching around on the internet to figure out. 

Sadly Boost.Python only seems to support Python 2. I know a lot of people using
Python 2 who have no interest in migrating to Python 3, but if you end up
wrapping a lot of your code in Boost.Python, you're really sealing your fate. I
don't recommend it.

{% gist 53a5d6b8a4e593f90447 %}

## Cython
Cython is a fantastic tool for writing Python bindings. It's a dialect of Python
that looks so much like Python that you can almost copy paste your code into a
`pyx` module, configure `setuptools` to find it, and you're running at a much
faster speed. If you take a look at the Cython code, set the types on the
variables, turn off array bounds checking, and other tweaks, it competes with
hand written C. It really is awesome.

{% gist 587e2b8982e7690a85dd %}

## Rusta (rust-cpython)
Writing this in Rust took some trial and error since it's so new. But it was
actually a very pleasant experience. With a little manipulation of the
`Cargo.toml` file (used to define a reproducable build in Rust), you can build
against Python 2, or Python 3. None of the other wrappers, aside from Cython can
do this.
Finally here's how it looks in Rust:

{% gist bde1733d977f84b5509b %}

I ran some basic tests so see how the performance fares, and it turns out that
it's pretty competetive. I wouldn't put too much stock into the actual numbers
aside from the fact that they're within an order of magnitude of each other. I
used GCC for C and Rust is built on LLVM. 

The test is a really noddy one. It's just using 'timeit' on the 91st fibonacci
number from the IPython shell:  

`timeit fib(90)`

| Python Version | Implementing language | timeit result |
|----------------|-----------------------|---------------|
| Python 3.4.2   | Python | `100000 loops, best of 3: 7.05 µs per loop` |
| Python 3.4.2   | C      | `1000000 loops, best of 3: 288 ns per loop` |
| Python 3.4.2   | Rust   | `1000000 loops, best of 3: 229 ns per loop` |
| Python 3.4.2   | Cython | `10000000 loops, best of 3: 192 ns per loop` |
| Python 2.7.10  | Python | `100000 loops, best of 3: 6.11 µs per loop` |
| Python 2.7.10  | C++    | `1000000 loops, best of 3: 219 ns per loop` |
| Python 2.7.10  | Rust   | `1000000 loops, best of 3: 177 ns per loop` |
| Python 2.7.10  | Cython | `10000000 loops, best of 3: 171 ns per loop` |

It's not really interesting either aside from the confidence that the Rust API
wrapper probably isn't getting into the way too much.


# Current work
Some Rustaceans/Pythonistas are working on this infrastructure and it's pretty
exciting. Other pieces of work being put into place are integration with 
[Python's setuptools](https://github.com/novocaine/rust-python-ext) so you can
run `python setup.py install` and it will be able to build the Python modules
just like you would expect.


## Current issues
1. I haven't found anyone who is looking to support Numpy yet, so anyone who
   wants to do their numeric/scientific development in Python/Rust should hold
   off at the moment. 
2. I don't see how to specify the library name in Cargo. This means all the
   shared objects are always prefixed with `lib`. This means you need to import
   the module as `import libXYZ` which isn't really what you want.


# Further work
I haven't tested managing objects on the Rust side, but this is apparently a
place where Python can potentially get some performance improvements. For
example, [`collections.OrderedDict` was implemented in raw Python until Python
3.5](https://bugs.python.org/issue16991). Of course, the ability to use
[RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)
on the Rust side should make writing containers in Rust a lot easier than in C.


# Conclusion
Rust looks like a really great language to implement Python modules. It's not
there yet if you want to integrate with the numeric/stats/machine learning
libraries that Python offers. But with some love and more community effort, I
could see a lot of work extending Python with Rust. I think if you have some
performance sensitive systems project and want to control it from a scripting
language, you could very well do this by combining Rust and Python.
