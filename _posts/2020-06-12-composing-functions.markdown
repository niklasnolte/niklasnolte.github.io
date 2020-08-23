---
layout: post
title:  "Composing functions in Python"
date:   2020-06-12 15:00 +0100
categories: jekyll update
---

Learning a functional language is a very enjoyable experience. Haskell, in my
case, is very different from imperative languages like C++ and Python. Even a
"simple" thing like IO suddenly becomes a difficult piece of functionality. On
the other hand, Haskell has some features that I miss elsewhere, like lazy
evaluation or the way one can natively bind function arguments and compose
functions.

Well, function composition is something that seems achievable with Python:

```python
def f(x):
  return x - 2

def g(x):
  return 2 * x

x = 7

# now I want (g . f)
g(f(x)) # applied g on the output of f
```

What if i want to only compose, without immediate application?
I just want the function that can be represented by h = (g . f)

```python
x = list(range(10))

h = lambda x : g(f(x))

map(h, x)
```
Ok, that works fine. What if i wanted to have the incredibly convenient syntax as haskell has it?
What we need to do then is to override a binary operator to compose. How about `__mul__`?
The problem: Builtins cannot be extended

```python
def compose(g,f):
  return lambda x : g(f(x))

type(f).__mul__ = compose
# TypeError: can't set attributes of built-in/extension type 'function'
```

Fortunately for us, [clarete](https://github.com/clarete) hacked around in the C-python bindings to make
builtin extensions possible directly from python:
[forbiddenfruit](https://github.com/clarete/forbiddenfruit). Whether or not that is a good idea, who knows?

```python
from forbiddenfruit import curse

def f(x):
  return x - 2

def g(x):
  return 2 * x

curse(type(f), '__mul__', compose)

x = list(range(10))

list(map(g*f, x))
# [-4, -2, 0, 2, 4, 6, 8, 10, 12, 14]
```

One can also compose more than functions, `h*g*f`, or adjust the compose function to take `*args` or `**kwargs`.

What can you do with extended builtins?
