---
title: "Python: remove_duplicates"
date:  2017-04-08 07:04:50 +0100
tags:
 - python
---

<em>
Write a function called remove_duplicates which will take one argument called string. 
This string input will only have characters between a-z.  
The function should remove all repeated characters in the string and return a tuple with two values:  
  A new string with only unique, sorted characters.  
  The total number of duplicates dropped.
</em>

```python
remove_duplicates('aaabbbac')  => ('abc', 5)
remove_duplicates('a')         => ('a', 0)
remove_duplicates('thelexash') => ('aehlstx', 2)
```

## solution

```python
>>> def remove_duplicates(string):
...     unique  = ''.join(sorted(set(string)))
...     removed = len(string) - len(unique)
...     return unique, removed
... 
>>> remove_duplicates('aaabbbac')
('abc', 5)
>>> remove_duplicates('a')
('a', 0)
>>> remove_duplicates('thelexash')
('aehlstx', 2)
```

## explanation

A `set` is a container for storing items similar to a `list`. The main differences
being that they are unordered and they cannot store duplicated items:

```python
>>> s = set()
>>> s.add('one')
>>> s
set(['one'])
>>> s.add('one')
>>> s.add('one')
>>> s
set(['one'])
>>> s.add('two')
>>> s
set(['two', 'one'])
```

We added `'one'` multiple times but as the values of a set are unique the subsequent
adds are ignored. 

We can also pass an iterable to `set()`:

```python
>>> set('aaabbbac')
set(['a', 'c', 'b'])
>>> set(['bob', 'alice', 'bob'])
set(['bob', 'alice'])
```

A common question is <em>"How do I remove duplicates from a list?"</em> and you can
do that using sets.

So let's step through the code to see what's going on:

```python
>>> string = 'thelexash'
>>> set(string)
set(['a', 'e', 'h', 'l', 's', 't', 'x'])
>>> sorted(set(string))
['a', 'e', 'h', 'l', 's', 't', 'x']
>>> ''.join(sorted(set(string)))
'aehlstx'
```

As we cannot rely on the order we use `sorted()` to sort the set for us
and give us back a list. We then use `''.join()` to turn the list
into a string containing the unique characters.

Now that we have the unique characters how do we calculate how many were 
removed? We can simply take the length of the unique character string
and substract it from the length of the original string.

```python
>>> unique = ''.join(sorted(set(string)))
>>> len(string) - len(unique)
2
```

We then have the 2 values needed which we `return` from the function as a tuple.
