---
title: "Python: add line to file if not already there"
date: 2017-03-22 10:53:42 +0100
tags:
 - python
 - regex
---

So the goal is to search through a file looking for a line that matches a
particular pattern. If there is no match add a new line to the end 
of the file i.e. <em>"append a line to a file"<em>.

## with open r+

To open a file for reading and writing in Python you can use `r+` e.g.

```python
with open('myfile', 'r+') as fh:
    for line in fh:
        ...
```

If you have not seen the `with` statement before it will automatically call `close()` 
on the file when the block is exited so we don't have to do:

```python
fh = open('myfile', 'r+')
for line in fh:
   ...
fh.close()
```

`fh` here is short for "filehandle". Feel free to choose whatever name you prefer.

## Sample input

Let's get some sample input:

`lines.txt`

```
line 1
line 2
line 3
line 4
line 5
```

There are many ways to go about doing this we will start with simply processing the file
line-by-line.

For this particular task the goal was to search for a line starting with `Message-ID:` which
can be done simply by using the `startswith()` method.

`add-line.py`

```python
from __future__ import print_function

with open('lines.txt', 'r+') as fh:
    for line in fh:
        if line.startswith('Message-ID:'):
            print('Found.')
            print(line, end='')
            break
    else:
        print('Message-ID: omg lol', file=fh)
```

## Fur Elise? WTF?

Your eyes have not deceived you. `for` loops can have an `else` in 
Python. If no `break` statement is executed within the loop the `else` 
clause will be entered. 

A `break` statement will exit immediately from a for loop.

```python
>>> for name in 'Alice', 'Bob', 'Frank', 'Ted':
...     name
...     if name == 'Bob':
...         break 
... 
'Alice'
'Bob'
```

The `__future__` line gives Python 3's `print()` functionality if we're running on Python 2.
We're using it here for `end=''` which prevents print from adding a line return. Without it
we would have to `line.rstrip('\n')`. It also allows us to use `print(..., file=fh)` which
will add the line ending for us to save us from using `fh.write(line + '\n')`.

So let's see it in action:

```bash
$ python add-line.py lines.txt
Not found.
$ cat lines.txt
line 1
line 2
line 3
line 4
line 5
Message-ID: omg lol
$ python add-line.py lines.txt
Found.
Message-ID: omg lol
```
As previously mentioned there are many different ways one could approach this task...

## stand back... I know regex

```bash
$ python add-line-re.py lines.txt
$ cat lines.txt
line 1
line 2
line 3
line 4
line 5
Message-ID: omg lol
$ python add-line-re.py lines.txt
Found.
```

And the code:

`add-line-re.py`

```python
import re

with open('lines.txt', 'r+') as fh:
    text = fh.read()
    if re.search(r'(?m)^Message-ID:', text):
        print('Found.')
    else:
        fh.write('Message-ID: omg lol\n')
```

`fh.read()` reads all of the contents into a single string (you may see this 
referred to as <em>"slurping"</em>a file). If the file is too large to fit 
into memory this will raise an exception whereas with processing line-by-line 
there are no such constraints. (Unless of course the lines themselves are too
big... in which case... `¯\_(ツ)_/¯`)

## insert a new line at a certain line number

The requirements of this task then changed to stipulate that they now wanted to
<em>"add a new line at a particular line number"</em> i.e. insert a new line.
Specifically they wanted to insert a new line 3 e.g.

```
line 1
line 2
line 3
line 4
line 5
```

turns into:

```
line 1
line 2
Message-ID: lol omg
line 3
line 4
line 5
```

So line 3 becomes line 4, line 4 becomes line 5, etc. 

## seek()

```bash
$ python add-line-n.py lines.txt
Not found.
$ cat lines.txt
line 1
line 2
Message-ID: omg lol
line 3
line 4
line 5
$ python add-line-n.py lines.txt
Found.
Message-ID: omg lol
```

Here's the code:

`add-line-n.py`

```python
from __future__ import print_function

line_number = 3

with open('lines.txt', 'r+') as fh:
    lines = fh.readlines()
    for line in lines:
        if line.startswith('Message-ID:'):
            print('Found.')
            print(line, end='')
            break
    else:
        print('Not found.')
        fh.seek(0)
        lines.insert(line_number - 1, 'Message-ID: omg lol\n')
        fh.writelines(lines)
```

So we've had to use `readlines()` to store all the lines into a list. We loop through
looking for a match like the first solution. If there is no match found we use
`fh.seek(0)` to move the file pointer back to the beginning of the file because when we used
`readlines()` all of the data was read and the file pointer was at the end of the file so we must
point it back to the start before overwriting the content.

We use the `list.insert()` method to insert a new item at a specific index. Because lists are 0-indexed
we must `- 1` from the line number where we want to add the new line. 

```python
>>> lines = ['a', 'b', 'c', 'd']
>>> lines.insert(2, 'omg new')
>>> lines
['a', 'b', 'omg new', 'c', 'd']
```

We have also resorted back to using `writelines()` as all our lines have line endings. We could have instead used

```python 
...
lines = fh.read().splitlines()
...
for line in lines:
    print(line, file=fh)
```

`splitlines()` will take a string and split it up into a list of lines whilst removing the line endings.

## seek() and destroy()

You may ask seeing as we can `seek(0)` back to the start could be not then skip to where we want to
insert our new line instead of having to store all the lines with `readlines()`?

```python
fh.seek(0)
fh.readline()
fh.readline()
fh.write('looooooooooooooooooooool\n')
```

```bash
$ cat lines.txt
line 1
line 2
looooooooooooooooooooool
```

The answer is yes we can do that however we will destroy any existing data when we `write()`.

## stand back again... I still know regex...

```python
import re

line_number = 3

with open('lines.txt', 'r+') as fh:
    text = fh.read()
    if re.search(r'(?m)^Message-ID:', text):
        print('Found.')
    else:
        print('Not found.')
        pattern = r'^((?:[^\n]+\n){% raw %}{%d}{% endraw %})' % (line_number - 1)
        fh.seek(0)
        fh.write(
            re.sub(pattern, r'\1Message-ID: omg lol\n', text)
        )
```
    
So we've essentially just reimplemented the following:

    grep -q '^MessageID:' file || sed -i '3iMessageID: omg lol' file

Also if you'd like to stand back a little further you can actually do it in a single regex
without the need for the `search()` check:

`add-line-n-single-re.py`

```python
import re

line_number = 3

with open('lines.txt', 'r+') as fh:
    text = fh.read()

    pattern = (
        r'(?m)\A((?:(?!^Message-ID:).+\n){% raw %}{%d}{% endraw %})' 
        r'((?:(?!^Message-ID:).+\n)*)\Z' % (line_number - 1)
    )

    fh.seek(0)
    fh.write(
        re.sub(pattern, r'\1Message-ID: omg lol\n\2', text)
    )
```

All of these approaches slurp the file. Is there a way to insert by reading the file line-by-line?

Generally the way to do this involves processing the file in 2 passes. First to check if there is
a match then a second time to make our edits. However to make an edit we need to store the whole
file in memory. Pls halp?

## fileinput

The answer is to read it line-by-line whilst writing output line-by-line to a temporary file. 
Afterwards you then overwrite the original file with the temporary file. You can do this manually, 
but Python's `fileinput` module does most of the work for you.

```bash
$ python add-line-n-fileinput.py 
$ cat lines.txt
line 1
line 2
Message-ID: lol
line 3
line 4
line 5
```

The code:

`add-line-n-fileinput.py`

```python
from __future__ import print_function

filename    = 'lines.txt'
line_number = 3

add_line = True
with open(filename) as fh:
    for line in fh:
        if line.startswith('Message-ID:'):
            add_line = False
            break

if add_line:
    fh = fileinput.input(filename, inplace=True)
    for n, line in enumerate(fh, start=1):
        if n == line_number:
            print('Message-ID: lol')
        print(line, end='')
```
So without `inplace=True` it would just print the output. When using `inplace=True` 
the content of the original file will be replaced by any output generated from `print()`.

There is no `else` this time because we need to re-open the file using `fileinput` so 
we use `add_line` as a boolean we can check against after the first pass.

We're also using `enumerate()` with `start=1` to give us a line count that we can
compare against `line_number`.
