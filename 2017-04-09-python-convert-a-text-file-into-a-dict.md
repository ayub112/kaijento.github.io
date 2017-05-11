---
title: "Python: convert a text file into a dictionary"
date:  2017-04-09 06:24:34 +0100
tags:
  - python
  - regex
---

Given the following data in a text file the task is
to convert it into a Python dict having the command
names as the keys and the command descriptions as the
values.

`commands.txt`

```
ABBREV                                          Abbreviation processsor
ADDISK                                               (Operator command)
ADD_REMOTE_ID                        Sets up user id for remote systems
AMLC                                                 (Operator command)
ARCHIVE                                     Archives disk files to tape
ARCHIVE_RELEASE                 Releases an ARCHIVE tape for future use
ARCHIVE_RESTORE                 Restores files archived on tape to disk
```

We will start with some output:

```bash
$ python build-commands.py 
{
    "ABBREV": "Abbreviation processsor", 
    "ADDISK": "(Operator command)", 
    "ADD_REMOTE_ID": "Sets up user id for remote systems", 
    "AMLC": "(Operator command)", 
    "ARCHIVE": "Archives disk files to tape", 
    "ARCHIVE_RELEASE": "Releases an ARCHIVE tape for future use", 
    "ARCHIVE_RESTORE": "Restores files archived on tape to disk"
}
```

The code that generated it:

```python
import json

filename = 'commands.txt'

commands = {}
with open(filename) as fh:
    for line in fh:
        command, description = line.strip().split(' ', 1)
        commands[command] = description.strip()

print(json.dumps(commands, indent=4, sort_keys=True))
```

The `json` module is only being used here as a way to pretty-print our dict.
If you've never seen `with` before it's commonly used for opening files.
When the `with` block is exited it will automatically call `.close()` on the
file for us.

## split()

You have probably encountered`split()` before but its second argument may be new to
you.

Note that in Python 3 you can pass it as the named argument `maxsplit`. 

```python
>>> line = 'ABBREV                                          Abbreviation processsor\n'
>>> line.split()
>>> ['ABBREV', 'Abbreviation', 'processsor']
>>> line.split(' ')
>>> ['ABBREV', '', '', '', '', '', '', '', '', '', '', '', 
'', '', '', '', '', '', '', '', '', '', '', '', '', '', '', '',
'', '', '', '', '', '', '', '', '', '', '', '', '', '', 
'Abbreviation', 'processsor\n']
>>> line.split(' ', 1)
['ABBREV', '                                         Abbreviation processsor\n']
```

So instead of splitting on each space it splits on the first space only which leaves 
the description string intact albeit with surrounding whitespace that needs to be trimmed.

```python
>>> line.strip().split(' ', 1)
['ABBREV', '                                         Abbreviation processsor']
>>> command, description = line.strip().split(' ', 1)
>>> command
'ABBREV'
>>> description
'                                         Abbreviation processsor'
>>> description.strip()
'Abbreviation processsor'
```

So now we have `command` and `description` trimmed correctly we simply add them to 
the dict.

There is, however, another possible approach.

## split(None, 1)

Instead of specifying to split on a space character we can however take advantage of the
default behaviour of `split()` which is `sep=None`:

```python
>>> line.split(' ', 1)
['ABBREV', '                                         Abbreviation processsor\n']
>>> line.split(None, 1)
['ABBREV', 'Abbreviation processsor\n']
>>> line.strip().split(None, 1)
['ABBREV', 'Abbreviation processsor']
```

In Python 3 you could also use the named-argument approach i.e. `.split(maxsplit=1)`

This would allow us to use `dict()` with a <em>generator statement</em> to construct our
dict:

```python
>>> with open(filename) as fh:
...     commands = dict(line.strip().split(None, 1) for line in fh)
... 
>>> print(json.dumps(commands, indent=4, sort_keys=True))
{
    "ABBREV": "Abbreviation processsor", 
    "ADDISK": "(Operator command)", 
    "ADD_REMOTE_ID": "Sets up user id for remote systems", 
    "AMLC": "(Operator command)", 
    "ARCHIVE": "Archives disk files to tape", 
    "ARCHIVE_RELEASE": "Releases an ARCHIVE tape for future use", 
    "ARCHIVE_RESTORE": "Restores files archived on tape to disk"
}
```

## dict(re.findall())

Finally you may have experience using regular expressions in which
case you could pass the result of `re.findall()` directly to `dict()`
to create the desired result:

```python
>>> import re
>>> 
>>> with open(filename) as fh:
...     commands = dict(re.findall(r'(\S+)\s+(.+)', fh.read()))
... 
>>> print(json.dumps(commands, indent=4, sort_keys=True))
{
    "ABBREV": "Abbreviation processsor", 
    "ADDISK": "(Operator command)", 
    "ADD_REMOTE_ID": "Sets up user id for remote systems", 
    "AMLC": "(Operator command)", 
    "ARCHIVE": "Archives disk files to tape", 
    "ARCHIVE_RELEASE": "Releases an ARCHIVE tape for future use", 
    "ARCHIVE_RESTORE": "Restores files archived on tape to disk"
}
```

`(\S+)` matches and captures the command name i.e. <em>1 or more non-whitespace
characters</em>. `\s+` then matches the sequence of <em>1 or more whitespace 
characters</em> in between the command name and description. 

Finally, the description is captured with `(.+)`

It may also be worth noting that with this approach we're passing the full 
file contents as a single string with the `fh.read()` call although that
could be changed if needed.
