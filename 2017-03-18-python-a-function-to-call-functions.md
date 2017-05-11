---
title: "Python: a function to call functions"
tags: 
- python
---

<em>How do I make a function that can repeat an arbitrary function 
N times in Python?</em>

`repeat.py`

```python
def foo():
    print('In foo')

def bar():
    print('In bar')

def baz():
    print('In baz')

def repeat(func, times):
    for _ in range(times):
        func()

repeat(foo, 2)
repeat(bar, 1)
repeat(baz, 1)
```

Which gives the following output:

    In foo
    In foo
    In bar
    In baz

## Function object

So referring to a function without the `()` gives you the
<em>"function object"</em> whereas using `()` calls the function.

```python
>>> foo
<function foo at 0x7f6b99174b90>
>>> f = foo
>>> f
<function foo at 0x7f6b99174b90>
```

So `f = foo` essentially created an <em>"alias"</em> as we can now refer to
the `foo` function using both names. 

```python
>>> f()
In foo
```

## globals()

One may ask can we modify `repeat()` to take the name of the function we want to call
as a string as opposed to passing the function itself? 

Well yes, there is:

```python
def repeat(func, times):
    for _ in range(times):
        globals()[func]()
```

And here it is in action:

```python
>>> repeat('foo', 2)
In foo
In foo
```

`globals()` returns a dict of <em>"variables"</em> defined in the current scope.

```python
>>> globals()
{'repeat': <function repeat at 0x7fdb9b9628c0>, 
 'foo': <function foo at 0x7f6b99174b90>,
 ...
>>> globals()['foo']
<function foo at 0x7f6b99174b90>
```

`globals()['foo']` refers to the function object just as `foo`
does. In order to call it we add `()`

```python
>>> globals()['foo']()
In foo
```

So what happens if we try call a function that doesn't exist?

```
>>> repeat('lol', 1)
Traceback (most recent call last):
  File "<stdin>", line  1, in <module>
    repeat('lol', 2)
  File "<stdin>", line 22, in repeat
    globals()[func]()
KeyError: 'lol'
```

## dict.get() and callable()

It could be made more robust by making sure `func` is actually a valid function name

```python
def repeat(func, times):
    if callable(globals().get(func)):
        for _ in range(times):
            globals()[func]()
```

We use the `dict.get()` method to avoid any possible `KeyError` and we use `callable()`
to check if what we got is actually a function.

So is `repeat()` good to go?

```
>>> repeat('repeat', 1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 22, in repeat
    globals()[func]()
TypeError: repeat() takes exactly 2 arguments (0 given)
```

Probably not, no. It allows access to <strong>ANY</strong> function visible in `globals()`.

Instead, we could specify a <em>"whitelist"</em> of valid names for `func`:

```python
def repeat(func, times):
    if func in ('foo', 'bar', 'baz'):
        for _ in range(times):
            globals()[func]()
```

## Dispatch Table

Usually though, in the case of wanting to map strings to functions you would 
create your own dict of function objects.

```python
def foo():
    print('In foo')

def bar():
    print('In bar')

def baz():
    print('In baz')

commands = {
    'foo': foo,
    'bar': bar,
    'baz': baz
}

# command = input('What command would you like to run? ')
command = 'callme'

if command in commands:
    commands[command]()
else:
    print('ERROR: {} is not a valid command.')
```

This type of structure is commonly referred to as a <em>"Dispatch Table"</em>.
