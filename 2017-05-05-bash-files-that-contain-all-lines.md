---
title: "bash: files that contain all lines from file"
date:  2017-05-10 07:33:04 +0100
tags:
 - bash
 - python
---

*Given `file-a` I need to find all files recursively ("using bash")
that contain all of the lines from `file-a` regardless of order.
How can I do this?*

Usually *"using bash"* just means *"from the command-line"* so
we're just going to skip ahead directly to `Python` because
doing this *"in bash"* is a world of pain.

Let's create some example files for testing.

```bash
$ cat all-lines/file-a
a
b
c
$ cat all-lines/dir/file1
c
b
a
$ cat all-lines/dir/file2
a
b
c
d
$ cat all-lines/dir/file3
d
c
e
```

So we have `2` matching files here: `file1` and `file2`.

## Code

First we have the output.

```bash
$ python find-files.py
all-lines/dir/file1
all-lines/dir/file2
```

The code that generated it.

```python
import os

file_a    = 'all-lines/file-a'
start_dir = 'all-lines/dir'

with open(file_a) as f:
    lines_a = set(line for line in f)

for root, dirs, files in os.walk(start_dir):
    for filename in files:
        path = os.path.join(root, filename)
        with open(path) as f:
            lines_b = set()
            for line in f:
                lines_b.add(line)
                #if lines_a.issubset(lines_b):
                if lines_a <= lines_b:
                    print(path)
                    break
```

If you've not seen [the with statement](https://docs.python.org/3/reference/compound_stmts.html#with) 
before it's being used here as when the block exits it will
automatically call `close()` on the file for us.

We're creating the set `lines_a` to store all the lines
from `file-a` which we will use to compare against the 
other files. 

Two things to note:

* this assumes `file-a` fits into memory
* line endings are not being removed

If we wanted to ignore line endings we could instead store
`line.rstrip('\r\n')` in our `lines_a` and `lines_b` sets.

A [set](https://docs.python.org/3/tutorial/datastructures.html#sets) 
is an unordered collection with no duplicate elements. Set objects 
also support mathematical operations like union, intersection, 
difference, and symmetric difference.

```python
>>> x = set('abcd')
>>> y = set('abcdef')
>>> 
>>> x <= y
True
>>> x.issubset(y)
True
```

`<=` here is the same as using [issubset()](https://docs.python.org/3/library/stdtypes.html#set.issubset)
which provides a simple way to test if all the elements in `x` are contained in `y`.

## os.walk()

To process a directory structure recursively in Python we can use [os.walk()](https://docs.python.org/3/library/os.html#os.walk)
which would be like using the `find` command.

`os.walk()` yields a 3-tuple `(dirpath, dirnames, filenames)`

* `dirpath` being the path to the current directory being processed
* `dirnames` being a list of directory names contained in `dirpath`
* `filenames` being a list of file names contained in `dirpath`

As `filenames` only gives us the names and not the full path we use `os.path.join(root, f)` 
to build the full path to the filename in our code. Also note that `root` appears to be
the name most commonly used in examples to store the `dirpath` value.

So for each file we create our `lines_b` set and process the file
*line-by-line* adding each line to the set. 

The reason we're processing each `file-b` line-by-line and using `add()` 
is because this allows us to stop processing the file as soon as there 
is a match. If there is a match we call `break` which exits the `for` 
loop immediately.

If we had a large file that contained a match in the first few lines
this would save from having to process the full file. It could also
allow us to process files that are *"too large to fit into memory"*.
