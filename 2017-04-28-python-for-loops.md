---
title: "Python: for loops"
date:  2017-04-28 12:11:59 +0100
tags:
 - python
---

Python's `for` loop has *"super magic powahz!!"* It may help
to think of it like a *"for each"* loop e.g. 

*for each something in something:*

## list

When you iterate over a list with `for` you will iterate
over each item <small>(*"element"*)</small> in the list.

```python
>>> for word in ['hello', 'hi', 'sup']:
...     print(word)
...
'hello'
'hi'
'sup'
```

Inlining the definition of the list may look strange there 
so we'll create a list with a name and loop through it.

```python
>>> words = ['hello', 'hi', 'sup']
>>> for word in words:
...     print(word)
'hello'
'hi'
'sup'
```

We don't need to worry about indexes <small>(*"indices"*)</small>
because as we said you iterate directly over the items contained in 
the list.

What if we wanted the indexes? Let's say for example we 
had a leaderboard of names and wanted to print their position
and name.

```python
>>> top3 = ['bob', 'alice', 'ted']
```

Well we could use `range()` to give us a list of indexes 

```python
>>> range(len(top3))
range(0, 3)
```

Which we could then loop over.

```python
>>> for n in range(len(top3)):
...     print(n, top3[n])
... 
0 bob
1 alice
2 ted
```

Oops, we'll need to use `n + 1` for the *"position"* because list 
indexes being at `0`

```python
>>> for n in range(len(top3)):
...     print(n + 1, top3[n])
... 
1 bob
2 alice
3 ted
```

## enumerate()

Another option is to use Python's `enumerate()` which will take an 
iterable and give you back an iterable of `index, value` pairs 
(i.e. tuples)

```python
>>> enumerate(top3)
<enumerate object at 0x7ffa58a3e828>
```

We'll use `list()` to turn it into a list to see its
contents.

```
>>> list(enumerate(top3))
[(0, 'bob'), (1, 'alice'), (2, 'ted')]
```

Okay so we can see it gives us the indexes and the
values which we can loop over. 

```
>>> for n, name in enumerate(top3):
...     print(n + 1, name)
...
1 bob
2 alice
3 ted
```

We have to use `n + 1` because the indexing starts
at 0 however could have also used the `start` argument 
of `enumerate()` to specify what number to start counting
from.

```python
>>> for n, name in enumerate(top3, start=1):
...     print(n, name)
...
1 bob
2 alice
3 ted
```

## list of lists

So even armed with the knowledge that `for` iterates directly over the 
items contained in a list it seems people still have trouble with the 
question *"How do I loop through a list of lists in Python?"*

First we'll define a list of lists <small>(*"nested list"*)</small>

```python
>>> list_of_lists = [[1, 2, 3], [4, 5, 6]]
```

We can use `len()` to see that it contains 2 *items*

```python
>>> len(list_of_lists)
2
```

And each item in this case is a list.

```python
>>> for my_list in list_of_lists:
...     print(my_list)
... 
[1, 2, 3]
[4, 5, 6]
```

So we have a list in `my_list` and we know how to iterate
over a list already e.g. `for num in my_list` meaning the 
combination will look like

```python
>>> for my_list in list_of_lists:
...     for num in my_list:
...         print(num)
...
1
2
3
4
5
6
```

Having a `for` loop inside another `for` loop like this is commonly
referred to as *"nested for loops"*.

The super magic powers of `for` are not limited to just lists.

## string

When you iterate over a *"string"* with `for` you will iterate over
each *"character"* in the string.

```python
>>> for char in 'abcd':
...     print(char)
...
'a'
'b'
'c'
'd'
```

## dict 

With a dict <small>(*"dictionary"*)</small> you will iterate over the keys.

```python
>>> for key in {'foo': 'bar', 'omg': 'lol'}:
...     print(key)
...
'omg'
'foo'
```

If you wanted to also get the values you could use
dict indexing e.g. `dict_name[key_name]`

Let's create a dict with a name to show an example.

```python
>>> students = {'bob': 19, 'alice': 23, 'ted': 22}
>>> for name in students:
...     print(name, students[name])
... 
bob 19
alice 23
ted 22
```

dicts also have an `items()` method which will give you
back an iterable of `key, value` pairs (i.e. tuples)

```python
>>> students.items()
dict_items([('bob', 19), ('alice', 23), ('ted', 22)])
```

We could iterate over it like so

```python
>>> for name, age in students.items():
...    print(name, age)
...
bob 19
alice 23
ted 22
```

If you wanted just the values from a dict you could use the 
`.values()` method.

## tuple

Iterating over a tuple will have the same behaviour as the list
example you iterate over each item.

```python
>>> for num in 10, 11, 12:
...     print(num)
... 
10
11
12
```

You may be wondering why there are no `()` there? 

> Note that tuples are not formed by the parentheses, but rather by use of the comma operator

Some people seem to think that you need to declare 
tuples using the `t = (1, 2)` syntax but that is not the case.

```python
>>> t = 1, 2
>>> t
(1, 2)
```

There's also the strange looking example of declaring a single item tuple.

```python
>>> t = 1,
>>> t
(1,)
>>> type(t)
<class 'tuple'>
```

You would need `()` for example if you wanted to create a nested tuple structure.

```python
>>> t = (1, 2), 3
>>> t
((1, 2), 3)
```

So `()` can be used to give Python a *"hint"* with regards to how an expression 
should be parsed if there is any ambiguity.

## file

Finally another very useful feature is when dealing with *"files"*. 

```python
>>> print(open('myfile').read(), end='')
This Is
My example FILE
with 3 lines
```

<small>(If you're using Python 2 you can use `from __future__ import print_function`
to get access to the superior `print()` function from Python 3)</small>

When you iterate over a *"file object"* with `for` you iterate
over each line.

```python
>>> with open('myfile') as f:
...     for line in f:
...         print(line, end='')
... 
This Is
My example FILE
with 3 lines
```

This is the *idiom* used to read a file *line-by-line* in Python. 
Using the `with` block will automatically call `.close()` on the 
*"file object"* when the block exits. 
